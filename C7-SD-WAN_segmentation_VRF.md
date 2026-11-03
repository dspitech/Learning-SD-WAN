# Cours complet SD-WAN : segmentation VRF

Ce cours fait suite aux cours sur OMP, BFD, SLA, blackout et brownout. Il traite la segmentation : comment un même réseau SD-WAN, avec les mêmes liens et les mêmes tunnels, transporte plusieurs réseaux logiques totalement isolés les uns des autres. C'est l'une des raisons majeures d'adopter le SD-WAN dans une enseigne de distribution, où les flux de paiement, les caméras, les objets connectés, le Wi-Fi invités et les partenaires doivent cohabiter sans jamais se mélanger. Le fil rouge reste le réseau d'une enseigne de grande distribution de type Carrefour.

> Note importante : le cas Carrefour est un scénario pédagogique fictif. Les plans d'adressage, les numéros de VPN, les identifiants de sites, les volumes et les exigences de conformité sont inventés ou simplifiés à des fins d'illustration et ne décrivent ni l'architecture réelle de l'enseigne ni ses exigences réelles.

> Note de vocabulaire : Cisco a rebaptisé sa solution Catalyst SD-WAN (vManage devient SD-WAN Manager, vSmart devient SD-WAN Controller, vBond devient SD-WAN Validator). Ce cours conserve les noms historiques, plus répandus.

---

## Table des matières

1. Objectifs et prérequis
2. Pourquoi segmenter
3. Vocabulaire : VRF, VPN, segment
4. Les types de VPN dans Cisco SD-WAN
5. Comment fonctionne la segmentation de bout en bout
6. Segmentation et OMP
7. Contrôler la topologie VPN par VPN
8. VPN membership policy
9. Communication entre segments : route leaking et services partagés
10. Sécurité entre segments : pare-feu et service chaining
11. Accès internet par segment (DIA)
12. Segmentation côté LAN : VLAN, VRF et Wi-Fi
13. Adressage et chevauchement de plans IP
14. Segmentation et conformité PCI DSS
15. Cas d'étude fil rouge : le réseau d'une enseigne de distribution
16. Configuration pas à pas
17. Vérification et dépannage
18. Passage à l'échelle
19. Pièges classiques
20. Bonnes pratiques
21. Travaux pratiques
22. Questionnaire d'auto-évaluation
23. Glossaire
24. Références

---

## 1. Objectifs et prérequis

### Objectifs pédagogiques

À la fin de ce cours, vous serez capable de :

- Expliquer pourquoi et comment le SD-WAN segmente un réseau d'entreprise.
- Distinguer VPN 0, VPN 512 et VPN de service, et la notion de VRF sur les routeurs IOS XE.
- Décrire le trajet d'un paquet d'un segment à l'autre site, avec le rôle du label dans OMP.
- Définir une topologie différente par segment avec des politiques de contrôle.
- Limiter les VPN présents sur chaque site avec une VPN membership policy.
- Autoriser des échanges contrôlés entre segments (route leaking, services partagés, pare-feu).
- Organiser la sortie internet locale par segment et l'interconnexion avec le datacenter.
- Concevoir un plan de segmentation cohérent, documenté et vérifiable.

### Prérequis

- Le cours sur OMP (TLOC, routes, politiques de contrôle).
- Notions de VLAN, de VRF et de routage (OSPF, BGP).
- Notions de pare-feu à états (zones, politiques).

---

## 2. Pourquoi segmenter

### 2.1 Besoins typiques d'une enseigne de distribution

| Besoin | Exemple |
|---|---|
| Protéger les données de paiement | Isoler les caisses et les terminaux de paiement du reste du réseau |
| Limiter la propagation d'une attaque | Un objet connecté compromis ne doit pas atteindre les caisses |
| Séparer les populations | Collaborateurs, invités, partenaires, prestataires |
| Séparer les usages | Vidéosurveillance, IoT, bureautique, supervision |
| Appliquer des politiques différentes | Qualité de service, chemins, sécurité propres à chaque segment |
| Réduire le périmètre d'audit | Cibler les contrôles de conformité sur un nombre limité de segments |

### 2.2 Segmentation et surface d'attaque

Sans segmentation, tous les équipements d'un magasin partagent le même réseau. Une imprimante, une balance connectée ou un point d'accès Wi-Fi compromis peut alors atteindre des systèmes sensibles. La segmentation limite le rayon d'impact (blast radius) d'un incident.

### 2.3 Segmentation et SD-WAN

Dans un WAN classique, la segmentation demandait de bâtir des VPN MPLS distincts par opérateur ou de superposer des tunnels, avec une configuration lourde. Dans le SD-WAN, la segmentation est native : les VPN sont créés une fois, sont portés par les mêmes tunnels chiffrés et sont pilotés de façon centralisée.

---

## 3. Vocabulaire : VRF, VPN, segment

| Terme | Signification |
|---|---|
| VRF | Virtual Routing and Forwarding : instance de table de routage isolée sur un routeur |
| VPN (Viptela OS, vEdge) | Équivalent d'un VRF, identifié par un numéro |
| VPN (overlay SD-WAN) | Segment logique de bout en bout, identifié par un numéro commun à tous les sites |
| Service VPN | VPN qui porte le trafic des utilisateurs et des services |
| Segment | Mot générique pour désigner un VPN de service dans la conception |

Sur un cEdge (Cisco IOS XE), un VPN est matérialisé par une VRF dont le nom est le numéro du VPN. Sur un vEdge, on parle directement de `vpn <numéro>`. Dans les deux cas, le numéro est le même sur tous les sites : le VPN 10 d'un magasin et le VPN 10 d'un datacenter appartiennent au même segment logique.

### 3.1 Différence avec un VPN MPLS classique

| | MPLS L3VPN | VPN SD-WAN |
|---|---|---|
| Identification d'un segment | Route Distinguisher et Route Target | Numéro de VPN |
| Protocole d'échange | MP-BGP | OMP |
| Matérialisation sur le chemin | Labels MPLS | Label de VPN dans l'encapsulation IPsec |
| Chiffrement | Non par défaut | IPsec par défaut |
| Pilotage | Configuration opérateur | Politique centralisée par le client |

Remarque : sur les routeurs IOS XE, la configuration des VRF contient des RD et des Route Targets (par exemple `1:10`). Ils servent notamment aux fonctions de fuite de routes locales, mais le transport de l'overlay repose sur le numéro de VPN et OMP.

---

## 4. Les types de VPN dans Cisco SD-WAN

### 4.1 VPN 0 : transport

- VPN réservé, présent sur tous les équipements.
- Contient les interfaces WAN (les couleurs), les routes de l'underlay et les connexions de contrôle.
- Aucune route d'un VPN de service n'y est visible, et réciproquement, sauf fonctions explicites (DIA, fuites de routes).

### 4.2 VPN 512 : gestion hors bande

- VPN réservé à l'administration (interface de management).
- Ne participe pas à l'overlay. À protéger strictement.

### 4.3 VPN de service

- Numéros utilisables : de 1 à une valeur maximale élevée (de l'ordre de 65 530), hors valeurs réservées (0 et 512). Vérifier les bornes dans la documentation de votre version.
- Contiennent les interfaces LAN, les protocoles de routage du site (statique, OSPF, BGP, EIGRP) et les routes apprises par OMP.
- Chaque interface appartient à un seul VPN.

### 4.4 Schéma d'un WAN Edge

```
                 WAN Edge (magasin 2750)

   LAN caisses  ----  VPN 10  ---+
   LAN backoffice -- VPN 20  ---+
   LAN caméras  ----  VPN 30  ---+---- OMP / label ---- IPsec ---- WAN
   LAN invités  ----  VPN 40  ---+     (un tunnel par couleur,
   LAN IoT      ----  VPN 50  ---+      tous VPN confondus)

   Interfaces WAN  ----  VPN 0  (transport, couleurs, contrôle)
   Gestion         ----  VPN 512
```

### 4.5 Règle fondamentale d'isolation

Deux VPN d'un même routeur ne partagent ni table de routage ni table ARP. Sans fuite de routes explicite, un équipement du VPN 40 ne peut pas joindre un équipement du VPN 10, même s'ils sont branchés sur le même routeur.

---

## 5. Comment fonctionne la segmentation de bout en bout

### 5.1 Chaîne complète pour une caisse (VPN 10)

1. L'interface LAN des caisses appartient au VPN 10. Le routeur apprend le préfixe local (connecté ou protocole de routage) dans la table du VPN 10.
2. Le WAN Edge annonce ce préfixe par OMP, avec l'attribut VPN 10, un label et ses TLOC.
3. Le vSmart distribue la route aux sites autorisés pour le VPN 10, selon les politiques.
4. Le WAN Edge distant installe la route dans la table de son VPN 10, avec pour next hop un TLOC.
5. À l'émission, un paquet du VPN 10 est encapsulé dans IPsec avec le label du VPN 10 et envoyé vers le TLOC.
6. À la réception, le WAN Edge lit le label, identifie le VPN 10 et injecte le paquet dans la bonne table de routage.

### 5.2 Un tunnel pour tous les segments

Les tunnels IPsec sont établis entre TLOC, pas entre VPN. Un même tunnel porte donc le trafic de tous les VPN, différenciés par le label. Conséquences :

- Le nombre de tunnels et de sessions BFD ne croît pas avec le nombre de VPN.
- Le chiffrement est propre au tunnel (dans le modèle standard, il n'y a pas une clé par VPN) : l'isolation entre VPN est logique, assurée par le label et les tables de routage.
- Les politiques de qualité de service et de routage applicatif peuvent en revanche être définies VPN par VPN.

### 5.3 Ce que la segmentation ne fait pas

- Elle n'inspecte pas le contenu du trafic : un pare-feu reste nécessaire pour filtrer les échanges autorisés entre segments.
- Elle ne protège pas un équipement contre une attaque venant de son propre segment.
- Elle ne remplace pas la segmentation du LAN (VLAN, contrôle d'accès, authentification des équipements).

---

## 6. Segmentation et OMP

### 6.1 Ce que porte OMP

Les routes OMP contiennent notamment :

- Le VPN d'appartenance du préfixe.
- Le label associé.
- Le TLOC next hop, la préférence, le tag, le site-id.

C'est le VPN, avec le label, qui oriente chaque route dans la table du bon VPN sur le WAN Edge récepteur.

### 6.2 Redistribution par VPN

Le routage du site est propre à chaque VPN. Sur un cEdge, on peut activer les protocoles et les redistributions par VRF :

- Un processus OSPF ou une session BGP par VRF.
- L'injection dans OMP (`advertise connected`, `advertise ospf`, `advertise bgp`, etc.), soit globalement, soit par VRF selon la plateforme et la version.
- La redistribution des routes OMP vers OSPF ou BGP, par VRF.

### 6.3 Routes en double dans plusieurs VPN

Un même préfixe peut exister dans deux VPN différents sans conflit : OMP les distingue par le VPN. Cette propriété permet le chevauchement d'adressage (section 13).

### 6.4 Lecture des routes par VPN

```
show sdwan omp routes vpn 10
show sdwan omp routes vpn 30
```

Chaque commande ne montre que les routes du VPN demandé.

---

## 7. Contrôler la topologie VPN par VPN

Par défaut, tous les sites d'un VPN reçoivent toutes les routes de ce VPN : topologie en full mesh. Les politiques de contrôle permettent de définir une topologie par VPN, avec une `vpn-list` dans les critères de correspondance.

### 7.1 Exemple : topologies différentes par segment

Objectifs pour l'enseigne :

- VPN 10 (caisses) : hub and spoke strict vers les datacenters.
- VPN 20 (back office) : hub and spoke vers les datacenters, avec accès direct possible entre entrepôts.
- VPN 30 (vidéosurveillance) : hub and spoke vers le datacenter qui héberge l'enregistrement.

```
policy
 lists
  site-list DATACENTERS
   site-id 1000-1001
  site-list ENTREPOTS
   site-id 3000-3099
  site-list MAGASINS
   site-id 2000-2999
  site-list DC-ET-ENTREPOTS
   site-id 1000-1001
   site-id 3000-3099
  vpn-list VPN-CAISSES
   vpn 10
  vpn-list VPN-BACKOFFICE
   vpn 20
  vpn-list VPN-CAMERAS
   vpn 30
 !
 control-policy TOPOLOGIE-PAR-VPN
  sequence 1
   match tloc
    site-list DC-ET-ENTREPOTS
   !
   action accept
  !
  sequence 10
   match route
    vpn-list VPN-CAISSES
    site-list DATACENTERS
   !
   action accept
  !
  sequence 20
   match route
    vpn-list VPN-BACKOFFICE
    site-list DC-ET-ENTREPOTS
   !
   action accept
  !
  sequence 30
   match route
    vpn-list VPN-CAMERAS
    site-list DATACENTERS
   !
   action accept
  !
  default-action reject
 !
!
apply-policy
 site-list MAGASINS
  control-policy TOPOLOGIE-PAR-VPN out
 !
!
```

La syntaxe exacte dépend de la version : en pratique, on construit ces politiques dans l'interface de vManage.

### 7.2 Remarques

- La séquence portant sur les TLOC conditionne la possibilité de monter des tunnels vers les sites listés.
- Une route non acceptée par la politique n'est pas envoyée au site : il n'a donc aucun chemin vers elle.
- Les datacenters doivent annoncer un agrégat ou une route par défaut dans chaque VPN pour que les magasins puissent joindre les autres sites via le hub.

---

## 8. VPN membership policy

### 8.1 Principe

La VPN membership policy est une politique centralisée appliquée sur vSmart qui définit, pour chaque groupe de sites, la liste des VPN auxquels ils ont le droit de participer. Un site ne reçoit alors que les routes des VPN autorisés.

### 8.2 Intérêt

| Intérêt | Explication |
|---|---|
| Sécurité | Un site qui n'a pas à voir un segment n'en reçoit aucune route |
| Allègement | Moins de routes dans les tables des petits sites |
| Gouvernance | Une règle centrale lisible : quels sites pour quels segments |
| Défense en profondeur | Une erreur de configuration locale (une interface placée dans le mauvais VPN) ne suffit pas à exposer un segment |

### 8.3 Exemple

```
policy
 lists
  site-list MAGASINS-PROXIMITE
   site-id 2600-2999
  site-list MAGASINS-HYPER-SUPER
   site-id 2000-2599
  vpn-list VPN-PROXIMITE
   vpn 10
   vpn 20
   vpn 30
   vpn 50
   vpn 90
  vpn-list VPN-GRANDS-MAGASINS
   vpn 10
   vpn 20
   vpn 30
   vpn 50
   vpn 60
   vpn 90
 !
 vpn-membership VPN-MEMBERSHIP-PROXIMITE
  sequence 10
   match
    vpn-list VPN-PROXIMITE
   !
   action accept
  !
  default-action reject
 !
 vpn-membership VPN-MEMBERSHIP-GRANDS-MAGASINS
  sequence 10
   match
    vpn-list VPN-GRANDS-MAGASINS
   !
   action accept
  !
  default-action reject
 !
!
apply-policy
 site-list MAGASINS-PROXIMITE
  vpn-membership VPN-MEMBERSHIP-PROXIMITE
 !
 site-list MAGASINS-HYPER-SUPER
  vpn-membership VPN-MEMBERSHIP-GRANDS-MAGASINS
 !
!
```

Ici, les magasins de proximité n'ont pas de VPN 60 (partenaires, par exemple les concessionnaires d'une galerie marchande). Les datacenters, eux, doivent participer à tous les VPN.

### 8.4 Attention

Les VPN uniquement locaux (par exemple un VPN 40 invités qui sort directement sur internet et n'est jamais annoncé dans OMP) n'ont pas besoin de figurer dans la politique de membership.

---

## 9. Communication entre segments : route leaking et services partagés

### 9.1 Quand autoriser des échanges entre segments

Certains services doivent être accessibles depuis plusieurs segments : DNS, NTP, annuaire, supervision, mises à jour. Les placer dans un VPN de services partagés et laisser fuiter certaines routes vers les VPN consommateurs est plus sûr que de les dupliquer dans chaque VPN.

### 9.2 Deux mécanismes

| Mécanisme | Où il agit | Usage |
|---|---|---|
| Fuite de routes locale (par VRF, avec Route Targets, ou via le modèle de configuration de vManage) | Sur un WAN Edge | Échanges entre deux VPN d'un même routeur |
| Fuite de routes pilotée par politique de contrôle (export vers une liste de VPN, route de statut extranet) | Sur vSmart, pour tout le réseau | Services partagés accessibles de plusieurs sites |

La disponibilité et la syntaxe de ces fonctions varient selon les versions. Les exemples ci-dessous sont indicatifs.

### 9.3 Exemple de fuite locale (cEdge, indicatif)

```
vrf definition 90
 rd 1:90
 address-family ipv4
  route-target export 1:90
  route-target import 1:10
  route-target import 1:20
 exit-address-family
!
vrf definition 10
 rd 1:10
 address-family ipv4
  route-target export 1:10
  route-target import 1:90
 exit-address-family
!
```

### 9.4 Exemple de fuite pilotée par politique de contrôle (indicatif)

```
policy
 lists
  vpn-list VPN-SERVICES-PARTAGES
   vpn 90
  vpn-list VPN-CONSOMMATEURS
   vpn 10
   vpn 20
  prefix-list DNS-NTP-SUPERVISION
   ip-prefix 10.90.0.0/24
 !
 control-policy FUITE-SERVICES-PARTAGES
  sequence 10
   match route
    vpn-list VPN-SERVICES-PARTAGES
    prefix-list DNS-NTP-SUPERVISION
   !
   action accept
    export-to
     vpn-list VPN-CONSOMMATEURS
    !
   !
  !
  default-action accept
 !
!
```

### 9.5 Règles de prudence

1. Ne faire fuiter que les préfixes nécessaires, jamais un VPN entier.
2. Préférer une fuite unidirectionnelle contrôlée par un pare-feu, plutôt qu'une fuite bidirectionnelle ouverte.
3. Tenir un registre des fuites : chaque fuite est une exception à l'isolation.
4. Surveiller les chevauchements d'adressage : une fuite entre deux VPN qui contiennent le même préfixe est ambiguë.
5. Éviter les fuites en cascade (VPN A vers B, puis B vers C) qui créent des chemins indirects imprévus.

---

## 10. Sécurité entre segments : pare-feu et service chaining

### 10.1 Pare-feu intégré (zones)

Les cEdge peuvent embarquer un pare-feu à états par zones (ZBFW). Principe :

- Une zone regroupe un ou plusieurs VPN.
- Une paire de zones (source, destination) porte une politique : autoriser, inspecter, rejeter.
- Le trafic entre zones est filtré selon ces règles. Le trafic intra-zone est autorisé par défaut.

Exemple de matrice de zones pour l'enseigne (fictive) :

| Source | Destination | Politique |
|---|---|---|
| Caisses (VPN 10) | Services partagés (VPN 90) | Autoriser DNS et NTP uniquement |
| Caisses (VPN 10) | Internet | Rejeter (le paiement passe par le datacenter) |
| Invités (VPN 40) | Tous les VPN internes | Rejeter |
| Invités (VPN 40) | Internet | Autoriser et inspecter |
| IoT (VPN 50) | Services partagés (VPN 90) | Autoriser les mises à jour et la supervision selon liste |
| Partenaires (VPN 60) | Services partagés (VPN 90) | Autoriser DNS uniquement |
| Back office (VPN 20) | Internet | Autoriser avec filtrage |

### 10.2 Service chaining

Pour des sites trop petits pour embarquer des fonctions de sécurité avancées, ou pour des besoins d'inspection centralisée, le trafic d'un VPN peut être acheminé vers un pare-feu central (service route de type FW annoncée par le hub, voir le cours sur OMP, section 10.4).

### 10.3 Fonctions de sécurité complémentaires

Selon la licence et la plateforme : prévention d'intrusion (IPS), filtrage d'URL, protection contre les logiciels malveillants, sécurité DNS, tunnels vers une passerelle internet sécurisée (SIG). Ces fonctions s'appliquent aux VPN concernés (par exemple VPN 40 et VPN 20) et non à tous les segments.

### 10.4 Répartition des rôles

| Mécanisme | Rôle |
|---|---|
| Segmentation (VPN) | Isolation par défaut |
| Membership et politiques de contrôle | Limiter qui voit quoi |
| Fuites de routes | Exceptions explicites |
| Pare-feu | Filtrer les exceptions |
| Chiffrement IPsec | Protéger les données en transit |

---

## 11. Accès internet par segment (DIA)

### 11.1 Principe

Le Direct Internet Access fait sortir un VPN directement sur internet par un lien WAN local, sans passer par le datacenter. Il est configuré VPN par VPN : on décide quels segments sortent localement.

### 11.2 Choix de l'enseigne (fictif)

| VPN | Accès internet | Justification |
|---|---|---|
| 10 caisses | Aucun | Flux de paiement vers le datacenter uniquement |
| 20 back office | Via le datacenter ou une passerelle sécurisée | Contrôle et journalisation centralisés |
| 30 vidéosurveillance | Aucun | Flux interne |
| 40 invités | Sortie locale (DIA) | Éviter de surcharger le WAN et de mélanger le trafic public avec le trafic interne |
| 50 IoT | Via le datacenter, liste de destinations limitée | Surface d'attaque réduite |
| 60 partenaires | Sortie locale ou propre à chaque concessionnaire | Selon contrat |

### 11.3 Configuration indicative (vEdge, la logique est la même sur cEdge)

```
vpn 0
 interface ge0/0
  ip address 203.0.113.10/30
  nat
   respond-to-ping
  !
  tunnel-interface
   encapsulation ipsec
   color biz-internet
  !
  no shutdown
 !
!
vpn 40
 interface ge0/4
  ip address 192.168.40.1/24
  no shutdown
 !
 ip route 0.0.0.0/0 vpn 0
!
```

Deux éléments : le NAT sur l'interface de transport du VPN 0, et la route par défaut du VPN 40 qui pointe vers le VPN 0. Cette route est l'une des rares fuites d'un VPN de service vers le VPN 0 et doit être limitée aux segments voulus.

### 11.4 DIA sélectif

Une politique de données peut envoyer vers l'internet local seulement certaines applications d'un VPN, le reste empruntant l'overlay. Cela relève de la politique de données et sera étudié séparément.

---

## 12. Segmentation côté LAN : VLAN, VRF et Wi-Fi

### 12.1 Chaîne de segmentation locale

```
Terminal ou point d'accès  ->  port ou SSID  ->  VLAN  ->  sous-interface du WAN Edge  ->  VRF (VPN)
```

(Cette chaîne se lit de gauche à droite.)

### 12.2 Bonnes pratiques

- Un VLAN par segment, jamais un VLAN partagé entre deux segments.
- Liaison de trunk 802.1Q entre le commutateur et le WAN Edge, avec une sous-interface par VLAN, rattachée au VPN correspondant.
- Pour le Wi-Fi : un SSID par segment (collaborateurs, invités, IoT), chacun associé à un VLAN distinct.
- Authentification des équipements (par exemple 802.1X ou affectation dynamique de VLAN) pour éviter qu'un terminal soit branché dans le mauvais segment.
- Désactiver les ports inutilisés, limiter les adresses MAC et activer les protections des commutateurs (DHCP snooping, inspection ARP dynamique).
- Option : étiquetage de groupes de sécurité (SGT) pour propager une identité de groupe dans l'overlay, selon plateforme et version.

### 12.3 Exemple de plan VLAN par magasin (fictif)

| VLAN | Usage | VPN |
|---|---|---|
| 110 | Caisses et terminaux de paiement | 10 |
| 120 | Back office et postes de gestion | 20 |
| 130 | Caméras et enregistreur | 30 |
| 140 | Wi-Fi invités | 40 |
| 150 | IoT (balances, étiquettes électroniques, capteurs de température) | 50 |
| 160 | Concessionnaires de la galerie marchande | 60 |
| 190 | Équipements de supervision locaux | 90 |

---

## 13. Adressage et chevauchement de plans IP

### 13.1 Chevauchement autorisé entre VPN

Comme chaque VPN a sa table de routage, deux VPN peuvent utiliser le même préfixe. Cela est utile pour :

- Les VPN purement locaux (invités, partenaires) qui utilisent le même plan d'adressage dans tous les magasins.
- L'intégration de partenaires dont les plans d'adressage ne sont pas maîtrisables.
- Les rachats d'enseignes ou de réseaux qui se recouvrent.

### 13.2 Exemple (fictif)

| VPN | Préfixe de magasin | Annoncé dans OMP |
|---|---|---|
| 10 caisses | 10.20.n.0/24 (n dérivé du site-id) | Oui |
| 20 back office | 10.21.n.0/24 | Oui |
| 30 caméras | 10.22.n.0/24 | Oui |
| 40 invités | 192.168.40.0/24 (identique partout) | Non, sortie locale |
| 50 IoT | 10.23.n.0/24 | Oui |
| 60 partenaires | 172.16.60.0/24 (identique partout) | Non ou avec NAT |
| 90 services partagés | 10.90.0.0/24 (datacenters) | Oui |

### 13.3 Limites

- Le chevauchement n'est pas possible à l'intérieur d'un même VPN qui doit être routé de bout en bout (sauf NAT).
- Les fuites de routes entre VPN qui contiennent le même préfixe sont à proscrire.
- Pour joindre un réseau qui chevauche, le NAT (au niveau du site ou du datacenter) est nécessaire.

### 13.4 Plan d'adressage dérivé du site-id

Dériver le préfixe du magasin de son site-id (par exemple le troisième octet) simplifie l'automatisation, la lecture des tables de routage et le dépannage. Les modèles de configuration de vManage utilisent des variables pour cela.

---

## 14. Segmentation et conformité PCI DSS

> Cette section est une vue pédagogique générale. Les exigences de conformité doivent être validées avec le responsable de la conformité et un évaluateur qualifié (QSA).

### 14.1 Rôle de la segmentation

La norme PCI DSS vise la protection des données de cartes de paiement. Isoler les systèmes qui traitent ces données (l'environnement de données de titulaires de cartes, CDE) du reste du réseau permet de réduire le périmètre à auditer. Dans le scénario de l'enseigne, le VPN 10 constitue le segment de paiement.

### 14.2 Ce que le SD-WAN apporte

| Exigence de principe | Apport du SD-WAN |
|---|---|
| Isoler le périmètre de paiement | VPN dédié, membership limitée, fuites minimales |
| Contrôler les flux entrant et sortant | Pare-feu par zones, politiques de contrôle |
| Chiffrer les transmissions sur les réseaux ouverts | IPsec sur tous les transports |
| Journaliser et superviser | Journaux, alarmes, tableaux de bord centralisés |
| Maîtriser la configuration | Modèles centralisés, historique des modifications |

### 14.3 Précautions

- La segmentation ne vaut que si elle est validée par des tests : tests d'intrusion et vérifications périodiques de l'isolation (la norme prévoit des tests réguliers des contrôles de segmentation).
- Les terminaux, les commutateurs, le Wi-Fi et les serveurs font partie du périmètre : la segmentation du WAN ne suffit pas.
- Les fuites de routes et les exceptions de pare-feu élargissent le périmètre : elles doivent être documentées et justifiées.
- La gestion hors bande (VPN 512) et les comptes d'administration de vManage sont eux-mêmes sensibles.

---

## 15. Cas d'étude fil rouge : le réseau d'une enseigne de distribution

> Rappel : scénario fictif à vocation pédagogique.

### 15.1 Rappel du contexte

- Deux datacenters (Paris, site-id 1000 et Lyon, site-id 1001).
- Environ 1000 magasins : hypermarchés (MPLS + 2 fibres), supermarchés (MPLS + fibre), proximité (fibre + 4G ou 5G).
- Entrepôts logistiques (site-id 3000 à 3099).
- Hub and spoke imposé par OMP, BFD sur chaque tunnel, classes SLA et AAR en place.

### 15.2 Plan de segmentation retenu

| VPN | Segment | Sites concernés | Topologie | Accès internet | Observations |
|---|---|---|---|---|---|
| 0 | Transport | Tous | Sans objet | Sans objet | Interfaces WAN, couleurs |
| 10 | Caisses et paiement | Tous les magasins et datacenters | Hub and spoke strict | Aucun | Périmètre de paiement |
| 20 | Back office | Magasins, entrepôts, datacenters | Hub and spoke, direct entre entrepôts | Via datacenter | ERP, stocks |
| 30 | Vidéosurveillance | Magasins, datacenter d'enregistrement | Hub and spoke | Aucun | Flux volumineux, classe SLA souple |
| 40 | Invités | Magasins | Local uniquement | Sortie locale | Non annoncé dans OMP |
| 50 | IoT | Magasins, datacenters | Hub and spoke | Via datacenter, liste restreinte | Balances, étiquettes, froid |
| 60 | Partenaires et concessionnaires | Grands magasins | Local ou vers passerelle dédiée | Selon contrat | Absent des magasins de proximité |
| 90 | Services partagés | Datacenters, magasins (consommateurs) | Hub and spoke | Aucun | DNS, NTP, supervision |
| 512 | Gestion hors bande | Tous | Sans objet | Aucun | Administration |

### 15.3 Adaptation par type de site

| Type de site | VPN présents |
|---|---|
| Hypermarché et supermarché | 10, 20, 30, 40, 50, 60, 90 |
| Proximité | 10, 20, 30, 40, 50, 90 |
| Entrepôt | 20, 30, 50, 90 |
| Datacenter | 10, 20, 30, 50, 60, 90 |

La VPN membership policy impose ces règles de façon centralisée.

### 15.4 Scénario 1 : un objet connecté compromis

Une balance connectée du VPN 50 est compromise par un attaquant qui cherche à atteindre les caisses.

1. L'attaquant est dans le VPN 50 du magasin. Aucune route vers le VPN 10 n'existe dans cette table de routage : les caisses sont inaccessibles.
2. Les routes du VPN 50 reçues par OMP ne pointent que vers le datacenter (hub and spoke) : l'attaquant ne voit pas les autres magasins.
3. Les échanges autorisés vers le VPN 90 (mises à jour, supervision) sont filtrés par le pare-feu de zones.
4. Les journaux du pare-feu et les alarmes de comportement déclenchent l'investigation.

Sans segmentation, l'attaquant aurait été sur le même réseau que les terminaux de paiement.

### 15.5 Scénario 2 : l'ouverture d'un nouveau magasin de proximité

1. Le modèle d'équipement « proximité » est appliqué au nouveau routeur (ZTP).
2. Les VPN 10, 20, 30, 40, 50 et 90 sont créés avec les variables du site (site-id, préfixes dérivés).
3. La VPN membership du groupe « proximité » n'envoie au site que les routes de ces VPN.
4. Aucun VPN 60 n'est créé : le magasin n'a pas de galerie marchande.
5. Les tests de segmentation sont exécutés avant l'ouverture au public.

### 15.6 Scénario 3 : une galerie marchande avec concessionnaires

Dans un hypermarché, des concessionnaires (boulangerie, parapharmacie, point de retrait) veulent un accès internet et un lien vers leur propre siège.

- Chaque concessionnaire est placé dans le VPN 60, sur un VLAN et un SSID dédiés.
- Le VPN 60 n'a aucune route vers les VPN internes.
- Le pare-feu autorise uniquement DNS vers le VPN 90 et la sortie internet.
- Les concessionnaires partagent le même plan d'adressage (172.16.60.0/24) dans tous les magasins : cela ne pose pas de problème car le VPN 60 n'est pas annoncé dans OMP.

Si l'enseigne devait isoler chaque concessionnaire des autres, elle utiliserait des VLAN et des règles de pare-feu distincts, ou des VPN supplémentaires.

### 15.7 Scénario 4 : erreur de câblage

Un technicien branche une caméra IP dans un port du VLAN des caisses.

- La caméra se retrouve dans le VPN 10 et peut communiquer avec des équipements de paiement.
- Défense en profondeur : authentification des équipements (802.1X ou liste d'adresses MAC), alarmes sur l'apparition d'un équipement inconnu, vérification périodique de l'inventaire, tests de segmentation.
- La segmentation SD-WAN seule ne détecte pas ce type d'erreur : elle suppose un LAN correct.

### 15.8 Scénario 5 : mise à jour des terminaux de paiement

Les terminaux de paiement doivent télécharger leurs mises à jour logicielles auprès d'un serveur du datacenter.

- Les terminaux (VPN 10) accèdent à un serveur de mise à jour exposé dans le VPN 90.
- Une fuite de routes limitée au préfixe du serveur (et non au VPN 90 entier) autorise ce flux.
- Le pare-feu de zones restreint l'échange au port et au protocole nécessaires.
- Cette exception est consignée dans le registre des fuites de routes.

### 15.9 Dimensionnement

Pour un datacenter, avec 6 VPN annoncés par 1000 magasins et un préfixe par VPN, l'ordre de grandeur est de 6000 préfixes, soit jusqu'à 12 000 chemins OMP si chaque magasin expose deux couleurs. Cette charge reste à vérifier au regard des limites de la plateforme choisie.

---

## 16. Configuration pas à pas

### 16.1 VPN de transport et de gestion (vEdge)

```
vpn 0
 interface ge0/0
  ip address 203.0.113.10/30
  tunnel-interface
   encapsulation ipsec
   color biz-internet
   allow-service dhcp
   allow-service dns
   allow-service icmp
  !
  no shutdown
 !
 ip route 0.0.0.0/0 203.0.113.9
!
vpn 512
 interface mgmt0
  ip address 192.0.2.10/24
  no shutdown
 !
!
```

### 16.2 VPN de service sur un magasin (vEdge)

```
vpn 10
 interface ge0/2.110
  ip address 10.20.75.1/24
  no shutdown
 !
!
vpn 20
 interface ge0/2.120
  ip address 10.21.75.1/24
  no shutdown
 !
!
vpn 30
 interface ge0/2.130
  ip address 10.22.75.1/24
  no shutdown
 !
!
vpn 50
 interface ge0/2.150
  ip address 10.23.75.1/24
  no shutdown
 !
!
omp
 no shutdown
 graceful-restart
 advertise connected
 advertise static
!
```

### 16.3 Équivalent sur cEdge (IOS XE, indicatif)

```
vrf definition 10
 rd 1:10
 !
 address-family ipv4
  route-target export 1:10
  route-target import 1:10
 exit-address-family
!
interface GigabitEthernet0/0/2.110
 encapsulation dot1Q 110
 vrf forwarding 10
 ip address 10.20.75.1 255.255.255.0
 no shutdown
!
sdwan
 omp
  no shutdown
  graceful-restart
  address-family ipv4
   advertise connected
   advertise static
  exit
 !
!
```

En exploitation réelle, on utilise les modèles de vManage, avec des variables de site, et non la ligne de commande.

### 16.4 Raccordement du datacenter : une sous-interface par VPN vers le pare-feu

Au datacenter, chaque VPN est remis au cluster de pare-feu sur une sous-interface dédiée (technique VRF-lite), avec un protocole de routage par VPN.

| VPN | VLAN de remise | Protocole | Remarque |
|---|---|---|---|
| 10 | 810 | eBGP | Vers le pare-feu de la zone paiement |
| 20 | 820 | eBGP | Vers le pare-feu de la zone applications |
| 30 | 830 | OSPF ou eBGP | Vers le système d'enregistrement vidéo |
| 50 | 850 | eBGP | Vers la zone IoT |
| 90 | 890 | eBGP | Services partagés |

Configuration indicative d'un VPN (vEdge) :

```
vpn 10
 interface ge0/2.810
  ip address 10.255.110.1/30
  no shutdown
 !
 router
  bgp 65001
   neighbor 10.255.110.2
    no shutdown
    remote-as 65100
    address-family ipv4-unicast
    !
   !
   address-family ipv4-unicast
    redistribute omp
   !
  !
 !
!
omp
 advertise bgp
!
```

### 16.5 Test d'isolation

Depuis un poste du VPN 40 (invités), tenter de joindre une adresse du VPN 10 : la tentative doit échouer. Répéter les tests dans les deux sens, pour chaque paire de segments, y compris sur des sites distants.

---

## 17. Vérification et dépannage

### 17.1 Commandes principales (cEdge)

| Commande | Utilité |
|---|---|
| `show vrf` | Liste des VRF et des interfaces associées |
| `show ip route vrf 10` | Table de routage du VPN 10 |
| `show sdwan omp routes vpn 10` | Routes OMP du VPN 10 |
| `show sdwan omp summary` | Résumé OMP (routes reçues, installées, envoyées) |
| `show sdwan policy from-vsmart` | Politiques reçues, dont la membership |
| `show sdwan omp services` | Service routes (pare-feu, etc.) |
| `show sdwan omp tlocs` | TLOC et leur état |
| `show sdwan bfd sessions` | Sessions BFD |
| `show sdwan ipsec outbound-connections` | Tunnels et labels utilisés |
| `show interfaces`, `show ip interface brief vrf 10` | Interfaces par VRF |

Sur vEdge : `show ip route vpn 10`, `show omp routes vpn 10`, `show policy from-vsmart`.

### 17.2 Exemple de sortie : routes du VPN 10 (illustratif)

```
Magasin-2750# show sdwan omp routes vpn 10

VPN  PREFIX           FROM PEER      PATH  LABEL  STATUS  TLOC IP      COLOR         ENCAP  PREF
-------------------------------------------------------------------------------------------------
10   10.50.0.0/16     10.255.255.1   1     1010   C,I,R   10.255.0.1   mpls          ipsec  200
10   10.50.0.0/16     10.255.255.1   2     1010   C,I,R   10.255.0.1   biz-internet  ipsec  200
10   10.50.0.0/16     10.255.255.1   3     1010   R       10.255.0.2   mpls          ipsec  100
```

Lecture : le magasin ne voit dans le VPN 10 que les routes du datacenter (hub and spoke). Le label identifie le VPN.

### 17.3 Exemple de sortie : une VPN membership effective (illustratif)

```
Magasin-2750# show sdwan omp summary

oper-state               UP
vsmart-peers             2
routes-received          5560
routes-installed         5560
routes-sent              18
vpn-membership-active    10,20,30,50,90
```

Les noms de champs sont donnés à titre illustratif et varient selon la version.

### 17.4 Méthodologie de dépannage

1. Vérifier l'appartenance de l'interface au bon VRF (`show vrf`, `show ip interface brief vrf 10`).
2. Vérifier la table de routage locale du VPN (`show ip route vrf 10`) : le préfixe local est-il présent ?
3. Vérifier l'annonce vers OMP : `show sdwan omp routes vpn 10` sur l'émetteur, section des routes envoyées.
4. Vérifier la politique reçue : la membership autorise-t-elle ce VPN pour ce site ? La politique de contrôle accepte-t-elle les routes ?
5. Vérifier les TLOC et BFD : la route est-elle résolue ?
6. Vérifier la route de retour : le site distant a-t-il une route vers le préfixe source, dans le même VPN ?
7. Vérifier les politiques de pare-feu et les fuites de routes pour les échanges inter-segments.
8. Vérifier le côté LAN : VLAN, trunk, sous-interface, adresse de la passerelle, DHCP.

### 17.5 Causes fréquentes d'incident

| Symptôme | Cause probable |
|---|---|
| Un poste ne joint pas le datacenter | Interface dans le mauvais VPN, route non annoncée dans OMP, membership trop restrictive |
| Route absente chez le site distant | Pas de `advertise` pour ce protocole, politique de contrôle ou membership qui la filtre |
| Route présente mais trafic perdu | Route de retour manquante, TLOC non résolu, pare-feu |
| Deux segments qui communiquent alors qu'ils ne devraient pas | Fuite de routes trop large, VLAN partagé, erreur de câblage |
| Trafic invité qui atteint le réseau interne | Route par défaut du VPN 40 mal conçue, fuite vers VPN 0, absence de pare-feu |
| Chevauchement qui provoque des comportements étranges | Fuite de routes entre deux VPN qui contiennent le même préfixe |
| VPN absent sur un site | Modèle d'équipement incomplet, membership refusée |

---

## 18. Passage à l'échelle

### 18.1 Facteurs de charge

- Nombre de VPN x nombre de préfixes par VPN x nombre de sites : taille de la table OMP.
- Nombre de chemins par préfixe (couleurs, `send-path-limit`, ECMP).
- Nombre de VRF supportées par la plateforme du WAN Edge.
- Nombre de voisins de routage (OSPF, BGP) par VRF au datacenter.

### 18.2 Leviers de maîtrise

- VPN membership pour que les petits sites ne reçoivent que ce dont ils ont besoin.
- Agrégation de préfixes au niveau du datacenter (route par défaut ou résumés dans chaque VPN).
- Limitation du nombre de chemins annoncés.
- Répartition des VPN ou des magasins sur plusieurs routeurs hub.
- Limitation du nombre de VPN à ceux qui sont vraiment justifiés.

### 18.3 Nombre de VPN : règle de jugement

Chaque VPN supplémentaire améliore l'isolation mais ajoute de la complexité (routage, politiques, pare-feu, tests). Un nombre de VPN de l'ordre de quelques unités à une dizaine suffit pour la plupart des enseignes. Au-delà, la gouvernance et l'automatisation deviennent indispensables.

---

## 19. Pièges classiques

1. Confondre VPN SD-WAN (segment) et VPN IPsec d'accès distant.
2. Croire que la segmentation remplace un pare-feu.
3. Créer trop de segments sans automatisation ni gouvernance.
4. Faire fuiter un VPN entier au lieu de préfixes précis.
5. Oublier la route de retour dans le VPN de destination.
6. Laisser le VPN 512 accessible depuis le LAN ou des segments utilisateurs.
7. Ouvrir trop de services sur les interfaces du VPN 0 (`allow-service`).
8. Utiliser des fuites de routes entre VPN contenant le même préfixe.
9. Oublier d'inclure un VPN dans la membership d'un datacenter ou d'un entrepôt.
10. Ne jamais tester l'isolation après un changement de politique.
11. Négliger la segmentation du LAN (VLAN, Wi-Fi) en comptant sur le WAN.
12. Documenter le plan de segmentation une fois, puis ne jamais le mettre à jour.

---

## 20. Bonnes pratiques

1. Définir le plan de segmentation par les besoins métier et de conformité avant de toucher la configuration.
2. Normaliser et documenter la numérotation des VPN et des VLAN.
3. Utiliser les modèles de vManage avec des variables de site pour garantir la cohérence.
4. Appliquer une VPN membership par type de site (principe du moindre privilège).
5. Définir une topologie par VPN, adaptée au besoin (par exemple hub and spoke strict pour les paiements).
6. Traiter chaque fuite de routes comme une exception : minimale, documentée, revue périodiquement.
7. Placer un pare-feu sur les échanges inter-segments autorisés.
8. Choisir VPN par VPN la politique d'accès internet.
9. Protéger le VPN 512 et le VPN 0 (accès administrateur, `allow-service` minimal).
10. Tester l'isolation de façon régulière et après chaque changement.
11. Aligner la segmentation du LAN (VLAN, SSID, authentification) sur celle du WAN.
12. Appliquer aux VPN critiques des politiques de qualité de service, de routage applicatif et de classes SLA spécifiques.

---

## 21. Travaux pratiques

### TP 1 : Créer deux segments

1. Dans un laboratoire à deux WAN Edge, créer les VPN 10 et 40 avec des préfixes LAN distincts.
2. Annoncer ces préfixes dans OMP.
3. Vérifier avec `show omp routes vpn 10` et `vpn 40` que chaque VPN voit ses propres routes.
4. Tenter un ping entre un poste du VPN 10 et un poste du VPN 40 : il doit échouer.

### TP 2 : Topologie par VPN

1. Créer trois sites : un hub et deux spokes.
2. Appliquer une politique hub and spoke pour le VPN 10 uniquement, en laissant le VPN 20 en full mesh.
3. Vérifier les tunnels et les routes de chaque VPN.

### TP 3 : VPN membership

1. Créer les VPN 10, 20 et 30 sur le hub.
2. Appliquer une membership qui n'autorise que les VPN 10 et 20 pour un spoke.
3. Constater l'absence de routes du VPN 30 sur ce spoke.

### TP 4 : Fuite de routes contrôlée

1. Créer un VPN 90 qui héberge un serveur DNS.
2. Faire fuiter uniquement le préfixe du serveur vers le VPN 10.
3. Vérifier que le VPN 10 joint le DNS mais pas les autres machines du VPN 90.
4. Ajouter une règle de pare-feu limitant l'accès au port DNS.

### TP 5 : DIA par segment

1. Activer la sortie internet locale pour le VPN 40 uniquement.
2. Vérifier que le VPN 10 n'a pas de route vers internet.
3. Tester l'accès internet depuis les deux VPN.

### TP 6 : Chevauchement d'adressage

1. Attribuer le même préfixe 192.168.40.0/24 à des VPN 40 sur deux sites.
2. Vérifier qu'aucune route n'est annoncée dans OMP.
3. Discuter de ce qui se passerait si on faisait fuiter ce préfixe.

### TP 7 : Audit de segmentation

1. Établir la matrice complète des paires de VPN (source, destination).
2. Pour chaque paire, indiquer le comportement attendu (interdit, autorisé, filtré).
3. Tester chaque paire et consigner les écarts.

---

## 22. Questionnaire d'auto-évaluation

**Q1. Quelle est la différence entre le VPN 0 et un VPN de service ?**
Réponse : le VPN 0 est le VPN de transport (interfaces WAN, couleurs, connexions de contrôle). Un VPN de service porte le trafic des utilisateurs et des services.

**Q2. À quoi sert le VPN 512 ?**
Réponse : à la gestion hors bande des équipements. Il ne participe pas à l'overlay.

**Q3. Comment un WAN Edge distant sait-il dans quel VPN injecter un paquet reçu ?**
Réponse : grâce au label de VPN présent dans l'encapsulation, associé aux routes OMP.

**Q4. Y a-t-il un tunnel IPsec par VPN ?**
Réponse : non. Les tunnels sont établis entre TLOC et portent tous les VPN, différenciés par le label.

**Q5. À quoi sert une VPN membership policy ?**
Réponse : à définir, pour chaque groupe de sites, les VPN auxquels ils ont le droit de participer, donc les routes qu'ils reçoivent.

**Q6. Comment permettre à des postes du VPN 10 d'atteindre uniquement un serveur DNS du VPN 90 ?**
Réponse : par une fuite de routes limitée au préfixe du serveur, complétée par une règle de pare-feu limitée au port DNS.

**Q7. Deux VPN peuvent-ils utiliser le même préfixe ?**
Réponse : oui, car chacun a sa table de routage. Il faut éviter les fuites entre eux.

**Q8. Pourquoi la segmentation ne suffit-elle pas pour être conforme à PCI DSS ?**
Réponse : elle réduit le périmètre mais doit être validée par des tests et complétée par d'autres contrôles (pare-feu, journalisation, gestion des accès, sécurité des terminaux et du LAN).

**Q9. Que se passe-t-il si une caméra est branchée dans le VLAN des caisses ?**
Réponse : elle se retrouve dans le VPN 10. La segmentation du WAN ne l'empêche pas : il faut une authentification des équipements, un inventaire et des alarmes.

**Q10. Quel est l'effet d'une route par défaut `ip route 0.0.0.0/0 vpn 0` dans un VPN de service ?**
Réponse : elle fait sortir ce VPN vers l'internet local via le VPN 0 (DIA). C'est une fuite maîtrisée à réserver aux segments voulus.

---

## 23. Glossaire

| Terme | Définition |
|---|---|
| CDE | Cardholder Data Environment, environnement de données de titulaires de cartes |
| DIA | Direct Internet Access, sortie internet locale |
| Fuite de routes (route leaking) | Partage contrôlé de routes entre deux VPN |
| Label de VPN | Étiquette qui identifie le VPN d'un paquet dans l'encapsulation de l'overlay |
| Membership policy | Politique centralisée qui définit les VPN autorisés pour un groupe de sites |
| QSA | Qualified Security Assessor, évaluateur PCI DSS qualifié |
| Segment | VPN de service dans la conception du réseau |
| SGT | Security Group Tag, étiquette de groupe de sécurité |
| SIG | Secure Internet Gateway, passerelle internet sécurisée |
| VLAN | Réseau local virtuel de niveau 2 |
| VPN de service | VPN qui porte le trafic des utilisateurs |
| VRF | Virtual Routing and Forwarding, instance de routage isolée |
| VRF-lite | Utilisation de VRF avec sous-interfaces sans MPLS pour étendre des segments |
| ZBFW | Zone-Based Firewall, pare-feu à états par zones |

---

## 24. Références

- Documentation officielle Cisco SD-WAN : guides de configuration Segmentation, Policies (VPN membership, route leaking) et Security (pare-feu par zones).
- Cisco SD-WAN Design Guide et guides de conception de la segmentation (Cisco Validated Design).
- PCI Security Standards Council : PCI DSS et guide sur la segmentation du réseau.
- RFC 4364 : BGP/MPLS IP Virtual Private Networks (pour la comparaison avec les L3VPN MPLS).
- Cours associés : Cours complet SD-WAN et protocole OMP, Cours complet SD-WAN et protocole BFD, Cours complet SD-WAN et SLA, Cours complet SD-WAN blackout et brownout.

Les bornes de numérotation, les limites de VRF par plateforme, la disponibilité des fonctions de fuite de routes et de pare-feu, ainsi que les syntaxes de commandes évoluent selon les versions et les plateformes : validez-les dans la documentation correspondant à votre version avant toute mise en production. Les exigences de conformité doivent être confirmées avec votre responsable conformité et un évaluateur qualifié.
