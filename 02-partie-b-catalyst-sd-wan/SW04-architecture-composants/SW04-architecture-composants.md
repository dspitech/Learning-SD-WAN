# SW04. Catalyst SD-WAN : architecture et composants

| | |
|---|---|
| **Partie** | B. Cisco Catalyst SD-WAN |
| **Durée indicative** | 1 à 1,5 semaine (10 h) |
| **Prérequis** | SW01 à SW03 ; BGP et OSPF ; IPsec, DTLS/TLS, PKI (notions) |
| **Niveau** | Intermédiaire à avancé |
| **Lien examen** | ENSDWI 300-415 : domaine « Architecture » ; ENCOR |
| **Lab associé** | T2 (début) |

> Module précédent : [SW03](../../01-partie-a-fondamentaux/SW03-fonctions-cles/SW03-fonctions-cles.md) | Module suivant : [SW05](../SW05-deploiement-controleurs-equipements/SW05-deploiement-controleurs-equipements.md)

> **Avertissement sur la syntaxe.** Les noms de produits ont changé (SD-WAN Manager = vManage, SD-WAN Controller = vSmart, SD-WAN Validator = vBond) et les commandes évoluent selon la version. Les exemples ci-dessous visent l'**IOS XE SD-WAN** (WAN Edge Catalyst 8000). **Vérifiez toujours** avec le point d'interrogation (`?`) et la documentation de **votre** version avant de reproduire une commande en production.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Vue d'ensemble de l'architecture](#2-vue-densemble-de-larchitecture)
3. [Les composants](#3-les-composants)
4. [Les identifiants](#4-les-identifiants)
5. [Les connexions de contrôle](#5-les-connexions-de-contrôle)
6. [TLOC et couleurs](#6-tloc-et-couleurs)
7. [OMP : Overlay Management Protocol](#7-omp--overlay-management-protocol)
8. [Tunnels de données et BFD](#8-tunnels-de-données-et-bfd)
9. [VPN et segmentation](#9-vpn-et-segmentation)
10. [Séquence complète : un site rejoint la fabrique](#10-séquence-complète--un-site-rejoint-la-fabrique)
11. [Modèles de déploiement et haute disponibilité](#11-modèles-de-déploiement-et-haute-disponibilité)
12. [Multi-régions, multi-tenance et affinité](#12-multi-régions-multi-tenance-et-affinité)
13. [Lire les sorties de commandes](#13-lire-les-sorties-de-commandes)
14. [Lab guidé : schéma annoté et exploration du bac à sable](#14-lab-guidé--schéma-annoté-et-exploration-du-bac-à-sable)
15. [Exercices](#15-exercices)
16. [Quiz](#16-quiz)
17. [Aide-mémoire](#17-aide-mémoire)
18. [Pour aller plus loin](#18-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Décrire le rôle de chaque composant et son ancien nom (vBond, vManage, vSmart, vEdge).
- Expliquer OMP, les TLOC, les couleurs et les types de routes OMP.
- Détailler les connexions de contrôle (protocoles, ports) et les tunnels de données.
- Expliquer le VPN 0, le VPN 512 et les VPN de service.
- Dérouler, étape par étape, l'arrivée d'un site dans la fabrique.
- Lire les sorties de `show sdwan control connections`, `omp`, `bfd`.

---

## 2. Vue d'ensemble de l'architecture

```text
                      +------------------------------+
                      |   SD-WAN Manager (vManage)   |   Plan de GESTION  (UI, API, templates, supervision)
                      +--------------+---------------+
                                     |  NETCONF/TLS ou DTLS
        +----------------------------+----------------------------+
        |                                                         |
+-------+-----------------+                        +--------------+----------+
| SD-WAN Validator (vBond)|                        | SD-WAN Controller(vSmart)|
| Plan d'ORCHESTRATION    |                        | Plan de CONTROLE (OMP)   |
+-------+-----------------+                        +--------------+----------+
        |  DTLS (authentification, découverte)                   |  DTLS/TLS + OMP
        |                                                         |
        +--------------------------+------------------------------+
                                   |
         +-------------------------+-------------------------+
         |                         |                         |
   +-----+------+            +-----+------+            +-----+------+
   | WAN Edge   |============| WAN Edge   |============| WAN Edge   |   Plan de DONNEES
   | (Site A)   |  IPsec+BFD | (Site B)   |  IPsec+BFD | (Datacenter)|  (tunnels entre TLOC)
   +-----+------+            +-----+------+            +-----+------+
         |  MPLS / Internet / LTE (UNDERLAY)
```

| Ancien nom | Nom actuel | Plan |
|---|---|---|
| vBond | **SD-WAN Validator** | Orchestration |
| vManage | **SD-WAN Manager** | Gestion |
| vSmart | **SD-WAN Controller** | Contrôle |
| vEdge, cEdge | **WAN Edge** (Catalyst 8000, ISR, ASR en mode contrôleur) | Données |

> Les anciens vEdge (Viptela) sont en fin de vie. Les routeurs IOS XE en mode « contrôleur » (SD-WAN) sont parfois encore appelés **cEdge**.

---

## 3. Les composants

### 3.1 SD-WAN Validator (vBond)

| Fonction | Détail |
|---|---|
| **Premier contact** | Tout équipement connaît l'adresse du Validator (configurée ou issue de PnP/ZTP) |
| **Authentification** | Vérifie que l'équipement est **autorisé** (liste des numéros de série/châssis) via les certificats |
| **Annuaire** | Informe l'équipement des adresses du Manager et des Controllers |
| **Découverte NAT** | Aide à déterminer l'adresse IP publique et le type de NAT de l'équipement (rôle de type STUN) |
| **Exigence réseau** | Doit être **joignable depuis Internet** (adresse publique ou NAT statique), car tous les équipements le contactent |

Le Validator ne transporte **jamais** de trafic utilisateur ni de routes ; une fois l'orchestration terminée, la connexion de l'équipement vers lui est fermée (elle est rétablie périodiquement).

### 3.2 SD-WAN Manager (vManage)

| Fonction | Détail |
|---|---|
| **Interface** | Interface web (HTTPS), API REST (`/dataservice/...`) |
| **Configuration** | Modèles de fonctionnalité, modèles d'équipement, **groupes de configuration**, politiques |
| **Supervision** | Tableaux de bord, alarmes, événements, statistiques |
| **Administration** | Utilisateurs, RBAC, certificats, mises à jour logicielles, sauvegardes |
| **Clustering** | Plusieurs nœuds en cluster (échelle et haute disponibilité) |
| **Communications** | Vers les équipements pour pousser la configuration (NETCONF sur la connexion de contrôle), vers les Controllers (pousser les politiques centralisées) |

### 3.3 SD-WAN Controller (vSmart)

| Fonction | Détail |
|---|---|
| **Cerveau du plan de contrôle** | Reçoit les routes de tous les sites par **OMP** et les redistribue |
| **Distribution des clés** | Distribue les clés de chiffrement du plan de données (les équipements ne négocient pas d'IKE entre eux) |
| **Politiques** | Exécute les **politiques de contrôle** (topologie, filtrage) et distribue les **politiques de données** et de **routage applicatif** |
| **Réflecteur de routes** | Fonctionne comme un « route-reflector » : les sites n'ont pas de sessions OMP entre eux |

### 3.4 WAN Edge

| Fonction | Détail |
|---|---|
| **Plan de données** | Termine les tunnels IPsec, transfère les paquets |
| **Connexions de contrôle** | Vers Validator (temporairement), Manager, Controllers |
| **Routage LAN** | OSPF, BGP, EIGRP, statique côté service |
| **Fonctions locales** | QoS, ACL, NAT, pare-feu, IPS, filtrage URL (selon licence et ressources) |
| **Plates-formes** | Catalyst 8000 (8300, 8500 en matériel ; 8000V en virtuel), ISR 1000/4000 et ASR 1000 selon version |

---

## 4. Les identifiants

| Identifiant | Description | Exemple | Règle |
|---|---|---|---|
| **System IP** | Identifiant unique de l'équipement dans l'overlay (de forme IPv4, **pas nécessairement routée**) | `10.255.0.11` | Unique par équipement ; ne change pas |
| **Site ID** | Numéro du site (1 à 4294967295) | `100` | Même valeur pour tous les équipements d'un même site ; sert aux politiques |
| **Organization name** | Nom de l'organisation, partagé par tous les composants | `LAB-SDWAN` | Identique partout ; sensible à la casse |
| **Chassis ID / UUID** | Identifiant matériel (ou virtuel) | `C8K-...` | Utilisé dans la liste d'autorisation |
| **Serial number / token** | Numéro de série du certificat ou jeton à usage unique (équipements virtuels) | | Rôle d'authentification |
| **Overlay ID** | Identifiant de l'overlay (versions récentes) | `1` | |

Bonnes pratiques de conception :

```text
System IP  : 10.255.0.0/16 réservé (loopback logique), par exemple 10.255.<région>.<équipement>
Site ID    : 1-9 contrôleurs/DC ; 10-99 hubs ; 100-999 agences ; 1000+ sites éphémères
```

> **Piège** : deux équipements avec la **même system IP** ou le même identifiant de châssis provoquent des connexions de contrôle instables. Les identifiants dupliqués font partie des pannes classiques (SW10).

---

## 5. Les connexions de contrôle

### 5.1 Protocoles

| Connexion | Protocole | Port par défaut | Rôle |
|---|---|---|---|
| WAN Edge ↔ Validator | **DTLS** | UDP 12346 | Authentification, découverte |
| WAN Edge ↔ Controller | DTLS ou TLS | UDP 12346 ou TCP 23456 | OMP, clés, politiques |
| WAN Edge ↔ Manager | DTLS ou TLS | UDP 12346 ou TCP 23456 | Configuration, supervision |
| Controller ↔ Manager | DTLS ou TLS | idem | Politiques centralisées |
| Navigateur ↔ Manager | HTTPS | TCP 443 | Interface et API |
| NETCONF | SSH | TCP 830 | Poussée de configuration (porté par la connexion de contrôle) |

Le choix **DTLS** (UDP) ou **TLS** (TCP) se fait dans la configuration de sécurité de l'équipement (paramètre de protocole de contrôle, `security` > `control` > `protocol`, valeur par défaut DTLS ; syntaxe exacte à vérifier selon la version). TLS est utile si l'UDP est filtré par un pare-feu intermédiaire ; DTLS est plus léger.

### 5.2 Décalage de ports et ports de saut

- **Port offset** : chaque processus de contrôle sur un même équipement utilise un port décalé (12346, 12366, 12386, 12406, 12426) pour traverser des NAT en parallèle.
- **Port hopping** : l'équipement change de port source lors d'une reconnexion pour contourner d'éventuels blocages/NAT.

> **À ouvrir sur les pare-feu** (liste de départ, à valider avec le guide de ports de votre version) : UDP 12346 et décalages ; TCP 23456 et décalages si TLS ; TCP 443 vers le Manager ; UDP 123 (NTP) ; UDP/TCP 53 (DNS) ; UDP 4500 et 500 selon le chiffrement ; ICMP.

### 5.3 États d'une connexion de contrôle

| État | Signification |
|---|---|
| `connect` | Établissement en cours |
| `handshake` | Négociation DTLS/TLS |
| `challenge` | Échange de certificats |
| `up` | **Établie** |
| `tear_down` | Fermée |

---

## 6. TLOC et couleurs

### 6.1 TLOC

Un **TLOC** (Transport Locator) est un **point de terminaison** de tunnel dans l'overlay. Il est identifié par un triplet :

```text
TLOC = ( System-IP , Color , Encapsulation )
Exemple : (10.255.0.11 , biz-internet , ipsec)
```

Un équipement avec deux interfaces de transport possède **deux TLOC**. Les tunnels de données s'établissent **entre TLOC**.

### 6.2 Couleurs

La **couleur** est un **nom** attribué à un transport. C'est une étiquette sémantique (et non un VPN) qui sert à :

1. décider entre qui des tunnels sont établis ;
2. exprimer les politiques (« préférer mpls pour la voix »).

Couleurs disponibles : `default`, `mpls`, `metro-ethernet`, `biz-internet`, `public-internet`, `lte`, `3g`, `red`, `green`, `blue`, `gold`, `silver`, `bronze`, `custom1`, `custom2`, `custom3`, `private1` à `private6`.

| Famille | Couleurs | Adressage utilisé pour les tunnels |
|---|---|---|
| **Publiques** | `default`, `biz-internet`, `public-internet`, `lte`, `3g`, `red`, `green`, `blue`, `gold`, `silver`, `bronze`, `custom1-3` | IP **publique** (après NAT) |
| **Privées** | `mpls`, `metro-ethernet`, `private1` à `private6` | IP **privée** (de l'interface) |

### 6.3 Mot-clé `restrict`

```text
color biz-internet restrict
```

Avec `restrict`, l'équipement n'établit des tunnels **qu'avec des TLOC de la même couleur**. Sans `restrict`, il peut établir des tunnels vers des TLOC de couleurs différentes, à condition qu'elles soient joignables.

| Cas | Résultat |
|---|---|
| `mpls` (sans restrict) et `mpls` | Tunnel |
| `mpls` **restrict** et `biz-internet` | **Pas de tunnel** |
| `biz-internet` et `biz-internet` | Tunnel |
| `mpls` (private) et `biz-internet` (public) sans restrict | Tunnel possible si joignable (rare, car les réseaux sont distincts) |

> **Piège classique** : mettre `restrict` sur un transport MPLS d'un côté et oublier le même nommage de couleur de l'autre côté : plus aucun tunnel ne monte. Les couleurs doivent être **cohérentes dans toute la fabrique**.

---

## 7. OMP : Overlay Management Protocol

OMP est le protocole de contrôle de l'overlay. Il tourne **sur la connexion de contrôle** (DTLS/TLS) entre chaque WAN Edge et chaque Controller. Il fonctionne comme BGP avec un réflecteur de routes, mais avec des spécificités SD-WAN.

### 7.1 Types de routes OMP

| Type | Contenu | Annoncé par | Usage |
|---|---|---|---|
| **Routes OMP (vRoutes)** | Préfixes IP des VPN de service (attributs : originateur, TLOC, VPN, label) | WAN Edge → Controller → autres WAN Edge | Accessibilité des réseaux des sites |
| **Routes TLOC** | Points de terminaison de transport (IP de transport, couleur, encapsulation, préférence, clés de chiffrement) | WAN Edge | Construction des tunnels |
| **Routes de service** | Services disponibles localement (pare-feu, IPS, etc.) pour le chaînage de services | WAN Edge | Insertion de services |

### 7.2 Attributs importants

| Attribut | Rôle |
|---|---|
| **Originator** | System IP de l'équipement qui annonce |
| **TLOC** (next hop) | Où envoyer le trafic (system-ip, couleur, encap) |
| **Preference** (route) | Préférence de route OMP (plus élevée = préférée) |
| **TLOC preference** | Préférence entre TLOC d'un même équipement |
| **Tag** | Étiquette pour les politiques et la prévention de boucles |
| **Site ID** | Pour le filtrage par site |
| **VPN** | Segment concerné |
| **Path ID** | Distingue plusieurs chemins vers un même préfixe |
| **Label** | Identifie le VPN dans l'encapsulation (étiquette de service) |

### 7.3 Sélection de route (résumé)

Plusieurs chemins OMP vers un même préfixe sont possibles (un par TLOC). Critères principaux, dans l'ordre simplifié : route valide et TLOC joignable → plus faible distance administrative selon l'origine → plus haute préférence OMP → plus haute préférence de TLOC → origine → plus petit identifiant. Les chemins retenus sont limités par `ecmp-limit`, et le nombre de chemins annoncés par le Controller par `send-path-limit`.

### 7.4 Configuration OMP de référence

```text
sdwan
 omp
  no shutdown
  graceful-restart
  no as-dot-notation
  timers
   holdtime 60
   advertisement-interval 1
   graceful-restart-timer 43200
   eor-timer 300
  exit
  address-family ipv4
   advertise connected
   advertise static
   advertise ospf external
   advertise bgp
  exit
  ecmp-limit 4
  send-path-limit 4
```

| Paramètre | Valeur usuelle | Rôle |
|---|---|---|
| `graceful-restart-timer` | 43 200 s (12 h) | Durée de conservation des routes si le Controller devient injoignable |
| `holdtime` | 60 s | Délai de détection d'un pair OMP silencieux |
| `advertise connected/static/ospf/bgp` | selon besoin | Origine des routes injectées dans OMP |
| `send-path-limit` | 4 | Nombre de chemins envoyés par préfixe |
| `ecmp-limit` | 4 | Nombre de chemins à coût égal utilisés |

> **Intérêt du graceful restart** : si les Controllers deviennent injoignables, les WAN Edge **conservent** les routes et les tunnels existants pendant le délai configuré (jusqu'à 12 h). Le trafic continue mais **aucun changement** (nouveau site, nouvelle politique) n'est possible.

---

## 8. Tunnels de données et BFD

- Les tunnels de données sont des **tunnels IPsec** (ESP) **entre TLOC**, construits **directement** entre WAN Edge (jamais à travers les Controllers).
- **Mode full mesh par défaut** : si les politiques ne limitent rien, chaque TLOC compatible monte un tunnel vers chaque autre TLOC compatible.
- Chaque tunnel porte une session **BFD** (hello 1 s, multiplicateur 7 par défaut) qui assure la vivacité et fournit les mesures de perte, délai, gigue.
- Les clés symétriques d'un tunnel sont **générées localement** par chaque extrémité et **annoncées dans OMP** (route TLOC) ; il n'y a pas d'IKE entre les deux WAN Edge.

Estimation du nombre de tunnels (maillage complet) : N sites, T TLOC par site, tous compatibles : `tunnels = N(N−1)/2 × T²` si les couleurs ne sont pas restreintes ; `N(N−1)/2 × T` si `restrict` limite à des paires de même couleur.

---

## 9. VPN et segmentation

Dans Cisco Catalyst SD-WAN, un **VPN** est un identifiant numérique (0 à 65530) correspondant à une **VRF** dans l'IOS XE.

| VPN | Rôle | Particularités |
|---|---|---|
| **VPN 0** | **Transport** : interfaces vers MPLS/Internet/LTE ; tunnels et connexions de contrôle | Dans l'IOS XE, correspond à la table de routage **globale** |
| **VPN 512** | **Gestion** hors bande | VRF `Mgmt-intf` ; ne participe pas à l'overlay |
| **VPN 1 à 511, 513 à 65530** | **VPN de service** (utilisateurs, invités, IoT…) | VRF définies par l'administrateur ; routes échangées via OMP |

Exemple de VRF de service en IOS XE :

```text
vrf definition 10
 description CORP
 rd 1:10
 address-family ipv4
  route-target export 1:10
  route-target import 1:10
 exit-address-family
!
interface GigabitEthernet3
 description LAN-CORP
 vrf forwarding 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
```

> Ne pas confondre : **couleur** = transport (étiquette de tunnel) ; **VPN** = segment de routage. Un même tunnel (une paire de TLOC) transporte **plusieurs VPN**, distingués par une étiquette dans l'encapsulation.

---

## 10. Séquence complète : un site rejoint la fabrique

```text
 WAN Edge (neuf)                Validator              Manager             Controller
      |                             |                     |                    |
 1.   |-- DHCP, DNS, NTP ---------->|                     |                    |
 2.   |== DTLS (UDP 12346) =========>|                     |                    |
      |   certificats, vérif. liste  |                     |                    |
      |<= adresses Manager/Controller|                     |                    |
 3.   |  (connexion vers le Validator fermée)             |                    |
 4.   |============ DTLS/TLS ===============================>|                    |
      |<=== configuration (NETCONF) ==========================|                    |
 5.   |============ DTLS/TLS =====================================================>|
      |<=== OMP : routes, TLOC, clés, politiques =============================>|
 6.   |<===========================  tunnels IPsec + BFD entre WAN Edge  ===============>
 7.   |  Trafic utilisateur dans l'overlay
```

| Étape | Ce qui se passe | Si ça échoue |
|---|---|---|
| 1 | Connectivité de base, résolution DNS du Validator, heure correcte | Pas de connexion de contrôle vers le Validator |
| 2 | Authentification mutuelle par certificats ; liste d'autorisation consultée | Certificat invalide, équipement non listé |
| 3 | Découverte des autres contrôleurs | Aucun contrôleur connu |
| 4 | Connexion au Manager : récupération de la configuration | Équipement sans configuration |
| 5 | Connexion au Controller, session OMP | Pas de routes |
| 6 | Annonce des TLOC et des clés, montée des tunnels | Couleurs incompatibles, NAT symétrique, ports filtrés |
| 7 | Trafic | Politiques, routage, MTU |

---

## 11. Modèles de déploiement et haute disponibilité

### 11.1 Où héberger les contrôleurs

| Modèle | Description | Avantages | Inconvénients |
|---|---|---|---|
| **Hébergés dans le cloud par Cisco** | Cisco déploie et exploite Manager, Controllers, Validators | Rapidité, haute disponibilité intégrée, moins d'exploitation | Souveraineté des données, dépendance |
| **Auto-hébergés** (cloud public ou centre de données du client) | Le client installe et exploite | Maîtrise, souveraineté | Exploitation, haute disponibilité à concevoir |
| **Fournis par un opérateur ou intégrateur** | Service géré | Délégation | Dépendance et latitude réduite |

### 11.2 Haute disponibilité

| Composant | Mécanisme |
|---|---|
| **Validator** | Au moins 2, résolus par un nom DNS à plusieurs adresses (les équipements essaient chaque adresse) |
| **Controller** | Au moins 2, sur des sites/zones différents ; chaque WAN Edge se connecte à plusieurs Controllers (limite par `max-omp-sessions`) |
| **Manager** | Cluster de 3 nœuds (production) ou 1 nœud (laboratoire) ; sauvegardes et reprise après sinistre (DR) |
| **WAN Edge** | Deux équipements par site, VRRP côté LAN, tunnels sur les deux |

Services d'un cluster Manager : serveur d'application, base de statistiques, base de configuration, serveur de messagerie, serveur de coordination, répartis sur les nœuds.

---

## 12. Multi-régions, multi-tenance et affinité

| Concept | Description | Utilité |
|---|---|---|
| **Multi-Region Fabric (MRF)** | Découpage de la fabrique en **régions** (cœur = région 0, accès = régions 1 à 63), routeurs de **bordure** entre régions | Passage à l'échelle, trafic inter-régions via les bordures, politiques régionales |
| **Multi-tenance** | Plusieurs organisations (tenants) sur un même ensemble de contrôleurs | Fournisseurs de services |
| **Affinité de contrôleur** (controller affinity) | Groupe de Controllers préférés par site (par `controller-group-id`) | Répartir la charge, rapprocher géographiquement |
| **Hubs régionaux** | Concentrateurs par région | Réduit le nombre de tunnels et la latence |

---

## 13. Lire les sorties de commandes

### 13.1 Connexions de contrôle

```text
BR1-EDGE1# show sdwan control connections
PEER                    PEER     PEER             SITE       DOMAIN PEER             PRIV  PEER              PUB
TYPE      PROT SYSTEM IP         ID    ... PRIVATE IP    PORT   PUBLIC IP       PORT   LOCAL COLOR  PROXY STATE UPTIME
vsmart    dtls 10.255.255.2      1     1    203.0.113.20 12346 203.0.113.20     12346  biz-internet No    up    0:02:14:05
vmanage   dtls 10.255.255.1      1     0    203.0.113.30 12346 203.0.113.30     12346  biz-internet No    up    0:02:14:03
vbond     dtls -                 0     0    203.0.113.10 12346 203.0.113.10     12346  biz-internet -     up    0:02:14:09
```

Lecture : un équipement sain montre une connexion **up** vers au moins un `vsmart` et le `vmanage` (le `vbond` peut apparaître ou non selon le moment). `LOCAL COLOR` indique l'interface de transport utilisée.

### 13.2 Pairs OMP

```text
BR1-EDGE1# show sdwan omp peers
R -> routes received   I -> routes installed   S -> routes sent
                         DOMAIN OVERLAY SITE
PEER           TYPE   ID     ID      ID     STATE  UPTIME       R/I/S
10.255.255.2   vsmart 1      1       1      up     2:14:05:41   6/6/3
```

`R/I/S` : routes reçues, installées, envoyées. Si `R` vaut 0 alors que d'autres sites existent, il y a un problème de politique de contrôle ou de VPN.

### 13.3 TLOC

```text
BR1-EDGE1# show sdwan omp tlocs
tloc entries for 10.255.0.1
                      mpls
                      ipsec
            RECEIVED FROM:
            peer            10.255.255.2
            status          C,I,R
            loss-reason     not set
            lost-to-peer    not set
            lost-to-path-id not set
            Attributes:
              attribute-type    installed
              encap-key         not set
              private-ip        192.0.2.9
              public-ip         192.0.2.9
              ...
```

`status C,I,R` : **C**hosen, **I**nstalled, **R**esolved. Un TLOC non résolu (`Red`) signifie que le tunnel n'est pas utilisable.

### 13.4 Sessions BFD

```text
BR1-EDGE1# show sdwan bfd sessions
SYSTEM IP     SITE ID  STATE  SOURCE TLOC COLOR  REMOTE TLOC COLOR  SOURCE IP     DST PUBLIC IP    ENCAP  MULT  TX INTERVAL  UPTIME
10.255.0.1    10       up     biz-internet       biz-internet       198.51.100.2  203.0.113.9      ipsec  7     1000         2:10:12:05
10.255.0.1    10       up     mpls               mpls               192.0.2.2     192.0.2.9        ipsec  7     1000         2:10:12:05
```

Un nombre de sessions `up` attendu = nombre de paires de TLOC compatibles. Une session `down` signale un problème de transport, de NAT ou de couleur.

### 13.5 Routes OMP

```text
BR1-EDGE1# show sdwan omp routes vpn 10
PATH                      ATTRIBUTE
VPN    PREFIX              FROM PEER     ID   LABEL  STATUS  TYPE  TLOC IP      COLOR        ENCAP PREFERENCE
10     10.1.0.0/16         10.255.255.2  1    1002   C,I,R   installed 10.255.0.1 mpls         ipsec -
10     10.1.0.0/16         10.255.255.2  2    1002   C,I,R   installed 10.255.0.1 biz-internet ipsec -
```

Deux lignes pour un même préfixe = deux chemins (deux TLOC) disponibles vers le même équipement : base d'un partage de charge ou d'une bascule.

---

## 14. Lab guidé : schéma annoté et exploration du bac à sable

### 14.1 Partie 1 : schéma annoté (papier ou draw.io)

Dessinez, pour un site « Agence-Lyon » (site-id 100) avec deux transports (MPLS et Internet) :

1. les trois contrôleurs et leurs adresses ;
2. les connexions de contrôle (avec ports) ;
3. les TLOC de l'agence ;
4. les tunnels de données vers un hub (site 10) ;
5. les VPN 0, 512 et deux VPN de service.

Annotez les 7 étapes de la séquence d'arrivée (section 10).

### 14.2 Partie 2 : bac à sable Cisco DevNet

1. Créez un compte sur <https://developer.cisco.com> et réservez le bac à sable **Catalyst SD-WAN** (laboratoire « Multi-Region Fabric » ou « Cisco SD-WAN » selon l'offre disponible).
2. Connectez-vous à l'interface du Manager avec les identifiants fournis.
3. Parcourez et notez :
   - **Monitor > Overview** : nombre d'équipements, de tunnels, alarmes.
   - **Configuration > Devices** : liste des WAN Edge et des contrôleurs.
   - **Configuration > Certificates** : état des certificats.
   - **Configuration > Templates** (ou **Configuration Groups**) : modèles.
   - **Monitor > Devices > <équipement>** : onglets *Control Connections*, *Real Time*, *WAN > Tunnel*.
4. Dans **Tools > SSH Terminal** (ou via CLI), exécutez : `show sdwan control connections`, `show sdwan omp peers`, `show sdwan bfd sessions`, `show sdwan omp tlocs`.
5. Renseignez le tableau :

| Élément | Valeur observée |
|---|---|
| Nombre de WAN Edge | |
| Nombre de Controllers et de Validators | |
| Couleurs utilisées | |
| VPN de service présents | |
| Nombre de sessions BFD d'un équipement choisi | |
| Système IP et site ID de cet équipement | |

### 14.3 Corrigé de la partie 1 (extrait)

```text
Étape 2 : Agence-Lyon (DTLS UDP 12346) -> Validator 203.0.113.10
Étape 4 : Agence-Lyon -> Manager 203.0.113.30 (UDP 12346 ou TCP 23456)
Étape 5 : Agence-Lyon -> Controller 203.0.113.20 (OMP)
Étape 6 : TLOC (10.255.1.100, mpls, ipsec) <-> TLOC (10.255.0.1, mpls, ipsec)
          TLOC (10.255.1.100, biz-internet, ipsec) <-> TLOC (10.255.0.1, biz-internet, ipsec)
```

---

## 15. Exercices

1. Un site possède trois interfaces de transport (mpls, biz-internet, lte) et le hub deux (mpls, biz-internet). Combien de tunnels directs entre le site et le hub avec `restrict` sur toutes les couleurs ?
2. Citez deux raisons pour lesquelles deux équipements peuvent ne pas établir de tunnel alors que leurs connexions de contrôle sont `up`.
3. Quelle commande sur un WAN Edge permet de voir si les routes d'autres sites ont bien été reçues ?
4. Expliquez la différence entre un TLOC et une route OMP.
5. Pourquoi le Validator doit-il être joignable depuis Internet ?

<details>
<summary>Corrigés</summary>

1. Avec `restrict`, seuls les TLOC de même couleur se parlent : mpls↔mpls et biz-internet↔biz-internet = **2 tunnels** (lte n'a pas de vis-à-vis).
2. Couleurs incompatibles (`restrict`), NAT symétrique, ports bloqués (UDP 12346/4500), MTU, politique de contrôle qui filtre les TLOC.
3. `show sdwan omp routes` et `show sdwan omp peers` (colonne R).
4. Un TLOC est un **point de terminaison de transport** (où établir un tunnel) ; une route OMP est un **préfixe** d'un VPN de service associé à un TLOC (next hop).
5. Parce que chaque équipement, quel que soit son site ou son NAT, doit pouvoir le contacter en premier (orchestration).

</details>

---

## 16. Quiz

1. Que signifie OMP ? À quoi sert-il ?
2. Quel est le port UDP par défaut des connexions de contrôle DTLS ?
   a) 500 ; b) 4500 ; c) 12346 ; d) 23456.
3. VPN 512 est :
   a) Le transport ; b) La gestion hors bande ; c) Un VPN de service ; d) Un VPN de secours.
4. Le Controller transporte-t-il le trafic utilisateur ? (oui / non)
5. Que signifie le statut `C,I,R` d'un TLOC ?
6. Que provoque `restrict` ?
7. Quel composant exécute les politiques de contrôle ?

<details>
<summary>Réponses</summary>

1. Overlay Management Protocol : échange routes, TLOC, clés, politiques. 2. c. 3. b. 4. Non. 5. Chosen, Installed, Resolved. 6. Limite les tunnels aux TLOC de la même couleur. 7. Le SD-WAN Controller (vSmart).

</details>

---

## 17. Aide-mémoire

```text
Plans :  Validator = orchestration | Manager = gestion | Controller = contrôle | WAN Edge = données
Ports :  DTLS UDP 12346 (+ décalages) | TLS TCP 23456 | HTTPS 443 | NETCONF 830
ID    :  system-ip (unique) | site-id (par site) | organization-name (commun) | chassis/serial (autorisation)
TLOC  :  (system-ip, color, encap)          VPN 0 = transport | VPN 512 = gestion | autres = service
OMP   :  routes OMP + TLOC + services       graceful-restart 12 h par défaut
Commandes : show sdwan control connections | omp peers | omp routes | omp tlocs | bfd sessions
```

## 18. Pour aller plus loin

- Documentation Catalyst SD-WAN : <https://www.cisco.com/c/en/us/solutions/enterprise-networks/sd-wan/index.html>
- Bacs à sable DevNet : <https://developer.cisco.com>
- Cisco Validated Designs SD-WAN (guides de conception) : à rechercher sur cisco.com avec les mots « SD-WAN design guide ».

---

## Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **Fabrique (fabric)** | L'ensemble des contrôleurs et des WAN Edge d'une organisation |
| **Overlay Management Protocol (OMP)** | Protocole de contrôle de l'overlay (routes, TLOC, clés, politiques) |
| **TLOC** | Point de terminaison de transport : (system-ip, couleur, encapsulation) |
| **Couleur** | Étiquette symbolique d'un transport |
| **System IP** | Identifiant unique d'un équipement dans l'overlay |
| **Site ID** | Identifiant d'un site, utilisé par les politiques |
| **vRoute** | Route annoncée par OMP |
| **DTLS / TLS** | Chiffrement de la connexion de contrôle (UDP / TCP) |
| **Graceful restart** | Conservation des routes pendant une indisponibilité des contrôleurs |
| **Control connection affinity** | Préférence de connexion à un groupe de Controllers |
| **WAN Edge List** | Liste des équipements autorisés à rejoindre la fabrique |
| **SUDI** | Certificat d'identité installé en usine sur les équipements matériels |
| **MRF** | Multi-Region Fabric : fabrique découpée en régions |
| **NMS** | Network Management System ; désigne le Manager côté logiciel |
| **Service chaining** | Insertion d'un service (pare-feu) dans le chemin du trafic |

### B. Cas d'usage détaillés

#### Cas 1. Pourquoi mon site n'apparaît pas dans le Manager ?

| Étape | Vérification | Résultat attendu |
|---|---|---|
| 1 | Le site joint-il le Validator ? | `show sdwan control connections` : ligne `vbond` |
| 2 | Le certificat est-il valide ? | `show sdwan certificate installed` |
| 3 | L'équipement est-il dans la WAN Edge List ? | État **Valid** |
| 4 | L'organisation est-elle la même ? | Même casse, mêmes caractères |
| 5 | Le Manager est-il joignable ? | Ligne `vmanage` en `up` |

Ce cas a une valeur pédagogique : **les cinq étapes suivent les plans** (orchestration → gestion → contrôle → données), et c'est l'ordre dans lequel on diagnostique.

#### Cas 2. Concevoir les couleurs d'une entreprise multi-opérateurs

| Site | Transport 1 | Transport 2 | Couleurs |
|---|---|---|---|
| Hub A | DIA opérateur X | DIA opérateur Y | `biz-internet`, `public-internet` |
| Hub B | MPLS | DIA | `mpls`, `biz-internet` |
| Agence | Fibre pro | 4G | `biz-internet`, `lte` |

Règles retenues : **une couleur désigne un rôle**, pas un opérateur ; `lte` n'est jamais utilisée pour du trafic rayon à rayon ; `mpls` est `restrict`. **Leçon** : un tableau de couleurs fixé **avant** le déploiement évite les refontes douloureuses.

#### Cas 3. Perte des Controllers pendant une panne cloud

Un incident chez un hébergeur rend les Controllers injoignables pendant 3 heures. **Constat** : les tunnels existants, les routes et les politiques en cache restent actifs grâce au graceful restart ; **ce qui ne fonctionne plus** : nouveaux sites, changement de configuration, mises à jour de clés. **Leçon** : connaître **ce que l'on perd et ce que l'on garde** quand le plan de contrôle est indisponible ; prévoir la haute disponibilité et tester.

#### Cas 4. Passage de 50 à 600 sites

Au-delà de quelques centaines de sites, le maillage complet devient coûteux (nombre de tunnels, BFD). Décision : hubs régionaux et politiques de topologie, puis évaluation du Multi-Region Fabric. **Leçon** : le **passage à l'échelle** se prépare à la conception (SW13), pas après.

#### Cas 5. Rachat d'une entreprise et fusion de deux fabriques

Deux organisations, deux ensembles de contrôleurs, des site-id qui se chevauchent. Le chantier : renuméroter les site-id et system-ip, harmoniser les couleurs, migrer par vagues. **Leçon** : choisissez dès le départ une **politique de numérotation** réservant des plages pour les acquisitions futures.

### C. Retours d'expérience

1. **Le Validator est le maillon invisible.** On l'oublie jusqu'au jour où il n'est plus joignable ; mettez-le sur au moins deux adresses et supervisez-le.
2. **Les identifiants doivent être uniques.** Une system-ip dupliquée (copie de machine virtuelle) provoque des comportements erratiques difficiles à reconnaître.
3. **Le DNS est une dépendance critique.** Un équipement qui ne résout pas le nom du Validator n'entre pas dans la fabrique.
4. **Ne pas laisser `allow-service all` en production** sur les interfaces exposées à Internet.
5. **Lisez la sortie `show sdwan control connections` en entier.** La colonne de couleur, le port et l'état disent presque toujours où chercher.
6. **Gardez un schéma vivant** : sans schéma des couleurs, des VPN et des sites, un architecte quittant l'équipe emporte la connaissance avec lui.

### D. Exercices supplémentaires

1. Dessinez les flux de contrôle d'un site avec deux transports et deux Controllers : combien de connexions de contrôle attendez-vous ?
2. Un site possède les couleurs `biz-internet` et `lte`, le hub `biz-internet` et `mpls`. Quels tunnels se forment (sans `restrict`, puis avec `restrict` partout) ?
3. Quelle commande vous dit si les routes des autres sites sont arrivées ? Que signifie `R/I/S` à `0/0/3` ?
4. Pourquoi une system-ip n'a-t-elle pas besoin d'être routable ?

<details>
<summary>Corrigés</summary>

1. En général, par interface de transport, une connexion vers chaque Controller (dans la limite du paramétrage) et vers le Manager : avec deux transports et deux Controllers, **jusqu'à 4 connexions OMP/contrôle**, plus la connexion au Manager (la limite exacte dépend de `max-control-connections`).
2. Sans `restrict` : `biz-internet`↔`biz-internet` et, si joignables, `lte` vers `biz-internet`/`mpls` ; avec `restrict` partout : **seul** `biz-internet`↔`biz-internet`.
3. `show sdwan omp routes` (et `omp peers`). `0/0/3` : aucune route reçue, aucune installée, 3 envoyées : le Controller ne vous transmet rien (politique de contrôle ou absence d'autres sites).
4. C'est un identifiant, utilisé comme nom de nœud dans OMP ; les tunnels utilisent les adresses de **transport**.

</details>

### E. Mini-projet (4 heures)

Rédigez le **document d'architecture « as-built »** d'une fabrique de laboratoire : schéma, tableau des identifiants (system-ip, site-id), couleurs et leur rôle, ports ouverts, procédure d'arrivée d'un nouveau site, matrice « panne d'un composant → impact ».

---

> Module précédent : [SW03](../../01-partie-a-fondamentaux/SW03-fonctions-cles/SW03-fonctions-cles.md) | Module suivant : [SW05](../SW05-deploiement-controleurs-equipements/SW05-deploiement-controleurs-equipements.md)
