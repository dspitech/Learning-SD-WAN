# SW06. Modèles, groupes de configuration et segmentation

| | |
|---|---|
| **Partie** | B. Cisco Catalyst SD-WAN |
| **Durée indicative** | 1,5 semaine (12 h) |
| **Prérequis** | SW04, SW05 ; OSPF, BGP, VRF, VRRP, NAT |
| **Niveau** | Avancé |
| **Lien examen** | ENSDWI 300-415 : « Déploiement des routeurs » (templates, routage service-side, DIA) |
| **Lab associé** | T3 (trois sites, deux segments, OSPF côté LAN, DIA, VRRP) |

> Module précédent : [SW05](../SW05-deploiement-controleurs-equipements/SW05-deploiement-controleurs-equipements.md) | Module suivant : [SW07](../SW07-politiques-centralisees-localisees/SW07-politiques-centralisees-localisees.md)

> **Avertissement.** Chemins de menus et noms de paramètres varient selon la version (20.x). Les configurations CLI sont indicatives : le Manager génère la configuration finale, à contrôler avec `show sdwan running-config`.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Deux approches de configuration](#2-deux-approches-de-configuration)
3. [Modèles de fonctionnalité et d'équipement](#3-modèles-de-fonctionnalité-et-déquipement)
4. [Variables et fichiers de paramètres](#4-variables-et-fichiers-de-paramètres)
5. [Groupes de configuration et groupes de politiques](#5-groupes-de-configuration-et-groupes-de-politiques)
6. [Comparaison : modèles et groupes de configuration](#6-comparaison--modèles-et-groupes-de-configuration)
7. [Interfaces de transport](#7-interfaces-de-transport)
8. [Segmentation par VPN](#8-segmentation-par-vpn)
9. [Routage côté service](#9-routage-côté-service)
10. [Prévention des boucles et redistribution](#10-prévention-des-boucles-et-redistribution)
11. [Accès direct à Internet (DIA)](#11-accès-direct-à-internet-dia)
12. [Haute disponibilité côté LAN](#12-haute-disponibilité-côté-lan)
13. [Génération de variables par script](#13-génération-de-variables-par-script)
14. [Lab guidé T3](#14-lab-guidé-t3)
15. [Dépannage](#15-dépannage)
16. [Exercices](#16-exercices)
17. [Quiz](#17-quiz)
18. [Aide-mémoire](#18-aide-mémoire)
19. [Pour aller plus loin](#19-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Choisir entre modèles d'équipement et groupes de configuration.
- Construire des modèles réutilisables avec des variables.
- Segmenter l'overlay par VPN et lier les VPN aux VRF.
- Intégrer le LAN : OSPF, BGP, routes statiques, redistribution et prévention des boucles.
- Configurer l'accès direct à Internet avec NAT.
- Mettre en place la haute disponibilité côté LAN (VRRP).

---

## 2. Deux approches de configuration

| Approche | Principe | Statut |
|---|---|---|
| **Modèles** (feature templates et device templates) | Un modèle d'équipement assemble des modèles de fonctionnalité | Historique, très répandu, encore pris en charge |
| **Groupes de configuration** (configuration groups) | Des **profils** composés de **parcelles** (parcels), des variables par équipement, un déploiement par site | Approche récente (versions 20.12 et suivantes), à privilégier pour les nouveaux déploiements (vérifier la disponibilité dans votre version) |
| **CLI add-on / CLI template** | Compléter le modèle ou écrire la configuration en CLI | À réserver aux fonctions non couvertes |

Dans les deux cas, le **Manager pousse** la configuration sur le WAN Edge par NETCONF via la connexion de contrôle. Une configuration **modifiée localement** en CLI sur un équipement géré est **écrasée** ou provoque un état « hors synchronisation ».

---

## 3. Modèles de fonctionnalité et d'équipement

### 3.1 Hiérarchie

```text
Modèle d'équipement (Device Template : un par "rôle", par exemple Agence-2-transports)
 |-- Informations système       : System, Logging, NTP, AAA, OMP, Security, BFD, Banner
 |-- Transport et gestion       : VPN 0 (Transport), interfaces de transport, VPN 512
 |-- Service                    : VPN 10 (Corp), VPN 20 (Invités), interfaces LAN, OSPF, BGP, VRRP
 |-- Modèles complémentaires    : CLI add-on, SNMP, ...
 `-- Politiques                 : Localisée, Sécurité, ...
```

### 3.2 Principaux modèles de fonctionnalité

| Modèle | Usage |
|---|---|
| **System** | Nom d'hôte, system IP, site ID, port-hop, nom d'organisation |
| **OMP** | Paramètres OMP, redistribution vers OMP |
| **BFD** | Intervalles par couleur, polling du routage applicatif |
| **Security** | Chiffrement des tunnels, rekey, anti-rejeu |
| **Logging / NTP / AAA / Banner** | Gestion |
| **VPN** (0, 512, service) | Segments, routes statiques, DNS, services |
| **VPN Interface Ethernet / Cellular / DSL** | Interfaces : adresse, tunnel-interface, couleur, NAT, QoS |
| **OSPF / BGP / EIGRP** | Routage côté service |
| **VRRP** | Dans le modèle d'interface de service |
| **CLI add-on** | Commandes CLI supplémentaires |

### 3.3 Cycle de travail

1. Créer les modèles de fonctionnalité (avec variables).
2. Les assembler dans un modèle d'équipement.
3. **Attach Devices** : choisir les équipements, saisir ou importer les variables (CSV).
4. **Preview** (aperçu de configuration), **Config Diff** (différences).
5. **Configure Devices** : la tâche est exécutée, les équipements sont mis à jour.

---

## 4. Variables et fichiers de paramètres

Dans un modèle, une valeur peut être **globale** (identique pour tous), **par équipement** (variable) ou par défaut.

Exemples de noms de variables :

| Variable | Exemple |
|---|---|
| `//system/host-name` | `BR1-E1` |
| `//system/system-ip` | `10.255.1.100` |
| `//system/site-id` | `100` |
| `vpn0_inet_if_ip` | `203.0.113.110/24` |
| `vpn0_inet_next_hop` | `203.0.113.1` |
| `vpn10_if_ip` | `192.168.10.1/24` |
| `vpn20_if_ip` | `192.168.20.1/24` |

### 4.1 Fichier CSV de variables (importable)

```csv
csv-deviceId,csv-deviceIP,csv-host-name,//system/host-name,//system/system-ip,//system/site-id,vpn0_inet_if_ip,vpn0_inet_next_hop,vpn0_mpls_if_ip,vpn10_if_ip,vpn20_if_ip
C8K-66666666-7777-8888-9999-000000000000,10.255.1.100,BR1-E1,BR1-E1,10.255.1.100,100,203.0.113.110/24,203.0.113.1,192.0.2.110/24,192.168.10.1/24,192.168.20.1/24
C8K-11111111-2222-3333-4444-555555555555,10.255.0.1,HUB-E1,HUB-E1,10.255.0.1,10,203.0.113.101/24,203.0.113.1,192.0.2.101/24,10.1.10.1/24,10.1.20.1/24
```

> Le format exact des en-têtes est celui produit par l'export du Manager (bouton de téléchargement du CSV lors de l'attachement). Partez toujours **d'un export** puis complétez-le.

### 4.2 Conventions de nommage (à fixer dès le départ)

| Objet | Convention | Exemple |
|---|---|---|
| Modèle de fonctionnalité | `<TYPE>-<ROLE>-<VARIANTE>` | `VPN0-INET-DHCP`, `VPN10-CORP-OSPF` |
| Modèle d'équipement | `DT-<ROLE>-<MODELE>` | `DT-BRANCH-C8300-2WAN` |
| Variable | `<vpn>_<objet>_<attribut>` | `vpn10_if_ip` |
| Site ID | par plage de rôle | hubs 10-99, agences 100-999 |
| System IP | `10.255.<région>.<n>` | `10.255.1.100` |

---

## 5. Groupes de configuration et groupes de politiques

### 5.1 Concepts

```text
Groupe de configuration (Configuration Group) : "Agences-2-transports"
 |-- Profil Système                (System Profile)      : system, OMP, BFD, logging, AAA
 |-- Profil Transport et gestion   (Transport & Management Profile) : VPN 0, interfaces WAN, VPN 512
 |-- Profil Service                (Service Profile)     : VPN de service, interfaces LAN, routage
 |-- Profil CLI add-on             (CLI Profile)
 `-- Profil Autre                  (Other Profile)       : NTP, SNMP, etc.

Groupe de politiques (Policy Group) : applications, QoS, sécurité, DNS, AAR, DIA associés à un groupe de configuration
```

| Terme | Définition |
|---|---|
| **Parcelle** (parcel) | Unité de configuration (par exemple « interface Ethernet de transport »). Contient des valeurs fixes ou des **variables** |
| **Profil** | Ensemble de parcelles d'un domaine |
| **Groupe de configuration** | Assemblage de profils applicable à des équipements |
| **Variables** | Valeurs spécifiques par équipement (adresses, noms) |
| **Associer** (associate) puis **Déployer** (deploy) | Associer des équipements au groupe, renseigner les variables, déployer |
| **Groupe de politiques** | Regroupe politiques d'application, QoS, sécurité au niveau d'un équipement |

### 5.2 Cycle de travail

1. Créer les **profils** et leurs parcelles.
2. Créer le **groupe de configuration** et y rattacher les profils.
3. **Associer** les équipements (par site, par rôle) ; renseigner les variables (formulaire ou CSV).
4. **Deploy** ; vérifier l'état de tâche et la synchronisation.
5. Modifier une parcelle : **le changement s'applique à tous les équipements** du groupe après redéploiement.

---

## 6. Comparaison : modèles et groupes de configuration

| Critère | Modèles | Groupes de configuration |
|---|---|---|
| Granularité | Modèle d'équipement monolithique | Profils et parcelles modulaires |
| Réutilisation | Moyenne (copies de modèles) | Meilleure (profils partagés) |
| Variables | Fichier de variables par modèle | Variables par équipement, formulaire guidé |
| Politiques locales et sécurité | Politique séparée (localisée) | Groupes de politiques intégrés |
| Visibilité de l'état | Tâches de poussée | Déploiement par groupe |
| Prise en charge | Large (historique) | Versions récentes, en extension |
| Migration | — | Outils de conversion partielle (voir la documentation) |
| Automatisation (API) | Matures | Plus récentes |

> **Conseil** : nouveau déploiement sur une version récente : groupes de configuration. Parc existant : maintenir les modèles et planifier la migration par lots.

---

## 7. Interfaces de transport

### 7.1 Configuration CLI d'une interface de transport (indicative)

```text
interface GigabitEthernet1
 description TRANSPORT-INTERNET
 ip address 203.0.113.110 255.255.255.0
 negotiation auto
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 203.0.113.1
!
sdwan
 interface GigabitEthernet1
  tunnel-interface
   encapsulation ipsec
   color biz-internet
   allow-service dhcp
   allow-service dns
   allow-service icmp
   no allow-service sshd
   no allow-service netconf
   allow-service ntp
   allow-service stun
   no allow-service bgp
   no allow-service ospf
  exit
 exit
```

### 7.2 Paramètres clés d'une interface de transport

| Paramètre | Rôle | Remarque |
|---|---|---|
| **Couleur** | Nom du transport | Cohérent dans toute la fabrique |
| **Restrict** | Tunnels seulement avec la même couleur | |
| **Encapsulation** | `ipsec` (par défaut) ou `gre` | GRE pour interopérer avec d'anciens équipements ou services |
| **Allow-service** | Services autorisés **sur l'interface de transport** (réponse à ping, DHCP, DNS, SSH…) | **Réduire** au strict nécessaire |
| **Bande passante amont et aval** | Pour la QoS et le calcul d'utilisation | |
| **Lissage** (shaping) | Limiter le débit en sortie au débit du contrat | Voir SW07, CP02 |
| **Dernier recours** (`last-resort-circuit`) | Interface (par exemple 4G) utilisée **seulement** quand les autres tombent | Économie de volume |
| **Préférence de connexion** (`vmanage-connection-preference`) | Interface préférée pour la connexion au Manager | |
| **Nombre de connexions de contrôle** | `max-control-connections` : limite par interface | Équilibrage |
| **Port hopping** | Changement de port source pour contourner des blocages | |
| **NAT-refresh** | Fréquence des paquets de maintien de NAT | |
| **MTU** | MTU de l'interface physique et du tunnel | `ip tcp adjust-mss` côté service |

### 7.3 Interface cellulaire (4G/5G) en dernier recours

```text
interface Cellular0/2/0
 ip address negotiated
 no shutdown
!
sdwan
 interface Cellular0/2/0
  tunnel-interface
   encapsulation ipsec
   color lte
   last-resort-circuit
   allow-service dhcp
   allow-service dns
   allow-service icmp
  exit
 exit
```

Comportement : avec `last-resort-circuit`, l'interface ne monte ses tunnels que lorsque **tous les autres transports** sont indisponibles.

---

## 8. Segmentation par VPN

### 8.1 Plan de segmentation d'exemple

| VPN | Nom | Usage | VRF | Route Target (exemple) |
|---|---|---|---|---|
| 0 | Transport | Underlay, contrôle | Table globale | — |
| 512 | Gestion | Administration hors bande | `Mgmt-intf` | — |
| 10 | CORP | Utilisateurs | 10 | 1:10 |
| 20 | GUEST | Invités | 20 | 1:20 |
| 30 | IOT | Objets connectés | 30 | 1:30 |
| 40 | VOICE | Téléphonie IP | 40 | 1:40 |

### 8.2 Configuration IOS XE

```text
vrf definition 10
 description CORP
 rd 1:10
 address-family ipv4
  route-target export 1:10
  route-target import 1:10
 exit-address-family
!
vrf definition 20
 description GUEST
 rd 1:20
 address-family ipv4
  route-target export 1:20
  route-target import 1:20
 exit-address-family
!
interface GigabitEthernet3
 description LAN-CORP
 vrf forwarding 10
 ip address 192.168.10.1 255.255.255.0
 ip tcp adjust-mss 1350
 no shutdown
!
interface GigabitEthernet4
 description LAN-GUEST
 vrf forwarding 20
 ip address 192.168.20.1 255.255.255.0
 ip tcp adjust-mss 1350
 no shutdown
```

> **Important** : même numéro de VPN sur tous les sites pour un même segment (le VPN 10 est la CORP partout) : l'**identifiant du VPN est global** à la fabrique.

### 8.3 Isolation et communication entre segments

Par défaut, **aucune fuite** de routes entre VPN. Si besoin, trois mécanismes :

| Mécanisme | Description | Remarque |
|---|---|---|
| **Pare-feu externe** | Les segments se rejoignent sur un pare-feu (service chaining) | Le plus sûr et auditable |
| **Politique de contrôle** | Fuite de routes entre VPN par politique centralisée | Avancé (SW07) |
| **Fuite locale** (VRF leaking par `route-target`) | Import d'un autre VPN sur un même équipement | À éviter, difficile à auditer |

---

## 9. Routage côté service

### 9.1 Principes

Le WAN Edge est un routeur : côté **service** (LAN), il échange des routes avec le réseau local ; côté **overlay**, il les annonce dans **OMP**. Il faut donc :

1. apprendre les routes LAN (OSPF, BGP, statique, EIGRP) ;
2. les **injecter dans OMP** pour les autres sites ;
3. éventuellement injecter les routes OMP dans les protocoles LAN (pour que le LAN apprenne les réseaux distants).

```text
Réseau local  <-- OSPF -->  WAN Edge (VRF 10)  <-- OMP -->  Controllers  <-- OMP -->  Autres WAN Edge
          (routes LAN)      redistribute omp                advertise ospf
```

### 9.2 OSPF côté service

```text
router ospf 10 vrf 10
 router-id 192.168.10.1
 auto-cost reference-bandwidth 10000
 redistribute omp subnets route-map OMP-TO-OSPF
 passive-interface default
 no passive-interface GigabitEthernet3
 network 192.168.10.0 0.0.0.255 area 0
!
sdwan
 omp
  address-family ipv4
   advertise ospf external
  exit
 exit
```

Vérifications :

```text
show ip ospf neighbor
show ip route vrf 10 ospf
show sdwan omp routes vpn 10
show ip route vrf 10 omp
```

### 9.3 BGP côté service

```text
router bgp 65001
 bgp router-id 10.255.1.100
 bgp log-neighbor-changes
 address-family ipv4 vrf 10
  redistribute omp route-map OMP-TO-BGP
  neighbor 192.168.10.2 remote-as 65100
  neighbor 192.168.10.2 activate
  neighbor 192.168.10.2 route-map BGP-IN in
  neighbor 192.168.10.2 route-map BGP-OUT out
 exit-address-family
!
sdwan
 omp
  address-family ipv4
   advertise bgp
```

> **Particularité** : en iBGP, les routes apprises par OMP ne sont pas toujours redistribuées dans BGP sans précautions (règle de l'iBGP). Utilisez `redistribute omp` avec une `route-map`, et si nécessaire `neighbor ... route-reflector-client` ou un AS dédié par site selon la conception.

### 9.4 Route statique côté service

```text
ip route vrf 10 172.16.0.0 255.255.0.0 192.168.10.254
!
sdwan
 omp
  address-family ipv4
   advertise static
```

### 9.5 Annoncer le réseau connecté

```text
sdwan
 omp
  address-family ipv4
   advertise connected
```

> **Bonne pratique** : n'annoncer que ce qui est nécessaire (filtrer par route-policy), éviter de propager des routes par défaut non voulues dans l'overlay.

---

## 10. Prévention des boucles et redistribution

### 10.1 Le risque

Si **deux sites** (ou deux équipements d'un même site) redistribuent mutuellement OMP → OSPF/BGP → OMP, un préfixe peut **tourner en boucle** ou prendre un chemin sous-optimal.

```text
Hub A  --OMP-->  Hub B
 | OSPF            | OSPF
 +---- LAN commun -+      Une route apprise par OMP sur A est redistribuée en OSPF,
                          réapprise par B comme "route OSPF" puis redistribuée en OMP : boucle.
```

### 10.2 Balises (tags) et filtres

```text
! 1) Marquer ce qui vient d'OMP lorsqu'on le redistribue dans OSPF
route-map OMP-TO-OSPF permit 10
 set tag 777
!
! 2) Refuser dans OMP ce qui porte la balise 777 (déjà passé par OMP)
route-map OSPF-TO-OMP deny 10
 match tag 777
route-map OSPF-TO-OMP permit 20
!
router ospf 10 vrf 10
 redistribute omp subnets route-map OMP-TO-OSPF
```

Côté injection dans OMP, appliquez la route-policy `OSPF-TO-OMP` à `advertise ospf` (dans le modèle OMP ou dans la politique de redistribution de l'interface de service).

### 10.3 Variante BGP : communautés

```text
ip community-list standard FROM-SDWAN permit 65001:777
route-map OMP-TO-BGP permit 10
 set community 65001:777 additive
route-map BGP-TO-OMP deny 10
 match community FROM-SDWAN
route-map BGP-TO-OMP permit 20
```

### 10.4 Autres règles

| Règle | Pourquoi |
|---|---|
| Un seul point de redistribution OMP ↔ protocole LAN par site, ou balises systématiques | Éviter les boucles |
| Résumer les préfixes vers OMP | Diminuer la table |
| Préférer une **route par défaut** depuis le hub vers les rayons | Limiter les tables sur petits sites |
| Distances et coûts maîtrisés | Éviter les routes asymétriques |

---

## 11. Accès direct à Internet (DIA)

### 11.1 Principe

Au lieu de renvoyer le trafic Internet vers un concentrateur par l'overlay, le site le sort **localement** par une interface de transport, avec **NAT**.

```text
Poste VPN 20 --> WAN Edge --(NAT)--> Interface de transport (VPN 0) --> Internet
```

### 11.2 Méthode 1 : route statique « globale » + NAT (CLI)

```text
! Liste des sources à traduire (réseau des invités)
ip access-list standard NAT-GUEST
 permit 192.168.20.0 0.0.0.255
!
! NAT overload sur l'interface de transport, pour la VRF 20
ip nat inside source list NAT-GUEST interface GigabitEthernet1 vrf 20 overload
!
interface GigabitEthernet1
 ip nat outside
!
interface GigabitEthernet4
 ip nat inside
!
! Route par défaut du VPN 20 vers la table globale (VPN 0)
ip route vrf 20 0.0.0.0 0.0.0.0 GigabitEthernet1 203.0.113.1 global
```

La route `... global` fait « fuiter » le trafic du VPN 20 vers la table globale ; le NAT assure le retour.

### 11.3 Méthode 2 : politique de données centralisée (`nat use-vpn 0`)

Une **politique de données** peut sélectionner le trafic (par application ou destination) et l'envoyer directement par le VPN 0 avec NAT (voir SW07) :

```text
data-policy DP-DIA
 vpn-list VPN-GUEST
  sequence 10
   match
    source-ip 0.0.0.0/0
   action accept
    nat use-vpn 0
    nat fallback
```

`nat fallback` renvoie le trafic via l'overlay si aucune interface de sortie n'est disponible.

### 11.4 Choix de la méthode

| Critère | Route + NAT (locale) | Politique de données |
|---|---|---|
| Simplicité | Simple, statique | Souple (par application) |
| Granularité | Par VPN | Par flux, application, destination |
| Bascule (fallback) | Manuelle ou suivi | `nat fallback` |
| Gestion centralisée | Via modèle | Via politique centralisée |

### 11.5 Vérifications

```text
show ip nat translations vrf 20
show ip nat statistics
show ip route vrf 20 0.0.0.0
```

> **Sécurité** : le DIA sans protection est dangereux. Associer pare-feu, filtrage DNS et URL (SW08) ou redirection vers un service de sécurité cloud (SW12).

---

## 12. Haute disponibilité côté LAN

### 12.1 Deux équipements de bordure

```text
           LAN (VLAN 10)
        +-------+-------+
        |               |
   +----+----+     +----+----+
   | Edge 1  |     | Edge 2  |   VRRP : IP virtuelle 192.168.10.1
   | prio 110|     | prio 100|   Edge 1 maître, Edge 2 secours
   +----+----+     +----+----+
        |               |
   Transport 1     Transport 2
```

### 12.2 Configuration VRRP (IOS XE)

Edge 1 (maître), forme standard de la configuration :

```text
interface GigabitEthernet3
 vrf forwarding 10
 ip address 192.168.10.2 255.255.255.0
 vrrp 10 address-family ipv4
  address 192.168.10.1 primary
  priority 110
  preempt
 exit-vrrp
```

> Dans le Manager, le modèle d'interface de service propose les paramètres VRRP et une option de suivi (**Track OMP**) qui **abaisse la priorité** de l'équipement quand sa session OMP tombe, afin que l'autre prenne le relais. Utilisez-la pour éviter qu'un équipement « aveugle » vis-à-vis de l'overlay reste maître. Le nom exact de l'option dépend de la version.

Edge 2 : adresse `192.168.10.3`, `priority 100`.

Vérification :

```text
show vrrp brief
show vrrp 10
```

### 12.3 TLOC extension

Si l'un des équipements n'a qu'un transport et l'autre un autre, on peut **partager** l'interface de transport de l'un avec l'autre via une **extension de TLOC** (lien L2/L3 entre les deux routeurs) afin que chacun ait accès aux deux transports. Option avancée à étudier dans la documentation de conception.

### 12.4 Points de vigilance

| Point | Explication |
|---|---|
| **Chemin retour** | Si le maître VRRP ne reçoit pas le retour de l'overlay (asymétrie), le trafic échoue |
| **Préemption** | Activer la préemption pour que le maître reprenne la main |
| **Routage** | Les deux équipements annoncent le LAN dans OMP (priorité de route pour préférer l'un) |
| **Équilibrage** | Utiliser plusieurs groupes VRRP (maître alterné par VLAN) |

---

## 13. Génération de variables par script

Pour une centaine de sites, générer le CSV par script évite les erreurs.

```python
#!/usr/bin/env python3
"""Génère un CSV de variables pour N agences à partir d'un plan d'adressage simple.

Plan :
  site_id     : 100 + n
  system-ip   : 10.255.<n // 250>.<(n % 250) + 1>
  LAN CORP    : 10.<100 + n // 250>.<n % 250>.0/24 (passerelle .1)
  LAN GUEST   : 10.<200 + n // 250>.<n % 250>.0/24 (passerelle .1)
  Internet    : 203.0.113.<10 + n>/24  (laboratoire ; en production, adresses de l'opérateur)
"""
import csv
import ipaddress

NB_SITES = 20
ENTETE = ["csv-deviceId", "csv-deviceIP", "csv-host-name", "//system/host-name",
          "//system/system-ip", "//system/site-id", "vpn0_inet_if_ip", "vpn0_inet_next_hop",
          "vpn10_if_ip", "vpn20_if_ip"]

with open("variables-agences.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.writer(f)
    w.writerow(ENTETE)
    for n in range(1, NB_SITES + 1):
        site_id = 100 + n
        sys_ip = f"10.255.{n // 250}.{(n % 250) + 1}"
        lan_corp = ipaddress.ip_network(f"10.{100 + n // 250}.{n % 250}.0/24")
        lan_guest = ipaddress.ip_network(f"10.{200 + n // 250}.{n % 250}.0/24")
        host = f"BR{site_id}-E1"
        w.writerow([
            f"UUID-A-REMPLACER-{n:03d}", sys_ip, host, host, sys_ip, site_id,
            f"203.0.113.{10 + n}/24", "203.0.113.1",
            f"{lan_corp.network_address + 1}/24", f"{lan_guest.network_address + 1}/24",
        ])
print("variables-agences.csv généré")
```

> Remplacez `UUID-A-REMPLACER-xxx` par les identifiants de châssis réels (export de la WAN Edge List) et alignez les en-têtes sur l'export du Manager.

Contrôle de non-chevauchement des adresses :

```python
import csv, ipaddress
reseaux = []
for ligne in csv.DictReader(open("variables-agences.csv")):
    for cle in ("vpn10_if_ip", "vpn20_if_ip"):
        reseaux.append((ligne["//system/host-name"], ipaddress.ip_interface(ligne[cle]).network))
for i, (h1, n1) in enumerate(reseaux):
    for h2, n2 in reseaux[i + 1:]:
        if n1.overlaps(n2):
            print("CHEVAUCHEMENT :", h1, n1, "<->", h2, n2)
print("Contrôle terminé")
```

---

## 14. Lab guidé T3

### 14.1 Objectif

Déployer **trois sites** (un hub et deux agences) avec **deux segments** (CORP VPN 10, GUEST VPN 20), **OSPF côté LAN**, **accès direct à Internet** pour les invités et **VRRP** sur une agence.

### 14.2 Topologie et plan

```text
              +------------------+ Hub (site 10)
              |     HUB-E1       | VPN10: 10.1.10.0/24 (OSPF area 0 avec cœur)
              +--------+---------+
                       |  overlay (biz-internet + mpls)
        +--------------+---------------+
        |                              |
+-------+--------+            +--------+--------+
| BR1-E1 (100)   |            | BR2-E1 (101)    | + BR2-E2 (VRRP, site 101)
| VPN10 192.168.10.0/24       | VPN10 192.168.11.0/24
| VPN20 192.168.20.0/24 (DIA) | VPN20 192.168.21.0/24 (DIA)
+----------------+            +-----------------+
```

| Site | Équipement | System IP | VPN 10 | VPN 20 |
|---|---|---|---|---|
| 10 | HUB-E1 | 10.255.0.1 | 10.1.10.1/24 | — |
| 100 | BR1-E1 | 10.255.1.100 | 192.168.10.1/24 | 192.168.20.1/24 |
| 101 | BR2-E1 | 10.255.1.101 | 192.168.11.2/24 (VRRP .1) | 192.168.21.1/24 |
| 101 | BR2-E2 | 10.255.1.102 | 192.168.11.3/24 (VRRP .1) | — |

### 14.3 Étapes

| # | Action | Vérification |
|---|---|---|
| 1 | Créer les modèles de fonctionnalité (ou profils) : System, Logging, OMP, BFD, Security, VPN 0, interfaces de transport (2 couleurs), VPN 512, VPN 10, VPN 20, OSPF, interface LAN VPN 10 et VPN 20 | Modèles listés |
| 2 | Assembler en modèle d'équipement `DT-BRANCH` et `DT-HUB` (ou groupes de configuration) | Aperçu de configuration sans erreur |
| 3 | Préparer le CSV de variables (script de la section 13 adapté) | Contrôle de chevauchement OK |
| 4 | Attacher les équipements, vérifier **Config Diff**, déployer | Tâche réussie |
| 5 | Configurer OSPF entre le LAN simulé et chaque WAN Edge ; ajouter `advertise ospf external` | `show ip ospf neighbor` |
| 6 | Vérifier la table OMP et la connectivité CORP : PC1 → serveur du hub | `show sdwan omp routes vpn 10`, ping |
| 7 | Mettre en place le DIA du VPN 20 (méthode route + NAT) | PC invité atteint 8.8.8.8 ; `show ip nat translations vrf 20` |
| 8 | Vérifier l'**isolation** : un PC invité ne peut pas joindre un poste CORP | Ping échoue |
| 9 | Configurer VRRP sur BR2-E1/BR2-E2 ; couper BR2-E1 | `show vrrp brief` ; le trafic continue via BR2-E2 |
| 10 | Documenter | Captures et tableaux |

### 14.4 Résultats attendus (extraits)

```text
BR1-E1# show ip route vrf 10
O E2     10.1.10.0/24 [110/20] via 192.168.10.254      (si le hub est vu par OSPF local)
m        10.1.10.0/24 [251/0]  via 10.255.0.1, 00:12:01, Sdwan-system-intf    (route OMP)
...
BR1-E1# show ip nat translations vrf 20
Pro Inside global      Inside local       Outside local      Outside global
icmp 203.0.113.110:1   192.168.20.100:1   8.8.8.8:1          8.8.8.8:1
```

Les routes OMP apparaissent dans la table de la VRF avec le code **`m`** (et une distance administrative de 251).

### 14.5 Grille

| Critère | Points |
|---|---|
| Modèles/groupes corrects, variables propres | 5 |
| OSPF et OMP fonctionnels (reachability CORP) | 5 |
| Segmentation prouvée (isolation GUEST/CORP) | 3 |
| DIA du VPN 20 fonctionnel | 3 |
| VRRP et bascule démontrée | 3 |
| Documentation | 1 |

---

## 15. Dépannage

| Symptôme | Causes probables | Vérifications |
|---|---|---|
| Équipement « out of sync » | Modification locale en CLI | Config Diff ; redéployer |
| Poussée de modèle en échec | Variable manquante, syntaxe, contrainte de version | Message d'erreur de la tâche |
| Routes LAN absentes dans OMP | `advertise ospf` manquant, filtre, VPN différent | `show sdwan omp routes`, route-policy |
| Routes OMP absentes dans le LAN | `redistribute omp` manquant, route-map | `show ip route vrf N` |
| Boucle ou instabilité de routes | Redistribution mutuelle sans balise | Balises, `show ip route vrf N tag` |
| DIA sans effet | Route `global` manquante, NAT non configuré, ACL incorrecte | `show ip nat translations`, route VRF |
| VRRP maître des deux côtés | Pas de communication L2 entre les deux, ID différent | `show vrrp`, VLAN |
| Tunnel absent | Couleur `restrict` incompatible | `show sdwan omp tlocs` |

---

## 16. Exercices

1. Vous avez 300 agences identiques à 2 transports. Comment structurez-vous modèles ou groupes de configuration ? Quelles variables par équipement ?
2. Écrivez la route-map qui empêche qu'une route apprise par OMP et redistribuée en BGP revienne dans OMP (communauté `65001:777`).
3. Expliquez pourquoi une route par défaut dans OMP depuis plusieurs sites peut être dangereuse.
4. Un équipement de site est en VRRP maître mais n'a plus de connexion OMP. Que se passe-t-il pour les utilisateurs ? Comment y remédier ?
5. Pourquoi le VPN de service doit-il porter le même numéro sur tous les sites pour un même segment ?

<details>
<summary>Corrigés</summary>

1. Un seul modèle d'équipement (ou groupe) `BRANCH-2WAN` ; variables : nom d'hôte, system-ip, site-id, adresses de transport (inet, mpls) et passerelles, adresses LAN par VPN, éventuellement paramètres VRRP. Import par CSV généré par script.
2. `ip community-list standard FROM-SDWAN permit 65001:777` ; `route-map BGP-TO-OMP deny 10` + `match community FROM-SDWAN` ; `route-map BGP-TO-OMP permit 20`.
3. Plusieurs sites annonçant `0.0.0.0/0` créent un détournement du trafic Internet vers des sites non prévus et des chemins asymétriques ; il faut décider un ou deux sites de sortie et filtrer.
4. Les utilisateurs envoient leur trafic vers une passerelle qui ne peut plus l'acheminer dans l'overlay (trou noir). Remède : suivi OMP (abaisse la priorité VRRP) ou suivi d'interface pour céder la main.
5. Parce que l'identifiant de VPN est la clé d'association de segments entre sites : routes et étiquettes OMP sont rattachées au numéro de VPN.

</details>

---

## 17. Quiz

1. Quelle commande affiche les routes OMP du VPN 10 ?
2. Quel code porte une route OMP dans la table de routage d'une VRF IOS XE ?
3. À quoi sert `nat fallback` ?
4. Quel est l'avantage principal des groupes de configuration sur les modèles ?
5. Que fait `last-resort-circuit` ?
6. Pourquoi marquer les routes redistribuées ?

<details>
<summary>Réponses</summary>

1. `show sdwan omp routes vpn 10`. 2. `m`. 3. Il renvoie le trafic par l'overlay si la sortie locale est indisponible. 4. Modularité, réutilisation, variables guidées, politiques intégrées. 5. Utilise l'interface seulement quand tous les autres transports sont hors service. 6. Pour détecter et refuser les routes qui ont déjà traversé l'autre protocole (prévention des boucles).

</details>

---

## 18. Aide-mémoire

```text
VPN 0 = transport | VPN 512 = gestion | autres = service (VRF)
OMP <- LAN :  advertise connected|static|ospf|bgp           LAN <- OMP :  redistribute omp [subnets]
Boucles    :  tag/communauté sur OMP->LAN ; refuser ce tag sur LAN->OMP
DIA        :  ip nat inside source list X interface Gi1 vrf N overload  + ip route vrf N 0.0.0.0 0.0.0.0 Gi1 <gw> global
              ou data-policy "nat use-vpn 0" (+ nat fallback)
VRRP       :  vrrp N address-family ipv4 / address <vip> primary / priority / preempt  (+ Track OMP dans le modèle)
Commandes  :  show ip route vrf N | show sdwan omp routes vpn N | show ip ospf neighbor | show vrrp brief | show ip nat translations
```

## 19. Pour aller plus loin

- Guides de configuration Cisco Catalyst SD-WAN (Systems and Interfaces, Routing) pour votre version : <https://www.cisco.com/c/en/us/solutions/enterprise-networks/sd-wan/index.html>
- Documentation des groupes de configuration (Configuration Groups) de votre version.

---

## Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **Feature template** | Modèle d'une fonction (interface, OSPF, NTP…) |
| **Device template** | Modèle d'équipement assemblant des feature templates |
| **Configuration group** | Ensemble de profils de configuration associé à des équipements |
| **Parcel (parcelle)** | Unité de configuration d'un profil |
| **Profil** | Ensemble de parcelles d'un domaine (système, transport, service…) |
| **Variable** | Valeur propre à chaque équipement renseignée au déploiement |
| **CLI add-on** | Commandes CLI ajoutées à un modèle pour les fonctions non couvertes |
| **Service VPN / VRF** | Segment de routage de l'overlay |
| **Service-side** | Côté LAN du WAN Edge |
| **Transport-side** | Côté WAN (VPN 0) |
| **Route leaking** | Fuite contrôlée de routes entre VRF |
| **Tag de route** | Marque utilisée pour filtrer ou éviter les boucles |
| **TLOC extension** | Partage d'un transport entre deux WAN Edge d'un site |
| **Config drift** | Écart entre la configuration voulue et celle présente sur l'équipement |

### B. Cas d'usage détaillés

#### Cas 1. Chaîne de 300 agences identiques

| Rubrique | Détail |
|---|---|
| **Objectif** | Un seul modèle, 300 déploiements sans copier-coller |
| **Méthode** | Un groupe de configuration `BRANCH-2WAN` ; variables = nom d'hôte, system-ip, site-id, adresses WAN, adresses LAN |
| **Génération** | CSV produit par script depuis la base d'inventaire |
| **Contrôles** | Vérification de chevauchement d'adresses, aperçu de configuration sur 3 sites témoins |
| **Leçon** | La **qualité des données d'entrée** (inventaire, adressage) détermine la qualité du déploiement ; automatisez la génération et la validation |

#### Cas 2. Quatre segments avec des règles différentes

| Segment | VPN | Règle |
|---|---|---|
| CORP | 10 | Accès aux applications internes, sortie Internet filtrée |
| VOICE | 40 | Priorité stricte, jamais de sortie Internet |
| IOT | 30 | Isolé, accès à un seul serveur de collecte |
| GUEST | 20 | Sortie Internet locale, aucun accès interne |

**Leçon** : écrivez la matrice « qui parle à qui » avant de configurer ; elle devient le contrat de sécurité de la segmentation.

#### Cas 3. Deux hubs en DC avec OSPF : boucle de redistribution

Le centre de données possède deux WAN Edge, tous deux redistribuant OMP vers OSPF et OSPF vers OMP. Après une bascule, des préfixes « tournent » entre les hubs. Correction : balises (tag 777) sur OMP→OSPF, refus de ce tag sur OSPF→OMP ; un seul équipement est le **point d'injection préféré**. **Leçon** : sur un site à deux routeurs de bordure, concevez la redistribution **avant** de câbler.

#### Cas 4. Le gérant branche le Wi-Fi invité sur le VLAN caisse

Un switch mal configuré met les invités dans le VLAN CORP. Conséquence : un client accède aux caisses. Parade : ports du switch en configuration standard, 802.1X ou VLAN fixe par port, contrôles de conformité (SW17). **Leçon** : la segmentation SD-WAN est inutile si l'**accès LAN** laisse tout passer.

#### Cas 5. Migration des modèles vers les groupes de configuration

Parc de 450 équipements gérés par 12 modèles d'équipement. Approche : (1) inventaire des variantes réelles, (2) création de 4 groupes de configuration couvrant 95 % des cas, (3) migration par lots de 20 sites, avec comparaison de configuration avant/après, (4) conservation des 5 % restants en CLI add-on. **Leçon** : migrer une **fonctionnalité à la fois** et comparer, plutôt que tout reconstruire.

### C. Retours d'expérience

1. **Une variable mal nommée coûte des heures.** Imposez une convention (`vpn10_if_ip`) et refusez les variantes.
2. **N'éditez pas en CLI un équipement géré.** Au prochain déploiement, vos changements disparaissent, ou pire, l'équipement reste « hors synchronisation ».
3. **Lisez toujours l'aperçu (preview) de configuration.** Il révèle les erreurs de variable avant qu'elles ne partent sur 300 sites.
4. **Le VRRP sans suivi OMP crée des trous noirs.** Un équipement peut être maître sans plus voir l'overlay.
5. **Testez les routes par défaut.** Une route `0.0.0.0/0` annoncée par erreur dans OMP détourne tout le trafic.
6. **Documentez les VPN.** Tableau (numéro, nom, usage, propriétaire, règles) maintenu sous contrôle de version.
7. **Un segment de plus est une règle de plus.** Ne segmentez pas par principe, segmentez par **risque**.

### D. Exercices supplémentaires

1. Produisez le tableau des variables d'un modèle de **site à deux routeurs en VRRP** (variables communes et variables par équipement).
2. Écrivez la route-map OMP→BGP et la route-map BGP→OMP pour une agence connectée à un CE opérateur en eBGP.
3. Un invité accède au VPN CORP. Listez 6 causes possibles, de la plus probable à la plus rare, et la commande pour confirmer chacune.
4. Dessinez la matrice « qui parle à qui » des quatre segments du cas 2.

<details>
<summary>Corrigés</summary>

1. Communes : site-id, VPN, plan VRRP (groupe, VIP) ; par équipement : nom d'hôte, system-ip, adresse physique de l'interface LAN, priorité VRRP, adresses de transport.
2. OMP→BGP : `set community 65001:777 additive` ; BGP→OMP : `deny` si communauté 65001:777, sinon `permit`.
3. VLAN du port, DHCP invité distribuant une plage CORP, trunk avec VLAN natif, VRF de l'interface LAN, fuite de routes (route-target), politique de contrôle qui injecte des routes ; confirmer avec `show vlan`, `show ip interface brief vrf`, `show ip route vrf`, `show run | section vrf`.
4. CORP→Internet (filtré) ; CORP→VOICE : non ; IOT→serveur de collecte : oui ; GUEST→Internet : oui ; tout le reste : non.

</details>

### E. Mini-projet (6 heures)

Construisez un **catalogue de modèles/groupes** pour une entreprise de 5 types de sites (siège, DC, grande agence, petite agence, site 4G) avec : arbre des profils, liste de variables, conventions de nommage, matrice de segmentation et procédure de déploiement d'un nouveau site. Validez-le sur votre lab en déployant au moins trois types.

---

> Module précédent : [SW05](../SW05-deploiement-controleurs-equipements/SW05-deploiement-controleurs-equipements.md) | Module suivant : [SW07](../SW07-politiques-centralisees-localisees/SW07-politiques-centralisees-localisees.md)
