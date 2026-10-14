# SW07. Politiques centralisées et localisées

| | |
|---|---|
| **Partie** | B. Cisco Catalyst SD-WAN |
| **Durée indicative** | 1,5 à 2 semaines (14 h) |
| **Prérequis** | SW04 à SW06 ; QoS (DSCP, files, shaping) ; BGP (notions de filtrage de routes) |
| **Niveau** | Avancé |
| **Lien examen** | ENSDWI 300-415 : « Politiques », « Sécurité et QoS » |
| **Labs associés** | T4 (hub and spoke par politique de contrôle), T5 (voix avec classes de SLA) |

> Module précédent : [SW06](../SW06-modeles-groupes-segmentation/SW06-modeles-groupes-segmentation.md) | Module suivant : [SW08](../SW08-securite-integree/SW08-securite-integree.md)

> **Avertissement.** La syntaxe des politiques ci-dessous est celle de la CLI du SD-WAN Controller (équivalent de ce que génère l'assistant graphique du Manager). Elle varie selon la version : en production, construisez les politiques dans le Manager, puis comparez avec `show running-config policy` sur le Controller. Les valeurs d'exemple sont des points de départ de laboratoire.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Panorama des politiques](#2-panorama-des-politiques)
3. [Les listes](#3-les-listes)
4. [Politique de contrôle et topologies](#4-politique-de-contrôle-et-topologies)
5. [Politique de données](#5-politique-de-données)
6. [Politique de routage applicatif et classes de SLA](#6-politique-de-routage-applicatif-et-classes-de-sla)
7. [Chaînage de services](#7-chaînage-de-services)
8. [Politiques localisées : QoS, ACL, routage](#8-politiques-localisées--qos-acl-routage)
9. [Visibilité applicative](#9-visibilité-applicative)
10. [Application des politiques : sens, ordre, activation](#10-application-des-politiques--sens-ordre-activation)
11. [Vérification et dépannage](#11-vérification-et-dépannage)
12. [Lab guidé T4 : concentrateur et rayons, puis maillage partiel](#12-lab-guidé-t4--concentrateur-et-rayons-puis-maillage-partiel)
13. [Lab guidé T5 : routage applicatif de la voix](#13-lab-guidé-t5--routage-applicatif-de-la-voix)
14. [Exercices](#14-exercices)
15. [Quiz](#15-quiz)
16. [Aide-mémoire](#16-aide-mémoire)
17. [Pour aller plus loin](#17-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Distinguer politiques centralisées (contrôle, données, routage applicatif) et localisées (QoS, ACL, routage).
- Construire des listes (sites, VPN, TLOC, applications, préfixes) et des classes de SLA.
- Concevoir une topologie (concentrateur et rayons, maillage partiel, hubs régionaux) par politique de contrôle.
- Écrire un routage applicatif conforme à un SLA de voix.
- Choisir le sens d'application (entrant, sortant) et vérifier l'effet réel.
- Diagnostiquer une politique sans effet.

---

## 2. Panorama des politiques

```text
                         POLITIQUES CENTRALISEES (SD-WAN Controller)
   +--------------------+ +-----------------------+ +---------------------------+
   | Politique de       | | Politique de données  | | Politique de routage      |
   | CONTROLE           | | (data policy)         | | APPLICATIF (AAR)          |
   | topologie, routes, | | transfert selon       | | classes de SLA et chemins |
   | TLOC (OMP)         | | adresse/application   | | préférés                  |
   +--------------------+ +-----------------------+ +---------------------------+
   + politique d'appartenance aux VPN (VPN membership) + chaînage de services

                         POLITIQUES LOCALISEES (sur chaque WAN Edge, via modèle)
   +----------------+ +--------+ +-------------------+ +-----------------+
   | QoS            | | ACL    | | Routage (route    | | Accès à         |
   | (files, shaping)| | L3/L4  | | policy)           | | l'équipement    |
   +----------------+ +--------+ +-------------------+ +-----------------+
```

| Politique | Où elle s'exécute | Comment elle arrive | Rôle |
|---|---|---|---|
| **Contrôle** | Sur le Controller (traitement des mises à jour OMP) | Reste sur le Controller | Filtrer/modifier routes et TLOC : topologie |
| **Données** | Sur le WAN Edge (plan de données) | Envoyée par le Controller via OMP | Décider du transfert (NAT DIA, chaînage, chemin) |
| **Routage applicatif** | Sur le WAN Edge | Envoyée par le Controller via OMP | Choisir le tunnel selon les SLA |
| **Localisée** | Sur le WAN Edge | Par le modèle/groupe de configuration | QoS, ACL, route policy |

> Une seule **politique centralisée** peut être **active** à la fois sur la fabrique ; elle contient tous les types (contrôle, données, AAR…). Les politiques localisées sont, elles, rattachées à chaque équipement.

---

## 3. Les listes

Les politiques référencent des **listes** réutilisables.

| Liste | Contenu | Exemple |
|---|---|---|
| **Site** | Plages de site-id | `HUBS` = 10-11 ; `SPOKES` = 100-499 |
| **VPN** | Numéros de VPN | `CORP` = 10 ; `GUEST` = 20 |
| **TLOC** | TLOC avec préférence | `(10.255.0.1, mpls, ipsec, preference 200)` |
| **Application** | Applications et familles | `webex`, `ms-teams`, `youtube` |
| **Préfixes de données** | Plages d'adresses (source/destination) | `RFC1918`, `SAAS-IPS` |
| **Préfixes** (route) | Préfixes pour filtrer des routes | `LAN-CORP` |
| **Couleur** | Couleurs de transport | `PRIVEES` = mpls, metro-ethernet |
| **Classe de SLA** | Seuils perte, délai, gigue | `VOICE`, `VIDEO` |
| **Politique** | Politiques réutilisables | |

Exemple en CLI :

```text
policy
 lists
  site-list HUBS
   site-id 10-11
  !
  site-list SPOKES
   site-id 100-499
  !
  vpn-list CORP
   vpn 10
  !
  vpn-list GUEST
   vpn 20
  !
  tloc-list HUB-TLOCS
   tloc 10.255.0.1 color mpls encap ipsec preference 200
   tloc 10.255.0.1 color biz-internet encap ipsec preference 100
  !
  data-prefix-list RFC1918
   ip-prefix 10.0.0.0/8
   ip-prefix 172.16.0.0/12
   ip-prefix 192.168.0.0/16
  !
  app-list VOICE-APPS
   app-family voice-and-video
  !
```

Les listes d'applications peuvent contenir des **applications** ou des **familles d'applications** reconnues par l'inspection de paquets (selon la bibliothèque de signatures de la version).

---

## 4. Politique de contrôle et topologies

### 4.1 Principe

Par défaut, le Controller redistribue **tout** à tous : le résultat est un **maillage complet** (chaque site peut monter un tunnel vers tous les autres). Pour obtenir un autre graphe, la **politique de contrôle** filtre ou réécrit les routes et les TLOC que chaque site reçoit.

```text
Sans politique : tous voient tout  ->  tunnels A-B, A-C, B-C  (maillage)
Avec politique  : les rayons ne voient que les TLOC et routes du hub  ->  tunnels A-HUB, B-HUB (concentrateur et rayons)
```

Sens d'application de la politique de contrôle :

| Sens | Effet |
|---|---|
| **In** | Appliquée aux mises à jour reçues **du** site (avant d'entrer dans la table du Controller) |
| **Out** | Appliquée aux mises à jour envoyées **vers** le site |

### 4.2 Topologie concentrateur et rayons (hub and spoke)

```text
policy
 control-policy HUB-AND-SPOKE
  sequence 10
   match tloc
    site-list HUBS
   !
   action accept
   !
  !
  sequence 20
   match route
    site-list HUBS
   !
   action accept
   !
  !
  default-action reject
 !
!
apply-policy
 site-list SPOKES
  control-policy HUB-AND-SPOKE out
 !
!
```

Lecture :

1. La politique est appliquée **sortante** vers les rayons (`SPOKES`).
2. Les rayons ne reçoivent que les **TLOC** des hubs (séquence 10) et les **routes** originées par les hubs (séquence 20) ; tout le reste est rejeté (`default-action reject`).
3. Les rayons n'ont donc de tunnels que vers les hubs. Le hub, lui, reçoit tout (la politique n'est pas appliquée vers lui).

> **Piège majeur** : pour que deux rayons communiquent **via le hub**, le hub doit **annoncer un résumé ou une route par défaut** couvrant les réseaux des rayons (sinon les rayons ne connaissent pas les préfixes des autres rayons). Exemple de résumé : le hub annonce `10.0.0.0/8` dans OMP.

### 4.3 Maillage partiel : rayons avec liaison directe entre certains sites

```text
policy
 lists
  site-list REGION-SUD
   site-id 100-199
  !
 !
 control-policy PARTIAL-MESH
  sequence 10
   match tloc
    site-list HUBS
   !
   action accept
  !
  sequence 20
   match tloc
    site-list REGION-SUD
   !
   action accept
  !
  sequence 30
   match route
    site-list HUBS
   !
   action accept
  !
  sequence 40
   match route
    site-list REGION-SUD
   !
   action accept
  !
  default-action reject
 !
!
apply-policy
 site-list REGION-SUD
  control-policy PARTIAL-MESH out
 !
```

Les sites de la région Sud se voient entre eux (tunnels directs) et voient les hubs, mais pas les autres régions.

### 4.4 Hubs régionaux et hiérarchie

| Modèle | Description | Usage |
|---|---|---|
| **Hub unique** | Un concentrateur (ou une paire) | Réseaux modestes |
| **Hubs régionaux** | Un hub par région, hubs interconnectés | Réduire les latences et les tunnels |
| **Multi-Region Fabric (MRF)** | Régions et routeurs de bordure natifs | Grandes fabriques (SW04, SW13) |

### 4.5 Modification de la préférence de TLOC et de route

```text
control-policy PREFER-MPLS
 sequence 10
  match route
   site-list HUBS
  !
  action accept
   set
    preference 200
   !
  !
 !
 sequence 20
  match tloc
   color mpls
  !
  action accept
   set
    preference 300
   !
  !
 !
 default-action accept
```

Ici, les routes des hubs ont une préférence plus haute (200) et les TLOC `mpls` une préférence supérieure (300). Le trafic privilégiera MPLS tant qu'il est disponible.

### 4.6 Politique d'appartenance aux VPN

Une politique d'appartenance (`vpn-membership-policy`) limite les VPN reçus par un site : un site qui n'a pas besoin du VPN 30 n'en reçoit pas les routes.

```text
policy
 vpn-membership-policy VPN-MEMBERSHIP-BR
  vpn-list CORP
  vpn-list GUEST
 !
apply-policy
 site-list SPOKES
  vpn-membership-policy VPN-MEMBERSHIP-BR
```

---

## 5. Politique de données

La **politique de données** (data policy) décide du transfert d'un flux **au moment de sa traversée** du WAN Edge (plan de données) : accepter, rejeter, marquer, **NAT DIA**, rediriger vers un service, choisir un TLOC, compter.

### 5.1 Champs de correspondance (match)

`source-ip`, `destination-ip`, `source-data-prefix-list`, `destination-data-prefix-list`, `source-port`, `destination-port`, `protocol`, `dscp`, `packet-length`, `app-list`, `dns-app-list`, `plp`, `tcp`, etc.

### 5.2 Actions

| Action | Effet |
|---|---|
| `accept` | Accepter, avec des actions complémentaires |
| `drop` | Rejeter |
| `set dscp` | Marquer |
| `set local-tloc-list color <c>` | Choisir le transport local de sortie |
| `set tloc-list <liste>` / `set tloc <ip>` | Choisir le TLOC distant |
| `set next-hop <ip>` | Imposer le prochain saut |
| `set service <svc> vpn <n>` | Insertion de service (chaînage) |
| `nat use-vpn 0` | Sortie directe vers Internet avec NAT |
| `nat fallback` | Retour sur l'overlay si la sortie directe est indisponible |
| `count <nom>` | Compteur |
| `log` | Journaliser |
| `cflowd` | Exporter des flux |

### 5.3 Exemple : DIA pour les invités et pour des applications SaaS

```text
policy
 lists
  app-list SAAS-APPS
   app office365
   app webex
   app salesforce
  !
 !
 data-policy DP-DIA
  vpn-list GUEST
   sequence 10
    match
     source-ip 0.0.0.0/0
    !
    action accept
     nat use-vpn 0
     nat fallback
    !
   !
   default-action accept
  !
  vpn-list CORP
   sequence 10
    match
     app-list SAAS-APPS
    !
    action accept
     nat use-vpn 0
     nat fallback
    !
   !
   default-action accept
  !
 !
!
apply-policy
 site-list SPOKES
  data-policy DP-DIA from-service
 !
```

Lecture : sur les rayons, le trafic des invités et les applications SaaS du VPN 10 sortent **directement** par le transport Internet avec NAT ; tout le reste emprunte l'overlay.

### 5.4 Sens d'application

| Sens | Signification |
|---|---|
| `from-service` | Trafic **entrant côté LAN** (du site vers l'overlay) |
| `from-tunnel` | Trafic **arrivant de l'overlay** |
| `all` | Les deux sens |

> Le DIA se décide au **départ** du flux (`from-service`) ; un `from-tunnel` ne permet pas de décider de la sortie locale du flux « retour » qui a la forme du trafic de retour de la traduction NAT.

### 5.5 Exemple : orienter un flux vers un transport

```text
data-policy DP-BACKUP-VIA-INET
 vpn-list CORP
  sequence 10
   match
    app-list BACKUP-APPS
   !
   action accept
    set
     local-tloc-list
      color biz-internet
      encap ipsec
     !
    !
   !
  !
  default-action accept
```

Les sauvegardes sont envoyées par le transport `biz-internet`, préservant MPLS pour la voix.

---

## 6. Politique de routage applicatif et classes de SLA

### 6.1 Classes de SLA

```text
policy
 sla-class VOICE
  loss 1
  latency 150
  jitter 30
 !
 sla-class VIDEO
  loss 2
  latency 200
  jitter 50
 !
 sla-class BUSINESS
  loss 3
  latency 300
  jitter 100
 !
 sla-class DEFAULT
  loss 25
  latency 500
  jitter 500
 !
```

| Classe | Perte max (%) | Délai max (ms) | Gigue max (ms) | Usage |
|---|---|---|---|---|
| VOICE | 1 | 150 | 30 | Téléphonie |
| VIDEO | 2 | 200 | 50 | Visioconférence |
| BUSINESS | 3 | 300 | 100 | Applications d'entreprise |
| DEFAULT | 25 | 500 | 500 | Meilleur effort |

> Le **délai** de la classe de SLA est mesuré par BFD sur le chemin (latence **aller-retour divisée par deux** dans les mesures ; vérifiez la définition utilisée par votre version avant de comparer à des seuils G.114).

### 6.2 Politique de routage applicatif

```text
policy
 app-route-policy AAR-CORP
  vpn-list CORP
   sequence 10
    match
     dscp 46
    !
    action
     sla-class VOICE preferred-color mpls
    !
   !
   sequence 20
    match
     app-list VIDEO-APPS
    !
    action
     sla-class VIDEO preferred-color mpls biz-internet
    !
   !
   sequence 30
    match
     app-list BUSINESS-APPS
    !
    action
     sla-class BUSINESS preferred-color biz-internet
    !
   !
   sequence 40
    match
     source-ip 0.0.0.0/0
    !
    action
     sla-class DEFAULT
    !
   !
  !
 !
!
apply-policy
 site-list SPOKES
  app-route-policy AAR-CORP
 !
```

Comportement :

1. Les flux marqués DSCP 46 (voix) empruntent de préférence **MPLS**, tant que MPLS respecte la classe `VOICE`.
2. Si MPLS ne respecte plus le SLA, le flux bascule sur **un autre chemin conforme** (par exemple biz-internet).
3. Si **aucun** chemin n'est conforme, le comportement par défaut est d'utiliser le **meilleur chemin disponible** (selon les critères de repli configurables dans la classe de SLA : `fallback-best-tunnel`). Des options existent pour interdire tout autre chemin (`strict`) ou préciser des couleurs de secours ; leur syntaxe dépend de la version.

> L'AAR s'applique aux flux **sortants** des sites ciblés. Pour un comportement symétrique (aller et retour par les mêmes chemins), appliquez la politique **sur les deux sites** extrémités ou ajoutez des préférences de TLOC cohérentes.

### 6.3 Réglage des temporisations de mesure

```text
! Extrait indicatif (modèle BFD du Manager ; syntaxe à vérifier selon la version)
sdwan
 bfd app-route multiplier 6
 bfd app-route poll-interval 120000
 bfd color mpls
  hello-interval 1000
  multiplier 7
 !
```

| Paramètre | Valeur par défaut | Conséquence |
|---|---|---|
| Intervalle BFD (hello) | 1 000 ms | Fréquence des sondes |
| Poll interval AAR | 600 000 ms (10 min) | Longueur d'une fenêtre de mesure |
| Multiplicateur AAR | 6 | Nombre de fenêtres moyennées |

Réactivité : la décision se base sur la **moyenne de `multiplicateur` fenêtres de `poll-interval`**. Avec 600 s × 6 : **1 heure** d'historique. Avec 120 s × 6 : 12 minutes, détection en quelques minutes. Plus l'intervalle est court, plus la réaction est rapide, mais plus la sensibilité aux pics est grande (risque d'oscillations).

### 6.4 Sélecteurs de chemin alternatifs

| Option | Rôle |
|---|---|
| `preferred-color` | Couleurs préférées (conformes au SLA) |
| `backup-preferred-color` | Couleurs de secours (si les préférées sont non conformes) selon la version |
| `strict` | Interdire les couleurs hors liste |
| `fallback-best-tunnel` (dans la classe de SLA) | Critère de repli : choisir le meilleur tunnel même non conforme selon perte, délai, gigue |
| `count`, `log` | Observabilité |

---

## 7. Chaînage de services

Le **chaînage de services** insère un service (pare-feu, IPS, proxy) dans le chemin du trafic, même s'il est hébergé sur un autre site.

```text
Poste (site A) --> WAN Edge A --[overlay]--> WAN Edge Hub --> Pare-feu (service FW) --> retour --> WAN Edge Hub --> destination
```

Configuration (principe) :

1. Le site qui héberge le pare-feu **annonce un service** `FW` dans OMP (routes de service).
2. Une **politique de contrôle** ou de données force les flux vers ce service.

```text
control-policy CP-FW-INSERT
 sequence 10
  match route
   vpn-list GUEST
   prefix-list ANY
  !
  action accept
   set
    service FW vpn 10
   !
  !
 !
 default-action accept
```

Intérêts : centraliser l'inspection sur quelques pare-feu ; **limites** : latence, goulot, point de défaillance (à doubler).

---

## 8. Politiques localisées : QoS, ACL, routage

### 8.1 QoS localisée (modèle SD-WAN)

Rappel : la QoS agit **à la sortie** de l'interface (WAN). Les politiques localisées s'appliquent à l'**équipement**.

```text
policy
 class-map
  class VOICE queue 0
  class VIDEO queue 1
  class BUSINESS queue 2
  class DEFAULT queue 3
 !
 qos-scheduler Q0-VOICE
  class VOICE
  bandwidth-percent 15
  buffer-percent 15
  scheduling llq
  drops tail-drop
 !
 qos-scheduler Q1-VIDEO
  class VIDEO
  bandwidth-percent 25
  buffer-percent 25
  scheduling wrr
  drops red-drop
 !
 qos-scheduler Q2-BUSINESS
  class BUSINESS
  bandwidth-percent 30
  buffer-percent 30
  scheduling wrr
  drops red-drop
 !
 qos-scheduler Q3-DEFAULT
  class DEFAULT
  bandwidth-percent 30
  buffer-percent 30
  scheduling wrr
  drops red-drop
 !
 qos-map QOS-WAN
  qos-scheduler Q0-VOICE
  qos-scheduler Q1-VIDEO
  qos-scheduler Q2-BUSINESS
  qos-scheduler Q3-DEFAULT
 !
```

L'association du **qos-map** à une interface se fait dans le modèle d'interface de transport (champ « QoS Map »), avec le **lissage** (shaping rate) du débit du contrat.

### 8.2 Équivalent MQC sur IOS XE (pour comprendre ce qui est réellement appliqué)

```text
class-map match-any VOICE
 match dscp ef
class-map match-any VIDEO
 match dscp af41 af42
class-map match-any BUSINESS
 match dscp af31 af32
!
policy-map CHILD-WAN
 class VOICE
  priority percent 15
 class VIDEO
  bandwidth remaining percent 30
 class BUSINESS
  bandwidth remaining percent 35
 class class-default
  bandwidth remaining percent 35
  random-detect dscp-based
!
policy-map PARENT-SHAPE-100M
 class class-default
  shape average 100000000
  service-policy CHILD-WAN
!
interface GigabitEthernet1
 service-policy output PARENT-SHAPE-100M
```

Contrôles :

```text
show policy-map interface GigabitEthernet1 output
show platform hardware qfp active feature qos interface GigabitEthernet1
```

### 8.3 ACL et politique de routage localisées

| Politique | Usage | Exemple |
|---|---|---|
| **ACL** | Filtrer le trafic sur une interface | Autoriser seulement 443 depuis les invités |
| **Route policy** | Filtrer ou modifier les routes à la redistribution | Balises anti-boucle |
| **Politique d'accès à l'équipement** (device access policy) | Limiter qui peut se connecter en SSH/SNMP | Seulement le réseau d'administration |

Exemple d'ACL localisée appliquée côté invités :

```text
policy
 access-list GUEST-IN
  sequence 10
   match
    destination-data-prefix-list RFC1918
   !
   action drop
   !
  !
  sequence 20
   match
    protocol 6
    destination-port 80 443
   !
   action accept
  !
  default-action drop
 !
```

---

## 9. Visibilité applicative

- L'**inspection de paquets** (DPI, moteur NBAR2 sur IOS XE) classe les flux par application. Les signatures se mettent à jour par **paquet de protocole** (protocol pack).
- Pour les applications SaaS, la classification peut aussi s'appuyer sur la **résolution DNS** (listes `dns-app-list`).
- Le premier paquet d'un flux n'est pas toujours identifiable : l'application est **reconnue après quelques paquets** (dont chiffrés par SNI/certificat) ; la politique peut s'appliquer au flux seulement après classification.

Commandes de vérification (IOS XE) :

```text
show sdwan app-fwd cflowd flows
show sdwan app-fwd dpi flows
show ip nbar protocol-discovery
```

> Utilisez des `dscp` et des préfixes pour les flux **critiques** et **faciles à identifier** (voix) et réservez la reconnaissance applicative aux cas où elle apporte une valeur (SaaS).

---

## 10. Application des politiques : sens, ordre, activation

### 10.1 Procédure dans le Manager (politique centralisée)

**Configuration > Policies > Centralized Policy > Add Policy** :

1. **Créer les listes** (sites, VPN, applications, préfixes, SLA).
2. **Configurer la topologie** (hub and spoke, mesh, custom control, VPN membership).
3. **Configurer les règles de trafic** (routage applicatif, données, cflowd).
4. **Appliquer aux sites et VPN** (liens vers les listes de sites avec le sens).
5. Enregistrer la politique.
6. **Activate** : le Manager la pousse au Controller.

### 10.2 Ordre d'évaluation

- À l'intérieur d'une politique : les **séquences** sont évaluées par numéro croissant ; **la première correspondance** s'applique.
- Si aucune ne correspond : **action par défaut** (`default-action`), qui est `reject` pour une politique de contrôle de topologie « hub and spoke » ; `accept` pour une politique de données et d'AAR.
- Plusieurs politiques de **natures différentes** peuvent s'appliquer au même site (une de contrôle, une de données, une AAR).

### 10.3 Pièges fréquents

| Piège | Conséquence | Parade |
|---|---|---|
| Mauvais sens (`in` au lieu de `out`) | Effet inverse | Relire la section 4.1 |
| Liste de sites erronée | Politique sans effet | Vérifier l'appartenance du site-id |
| `default-action reject` oublié ou mal compris | Perte de routes | Tester sur un site pilote |
| Séquences dans le mauvais ordre | Une règle générale masque une règle spécifique | Spécifique avant général |
| Politique non activée | Aucune application | Statut « Active » |
| VPN absent de la liste | Aucune application sur ce VPN | Vérifier `vpn-list` |

---

## 11. Vérification et dépannage

### 11.1 Sur le Controller

```text
show running-config policy
show running-config apply-policy
show omp routes vpn 10
show omp tlocs
show omp peers
show policy from-vsmart            # (selon version)
```

### 11.2 Sur le WAN Edge

```text
show sdwan policy from-vsmart                     # politiques reçues
show sdwan policy app-route-policy-filter         # AAR : compteurs par séquence
show sdwan policy data-policy-filter              # données : compteurs
show sdwan app-route stats                        # mesures perte, délai, gigue par tunnel
show sdwan app-route sla-class                    # classes de SLA et tunnels conformes
show sdwan bfd sessions
show sdwan omp tlocs
show sdwan policy service-path vpn 10 interface GigabitEthernet3 source-ip 192.168.10.100 dest-ip 10.1.10.50 protocol 1 all
show sdwan policy tunnel-path vpn 10 interface GigabitEthernet3 source-ip 192.168.10.100 dest-ip 10.1.10.50 protocol 1 all
```

La commande `show sdwan policy service-path` indique, pour un flux donné, **quel chemin la politique choisirait** : indispensable pour valider une AAR.

### 11.3 Méthode de dépannage d'une politique

| Étape | Question | Commande |
|---|---|---|
| 1 | La politique est-elle active sur le Controller ? | `show running-config policy`, statut dans le Manager |
| 2 | Le site est-il dans la bonne liste ? | `show running-config apply-policy` |
| 3 | Le WAN Edge l'a-t-il reçue ? | `show sdwan policy from-vsmart` |
| 4 | La correspondance se produit-elle ? | Compteurs de séquence (`*-policy-filter`) |
| 5 | Le chemin attendu est-il choisi ? | `show sdwan policy service-path ...` |
| 6 | Les mesures du chemin sont-elles saines ? | `show sdwan app-route stats` |

### 11.4 Cas : « La voix passe par Internet alors que MPLS est sain »

1. `show sdwan policy from-vsmart` : l'AAR est-elle présente ?
2. `show sdwan app-route sla-class` : MPLS est-il conforme à `VOICE` ? (si la gigue mesurée vaut 35 ms, MPLS n'est pas conforme)
3. `show sdwan policy service-path` pour un flux marqué DSCP 46 : quel chemin ?
4. Vérifier que les paquets de voix sont bien marqués **DSCP 46** à l'entrée (sinon la séquence ne correspond pas).
5. Corriger : marquage ou seuil de la classe de SLA.

---

## 12. Lab guidé T4 : concentrateur et rayons, puis maillage partiel

### 12.1 Objectif

Observer l'effet d'une politique de contrôle sur le graphe des tunnels.

### 12.2 Préparation

Fabrique du lab T2/T3 avec un hub (site 10) et trois agences (sites 100, 101, 102). Aucune politique active : **maillage complet**.

### 12.3 Étapes

| # | Action | Vérification |
|---|---|---|
| 1 | Relever l'état initial : `show sdwan bfd sessions` sur l'agence 100 | Sessions vers hub **et** vers 101, 102 |
| 2 | Créer les listes `HUBS` (10) et `SPOKES` (100-102) | Listes présentes |
| 3 | Créer `HUB-AND-SPOKE` (section 4.2), l'appliquer à `SPOKES` en `out`, **activer** | Statut actif |
| 4 | `show sdwan bfd sessions` sur l'agence 100 | Plus de sessions vers 101 et 102 (après quelques minutes) |
| 5 | `ping` PC agence 100 → PC agence 101 | **Échec** (pas de routes) |
| 6 | Sur le hub, annoncer un résumé (`ip route vrf 10 192.168.0.0 255.255.0.0 Null0` + `advertise static`) | Route résumée apparaît sur les agences |
| 7 | Relancer le ping | Succès, via le hub (`traceroute vrf 10`) |
| 8 | Passer à `PARTIAL-MESH` pour les agences 100 et 101 (région) | Tunnel direct 100 ↔ 101 apparaît ; 102 reste via le hub |
| 9 | Rapport : schéma avant/après, nombre de tunnels, impact sur la latence | |

### 12.4 Questions

1. Combien de tunnels BFD par agence avant et après ?
2. Quelle est la conséquence de `default-action reject` sur une route non listée ?
3. Qu'arrive-t-il si le hub tombe ? Comment éviter ce point de défaillance ?

<details>
<summary>Corrigé</summary>

1. Avant : (3 autres sites) × 2 TLOC = 6 sessions ; après : 2 (vers le hub). 2. Elle n'est pas distribuée au site visé par la politique. 3. Les rayons perdent la connectivité inter-sites ; ajouter un second hub (liste `HUBS` de deux sites) et des préférences de TLOC pour le primaire.

</details>

---

## 13. Lab guidé T5 : routage applicatif de la voix

### 13.1 Objectif

Configurer une classe de SLA `VOICE`, une politique AAR, injecter une dégradation sur MPLS et observer la bascule.

### 13.2 Étapes

| # | Action | Vérification |
|---|---|---|
| 1 | Déployer deux transports (mpls, biz-internet) sur l'agence et le hub | 4 sessions BFD |
| 2 | Créer la classe `VOICE` (1 %, 150 ms, 30 ms) | Classe présente |
| 3 | Créer la politique AAR (section 6.2) pour le VPN 10, préférence mpls ; l'appliquer aux rayons ; activer | `show sdwan policy from-vsmart` |
| 4 | Réduire le `poll-interval` à 60-120 s pour le lab (modèle BFD) | Valeurs appliquées |
| 5 | Générer un flux marqué DSCP 46 : `iperf3 -u -b 100k -l 160 --tos 184 -c <hub>` | `show sdwan policy service-path` indique mpls |
| 6 | Dégrader MPLS sur le routeur d'underlay : `tc qdisc add dev ethX root netem delay 200ms 20ms loss 5%` | `show sdwan app-route stats` : MPLS hors SLA |
| 7 | Observer la bascule | `service-path` indique biz-internet |
| 8 | Rétablir MPLS : `tc qdisc del dev ethX root` | Retour sur mpls après stabilisation |
| 9 | Mesurer le temps de bascule, comparer avec les paramètres de poll | Tableau |

> `--tos 184` correspond à DSCP 46 (EF) : 46 × 4 = 184.

### 13.3 Script de chronométrage (Linux)

```bash
#!/usr/bin/env bash
# Chronomètre le temps entre l'injection de la dégradation et le changement de chemin
# (le chemin est interrogé par SSH sur le WAN Edge avec la commande service-path).
EDGE=10.255.1.100
CMD='show sdwan policy service-path vpn 10 interface GigabitEthernet3 source-ip 192.168.10.100 dest-ip 10.1.10.50 protocol 17 all'
chemin() { ssh -o StrictHostKeyChecking=no admin@$EDGE "$CMD" | grep -oE 'mpls|biz-internet' | head -1; }

AVANT=$(chemin)
echo "chemin initial : $AVANT"
DEBUT=$(date +%s)
# injecter la dégradation (à adapter)
ssh underlay "sudo tc qdisc add dev eth1 root netem delay 200ms 20ms loss 5%"
while [ "$(chemin)" = "$AVANT" ]; do sleep 5; done
echo "bascule en $(( $(date +%s) - DEBUT )) secondes vers $(chemin)"
```

### 13.4 Livrable

Captures (`app-route stats`, `service-path`) avant, pendant, après ; temps de bascule mesuré ; analyse du réglage poll-interval × multiplicateur.

---

## 14. Exercices

1. Écrivez une politique de contrôle « hub and spoke avec deux hubs, préférence pour HUB1 » (listes, TLOC préférés, application).
2. Dans une AAR, la voix doit utiliser MPLS puis Internet fibre, jamais la 5G. Comment l'exprimer ?
3. Expliquez la différence entre `from-service` et `from-tunnel` dans une politique de données.
4. Les agences ne communiquent plus entre elles après une politique hub-and-spoke. Cause et solution ?
5. Calculez la fenêtre d'historique pour poll-interval 120 s et multiplicateur 6.

<details>
<summary>Corrigés</summary>

1. Listes : `HUBS` = 10-11, `TLOC-HUB1` préférence 200, `TLOC-HUB2` préférence 100 ; séquence `match tloc site-list HUBS`/`match route site-list HUBS` ; politique `out` sur `SPOKES` ; préférence de TLOC via `set preference` dans la séquence.
2. `sla-class VOICE preferred-color mpls biz-internet` + ne pas lister `lte` ; mode `strict` si l'on interdit même le repli sur la 5G.
3. `from-service` : trafic venant du LAN ; `from-tunnel` : trafic venant de l'overlay.
4. Les rayons ne reçoivent que les routes des hubs ; ajouter un résumé ou une route par défaut annoncé par le hub.
5. 120 × 6 = 720 s = 12 minutes.

</details>

---

## 15. Quiz

1. Quelle politique est exécutée **sur le Controller** ?
2. Combien de politiques centralisées peuvent être actives simultanément ?
3. Quelle commande indique le chemin qu'une politique choisirait pour un flux ?
4. Que fait `nat use-vpn 0` ?
5. À quoi correspond DSCP 46 ?
6. Pourquoi la politique de données s'applique-t-elle surtout `from-service` pour le DIA ?
7. Qu'est-ce qu'une classe de SLA ?

<details>
<summary>Réponses</summary>

1. La politique de contrôle. 2. Une seule. 3. `show sdwan policy service-path ...`. 4. Il envoie le trafic directement par le VPN 0 avec NAT (sortie Internet locale). 5. EF (Expedited Forwarding), la voix. 6. Parce que la décision de sortie locale se prend à l'entrée du flux côté LAN. 7. Un ensemble de seuils (perte, délai, gigue) qui définit un chemin « conforme ».

</details>

---

## 16. Aide-mémoire

```text
Contrôle  (Controller)  : topologie, routes, TLOC       apply-policy site-list X control-policy P in|out
Données   (WAN Edge)    : NAT DIA, chemin, chaînage      apply-policy site-list X data-policy P from-service|from-tunnel|all
AAR       (WAN Edge)    : sla-class + preferred-color    apply-policy site-list X app-route-policy P
Localisée (WAN Edge)    : QoS, ACL, route policy          via le modèle/groupe de l'équipement
Évaluation : séquences par numéro croissant, première correspondance, puis default-action
Vérifier   : show sdwan policy from-vsmart | app-route stats | app-route sla-class | policy service-path
Hub-and-spoke : rayons reçoivent TLOC+routes des hubs seulement ; hub annonce un résumé
```

## 17. Pour aller plus loin

- Cisco Catalyst SD-WAN, Policies Configuration Guide (votre version) : <https://www.cisco.com/c/en/us/solutions/enterprise-networks/sd-wan/index.html>
- RFC 4594 (recommandations DiffServ) : <https://www.rfc-editor.org/rfc/rfc4594>
- Suite : [CP02. QoS en détail pour le WAN](../../complements/CP02-qos-wan/CP02-qos-wan.md)

---

## Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **Centralized policy** | Politique définie sur le Manager et exécutée par le Controller / distribuée aux WAN Edge |
| **Localized policy** | Politique appliquée localement sur un équipement (QoS, ACL, routage) |
| **Control policy** | Filtre ou modifie routes et TLOC dans OMP |
| **Data policy** | Règle de transfert appliquée au plan de données |
| **App-route policy** | Choix de chemin selon des classes de SLA |
| **Site list / VPN list / TLOC list** | Listes réutilisables référencées par les politiques |
| **Sequence** | Règle numérotée d'une politique ; évaluée par ordre croissant |
| **Default action** | Action appliquée si aucune séquence ne correspond |
| **Hub-and-spoke / mesh** | Topologies produites par politique de contrôle |
| **Service chaining** | Insertion d'un service dans le chemin d'un flux |
| **Cflowd** | Export de statistiques de flux |
| **NBAR2** | Moteur de reconnaissance d'applications de Cisco |
| **LLQ / CBWFQ** | Files à priorité stricte / à bande passante garantie |
| **Policing / shaping** | Écrêtage / lissage du débit |
| **Preferred color** | Couleurs préférées dans une politique de routage applicatif |

### B. Cas d'usage détaillés

#### Cas 1. Un hub principal et un hub de secours

| Rubrique | Détail |
|---|---|
| **Besoin** | Rayons connectés à deux hubs ; le hub A est préféré, B en secours |
| **Politique** | Contrôle `hub-and-spoke` avec deux sites hubs ; préférence de TLOC 200 pour A, 100 pour B |
| **Test** | Coupure du hub A : les rayons basculent sur B en quelques secondes (BFD) |
| **Piège** | Les deux hubs doivent annoncer **les mêmes** résumés, sinon les rayons n'ont pas les mêmes routes |
| **Leçon** | Un test de bascule **planifié et chronométré** est la meilleure recette |

#### Cas 2. Régions Nord et Sud avec maillage partiel

Chaque région voit ses sites en direct et les hubs ; les régions ne se voient qu'au travers des hubs. Résultat : 70 % de tunnels en moins qu'un maillage complet, latence régionale réduite. **Leçon** : la politique de contrôle est **le levier d'échelle** du SD-WAN.

#### Cas 3. Voix de qualité entre deux continents

| Rubrique | Détail |
|---|---|
| **Problème** | Appels Paris–Singapour : la classe `VOICE` (150 ms sens unique) n'est jamais respectée, la politique bascule en permanence |
| **Constat** | Les mesures donnent 170 à 190 ms sens unique sur tous les transports : la propagation en fibre (environ 5 µs/km sur des chemins de 15 000 km et plus) et les détours des câbles sous-marins interdisent de tenir 150 ms |
| **Décision** | Redéfinir une classe de SLA **adaptée à la distance** (`VOICE-INTERCONTINENTAL` : 220 ms sens unique, perte 1 %, gigue 30 ms) plutôt que forcer un seuil inatteignable |
| **Leçon** | Un SLA **inatteignable** provoque des bascules permanentes : fixez des seuils **réalistes**, par type de relation |

#### Cas 4. DIA sélectif pour Microsoft 365

Seul le trafic Microsoft 365 sort localement ; le reste passe par le hub (inspection). Mise en œuvre : politique de données avec liste d'applications, `nat use-vpn 0`, `nat fallback`. **Piège** : les adresses de M365 évoluent ; la classification par application ou par nom de domaine est plus fiable que par listes d'adresses.

#### Cas 5. Sauvegarde qui sature MPLS

Politique de données : classe `BACKUP` envoyée par `biz-internet` (`set local-tloc-list color biz-internet`) ; MPLS reste réservé à la voix et aux applications critiques. **Leçon** : orienter par **transport** fait gagner plus que d'ajouter de la bande passante.

#### Cas 6. Quand une politique casse un site

Un technicien applique une politique `hub-and-spoke` à une liste de sites trop large (la liste `SPOKES` inclut aussi les hubs). Les hubs ne voient plus les routes les uns des autres. **Correction** : liste de sites corrigée en 3 minutes ; **prévention** : relecture obligatoire par un second ingénieur, test sur un site pilote.

### C. Retours d'expérience

1. **Une politique centralisée est un levier extrêmement puissant** : une erreur touche tous les sites. Imposez **revue à quatre yeux, environnement de test, déploiement progressif**.
2. **Gardez les politiques simples.** Au-delà d'une dizaine de séquences par politique, la maintenance devient un risque.
3. **Évitez les politiques « en réaction » à un incident** qui restent en place des années. Datez et documentez chaque règle, avec son propriétaire.
4. **Mesurez l'effet.** Avant/après avec des compteurs : sans mesure, on ne sait pas si la politique sert à quelque chose.
5. **Ne confondez pas qualité mesurée et qualité perçue.** Une classe de SLA conforme n'implique pas que l'utilisateur est satisfait : croisez avec les indicateurs applicatifs.
6. **L'asymétrie est le piège du routage applicatif.** L'aller peut passer par MPLS et le retour par Internet ; appliquez la politique **aux deux extrémités**.
7. **Prévoyez la sortie.** Chaque politique doit pouvoir être **désactivée** proprement ; testez le retour à l'état sans politique.

### D. Exercices supplémentaires

1. Écrivez la politique de contrôle qui connecte directement deux régions (Nord et Sud) mais interdit toute communication de la région Est avec la région Ouest.
2. Rédigez trois classes de SLA pour un hôpital et justifiez les seuils.
3. Un administrateur a mis `default-action accept` dans une politique de topologie hub-and-spoke et constate que le maillage persiste. Pourquoi ?
4. Concevez le plan de test d'une nouvelle politique AAR (préparation, exécution, critères d'acceptation, retour arrière).

<details>
<summary>Corrigés</summary>

1. Séquences acceptant TLOC et routes des sites Nord et Sud pour les listes `NORD` et `SUD`, rien d'Est/Ouest vers l'autre ; `default-action reject`, appliquée `out` à chaque région.
2. Exemple : télé-expertise vidéo (perte 1 %, 150 ms, gigue 30 ms), dossiers patients (3 %, 300 ms, 100 ms), administratif (10 %, 500 ms, 300 ms).
3. Avec `accept` par défaut, toutes les routes et TLOC non listés sont distribués : rien n'est filtré, donc maillage complet.
4. Préparation : sauvegarde, liste de sites pilote ; exécution : activer sur 2 sites, injecter une dégradation, mesurer ; acceptation : bascule < X s, pas d'effet de bord ; retour arrière : désactiver la politique et vérifier `show sdwan policy from-vsmart`.

</details>

### E. Mini-projet (6 heures)

Concevez et documentez le **jeu de politiques complet** d'une entreprise de 40 sites (listes, topologie, AAR, DIA, QoS) avec : schéma de la topologie, tableau des classes de SLA, matrice de décision de chaque application, plan de test et plan de retour arrière. Validez au minimum la topologie et l'AAR sur le lab T4/T5.

---

> Module précédent : [SW06](../SW06-modeles-groupes-segmentation/SW06-modeles-groupes-segmentation.md) | Module suivant : [SW08](../SW08-securite-integree/SW08-securite-integree.md)
