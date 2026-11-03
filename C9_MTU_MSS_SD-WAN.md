# Cours MTU : MTU, MSS et fragmentation dans les réseaux SD-WAN

Ce cours est entièrement consacré au MTU et aux notions associées (MSS, fragmentation, Path MTU Discovery), dans le contexte d'un réseau SD-WAN. Il suppose acquises les bases du SD-WAN et du ZTP. Le fil rouge est le réseau de magasins d'un distributeur de type Carrefour, où les problèmes de MTU se manifestent de façon très concrète : caisses lentes, paiements en échec, applications qui "bloquent" sans raison apparente.

> Note : le scénario Carrefour est un cas d'école. Chiffres, noms de sites, adresses IP et incidents sont fictifs et ne décrivent pas l'organisation réelle de l'enseigne. Les valeurs de MTU par défaut des éditeurs et des opérateurs sont données à titre indicatif : mesurez et vérifiez toujours sur votre plateforme.

---

## Table des matières

1. Pourquoi le MTU est un sujet central en SD-WAN
2. Définitions : MTU, MRU, MSS, PMTU
3. Fragmentation IPv4 et IPv6
4. Path MTU Discovery et le piège du trou noir ICMP
5. Calcul de l'overhead : encapsulations courantes
6. MTU et SD-WAN : tunnels, transports, couleurs
7. MSS clamping : principe et calcul
8. Symptômes d'un problème de MTU
9. Méthodes de diagnostic
10. Exemples de configuration
11. Cas pratique : magasin de proximité en ADSL et 4G
12. Cas pratique : paiement qui échoue après le handshake TLS
13. Cas pratique : vidéosurveillance et jumbo frames au data center
14. Conception : choisir ses valeurs de MTU et de MSS
15. Plan de test et de validation
16. Bonnes pratiques et pièges
17. Exercices et corrigés
18. Quiz de révision
19. Glossaire

---

## 1. Pourquoi le MTU est un sujet central en SD-WAN

Un overlay SD-WAN **encapsule** chaque paquet utilisateur dans un tunnel (IPsec, parfois GRE). L'encapsulation ajoute des octets. Or les liens sous-jacents (fibre, ADSL, 4G, 5G, MPLS) ont une taille maximale de paquet qui, elle, ne bouge pas.

Conséquence : un paquet de 1500 octets, parfaitement normal dans le LAN du magasin, ne passe plus tel quel dans le tunnel. Il faut soit le fragmenter, soit l'empêcher d'être aussi grand. Si ce point est mal traité, on obtient des pannes **partielles et déroutantes** : le ping fonctionne, mais l'application ne marche pas.

Enjeux en distribution :

| Contexte | Risque lié au MTU |
|---|---|
| Caisses vers serveurs centraux | Lenteurs, timeouts, transactions bloquées |
| Terminaux de paiement | Échec après la négociation TLS |
| ADSL en PPPoE | MTU de transport réduit, cumul avec l'overhead IPsec |
| 4G et 5G de secours | MTU variable selon l'opérateur, souvent inférieur à 1500 |
| Vidéosurveillance, sauvegardes | Gros volumes, forte sensibilité à la fragmentation |
| Data center | Jumbo frames possibles, risque d'incohérence avec le WAN |

---

## 2. Définitions : MTU, MRU, MSS, PMTU

### 2.1 MTU (Maximum Transmission Unit)

Taille maximale, en octets, d'un **paquet de couche 3** (en-tête IP inclus) pouvant être transmis sur une interface sans fragmentation.

- Sur Ethernet standard : **1500 octets**.
- Le MTU ne comprend pas l'en-tête Ethernet (14 octets, plus 4 pour une étiquette VLAN 802.1Q).
- La trame Ethernet maximale vaut donc 1514 octets (1518 avec VLAN) hors FCS selon la façon de compter.

### 2.2 MRU (Maximum Receive Unit)

Taille maximale d'un paquet que l'interface accepte de recevoir. Dans la plupart des cas MRU et MTU sont égaux, mais certains protocoles (PPP) les négocient séparément.

### 2.3 MSS (Maximum Segment Size)

Taille maximale de la **charge utile TCP** (données sans en-têtes IP et TCP) qu'un hôte accepte de recevoir dans un segment. Elle est annoncée dans l'option MSS du paquet TCP SYN.

Relation :

```
MSS IPv4 = MTU - 20 (en-tête IP) - 20 (en-tête TCP) = MTU - 40
MSS IPv6 = MTU - 40 (en-tête IPv6) - 20 (en-tête TCP) = MTU - 60
```

Pour un MTU de 1500 : MSS IPv4 = 1460, MSS IPv6 = 1440.

Le MSS ne concerne que TCP. UDP (voix, vidéo, DNS, QUIC) n'a pas de mécanisme équivalent.

### 2.4 PMTU (Path MTU)

Plus petit MTU rencontré sur **l'ensemble du chemin** entre deux hôtes. C'est la valeur réellement utilisable de bout en bout.

### 2.5 Schéma récapitulatif

```
  Trame Ethernet
  +----------+----------------------------------------------+-----+
  | En-tête  |             Paquet IP (MTU = 1500)           | FCS |
  | Ethernet |  +----------+----------+------------------+  |     |
  | 14 octets|  | IP  (20) | TCP (20) | Données (MSS)    |  |     |
  +----------+--+----------+----------+------------------+--+-----+
                                      <---- 1460 octets --->
```

---

## 3. Fragmentation IPv4 et IPv6

### 3.1 IPv4

Quand un routeur doit transmettre un paquet plus grand que le MTU de l'interface de sortie :

- si le bit **DF (Don't Fragment)** est à 0, le routeur **fragmente** le paquet en plusieurs morceaux, chacun avec son en-tête IP ;
- si le bit DF est à 1, le routeur **abandonne** le paquet et renvoie un message ICMP "Destination Unreachable, Fragmentation Needed" (type 3, code 4), en indiquant le MTU du prochain saut.

Le réassemblage se fait **uniquement à la destination finale**.

### 3.2 Coûts de la fragmentation

| Impact | Explication |
|---|---|
| Performances CPU | Fragmenter et réassembler consomme des ressources, surtout sur les routeurs d'accès |
| Perte amplifiée | La perte d'un seul fragment oblige à retransmettre tout le paquet d'origine |
| Pare-feu et NAT | Les fragments (sauf le premier) ne portent pas d'en-tête TCP ou UDP, ce qui complique le filtrage |
| Sécurité | Les fragments sont une surface d'attaque connue (fragments chevauchants, saturation de la file de réassemblage) |
| Débit | Chute de débit sensible dès qu'une fraction notable du trafic est fragmentée |

### 3.3 IPv6

- Les **routeurs ne fragmentent jamais** : seul l'émetteur peut le faire, via un en-tête d'extension de fragmentation.
- Le MTU minimal d'un lien IPv6 est de **1280 octets**.
- Si un paquet est trop gros, le routeur l'abandonne et renvoie un message ICMPv6 **Packet Too Big** (type 2).

Conséquence : en IPv6, bloquer l'ICMPv6 type 2 casse la communication de façon plus franche encore qu'en IPv4.

---

## 4. Path MTU Discovery et le piège du trou noir ICMP

### 4.1 Principe de la PMTUD

Pour éviter la fragmentation, un hôte peut **découvrir** la PMTU :

1. Il envoie ses paquets avec le bit DF à 1.
2. Si un routeur intermédiaire ne peut pas transmettre le paquet, il le jette et renvoie un ICMP de type 3 code 4 (IPv4) ou ICMPv6 type 2 (IPv6), avec le MTU à respecter.
3. L'émetteur réduit la taille de ses paquets et retransmet.

### 4.2 Le trou noir PMTUD (PMTUD black hole)

Si un pare-feu ou un équipement intermédiaire **filtre l'ICMP**, l'émetteur ne reçoit jamais le message. Il continue d'envoyer des paquets trop gros, qui sont abandonnés en silence.

Symptômes typiques :

- le handshake TCP (petits paquets) passe ;
- les premières données passent, puis la session se fige dès que de gros paquets circulent ;
- un transfert s'arrête à un volume précis, ou une page web se charge à moitié.

```
Hôte A ----[1500]----> R1 ----[1500]----> R2 ----[MTU 1400]----> R3
                                          |
                                          X  Paquet abandonné
                                          |  (DF=1), ICMP "fragmentation needed"
                              <---ICMP-----+
                                 bloqué par un pare-feu : A ne le reçoit jamais
```

### 4.3 PLPMTUD (Packetization Layer PMTUD)

Variante qui ne dépend pas de l'ICMP : la couche de transport (par exemple TCP avec l'option activée, ou QUIC) teste des tailles de paquets et en déduit la PMTU en observant les pertes. Plus robuste, mais pas toujours activée par défaut selon les systèmes.

### 4.4 Règle de bon sens pour les pare-feux

Laisser passer au minimum :

- ICMP type 3 code 4 (IPv4, fragmentation needed) ;
- ICMPv6 type 2 (Packet Too Big) ;
- ICMP echo (ping) pour le diagnostic, selon la politique de sécurité.

---

## 5. Calcul de l'overhead : encapsulations courantes

### 5.1 Tableau des surcoûts usuels

| Encapsulation | Overhead (octets) | MTU résultant sur lien 1500 |
|---|---|---|
| Ethernet standard | 0 | 1500 |
| PPPoE | 8 | 1492 |
| Étiquette VLAN 802.1Q | 4 (niveau trame, MTU L3 inchangé si le lien l'accepte) | 1500 |
| GRE (IPv4) | 24 (20 IP + 4 GRE) | 1476 |
| VXLAN | 50 | 1450 |
| IPsec ESP, mode tunnel | environ 50 à 73 selon l'algorithme et le padding | environ 1430 à 1450 |
| IPsec ESP avec NAT-T (UDP 4500) | 8 octets de plus qu'ESP seul | environ 1425 à 1440 |
| GRE sur IPsec | cumul des deux | environ 1400 à 1420 |

Ces valeurs sont indicatives. Les chiffres réels dépendent des algorithmes de chiffrement et d'authentification.

### 5.2 Exemple détaillé : IPsec ESP en mode tunnel avec NAT-T, AES-GCM

| Élément | Octets |
|---|---|
| Nouvel en-tête IPv4 externe | 20 |
| En-tête UDP (NAT-T) | 8 |
| En-tête ESP (SPI plus numéro de séquence) | 8 |
| Vecteur d'initialisation (IV) AES-GCM | 8 |
| Fin ESP : longueur du padding plus champ "next header" | 2 |
| Padding (alignement, 0 à 3 octets) | 0 à 3 |
| Valeur de contrôle d'intégrité (ICV, GCM) | 16 |
| **Total** | **62 à 65** |

Sur un lien de MTU 1500, le paquet interne maximal avant encapsulation est donc d'environ :

```
1500 - 65 = 1435 octets (cas pire, padding maximal)
1500 - 62 = 1438 octets (cas favorable)
```

Les éditeurs affichent souvent une valeur par défaut de MTU de tunnel un peu plus conservatrice. La documentation Cisco Catalyst SD-WAN mentionne par exemple une valeur de l'ordre de 1442 octets pour un tunnel IPsec sur lien en 1500 (à confirmer pour votre version).

### 5.3 Cumul : transport dégradé plus tunnel

Exemple : lien ADSL en PPPoE (MTU 1492) plus tunnel IPsec :

```
1492 - 65 = 1427 octets pour le paquet interne (cas pire)
```

Exemple : lien 4G dont l'opérateur impose un MTU de 1400 plus tunnel IPsec :

```
1400 - 65 = 1335 octets pour le paquet interne
```

Les tailles d'un paquet TCP "plein" de 1500 octets, issu du LAN du magasin, dépassent donc largement ces limites : sans correction, il sera fragmenté ou abandonné.

### 5.4 MTU des liens mobiles

Le MTU d'un accès 4G ou 5G n'est pas standardisé de façon universelle. Selon les opérateurs, les APN et les passerelles, on observe souvent des valeurs inférieures à 1500 (par exemple autour de 1400 à 1450), parfois différentes selon le forfait. **Il faut le mesurer** (voir section 9), pas le supposer.

---

## 6. MTU et SD-WAN : tunnels, transports, couleurs

### 6.1 Où se joue le MTU dans l'overlay

Dans un edge SD-WAN, plusieurs valeurs coexistent :

| Valeur | Où | Rôle |
|---|---|---|
| MTU de l'interface physique (WAN) | Interface du lien (VPN 0) | MTU du transport sous-jacent |
| MTU de l'interface de tunnel | Dérivé ou configuré | Taille maximale du paquet interne à encapsuler |
| MTU des interfaces de service (LAN) | VPN de service | Taille maximale vue par les hôtes du magasin |
| MSS imposé (clamping) | Interface de service ou de tunnel | Limite les segments TCP des hôtes |

### 6.2 Une couleur, un MTU

Chaque transport (couleur) peut avoir un MTU sous-jacent différent. Exemple dans un magasin :

| Couleur | Transport | MTU sous-jacent probable | Paquet interne maximal approximatif |
|---|---|---|---|
| `biz-internet` | Fibre FAI (Ethernet) | 1500 | environ 1435 |
| `mpls` | MPLS opérateur | 1500 (parfois moins selon l'opérateur) | environ 1435 ou moins |
| `lte` | 5G de secours | 1400 à 1450 (variable) | environ 1335 à 1385 |
| `biz-internet` | ADSL PPPoE | 1492 | environ 1427 |

Pour qu'une session ne casse pas lors d'une **bascule** de transport, les hôtes doivent émettre des paquets qui passent **sur le plus contraignant** des chemins possibles. C'est la raison d'être du MSS clamping unique, calé sur le pire cas.

### 6.3 Effets de la bascule

Une application TCP établie sur la fibre avec un MSS de 1360 peut se retrouver sur un lien 5G où l'MTU est plus faible. Si le MSS n'a pas été prévu pour ce cas, la session se dégrade (fragmentation, pertes) ou se fige juste après la bascule.

### 6.4 Découverte du PMTU sur les tunnels

Certaines plateformes SD-WAN proposent une **découverte automatique du MTU de tunnel** (envoi de sondes de tailles croissantes pour déterminer ce que le transport accepte). Cela fiabilise les liens atypiques (4G, ADSL), au prix d'un peu de trafic de sonde et de dépendance à la version logicielle. Vérifiez la disponibilité et le comportement par défaut sur votre version.

### 6.5 Fonctions qui ajoutent de l'overhead

- **FEC (Forward Error Correction)** : paquets de redondance, éventuellement plus gros.
- **Duplication de paquets** : chaque paquet est envoyé deux fois, la taille unitaire est inchangée, mais la charge double.
- **Compression, optimisation TCP** : peuvent modifier la taille des segments.
- **Encapsulations supplémentaires** : si le site passe par un service SASE ou un tunnel vers un cloud de sécurité, un second niveau d'overhead s'ajoute.

---

## 7. MSS clamping : principe et calcul

### 7.1 Principe

Le **MSS clamping** (ou "TCP MSS adjust") consiste, pour l'edge, à **réécrire l'option MSS** dans les paquets SYN et SYN-ACK qui le traversent, afin qu'aucun hôte n'envoie des segments plus gros que ce que le chemin supporte.

```
Hôte caisse                Edge SD-WAN                   Serveur central
    |--- SYN (MSS 1460) ------->|                              |
    |                           |--- SYN (MSS 1350) ---------->|
    |                           |<-- SYN-ACK (MSS 1460) -------|
    |<-- SYN-ACK (MSS 1350) ----|                              |
    |     Les deux côtés utilisent MSS 1350 pour les données   |
```

Avantage : évite la fragmentation **sans dépendre de l'ICMP**. Limite : ne protège que TCP.

### 7.2 Formule

```
MSS cible = PMTU interne - 40 (IPv4)
PMTU interne = MTU du pire transport - overhead du tunnel
```

### 7.3 Exemples de calcul

| Pire transport du site | MTU transport | Overhead tunnel (pire cas) | PMTU interne | MSS cible (IPv4) |
|---|---|---|---|---|
| Fibre Ethernet | 1500 | 65 | 1435 | 1395 |
| ADSL PPPoE | 1492 | 65 | 1427 | 1387 |
| 5G à 1400 | 1400 | 65 | 1335 | 1295 |
| 5G à 1450 | 1450 | 65 | 1385 | 1345 |

### 7.4 Le piège de la valeur "universelle"

Une valeur couramment recommandée est un MSS de **1350** : elle convient aux fibres et à l'ADSL, avec une marge confortable.

Mais avec un accès 5G à 1400 :

```
Segment TCP = MSS 1350 + 40 = 1390 octets (paquet interne)
Après encapsulation = 1390 + 65 = 1455 octets
Lien 5G : MTU 1400  ->  1455 > 1400  ->  fragmentation ou abandon
```

Le MSS de 1350 est donc **trop élevé** pour ce lien. Il faut soit un MSS plus bas pour l'ensemble du site (environ 1290), soit un réglage différencié par transport, soit une découverte automatique du PMTU.

### 7.5 IPv6

Pour IPv6, le MSS vaut PMTU interne moins 60. Aligner la valeur de clamping sur le pire cas IPv4 et IPv6 si les deux sont utilisés.

### 7.6 Clamping et MTU d'interface : à utiliser ensemble

- **MTU d'interface** : protège tout le trafic (UDP compris), mais dépend de l'ICMP pour la PMTUD ou provoque de la fragmentation.
- **MSS clamping** : protège TCP sans ICMP.

En pratique, on combine **un MTU de tunnel cohérent** et **un MSS clamping** sur les interfaces de service.

---

## 8. Symptômes d'un problème de MTU

### 8.1 Tableau des symptômes

| Symptôme | Explication probable |
|---|---|
| Le ping fonctionne, l'application non | Les petits paquets passent, pas les gros |
| Le handshake TCP aboutit, puis la session se fige | Premiers segments petits, puis segments pleins abandonnés |
| Une page web charge partiellement | Taille de paquets supérieure à la PMTU, ICMP filtré |
| HTTPS bloqué après "Client Hello" ou "Server Hello" | Messages TLS volumineux (certificats), dépassant la PMTU |
| SSH se connecte puis se bloque à l'ouverture de session | Paquets de grande taille lors de l'échange de clés ou de l'affichage |
| Transferts SMB ou de sauvegarde très lents | Fragmentation ou retransmissions massives |
| Voix correcte, mais vidéo ou données dégradées | La voix (petits paquets RTP) passe, pas les gros paquets |
| Problème uniquement lors d'une bascule | Le MSS ou le MTU n'est pas adapté au lien de secours |
| Un site isolé touché, les autres non | MTU spécifique au transport de ce site (ADSL PPPoE, 4G) |
| VPN client ou accès distant instable | Double encapsulation (tunnel dans tunnel) |

### 8.2 Pourquoi les pannes sont partielles

Le MTU n'affecte que les paquets **dont la taille dépasse le seuil**. Les petits échanges (ping, DNS, handshake) passent. Les échanges volumineux échouent. D'où des tickets du type "ça marche, mais pas toujours".

---

## 9. Méthodes de diagnostic

### 9.1 Test de base avec ping et DF

Taille du paquet IP = taille de la charge ICMP plus 28 octets (20 IP plus 8 ICMP).

| Paquet IP testé | Charge ICMP (`-s` ou `-l`) |
|---|---|
| 1500 | 1472 |
| 1492 | 1464 |
| 1450 | 1422 |
| 1400 | 1372 |
| 1350 | 1322 |

**Windows**

```
ping -f -l 1472 10.10.0.1
```

**Linux**

```
ping -M do -s 1472 10.10.0.1
```

**macOS**

```
ping -D -s 1472 10.10.0.1
```

**Cisco IOS XE**

```
ping 10.10.0.1 size 1500 df-bit
```

Réponse "Packet needs to be fragmented but DF set" ou absence de réponse : la taille est trop grande pour le chemin.

### 9.2 Recherche dichotomique

1. Testez 1472, puis 1400, puis 1436, et ainsi de suite.
2. Notez la plus grande taille qui passe.
3. PMTU = charge maximale plus 28.

### 9.3 Traceroute avec découverte de PMTU

Sous Linux :

```
tracepath 10.10.0.1
```

L'outil affiche le MTU découvert ("pmtu") et signale les sauts qui le réduisent.

### 9.4 Capture de paquets

Capturer simultanément côté LAN et côté WAN de l'edge :

- vérifier la valeur de l'option MSS dans les SYN (avant et après clamping) ;
- repérer les fragments IP (champ "More Fragments", offset non nul) ;
- repérer les messages ICMP "fragmentation needed" ou "packet too big" ;
- observer les retransmissions TCP en rafale après un certain volume.

### 9.5 Mesures sur l'edge SD-WAN

Selon la plateforme, consulter :

- l'état des tunnels et leur MTU effectif ;
- les compteurs de paquets fragmentés ou abandonnés pour cause de MTU ;
- les statistiques de découverte de PMTU de tunnel, si disponibles.

Exemple de commandes Cisco (à adapter à la version) :

```
show sdwan tunnel statistics
show sdwan tunnel statistics table
show interfaces GigabitEthernet0/0/0 | include MTU
show ip interface GigabitEthernet0/0/0 | include MTU
show ip traffic | include fragment
```

### 9.6 Méthode de démonstration à un tiers

Pour prouver un problème de MTU à un opérateur ou à un éditeur :

1. Fournir les résultats de ping avec DF sur plusieurs tailles.
2. Fournir une capture montrant un paquet abandonné sans ICMP en retour.
3. Montrer que la taille limite correspond à une valeur de MTU spécifique (par exemple 1492 pour PPPoE).

---

## 10. Exemples de configuration

Les extraits ci-dessous sont **pédagogiques**. Les noms de paramètres et leur disponibilité dépendent de la plateforme et de la version.

### 10.1 Cisco Catalyst SD-WAN, mode CLI (équipement IOS XE)

Interface de transport (WAN) et tunnel SD-WAN :

```
interface GigabitEthernet0/0/0
 description WAN1-Fibre
 ip address dhcp
 mtu 1500
!
interface Tunnel1
 tunnel source GigabitEthernet0/0/0
 tunnel mode sdwan
 ip mtu 1442
!
sdwan
 interface GigabitEthernet0/0/0
  tunnel-interface
   encapsulation ipsec
   color biz-internet
  exit
 exit
!
```

Interface de service (LAN, VPN 20) avec clamping MSS :

```
interface GigabitEthernet0/1/0.20
 encapsulation dot1Q 20
 vrf forwarding 20
 ip address 10.20.34.1 255.255.255.0
 ip mtu 1500
 ip tcp adjust-mss 1350
!
```

### 10.2 Configuration dans les anciennes syntaxes de type vEdge

```
vpn 0
 interface ge0/0
  mtu 1500
  tunnel-interface
   encapsulation ipsec
   color biz-internet
  !
 !
!
vpn 20
 interface ge0/1.20
  ip address 10.20.34.1/24
  mtu 1500
  tcp-mss-adjust 1350
 !
!
```

### 10.3 Valeur adaptée aux sites avec 5G à MTU réduit

Si le site possède un lien cellulaire à 1400 :

```
interface GigabitEthernet0/1/0.20
 ip tcp adjust-mss 1290
```

On évite ainsi la fragmentation y compris après bascule sur le lien 5G.

### 10.4 Pare-feu : autoriser l'ICMP utile (exemple générique)

```
permit icmp any any packet-too-big
permit icmp any any unreachable
permit icmp any any echo
permit icmp any any echo-reply
```

À adapter à votre politique (par exemple ne pas autoriser l'écho depuis Internet, mais laisser passer les types 3 et ICMPv6 type 2).

### 10.5 Réglage côté hôte (Linux), à titre de test

```
ip link set dev eth0 mtu 1400
ip route change default via 10.20.34.1 mtu 1400
```

Ce type de réglage est un **outil de test**. En production, on corrige au niveau de l'edge.

---

## 11. Cas pratique : magasin de proximité en ADSL et 4G

### 11.1 Contexte

Le magasin "Proximité Bordeaux Chartrons" (site fictif 3877) n'est pas éligible à la fibre. Liens :

- ADSL en PPPoE (MTU 1492) ;
- 4G de secours (MTU observé 1400).

Edge compact, template standard de proximité, avec un MSS clamping de 1350 hérité du modèle.

### 11.2 Symptômes remontés

- Le lundi, les caisses ont des ralentissements, des transactions échouent par intermittence.
- Les téléphones IP fonctionnent correctement.
- Lors d'une coupure ADSL, la bascule 4G se produit, mais les caisses se figent pendant plusieurs minutes.

### 11.3 Analyse pas à pas

**Étape 1 : mesurer le MTU de chaque transport**

| Transport | Test (ping DF depuis l'edge vers le hub) | Résultat |
|---|---|---|
| ADSL PPPoE | Taille max passante : paquet de 1492 | MTU 1492 |
| 4G | Taille max passante : paquet de 1400 | MTU 1400 |

**Étape 2 : calculer le paquet interne maximal**

| Transport | MTU | Overhead IPsec (65) | Paquet interne maximal |
|---|---|---|---|
| ADSL | 1492 | 65 | 1427 |
| 4G | 1400 | 65 | 1335 |

**Étape 3 : comparer au MSS en place**

Avec un MSS de 1350, le paquet interne est de 1390 octets :

| Transport | Paquet interne généré | Maximum accepté | Résultat |
|---|---|---|---|
| ADSL | 1390 | 1427 | Passe |
| 4G | 1390 | 1335 | **Trop gros** |

**Conclusion.** Tout fonctionne sur l'ADSL, mais dès que le trafic bascule sur la 4G, les paquets TCP "pleins" dépassent la PMTU. Comme certains équipements intermédiaires de l'opérateur mobile ne renvoient pas l'ICMP, la PMTUD échoue : c'est un trou noir. Les caisses se figent.

### 11.4 Correction

1. **Réduire le MSS** du profil "proximité" à **1290** (calcul : 1400 moins 65 moins 40, arrondi à la baisse).
2. Alternative : activer la **découverte de PMTU** de tunnel si la version le permet.
3. Documenter la valeur dans le template (variable `tcp_mss`) et prévoir une valeur plus élevée uniquement pour les sites sans transport mobile.
4. Retest : simulation de coupure ADSL, vérification que les caisses restent fluides pendant la bascule.

### 11.5 Coût de la correction

Un MSS de 1290 au lieu de 1350 représente environ 4 % de données utiles en moins par segment, soit un impact négligeable pour des flux de caisse.

---

## 12. Cas pratique : paiement qui échoue après le handshake TLS

### 12.1 Contexte

Au siège, le service monétique constate que, sur quelques dizaines de magasins, des transactions de paiement échouent de façon aléatoire. Les terminaux se connectent à un serveur de paiement central en TLS.

### 12.2 Observations

- Le TCP s'établit. Le "Client Hello" part. Le "Server Hello" et la chaîne de certificats reviennent en plusieurs segments.
- Dans les captures, le serveur renvoie des segments de 1400 octets, qui disparaissent entre le hub et le magasin.
- Aucun ICMP "fragmentation needed" n'est reçu côté terminal.

### 12.3 Diagnostic

1. Le message "Certificate" du serveur est volumineux (chaîne de plusieurs certificats) et est découpé en segments TCP **pleins**.
2. Sur le chemin retour, le tunnel SD-WAN ajoute 65 octets, et un lien mobile ou ADSL réduit la limite.
3. Le pare-feu en amont du serveur de paiement filtre l'ICMP type 3 code 4.
4. Les gros segments sont donc abandonnés et la session ne dépasse jamais l'étape de certificat.

### 12.4 Correction

| Action | Détail |
|---|---|
| Autoriser l'ICMP utile | Types nécessaires à la PMTUD sur le pare-feu de la zone monétique |
| Ajuster le MSS côté serveur | Clamping au niveau des hubs pour les flux retour |
| Réduire la taille de la chaîne TLS | Si possible, retirer les certificats superflus de la chaîne envoyée |
| Aligner le MSS d'edge | Valeur cohérente avec le pire transport de chaque profil de site |
| Surveillance | Alerte sur le nombre d'échecs de handshake TLS par site |

### 12.5 Leçon

Les problèmes de MTU apparaissent souvent aux moments où un **message volumineux** est échangé (certificats TLS, listes de produits, mises à jour). Le trafic quotidien normal peut sembler sain.

---

## 13. Cas pratique : vidéosurveillance et jumbo frames au data center

### 13.1 Contexte

Un hypermarché envoie des exports vidéo vers un serveur central. Le data center, lui, utilise des **jumbo frames** (MTU 9000) sur certains segments de stockage.

### 13.2 Risque

Si une partie du chemin est en 9000 et une autre en 1500, des paquets de grande taille générés côté data center peuvent être abandonnés (si DF est activé) ou fragmentés à la frontière du WAN, avec une grosse perte d'efficacité.

### 13.3 Règles de conception

1. **Isoler** les segments jumbo : ne les étendre ni dans le WAN ni dans l'overlay SD-WAN.
2. Garantir que les flux provenant des magasins ne rencontrent **jamais** un chemin à MTU supérieur sans mécanisme de réduction.
3. Documenter le MTU de chaque VLAN et de chaque VRF du data center.
4. Vérifier que les hubs SD-WAN ont une MTU cohérente côté LAN data center (1500 pour les flux WAN).
5. Pour les exports vidéo massifs, prévoir un **MSS adapté** et mesurer le taux de retransmission TCP.

### 13.4 Autres cas d'overlay interne

Un overlay de data center de type VXLAN ajoute environ 50 octets. S'il traverse ensuite un tunnel SD-WAN (65 octets), l'overhead se cumule : prévoir une marge ou des MTU plus élevés dans le data center si l'équipement le permet.

---

## 14. Conception : choisir ses valeurs de MTU et de MSS

### 14.1 Démarche

1. **Inventorier les transports** par profil de site (fibre, ADSL, MPLS, 4G, 5G) et leur MTU réel mesuré.
2. **Calculer l'overhead** du tunnel avec les algorithmes de chiffrement retenus, dans le pire cas.
3. **Déterminer la PMTU interne** du transport le plus contraignant par profil.
4. **En déduire le MSS** (PMTU interne moins 40 en IPv4).
5. **Arrondir à la baisse** et prévoir une marge de sécurité de quelques dizaines d'octets.
6. **Valider** par tests de bascule et mesures de performance.

### 14.2 Exemple de valeurs par profil (distributeur fictif)

| Profil de site | Transports | Pire PMTU interne | MSS retenu |
|---|---|---|---|
| Hypermarché | MPLS plus fibre | 1427 environ | 1350 |
| Supermarché | Fibre plus 2e fibre ou MPLS | 1427 environ | 1350 |
| Proximité avec fibre | Fibre plus 5G (MTU 1450) | 1385 | 1320 |
| Proximité sans fibre | ADSL PPPoE plus 4G (MTU 1400) | 1335 | 1290 |
| Entrepôt | Liens dédiés Ethernet | 1435 | 1350 |

### 14.3 Variable de template

Ajouter dans le template une variable `tcp_mss` dont la valeur est dérivée du profil de site, plutôt qu'une valeur unique pour tout le parc.

### 14.4 MTU de l'interface de service

Laisser **1500** côté LAN du magasin pour ne pas imposer de réglage aux postes, caisses et terminaux, et compter sur le MSS clamping et la découverte de PMTU pour la compatibilité. Réduire le MTU des hôtes est possible mais lourd à gérer à grande échelle.

### 14.5 Sortie Internet locale (DIA)

Le trafic qui sort directement vers Internet **n'est pas encapsulé** dans l'overlay : il n'a donc pas besoin de l'overhead IPsec. Il faut néanmoins respecter le MTU du transport (par exemple 1492 en PPPoE). Le MSS clamping s'applique selon la configuration de l'interface de service et peut pénaliser légèrement ce trafic : décider s'il faut un MSS différent pour les flux DIA.

---

## 15. Plan de test et de validation

### 15.1 Tests avant déploiement

| Test | Objectif | Résultat attendu |
|---|---|---|
| Mesure du MTU de chaque transport | Connaître la vraie valeur | Valeurs documentées |
| Ping DF de taille décroissante via l'overlay | Vérifier le PMTU interne | Conforme au calcul |
| Capture des SYN avant et après l'edge | Vérifier le clamping MSS | MSS conforme |
| Téléchargement de gros fichier (HTTP, SMB) | Vérifier absence de fragmentation | Débit nominal, pas de retransmissions anormales |
| Test TLS avec chaîne de certificats longue | Vérifier le handshake | Réussite |
| Bascule forcée vers le lien le plus contraignant | Vérifier la continuité | Aucun gel de session |
| Retour sur le lien principal | Vérifier la reprise | Session stable |

### 15.2 Tests en production (par échantillonnage)

- Surveiller le taux de retransmissions TCP par site.
- Surveiller les compteurs de fragmentation et d'abandons pour cause de MTU.
- Alerter sur des échecs de handshake TLS anormaux.

### 15.3 Gestion des changements

- Tout changement de transport ou d'opérateur déclenche un **re-test de MTU**.
- Tout changement d'algorithme de chiffrement (donc d'overhead) déclenche un recalcul.
- Les nouvelles valeurs sont déployées par vagues, avec retour arrière possible.

---

## 16. Bonnes pratiques et pièges

### Bonnes pratiques

1. **Mesurer**, ne pas supposer : le MTU d'un lien mobile ou d'un opérateur n'est jamais garanti.
2. **Calculer** l'overhead dans le pire cas, avec le padding maximal.
3. **Caler le MSS sur le pire transport** du site, y compris le lien de secours.
4. **Combiner** MSS clamping et MTU de tunnel cohérent.
5. **Laisser passer l'ICMP** nécessaire à la PMTUD sur tous les pare-feux.
6. **Documenter** MTU et MSS par profil dans les templates.
7. **Tester les bascules**, pas seulement le fonctionnement nominal.
8. **Surveiller** les indicateurs de fragmentation et de retransmission.
9. **Isoler** les jumbo frames du reste du réseau.
10. **Réévaluer** après tout changement de transport, d'opérateur ou de chiffrement.

### Pièges

- Croire que "le ping marche" prouve que le MTU est correct.
- Appliquer une valeur unique de MSS à tout le parc sans tenir compte du lien de secours.
- Oublier l'overhead du NAT-T ou du padding.
- Filtrer tout l'ICMP "par sécurité".
- Ignorer UDP : le MSS clamping ne protège pas la voix, la vidéo ni QUIC.
- Empiler les tunnels (SD-WAN plus VPN client plus SASE) sans recalculer le MTU.
- Modifier le MTU des interfaces sans coordonner les deux extrémités d'un lien.
- Confondre MTU (couche 3) et taille de trame Ethernet.

---

## 17. Exercices et corrigés

### Exercice 1 : calcul de MSS

**Énoncé.** Quel est le MSS IPv4 pour un MTU de 1492 ? Et le MSS IPv6 ?

**Corrigé.** IPv4 : 1492 moins 40 égale 1452. IPv6 : 1492 moins 60 égale 1432.

### Exercice 2 : overhead IPsec

**Énoncé.** Un tunnel IPsec ESP en mode tunnel avec NAT-T utilise AES-GCM (IV de 8 octets, ICV de 16 octets). Calculez l'overhead minimal (sans padding) et la PMTU interne sur un lien de MTU 1500.

**Corrigé.** 20 (IP externe) plus 8 (UDP) plus 8 (ESP) plus 8 (IV) plus 2 (fin ESP) plus 16 (ICV) égale 62 octets. PMTU interne : 1500 moins 62 égale 1438 octets (avant padding éventuel).

### Exercice 3 : taille du ping

**Énoncé.** Vous voulez tester un paquet IP de 1400 octets avec le bit DF. Quelle valeur de charge ICMP utiliser sous Linux ?

**Corrigé.** 1400 moins 28 égale 1372. Commande : `ping -M do -s 1372 <adresse>`.

### Exercice 4 : choix du MSS

**Énoncé.** Un magasin possède une fibre (MTU 1500) et une 4G (MTU 1380). L'overhead IPsec maximal est de 65 octets. Quel MSS choisir pour que la bascule soit sans problème ?

**Corrigé.** Pire transport : 1380. PMTU interne : 1380 moins 65 égale 1315. MSS : 1315 moins 40 égale 1275. On retient 1270 ou 1275 avec marge.

### Exercice 5 : diagnostic

**Énoncé.** Après un changement d'opérateur, un site voit ses caisses se figer sur les gros échanges, alors que le ping et le DNS fonctionnent. Proposez quatre vérifications.

**Corrigé.**

1. Mesurer le MTU du nouveau lien avec un ping DF par dichotomie.
2. Vérifier le MSS dans les SYN avant et après l'edge.
3. Vérifier si l'ICMP "fragmentation needed" est filtré quelque part.
4. Comparer le MTU effectif du tunnel avec la valeur configurée dans le template.

### Exercice 5 bis : lien PPPoE

**Énoncé.** Un site passe de fibre Ethernet à de l'ADSL en PPPoE. Quelle conséquence sur la PMTU interne si l'overhead IPsec est de 65 ?

**Corrigé.** Fibre : 1500 moins 65 égale 1435. ADSL : 1492 moins 65 égale 1427. La PMTU interne diminue de 8 octets. Un MSS de 1350 reste valide dans ce cas.

### Exercice 6 : conception

**Énoncé.** Définissez un MSS par profil pour : (a) hypermarché avec MPLS à 1500 et fibre à 1500 ; (b) proximité avec ADSL PPPoE et 4G à 1420.

**Corrigé.**

(a) PMTU interne : 1500 moins 65 égale 1435, MSS : 1395. On retient 1350 avec marge, ou 1380 selon la politique.

(b) Pire transport : 1420. PMTU interne : 1420 moins 65 égale 1355, MSS : 1315. On retient 1300 avec marge.

---

## 18. Quiz de révision

1. Que représente le MTU et à quelle couche appartient-il ?
2. Quelle est la différence entre MTU et MSS ?
3. Que fait un routeur IPv4 qui reçoit un paquet trop gros avec DF à 1 ?
4. Pourquoi les routeurs IPv6 ne fragmentent-ils pas ?
5. Qu'est-ce qu'un trou noir PMTUD ?
6. Quel est l'overhead approximatif d'un tunnel IPsec ESP avec NAT-T et AES-GCM ?
7. Pourquoi le MSS clamping ne protège-t-il pas la voix ?
8. Pourquoi un MSS de 1350 peut-il poser problème sur un lien 4G à 1400 ?
9. Comment tester le MTU d'un chemin avec un ping ?
10. Pourquoi tester les bascules de transport en plus du fonctionnement nominal ?

**Éléments de réponse**

1. Taille maximale d'un paquet de couche 3, en-tête IP inclus.
2. Le MTU concerne le paquet IP entier, le MSS uniquement la charge utile TCP.
3. Il l'abandonne et renvoie un ICMP "fragmentation needed" (type 3 code 4).
4. Le protocole le prévoit ainsi : seul l'émetteur fragmente, les routeurs renvoient un ICMPv6 "Packet Too Big".
5. Situation où l'ICMP nécessaire à la PMTUD est filtré, ce qui fait abandonner silencieusement les gros paquets.
6. Environ 62 à 65 octets.
7. Parce que le MSS est une option TCP, alors que la voix utilise UDP.
8. 1350 plus 40 plus 65 donne 1455, supérieur à 1400 : le paquet encapsulé ne passe pas.
9. Avec l'option DF et une taille de charge variable (taille IP moins 28), en recherchant la plus grande taille qui passe.
10. Parce que le lien de secours a souvent un MTU plus contraignant, et que les problèmes n'apparaissent qu'à la bascule.

---

## 19. Glossaire

| Terme | Définition |
|---|---|
| DF (Don't Fragment) | Bit de l'en-tête IPv4 interdisant la fragmentation du paquet |
| ESP | Encapsulating Security Payload, protocole de chiffrement d'IPsec |
| Fragmentation | Découpage d'un paquet IP en plusieurs morceaux plus petits |
| ICMP type 3 code 4 | Message IPv4 "Fragmentation needed and DF set" |
| ICMPv6 type 2 | Message "Packet Too Big" d'IPv6 |
| ICV | Integrity Check Value, valeur de contrôle d'intégrité ajoutée par IPsec |
| Jumbo frame | Trame Ethernet dont le MTU dépasse 1500, typiquement 9000 |
| MRU | Maximum Receive Unit, taille maximale acceptée en réception |
| MSS | Maximum Segment Size, taille maximale de la charge utile TCP |
| MSS clamping | Réécriture de l'option MSS dans les SYN pour limiter la taille des segments |
| MTU | Maximum Transmission Unit, taille maximale d'un paquet de couche 3 sur un lien |
| NAT-T | NAT Traversal, encapsulation UDP des paquets IPsec pour traverser les NAT |
| Overhead | Octets ajoutés par une encapsulation |
| PLPMTUD | Packetization Layer PMTUD, découverte de PMTU par la couche transport sans ICMP |
| PMTU | Path MTU, plus petit MTU rencontré sur un chemin |
| PMTUD | Path MTU Discovery, mécanisme de découverte de la PMTU |
| PPPoE | Point-to-Point Protocol over Ethernet, encapsulation courante des accès ADSL |
| VXLAN | Encapsulation de réseau virtuel (environ 50 octets d'overhead) |

---

## Pour aller plus loin

- RFC 791 (IPv4), RFC 1191 (PMTUD IPv4), RFC 8201 (PMTUD IPv6), RFC 4821 (PLPMTUD).
- Documentation de votre éditeur SD-WAN sur le MTU de tunnel, la découverte de PMTU et le MSS clamping.
- Outils : ping, tracepath, Wireshark, iperf3 (avec options de taille de segment).

---

*Fin du cours.*
