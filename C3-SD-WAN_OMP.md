# Cours complet SD-WAN et protocole OMP (Overlay Management Protocol)

Ce cours présente l'architecture SD-WAN (en s'appuyant sur la solution Cisco SD-WAN, ex-Viptela) puis étudie en profondeur le protocole OMP, le cerveau du plan de contrôle. Un fil rouge illustre chaque notion : le déploiement d'un réseau SD-WAN pour une enseigne de grande distribution de type Carrefour.

> Note importante : le cas Carrefour est un scénario pédagogique fictif, inspiré des besoins typiques d'une grande enseigne de distribution. Les plans d'adressage, les identifiants de sites et les volumes sont inventés à des fins d'illustration et ne décrivent pas l'architecture réelle de l'entreprise.

---

## Table des matières

1. Objectifs et prérequis
2. Rappels : pourquoi le SD-WAN
3. Architecture SD-WAN : les quatre plans
4. Les composants Cisco SD-WAN
5. Vocabulaire indispensable : TLOC, color, VPN, site-id, system-ip
6. Le protocole OMP en détail
7. Les trois types de routes OMP
8. Les attributs OMP
9. Sélection du meilleur chemin et ECMP
10. Politiques centralisées de contrôle
11. Cas d'étude fil rouge : le réseau d'une enseigne de distribution
12. Configuration d'OMP pas à pas
13. Vérification et dépannage
14. Sécurité et résilience du plan de contrôle
15. Bonnes pratiques de conception
16. Travaux pratiques
17. Questionnaire d'auto-évaluation
18. Glossaire
19. Références

---

## 1. Objectifs et prérequis

### Objectifs pédagogiques

À la fin de ce cours, vous serez capable de :

- Expliquer le rôle du SD-WAN et ses différences avec un WAN MPLS classique.
- Décrire l'architecture à quatre plans et les composants Cisco SD-WAN.
- Expliquer le fonctionnement d'OMP : sessions, types de routes, attributs, annonces.
- Lire et interpréter les sorties `show sdwan omp ...`.
- Écrire une politique de contrôle centralisée (hub and spoke, préférence de TLOC, service chaining).
- Concevoir et dépanner le plan de contrôle d'un réseau multi-sites.

### Prérequis

- Bases du routage IP (OSPF, BGP, VRF).
- Notions de VPN IPsec et de NAT.
- Notions de TLS/DTLS et de PKI (certificats).

---

## 2. Rappels : pourquoi le SD-WAN

### 2.1 Limites du WAN traditionnel

Un WAN d'entreprise classique repose sur :

- Un transport MPLS opérateur, fiable mais coûteux et long à provisionner.
- Une configuration équipement par équipement (CLI), source d'erreurs.
- Un trafic internet et cloud ramené (backhauling) vers le datacenter, ce qui dégrade l'expérience SaaS.
- Une visibilité limitée sur les applications et la qualité réelle des liens.

### 2.2 Ce qu'apporte le SD-WAN

| Besoin | Réponse SD-WAN |
|---|---|
| Réduire le coût du transport | Combiner MPLS, internet haut débit et 4G/5G dans un même overlay |
| Agilité de déploiement | Zero Touch Provisioning (ZTP) et templates centralisés |
| Qualité applicative | Choix du chemin selon perte, latence et gigue mesurées en temps réel |
| Sécurité | Chiffrement IPsec de bout en bout, segmentation par VPN, pare-feu intégré |
| Accès cloud | Local internet breakout et intégration aux fournisseurs cloud |
| Pilotage | Console unique, politiques centralisées, télémétrie |

### 2.3 Principe fondamental

Le SD-WAN sépare :

- L'underlay : les transports physiques (MPLS, internet, 4G/5G).
- L'overlay : un réseau virtuel chiffré construit au-dessus des transports, piloté de façon centralisée.

---

## 3. Architecture SD-WAN : les quatre plans

Cisco SD-WAN repose sur une séparation claire des fonctions :

```
                +---------------------------------------+
  Plan de       |   vManage (NMS : config, supervision) |
  gestion       +---------------------------------------+
                                  |
                +---------------------------------------+
  Plan          |   vBond (authentification, NAT, SD    |
  d'orchestration|  orchestrateur de la découverte)    |
                +---------------------------------------+
                                  |
                +---------------------------------------+
  Plan de       |   vSmart (contrôleur : OMP, politiques)|
  contrôle      +---------------------------------------+
                      |  DTLS/TLS + OMP       |
              +-------+-----+          +------+------+
              | WAN Edge A  |==IPsec===| WAN Edge B  |   Plan de données
              +-------------+          +-------------+
```

| Plan | Composant | Rôle |
|---|---|---|
| Gestion | vManage | Configuration, templates, supervision, API |
| Orchestration | vBond | Premier contact, authentification mutuelle, aide au franchissement du NAT |
| Contrôle | vSmart | Distribution des routes, clés et politiques via OMP |
| Données | WAN Edge (vEdge ou cEdge) | Acheminement du trafic utilisateur, IPsec, BFD, politiques de données |

Idée clé : le plan de contrôle est centralisé et logiquement séparé du plan de données. Les WAN Edge ne forment pas de sessions OMP entre eux : ils dialoguent uniquement avec les vSmart.

---

## 4. Les composants Cisco SD-WAN

### 4.1 vManage

- Interface graphique et API REST.
- Gère les templates de configuration (feature templates et device templates), les politiques, les mises à jour logicielles, la supervision.

### 4.2 vBond

- Seul composant qui doit avoir une adresse IP publique joignable et fixe.
- Authentifie les équipements (vEdge, vSmart, vManage) et leur indique quels vSmart et vManage contacter.
- Joue un rôle de STUN-like pour détecter le NAT.
- Après la phase initiale, les WAN Edge conservent une connexion DTLS avec vBond uniquement le temps nécessaire (ils ne maintiennent pas de session permanente avec lui, contrairement à vSmart et vManage).

### 4.3 vSmart

- Contrôleur central. Terminaison des sessions OMP.
- Se comporte comme un route reflector : il reçoit les routes des WAN Edge, applique les politiques de contrôle, puis les redistribue aux autres WAN Edge.
- Il peut y en avoir plusieurs (redondance, au moins deux en production).

### 4.4 WAN Edge

- vEdge : plateforme historique Viptela (logiciel Viptela OS).
- cEdge : routeurs Cisco IOS XE exécutant le mode SD-WAN (ISR, ASR, Catalyst 8000).
- Rôle : construire les tunnels IPsec, exécuter BFD, appliquer les politiques, interfacer le LAN (OSPF, BGP, EIGRP, statique).

---

## 5. Vocabulaire indispensable

### 5.1 System IP

Identifiant unique d'un équipement dans l'overlay, au format IPv4 (exemple 10.255.0.1). Il n'est pas forcément routable : il sert d'identifiant, comparable à un router-id.

### 5.2 Site ID

Entier (1 à 4294967295) identifiant le site. Tous les équipements d'un même site partagent le même site-id. Ce paramètre est capital dans les politiques de contrôle (site-list).

### 5.3 VPN (VRF de service)

Segment logique de l'overlay. Conventions Cisco :

- VPN 0 : transport (underlay, interfaces WAN, contrôle).
- VPN 512 : gestion hors bande.
- VPN 1 à 511 : VPN de service (LAN utilisateurs, caisses, etc.).

### 5.4 TLOC (Transport Locator)

Un TLOC est le point d'attache d'un équipement à un transport. Il est identifié par le triplet :

```
TLOC = ( System IP , Color , Encapsulation )
```

Exemple : (10.255.1.1, mpls, ipsec). Il joue le rôle du "next hop" dans l'overlay.

### 5.5 Color

Étiquette symbolique d'un type de transport : `mpls`, `biz-internet`, `public-internet`, `lte`, `3g`, `metro-ethernet`, `gold`, `silver`, `bronze`, `custom1`, etc.

- Les couleurs privées (mpls, metro-ethernet, private1 à private6) supposent une adresse joignable directement, sans NAT.
- Les couleurs publiques (biz-internet, public-internet, lte, 3g...) supposent un possible NAT, l'équipement utilise donc l'adresse publique pour tenter l'établissement des tunnels.
- Par défaut, deux TLOC ne montent un tunnel que s'ils ont la même couleur, sauf usage du paramètre `restrict` (il restreint l'établissement aux TLOC de même couleur) ou de la politique de contrôle.

### 5.6 Encapsulation

IPsec (par défaut, chiffré) ou GRE (non chiffré, usage de transition).

---

## 6. Le protocole OMP en détail

### 6.1 Définition

OMP (Overlay Management Protocol) est le protocole de contrôle de l'overlay SD-WAN. Il fonctionne entre chaque WAN Edge et les vSmart. Inspiré de BGP, il s'exécute à l'intérieur d'une connexion de contrôle chiffrée (DTLS ou TLS) et transporte :

- Les routes de préfixes des sites (comme le ferait un IGP/BGP).
- Les TLOC et leurs adresses (privées et publiques) pour construire le plan de données.
- Les routes de services (pare-feu, IDS, etc.) pour le service chaining.
- Les clés de chiffrement des tunnels de données (dans le modèle de distribution des clés par vSmart).
- Les politiques de contrôle et de données issues de vManage, relayées par vSmart.

### 6.2 Topologie des sessions

```
        vSmart-1            vSmart-2
         /  |  \            /  |  \
        /   |   \          /   |   \
   Edge1  Edge2  Edge3  Edge1  Edge2  Edge3
```

- Chaque WAN Edge établit une session OMP avec chaque vSmart (maille complète côté Edge vers vSmart).
- Il n'y a pas de session OMP Edge vers Edge.
- Les vSmart partagent une vue cohérente grâce à la redondance des sessions (ils ne se synchronisent pas entre eux via OMP, c'est chaque Edge qui les alimente).

### 6.3 Établissement de la session

1. Le WAN Edge démarre (ZTP ou configuration manuelle).
2. Il contacte vBond (DTLS, port UDP 12346 par défaut), s'authentifie par certificat.
3. vBond lui renvoie l'adresse des vSmart et vManage.
4. Le WAN Edge ouvre une connexion de contrôle DTLS (ou TLS en TCP) avec chaque vSmart.
5. La session OMP est lancée à l'intérieur de cette connexion.
6. Échange des routes OMP, TLOC et services, puis des politiques.
7. Le WAN Edge construit ensuite les tunnels IPsec vers les TLOC distants appris et lance BFD.

### 6.4 Ports et transport

- Port de base DTLS : UDP 12346.
- Port hopping : le WAN Edge peut utiliser un décalage (jusqu'à 12446) lorsque plusieurs instances ou des NAT sont présents, afin de faciliter l'établissement des tunnels.
- Pour TLS, le contrôleur écoute généralement en TCP 23456 (valeur par défaut Cisco selon version, à vérifier dans la documentation de votre release).

### 6.5 Timers OMP (valeurs usuelles par défaut)

| Paramètre | Valeur par défaut | Rôle |
|---|---|---|
| Keepalive (hello) | 20 s | Vérifie que le pair est vivant |
| Hold time | 60 s | Délai avant déclaration de session perdue |
| Advertisement interval | 1 s | Cadence minimale d'envoi des mises à jour |
| Graceful restart timer | 43200 s (12 h) | Durée de conservation des routes d'un pair injoignable |
| EOR timer | 300 s | Attente de la fin de redémarrage (End of RIB) |

### 6.6 Graceful restart

Si le plan de contrôle est interrompu (par exemple panne des deux vSmart), les WAN Edge conservent les routes et les clés apprises pendant la durée du graceful restart. Le trafic continue donc de circuler sur le plan de données pendant la panne du contrôleur. Ce point est essentiel pour la résilience : une panne du plan de contrôle ne coupe pas immédiatement les flux existants.

### 6.7 Redistribution entre OMP et le LAN

Un WAN Edge apprend des préfixes côté service (LAN) par OSPF, BGP, EIGRP, statique ou connecté, puis les injecte dans OMP avec la commande `advertise`. Inversement, les routes OMP apprises sont installées dans la table de routage du VPN et peuvent être redistribuées vers OSPF ou BGP côté LAN.

---

## 7. Les trois types de routes OMP

### 7.1 Routes OMP (préfixes)

Ce sont les routes de préfixes IPv4/IPv6 des VPN de service. Elles ressemblent à des routes BGP : un préfixe, un next hop (qui est ici un TLOC), des attributs.

### 7.2 TLOC routes

Elles annoncent les TLOC. Elles contiennent l'adresse privée et publique du TLOC, le port, la couleur, l'encapsulation, le carrier, la préférence et, selon le mode, les éléments liés aux clés de chiffrement. Une route de préfixe n'est utilisable que si le TLOC associé est résolu (joignable et BFD up).

### 7.3 Service routes

Elles annoncent les services disponibles derrière un WAN Edge (pare-feu `FW`, `IDS`, `IDP`, `netsvc1` à `netsvc4`, etc.). Elles permettent d'insérer ces services dans un chemin (service chaining) par une politique de contrôle.

---

## 8. Les attributs OMP

### 8.1 Attributs d'une route de préfixe

| Attribut | Signification |
|---|---|
| VPN | VPN de service concerné |
| Prefix | Préfixe annoncé |
| Originator | System IP de l'équipement qui a émis la route |
| From peer | Pair OMP qui a transmis la route |
| Path ID | Identifiant unique du chemin pour un même préfixe (permet plusieurs chemins) |
| Label | Étiquette de service/VPN utilisée dans l'encapsulation IPsec |
| Status | État de la route (voir codes ci-dessous) |
| Attach point | TLOC next hop (System IP, color, encap) |
| Preference | Préférence OMP de la route (plus élevé est meilleur) |
| Site ID | Site d'origine |
| Tag | Étiquette administrative utilisable dans les politiques |
| Origin protocol | Protocole d'origine (connected, static, ospf, bgp, eigrp...) |
| Origin metric | Métrique d'origine (par exemple métrique OSPF) |

### 8.2 Codes d'état d'une route OMP

| Code | Signification |
|---|---|
| C | Chosen : route retenue comme meilleure |
| I | Installed : installée dans la table de routage |
| Red | Redistributed : redistribuée |
| Rej | Rejected : rejetée par une politique |
| L | Looped : boucle détectée |
| R | Resolved : le TLOC next hop est résolu |
| S | Stale : route conservée en graceful restart |
| Inv | Invalid : route invalide |
| U | Unresolved : TLOC non résolu |
| Ext | Extranet |
| Stg | Staged |

Une route utile apparaît typiquement avec les statuts `C,I,R`.

### 8.3 Attributs d'un TLOC

- System IP, Color, Encapsulation.
- Public IP et port, Private IP et port (utiles pour le NAT traversal).
- Carrier, Preference, Weight, Tag, Site ID.
- État BFD, adresse d'origine, groupe de restriction (`restrict`).

---

## 9. Sélection du meilleur chemin et ECMP

### 9.1 Principe

Un préfixe peut être annoncé par plusieurs sites (par exemple deux datacenters). Chaque WAN Edge reçoit plusieurs chemins et applique un algorithme de sélection, puis installe le ou les meilleurs dans sa table de routage.

### 9.2 Ordre de décision (simplifié)

Cet ordre est une présentation pédagogique simplifiée. Vérifiez l'ordre exact dans la documentation de votre version logicielle.

1. Écarter les routes invalides ou dont le TLOC n'est pas résolu.
2. Préférer la préférence OMP la plus élevée.
3. Préférer la préférence de TLOC la plus élevée.
4. Comparer le type de protocole d'origine selon un ordre fixe (connected, static, puis protocoles de routage).
5. Préférer la métrique d'origine la plus faible.
6. Départager par le System IP (router-id) de l'originateur le plus faible, puis par l'adresse privée du TLOC.

### 9.3 ECMP et send-path-limit

- `ecmp-limit` : nombre maximal de chemins à coût égal installés (défaut 4, maximum 16).
- `send-path-limit` : nombre maximal de chemins qu'un WAN Edge envoie au vSmart pour un même préfixe, et qu'un vSmart redistribue (défaut 4, maximum 16).

Exemple : un hypermarché dispose de 3 liens (MPLS, internet, 4G). Chaque lien produit un TLOC. Le datacenter reçoit donc plusieurs chemins vers le préfixe du magasin et peut répartir la charge en ECMP.

---

## 10. Politiques centralisées de contrôle

Les politiques de contrôle sont écrites sur vSmart (via vManage) et agissent sur les annonces OMP. Elles définissent la topologie logique de l'overlay.

### 10.1 Structure

- Listes : `site-list`, `vpn-list`, `tloc-list`, `prefix-list`.
- Une `control-policy` composée de séquences `match` et `action`.
- Un `apply-policy` qui attache la politique à une site-list, avec un sens `in` ou `out`.

Sens :

- `in` : appliqué aux routes reçues par vSmart depuis les sites concernés.
- `out` : appliqué aux routes que vSmart envoie vers les sites concernés.

Sans politique, la topologie par défaut est un full mesh : chaque site reçoit toutes les routes et peut établir des tunnels directs avec tous les autres.

### 10.2 Exemple 1 : topologie hub and spoke

Objectif : les magasins (spokes) ne communiquent qu'avec les datacenters (hubs), pas directement entre eux.

```
policy
 lists
  site-list DATACENTERS
   site-id 1000-1001
  site-list MAGASINS
   site-id 2000-2999
 !
 control-policy HUB-AND-SPOKE
  sequence 1
   match tloc
    site-list DATACENTERS
   !
   action accept
   !
  !
  sequence 11
   match route
    site-list DATACENTERS
   !
   action accept
   !
  !
  default-action reject
 !
!
apply-policy
 site-list MAGASINS
  control-policy HUB-AND-SPOKE out
 !
!
```

Explication :

- Les magasins ne reçoivent que les TLOC et les routes des datacenters.
- Aucun tunnel direct magasin vers magasin n'est donc construit.
- Les datacenters doivent annoncer un agrégat ou une route par défaut pour que les magasins puissent joindre les autres sites via le hub.

### 10.3 Exemple 2 : préférer un transport

Objectif : privilégier le MPLS pour les datacenters, l'internet servant de secours.

```
policy
 lists
  site-list DATACENTERS
   site-id 1000-1001
  tloc-list TLOC-DC1-PREFERES
   tloc 10.255.0.1 color mpls encap ipsec preference 200
   tloc 10.255.0.1 color biz-internet encap ipsec preference 100
  !
 !
 control-policy PREFER-MPLS
  sequence 10
   match route
    site-list DATACENTERS
   !
   action accept
    set
     tloc-list TLOC-DC1-PREFERES
    !
   !
  !
  default-action accept
 !
!
```

### 10.4 Exemple 3 : service chaining (pare-feu central)

Objectif : le trafic du VPN invité (Wi-Fi clients) doit traverser le pare-feu du datacenter avant de sortir.

```
policy
 lists
  site-list MAGASINS
   site-id 2000-2999
  vpn-list VPN-INVITES
   vpn 40
  tloc-list TLOC-FW-DC1
   tloc 10.255.0.1 color mpls encap ipsec
  !
 !
 control-policy FW-INVITES
  sequence 10
   match route
    vpn-list VPN-INVITES
   !
   action accept
    set
     service FW vpn 40
     tloc-list TLOC-FW-DC1
    !
   !
  !
  default-action accept
 !
!
```

Remarque : le WAN Edge du datacenter doit annoncer une service route de type FW dans le VPN concerné.

### 10.5 Politiques de données (pour mémoire)

Ne pas confondre :

| Type | Agit sur | Où |
|---|---|---|
| Control policy | Annonces OMP (routes, TLOC) | vSmart |
| Data policy | Paquets de données (QoS, redirection, NAT, DIA) | Poussée vers les WAN Edge |
| Application-aware routing | Choix du tunnel selon SLA mesuré (perte, latence, gigue) | Poussée vers les WAN Edge |

---

## 11. Cas d'étude fil rouge : le réseau d'une enseigne de distribution

> Rappel : scénario fictif à vocation pédagogique.

### 11.1 Contexte

Une grande enseigne de distribution exploite :

- 2 datacenters (Paris et Lyon) hébergeant les applications centrales : encaissement, gestion de stocks, ERP, supply chain.
- 1 siège social.
- Environ 1000 magasins de tailles variées : hypermarchés, supermarchés, magasins de proximité.
- Des entrepôts logistiques.
- Des applications SaaS (messagerie, collaboration, relation client) et des services cloud.

Problèmes initiaux :

- Un réseau MPLS coûteux, avec un seul lien par petit magasin.
- Une panne du lien d'un magasin de proximité provoque l'arrêt des caisses connectées.
- Les flux SaaS passent par le datacenter, ce qui crée de la latence.
- Les évolutions de configuration prennent des semaines.

### 11.2 Objectifs du projet

- Résilience des caisses : jamais de rupture d'encaissement.
- Conformité PCI DSS : isolation des flux de paiement.
- Réduction du coût du WAN.
- Accès direct à l'internet pour le SaaS (Direct Internet Access).
- Déploiement rapide des nouveaux magasins (ZTP).

### 11.3 Plan d'identification

| Élément | Valeur |
|---|---|
| Siège | site-id 100 |
| Datacenter Paris | site-id 1000, system-ip 10.255.0.1 |
| Datacenter Lyon | site-id 1001, system-ip 10.255.0.2 |
| Hypermarchés | site-id 2000-2199 |
| Supermarchés | site-id 2200-2599 |
| Proximité | site-id 2600-2999 |
| Entrepôts | site-id 3000-3099 |
| vSmart | 10.255.255.1 et 10.255.255.2 |
| vBond | adresse publique fixe |

### 11.4 Plan de segmentation (VPN)

| VPN | Usage | Remarques |
|---|---|---|
| 10 | Caisses et terminaux de paiement | Périmètre PCI DSS, hub and spoke strict |
| 20 | Back office, ERP, gestion de stock | Accès aux datacenters |
| 30 | Vidéosurveillance | Flux vers datacenters, QoS dédiée |
| 40 | Wi-Fi invités | Sortie internet locale ou via pare-feu central |
| 50 | IoT : balances, étiquettes électroniques, chaîne du froid | Isolé, flux limités |
| 0 | Transport | Interfaces WAN |
| 512 | Gestion hors bande | Administration |

### 11.5 Transports

| Type de site | Liens | Colors |
|---|---|---|
| Hypermarché | MPLS + 2 internet fibre | mpls, biz-internet, public-internet |
| Supermarché | MPLS + internet | mpls, biz-internet |
| Proximité | Internet ADSL/fibre + 4G/5G | biz-internet, lte |
| Datacenters | 2 MPLS + 2 internet | mpls, biz-internet |

Un magasin de proximité n'a plus de lien MPLS : son overlay SD-WAN utilise l'internet en nominal et la 4G/5G en secours. Le chiffrement IPsec protège les flux.

### 11.6 Topologie logique voulue

- VPN 10 (caisses) : hub and spoke strict vers les datacenters, pas de flux magasin vers magasin.
- VPN 20 (back office) : hub and spoke, avec possibilité de flux inter-sites pour certains entrepôts.
- VPN 40 (invités) : sortie internet locale, filtrée par le pare-feu du magasin ou service chaining vers un pare-feu central pour les petites surfaces.
- Contrôleurs : 2 vSmart répartis sur les deux datacenters, 2 vBond, 1 cluster vManage.

### 11.7 Le rôle d'OMP dans ce scénario

Voici ce qui se passe lorsqu'un nouveau magasin de proximité (site-id 2750) ouvre :

1. Le routeur est branché au LAN et à l'internet. Il démarre en ZTP et contacte vBond.
2. Après authentification, il reçoit l'adresse des vSmart et monte deux connexions de contrôle DTLS (une par vSmart).
3. Il ouvre deux sessions OMP.
4. Il annonce par OMP :
   - Ses préfixes LAN pour les VPN 10, 20, 30, 40, 50 (ex. 10.20.75.0/24 pour la caisse).
   - Ses TLOC : (10.255.27.50, biz-internet, ipsec) et (10.255.27.50, lte, ipsec).
5. Le vSmart applique la politique HUB-AND-SPOKE : il ne transmet aux autres magasins ni ses routes ni ses TLOC, mais le transmet aux datacenters.
6. En retour, le magasin reçoit uniquement les routes et TLOC des datacenters.
7. Il monte des tunnels IPsec vers les TLOC des datacenters (4 tunnels typiques : 2 couleurs côté magasin, 2 datacenters) et lance BFD.
8. Les caisses atteignent les serveurs d'encaissement via l'overlay, chiffrées, avec bascule automatique sur la 4G si le lien internet principal tombe.

### 11.8 Scénarios de panne

| Panne | Comportement |
|---|---|
| Lien internet principal du magasin | BFD détecte la perte (de l'ordre de la seconde à quelques secondes). Le trafic bascule sur le TLOC lte. OMP annonce l'état du TLOC si nécessaire |
| Datacenter Paris entier | Les routes annoncées par Lyon restent valides. Le trafic bascule vers Lyon (même préfixe annoncé par les deux datacenters avec préférences différentes) |
| Un vSmart | Les WAN Edge ont une autre session OMP avec l'autre vSmart, aucune interruption |
| Les deux vSmart | Graceful restart : routes et clés conservées (jusqu'à 12 h par défaut), le trafic existant continue |
| vManage | Pas d'impact sur le plan de données ni sur OMP. Perte de supervision et de modification de configuration |

### 11.9 Application des préférences entre datacenters

Pour que Paris soit actif et Lyon en secours pour les applications d'encaissement :

- Les WAN Edge de Paris annoncent les préfixes applicatifs avec une préférence OMP élevée (par exemple 200).
- Ceux de Lyon annoncent les mêmes préfixes avec une préférence plus basse (par exemple 100).
- Les magasins choisissent Paris tant qu'il est disponible, sinon Lyon.

Configuration côté Paris, par exemple avec une politique de contrôle `in` sur vSmart :

```
control-policy PREFERENCE-DC
 sequence 10
  match route
   site-list DC-PARIS
  !
  action accept
   set
    preference 200
   !
  !
 !
 sequence 20
  match route
   site-list DC-LYON
  !
  action accept
   set
    preference 100
   !
  !
 !
 default-action accept
!
apply-policy
 site-list DC-PARIS
  control-policy PREFERENCE-DC in
 !
 site-list DC-LYON
  control-policy PREFERENCE-DC in
 !
!
```

### 11.10 Dimensionnement et modèle de session

Estimation pédagogique : 1000 magasins × 2 sessions OMP = 2000 sessions OMP à raison de 1 par vSmart. Chaque vSmart doit pouvoir supporter l'ensemble des sessions (il faut vérifier les limites de la plateforme choisie) et chaque vSmart doit pouvoir porter seul la charge si l'autre tombe. Dans une grande enseigne, on utilise souvent plus de deux vSmart, répartis par régions.

---

## 12. Configuration d'OMP pas à pas

### 12.1 Configuration de base sur un WAN Edge (Cisco IOS XE SD-WAN, cEdge)

```
system
 system-ip             10.255.27.50
 site-id               2750
 organization-name     "ENSEIGNE-DEMO"
 sp-organization-name  "ENSEIGNE-DEMO"
 vbond vbond.enseigne-demo.example
!
sdwan
 omp
  no shutdown
  graceful-restart
  send-path-limit 4
  ecmp-limit 4
  timers
   holdtime 60
   advertisement-interval 1
   graceful-restart-timer 43200
  exit
  address-family ipv4
   advertise connected
   advertise static
   advertise ospf external
  exit
 !
!
```

Les instructions exactes varient selon la version. Dans une exploitation réelle, ces paramètres sont généralement définis par un feature template OMP dans vManage et non à la main.

### 12.2 Configuration équivalente sur vEdge (Viptela OS)

```
omp
 no shutdown
 graceful-restart
 send-path-limit 4
 ecmp-limit 4
 timers
  holdtime 60
  advertisement-interval 1
  graceful-restart-timer 43200
 !
 advertise connected
 advertise static
 advertise ospf external
!
```

### 12.3 Interface de transport (VPN 0) avec couleur

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
 interface cellular0
  tunnel-interface
   encapsulation ipsec
   color lte restrict
  !
  no shutdown
 !
 ip route 0.0.0.0/0 203.0.113.9
!
```

La couleur `lte restrict` empêche le magasin de monter des tunnels vers des TLOC d'une autre couleur que lte, ce qui évite d'utiliser le lien cellulaire (coûteux) en nominal tant qu'une politique n'y autorise pas explicitement l'usage.

### 12.4 VPN de service et routage LAN

```
vpn 10
 interface ge0/2
  ip address 10.20.75.1/24
  no shutdown
 !
 router
  ospf
   router-id 10.255.27.50
   area 0
    interface ge0/2
    exit
   exit
  !
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

La route `ip route 0.0.0.0/0 vpn 0` du VPN 40 permet la sortie internet locale (Direct Internet Access) pour les invités.

### 12.5 Configuration de redistribution OMP vers OSPF (côté datacenter)

Pour que le LAN du datacenter apprenne les préfixes des magasins :

```
vpn 10
 router
  ospf
   redistribute omp
   area 0
    interface ge0/2
    exit
   exit
  !
 !
!
```

---

## 13. Vérification et dépannage

### 13.1 Commandes principales (cEdge, IOS XE SD-WAN)

| Commande | Utilité |
|---|---|
| `show sdwan control connections` | État des connexions de contrôle vers vSmart, vBond, vManage |
| `show sdwan omp summary` | Résumé : sessions, routes reçues, installées, envoyées |
| `show sdwan omp peers` | État des sessions OMP |
| `show sdwan omp routes` | Routes OMP reçues |
| `show sdwan omp routes vpn 10` | Routes du VPN 10 |
| `show sdwan omp tlocs` | TLOC reçus et annoncés |
| `show sdwan omp services` | Service routes |
| `show sdwan bfd sessions` | Sessions BFD vers les autres TLOC |
| `show sdwan control local-properties` | Propriétés locales (certificat, system-ip, site-id, couleur) |
| `show sdwan policy from-vsmart` | Politique reçue du vSmart |
| `show ip route vrf 10` | Table de routage du VRF (cEdge) |

Sur vEdge, les commandes sont équivalentes sans le mot `sdwan` : `show omp peers`, `show omp routes`, `show control connections`, `show bfd sessions`.

### 13.2 Exemple de sortie : sessions OMP (illustratif)

```
Magasin-2750# show sdwan omp peers

R -> routes received
I -> routes installed
S -> routes sent

                                         DOMAIN   OVERLAY   SITE
PEER            TYPE   ID                ID       ID        ID     STATE   UPTIME     R/I/S
-----------------------------------------------------------------------------------------------
10.255.255.1    vsmart 1                 1        1         5000   up      12:04:33   220/220/18
10.255.255.2    vsmart 1                 1        1         5001   up      12:04:31   220/220/18
```

Lecture : le magasin reçoit 220 routes de chaque vSmart, en installe 220 et en envoie 18 (ses préfixes et services locaux).

### 13.3 Exemple de sortie : routes OMP (illustratif)

```
Magasin-2750# show sdwan omp routes vpn 10

                                           PATH                      ATTACH
VPN  PREFIX           FROM PEER       ID   LABEL  STATUS   TYPE      TLOC IP        COLOR         ENCAP  PREFERENCE
------------------------------------------------------------------------------------------------------------------
10   10.50.0.0/16     10.255.255.1    1    1003   C,I,R    installed 10.255.0.1     mpls          ipsec  200
10   10.50.0.0/16     10.255.255.1    2    1003   C,I,R    installed 10.255.0.1     biz-internet  ipsec  200
10   10.50.0.0/16     10.255.255.1    3    1003   R        installed 10.255.0.2     mpls          ipsec  100
10   10.50.0.0/16     10.255.255.2    1    1003   C,I,R    installed 10.255.0.1     mpls          ipsec  200
```

Lecture :

- Le préfixe 10.50.0.0/16 (par exemple le réseau applicatif de l'encaissement) est joignable via Paris (10.255.0.1) avec préférence 200.
- Lyon (10.255.0.2) est disponible en secours avec préférence 100 (statut R : résolu mais non retenu).
- Plusieurs chemins C,I,R : ECMP entre les couleurs de Paris.

### 13.4 Exemple de sortie : TLOC (illustratif)

```
Magasin-2750# show sdwan omp tlocs

tloc entries for 10.255.0.1
                 mpls
                 ipsec
            ADDRESS
 TLOC IP        COLOR  ENCAP  FAMILY  PUBLIC IP    PORT  PRIVATE IP   PORT  BFD STATUS  STATUS
 10.255.0.1     mpls   ipsec  ipv4    198.51.100.1 12346 198.51.100.1 12346 up         C,I,R
```

### 13.5 Méthodologie de dépannage

Suivre l'ordre des couches :

1. Connexion de contrôle : `show sdwan control connections`. Si la session est down, vérifier le certificat, l'horloge (NTP), le site-id, l'organization-name, l'accès UDP 12346, la couleur, le DNS de vBond.
2. Session OMP : `show sdwan omp peers`. Une session up avec 0 routes reçues suggère une politique `out` trop restrictive.
3. Routes reçues : `show sdwan omp routes`. Si absentes, la route n'est pas annoncée (pas de `advertise` sur l'émetteur) ou la politique la filtre.
4. Statut de la route : si le statut est `Inv` ou `U`, le TLOC n'est pas résolu (tunnel absent, BFD down, mauvaise couleur, restrict).
5. TLOC : `show sdwan omp tlocs`, puis `show sdwan bfd sessions`.
6. Table de routage : `show ip route vrf <n>`. Vérifier qu'une route OMP est installée.
7. Plan de données : traceroute, `show sdwan tunnel statistics`, `show sdwan app-route stats`.

### 13.6 Causes fréquentes d'incident

| Symptôme | Cause probable |
|---|---|
| Connexion de contrôle en `connect` ou `challenge` | Certificat invalide ou horloge fausse |
| Sessions OMP up mais aucune route | Absence de `advertise`, politique de contrôle trop restrictive |
| Route en statut `Inv` ou `U` | TLOC non résolu, tunnel IPsec ou BFD inactif |
| Aucun tunnel entre deux sites | Couleurs incompatibles, `restrict`, NAT symétrique, ports bloqués |
| Perte de joignabilité après changement de politique | Mauvais sens `in` ou `out`, mauvaise site-list |
| Routes qui oscillent | Redistribution mutuelle OMP et IGP sans filtrage (boucles) |

### 13.7 Prévention des boucles de redistribution

Dans un site double routeur, la redistribution OMP vers OSPF puis OSPF vers OMP peut créer des boucles. Les protections :

- Utiliser des tags de route (`tag`) lors de la redistribution et les filtrer.
- Limiter les routes injectées avec des route-policies.
- Utiliser des attributs BGP (AS path, communities) lorsque BGP est employé.
- Appliquer les mécanismes de détection de boucle d'OMP (statut `L`).

---

## 14. Sécurité et résilience du plan de contrôle

### 14.1 Authentification et chiffrement

- Authentification mutuelle des équipements par certificats (PKI propre à l'entreprise ou PKI Cisco).
- Contrôle : DTLS (UDP) ou TLS (TCP) entre WAN Edge et contrôleurs.
- Données : IPsec (AES-256 en général) avec rotation périodique des clés.
- Liste blanche des équipements autorisés dans vManage (chaque numéro de série est déclaré).

### 14.2 Segmentation

Le label de VPN dans l'encapsulation IPsec garantit l'isolation des VPN de bout en bout. Les flux de paiement (VPN 10) ne peuvent pas atteindre le VPN 40 (invités) sans passer par une politique explicite.

### 14.3 Redondance

- Au moins deux vSmart, deux vBond, et un cluster vManage de trois noeuds en production importante.
- Répartition géographique des contrôleurs.
- Deux liens WAN minimum sur les sites critiques.

### 14.4 Protection du plan de contrôle

- Restreindre par `allow-service` les services autorisés sur chaque interface de transport.
- Utiliser des listes de contrôle d'accès sur les interfaces WAN.
- Superviser les alarmes de sessions de contrôle (down, flaps).
- Sauvegarder régulièrement la configuration des contrôleurs.

---

## 15. Bonnes pratiques de conception

1. Normaliser les identifiants : plan de site-id et system-ip cohérent et documenté.
2. Utiliser des templates dans vManage, jamais de configuration manuelle durable.
3. Définir la topologie cible avant d'écrire les politiques (hub and spoke, partial mesh, full mesh).
4. Limiter le nombre de chemins ECMP et de chemins annoncés à ce qui est nécessaire (taille des tables et charge des vSmart).
5. Agréger les préfixes côté datacenter pour réduire les tables des petits sites.
6. Séparer les rôles : un routeur de bordure pour les hubs, des routeurs distincts pour le service chaining.
7. Tester les politiques de contrôle en préproduction ou sur un site pilote.
8. Planifier la redondance des contrôleurs avant la montée en charge.
9. Documenter les couleurs et le comportement `restrict`.
10. Superviser les KPI : latence, perte, gigue par tunnel, disponibilité des sessions OMP et BFD.

---

## 16. Travaux pratiques

### TP 1 : Observer OMP dans un laboratoire

Matériel : Cisco CML, EVE-NG ou une version d'essai avec contrôleurs virtuels (vManage, vBond, vSmart) et deux ou trois vEdge/cEdge virtuels.

1. Déployer les contrôleurs et enrôler les équipements.
2. Configurer deux sites (site-id 101 et 102) avec un VPN de service 10.
3. Vérifier `show omp peers`, `show omp routes`, `show omp tlocs`.
4. Désactiver `advertise connected` sur un site et observer la disparition des routes chez l'autre.

### TP 2 : Hub and spoke

1. Créer deux spokes et un hub.
2. Appliquer la politique HUB-AND-SPOKE de la section 10.2.
3. Vérifier qu'aucun tunnel direct spoke vers spoke n'est présent (`show bfd sessions`).
4. Annoncer une route par défaut depuis le hub et valider la joignabilité entre spokes.

### TP 3 : Préférence entre deux datacenters

1. Configurer deux hubs annonçant le même préfixe.
2. Appliquer la politique de préférence (section 11.9).
3. Couper le hub principal et mesurer le temps de bascule.

### TP 4 : Graceful restart

1. Arrêter tous les vSmart du laboratoire.
2. Vérifier que le trafic continue (ping continu entre deux sites).
3. Observer le statut `S` (stale) dans `show omp routes`.
4. Redémarrer les vSmart et observer le retour à la normale.

### TP 5 : Scénario magasin de proximité

1. Simuler un site avec deux transports : `biz-internet` et `lte restrict`.
2. Couper le lien internet.
3. Observer la bascule et l'état des TLOC.
4. Mesurer l'impact sur un flux de test représentant une caisse (ping, iperf).

---

## 17. Questionnaire d'auto-évaluation

**Q1. Entre quels équipements s'établissent les sessions OMP ?**
Réponse : entre chaque WAN Edge et chaque vSmart. Il n'y a pas de session OMP directe entre WAN Edge.

**Q2. Que contient un TLOC ?**
Réponse : le triplet System IP, color, encapsulation, avec ses adresses privée et publique, ports, préférence et autres attributs.

**Q3. Quelle est la topologie par défaut de l'overlay sans politique ?**
Réponse : un full mesh.

**Q4. Que fait le graceful restart d'OMP ?**
Réponse : il conserve les routes et clés d'un pair injoignable pendant une durée donnée (43200 s par défaut), de sorte que le trafic ne s'arrête pas pendant une panne du plan de contrôle.

**Q5. Quel est l'effet de `color lte restrict` ?**
Réponse : le TLOC ne montera des tunnels qu'avec des TLOC de même couleur lte.

**Q6. Quelle différence entre une control policy et une data policy ?**
Réponse : la première agit sur les annonces OMP (routes, TLOC) et s'applique sur vSmart. La seconde agit sur les paquets de données au niveau des WAN Edge.

**Q7. Pourquoi une route OMP peut-elle être installée avec le statut U ou Inv ?**
Réponse : son TLOC next hop n'est pas résolu (tunnel absent, BFD down, incompatibilité de couleur ou de restriction).

**Q8. Dans le scénario de l'enseigne, comment éviter que deux magasins communiquent directement ?**
Réponse : appliquer une control policy hub and spoke en sens `out` aux magasins afin qu'ils ne reçoivent que les TLOC et routes des datacenters.

**Q9. Comment faire en sorte que Paris soit préféré à Lyon ?**
Réponse : donner une préférence OMP plus élevée aux routes annoncées par Paris, par exemple 200 contre 100.

**Q10. Que se passe-t-il si vManage tombe ?**
Réponse : le plan de données et OMP continuent de fonctionner. Seules la supervision et les modifications de configuration sont affectées.

---

## 18. Glossaire

| Terme | Définition |
|---|---|
| BFD | Bidirectional Forwarding Detection, détection rapide de panne et mesure de qualité des tunnels |
| Color | Étiquette qualifiant un transport |
| DIA | Direct Internet Access, sortie internet locale |
| DTLS | TLS sur UDP, utilisé pour les connexions de contrôle |
| ECMP | Equal Cost Multi-Path, répartition sur plusieurs chemins de même coût |
| OMP | Overlay Management Protocol |
| Overlay | Réseau virtuel construit sur les transports |
| Underlay | Transports physiques (MPLS, internet, 4G/5G) |
| PKI | Infrastructure à clés publiques |
| Service chaining | Insertion d'un service (pare-feu, IDS) dans le chemin d'un flux |
| Site ID | Identifiant du site |
| System IP | Identifiant unique d'un équipement dans l'overlay |
| TLOC | Transport Locator, point d'attache au transport, next hop de l'overlay |
| vBond | Orchestrateur |
| vEdge, cEdge | Routeurs WAN Edge |
| vManage | Plateforme de gestion |
| vSmart | Contrôleur OMP |
| VPN | Segment logique (VRF) de l'overlay |
| ZTP | Zero Touch Provisioning |

---

## 19. Références

- Documentation officielle Cisco SD-WAN (guides de configuration Routing, Policies, Systems and Interfaces).
- Cisco SD-WAN Design Guide (Cisco Validated Design).
- Cisco Press : ouvrages sur le déploiement Cisco SD-WAN (cursus CCNP Enterprise ENCOR et ENSDWI).
- Certifications associées : CCNP Enterprise avec examen spécialisé 300-415 (ENSDWI).
- RFC utiles pour la culture générale : RFC 4271 (BGP-4), RFC 4301 (architecture IPsec), RFC 6347 (DTLS), RFC 5880 (BFD).

Les valeurs par défaut, numéros de ports et syntaxes de commandes peuvent évoluer selon les versions logicielles : validez toujours dans la documentation correspondant à votre version avant toute mise en production.
