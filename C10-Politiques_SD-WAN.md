# Cours Politiques SD-WAN : types, composants et cas pratiques

Ce cours est entièrement consacré aux **politiques** (policies) d'un réseau SD-WAN : leurs types, leur anatomie, leur mode d'application, leurs exemples de configuration et leur gouvernance. Il suppose acquises les bases du SD-WAN, du ZTP et du MTU. Le fil rouge est le réseau de magasins d'un distributeur de type Carrefour.

> Note : le scénario Carrefour est un cas d'école. Chiffres, noms de sites, adresses IP et règles sont fictifs et ne décrivent pas l'organisation réelle de l'enseigne. Les exemples de configuration suivent la logique de Cisco Catalyst SD-WAN, mais sont **simplifiés et pédagogiques** : la syntaxe exacte, les noms de paramètres et les valeurs par défaut varient selon la version. Vérifiez toujours dans la documentation de votre plateforme.

---

## Table des matières

1. Qu'est-ce qu'une politique SD-WAN
2. Taxonomie générale des politiques
3. Anatomie d'une politique
4. Les politiques dans Cisco Catalyst SD-WAN
5. Politiques de contrôle : topologie, VPN membership, service chaining
6. Politiques de données (data policies)
7. Politique de routage applicatif (AAR) et classes SLA
8. Politiques de QoS (localisées)
9. Politiques de sécurité
10. Politiques de NAT, DIA et accès cloud
11. Politiques d'optimisation (FEC, duplication, TCP)
12. Politiques de visibilité et de télémétrie
13. Politiques de gestion et de cycle de vie
14. Les politiques chez les autres éditeurs
15. Ordre d'évaluation, conflits et priorités
16. Cas pratique : le catalogue de politiques d'un distributeur
17. Scénarios dynamiques : Black Friday, panne, mode dégradé
18. Gouvernance : conception, versionnement, tests, retour arrière
19. Dépannage des politiques
20. Bonnes pratiques et pièges
21. Exercices et corrigés
22. Quiz de révision
23. Glossaire

---

## 1. Qu'est-ce qu'une politique SD-WAN

### 1.1 Définition

Une **politique** est un ensemble de règles qui exprime l'**intention** de l'entreprise sur la façon dont le réseau doit se comporter : quel trafic emprunte quel chemin, avec quelle qualité de service, avec quelle sécurité, vers quelle destination. Elle est définie **de façon centralisée** puis distribuée aux équipements d'extrémité (edges) qui l'appliquent localement.

### 1.2 Ce que les politiques remplacent

Dans un WAN traditionnel, ces comportements sont codés équipement par équipement : listes de contrôle d'accès, route-maps, class-maps, policy-based routing. Chaque modification exige des interventions multiples et expose à des écarts entre sites.

Avec le SD-WAN :

| WAN traditionnel | SD-WAN |
|---|---|
| Configuration CLI par équipement | Politique centralisée, déployée par liste de sites |
| Routage statique ou PBR manuel | Routage applicatif basé sur la qualité mesurée |
| QoS statique sur chaque interface | Classes de service définies une fois, appliquées par profil |
| Règles de pare-feu locales | Politique de sécurité commune et variables par site |
| Changement lent et risqué | Changement central, par vagues, avec retour arrière |

### 1.3 Deux logiques de distribution

1. **Politiques centralisées** : définies sur le manager, évaluées ou distribuées par les contrôleurs. Elles s'appliquent à un ensemble de sites.
2. **Politiques localisées** : définies par template ou profil, appliquées sur un équipement ou une interface précise (QoS, ACL, politique de routage local).

Cette distinction structure la plupart des plateformes, même si les noms diffèrent.

---

## 2. Taxonomie générale des politiques

### 2.1 Vue d'ensemble (indépendante de l'éditeur)

| Famille | Question à laquelle elle répond | Exemple |
|---|---|---|
| Topologie et contrôle | Qui peut parler à qui, par quels points d'accès ? | Hub and spoke, mesh partiel, appartenance à un VRF |
| Pilotage du trafic (data) | Où envoyer ce flux ? | Sortie Internet locale pour Microsoft 365 |
| Choix de chemin selon la qualité (AAR) | Quel lien privilégier selon la perte, la latence et la gigue ? | Paiement sur MPLS, bascule sur Internet puis 5G |
| QoS | Quelle priorité et quelle bande passante par classe ? | Voix en priorité stricte |
| Sécurité | Qu'est-ce qui est autorisé, inspecté, bloqué ? | Pare-feu, IPS, filtrage URL, DNS |
| NAT et accès cloud | Comment sortir vers Internet ou le cloud ? | DIA avec NAT, passerelle SASE |
| Optimisation | Comment améliorer la performance d'un lien dégradé ? | FEC, duplication de paquets, optimisation TCP |
| Visibilité | Que faut-il mesurer et exporter ? | Export de flux, visibilité applicative |
| Gestion et cycle de vie | Comment maintenir les équipements ? | Fenêtres de mise à jour, conformité de version |

### 2.2 Niveau d'application

| Niveau | Exemples | Portée |
|---|---|---|
| Global (entreprise) | Politique de topologie, catalogue d'applications | Tous les sites |
| Par région ou par format | Variante de politique pour proximités vs hypermarchés | Groupe de sites |
| Par site | Exceptions (site pilote, site sans fibre) | Un site |
| Par interface | QoS sur lien WAN, ACL sur LAN | Une interface |
| Par flux | Règle qui concerne une application | Séquence de politique |

### 2.3 Principe d'économie

Plus il y a de politiques distinctes, plus la maintenance et le dépannage deviennent difficiles. On cherche le **plus petit nombre de politiques** couvrant les besoins, avec des **listes** (de sites, d'applications, de couleurs) qui absorbent la variation.

---

## 3. Anatomie d'une politique

### 3.1 Les éléments communs

| Élément | Rôle |
|---|---|
| Listes (objets) | Regroupements réutilisables : sites, VPN, applications, préfixes, couleurs, classes SLA |
| Séquences | Règles numérotées, évaluées dans l'ordre |
| Critères de correspondance (match) | Conditions : application, source, destination, port, DSCP, protocole |
| Actions | Ce qui est fait : accepter, rejeter, choisir un chemin, marquer, copier, optimiser |
| Action par défaut | Ce qui se passe quand aucune séquence ne correspond |
| Point d'application | Où la politique s'applique : site, VPN, direction (entrée, sortie), interface |

### 3.2 Les listes

| Type de liste | Exemple de contenu |
|---|---|
| Liste de sites | Tous les sites de proximité (identifiants 3000 à 3999) |
| Liste de VPN (VRF) | VPN 10 monétique, 20 caisses, 30 voix |
| Liste d'applications | Microsoft 365, SAP, voix, vidéo, paiement |
| Liste de préfixes de données | Serveurs de paiement du data center |
| Liste de couleurs | Primaires (mpls, biz-internet), secours (lte) |
| Liste de TLOC | Points d'accès des hubs |
| Classes SLA | Seuils de perte, latence et gigue |

Bonne pratique : **ne jamais écrire de valeur en dur dans une séquence** si elle peut être placée dans une liste. Une liste se corrige à un seul endroit.

### 3.3 Les séquences

- Numérotées avec des intervalles (10, 20, 30...) pour pouvoir insérer des règles.
- Évaluées **dans l'ordre** : la première qui correspond s'applique.
- Les règles les plus spécifiques sont placées avant les plus générales.

### 3.4 L'action par défaut

Chaque type de politique a un comportement par défaut lorsque rien ne correspond. Selon le type et la plateforme, il peut être "accepter" ou "rejeter". Cela a un impact énorme : une politique de contrôle dont l'action par défaut est "rejeter" peut isoler des sites par omission. **Vérifiez systématiquement la valeur par défaut** de chaque type de politique de votre plateforme.

### 3.5 Les points d'application

- **Par liste de sites** : "cette politique s'applique aux sites de proximité".
- **Par direction** : trafic venant du LAN (from-service), trafic venant du tunnel (from-tunnel), entrée ou sortie.
- **Par VPN** : politique valable pour une liste de VRF.

---

## 4. Les politiques dans Cisco Catalyst SD-WAN

### 4.1 Classification officielle

Cisco distingue trois grandes catégories :

```
Politiques
|
+-- Centralisées (définies sur le manager, activées sur le contrôleur)
|     |
|     +-- Politiques de contrôle (control policy)
|     |     +-- Topologie : hub and spoke, mesh, personnalisée
|     |     +-- Appartenance aux VPN (VPN membership)
|     |     +-- Politiques de route et de TLOC personnalisées
|     |
|     +-- Politiques de trafic (data / traffic policy)
|           +-- Application-Aware Routing (AAR)
|           +-- Traffic data policy
|           +-- Cflowd (export de flux)
|
+-- Localisées (appliquées à l'edge via template ou groupe de configuration)
|     +-- ACL (IPv4, IPv6)
|     +-- QoS (classes, ordonnanceurs, mise en forme, rewrite)
|     +-- Politiques de routage (route policy) pour BGP, OSPF, etc.
|     +-- Policer, mirror, visibilité applicative et de flux
|
+-- Sécurité
      +-- Pare-feu à zones
      +-- IPS / IDS
      +-- Filtrage URL
      +-- Protection contre les logiciels malveillants
      +-- Sécurité DNS
      +-- Proxy SSL/TLS
```

### 4.2 Où s'exécute chaque politique

| Politique | Évaluée par | Effet |
|---|---|---|
| Contrôle | Contrôleur | Modifie les routes et TLOC annoncés aux edges |
| Données centralisée | Edge (reçue du contrôleur) | Oriente et modifie les paquets |
| AAR | Edge | Sélectionne le tunnel selon les mesures de qualité |
| Localisée (QoS, ACL) | Edge | Priorise, filtre, façonne |
| Sécurité | Edge ou service cloud | Inspecte et autorise ou bloque |

### 4.3 Cycle de vie d'une politique centralisée

1. Création des **listes**.
2. Création des **politiques** (topologie, trafic).
3. Regroupement dans une **politique centralisée** et rattachement aux **listes de sites**.
4. **Activation** : le contrôleur reçoit la politique et la distribue (ou l'applique).
5. **Vérification** sur les edges.
6. **Désactivation** ou modification, selon le besoin.

### 4.4 Remarque sur les évolutions

Les interfaces de gestion évoluent (groupes de politiques, assistants basés sur l'intention). Les concepts ci-dessus restent valables, mais les écrans, les noms et certaines fonctions changent selon les versions.

---

## 5. Politiques de contrôle : topologie, VPN membership, service chaining

### 5.1 Rôle

Elles agissent sur le **plan de contrôle** : quelles routes et quels TLOC sont annoncés à quels sites. Elles façonnent la topologie logique de l'overlay.

### 5.2 Topologie hub and spoke

Objectif : les magasins communiquent uniquement avec les hubs (data centers), pas directement entre eux.

```
Magasin A  ----+                 +---- Magasin B
               |                 |
               +--->  Hub Paris  <---+
               |                 |
Magasin C  ----+                 +---- Magasin D

Pas de tunnel direct A-B, A-C, B-D : tout passe par le hub.
```

Logique de la politique :

- Les spokes ne reçoivent que les TLOC et les routes des hubs.
- Les routes d'un spoke sont réannoncées via les hubs.
- Action par défaut : rejeter ce qui n'est pas explicitement autorisé.

### 5.3 Topologie partielle : mesh pour certains sites

Pour la voix entre magasins d'une même région, on peut autoriser des tunnels directs dans un **groupe** précis de sites, par exemple un maillage régional entre les supermarchés d'une ville.

### 5.4 Pseudo-syntaxe de politique de topologie (simplifiée)

```
policy
 lists
  site-list HUBS
   site-id 100
   site-id 101
  !
  site-list MAGASINS
   site-id 1000-3999
  !
 !
 control-policy POL_TOPO_HUB_SPOKE
  sequence 10
   match route
    site-list HUBS
   !
   action accept
  !
  sequence 20
   match tloc
    site-list HUBS
   !
   action accept
  !
  default-action reject
 !
!
apply-policy
 site-list MAGASINS
  control-policy POL_TOPO_HUB_SPOKE out
 !
!
```

Ce schéma est **conceptuel** : il illustre la logique "accepter les hubs, refuser le reste". La syntaxe réelle dépend de la version.

### 5.5 VPN membership

Une politique d'appartenance aux VPN limite les VPN (VRF) visibles par un site. Exemple : seuls les sites équipés en caméras reçoivent le VPN 40 (vidéosurveillance).

| Liste de sites | VPN autorisés |
|---|---|
| Hypermarchés | 10, 20, 30, 40, 50, 60 |
| Supermarchés | 10, 20, 30, 40, 60 |
| Proximité | 10, 20, 30, 60 |
| Hubs | Tous |

### 5.6 Politiques de route et de TLOC

On peut :

- modifier la **préférence** d'un TLOC (préférer un hub plutôt qu'un autre) ;
- filtrer des préfixes ;
- modifier des attributs de route (par exemple pour de l'ingénierie de trafic) ;
- imposer un **service chaining** : faire passer certains flux par un service intermédiaire (pare-feu centralisé, inspection) avant leur destination.

### 5.7 Service chaining

```
Magasin --> overlay --> Hub (pare-feu de service) --> Serveur central
```

Une politique de contrôle annonce une route avec une **étape de service**. Utile pour inspecter certains flux au hub sans déployer un pare-feu sur chaque magasin.

### 5.8 Exemple retail

| Besoin | Politique |
|---|---|
| Magasins isolés entre eux, sauf voix régionale | Hub and spoke avec exception de maillage voix pour un groupe de sites |
| Vidéosurveillance non visible des proximités | VPN membership par format |
| Flux monétique inspecté au hub | Service chaining vers le pare-feu du data center |
| Deux hubs actifs | Préférence de TLOC régionale (hub Paris pour le nord, hub Lyon pour le sud) |

---

## 6. Politiques de données (data policies)

### 6.1 Rôle

Elles agissent sur le **plan de données** : elles examinent des paquets réels et décident de leur traitement.

### 6.2 Critères de correspondance usuels

- adresses et préfixes source et destination ;
- ports et protocole ;
- DSCP ;
- application ou famille d'applications ;
- nom de domaine (selon fonctionnalités) ;
- VPN.

### 6.3 Actions usuelles

| Action | Usage |
|---|---|
| Accepter ou rejeter | Filtrage simple |
| Définir un DSCP | Marquage pour la QoS |
| Choisir un TLOC ou une couleur locale | Ingénierie de trafic |
| NAT vers le VPN de transport | Sortie Internet locale (DIA) |
| Choisir un service | Service chaining |
| FEC, duplication de paquets | Résilience sur lien dégradé |
| Optimisation TCP | Amélioration de débit |
| Journalisation et export de flux | Visibilité |

### 6.4 Sens d'application

| Direction | Signification |
|---|---|
| Depuis le LAN (from-service) | Trafic qui entre sur l'edge depuis le réseau du site |
| Depuis le tunnel (from-tunnel) | Trafic qui arrive de l'overlay |
| Tous (all) | Les deux sens |

Choisir la bonne direction est essentiel : une règle placée du mauvais côté ne correspond jamais.

### 6.5 Exemple : sortie Internet locale pour Microsoft 365

Pseudo-syntaxe simplifiée :

```
policy
 lists
  vpn-list VPN_BUREAUTIQUE
   vpn 20
  !
  site-list SITES_PROXIMITE
   site-id 3000-3999
  !
  app-list APPS_M365
   app office365
  !
 !
 data-policy POL_DATA_DIA_M365
  vpn-list VPN_BUREAUTIQUE
   sequence 10
    match
     app-list APPS_M365
    !
    action accept
     nat use-vpn 0
    !
   !
   default-action accept
  !
 !
!
apply-policy
 site-list SITES_PROXIMITE
  data-policy POL_DATA_DIA_M365 from-service
 !
!
```

Les noms d'applications dépendent du catalogue de votre plateforme.

### 6.6 Exemple : marquage DSCP des flux monétiques

```
 data-policy POL_DATA_MARQUAGE
  vpn-list VPN_METIER
   sequence 10
    match
     destination-data-prefix-list DC_MONETIQUE
     destination-port 443
    !
    action accept
     set
      dscp 34
     !
    !
   !
   default-action accept
  !
 !
```

Ce marquage est ensuite exploité par la politique de QoS locale.

---

## 7. Politique de routage applicatif (AAR) et classes SLA

### 7.1 Principe

Les edges **mesurent en continu** la perte, la latence et la gigue de chaque tunnel. La politique AAR :

1. définit des **classes SLA** (seuils acceptables) ;
2. associe des flux à une classe SLA ;
3. indique des couleurs préférées et des secours.

Le flux est dirigé sur un tunnel qui **respecte la classe SLA**. Si aucun tunnel ne la respecte, la politique applique son comportement de repli.

### 7.2 Classes SLA

| Classe | Perte | Latence | Gigue | Usage |
|---|---|---|---|---|
| SLA_VOIX | 1 % ou moins | 150 ms ou moins | 30 ms ou moins | Téléphonie IP |
| SLA_PAIEMENT | 1 % ou moins | 150 ms ou moins | 50 ms ou moins | Monétique |
| SLA_METIER | 2 % ou moins | 200 ms ou moins | 100 ms ou moins | ERP, caisses |
| SLA_TOLERANT | 5 % ou moins | 400 ms ou moins | Non applicable | Applications peu sensibles |

Les seuils sont indicatifs et se calibrent d'après les mesures réelles du parc.

### 7.3 Options de repli

| Option | Comportement quand aucun tunnel ne respecte la SLA |
|---|---|
| Comportement par défaut | Le flux utilise le meilleur tunnel disponible (ECMP ou le moins mauvais) |
| Couleurs de secours préférées | Le flux bascule vers une couleur de secours (par exemple 5G) |
| Mode strict | Le flux est abandonné plutôt que d'emprunter un chemin non conforme |

Le mode strict est réservé aux flux pour lesquels un chemin dégradé est pire qu'aucun chemin (par exemple certaines sauvegardes sensibles). Pour le paiement, on préfère généralement un repli sur un chemin dégradé plutôt qu'une coupure.

### 7.4 Pseudo-syntaxe d'AAR

```
policy
 lists
  data-prefix-list DC_MONETIQUE
   ip-prefix 10.100.10.0/24
  !
  data-prefix-list DC_VOIX
   ip-prefix 10.100.30.0/24
  !
  vpn-list VPN_METIER
   vpn 10
   vpn 20
   vpn 30
  !
 !
 sla-class SLA_PAIEMENT
  loss 1
  latency 150
  jitter 50
 !
 sla-class SLA_VOIX
  loss 1
  latency 150
  jitter 30
 !
 app-route-policy POL_AAR_RETAIL
  vpn-list VPN_METIER
   sequence 10
    match
     destination-data-prefix-list DC_MONETIQUE
     destination-port 443
    !
    action
     sla-class SLA_PAIEMENT preferred-color mpls biz-internet
     backup-sla-preferred-color lte
    !
   !
   sequence 20
    match
     destination-data-prefix-list DC_VOIX
    !
    action
     sla-class SLA_VOIX preferred-color mpls biz-internet
    !
   !
  !
 !
!
apply-policy
 site-list SITES_MAGASINS
  app-route-policy POL_AAR_RETAIL
 !
!
```

### 7.5 Paramètres de mesure et réactivité

L'AAR s'appuie sur des sondes (BFD) et sur une **fenêtre de moyenne** de mesure. Deux paramètres structurent la réactivité :

| Paramètre | Rôle | Conséquence |
|---|---|---|
| Intervalle de sondage (poll interval) | Durée d'un échantillon de mesure | Plus court : réaction plus rapide, plus de bruit |
| Multiplicateur (poll multiplier) | Nombre d'intervalles utilisés pour la moyenne | Plus grand : moyenne plus lissée, réaction plus lente |

Sur de nombreuses plateformes les valeurs par défaut sont **conservatrices** (de l'ordre de plusieurs minutes d'intervalle). Pour des flux très sensibles, on ajuste ces valeurs avec prudence, en testant l'impact sur la stabilité (risque de flapping). Consultez la documentation de votre version pour les valeurs par défaut et les limites.

### 7.6 Pièges de l'AAR

- Des **seuils trop serrés** provoquent des bascules fréquentes.
- Un flux asymétrique (aller et retour par des chemins différents) peut compliquer l'analyse.
- Les politiques AAR s'appliquent aux flux **qui traversent l'overlay**, pas à ceux qui sortent en DIA.
- Les mesures reflètent le **tunnel**, pas forcément la qualité de l'application elle-même.

---

## 8. Politiques de QoS (localisées)

### 8.1 Pourquoi une QoS locale

L'AAR choisit le **chemin**. La QoS gère la **contention** sur un lien : quand plusieurs flux se disputent la bande passante, qui passe en premier ? Elle s'applique sur l'interface de sortie WAN de l'edge.

### 8.2 Les étapes

1. **Classification** : identifier le trafic (DSCP, application, ACL).
2. **Marquage** : positionner ou normaliser le DSCP.
3. **Files d'attente** (queues) : répartir les classes dans des files.
4. **Ordonnancement** : priorité stricte (LLQ) ou bande passante garantie.
5. **Mise en forme (shaping)** : limiter le débit sortant au débit réel du lien.
6. **Policing** : plafonner un flux.

### 8.3 Modèle de classes

| Classe | DSCP typique | File | Traitement |
|---|---|---|---|
| Contrôle et gestion | CS6 | 0 ou prioritaire | Protégé |
| Voix | EF (46) | LLQ | Priorité stricte, bande plafonnée |
| Monétique et caisses | AF41 (34) | 1 | Bande garantie |
| Applications métier interactives | AF31 (26) | 2 | Bande garantie |
| Défaut | 0 | 3 | Reste de la bande |
| Bulk (sauvegardes, mises à jour) | CS1 (8) | 4 | Minimum garanti, premier à être limité |

### 8.4 Shaping

Le **shaping** limite le débit sortant d'un lien au débit contractuel réel : sans cela, la file d'attente se forme chez l'opérateur, hors de contrôle de l'edge, et la QoS locale devient inefficace.

Exemple : lien fibre de 200 Mbit/s, shaping à 190 Mbit/s pour garder la file d'attente côté edge.

### 8.5 Pseudo-syntaxe de QoS

```
policy
 class-map
  class VOIX queue 0
  class MONETIQUE queue 1
  class METIER queue 2
  class DEFAUT queue 3
  class BULK queue 4
 !
 qos-scheduler SCHED_VOIX
  class VOIX
  bandwidth-percent 10
  buffer-percent 10
  scheduling llq
  drops tail-drop
 !
 qos-scheduler SCHED_MONETIQUE
  class MONETIQUE
  bandwidth-percent 30
  buffer-percent 30
  scheduling wrr
  drops red-drop
 !
 qos-scheduler SCHED_METIER
  class METIER
  bandwidth-percent 25
  buffer-percent 25
  scheduling wrr
  drops red-drop
 !
 qos-scheduler SCHED_DEFAUT
  class DEFAUT
  bandwidth-percent 25
  buffer-percent 25
  scheduling wrr
  drops red-drop
 !
 qos-scheduler SCHED_BULK
  class BULK
  bandwidth-percent 10
  buffer-percent 10
  scheduling wrr
  drops red-drop
 !
 qos-map QOS_WAN
  qos-scheduler SCHED_VOIX
  qos-scheduler SCHED_MONETIQUE
  qos-scheduler SCHED_METIER
  qos-scheduler SCHED_DEFAUT
  qos-scheduler SCHED_BULK
 !
!
```

Ici, la somme des pourcentages doit rester cohérente avec les règles de la plateforme (souvent 100 %).

### 8.6 Association à l'interface

```
interface GigabitEthernet0/0/0
 shaping-rate 190000
 qos-map QOS_WAN
!
```

Sur certaines plateformes, la QoS peut aussi s'appliquer **par tunnel** ou par interface logique.

### 8.7 Correspondance entre DSCP de l'entreprise et DSCP de l'opérateur

Un opérateur MPLS impose souvent son propre modèle de classes de service. Il faut **mapper** les DSCP de l'entreprise vers les classes de l'opérateur (rewrite). Sur un transport Internet, le marquage DSCP n'est généralement pas honoré hors du réseau local : la QoS ne protège que **le départ du lien** de l'edge.

---

## 9. Politiques de sécurité

### 9.1 Familles

| Politique | Objectif |
|---|---|
| ACL | Filtrage de base sans état |
| Pare-feu à zones | Filtrage avec état, entre zones logiques |
| IPS et IDS | Détection et blocage de menaces connues |
| Filtrage URL | Catégories de sites autorisées ou bloquées |
| Sécurité DNS | Blocage de domaines malveillants |
| Protection antimalware | Analyse de fichiers téléchargés |
| Proxy SSL/TLS | Déchiffrement et inspection (selon la politique de confidentialité) |
| Politique d'accès aux applications (ZTNA) | Accès basé sur l'identité et le contexte |

### 9.2 Zones et règles inter-zones

Zones typiques d'un magasin :

| Zone | VPN | Exemple |
|---|---|---|
| ZONE_MONETIQUE | 10 | Terminaux de paiement |
| ZONE_CAISSES | 20 | Caisses, balances |
| ZONE_VOIX | 30 | Téléphones IP |
| ZONE_VIDEO | 40 | Caméras |
| ZONE_INVITES | 60 | Wi-Fi clients |
| ZONE_INTERNET | 0 | Sortie vers Internet |

Matrice de règles (extrait) :

| De vers | Monétique | Caisses | Voix | Vidéo | Invités | Internet |
|---|---|---|---|---|---|---|
| Monétique | Interne | Refusé | Refusé | Refusé | Refusé | Refusé (sauf vers prestataire via overlay) |
| Caisses | Refusé | Interne | Refusé | Refusé | Refusé | Filtré |
| Voix | Refusé | Refusé | Interne | Refusé | Refusé | Refusé (sauf cloud voix) |
| Vidéo | Refusé | Refusé | Refusé | Interne | Refusé | Autorisé vers cloud vidéo |
| Invités | Refusé | Refusé | Refusé | Refusé | Interne | Filtré |

Principe : **deny by default** entre zones, exceptions explicites.

### 9.3 Conformité PCI DSS

- Zone monétique isolée, règles minimales documentées.
- Journalisation des accès et des refus.
- Aucune sortie directe vers Internet depuis la zone monétique, sauf flux strictement nécessaires.
- Revue périodique des règles.

### 9.4 Sécurité intégrée ou déportée

| Modèle | Avantage | Limite |
|---|---|---|
| Sécurité sur l'edge | Inspection locale, faible latence | Charge sur l'équipement, licences |
| Sécurité cloud (SSE, SASE) | Inspection homogène, mise à l'échelle | Dépendance au service, latence vers le point de présence |
| Mixte | Pare-feu local plus inspection cloud pour l'Internet | Complexité de gestion |

---

## 10. Politiques de NAT, DIA et accès cloud

### 10.1 DIA (Direct Internet Access)

Le trafic Internet sort **localement** au magasin plutôt que de remonter au data center. Deux approches :

1. **Route par défaut** dans les VPN de service vers le VPN de transport, avec NAT sur l'interface WAN.
2. **Politique de données** avec action NAT, applicable à des flux sélectionnés (Microsoft 365, SaaS).

### 10.2 Quels flux sortent en local

| Flux | Sortie |
|---|---|
| Microsoft 365, SaaS d'entreprise | Locale (DIA) |
| Wi-Fi invités | Locale (DIA), inspection cloud |
| Mises à jour de systèmes | Locale en horaires creux ou via hub |
| Paiement | Via overlay vers data center |
| ERP central | Via overlay |
| Vidéosurveillance | Via overlay vers stockage central, ou vers cloud vidéo |

### 10.3 Choix de la sortie (local, hub, cloud de sécurité)

La politique peut définir un ordre : sortie locale si le lien est sain, sinon repli vers le hub. Elle peut aussi rediriger vers un service de sécurité cloud par un tunnel dédié.

### 10.4 Optimisation de l'accès aux SaaS

Certaines plateformes mesurent la **qualité vers l'application cloud** (via plusieurs sorties) et choisissent la meilleure, de façon dynamique. C'est une extension de l'AAR vers le cloud (fonction nommée différemment selon l'éditeur).

### 10.5 Dimensionnement NAT

- Capacité de la table de traduction de l'edge.
- Nombre de sessions simultanées (Wi-Fi invités notamment).
- Journalisation des traductions (exigences de conservation de logs en France pour l'accès invités, à valider avec le juridique).

---

## 11. Politiques d'optimisation (FEC, duplication, TCP)

### 11.1 FEC (Forward Error Correction)

Ajout de paquets de redondance pour reconstruire les paquets perdus sans retransmission. Adapté aux flux sensibles à la perte et peu tolérants à la latence (voix, vidéo).

| Mode | Principe |
|---|---|
| Adaptatif | Activé seulement si la perte dépasse un seuil |
| Toujours actif | Redondance constante (coûteux en bande passante) |

### 11.2 Duplication de paquets

Chaque paquet est envoyé simultanément sur deux tunnels. Le récepteur garde le premier arrivé. Très efficace pour les flux critiques, au prix de **doubler la consommation** de bande passante. À réserver à des flux légers et essentiels (paiement, voix).

### 11.3 Optimisation TCP

Techniques : ajustement de fenêtre, accélération du handshake, gestion de la perte. Efficace sur des liens à forte latence. Peut interagir avec le chiffrement applicatif : à tester.

### 11.4 Compression et déduplication

Gain sur certains flux peu compressibles à l'origine. Bénéfice limité aujourd'hui, car une grande partie du trafic est déjà chiffrée ou compressée.

### 11.5 Où les activer

| Besoin | Mécanisme | Cible |
|---|---|---|
| Voix sur lien dégradé | FEC adaptatif | Voix |
| Paiement sur 5G | Duplication de paquets (fibre plus 5G) | Flux monétique |
| Lien long | Optimisation TCP | Transferts volumineux |

### 11.6 MTU et optimisation

Le FEC et la duplication peuvent modifier la taille des paquets : tenir compte du **MTU** et du MSS (voir cours dédié).

---

## 12. Politiques de visibilité et de télémétrie

### 12.1 Objectifs

- Connaître les applications qui consomment la bande passante.
- Prouver la qualité d'un service aux utilisateurs ou aux fournisseurs.
- Alimenter l'analyse de sécurité.
- Planifier la capacité.

### 12.2 Types de politiques

| Type | Contenu |
|---|---|
| Visibilité applicative | Reconnaissance des applications (DPI) par VPN et par interface |
| Export de flux (type cflowd, NetFlow, IPFIX) | Envoi vers un collecteur |
| Mirroring | Copie d'un flux vers un outil d'analyse |
| Mesure de performance | Suivi de perte, latence, gigue par tunnel |
| Journalisation | Événements de sécurité et de politique |

### 12.3 Bonnes pratiques

- Activer la visibilité applicative avant d'écrire des politiques par application : on découvre ce qui circule réellement.
- Exporter les flux au bon niveau d'échantillonnage pour ne pas saturer le collecteur.
- Protéger la confidentialité des données collectées (RGPD).
- Conserver les journaux selon les durées réglementaires.

---

## 13. Politiques de gestion et de cycle de vie

### 13.1 Politiques de mise à jour

| Paramètre | Exemple |
|---|---|
| Version cible par profil de site | Version standard "N" pour tous les formats |
| Fenêtre de maintenance | Hypermarchés : 02h00 à 04h00, proximité : 03h00 à 05h00 |
| Vagues | Pilote, 1 %, 10 %, 100 % |
| Critères d'arrêt | Seuil d'échec déclenchant la pause |

### 13.2 Politiques de conformité

- Détection d'équipements hors version.
- Contrôle des configurations hors modèle (dérive de configuration).
- Contrôle de l'expiration des certificats.

### 13.3 Politiques d'accès et d'administration

| Sujet | Règle |
|---|---|
| RBAC | Rôles distincts : lecture, exploitation, administration, sécurité |
| Authentification forte | MFA pour l'accès au manager |
| Journalisation des changements | Qui a modifié quoi et quand |
| Approbation | Revue obligatoire avant activation d'une politique globale |

### 13.4 Politiques de continuité

- Utilisation du lien cellulaire limitée aux classes de trafic critiques.
- Plafond de consommation de données mobiles par site.
- Alertes à 80 % et 100 % du forfait.

---

## 14. Les politiques chez les autres éditeurs

Les concepts sont identiques, les noms et l'organisation diffèrent. Les informations ci-dessous sont **indicatives** : les gammes évoluent, vérifiez la documentation courante.

### 14.1 Correspondance générale

| Concept | Cisco Catalyst SD-WAN | Fortinet Secure SD-WAN | VMware VeloCloud | HPE Aruba EdgeConnect | Palo Alto Prisma SD-WAN |
|---|---|---|---|---|---|
| Choix de chemin selon qualité | Politique AAR avec classes SLA | Règles SD-WAN avec contrôles de santé (performance SLA) | Business Policy, link steering | Business Intent Overlay (BIO) | Politique de chemin (path policy set) |
| QoS | Politique QoS localisée | Shaping et profils de QoS | Classes de service et business policy | Politique QoS | Politique QoS (QoS policy set) |
| Sécurité | Pare-feu à zones, IPS, URL | Politiques de pare-feu FortiGate | Pare-feu d'entreprise | Pare-feu à zones | Politique de sécurité à zones |
| Sortie Internet locale | Data policy avec NAT | Règle SD-WAN vers interface Internet | Business Policy avec NAT direct | Politique de breakout | Politique de NAT et de chemin |
| Topologie | Politiques de contrôle | Configuration de zones et de superpositions | Profils et hubs | Orchestrator et overlays | Configuration de réseaux de sites |

### 14.2 Fortinet : règles SD-WAN

La logique repose sur :

- des **membres** SD-WAN (interfaces des liens) regroupés en zones ;
- des **contrôles de santé** (performance SLA) mesurant la perte, la latence et la gigue ;
- des **règles SD-WAN** (service rules) qui associent un trafic à une **stratégie** de sélection.

Stratégies courantes (selon version) :

| Stratégie | Principe |
|---|---|
| Manuelle | Interface désignée |
| Meilleure qualité | Lien avec le meilleur critère mesuré (latence, gigue, perte, bande passante) |
| Moindre coût (SLA) | Lien le moins coûteux qui respecte la SLA |
| Maximiser la bande passante (SLA) | Répartition sur les liens conformes |

Les politiques de pare-feu restent distinctes et s'appliquent à la zone SD-WAN.

### 14.3 VeloCloud : Business Policy

- Règles qui associent une application ou un flux à une **classe de service** (temps réel, transactionnel, volume).
- Choix de **mode de pilotage de lien** (par exemple privilégier un type de lien ou répartir la charge).
- Les mécanismes de correction (par exemple DMPO) agissent sur les liens dégradés.
- Pare-feu et NAT configurés dans des sections distinctes du profil.

### 14.4 HPE Aruba EdgeConnect : Business Intent Overlays

- Un **overlay** décrit l'intention pour une catégorie de trafic : chemins préférés, secours, QoS, sécurité.
- Politiques de routage, de QoS, de NAT et de sécurité gérées par l'orchestrateur.
- Optimisations (déduplication, compression, correction de perte) activables par overlay.

### 14.5 Palo Alto Prisma SD-WAN

- Ensembles de politiques (policy sets) pour : chemin, QoS, NAT, sécurité.
- Association de ces ensembles aux sites ou aux domaines.
- Intégration avec le service de sécurité cloud du même éditeur pour la sécurité Internet.

### 14.6 Versa Networks

- Règles par catégorie (SD-WAN, accès, QoS, NAT, sécurité, déchiffrement).
- Profils de transfert (forwarding profiles) et profils SLA pour le choix de chemin.
- Modèle multi-locataire dans le même outil de gestion.

### 14.7 Conseil de lecture

Lorsque vous découvrez un nouvel éditeur, cherchez les équivalents de ces cinq concepts :

1. où se définissent les **classes de qualité** (SLA) ;
2. où se choisit le **chemin préféré** ;
3. où se définit la **QoS** ;
4. où s'écrivent les **règles de sécurité** ;
5. comment on **applique** une politique à un ensemble de sites.

---

## 15. Ordre d'évaluation, conflits et priorités

### 15.1 Dans une politique

- Les séquences sont évaluées **de la plus petite à la plus grande** numérotation.
- La **première correspondance** s'applique (sauf action qui autorise la poursuite, selon la plateforme).
- Si rien ne correspond, l'**action par défaut** s'applique.

### 15.2 Entre politiques

Plusieurs politiques peuvent concerner le même flux : par exemple une **data policy** (NAT, marquage), une **AAR** (choix du tunnel), une **QoS** (file d'attente) et un **pare-feu** (autorisation). L'ordre d'intervention dépend de l'architecture de traitement de l'edge.

Schéma simplifié du traitement d'un paquet venant du LAN :

```
Paquet du LAN
   |
   v
[ACL / pare-feu d'entrée]  ---> bloqué ou autorisé
   |
   v
[Data policy]  ---> marquage, NAT, service, choix de VPN
   |
   v
[AAR]  ---> choix du tunnel selon la classe SLA
   |
   v
[Encapsulation IPsec]
   |
   v
[QoS de sortie WAN]  ---> file d'attente, shaping
   |
   v
Lien WAN
```

L'ordre exact varie selon la plateforme et la version. Validez-le dans la documentation et par des tests.

### 15.3 Conflits classiques

| Conflit | Effet |
|---|---|
| Deux politiques de données sur le même site avec des critères qui se recoupent | Comportement difficile à prévoir |
| Politique de contrôle qui filtre des routes que l'AAR attend | Chemin non disponible |
| QoS qui reclasse le DSCP posé par la data policy | Classe inattendue |
| Pare-feu bloquant un flux que la data policy dirige vers un service | Flux coupé |
| Politique d'un hub contredisant celle d'un spoke | Asymétrie de chemin |

### 15.4 Règle de lisibilité

- **Une politique, un objectif.**
- **Spécifique avant général.**
- **Pas de recouvrement** de critères entre séquences sans raison documentée.

---

## 16. Cas pratique : le catalogue de politiques d'un distributeur

### 16.1 Contexte

Groupe Distri (fictif) : environ 3 800 sites, quatre formats (hypermarchés, supermarchés, proximité, entrepôts), deux hubs (Paris, Lyon), VRF : 10 monétique, 20 caisses, 30 voix, 40 vidéo, 50 IoT, 60 invités.

### 16.2 Listes de base

| Liste | Contenu |
|---|---|
| `SITES_HYPER` | Sites hypermarchés (identifiants 1000 à 1999) |
| `SITES_SUPER` | Supermarchés (2000 à 2999) |
| `SITES_PROX` | Proximité (3000 à 3999) |
| `SITES_ENTREPOT` | Entrepôts (4000 à 4099) |
| `HUBS` | Hubs Paris et Lyon (100, 101) |
| `SITES_PILOTE` | Sites de test (mélange de formats) |
| `VPN_METIER` | VPN 10, 20, 30 |
| `VPN_ANNEXES` | VPN 40, 50 |
| `VPN_INVITES` | VPN 60 |
| `COULEURS_PRIMAIRES` | mpls, biz-internet |
| `COULEURS_SECOURS` | lte |

### 16.3 Matrice d'applications

| Application | VPN | Chemin préféré | Secours | SLA | QoS | Optimisation | Sécurité |
|---|---|---|---|---|---|---|---|
| Paiement | 10 | MPLS ou fibre | 5G | SLA_PAIEMENT | AF41, bande garantie | Duplication sur 5G | Zone isolée, pas de DIA |
| Caisses vers ERP | 20 | MPLS ou fibre | 5G | SLA_METIER | AF31 | TCP opt | Zone caisses |
| Voix IP | 30 | MPLS ou fibre | 5G | SLA_VOIX | EF, LLQ | FEC adaptatif | Zone voix |
| Vidéosurveillance (export) | 40 | Fibre | Aucun | SLA_TOLERANT | CS1 | Aucune | Zone vidéo |
| IoT (froid, étiquettes) | 50 | Fibre | 5G (alertes seules) | SLA_METIER | AF21 | Aucune | Zone IoT |
| Microsoft 365 | 20 | DIA local | Hub | Non | AF21 | Aucune | Filtrage URL et DNS |
| Wi-Fi invités | 60 | DIA local | Aucun | Non | Défaut ou bulk | Aucune | Filtrage strict, isolation |
| Mises à jour et sauvegardes | 20 | Fibre | Aucun | SLA_TOLERANT | CS1 | Aucune | Horaires creux |
| Supervision et gestion | 512 | Tous | Tous | Non | CS6 | Aucune | Accès restreint |

### 16.4 Politiques qui en découlent

| Politique | Type | Portée |
|---|---|---|
| `POL_TOPO_HUB_SPOKE` | Contrôle | Tous les magasins |
| `POL_VPN_MEMBERSHIP` | Contrôle | Par format |
| `POL_AAR_RETAIL` | AAR | Magasins |
| `POL_DATA_DIA` | Données | Sites avec DIA |
| `POL_DATA_MARQUAGE` | Données | Tous |
| `POL_DATA_FEC_DUPLICATION` | Données | Sites avec 5G |
| `POL_QOS_WAN_PROX` | QoS localisée | Proximité |
| `POL_QOS_WAN_HYPER` | QoS localisée | Hypermarchés |
| `POL_FW_ZONES_MAG` | Sécurité | Tous les magasins |
| `POL_CFLOWD_EXPORT` | Visibilité | Tous |
| `POL_UPGRADE_WINDOWS` | Gestion | Par format |

### 16.5 Variantes par format

| Format | Particularité de politique |
|---|---|
| Hypermarché | QoS plus granulaire, pare-feu local renforcé, hautes performances, mesures plus fines |
| Supermarché | Profil intermédiaire, DIA pour M365 |
| Proximité | Plafond de données mobile, duplication limitée au paiement, QoS simple |
| Entrepôt | Priorité aux flux logistiques, scanners radio, QoS dédiée |

---

## 17. Scénarios dynamiques : Black Friday, panne, mode dégradé

### 17.1 Black Friday dans un hypermarché

Objectif : garantir les caisses malgré un pic de trafic.

Actions possibles (préparées en amont, activées par changement de politique ou de liste) :

1. Augmenter la part de bande passante de la classe monétique et caisses.
2. Passer les sauvegardes et mises à jour en classe bulk avec plafond strict.
3. Activer la duplication de paquets sur les flux monétiques pendant la période.
4. Bloquer ou limiter les flux non essentiels (streaming sur le Wi-Fi invités).
5. Renforcer la supervision (seuils d'alerte plus bas, tableau de bord dédié).

Bonne pratique : **préparer une version "événement"** de la politique, testée en pré-production, activée par un simple changement de liste de sites.

### 17.2 Fibre coupée dans un magasin de proximité

| Étape | Comportement |
|---|---|
| T+0 | La fibre tombe |
| T+quelques secondes | La détection de panne du tunnel bascule |
| Immédiat | La politique AAR dirige les flux de la liste "critiques" sur le tunnel 5G |
| Immédiat | La politique de données bloque vidéosurveillance, mises à jour et invités sur le lien mobile |
| Ensuite | Alerte, ticket automatique chez le fournisseur d'accès |
| Rétablissement | Retour automatique des flux sur la fibre |

Pseudo-règle : "sur le transport `lte`, autoriser seulement les VPN 10, 20, 30 et un minimum de supervision".

### 17.3 Mode dégradé "forfait mobile épuisé"

Si le plafond de données mobiles est atteint, une politique peut :

- limiter encore plus la liste des flux autorisés (paiement et voix seulement) ;
- ou, selon le contrat, passer le lien en bas débit, avec alerte vers l'équipe réseau.

### 17.4 Incident dans un hub

La politique de topologie préfère le hub Paris pour le nord et Lyon pour le sud. En cas de panne d'un hub :

- les magasins basculent sur l'autre hub (TLOC de secours) ;
- la capacité de chaque hub doit être dimensionnée pour absorber 100 % de la charge d'un hub défaillant (principe N+1).

### 17.5 Maintenance programmée

Politique "fenêtre de maintenance" : limite temporairement le trafic de sauvegarde et de mise à jour, active des seuils d'alerte différents, désactive les bascules automatiques pour certains équipements en intervention.

---

## 18. Gouvernance : conception, versionnement, tests, retour arrière

### 18.1 Nommage

Convention recommandée : `TYPE_OBJET_PERIMETRE_Vn`

Exemples :

- `AAR_RETAIL_PROX_V3`
- `DATA_DIA_M365_ALL_V2`
- `QOS_WAN_HYPER_V1`
- `FW_ZONES_MAG_V4`

### 18.2 Cycle de changement

```
Demande de changement
   |
   v
Conception (listes, séquences, impact)
   |
   v
Revue par les pairs
   |
   v
Test en laboratoire ou en pré-production
   |
   v
Activation sur SITES_PILOTE
   |
   v
Observation (indicateurs, 24 à 72 heures)
   |
   v
Déploiement par vagues (1 %, 10 %, 100 %)
   |
   v
Clôture et documentation
```

### 18.3 Versionnement

- Exporter la configuration des politiques (format texte ou JSON) vers un dépôt Git.
- Décrire chaque changement (objectif, date, auteur, retour arrière).
- Conserver l'historique pour l'audit (PCI DSS notamment).

### 18.4 Retour arrière

- Prévoir pour chaque politique la procédure de désactivation ou de restauration de la version précédente.
- Tester le retour arrière aussi souvent que le déploiement.
- Définir les critères de déclenchement (par exemple dégradation du taux d'échec de paiement).

### 18.5 Tests

| Test | Contenu |
|---|---|
| Fonctionnel | La règle fait-elle ce qui est prévu ? |
| Non-régression | Les autres flux sont-ils inchangés ? |
| Bascule | Que se passe-t-il quand un lien tombe ? |
| Charge | Comportement sous trafic élevé |
| Sécurité | Les flux interdits sont-ils bloqués ? |

### 18.6 Indicateurs de qualité des politiques

| Indicateur | Cible indicative |
|---|---|
| Nombre de politiques actives par type | Aussi bas que possible |
| Nombre de séquences par politique | Moins de 20 en règle générale |
| Temps moyen de déploiement d'un changement | Inférieur à une journée en pré-production |
| Taux d'incidents liés à un changement de politique | Inférieur à 1 % |
| Part des politiques documentées | 100 % |

### 18.7 Séparation des environnements

Laboratoire, pré-production, production avec des **listes de sites** différentes, de sorte qu'une même politique puisse être activée progressivement.

---

## 19. Dépannage des politiques

### 19.1 Méthode

1. **Vérifier que la politique est bien reçue** par l'edge.
2. **Vérifier l'application** au bon site, dans la bonne direction et sur le bon VPN.
3. **Simuler un flux** pour voir le chemin effectivement choisi.
4. **Contrôler les compteurs** de la séquence concernée.
5. **Vérifier les mesures de qualité** des tunnels (perte, latence, gigue).
6. **Examiner les interactions** avec QoS, pare-feu et NAT.

### 19.2 Commandes utiles (exemples Cisco, à adapter à la version)

```
show sdwan policy from-vsmart
show sdwan policy data-policy-filter
show sdwan policy app-route-policy-filter
show sdwan app-route stats
show sdwan app-route sla-class
show sdwan policy service-path vpn 20 interface GigabitEthernet0/1/0.20 source-ip 10.20.34.10 dest-ip 10.100.10.5 protocol 6 dest-port 443 all
show sdwan omp routes
show sdwan omp tlocs
show sdwan bfd sessions
```

### 19.3 Tableau de diagnostic

| Symptôme | Causes probables |
|---|---|
| La politique n'apparaît pas sur l'edge | Non activée, site hors de la liste, problème de connexion de contrôle |
| La règle ne correspond jamais | Mauvaise direction, VPN absent de la liste, critère trop précis, ordre des séquences |
| Le flux ne suit pas le chemin attendu | SLA non respectée sur le chemin préféré, couleur absente, tunnel down |
| Bascules incessantes (flapping) | Seuils SLA trop serrés, intervalle de mesure trop court |
| Voix dégradée malgré la QoS | Pas de shaping, marquage perdu, opérateur qui ne respecte pas le DSCP |
| Paiement refusé après changement de politique | Pare-feu ou NAT interfère, mauvaise classe SLA, DSCP modifié |
| Sortie Internet locale non fonctionnelle | NAT non appliqué, route absente dans le VPN, DNS |

### 19.4 Démarche de simulation

Utiliser la commande de **service-path** (ou son équivalent) pour demander : "pour ce flux (source, destination, protocole, port), quel chemin la politique sélectionne-t-elle ?". C'est l'outil le plus efficace pour valider une règle avant de lancer des tests réels.

### 19.5 Capture et comparaison

- Capture côté LAN et côté WAN de l'edge pour vérifier le DSCP, le tunnel utilisé, le NAT.
- Comparaison avec un site de référence pour isoler un problème de configuration locale.

---

## 20. Bonnes pratiques et pièges

### Bonnes pratiques

1. **Partir des applications et des besoins métier**, pas de la technologie.
2. **Utiliser des listes** pour tout ce qui varie (sites, applications, couleurs).
3. **Garder peu de politiques**, claires, documentées.
4. **Spécifique avant général** dans les séquences.
5. **Vérifier l'action par défaut** de chaque type de politique.
6. **Activer d'abord la visibilité** pour connaître le trafic réel.
7. **Calibrer les SLA sur des mesures**, pas sur des valeurs théoriques.
8. **Combiner AAR, QoS et shaping** : chacun traite un problème différent.
9. **Prévoir les modes dégradés** (5G, forfait épuisé, hub indisponible).
10. **Tester les bascules** et le **retour arrière**.
11. **Déployer par vagues** avec des sites pilotes.
12. **Versionner et auditer** les changements.

### Pièges

- Écrire des règles par application sans avoir observé le trafic réel.
- Mettre des seuils SLA irréalistes, provoquant du flapping.
- Oublier le shaping : la QoS ne sert à rien si la file d'attente est chez l'opérateur.
- Appliquer une règle dans la mauvaise direction.
- Laisser une action par défaut "rejeter" sur une politique de contrôle sans l'avoir mesurée.
- Autoriser trop de trafic sur le lien mobile.
- Dupliquer tout le trafic sur deux liens : doublement de la consommation.
- Cumuler des politiques qui se recouvrent sans l'avoir documenté.
- Faire un changement global le vendredi soir.
- Ne pas prévoir de retour arrière.

---

## 21. Exercices et corrigés

### Exercice 1 : classification

**Énoncé.** Classez chaque besoin dans une famille de politique : (a) isoler les magasins entre eux sauf pour la voix régionale, (b) mettre la voix en priorité stricte, (c) envoyer Microsoft 365 en sortie locale, (d) basculer le paiement sur la 5G si la perte dépasse 1 %, (e) bloquer les catégories de sites à risque pour les invités.

**Corrigé.** (a) Contrôle (topologie). (b) QoS localisée. (c) Données avec NAT (DIA). (d) AAR avec classe SLA. (e) Sécurité (filtrage URL ou DNS).

### Exercice 2 : classes SLA

**Énoncé.** Proposez des seuils pour une classe "SLA_PAIEMENT" et justifiez.

**Corrigé (exemple).** Perte 1 % ou moins, latence 150 ms ou moins, gigue 50 ms ou moins. Les transactions sont de petits échanges sensibles à la perte et aux retransmissions. Les seuils se calibrent ensuite sur les mesures réelles du parc.

### Exercice 3 : ordre des séquences

**Énoncé.** Une politique de données comporte : séquence 10 "tout le VPN 20 vers Internet : DIA" et séquence 20 "Microsoft 365 : DIA, DSCP AF21". Que se passe-t-il ?

**Corrigé.** Tout le trafic du VPN 20 correspond à la séquence 10 : la séquence 20 n'est jamais atteinte, donc Microsoft 365 n'est pas marqué AF21. Il faut placer la séquence spécifique (Microsoft 365) avant la séquence générale.

### Exercice 4 : QoS et shaping

**Énoncé.** Un lien fibre de 100 Mbit/s est configuré avec une QoS sans shaping. La voix reste hachée aux heures de pointe. Pourquoi ?

**Corrigé.** Sans shaping, l'edge envoie au débit de l'interface physique, la congestion se forme chez l'opérateur, où la QoS de l'edge n'agit pas. Il faut appliquer un shaping légèrement inférieur au débit contractuel (par exemple 95 Mbit/s).

### Exercice 5 : mode dégradé

**Énoncé.** Définissez la politique d'un magasin de proximité qui bascule sur la 5G, avec un forfait limité à 50 Go par mois.

**Corrigé (exemple).**

| Élément | Décision |
|---|---|
| Flux autorisés sur 5G | Paiement, caisses, voix, supervision minimale |
| Flux bloqués ou limités | Vidéosurveillance, mises à jour, invités, bulk |
| Optimisations | Duplication seulement pour le paiement |
| Alertes | À 80 % et 100 % du forfait |
| Mesure | Suivi de la consommation par site |

### Exercice 6 : conflit

**Énoncé.** Après l'activation d'un pare-feu à zones, un flux monétique redirigé vers le pare-feu du hub par service chaining échoue. Proposez trois pistes.

**Corrigé.**

1. La règle du pare-feu local interdit le flux entre les zones concernées.
2. La route de service n'est pas annoncée correctement vers le hub (politique de contrôle).
3. Le NAT ou le marquage de la data policy modifie le paquet de façon incompatible avec la règle du pare-feu du hub.

### Exercice 7 : conception complète

**Énoncé.** Pour un supermarché (fibre plus MPLS, pas de 5G), listez les politiques à activer et leurs paramètres clés.

**Corrigé (exemple).**

| Politique | Paramètres clés |
|---|---|
| Topologie | Hub and spoke avec deux hubs |
| VPN membership | VPN 10, 20, 30, 40, 60 |
| AAR | Paiement et voix : MPLS puis fibre, SLA stricte ; caisses : MPLS ou fibre |
| Données | DIA pour M365, marquage DSCP |
| QoS | LLQ voix, bande garantie paiement et caisses, bulk plafonné, shaping |
| Sécurité | Zones monétique, caisses, voix, vidéo, invités ; filtrage URL pour invités |
| Visibilité | Export de flux vers le collecteur |
| Gestion | Fenêtre de mise à jour la nuit |

---

## 22. Quiz de révision

1. Quelle est la différence entre une politique centralisée et une politique localisée ?
2. À quoi sert une politique de contrôle ?
3. Que mesure une classe SLA et quelles sont ses trois métriques classiques ?
4. Quelle est la différence entre AAR et QoS ?
5. Pourquoi faut-il un shaping sur les liens WAN ?
6. Que signifie "from-service" dans une politique de données ?
7. Pourquoi placer les séquences spécifiques avant les séquences générales ?
8. Quel risque présente une action par défaut "rejeter" sur une politique de contrôle ?
9. Citez deux mécanismes d'optimisation sur un lien dégradé.
10. Pourquoi versionner les politiques ?
11. Pourquoi ne pas dupliquer tous les flux sur deux liens ?
12. Quel est l'avantage d'une politique "événement" préparée pour le Black Friday ?

**Éléments de réponse**

1. La première est définie au niveau du manager et appliquée à un ensemble de sites, la seconde est appliquée par template à un équipement ou une interface.
2. Elle modifie les routes et TLOC annoncés, donc la topologie logique de l'overlay.
3. Les seuils acceptables de perte, de latence et de gigue.
4. L'AAR choisit le chemin selon la qualité mesurée, la QoS gère la contention sur un lien.
5. Pour que la file d'attente se forme sur l'edge, où la QoS peut agir, et non chez l'opérateur.
6. Le trafic entrant sur l'edge depuis le réseau local du site.
7. Parce que la première séquence qui correspond s'applique : une règle générale placée avant masquerait les règles spécifiques.
8. Isoler des sites ou rendre des routes inaccessibles par omission.
9. FEC, duplication de paquets (aussi optimisation TCP).
10. Pour tracer les changements, faciliter le retour arrière, répondre aux audits.
11. Doublement de la consommation de bande passante, donc coût et saturation possibles.
12. Elle est testée à l'avance, activable rapidement et sans improvisation.

---

## 23. Glossaire

| Terme | Définition |
|---|---|
| AAR | Application-Aware Routing, choix du chemin selon la qualité mesurée et l'application |
| ACL | Access Control List, liste de règles de filtrage |
| Action par défaut | Comportement appliqué quand aucune séquence ne correspond |
| BIO | Business Intent Overlay, concept d'intention chez Aruba EdgeConnect |
| Cflowd | Export de flux de trafic (type NetFlow) dans l'écosystème Cisco SD-WAN |
| Classe SLA | Ensemble de seuils de perte, de latence et de gigue |
| Control policy | Politique de contrôle agissant sur les routes et TLOC |
| Data policy | Politique agissant sur les paquets du plan de données |
| DIA | Direct Internet Access, sortie Internet locale |
| DSCP | Differentiated Services Code Point, marquage de priorité IP |
| FEC | Forward Error Correction, correction d'erreurs par redondance |
| LLQ | Low Latency Queuing, file à priorité stricte |
| Liste | Objet réutilisable (sites, VPN, applications, préfixes) |
| Match | Condition de correspondance d'une séquence |
| Policy | Politique, ensemble de règles exprimant une intention |
| QoS | Quality of Service, gestion des priorités et de la bande passante |
| Séquence | Règle numérotée d'une politique |
| Service chaining | Passage d'un flux par un service intermédiaire (pare-feu, inspection) |
| Shaping | Mise en forme du débit sortant pour éviter la congestion chez l'opérateur |
| SSE | Security Service Edge, services de sécurité fournis depuis le cloud |
| TLOC | Transport Locator, point d'attache d'un edge à un transport |
| VPN membership | Politique qui définit quels VPN sont visibles par quels sites |
| ZTNA | Zero Trust Network Access, accès applicatif fondé sur l'identité et le contexte |

---

## Pour aller plus loin

- Documentation officielle de la plateforme utilisée : guides de politiques centralisées, localisées et de sécurité.
- Guides de conception validés par les éditeurs (cas retail, multi-cloud).
- Laboratoires virtuels pour tester des politiques sans risque.
- Référentiels de sécurité : PCI DSS, recommandations de l'ANSSI.

---

*Fin du cours.*
