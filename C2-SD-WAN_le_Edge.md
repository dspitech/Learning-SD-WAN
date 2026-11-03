# Cours SD-WAN : le Edge et sa séquence d'intégration

Suite du cours "Cours complet SD-WAN : les 4 plans". Ce document se concentre sur le **Edge**, l'équipement de bordure du SD-WAN, depuis son rôle et son anatomie jusqu'à sa **séquence d'intégration** (onboarding) complète, illustrée par le fil rouge d'un groupe de distribution inspiré de Carrefour.

> Avertissement : le scénario Carrefour est un cas pédagogique fictif. Les chiffres, noms, adresses et choix techniques sont inventés pour illustrer les concepts et ne décrivent pas l'architecture réelle de l'entreprise. Le vocabulaire et les commandes sont ceux de l'écosystème Cisco Catalyst SD-WAN (ex-Viptela), donné à titre de référence. Les ports, minuteurs, codes d'erreur et chemins de menu varient selon la version : toujours confirmer dans la documentation du produit utilisé.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Définition du Edge](#2-définition-du-edge)
3. [Le Edge dans les 4 plans](#3-le-edge-dans-les-4-plans)
4. [Les familles de Edge et leur choix](#4-les-familles-de-edge-et-leur-choix)
5. [Anatomie d'un Edge](#5-anatomie-dun-edge)
6. [Transport, couleur et TLOC en détail](#6-transport-couleur-et-tloc-en-détail)
7. [Les identités du Edge](#7-les-identités-du-edge)
8. [La séquence d'intégration du Edge](#8-la-séquence-dintégration-du-edge)
9. [Variantes d'intégration](#9-variantes-dintégration)
10. [Exemple Carrefour : intégration de bout en bout](#10-exemple-carrefour--intégration-de-bout-en-bout)
11. [Après l'intégration : Day-0, Day-1, Day-2](#11-après-lintégration--day-0-day-1-day-2)
12. [Le plan de données sur le Edge](#12-le-plan-de-données-sur-le-edge)
13. [Services de sécurité et réseau sur le Edge](#13-services-de-sécurité-et-réseau-sur-le-edge)
14. [Haute disponibilité du Edge](#14-haute-disponibilité-du-edge)
15. [Edge dans le cloud public](#15-edge-dans-le-cloud-public)
16. [Configuration d'exemple](#16-configuration-dexemple)
17. [Supervision et dépannage du Edge](#17-supervision-et-dépannage-du-edge)
18. [Exercices et corrigés](#18-exercices-et-corrigés)
19. [Bonnes pratiques et erreurs fréquentes](#19-bonnes-pratiques-et-erreurs-fréquentes)
20. [Glossaire](#20-glossaire)
21. [Fiche de révision](#21-fiche-de-révision)

---

## 1. Objectifs pédagogiques

À la fin de ce cours, vous serez capable de :

- donner une définition précise du Edge et de ses rôles (spoke, hub, passerelle) ;
- expliquer et calculer les notions de transport, de couleur et de TLOC (nombre de tunnels, BFD, MTU, disponibilité) ;
- situer le Edge par rapport aux 4 plans et expliquer ce qu'il **exécute** et ce qu'il **consomme** ;
- choisir un type de Edge selon le profil d'un site ;
- décrire l'anatomie logique d'un Edge : VPN 0, VPN 512, VPN de service, interfaces de tunnel, TLOC, couleurs ;
- expliquer le rôle de chaque **identité** (certificat, numéro de série, chassis number, system-IP, site-id, organization-name) ;
- dérouler de mémoire la **séquence d'intégration** d'un Edge, phase par phase, avec les protocoles, les ports et les points de contrôle ;
- préparer un projet de déploiement (prérequis, checklist, pilote) ;
- diagnostiquer un échec d'intégration en identifiant la phase en cause ;
- configurer et exploiter un Edge au quotidien.

---

## 2. Définition du Edge

### 2.1 Définition

Le **Edge**, appelé aussi **WAN Edge** ou **routeur de bordure SD-WAN**, est l'équipement (physique, virtuel ou logiciel) placé à la **frontière entre le réseau local (LAN) d'un site et les transports WAN** de ce site. Il **termine l'overlay SD-WAN** : c'est lui qui entre dans le réseau SD-WAN, construit les tunnels chiffrés et fait circuler le trafic des utilisateurs du site.

Définition courte à retenir :

> Le Edge est le point d'entrée et de sortie d'un site dans l'overlay SD-WAN.

Quatre verbes pour le retenir :

1. **Joindre** : il se rattache aux contrôleurs (Validator, Manager, Controllers) par des sessions de contrôle authentifiées.
2. **Annoncer** : il informe le plan de contrôle des réseaux du site (préfixes), de ses transports (TLOC) et de ses clés de chiffrement.
3. **Construire** : il établit les tunnels IPsec vers les autres Edges et mesure leur qualité en continu.
4. **Appliquer** : il exécute les politiques (segmentation, choix du chemin, QoS, sécurité) sur chaque paquet.

### 2.2 Les trois "côtés" d'un Edge

Un Edge a toujours trois façons de communiquer :

```
                  COTE CONTROLE
        (sessions DTLS/TLS vers Validator, Manager, Controllers)
                           |
                           |
  COTE LAN             +---+---+            COTE TRANSPORT
  (VPN de service)     |       |            (VPN 0)
  caisses, serveurs,---| EDGE  |----  MPLS | Internet | 5G
  Wi-Fi, caméras, IoT  |       |      tunnels IPsec vers les autres Edges
                       +-------+
```

| Côté | Ce qui y circule | VPN concerné |
|---|---|---|
| **LAN** | Trafic des utilisateurs et des équipements du site | VPN de service (1 à 511) |
| **Transport** | Trafic encapsulé et chiffré, sessions de contrôle, sondes BFD | VPN 0 |
| **Contrôle** | Authentification, configuration, routes OMP, clés | Sessions de contrôle établies depuis le VPN 0 |

### 2.3 Les six fonctions du Edge

| N° | Fonction | Détail |
|---|---|---|
| 1 | **Raccordement aux transports** | Termine les liens WAN : MPLS, fibre, xDSL, 4G/5G |
| 2 | **Rattachement à l'overlay** | S'authentifie, reçoit sa configuration, maintient ses sessions de contrôle |
| 3 | **Segmentation et routage** | Sépare les flux en VPN, route localement (statique, OSPF, BGP) et annonce ses préfixes en OMP |
| 4 | **Tunnels chiffrés** | Établit les tunnels IPsec, chiffre et déchiffre le trafic |
| 5 | **Politiques et qualité** | Mesure latence, gigue et perte (BFD), choisit le chemin selon le SLA, applique la QoS |
| 6 | **Services locaux** | NAT et sortie Internet directe, DHCP, pare-feu, filtrage, selon la plateforme |

### 2.4 Edge, routeur classique, pare-feu et Controller

| Critère | Routeur WAN classique | Edge SD-WAN | Pare-feu | Controller |
|---|---|---|---|---|
| Configuration | Par équipement, en CLI | Par modèles depuis le Manager | Par équipement ou gestionnaire dédié | Pas d'équipement de site |
| Choix du chemin | Table de routage statique ou dynamique | Selon l'application et la qualité mesurée | Pas son rôle principal | Définit la politique, ne transporte pas |
| Tunnels | Configurés un par un | Construits automatiquement selon la topologie | Parfois, site à site | Aucun tunnel de données |
| Trafic utilisateur | Oui | Oui | Oui | Non |
| Intelligence centralisée | Non | Oui, reçue du plan de contrôle | Partielle | Oui, c'est son rôle |

### 2.5 Vocabulaire autour du Edge

| Terme | Signification |
|---|---|
| **WAN Edge** | Terme générique pour l'équipement de bordure SD-WAN |
| **cEdge** | Edge fonctionnant sur une plateforme IOS XE (Catalyst 8000, ISR, ASR) |
| **vEdge** | Edge de la génération historique Viptela |
| **Edge virtuel** | Instance logicielle sur hyperviseur ou dans un cloud public |
| **CPE** | Customer Premises Equipment : équipement installé chez le client, terme plus général |
| **uCPE** | CPE universel pouvant héberger plusieurs fonctions réseau virtualisées |
| **Branch** | Site distant (magasin, agence) |

### 2.6 Les rôles d'un Edge dans la topologie

| Rôle | Description | Exemple Carrefour |
|---|---|---|
| **Spoke** | Edge d'un site distant, relié aux hubs | Magasin de proximité, supermarché |
| **Hub** | Edge qui agrège les tunnels de nombreux spokes | Edge d'un datacenter |
| **Hub régional** | Hub intermédiaire qui limite les distances et les charges | Plateforme logistique régionale |
| **Passerelle Internet ou sécurité** | Edge qui concentre la sortie Internet filtrée | Edge du siège avec inspection avancée |
| **Passerelle cloud** | Edge virtuel qui relie l'overlay à un cloud public | Edge virtuel dans le cloud de l'ERP |

Un même modèle d'Edge peut jouer plusieurs rôles : c'est la **politique de contrôle** qui décide de la topologie, pas le matériel.

### 2.7 Idées reçues sur le Edge

- **"Un Edge est un simple routeur VPN."** Non : il est piloté par un plan de contrôle central, mesure la qualité en continu et applique des politiques par application.
- **"Le Edge décide seul de la topologie."** Non : il reçoit ses routes, ses TLOC distants et ses clés du Controller.
- **"Si les contrôleurs tombent, le Edge ne transporte plus rien."** Non : le plan de données continue de fonctionner un temps avec les dernières informations connues.
- **"Un Edge n'a qu'une seule adresse."** Non : il a un **system-IP** logique, plusieurs adresses d'interface et plusieurs **TLOC**.

### 2.8 Exemple Carrefour : le Edge d'un supermarché

Le Edge du supermarché FR-IDF-SUP-0123 (cas fictif) :

- **côté LAN**, il raccorde 5 VPN : caisses (10), back-office (20), Wi-Fi invités (30), vidéosurveillance (40), IoT (50) ;
- **côté transport**, il termine 3 liens : fibre Internet, MPLS, 5G ;
- **côté contrôle**, il entretient des sessions avec les Controllers et le Manager ;
- il porte **3 TLOC**, construit **6 tunnels** (3 vers chacun des 2 hubs) et applique la politique "paiement d'abord sur MPLS, bascule sur Internet si le SLA n'est plus respecté".

Le chapitre 6 détaille précisément les notions de **transport**, de **couleur** et de **TLOC**, avec les calculs associés.

---

## 3. Le Edge dans les 4 plans

Le Edge est le seul composant présent dans **tous les plans** : il est le **client** des plans d'orchestration, de gestion et de contrôle, et le **moteur** du plan de données.

| Plan | Rôle du Edge | Ce qu'il fait concrètement |
|---|---|---|
| Orchestration | Client | Contacte le Validator, s'authentifie, apprend l'adresse des autres contrôleurs |
| Gestion | Client | Reçoit sa configuration et ses mises à jour du Manager, remonte ses statistiques |
| Contrôle | Client | Maintient des sessions de contrôle avec les Controllers, échange des routes, TLOC et clés via OMP |
| Données | Moteur | Construit les tunnels IPsec, chiffre le trafic, mesure la qualité, applique QoS et politiques |

```
                 Orchestration      Gestion        Contrôle
                  (Validator)      (Manager)      (Controller)
                       ^               ^               ^
                       |               |               |
                       +---- sessions de contrôle -----+
                                       |
                                  +----+----+
                                  |  EDGE   |
                                  +----+----+
                                       |
                    tunnels IPsec vers les autres Edges (plan de données)
```

Le Edge reçoit donc ses instructions des trois autres plans, et exécute le travail réel : faire passer le trafic.

---

## 4. Les familles de Edge et leur choix

### 4.1 Les formes d'un Edge

| Forme | Description | Usage typique |
|---|---|---|
| **Physique** | Routeur dédié (gamme Catalyst 8000 Edge, ISR, ASR, ou anciens vEdge) | Magasins, entrepôts, sièges, datacenters |
| **Virtuel** | Machine virtuelle sur hyperviseur ou sur un boîtier d'accueil (NFV) | Sites souhaitant consolider plusieurs fonctions |
| **Cloud** | Instance dans un cloud public (par exemple Catalyst 8000V) | Connexion des VPC et VNet à l'overlay |
| **Logiciel sur équipement tiers** | Fonction SD-WAN intégrée à un pare-feu ou à un autre routeur selon l'éditeur | Sites où l'on souhaite un seul équipement |

### 4.2 Critères de choix

- **Débit chiffré réel** nécessaire, et non le débit nominal des ports.
- **Nombre et type de transports** : fibre, MPLS, xDSL, 4G/5G (modem intégré ou externe).
- **Nombre de tunnels** à tenir (important pour les hubs).
- **Services embarqués** : pare-feu, IPS, filtrage URL, Wi-Fi, switching.
- **Haute disponibilité** : un ou deux équipements.
- **Contraintes physiques** : format, refroidissement, alimentation, environnement (réserve de magasin, local technique).
- **Coût total** : matériel, licences, support.

### 4.3 Exemple Carrefour : catalogue de Edges par profil de site

| Profil | Edge (illustratif) | Justification |
|---|---|---|
| Hypermarché | 2 Edges physiques de milieu de gamme, en haute disponibilité | Fort volume, criticité élevée, plusieurs transports |
| Supermarché | 1 Edge physique de milieu de gamme, option de second Edge | Équilibre coût et débit |
| Proximité et franchisé | 1 Edge compact avec modem 4G/5G intégré | Faible coût, installation simple, secours cellulaire |
| Entrepôt | 2 Edges physiques, QoS avancée | Flux WMS et scanners RF sensibles |
| Siège | 2 Edges physiques haut de gamme ou virtuels | Gros débits, nombreux segments |
| Datacenter (hub) | 2 à 4 Edges haut de gamme | Agrégation de milliers de tunnels |
| Cloud | Edges virtuels dans 2 régions cloud | Accès aux applications hébergées |

---

## 5. Anatomie d'un Edge

### 5.1 Les VPN (VRF) internes

Un Edge sépare son fonctionnement en plusieurs **VPN**, c'est-à-dire des instances de routage isolées :

| VPN | Nom | Rôle |
|---|---|---|
| **VPN 0** | Transport | Porte les interfaces WAN, les sessions de contrôle et les tunnels IPsec. Ne contient jamais de trafic utilisateur. |
| **VPN 512** | Gestion | Accès d'administration hors bande (interface de management locale). |
| **VPN 1 à 511** | Service | Réseaux de service côté LAN : un VPN par segment (caisses, back-office, invités, etc.). |

Règles à retenir :

- le VPN 0 et le VPN 512 sont **réservés** et ont des rôles précis ;
- chaque VPN de service possède **sa propre table de routage** ;
- un VPN de service n'a **aucune connectivité** avec un autre VPN sauf si une politique ou une configuration explicite le permet (route leaking, pare-feu).

### 5.2 Les interfaces

- **Interfaces de transport (WAN)** : dans le VPN 0, avec la configuration `tunnel-interface` qui les active comme points d'extrémité de l'overlay. Elles portent une **couleur** et une **encapsulation**.
- **Interfaces de service (LAN)** : rattachées à un VPN de service, elles connectent les équipements du site (caisses, commutateurs, bornes Wi-Fi).
- **Interface de gestion** : dans le VPN 512, facultative.
- **Interfaces cellulaires** : modem 4G/5G intégré ou module, traitées comme des interfaces WAN.
- **Interfaces de bouclage (loopback)** : utilisées pour le **system-IP**, pour certains services de routage ou d'adressage.

### 5.3 Transports, couleurs et TLOC

Ces trois notions, centrales pour comprendre l'overlay, sont détaillées au chapitre 6 avec des exemples et des calculs. En résumé : le **transport** est le lien opérateur, la **couleur** est l'étiquette SD-WAN de ce lien, et le **TLOC** (system-IP, couleur, encapsulation) est l'adresse complète du point d'accès annoncée au plan de contrôle.

### 5.4 Les trois mécanismes qui tournent en permanence

1. **Sessions de contrôle** (DTLS ou TLS) vers les Controllers et le Manager.
2. **OMP** pour échanger routes, TLOC et clés avec les Controllers.
3. **BFD dans chaque tunnel** pour détecter les pannes et mesurer la qualité.

---

## 6. Transport, couleur et TLOC en détail

Trois notions sont indissociables sur un Edge :

| Notion | Ce que c'est | Image |
|---|---|---|
| **Transport** | Le lien réel fourni par un opérateur (MPLS, fibre, 5G) | La route |
| **Couleur** | L'étiquette logique donnée à ce transport dans le SD-WAN | Le type de route (autoroute, route nationale) |
| **TLOC** | L'adresse complète d'un point d'accès au transport, annoncée au réseau | Le numéro de la sortie |

La chaîne complète, de la réalité physique au trafic :

```
Transport (lien opérateur)
   -> interface WAN avec "tunnel-interface"
      -> couleur + encapsulation
         -> TLOC = system-IP + couleur + encapsulation
            -> TLOC annoncé en OMP au Controller
               -> tunnel IPsec entre deux TLOC
                  -> session BFD dans le tunnel (mesure de qualité)
```

### 6.1 Le transport

#### Définition

Un **transport** est un moyen d'acheminer des paquets entre le site et le reste du réseau, fourni par un opérateur. Dans le vocabulaire SD-WAN, c'est la couche **underlay**. Le Edge raccorde un ou plusieurs transports et construit l'**overlay** par-dessus.

#### Propriétés d'un transport

| Propriété | Pourquoi elle compte |
|---|---|
| **Type** : privé (MPLS, Ethernet métropolitain) ou public (Internet, cellulaire) | Détermine l'adresse utilisée pour joindre le site et le niveau de garantie |
| **Débit montant et descendant** | Dimensionnement, shaping, QoS (le montant est souvent le facteur limitant) |
| **Qualité (latence, gigue, perte)** | Définit les SLA que l'on peut promettre aux applications |
| **Coût** | Arbitrage entre transports (MPLS cher, Internet économique) |
| **Adressage et NAT** | Une IP publique dynamique ou un NAT d'opérateur complique la traversée |
| **MTU** | Plus petite sur certains accès (cellulaire, PPPoE) |
| **Plafond de données** | Critique pour la 4G/5G (voir le calcul C4) |
| **Opérateur et chemin physique** | Pour juger de l'indépendance réelle de deux liens |

#### Transport, interface, couleur : à ne pas confondre

- Un **transport** est un service opérateur.
- Une **interface** est un port de l'Edge raccordé à ce service.
- Une **couleur** est l'étiquette SD-WAN que l'on attribue à cette interface.

Un transport correspond en général à une interface et à une couleur, mais deux accès Internet de deux opérateurs distincts sont **deux transports**, qui méritent en général **deux couleurs différentes**.

#### Exemple Carrefour : transports par profil de site

| Profil | Transports | Couleurs choisies |
|---|---|---|
| Hypermarché | MPLS, Internet opérateur A, Internet opérateur B, 5G | `mpls`, `biz-internet`, `custom1`, `lte` |
| Supermarché | MPLS, Internet, 5G | `mpls`, `biz-internet`, `lte` |
| Proximité | Internet, 4G | `biz-internet`, `lte` |
| Entrepôt | MPLS, Internet, 5G | `mpls`, `biz-internet`, `lte` |
| Hub (datacenter) | MPLS, Internet | `mpls`, `biz-internet` |

### 6.2 La couleur

#### Définition

La **couleur** est un **mot-clé normalisé** attribué à une interface de transport. Elle ne change pas le comportement physique du lien : elle **qualifie** le transport pour le plan de contrôle et pour les politiques.

#### À quoi sert la couleur

1. **Qualifier le transport** : le Controller et les politiques savent s'il s'agit d'un transport privé ou public.
2. **Piloter la formation des tunnels** : la couleur et l'option `restrict` déterminent quels TLOC peuvent se parler.
3. **Écrire les politiques** : on exprime "préférer `mpls`", "éviter `lte`", "utiliser les couleurs de la liste X".
4. **Lire les statistiques** : chaque tunnel est identifié par les couleurs de ses deux extrémités.

#### Les couleurs disponibles (écosystème Cisco)

| Classe | Couleurs |
|---|---|
| **Privées** | `metro-ethernet`, `mpls`, `private1` à `private6` |
| **Publiques** | `3g`, `biz-internet`, `blue`, `bronze`, `custom1` à `custom3`, `default`, `gold`, `green`, `lte`, `public-internet`, `red`, `silver` |

#### Couleur privée ou publique : l'adresse utilisée

C'est la différence essentielle entre les deux classes :

| Classe | Adresse utilisée pour joindre le TLOC | Hypothèse implicite |
|---|---|---|
| **Privée** | Adresse **privée** (celle de l'interface, sans NAT) | Le réseau est fermé, les deux sites se voient directement |
| **Publique** | Adresse **publique** vue de l'extérieur (après NAT, apprise grâce au Validator) | Le trafic traverse Internet, des NAT peuvent exister |

Conséquence : un TLOC de couleur privée n'est joignable que par un TLOC qui partage le même réseau privé. D'où l'intérêt de l'option `restrict`.

#### L'option `restrict`

| Configuration | Comportement |
|---|---|
| Couleur **sans** `restrict` | Le TLOC **tente** de monter un tunnel vers **tous** les TLOC distants, quelle que soit leur couleur |
| Couleur **avec** `restrict` | Le TLOC ne monte des tunnels qu'avec des TLOC de **même couleur** |

Bonne pratique : appliquer `restrict` aux couleurs privées, par exemple `color mpls restrict`. Sans cela, le TLOC `mpls` tentera de joindre les TLOC Internet des autres sites, ce qui échoue (adresse privée non routable sur Internet) et consomme des ressources pour rien.

#### Exemple de formation des tunnels

Deux sites A et B, chacun avec trois TLOC : `mpls` (avec `restrict`), `biz-internet`, `lte`.

| TLOC de A | TLOC de B | Tunnel ? | Raison |
|---|---|---|---|
| mpls | mpls | Oui | Même couleur, `restrict` respecté |
| mpls | biz-internet | Non | `mpls` est restreint à sa propre couleur |
| mpls | lte | Non | Idem |
| biz-internet | mpls | Non | Le TLOC `mpls` de B est restreint |
| biz-internet | biz-internet | Oui | Deux couleurs publiques |
| biz-internet | lte | Oui | Deux couleurs publiques |
| lte | mpls | Non | Le TLOC `mpls` de B est restreint |
| lte | biz-internet | Oui | Deux couleurs publiques |
| lte | lte | Oui | Deux couleurs publiques |

Total : **5 tunnels** possibles entre A et B (calcul C2).

#### Bonnes pratiques de coloration

- **Une couleur par type de transport**, nommée selon le rôle et non selon l'opérateur.
- **Deux Internet = deux couleurs** (par exemple `biz-internet` et `custom1`) si l'on veut les distinguer dans les politiques.
- **`lte` pour le cellulaire**, afin de pouvoir le traiter comme transport de secours dans les politiques.
- **Cohérence globale** : la même couleur doit désigner le même type de transport sur tous les sites.
- **`restrict` sur toutes les couleurs privées**.
- **Documenter** la table des couleurs dans le dossier d'architecture.

### 6.3 Le TLOC

#### Définition

Le **TLOC (Transport Locator)** identifie un **point d'accès au transport** d'un Edge. Il est défini par trois éléments :

```
TLOC = ( system-IP , couleur , encapsulation )
Exemple : ( 10.255.10.123 , biz-internet , ipsec )
```

| Élément | Rôle |
|---|---|
| **System-IP** | Identifie l'Edge dans l'overlay |
| **Couleur** | Identifie le transport |
| **Encapsulation** | Format du tunnel : `ipsec` (par défaut dans la plupart des déploiements) ou `gre` |

Un Edge porte **un TLOC par interface de transport activée** (par encapsulation). Un site avec trois transports a trois TLOC.

#### Les attributs d'un TLOC annoncé en OMP

Quand l'Edge annonce son TLOC au Controller, il y joint des informations :

| Attribut | Rôle |
|---|---|
| **Adresse privée et port** | Adresse réelle de l'interface |
| **Adresse publique et port** | Adresse vue de l'extérieur après NAT (apprise via le Validator) |
| **Preference** | Priorité du TLOC : plus elle est élevée, plus le TLOC est préféré (0 par défaut) |
| **Weight** | Poids pour répartir la charge entre TLOC de même préférence (1 par défaut) |
| **Site-ID** | Site auquel appartient le TLOC |
| **Tag** | Marqueur pour les politiques |
| **Clés de chiffrement** | Éléments pour établir les tunnels IPsec |
| **Carrier** | Indication sur le type de transport utilisée par certaines fonctions |
| **État BFD** | Utilisable ou non |

#### Lien entre routes et TLOC : le "next hop" de l'overlay

Une **route OMP** (un préfixe LAN d'un site) ne pointe pas vers une adresse IP classique, mais vers un **TLOC**. Le TLOC joue le rôle du **next hop** de BGP.

```
Préfixe 10.10.123.0/24 (VPN 10)
      |
      +-- next hop : TLOC ( 10.255.10.123 , mpls , ipsec )          preference 200
      +-- next hop : TLOC ( 10.255.10.123 , biz-internet , ipsec )  preference 100
      +-- next hop : TLOC ( 10.255.10.123 , lte , ipsec )           preference 10
```

Pour qu'une route soit **utilisable**, il faut que son TLOC soit **joignable**, c'est-à-dire qu'au moins un **tunnel** vers ce TLOC soit actif (BFD up). Si le tunnel tombe, la route correspondante n'est plus utilisée et le trafic bascule sur le TLOC suivant.

#### Chaîne de résolution

| Étape | Question | Réponse |
|---|---|---|
| 1 | Quel est le préfixe de destination ? | Table de routage du VPN |
| 2 | Quel est le TLOC next hop ? | Route OMP |
| 3 | Le TLOC est-il joignable ? | Tunnel BFD up vers ce TLOC |
| 4 | Quel tunnel et quelle interface locale ? | Tunnel IPsec entre TLOC local et TLOC distant |

#### La TLOC extension

La **TLOC extension** permet à un Edge d'utiliser le transport de son voisin sur le même site :

- l'Edge B n'a pas de lien MPLS, mais l'Edge A en a un ;
- une liaison interne relie A et B ;
- B annonce alors un TLOC `mpls` **qui sort physiquement par A**.

Cela évite de dupliquer les abonnements opérateur sur les deux Edges d'un site en haute disponibilité (chapitre 14).

#### Cycle de vie d'un TLOC

1. L'interface de transport est configurée avec `tunnel-interface`.
2. Le Edge établit sa session de contrôle et **annonce** son TLOC en OMP.
3. Les autres Edges reçoivent ce TLOC et, selon les politiques, **montent des tunnels** vers lui.
4. Si le lien tombe, le TLOC est **retiré** ou marqué inutilisable ; les routes qui le référencent cessent d'être utilisées.
5. Au retour du lien, le TLOC est ré-annoncé et les tunnels sont reconstruits.

### 6.4 Les calculs essentiels

> Les valeurs par défaut et les tailles de paquets cités dans les calculs sont des **hypothèses pédagogiques**. Elles varient selon la plateforme, la version et la configuration : toujours les confirmer sur l'équipement et dans la documentation de l'éditeur. Les calculs sont reproductibles avec vos propres valeurs.

#### C1. Nombre de TLOC d'un site

```
Nombre de TLOC = nombre d'Edges x nombre d'interfaces de transport par Edge
                 (x nombre d'encapsulations activées par interface)
```

Exemple : un hypermarché avec 2 Edges, chacun avec 3 transports (mpls, biz-internet, lte) en IPsec :

```
2 Edges x 3 transports = 6 TLOC
```

#### C2. Nombre de tunnels entre deux sites

Chaque **paire de TLOC compatibles** forme un tunnel (et une session BFD). Avec `restrict` sur la couleur privée :

```
Tunnels(A,B) = nombre de paires (TLOC de A, TLOC de B) compatibles
```

Pour les deux sites A et B du tableau de la section 6.2 :

```
Paires possibles sans aucune restriction : 3 x 3 = 9
Paires compatibles avec restrict sur mpls : 5
  (mpls-mpls, biz-biz, biz-lte, lte-biz, lte-lte)
Tentatives inutiles évitées : 9 - 5 = 4, soit 44 % des tentatives
```

**Maillage complet entre N sites** (chaque paire de sites a 5 tunnels) :

```
Tunnels totaux = 5 x N x (N - 1) / 2
Pour N = 5 000 sites : 5 x 5 000 x 4 999 / 2 = 62 487 500 tunnels
```

Ce chiffre montre pourquoi on ne fait **pas** de maillage complet à grande échelle : on utilise le **hub-and-spoke** ou un maillage partiel.

**Hub-and-spoke.** Hub avec 2 TLOC (`mpls` restreint et `biz-internet`), spoke avec 3 TLOC (`mpls` restreint, `biz-internet`, `lte`) :

```
Paires compatibles hub-spoke : mpls-mpls, biz-biz, biz-lte = 3
Un spoke relié à 2 hubs : 3 x 2 = 6 tunnels
Un hub qui reçoit 5 000 spokes : 5 000 x 3 = 15 000 tunnels par hub
```

On retrouve les **6 tunnels** du supermarché de la section 2.8, et la charge de **15 000 tunnels** que le hub doit supporter.

#### C3. Coût des sondes BFD sur les liens et sur le hub

Hypothèses : un paquet BFD fait environ 150 octets sur le fil (en-têtes et chiffrement compris), envoyé toutes les 1 seconde dans chaque sens.

```
Débit par tunnel et par sens = 150 octets x 8 bits / 1 s = 1 200 bit/s = 1,2 kbit/s
```

**Côté hub** (15 000 tunnels) :

```
Débit BFD par sens = 15 000 x 1,2 kbit/s = 18 000 kbit/s = 18 Mbit/s
Paquets par seconde = 15 000 paquets/s dans chaque sens
```

C'est modeste en débit, mais cela représente un **trafic permanent** et une charge de traitement à prendre en compte dans le dimensionnement du hub.

**Côté spoke** (6 tunnels) :

```
Débit BFD par sens = 6 x 1,2 kbit/s = 7,2 kbit/s
```

#### C4. Détection de panne et volume de données sur la 5G

**Temps de détection d'une panne de tunnel :**

```
Temps de détection = intervalle hello x multiplicateur
Par défaut : 1 000 ms x 7 = 7 s
```

**Compromis réactivité, consommation et stabilité** (tableau pour un tunnel) :

| Hello | Multiplicateur | Détection | Débit BFD par sens | Remarque |
|---|---|---|---|---|
| 500 ms | 3 | 1,5 s | 2,4 kbit/s | Très réactif, risque de bascules intempestives sur lien bruité |
| 1 000 ms | 7 | 7 s | 1,2 kbit/s | Valeur par défaut |
| 5 s | 3 | 15 s | 0,24 kbit/s | Adapté à un lien cellulaire |
| 10 s | 7 | 70 s | 0,12 kbit/s | Très économe, détection lente |

**Volume sur un lien 5G.** Le spoke a 2 tunnels qui passent par l'interface `lte` (un vers chaque hub). Dans les deux sens :

```
Débit total = 2 tunnels x 2 sens x 150 octets/s = 600 octets/s
Par jour  : 600 x 86 400 = 51 840 000 octets, soit environ 51,8 Mo
Par mois  : 51,8 Mo x 30 = environ 1 555 Mo, soit 1,56 Go
```

Avec un hello de **10 s** au lieu de 1 s :

```
1,56 Go / 10 = environ 155 Mo par mois
```

Conclusion : sur un forfait de données limité, **allonger l'intervalle hello** sur l'interface cellulaire réduit fortement le coût des sondes, au prix d'une détection plus lente.

#### C5. Mesure de qualité applicative : résolution, fenêtre et réactivité

Le routage applicatif s'appuie sur les mesures BFD regroupées par **intervalles de collecte** (poll-interval), puis moyennées sur plusieurs intervalles (multiplicateur).

Valeurs par défaut usuelles (à confirmer) : poll-interval de 600 s (10 min) et multiplicateur de 6.

```
Sondes par intervalle = poll-interval / hello = 600 s / 1 s = 600
Résolution de la mesure de perte = 1 / 600 = 0,1667 %
Fenêtre de moyenne = 600 s x 6 = 3 600 s = 1 heure
```

**Exemple de moyenne glissante.** Perte mesurée dans les 6 derniers intervalles : 0 %, 0 %, 0 %, 0 %, 0 %, 7 % :

```
Perte moyenne = (0 + 0 + 0 + 0 + 0 + 7) / 6 = 1,17 %
Avec un SLA de perte maximale à 1 % : violation détectée
```

Mais la violation n'est constatée qu'à la **fin de l'intervalle** de 10 minutes concerné : une dégradation peut passer jusqu'à 10 minutes avant d'être prise en compte. C'est pourquoi on **réduit** souvent le poll-interval. Il existe néanmoins une limite :

```
Poll-interval 30 s et hello 1 s : 30 sondes par intervalle
Résolution = 1 / 30 = 3,33 % par sonde perdue
```

Une seule sonde perdue représente alors 3,33 % de perte : le SLA à 1 % de perte devient **inutilisable** (trop sensible aux événements isolés). Il faut donc choisir un couple (hello, poll-interval) qui conserve une **résolution suffisante** :

```
Résolution (%) = 100 x hello / poll-interval
```

| Hello | Poll-interval | Sondes | Résolution | Commentaire |
|---|---|---|---|---|
| 1 s | 600 s | 600 | 0,17 % | Précis, réaction lente |
| 1 s | 120 s | 120 | 0,83 % | Bon compromis courant |
| 1 s | 30 s | 30 | 3,33 % | Trop grossier pour un SLA de perte à 1 % |

#### C6. MTU, surcoût IPsec et MSS

**Surcoût d'encapsulation** (hypothèse : IPsec ESP en mode tunnel, AES-256-GCM, encapsulation UDP) :

| Composant | Octets |
|---|---|
| En-tête IP externe | 20 |
| En-tête UDP | 8 |
| En-tête ESP (SPI et numéro de séquence) | 8 |
| Vecteur d'initialisation (IV) | 8 |
| Marqueurs de fin (longueur de bourrage et en-tête suivant) | 2 |
| Bourrage (de 0 à 3) | jusqu'à 3 |
| Valeur d'intégrité (ICV) | 16 |
| **Total** | **62 à 65 octets** |

On retient **65 octets** (cas défavorable). Certaines versions ajoutent quelques octets propres à l'overlay : conserver une marge et lire la **MTU de tunnel réelle** sur l'équipement.

```
MTU de tunnel = MTU du lien - surcoût
Fibre (MTU 1 500) : 1 500 - 65 = 1 435
Cellulaire (MTU 1 428 par hypothèse) : 1 428 - 65 = 1 363

MSS TCP = MTU de tunnel - 40 (en-têtes IP et TCP)
Fibre : 1 435 - 40 = 1 395
Cellulaire : 1 363 - 40 = 1 323
```

Une valeur de MSS unique et prudente pour tous les transports (par exemple 1 350) simplifie l'exploitation.

**Part du surcoût selon la taille du paquet :**

| Taille du paquet utile | Surcoût de 65 octets | Expansion |
|---|---|---|
| 64 octets | 65 / 64 | environ 102 % |
| 200 octets | 65 / 200 | 32,5 % |
| 576 octets | 65 / 576 | environ 11,3 % |
| 1 400 octets | 65 / 1 400 | environ 4,6 % |

**Exemple : un appel voix G.711.** Paquet RTP toutes les 20 ms, soit 50 paquets par seconde :

```
Paquet sans IPsec : 160 (voix) + 12 (RTP) + 8 (UDP) + 20 (IP) = 200 octets
Débit sans IPsec : 200 x 8 x 50 = 80 000 bit/s = 80 kbit/s
Paquet avec IPsec : 200 + 65 = 265 octets
Débit avec IPsec : 265 x 8 x 50 = 106 000 bit/s = 106 kbit/s
Augmentation : 106 / 80 = 1,325 soit + 32,5 %
```

Pour dimensionner la voix, il faut donc compter **106 kbit/s par appel** (hors en-têtes de niveau 2) et non 80.

#### C7. Répartition de charge et préférence entre TLOC

**Préférence (preference).** Entre plusieurs TLOC, le plus préféré est utilisé tant qu'il est disponible et conforme au SLA.

Exemple pour le paiement : TLOC `mpls` preference 200, `biz-internet` preference 100, `lte` preference 10. Tout le trafic de paiement utilise `mpls`. Si `mpls` tombe, il passe sur `biz-internet`, puis sur `lte` en dernier recours.

**Poids (weight).** Entre TLOC de **même** préférence, le trafic est réparti proportionnellement au poids :

```
Part du TLOC i = poids(i) / somme des poids
Exemple : biz-internet poids 3, custom1 poids 1
biz-internet : 3 / (3 + 1) = 75 %
custom1      : 1 / (3 + 1) = 25 %
```

Usage typique pour un hypermarché avec deux accès Internet de capacités différentes (300 Mbit/s et 100 Mbit/s) : poids 3 et 1, soit une répartition **75 % et 25 %** proportionnelle aux débits.

#### C8. Dimensionnement de la bande passante d'un transport

```
Débit nécessaire = (somme des flux) x (1 + surcoût moyen) x (1 + marge)
```

Exemple pour le lien MPLS du supermarché (flux de paiement et de back-office) :

```
Paiement : 20 caisses x 50 kbit/s = 1 Mbit/s
Back-office : 30 postes x 0,5 Mbit/s = 15 Mbit/s
Somme des flux = 16 Mbit/s

Surcoût moyen IPsec (hypothèse, mélange de tailles de paquets) : 12 %
Marge de croissance et de pointe : 30 %

Débit nécessaire = 16 x 1,12 x 1,3 = 23,3 Mbit/s
Débit à commander : 30 Mbit/s (palier supérieur)
```

**Shaping.** On configure le shaper de l'Edge **un peu en dessous** du débit contractuel pour que la file d'attente soit gérée chez nous et non chez l'opérateur :

```
Shaper = débit contractuel x 0,9
Lien de 100 Mbit/s : shaper à 90 Mbit/s
```

**Vérification d'un flux volumineux.** La vidéosurveillance de 24 caméras en 2 Mbit/s :

```
24 x 2 Mbit/s = 48 Mbit/s
```

Ce flux n'a pas sa place sur le MPLS à 30 Mbit/s : il est dirigé vers l'accès Internet (100 Mbit/s en montant, soit 48 % de sa capacité), avec une classe de QoS de faible priorité pour ne pas gêner le reste.

#### C9. Disponibilité cumulée de plusieurs transports

Si les transports sont **indépendants**, la disponibilité totale se calcule à partir des indisponibilités :

```
A totale = 1 - (1 - A1) x (1 - A2) x ... x (1 - An)
```

| Configuration | Calcul | Disponibilité | Indisponibilité par an |
|---|---|---|---|
| Fibre seule (99,5 %) | 1 - 0,005 | 99,5 % | 0,005 x 525 600 min = 2 628 min, soit 43,8 h |
| Fibre (99,5 %) + 5G (99,0 %) | 1 - 0,005 x 0,01 = 1 - 0,00005 | 99,995 % | 0,00005 x 525 600 = 26,3 min |
| Fibre + 5G + MPLS (99,9 %) | 1 - 0,005 x 0,01 x 0,001 | 99,999995 % | environ 1,6 s |

**Limites du calcul :**

- l'**indépendance** est une hypothèse : deux liens dans le même fourreau, chez le même opérateur ou sur la même alimentation tombent ensemble ;
- chaque incident coûte le **temps de bascule** (détection BFD de 7 s par défaut, plus la convergence). Avec par exemple 12 incidents par an et 8 s de bascule :

```
12 x 8 s = 96 s, soit environ 1,6 minute d'interruption par an
```

La disponibilité réelle est donc bornée par le **temps de détection**, d'où l'intérêt de bien régler BFD pour les flux critiques comme le paiement.

#### C10. Taille des tables du plan de contrôle

Pour 5 000 Edges avec 3 TLOC chacun, 5 VPN et 2 préfixes par VPN :

```
Sessions de contrôle vers les Controllers
  = Edges x TLOC x connexions de contrôle par TLOC
  = 5 000 x 3 x 2 = 30 000   (2 connexions par défaut dans la plupart des versions)

Routes OMP de préfixes = 5 000 x 5 x 2 = 50 000
Routes TLOC            = 5 000 x 3     = 15 000
```

Ces chiffres sont à comparer aux **limites de l'éditeur** pour chaque Controller, et déterminent le nombre de Controllers à déployer (par exemple 4 Controllers : 30 000 / 4 = 7 500 sessions en moyenne chacun).

En topologie hub-and-spoke, un spoke ne reçoit **que** les routes autorisées par la politique de contrôle, par exemple celles des hubs : sa table reste petite (de l'ordre de quelques centaines de routes), ce qui allège les petits Edges.

### 6.5 Formules à retenir

| Besoin | Formule |
|---|---|
| Nombre de TLOC d'un site | Edges x transports x encapsulations |
| Tunnels entre deux sites | Nombre de paires de TLOC compatibles (couleurs, `restrict`) |
| Tunnels en maillage complet | (tunnels par paire de sites) x N x (N - 1) / 2 |
| Tunnels sur un hub | Spokes x tunnels par spoke et par hub |
| Débit des sondes BFD | Taille du paquet x 8 / intervalle hello |
| Détection de panne | Hello x multiplicateur |
| Résolution de la mesure de perte | 100 x hello / poll-interval (en %) |
| Fenêtre de mesure applicative | Poll-interval x multiplicateur |
| MTU de tunnel | MTU du lien moins le surcoût IPsec |
| MSS TCP | MTU de tunnel moins 40 |
| Part de trafic ECMP | Poids du TLOC / somme des poids |
| Débit à commander | Flux x (1 + surcoût) x (1 + marge) |
| Disponibilité cumulée | 1 moins le produit des indisponibilités |
| Sessions de contrôle | Edges x TLOC x connexions par TLOC |

### 6.6 Pièges courants

| Piège | Conséquence |
|---|---|
| Oublier `restrict` sur `mpls` | Tentatives de tunnels inutiles, sessions BFD en échec, bruit dans les alarmes |
| Mêmes couleurs sur deux Internet de deux opérateurs | Impossible de les distinguer dans les politiques |
| Mauvaise cohérence des couleurs entre sites | Tunnels absents ou politiques qui ne s'appliquent pas |
| BFD agressif sur du cellulaire | Bascules intempestives et consommation de données |
| Poll-interval trop court avec un hello long | Résolution de perte trop grossière, SLA instable |
| MSS non ajusté | Lenteurs, fragmentation, applications qui bloquent sur certains liens |
| Hypothèse d'indépendance fausse pour la disponibilité | Surestimation de la résilience réelle |
| Maillage complet à grande échelle | Explosion du nombre de tunnels, saturation des Edges |

---

## 7. Les identités du Edge

Un Edge s'identifie par plusieurs éléments qu'il faut bien distinguer. L'intégration repose sur leur cohérence.

| Identité | Origine | Rôle |
|---|---|---|
| **Numéro de série et certificat de l'équipement** | Installés en usine (certificat matériel type SUDI pour les équipements Cisco) | Prouver l'authenticité du matériel |
| **Chassis number (UUID)** | Identifiant unique de l'équipement | Clé de recherche dans les listes d'équipements autorisés |
| **Organization-name** | Configuré par l'entreprise | Doit être **identique** sur tous les composants de l'overlay ; il isole l'entreprise des autres clients |
| **Adresse du Validator** | Configurée ou fournie par le service de découverte | Premier point de contact |
| **System-IP** | Configuré dans le template | Identifiant **logique et unique** de l'équipement dans l'overlay (comme un router-id) |
| **Site-ID** | Configuré dans le template | Identifiant du **site** ; deux Edges du même site partagent le même site-id |
| **Token OTP** | Généré par le Manager pour certains équipements virtuels | Mot de passe à usage unique pour l'authentification initiale |

### 7.1 Ce qu'il faut absolument garantir

- Le **system-IP est unique** dans toute l'entreprise (une collision provoque des sessions instables).
- Le **site-id est cohérent** : même valeur pour deux Edges du même site, valeur différente pour deux sites différents.
- L'**organization-name est strictement identique** (casse comprise) sur le Validator, le Manager, les Controllers et les Edges.
- Les **certificats** sont valides, non révoqués, signés par une autorité de confiance reconnue par tous.
- L'**heure** de l'équipement est correcte (via NTP), sinon les certificats peuvent sembler expirés.

### 7.2 Les états d'autorisation d'un Edge dans le Manager

| État | Signification |
|---|---|
| **Valid** | L'équipement est autorisé à rejoindre l'overlay |
| **Staging** | L'équipement est accepté pour une phase de préparation, sans tunnels de données vers la production |
| **Invalid** | L'équipement est refusé |

Seuls les équipements marqués **Valid** (ou Staging selon le scénario) peuvent établir des sessions de contrôle durables.

---

## 8. La séquence d'intégration du Edge

C'est le coeur de ce document. L'**intégration** (onboarding, ZTP, "Day-0") est la suite d'étapes qui amène un équipement neuf, sorti de son carton, à un état pleinement opérationnel dans l'overlay.

### 8.1 Vue d'ensemble en 9 phases

| Phase | Nom | Plan principal | Résultat attendu |
|---|---|---|---|
| 0 | Préparation côté entreprise | Gestion | Équipement déclaré, autorisé, profil prêt |
| 1 | Démarrage et connectivité de base | Données (accès) | L'Edge a une IP, une passerelle, un DNS, une heure |
| 2 | Découverte du Validator (PnP/ZTP) | Orchestration | L'Edge connaît l'adresse et l'organization-name du Validator |
| 3 | Bascule en mode SD-WAN | Gestion (équipement) | L'Edge est en "controller mode" avec une configuration d'amorçage |
| 4 | Authentification auprès du Validator | Orchestration | Identité vérifiée, liste des contrôleurs reçue |
| 5 | Connexion au Manager | Gestion | Session de contrôle montée avec le Manager |
| 6 | Réception de la configuration | Gestion | Template appliqué, system-IP, site-id, VPN configurés |
| 7 | Connexion aux Controllers et OMP | Contrôle | Routes, TLOC et clés échangés |
| 8 | Montée des tunnels et BFD | Données | Overlay opérationnel, trafic utilisateur possible |

### 8.2 Schéma de la séquence

```
 Edge            DHCP/DNS     Service de      Validator     Manager      Controller
  |                 |         redirection         |            |             |
  |--- DHCP ------->|              |              |            |             |
  |<-- IP,GW,DNS ---|              |              |            |             |
  |--- NTP, DNS (résolution) ----->|              |            |             |
  |                                |              |            |             |
  |--- PnP / ZTP (HTTPS) --------->|              |            |             |
  |<-- adresse Validator, org -----|              |            |             |
  |                                               |            |             |
  |==== (bascule en mode SD-WAN + config d'amorçage) ===                     |
  |                                               |            |             |
  |--- DTLS/TLS + certificat -------------------->|            |             |
  |<-- validation + liste des contrôleurs --------|            |             |
  |    (+ adresse publique vue de l'extérieur)    |            |             |
  |                                               |            |             |
  |--- session de contrôle (DTLS/TLS) ---------------------->  |             |
  |<-- configuration (template) via NETCONF ------------------ |             |
  |                                                            |             |
  |--- session de contrôle (DTLS/TLS) ------------------------------------->|
  |<-- OMP : routes, TLOC, clés, politiques -------------------------------- |
  |                                                                          |
  |<==== tunnels IPsec + BFD vers les autres Edges et les hubs ====>         |
```

### 8.3 Phase 0 : préparation côté entreprise

Avant même que l'équipement soit livré, l'entreprise doit avoir préparé :

1. **Les contrôleurs** : Validator, Manager et Controllers déployés, joignables, avec des certificats valides.
2. **Le portail de découverte** (par exemple le service Plug and Play Connect de l'éditeur) :
   - un **profil de contrôleur** contenant l'adresse du Validator et l'organization-name ;
   - le **rattachement des numéros de série** des équipements achetés à ce profil.
3. **La liste des équipements autorisés** dans le Manager (synchronisation depuis le portail, ou import manuel), avec les chassis numbers et numéros de série.
4. **Le template d'équipement** adapté au profil du site, avec ses variables (system-IP, site-id, adressage LAN, nom d'hôte, etc.).
5. **Les règles de pare-feu et d'accès** côté datacenter pour autoriser les ports de contrôle vers les contrôleurs.
6. **L'accès réseau côté site** : lien Internet avec DHCP actif, ou paramètres IP connus à fournir.

**Point de contrôle.** Dans le Manager, l'équipement apparaît dans la liste des WAN Edges avec l'état **Valid** et un template prêt à être rattaché.

### 8.4 Phase 1 : démarrage et connectivité de base

À la mise sous tension, un équipement sans configuration (état "factory default") :

- active une interface WAN prévue pour l'amorçage (en général un port désigné de la plateforme) en **DHCP** ;
- obtient une adresse IP, une passerelle par défaut, des serveurs **DNS** et parfois un serveur **NTP** ;
- règle son **horloge** (par NTP ou par mécanisme équivalent), car la validation des certificats en dépend.

**Prérequis réseau sur le site :**

| Besoin | Détail |
|---|---|
| Adressage IP | DHCP actif sur l'accès Internet (ou configuration manuelle via console) |
| DNS | Résolution des noms de découverte et du Validator |
| Sortie Internet | HTTPS vers le service de découverte, UDP/TCP de contrôle vers les contrôleurs |
| NTP | UDP 123 vers un serveur de temps |

**Point de contrôle.** L'équipement peut joindre Internet, résoudre les noms et dispose d'une heure correcte.

### 8.5 Phase 2 : découverte du Validator (PnP ou ZTP)

L'équipement ne connaît encore rien de l'entreprise. Il cherche son "lieu de rattachement" :

1. Il contacte un **service de découverte** de l'éditeur (accessible en HTTPS), parfois précédé d'une indication donnée par le serveur DHCP (options dédiées) ou par un nom DNS prédéfini.
2. Il présente son **numéro de série**.
3. Le service identifie l'entreprise propriétaire grâce au portail préparé en phase 0.
4. Il renvoie à l'équipement les informations du profil : **adresse du Validator** et **organization-name**.

> Équipements Viptela historiques (vEdge) : le mécanisme s'appelle **ZTP** et s'appuie sur un service de redirection dédié. Équipements IOS XE : le mécanisme s'appelle **PnP (Plug and Play)**. Le principe est le même.

**Point de contrôle.** L'équipement a reçu l'adresse du Validator et l'organization-name.

### 8.6 Phase 3 : bascule en mode SD-WAN

Les routeurs de la famille IOS XE peuvent fonctionner en deux modes :

- **Mode autonome** (autonomous mode) : routeur classique, configuré en CLI.
- **Mode contrôleur** (controller mode) : routeur piloté par le Manager, avec une configuration SD-WAN.

Selon la plateforme et la version :

- le PnP peut **basculer automatiquement** l'équipement en mode contrôleur et lui fournir une **configuration d'amorçage** (bootstrap) ;
- sinon, la bascule se fait manuellement par une commande de type `controller-mode enable`, suivie d'un redémarrage.

La **configuration d'amorçage** est minimale. Elle contient en général :

- l'adresse du Validator ;
- l'organization-name ;
- l'identité de l'équipement (chassis number, éventuellement token OTP) ;
- l'interface de transport avec `tunnel-interface` dans le VPN 0, pour permettre les sessions de contrôle.

**Point de contrôle.** L'équipement est en mode contrôleur, il dispose d'une configuration d'amorçage et d'une interface de transport prête pour l'overlay.

### 8.7 Phase 4 : authentification auprès du Validator

L'Edge ouvre une session sécurisée (**DTLS** par défaut, sur UDP, ou **TLS** sur TCP selon la configuration) vers le **Validator**. L'authentification est **mutuelle** :

1. **L'Edge présente son certificat** (certificat matériel d'usine ou certificat d'entreprise).
2. Le **Validator vérifie** :
   - que le certificat est signé par une autorité de confiance ;
   - que le numéro de série et le chassis number figurent dans la **liste des équipements autorisés** ;
   - que l'organization-name correspond.
3. **L'Edge vérifie** le certificat du Validator (chaîne de confiance, organization-name).
4. Le Validator renvoie à l'Edge :
   - la **liste des adresses du Manager et des Controllers** ;
   - l'**adresse IP publique et le port** vus de l'extérieur (information essentielle pour franchir un NAT).
5. La session avec le Validator est **fermée** : son rôle est terminé pour cet Edge (il sera recontacté si nécessaire, par exemple après une perte de connexion).

**Point de contrôle.** L'Edge connaît les adresses des autres contrôleurs et sait quelle adresse publique il présente à l'extérieur.

### 8.8 Phase 5 : connexion au Manager

L'Edge ouvre une session de contrôle (DTLS/TLS) vers le **Manager**, avec le même principe d'authentification mutuelle. Cette session servira à :

- signaler l'équipement comme **en ligne** dans la console ;
- recevoir la **configuration** et les mises à jour ;
- remonter l'état, les statistiques et les alarmes.

À ce stade, **si aucun template n'est encore rattaché**, l'Edge reste avec sa configuration d'amorçage. Il apparaît dans le Manager comme équipement connecté **sans configuration complète**. Il n'est pas encore connecté aux Controllers de contrôle.

**Point de contrôle.** L'Edge est "en ligne" dans le Manager (état de connexion actif).

### 8.9 Phase 6 : réception de la configuration

Le template d'équipement est **rattaché** à l'Edge, soit :

- manuellement par un administrateur, après vérification des variables ;
- **automatiquement** si un mécanisme d'attachement automatisé est en place (API, workflow, règle de rattachement).

Le Manager pousse alors la configuration complète, en général par **NETCONF** sur la session de contrôle chiffrée. Elle contient :

- le **system-IP** et le **site-id** ;
- la configuration des **interfaces WAN** (couleurs, encapsulation, services autorisés) ;
- la configuration des **VPN de service** et des interfaces LAN ;
- le **routage local** (statique, OSPF, BGP) ;
- les paramètres de **QoS**, de **sécurité**, de **NAT** ;
- les paramètres d'**OMP** et de **BFD**.

L'Edge applique la configuration, parfois avec un court redémarrage de certains processus. Un **mécanisme de validation** permet un retour automatique à la configuration précédente si la nouvelle configuration rend l'équipement injoignable par le Manager.

**Point de contrôle.** Le Manager indique que le template est **appliqué** et que la **synchronisation de configuration** est correcte.

### 8.10 Phase 7 : connexion aux Controllers et OMP

Avec sa configuration complète (system-IP, site-id, interfaces de transport), l'Edge ouvre des sessions de contrôle vers **un ou plusieurs Controllers**.

1. **Authentification mutuelle**, comme précédemment.
2. Établissement d'une **session OMP**.
3. L'Edge **annonce** :
   - ses **préfixes** par VPN de service (réseaux LAN du site, routes apprises localement) ;
   - ses **TLOC** (un par transport) ;
   - ses **clés de chiffrement** pour les tunnels de données ;
   - ses **éventuels services**.
4. Le Controller **renvoie** :
   - les **routes** des autres sites que la politique autorise cet Edge à connaître ;
   - les **TLOC** des autres Edges ;
   - les **politiques de contrôle** applicables.

**Point de contrôle.** La session OMP est **établie**, des routes sont reçues, les TLOC distants sont connus.

### 8.11 Phase 8 : montée des tunnels et BFD

Le plan de données entre en action :

1. À partir des TLOC et des clés reçues, l'Edge établit des **tunnels IPsec** vers les autres Edges prévus par la topologie (en général les hubs pour un site de type spoke).
2. Dans chaque tunnel, il lance des sessions **BFD** pour détecter les pannes et mesurer **latence, gigue et perte**.
3. Les **politiques de données et de routage applicatif** sont chargées et deviennent actives.
4. Le trafic utilisateur peut circuler : les caisses accèdent aux applications du datacenter, les flux Internet sortent selon la politique.

**Point de contrôle.** Les sessions BFD sont **up**, le site est **joignable** depuis les hubs et peut joindre ses applications.

### 8.12 Tableau récapitulatif des flux et ports

Les numéros ci-dessous sont les valeurs usuelles de l'écosystème Cisco, à vérifier selon la version et la configuration.

| Phase | Flux | Protocole et port usuels |
|---|---|---|
| 1 | DHCP | UDP 67/68 |
| 1 | DNS | UDP/TCP 53 |
| 1 | NTP | UDP 123 |
| 2 | Découverte PnP/ZTP | HTTPS TCP 443 |
| 4 | Edge vers Validator | DTLS UDP 12346 par défaut, ou TLS TCP (port 23456 par défaut) |
| 5 | Edge vers Manager | DTLS UDP 12346 par défaut (port de base), ou TLS TCP |
| 7 | Edge vers Controller | DTLS/TLS, mêmes ports de base |
| 8 | Tunnels IPsec entre Edges | UDP à partir de 12346 (avec décalage de ports selon la configuration), UDP 4500 pour le NAT-T |

> Remarque : les ports de contrôle peuvent être décalés par configuration (port offset) quand plusieurs Edges partagent une même adresse publique derrière un NAT. Les pare-feu amont doivent autoriser la plage utile.

### 8.13 Ce qui peut échouer, phase par phase

| Phase | Symptôme | Causes probables | Vérifications |
|---|---|---|---|
| 0 | L'équipement n'apparaît pas dans le Manager | Numéro de série absent du portail ou non synchronisé | Liste des équipements, état de la synchronisation |
| 1 | Pas d'adresse IP | DHCP indisponible, mauvais port, câble | Voyants, port d'amorçage, console |
| 1 | Pas de résolution de noms | DNS non fourni ou filtré | Test de ping vers IP, vers un nom |
| 1 | Heure incorrecte | NTP bloqué | `show clock`, accès UDP 123 |
| 2 | Aucune redirection reçue | Service de découverte inaccessible, numéro de série inconnu ou mal rattaché | Accès HTTPS sortant, portail |
| 3 | L'équipement reste en mode autonome | Version non supportée, PnP sans bascule automatique | Mode actuel, commande de bascule |
| 4 | Authentification refusée | Chassis number ou série non autorisé, certificat invalide, organization-name différent, heure fausse | Liste autorisée, certificats, org-name, horloge |
| 4 | Session impossible à ouvrir | Pare-feu bloquant les ports de contrôle, adresse du Validator non joignable | Test de connectivité, journaux pare-feu |
| 5 | Session avec le Manager instable | NAT agressif, perte de paquets, MTU | Stabilité du lien, MTU, port offset |
| 6 | Template non appliqué | Variable manquante, conflit de validation, version non compatible | Journal de tâche dans le Manager |
| 6 | Perte de contact après application | Configuration d'interface erronée, retour automatique déclenché | Journaux de retour, configuration d'amorçage |
| 7 | Pas de session avec les Controllers | System-IP en doublon, site-id invalide, couleur ou tunnel-interface manquants | `show sdwan control connections` |
| 7 | Session OMP établie mais aucune route | Politique de contrôle trop restrictive, VPN manquant | `show sdwan omp routes` |
| 8 | Aucun tunnel de données | Ports IPsec bloqués, NAT, restriction de couleurs | `show sdwan bfd sessions` |
| 8 | Tunnels instables | Lien dégradé, MTU, seuils BFD trop agressifs | Statistiques de lien, ajustement des timers |

### 8.14 Codes de raison courants d'une session de contrôle

Les sessions de contrôle qui échouent affichent un code de raison. Quelques exemples fréquemment rencontrés (liste non exhaustive, libellés à confirmer dans la documentation de la version utilisée) :

| Code | Signification générale |
|---|---|
| `DCONFAIL` | Échec de la connexion DTLS/TLS (réseau, pare-feu, NAT) |
| `CRTREJSER` | Le certificat a été rejeté par le pair (par exemple série non autorisée) |
| `SERNTPRES` | Numéro de série absent de la liste autorisée |
| `BIDNTVRFD` | Identifiant de l'équipement non vérifié |
| `NOACTVB` | Aucun Validator actif joignable |
| `VMCRTREV` | Certificat du Manager révoqué |

---

## 9. Variantes d'intégration

### 9.1 Les grandes méthodes

| Méthode | Principe | Quand l'utiliser |
|---|---|---|
| **PnP / ZTP automatique** | L'équipement découvre seul son Validator grâce au service de découverte | Cas général : sites avec DHCP et sortie Internet |
| **Bootstrap manuel (fichier ou CLI)** | Un fichier de configuration d'amorçage généré par le Manager est chargé via console, USB ou bootflash | Sites sans DHCP, réseau restreint, environnements isolés |
| **Pré-staging en atelier** | L'équipement est intégré en atelier, puis expédié avec un état "staging" ou déjà opérationnel | Sites critiques, besoin de validation avant livraison |
| **Token OTP (équipements virtuels)** | Le Manager génère un token à usage unique, saisi sur l'équipement virtuel avec son chassis number | Edges virtuels dans le cloud ou sur hyperviseur |
| **Autorisation par certificat d'entreprise** | Le certificat de l'équipement est émis par l'autorité de certification de l'entreprise | Politique de sécurité imposant une PKI interne |

### 9.2 Variante : site sans DHCP

1. Le Manager génère le **fichier d'amorçage** (configuration minimale avec adresse IP statique, DNS, route par défaut, Validator, organization-name).
2. Un technicien charge ce fichier via la **console** ou une **clé USB**.
3. L'équipement redémarre en mode contrôleur et suit les phases 4 à 8 normalement.

### 9.3 Variante : site avec accès cellulaire uniquement

Pour les magasins de proximité équipés uniquement d'un lien 4G/5G :

- l'interface **cellulaire** joue le rôle d'interface d'amorçage ;
- le PnP fonctionne via le réseau mobile, à condition que la **carte SIM** soit activée et que le **profil APN** soit correct ;
- la couleur de l'interface est `lte` ;
- attention au **forfait de données** et à la **MTU** plus petite sur certains réseaux mobiles.

### 9.4 Variante : Edge derrière un NAT

- Le Validator détecte l'adresse et le port publics et les **communique à l'Edge** (traversée de NAT).
- Un **NAT symétrique** ou très restrictif peut empêcher la création de tunnels directs entre deux Edges ; dans ce cas, les tunnels passent par un hub.
- Plusieurs Edges derrière la même adresse publique nécessitent parfois des **décalages de ports** (port offset).

### 9.5 Variante : Edge virtuel (cloud ou hyperviseur)

1. Le Manager **ajoute l'équipement virtuel** à la liste et génère un **chassis number** et un **token OTP** à durée limitée.
2. L'administrateur **saisit** ces informations sur l'instance (commande d'activation).
3. L'instance contacte le Validator avec ces éléments, s'authentifie, puis les phases 5 à 8 se déroulent comme pour un équipement physique.

### 9.6 Variante : remplacement d'un Edge en panne (RMA)

1. Le Manager permet de **remplacer** l'ancien équipement par le nouveau en conservant son **template et ses variables**.
2. On **invalide** le chassis number de l'ancien équipement, on **autorise** le nouveau.
3. Le nouvel équipement suit la séquence d'intégration et reprend la configuration de l'ancien.
4. Le **system-IP** reste identique, ce qui évite de modifier les politiques.

---

## 10. Exemple Carrefour : intégration de bout en bout

> Rappel : scénario fictif.

### 10.1 Contexte

Un supermarché du groupe, **FR-IDF-SUP-0123**, ouvre dans une ville d'Île-de-France. Il dispose de :

- une fibre Internet professionnelle (couleur `biz-internet`) ;
- un lien MPLS de secours (couleur `mpls`) ;
- un modem 5G intégré (couleur `lte`) ;
- 5 segments : caisses (VPN 10), back-office (VPN 20), Wi-Fi invités (VPN 30), vidéosurveillance (VPN 40), IoT (VPN 50).

### 10.2 Préparation (de J-30 à J-7)

| Jour | Action | Responsable |
|---|---|---|
| J-30 | Commande de l'Edge, de la fibre et de la 5G | Achats et réseau |
| J-21 | Numéro de série rattaché au profil de contrôleur dans le portail de découverte | Réseau |
| J-14 | Synchronisation de la liste d'équipements, état **Valid**, vérification du chassis number | Réseau |
| J-14 | Création du site : site-id 10123, system-IP 10.255.10.123, plan d'adressage LAN | Réseau |
| J-10 | Choix du template "Supermarché", saisie des variables | Réseau |
| J-7 | Validation du plan de bascule MPLS et des fenêtres d'intervention | Réseau, exploitation, magasin |

### 10.3 Checklist avant expédition

| Contrôle | Fait |
|---|---|
| Numéro de série dans le portail, rattaché au bon profil | Oui |
| Équipement à l'état Valid dans le Manager | Oui |
| Template préparé avec toutes les variables | Oui |
| Règles de pare-feu des contrôleurs vérifiées | Oui |
| Instructions de branchement envoyées au magasin (schéma des ports) | Oui |
| Plan de repli (technicien disponible, contact opérateur) | Oui |

### 10.4 Jour J : chronologie type

Les durées sont indicatives.

| Heure | Phase | Événement |
|---|---|---|
| 07 h 00 | 1 | Le gérant branche le port WAN sur la box fibre et met l'Edge sous tension |
| 07 h 03 | 1 | L'Edge obtient une IP par DHCP, résout les noms, règle son horloge |
| 07 h 05 | 2 | Contact du service de découverte, réception de l'adresse du Validator et de l'organization-name |
| 07 h 08 | 3 | Bascule en mode contrôleur, application de la configuration d'amorçage, redémarrage éventuel |
| 07 h 14 | 4 | Authentification auprès du Validator, réception de la liste des contrôleurs |
| 07 h 15 | 5 | Session de contrôle avec le Manager, l'Edge apparaît "en ligne" |
| 07 h 16 | 6 | L'ingénieur réseau (ou l'automatisation) rattache le template, la configuration est poussée |
| 07 h 20 | 7 | Sessions de contrôle avec les Controllers, session OMP, échange de routes et de clés |
| 07 h 22 | 8 | Tunnels IPsec montés vers les deux hubs, BFD actif |
| 07 h 25 | 8 | Les caisses atteignent le datacenter de paiement, la supervision passe au vert |
| 07 h 30 | Validation | Tests de caisse, de Wi-Fi invité, de vidéosurveillance, de sortie Internet |

### 10.5 Validation post-intégration

Contrôles effectués par le NOC :

1. Les **sessions de contrôle** avec le Manager et les Controllers sont **up**.
2. La **session OMP** est établie, des routes sont reçues dans chacun des 5 VPN.
3. Les **tunnels BFD** sont up vers les deux hubs, sur chaque couleur prévue.
4. Un **test d'application** de paiement réussit depuis une caisse.
5. Le **Wi-Fi invité** sort directement sur Internet et ne peut pas joindre le VPN 10.
6. Une **simulation de panne** du lien fibre bascule les flux sur MPLS puis 5G dans le temps attendu.

### 10.6 Cas particulier : hypermarché avec deux Edges

Pour un hypermarché, on intègre les deux Edges **l'un après l'autre** :

1. **Edge A (primaire)** : séquence d'intégration complète, validation.
2. **Edge B (secondaire)** : même séquence, même site-id, system-IP différent.
3. Activation de la **redondance** (VRRP côté LAN, TLOC extension éventuelle entre les deux Edges).
4. **Test de bascule** : arrêt de l'Edge A, vérification que l'Edge B prend le relais, puis retour.

### 10.7 Cas particulier : magasin de proximité en 5G seule

- Fourniture d'un Edge compact avec SIM pré-activée et APN pré-configuré.
- L'intégration est identique mais le **débit** et la **latence** sont plus variables : on resserre les politiques (SLA moins stricts, priorité au paiement).
- Le Manager surveille la **consommation de données** pour éviter les dépassements de forfait.

---

## 11. Après l'intégration : Day-0, Day-1, Day-2

| Phase de vie | Contenu | Exemples |
|---|---|---|
| **Day-0** | Intégration initiale : l'Edge devient joignable et rejoint l'overlay | Séquence décrite au chapitre 8 |
| **Day-1** | Mise en service fonctionnelle : segmentation, politiques, services | Template complet, VPN 10 à 50, routage OSPF, QoS, politiques de SLA |
| **Day-2** | Exploitation continue : supervision, mises à jour, optimisation, incidents | Alertes, montée de version par vagues, tuning des SLA, remplacement d'équipement |

### 11.1 Mises à jour logicielles

- Test en **laboratoire**, puis **pilote** sur quelques sites, puis déploiement par **vagues**.
- Respect des **compatibilités** entre versions du Manager, des Controllers et des Edges (le Manager et les Controllers doivent généralement être plus récents ou égaux aux Edges).
- **Retour arrière** prévu en cas d'anomalie.

### 11.2 Sauvegarde et restauration

- Les configurations des Edges sont **reconstructibles** depuis le Manager : la sauvegarde du Manager est donc critique.
- Conserver une **copie hors ligne** des templates et variables (export, Git).

### 11.3 Fin de vie

- **Invalidation** du chassis number, effacement sécurisé de l'équipement (reset d'usine de la configuration SD-WAN), mise à jour de l'inventaire.

---

## 12. Le plan de données sur le Edge

### 12.1 Chaîne de traitement d'un paquet

Pour un paquet de caisse envoyé vers le datacenter :

```
LAN (VPN 10) -> classification -> politique de données -> choix du TLOC
-> QoS (file d'attente) -> chiffrement IPsec -> envoi sur l'interface WAN choisie
```

Pour un paquet entrant :

```
Interface WAN -> déchiffrement IPsec -> identification du VPN -> routage
-> QoS -> interface LAN
```

### 12.2 Choix du chemin

L'Edge dispose de plusieurs chemins (TLOC) pour une même destination. Il retient ceux qui :

1. sont **disponibles** (BFD up) ;
2. **respectent le SLA** de la classe d'application ;
3. correspondent à la **préférence** de la politique (par exemple MPLS en premier pour le paiement) ;
4. sont répartis selon la **règle d'équilibrage** (ECMP, pondération).

### 12.3 BFD et mesures

- Sondes BFD dans chaque tunnel, avec intervalle et multiplicateur configurables.
- Calcul de **latence, gigue, perte** sur une fenêtre glissante, comparés aux seuils de SLA.
- **Compromis** : des timers très courts détectent plus vite mais peuvent provoquer des bascules intempestives sur des liens bruités (5G par exemple).

### 12.4 QoS

- **Classification** par application ou par marquage DSCP.
- **Files d'attente** : priorité stricte (voix), bande passante garantie (paiement), reste en "best effort".
- **Shaping** à la capacité réelle du lien pour éviter la congestion côté opérateur.

### 12.5 Techniques d'amélioration

- **FEC** pour reconstruire des paquets perdus sans retransmission.
- **Duplication de paquets** sur deux tunnels pour les flux critiques.
- **Ajustement du MSS** TCP pour éviter la fragmentation due à l'encapsulation.

### 12.6 Sortie directe Internet (DIA) et NAT

- Un VPN de service peut sortir **localement** sur Internet via un **NAT** sur l'interface de transport.
- Exemple : le VPN 30 (Wi-Fi invités) sort toujours en local ; le VPN 20 sort en local uniquement pour les applications SaaS autorisées, le reste passe par le datacenter.

---

## 13. Services de sécurité et réseau sur le Edge

Selon la plateforme et les licences, un Edge peut embarquer :

| Service | Fonction | Exemple Carrefour |
|---|---|---|
| **Pare-feu à zones** | Filtrage entre zones (VPN, interfaces) | Séparer le VPN 20 d'Internet et du VPN 30 |
| **IPS / IDS** | Détection et blocage d'attaques | Sur les sorties Internet des hypermarchés |
| **Filtrage URL** | Blocage de catégories de sites | Wi-Fi invités et back-office |
| **Anti-malware** | Analyse de fichiers | Postes du back-office |
| **DNS sécurisé** | Blocage de domaines malveillants | Tous segments, via un service cloud |
| **Tunnels vers un service cloud de sécurité (SASE/SSE)** | Inspection déportée du trafic web | Magasins de proximité à faible puissance |
| **Routage dynamique** | OSPF, BGP, EIGRP avec le LAN | Hypermarché avec coeur de réseau LAN |
| **DHCP, NAT, DNS relay** | Services réseau de site | Tous les sites |

> Choix d'architecture : pour les petits Edges, il est fréquent de **déporter** la sécurité vers un service cloud plutôt que de saturer le CPU local avec de l'inspection approfondie.

---

## 14. Haute disponibilité du Edge

### 14.1 Niveaux de redondance

| Niveau | Mécanisme |
|---|---|
| Liens | Plusieurs transports de technologies et d'opérateurs différents |
| Chemins | Plusieurs tunnels par destination, bascule par SLA |
| Équipements | Deux Edges par site critique |
| Alimentation et matériel | Alimentations redondantes, onduleur |
| Contrôleurs | Plusieurs Validators, Controllers, cluster de Managers |

### 14.2 Deux Edges sur un même site

Éléments clés :

- **Même site-id**, **system-IP différents**.
- **VRRP** côté LAN pour offrir une passerelle virtuelle par VPN de service.
- Chaque Edge possède **ses propres transports**, ou les partage grâce à la **TLOC extension** (un Edge utilise le lien WAN de l'autre via une liaison interne).
- Politique de **préférence** : l'Edge primaire est actif, le secondaire est en attente, avec bascule sur détection de panne.

```
            MPLS            Internet            5G
              |                |                 |
         +----+----+      +----+----+            |
         | Edge A  |======| Edge B  |------------+
         +----+----+  TLOC+----+----+
              |     extension   |
              +----- LAN -------+
                (VRRP par VPN)
```

### 14.3 Points d'attention

- Éviter les **chemins asymétriques** entre les deux Edges (ajuster les préférences OMP et VRRP).
- **Tester** la bascule régulièrement, y compris le retour.
- Veiller à la **cohérence des templates** des deux Edges (même politique, adresses différentes).

---

## 15. Edge dans le cloud public

### 15.1 Principe

Pour que les applications hébergées dans un cloud public fassent partie de l'overlay, on déploie des **Edges virtuels** dans les réseaux virtuels du cloud (VPC, VNet).

### 15.2 Séquence d'intégration d'un Edge cloud

1. Déploiement de l'instance depuis la place de marché du cloud.
2. Ajout de l'équipement virtuel dans le Manager, génération du **chassis number et du token OTP**.
3. Activation de l'instance avec ces valeurs.
4. Phases 4 à 8 de la séquence d'intégration standard.
5. Annonce des **préfixes** du réseau virtuel dans l'overlay.

### 15.3 Exemple Carrefour

- Deux régions cloud, deux Edges virtuels par région, hébergeant la connexion vers l'ERP et le site e-commerce.
- Les magasins accèdent aux applications cloud **par l'overlay**, en choisissant le meilleur chemin.
- Les politiques de **segmentation** s'étendent jusque dans le cloud (VPN 10 uniquement vers le service de paiement).

---

## 16. Configuration d'exemple

> Exemple pédagogique simplifié, de style Cisco IOS XE SD-WAN. Noms, adresses et valeurs sont fictifs. En production, ces éléments se pilotent par des templates du Manager.

### 16.1 Configuration d'amorçage (Day-0)

```
system
 host-name              FR-IDF-SUP-0123
 organization-name      "RETAIL-DEMO"
 vbond vbond.retail-demo.example.com
!
interface GigabitEthernet0/0/0
 description WAN_AMORCAGE
 ip address dhcp
 tunnel-interface
  encapsulation ipsec
  color biz-internet
  allow-service dhcp
  allow-service dns
  allow-service icmp
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 dhcp
```

### 16.2 Configuration poussée par le Manager (Day-1)

```
system
 host-name              FR-IDF-SUP-0123
 system-ip              10.255.10.123
 site-id                10123
 organization-name      "RETAIL-DEMO"
 vbond vbond.retail-demo.example.com
!
vrf definition 10
 description CAISSES_PAIEMENT
!
vrf definition 20
 description BACK_OFFICE
!
vrf definition 30
 description WIFI_INVITES
!
vrf definition 40
 description VIDEOSURVEILLANCE
!
vrf definition 50
 description IOT
!
interface GigabitEthernet0/0/0
 description WAN_FIBRE_INTERNET
 ip address dhcp
 tunnel-interface
  encapsulation ipsec
  color biz-internet
  allow-service dhcp
  allow-service dns
  allow-service icmp
 no shutdown
!
interface GigabitEthernet0/0/1
 description WAN_MPLS
 ip address 192.0.2.2 255.255.255.252
 tunnel-interface
  encapsulation ipsec
  color mpls restrict
 no shutdown
!
interface Cellular0/2/0
 description WAN_5G_SECOURS
 tunnel-interface
  encapsulation ipsec
  color lte
 no shutdown
!
interface GigabitEthernet0/0/2.10
 description LAN_CAISSES
 encapsulation dot1Q 10
 vrf forwarding 10
 ip address 10.10.123.1 255.255.255.0
!
interface GigabitEthernet0/0/2.20
 description LAN_BACKOFFICE
 encapsulation dot1Q 20
 vrf forwarding 20
 ip address 10.20.123.1 255.255.255.0
!
interface GigabitEthernet0/0/2.30
 description LAN_WIFI_INVITES
 encapsulation dot1Q 30
 vrf forwarding 30
 ip address 10.30.123.1 255.255.255.0
!
omp
 no shutdown
 graceful-restart
 advertise connected
 advertise static
```

Points à noter :

- `system-ip` et `site-id` apparaissent **à partir du Day-1** : ils font partie du template ;
- le TLOC `mpls` porte `restrict` : il ne forme des tunnels qu'avec d'autres interfaces `mpls` ;
- chaque `vrf` correspond à un **segment** de l'overlay ;
- les sous-interfaces `.10`, `.20`, `.30` appartiennent chacune à un VPN de service différent.

---

## 17. Supervision et dépannage du Edge

### 17.1 Méthode : suivre la séquence d'intégration

Face à un Edge qui ne fonctionne pas, on **remonte la séquence** jusqu'à la première étape qui échoue :

1. L'Edge a-t-il une **IP, un DNS, une heure correcte** ?
2. A-t-il reçu l'adresse du **Validator** ?
3. Est-il en **mode contrôleur** ?
4. S'est-il **authentifié** auprès du Validator ?
5. La session avec le **Manager** est-elle établie ?
6. Le **template** est-il appliqué ?
7. Les sessions avec les **Controllers** et **OMP** sont-elles up ?
8. Les **tunnels BFD** sont-ils up ?

### 17.2 Commandes utiles (IOS XE SD-WAN)

Les commandes exactes varient selon la version.

```
show clock                                  ! heure de l'équipement
show ip interface brief                     ! état et adresses des interfaces
show sdwan control local-properties         ! identité, certificat, couleurs, validator
show sdwan control connections              ! état des sessions vers Validator, Manager, Controllers
show sdwan control connections-history      ! historique et raisons de déconnexion
show sdwan certificate installed            ! certificats présents
show sdwan omp summary                      ! état d'OMP
show sdwan omp routes                       ! routes reçues par OMP
show sdwan omp tlocs                        ! TLOC connus
show sdwan bfd sessions                     ! tunnels et BFD
show sdwan app-route stats                  ! mesures de qualité par tunnel
show sdwan running-config                   ! configuration SD-WAN active
show sdwan system status                    ! résumé de l'état de l'équipement
```

### 17.3 Lecture d'une session de contrôle

Un tableau simplifié des sessions de contrôle montre en général, pour chaque pair :

| Colonne | Exemple | Signification |
|---|---|---|
| Type de pair | vbond, vmanage, vsmart | Composant joint |
| Protocole | dtls, tls | Mode de session |
| Adresse du pair | 203.0.113.10 | IP du contrôleur |
| Interface locale / couleur | Gi0/0/0, biz-internet | Transport utilisé |
| État | up, connect, challenge, tear_down | Étape de la connexion |
| Durée | 2d 04:10:22 | Ancienneté de la session |

Un état **connect** ou **challenge** qui persiste indique un blocage dans l'authentification ou le transport ; un état **tear_down** répété indique une session instable.

### 17.4 Arbre de décision simple

| Constat | Diagnostic probable | Action |
|---|---|---|
| Pas d'IP sur l'interface WAN | Phase 1 | Vérifier DHCP, câble, port, console |
| IP OK, aucune session vers le Validator | Phases 2 à 4 | Vérifier DNS, pare-feu, adresse du Validator, org-name |
| Session Validator OK puis rejet | Phase 4 | Vérifier liste autorisée, certificats, heure |
| Session Manager OK, pas de config | Phase 6 | Rattacher le template, vérifier les variables |
| Config OK, pas de session Controller | Phase 7 | Vérifier system-IP, site-id, tunnel-interface |
| OMP up, pas de tunnel | Phase 8 | Vérifier ports IPsec, NAT, couleurs |
| Tunnels up, application lente | Plan de données | Vérifier MTU/MSS, QoS, SLA, charge |

### 17.5 Indicateurs de santé d'un Edge

- **Disponibilité des sessions de contrôle** (au moins une session vers chaque type de contrôleur).
- **Nombre de tunnels BFD up** par rapport au nombre attendu.
- **Latence, gigue, perte** par transport.
- **Utilisation CPU et mémoire** (surtout sur les petits Edges avec chiffrement et sécurité activés).
- **Validité des certificats** et **version logicielle** conforme à la cible.
- **Temps d'intégration** moyen d'un nouveau site.

---

## 18. Exercices et corrigés

### Exercice 1 : remettre la séquence dans l'ordre

Classez ces événements dans l'ordre chronologique de l'intégration.

a. Établissement des tunnels IPsec et BFD.
b. Obtention d'une IP par DHCP.
c. Authentification auprès du Validator.
d. Session de contrôle avec le Manager.
e. Réception de l'adresse du Validator par le service de découverte.
f. Session OMP avec un Controller.
g. Application du template d'équipement.
h. Bascule en mode contrôleur.

**Corrigé.** b, e, h, c, d, g, f, a.

### Exercice 2 : associer phase et problème

Associez chaque symptôme à la phase probable.

1. L'équipement n'obtient pas d'adresse IP.
2. Le Validator rejette le certificat de l'équipement.
3. Le template ne s'applique pas à cause d'une variable manquante.
4. La session OMP est établie, mais aucun tunnel de données ne monte.

**Corrigé.** 1 : phase 1. 2 : phase 4. 3 : phase 6. 4 : phase 8.

### Exercice 3 : diagnostic

Un magasin livré hier n'apparaît jamais dans le Manager. Le gérant indique que l'équipement est allumé et que la box fibre fonctionne. Quelles sont les 5 vérifications prioritaires ?

**Corrigé (exemple).**
1. Le numéro de série est-il bien rattaché au profil dans le portail de découverte ?
2. L'équipement figure-t-il dans la liste du Manager à l'état Valid ?
3. L'équipement obtient-il une IP, un DNS et une heure correcte (accès console si besoin) ?
4. Le pare-feu ou la box bloque-t-il l'accès HTTPS vers le service de découverte ou les ports de contrôle ?
5. L'organization-name et l'adresse du Validator du profil sont-ils corrects ?

### Exercice 4 : identités

Deux Edges d'un même hypermarché sont en haute disponibilité. Indiquez pour chacun des éléments suivants s'il doit être **identique** ou **différent** entre les deux : site-id, system-IP, chassis number, organization-name, hostname, couleurs des transports.

**Corrigé.**
- site-id : identique ;
- system-IP : différent ;
- chassis number : différent ;
- organization-name : identique ;
- hostname : différent ;
- couleurs : en général identiques si les deux Edges utilisent les mêmes types de transports, ce qui permet les mêmes politiques.

### Exercice 5 : choix de méthode d'intégration

Pour chaque site, choisissez la méthode la plus adaptée : PnP automatique, bootstrap manuel, token OTP, pré-staging en atelier.

a. Un magasin de proximité avec box fibre et DHCP.
b. Un entrepôt dont l'accès Internet est en IP fixe, sans DHCP.
c. Un Edge virtuel dans un cloud public.
d. Un hypermarché critique qui ouvre dans deux semaines et dont le réseau doit être validé en amont.

**Corrigé.** a : PnP automatique. b : bootstrap manuel (fichier d'amorçage avec IP statique). c : token OTP. d : pré-staging en atelier, puis livraison.

### Exercice 6 : planifier un déploiement

Le groupe doit intégrer 150 magasins de proximité en 6 semaines avec 3 personnes. Proposez une organisation.

**Corrigé (pistes).**
- Import en masse des numéros de série dans le portail de découverte et dans le Manager.
- Un seul **template** "Proximité" avec variables importées depuis un fichier ou une API.
- Automatisation du **rattachement du template** après détection de l'Edge en ligne.
- **Pilote** de 5 magasins, puis vagues de 25 à 30 sites.
- **Checklist de validation** identique pour chaque site, exécutée par le NOC.
- **Support de niveau 1** du magasin : une notice simple (brancher, allumer, appeler en cas de voyant rouge).

### Exercice 7 : QCM

1. Quel est le premier composant contacté par un Edge neuf après le service de découverte ?
   a) Manager b) Validator c) Controller d) Un autre Edge
2. Que se passe-t-il si l'organization-name est différent entre l'Edge et le Validator ?
   a) Rien b) L'Edge est rejeté c) L'Edge passe en mode secours d) L'Edge utilise un autre Validator
3. Quel VPN ne doit jamais contenir de trafic utilisateur ?
   a) VPN 10 b) VPN 0 c) VPN 30 d) VPN 100
4. Quel élément identifie de façon unique l'Edge dans l'overlay (identifiant logique) ?
   a) System-IP b) Site-id c) Couleur d) Hostname
5. Quel mécanisme assure la détection de panne et la mesure de qualité dans les tunnels ?
   a) OMP b) BFD c) NETCONF d) VRRP

**Corrigé.** 1b, 2b, 3b, 4a, 5b.

### Exercice 8 : étude de cas

Un Edge d'un supermarché affiche, dans `show sdwan control connections`, une session vers le Validator qui reste en état **connect**, et aucun autre contrôleur. Il a une IP, un DNS fonctionnel et la bonne heure. Quelles hypothèses formuler, et dans quel ordre les tester ?

**Corrigé (exemple).**
1. Un **pare-feu** ou la **box** bloque les ports de contrôle vers le Validator : tester la connectivité et vérifier les journaux.
2. L'adresse du Validator résout vers une **IP incorrecte ou injoignable** : tester la résolution et la route.
3. Un **NAT** ou un **MTU** perturbe la négociation DTLS : tester avec un autre lien (5G) pour comparer.
4. Un problème de **certificat ou d'autorisation** : vérifier l'historique des connexions et les codes de raison.

### Exercice 9 : compter les tunnels

Un hub possède 2 TLOC : `mpls` (avec `restrict`) et `biz-internet`. Un spoke possède 3 TLOC : `mpls` (avec `restrict`), `biz-internet` et `lte`. 

1. Combien de tunnels entre le hub et un spoke ?
2. Combien de tunnels sur le hub pour 2 000 spokes ?
3. Combien de tunnels et combien de débit BFD par sens sur le hub, avec un hello de 1 s et 150 octets par paquet ?

**Corrigé.**
1. Paires compatibles : mpls-mpls, biz-biz, biz-lte, soit **3 tunnels**.
2. 2 000 x 3 = **6 000 tunnels**.
3. 6 000 x 1,2 kbit/s = 7 200 kbit/s, soit **7,2 Mbit/s par sens** (et 6 000 paquets par seconde par sens).

### Exercice 10 : MTU, MSS et voix

Un lien cellulaire a une MTU de 1 400 octets. Le surcoût IPsec retenu est de 65 octets.

1. Quelle est la MTU de tunnel et la MSS TCP ?
2. Quel débit réseau consomme un appel G.711 (20 ms) une fois chiffré ?

**Corrigé.**
1. MTU de tunnel = 1 400 - 65 = **1 335** ; MSS = 1 335 - 40 = **1 295**.
2. Paquet voix = 160 + 12 + 8 + 20 = 200 octets ; chiffré = 265 octets ; 265 x 8 x 50 = **106 kbit/s**.

### Exercice 11 : disponibilité et BFD

Un magasin a une fibre à 99,5 % et une 5G à 99,0 %, supposées indépendantes.

1. Quelle est la disponibilité cumulée et l'indisponibilité annuelle en minutes ?
2. Le magasin subit 20 incidents par an avec une détection BFD de 7 s et 1 s de convergence. Quelle interruption s'ajoute ?

**Corrigé.**
1. 1 - 0,005 x 0,01 = **99,995 %** ; 0,00005 x 525 600 = **26,3 minutes** par an.
2. 20 x (7 + 1) = 160 s, soit **environ 2,7 minutes** par an d'interruptions de bascule, qui s'ajoutent aux 26,3 minutes théoriques (environ 10 % de plus) : la **détection BFD** pèse directement dans la disponibilité réelle des flux critiques.

### Exercice 12 : choisir les couleurs et vérifier les tunnels

Un site X a les TLOC `mpls` (restreint), `biz-internet`, `custom1`. Un site Y a les TLOC `mpls` (restreint), `biz-internet`. Combien de tunnels peut-on établir entre X et Y ? Lesquels ?

**Corrigé.** mpls-mpls (même couleur restreinte), biz-biz, custom1-biz (deux couleurs publiques). Soit **3 tunnels**. Les paires impliquant `mpls` avec une autre couleur sont exclues par `restrict`.

---

## 19. Bonnes pratiques et erreurs fréquentes

### Bonnes pratiques

- **Préparer en amont** : la grande majorité des échecs d'intégration viennent d'une préparation incomplète (portail, liste autorisée, variables).
- **Standardiser** : peu de profils de site, templates réutilisables, convention de nommage stricte.
- **Valider un pilote** avant toute industrialisation, avec un site de chaque profil.
- **Automatiser** le rattachement des templates et la vérification post-intégration via l'API.
- **Surveiller les certificats** et l'heure (NTP) de tous les composants.
- **Documenter** une procédure de dépannage par phase, utilisable par le support de niveau 1.
- **Prévoir un accès console** (ou un moyen de dépannage hors bande) pour les sites critiques.
- **Tester les pannes** : perte d'un transport, perte d'un Edge, perte des Controllers.
- **Prévoir la fin de vie** : invalidation et réinitialisation propre des équipements retirés.

### Erreurs fréquentes

- Oublier de **rattacher le numéro de série** au bon profil dans le portail de découverte.
- Laisser un **system-IP en doublon** entre deux sites.
- Utiliser des **organization-name** légèrement différents (casse, espaces).
- Négliger l'**heure** de l'équipement, ce qui invalide les certificats.
- Bloquer les **ports de contrôle** sur un pare-feu de site ou d'opérateur.
- Déployer un **template** sans vérifier toutes les variables obligatoires.
- Appliquer une configuration qui **coupe l'accès** du Edge au Manager, sans mécanisme de retour arrière.
- Ignorer la **MTU** sur les liens cellulaires.
- Sous-dimensionner un petit Edge en activant toute la sécurité locale.

---

## 20. Glossaire

| Terme | Définition |
|---|---|
| **Edge** | Équipement SD-WAN installé sur un site, ou dans le cloud, qui exécute le plan de données. |
| **WAN Edge** | Autre nom courant du Edge dans l'écosystème Cisco. |
| **cEdge** | Edge fonctionnant sur une plateforme IOS XE. |
| **vEdge** | Edge de la génération historique Viptela. |
| **Onboarding / intégration** | Ensemble des étapes qui amènent un Edge neuf à un état opérationnel. |
| **PnP** | Plug and Play : mécanisme de découverte automatique des équipements IOS XE. |
| **ZTP** | Zero-Touch Provisioning : mécanisme de découverte automatique équivalent, historique pour les vEdge. |
| **Bootstrap** | Configuration minimale d'amorçage qui permet de joindre les contrôleurs. |
| **Controller mode** | Mode où l'équipement est piloté par le Manager. |
| **Autonomous mode** | Mode où l'équipement fonctionne comme un routeur classique, configuré en CLI. |
| **Chassis number** | Identifiant unique de l'équipement utilisé pour l'autoriser. |
| **System-IP** | Identifiant logique unique de l'Edge dans l'overlay. |
| **Site-ID** | Identifiant du site ; partagé par les Edges d'un même site. |
| **Organization-name** | Nom d'organisation commun à tous les composants d'un même overlay. |
| **Token OTP** | Mot de passe à usage unique pour l'authentification d'un Edge virtuel. |
| **SUDI** | Identité unique du matériel, certificat installé en usine. |
| **Valid / Staging / Invalid** | États d'autorisation d'un Edge dans le Manager. |
| **VPN 0 / VPN 512** | VPN de transport / VPN de gestion. |
| **TLOC** | Transport Locator : system-IP, couleur et encapsulation d'un point d'accès au transport. |
| **TLOC extension** | Partage d'un transport entre deux Edges d'un même site. |
| **Couleur** | Étiquette qualifiant un type de transport. |
| **Restrict** | Option limitant les tunnels aux interfaces de même couleur. |
| **BFD** | Bidirectional Forwarding Detection : détection de panne et mesure de qualité. |
| **OMP** | Overlay Management Protocol : protocole de routage de l'overlay. |
| **DIA** | Direct Internet Access : sortie directe vers Internet. |
| **RMA** | Remplacement d'un équipement défectueux. |
| **Day-0 / Day-1 / Day-2** | Intégration initiale / mise en service / exploitation. |
| **Transport** | Lien réel fourni par un opérateur (MPLS, Internet, cellulaire), à la base de l'underlay. |
| **Preference** | Priorité d'un TLOC ou d'une route : la valeur la plus élevée est préférée. |
| **Weight** | Poids servant à répartir le trafic entre chemins de même préférence (ECMP). |
| **ECMP** | Equal-Cost Multi-Path : répartition du trafic sur plusieurs chemins équivalents. |
| **Hello interval / multiplicateur BFD** | Intervalle d'envoi des sondes et nombre de sondes perdues avant déclaration de panne. |
| **Poll-interval** | Durée de collecte des mesures BFD pour le routage applicatif. |
| **MSS** | Taille maximale de segment TCP, à ajuster pour tenir compte de l'encapsulation. |
| **Hub / Spoke** | Edge central qui agrège les tunnels / Edge de site relié aux hubs. |

---

## 21. Fiche de révision

### Les 9 phases de l'intégration en une ligne chacune

| Phase | Ce qui se passe |
|---|---|
| 0 | L'entreprise déclare l'équipement, l'autorise et prépare son template |
| 1 | L'Edge démarre, obtient IP, DNS et heure |
| 2 | Il interroge le service de découverte et apprend l'adresse du Validator |
| 3 | Il bascule en mode contrôleur avec une configuration d'amorçage |
| 4 | Il s'authentifie auprès du Validator et reçoit la liste des contrôleurs |
| 5 | Il se connecte au Manager |
| 6 | Il reçoit et applique sa configuration complète (template) |
| 7 | Il se connecte aux Controllers, OMP échange routes, TLOC et clés |
| 8 | Les tunnels IPsec et BFD montent, le trafic circule |

### Les trois mots à retenir pour réussir une intégration

1. **Identité** : numéro de série, chassis number, organization-name, certificats.
2. **Connectivité** : DHCP, DNS, heure, ports de contrôle ouverts.
3. **Configuration** : template complet avec system-IP, site-id et variables.

### Transport, couleur, TLOC en trois lignes

1. Le **transport** est le lien opérateur, la **couleur** son étiquette, le **TLOC** son adresse complète (system-IP, couleur, encapsulation).
2. Une **route OMP** pointe vers un TLOC comme un préfixe BGP pointe vers un next hop ; sans tunnel BFD vers ce TLOC, la route n'est pas utilisée.
3. Les tunnels se calculent par **paires de TLOC compatibles** (couleurs et `restrict`), et le temps de détection vaut **hello x multiplicateur**.

### Fil rouge à retenir

Un supermarché du groupe est livré avec un Edge déjà déclaré et autorisé. Le gérant le branche, l'Edge **trouve** son Validator, **prouve** son identité, **reçoit** sa configuration, **rejoint** le plan de contrôle, **monte** ses tunnels, et en moins d'une demi-heure les caisses sont opérationnelles.
