# Cours complet SD-WAN et protocole BFD (Bidirectional Forwarding Detection)

Ce cours fait suite au cours sur OMP. Après avoir vu comment le plan de contrôle distribue les routes et les TLOC, on étudie ici le BFD, le mécanisme qui surveille en permanence la santé du plan de données : détection de panne des tunnels IPsec et mesure de la qualité (perte, latence, gigue) qui alimente le routage applicatif. Le fil rouge reste le réseau d'une enseigne de grande distribution de type Carrefour.

> Note importante : le cas Carrefour est un scénario pédagogique fictif. Les plans d'adressage, les identifiants de sites, les volumes et les mesures sont inventés à des fins d'illustration et ne décrivent pas l'architecture réelle de l'enseigne.

---

## Table des matières

1. Objectifs et prérequis
2. Rappel : BFD en dehors du SD-WAN
3. BFD dans Cisco SD-WAN : principe
4. Les deux rôles de BFD : détection de panne et mesure de qualité
5. Paramètres BFD et leur réglage
6. Lien entre BFD et OMP
7. Le routage applicatif (Application-Aware Routing) et les classes SLA
8. Cas d'étude fil rouge : le réseau d'une enseigne de distribution
9. Configuration pas à pas
10. Vérification et dépannage
11. Dimensionnement et passage à l'échelle
12. BFD côté LAN (BGP, OSPF) : ne pas confondre
13. Bonnes pratiques
14. Travaux pratiques
15. Questionnaire d'auto-évaluation
16. Glossaire
17. Références

---

## 1. Objectifs et prérequis

### Objectifs pédagogiques

À la fin de ce cours, vous serez capable de :

- Expliquer pourquoi le SD-WAN a besoin de BFD et comment il fonctionne dans l'overlay.
- Distinguer les deux usages : détection de panne et mesure de qualité du lien.
- Régler les timers (hello, multiplicateur, poll interval) selon le type de transport.
- Expliquer le lien entre état BFD, résolution des TLOC et routes OMP installées.
- Construire une politique de routage applicatif avec des classes SLA.
- Lire les sorties `show sdwan bfd ...` et `show sdwan app-route ...`.
- Dimensionner le nombre de sessions BFD d'un hub.

### Prérequis

- Le cours sur OMP (TLOC, color, politiques de contrôle).
- Notions de tunnels IPsec et de QoS (DSCP).

---

## 2. Rappel : BFD en dehors du SD-WAN

BFD (Bidirectional Forwarding Detection) est défini par la RFC 5880 (mécanisme générique) et la RFC 5881 (BFD pour IPv4 et IPv6 sur un saut unique). C'est un protocole de détection de panne léger :

- Les deux extrémités échangent des paquets de hello à intervalle très court.
- Si aucun paquet n'est reçu pendant `intervalle x multiplicateur`, le lien est déclaré en panne.
- Il est indépendant du protocole de routage, ce qui permet de détecter une panne en moins d'une seconde là où OSPF ou BGP prendraient plusieurs secondes à plusieurs dizaines de secondes.

Exemple générique : avec un hello de 300 ms et un multiplicateur de 3, la panne est détectée en 900 ms environ.

---

## 3. BFD dans Cisco SD-WAN : principe

### 3.1 Une session BFD par tunnel IPsec

Dans Cisco SD-WAN, BFD est utilisé sur chaque tunnel du plan de données. Dès qu'un tunnel IPsec est établi entre deux TLOC, une session BFD démarre automatiquement à l'intérieur du tunnel.

Points clés :

- BFD est actif par défaut sur tous les tunnels entre WAN Edge et ne peut pas être désactivé.
- Les paquets BFD circulent à l'intérieur du tunnel chiffré : ils testent donc le chemin réel emprunté par les données, chiffrement compris.
- Il y a une session BFD par paire de TLOC (couleur locale vers couleur distante).
- Cette implémentation est propre à la solution SD-WAN (elle s'inspire du BFD standard mais ne s'interconnecte pas avec les équipements tiers).

### 3.2 Schéma

```
   Magasin 2750                                   Datacenter Paris
  +--------------+                               +-----------------+
  | TLOC         |=== tunnel IPsec (BFD) ========| TLOC mpls       |
  | biz-internet |=== tunnel IPsec (BFD) ========| TLOC biz-intern.|
  |              |                               |                 |
  | TLOC lte     |=== tunnel IPsec (BFD) ========| TLOC lte (si    |
  +--------------+                               | présent)        |
                                                 +-----------------+
   Chaque tunnel porte une session BFD indépendante.
```

Par défaut, un tunnel ne se monte qu'entre TLOC de même couleur (sauf configuration spécifique ou couleur privée/publique compatible). Une session BFD correspond donc à une paire de TLOC compatibles.

### 3.3 Paquets BFD et QoS

Les paquets BFD sont marqués avec un DSCP par défaut élevé (48 dans les valeurs usuelles) afin d'être traités en priorité sur le transport. Si l'opérateur applique un traitement QoS restrictif, la mesure peut refléter une qualité différente de celle des flux utilisateur. Il faut donc aligner le marquage avec la classe de service attendue.

---

## 4. Les deux rôles de BFD

### 4.1 Détection de panne (liveness)

BFD détecte la perte d'un tunnel. Si aucun paquet n'est reçu pendant `hello x multiplicateur`, la session passe à `down`. Le TLOC associé devient inutilisable pour ce chemin et le trafic bascule sur un autre tunnel disponible.

Valeurs usuelles par défaut :

| Paramètre | Valeur par défaut | Détection |
|---|---|---|
| Hello interval | 1000 ms | |
| Multiplier | 7 | 7 secondes pour détecter la panne |

### 4.2 Mesure de qualité (performance)

BFD sert aussi de sonde de mesure. Sur chaque tunnel, le WAN Edge calcule :

- Le taux de perte de paquets.
- La latence.
- La gigue (variation de délai).

Ces mesures sont agrégées sur un intervalle appelé `app-route poll interval` (par défaut 600000 ms, soit 10 minutes) et conservées sur plusieurs intervalles (par défaut un multiplicateur de 6). Elles alimentent le routage applicatif décrit en section 7.

Conséquence : la détection de panne (secondes) et la détection de dégradation de qualité (dizaines de secondes à minutes) sont deux échelles de temps différentes, réglables séparément.

---

## 5. Paramètres BFD et leur réglage

### 5.1 Paramètres globaux

| Paramètre | Rôle | Valeur par défaut usuelle |
|---|---|---|
| `hello-interval` (par couleur) | Cadence des paquets BFD | 1000 ms |
| `multiplier` (par couleur) | Nombre de hello manqués avant down | 7 |
| `app-route poll-interval` | Fenêtre d'agrégation des mesures | 600000 ms |
| `app-route multiplier` | Nombre de fenêtres conservées | 6 |
| `default-dscp` | Marquage DSCP des paquets BFD | 48 |
| `pmtu-discovery` | Découverte du MTU de chemin | activée |

Les valeurs par défaut et les bornes autorisées dépendent de la version logicielle et de la plateforme : vérifiez-les dans la documentation de votre release.

### 5.2 Réglage par couleur

Les paramètres BFD se règlent par couleur de transport, car les besoins diffèrent :

| Transport | Contrainte | Réglage typique |
|---|---|---|
| MPLS, fibre dédiée | Lien stable, pas de coût au volume | Valeurs par défaut ou plus agressives pour une bascule rapide |
| Internet haut débit | Qualité variable | Valeurs par défaut |
| 4G/5G | Forfait de données, consommation à limiter | Hello espacé (par exemple 10 s ou plus) |

### 5.3 Compromis de réglage

| Choix | Avantage | Inconvénient |
|---|---|---|
| Hello court, multiplicateur faible | Bascule rapide | Plus de faux positifs sur un lien instable, plus de charge CPU et de bande passante, flaps de tunnels |
| Hello long, multiplicateur élevé | Stabilité, économie de données | Bascule lente |
| Poll interval court | Réaction rapide aux dégradations | Mesures plus bruitées, risque d'oscillation entre chemins |
| Poll interval long | Mesures lissées | Réaction lente |

### 5.4 Estimation de la consommation sur un lien 4G

Ordre de grandeur pédagogique : un paquet BFD encapsulé en IPsec représente environ 150 octets (valeur approximative selon le chiffrement et l'encapsulation). À un paquet par seconde et par sens, une session consomme de l'ordre de 150 à 300 octets par seconde, soit environ 13 à 26 Mo par jour et par session. Avec plusieurs sessions sur le lien cellulaire, la consommation devient significative sur un forfait limité. C'est pourquoi on espace les hello sur la couleur `lte`.

---

## 6. Lien entre BFD et OMP

### 6.1 Rappel : TLOC résolu

Une route OMP n'est utilisable que si son TLOC (next hop) est résolu, c'est-à-dire qu'un tunnel existe et que la session BFD correspondante est `up`.

### 6.2 Chaîne de dépendance

```
 Session BFD up  ->  tunnel utilisable  ->  TLOC résolu (statut R)
                                      ->  route OMP installée (C,I,R)
 Session BFD down ->  tunnel inutilisable -> TLOC non résolu (U)
                                      ->  route retirée de la table de routage
                                      ->  trafic bascule sur un autre chemin
```

### 6.3 Ce qui est important à comprendre

- OMP annonce les routes et les TLOC (plan de contrôle). BFD valide qu'ils sont réellement joignables (plan de données).
- Sans BFD, une route OMP pourrait rester annoncée alors que le tunnel est cassé.
- La bascule locale est donc très rapide (décision du WAN Edge sur la base de BFD), sans attendre une réaction du vSmart.
- Si tous les tunnels vers un site distant tombent, les routes dont les TLOC sont non résolus ne sont plus installées.

### 6.4 Effet de la politique hub and spoke

Dans un design hub and spoke (voir le cours OMP, section 10.2), la politique de contrôle empêche les magasins de recevoir les TLOC des autres magasins. Il n'y a donc aucune session BFD entre magasins. Le nombre de sessions BFD est ainsi directement piloté par la politique OMP : la topologie logique détermine le nombre de tunnels, donc de sessions BFD à entretenir.

---

## 7. Le routage applicatif (Application-Aware Routing)

### 7.1 Principe

Le routage applicatif (AAR) choisit, pour chaque flux, un tunnel dont la qualité mesurée par BFD respecte un SLA défini pour l'application. Si le tunnel actuel ne respecte plus le SLA, le flux est déplacé vers un tunnel conforme.

### 7.2 Éléments

- `sla-class` : seuils maximaux de perte, latence et gigue.
- `app-route-policy` : séquences associant un type de trafic (application, DSCP, préfixe) à une classe SLA, avec couleurs préférées et secours éventuels.
- Politique centralisée, définie dans vManage et poussée vers les WAN Edge.

### 7.3 Exemple de classes SLA

```
policy
 sla-class SLA-CAISSES
  loss    1
  latency 100
  jitter  30
 !
 sla-class SLA-BUREAUTIQUE
  loss    5
  latency 300
  jitter  100
 !
 sla-class SLA-VIDEOSURVEILLANCE
  loss    3
  latency 200
  jitter  80
 !
!
```

Les unités : perte en pourcentage, latence et gigue en millisecondes.

### 7.4 Exemple de politique

```
policy
 lists
  vpn-list VPN-CAISSES
   vpn 10
  !
  vpn-list VPN-BACKOFFICE
   vpn 20
  !
  site-list MAGASINS
   site-id 2000-2999
  !
 !
 app-route-policy AAR-MAGASINS
  vpn-list VPN-CAISSES
   sequence 10
    match
     dscp 46
    !
    action
     sla-class SLA-CAISSES
      preferred-color biz-internet
     !
    !
   !
   default-action
    sla-class SLA-CAISSES
   !
  !
  vpn-list VPN-BACKOFFICE
   sequence 10
    match
     dscp 18
    !
    action
     sla-class SLA-BUREAUTIQUE
      preferred-color biz-internet
     !
    !
   !
  !
 !
!
apply-policy
 site-list MAGASINS
  app-route-policy AAR-MAGASINS
 !
!
```

La syntaxe exacte des actions (couleurs préférées, secours, option stricte) varie selon la version : en pratique ces politiques se construisent dans l'interface vManage.

### 7.5 Comportement

1. BFD mesure la perte, la latence et la gigue de chaque tunnel.
2. Le WAN Edge compare les mesures aux seuils de la classe SLA.
3. Les tunnels conformes forment l'ensemble éligible, la couleur préférée est choisie en priorité.
4. Si aucun tunnel n'est conforme, le comportement dépend de la politique : choix du moins mauvais tunnel, ou option stricte qui supprime le trafic (à manier avec prudence).
5. Le trafic ne bascule qu'après dégradation constatée sur la fenêtre de mesure, d'où le réglage du poll interval.

### 7.6 Attention aux oscillations

Si deux tunnels sont proches du seuil, le trafic peut osciller entre eux. Pour limiter ce phénomène :

- Utiliser un poll interval raisonnable.
- Fixer des seuils réalistes, avec une marge par rapport aux performances habituelles.
- Surveiller l'historique des changements de chemin.

---

## 8. Cas d'étude fil rouge : le réseau d'une enseigne de distribution

> Rappel : scénario fictif à vocation pédagogique.

### 8.1 Rappel du contexte

- 2 datacenters (Paris, site-id 1000 et Lyon, site-id 1001).
- Environ 1000 magasins, avec des transports différents selon la taille : hypermarchés (MPLS + 2 internet), supermarchés (MPLS + internet), proximité (internet + 4G/5G).
- VPN 10 caisses (PCI DSS), VPN 20 back office, VPN 30 vidéosurveillance, VPN 40 invités, VPN 50 IoT.
- Topologie hub and spoke imposée par OMP.

### 8.2 Besoin 1 : ne jamais interrompre les caisses

Un magasin de proximité a deux couleurs : `biz-internet` (nominal) et `lte` (secours). Vers chaque datacenter, il monte deux tunnels, un par couleur.

- BFD par défaut sur biz-internet : hello 1 s, multiplicateur 7, soit 7 secondes de détection.
- BFD allégé sur lte : hello 10 s, multiplicateur 7, soit 70 secondes de détection, acceptable car ce tunnel n'est utilisé qu'en secours (et un tunnel déjà monté est immédiatement disponible à la bascule).

Si la fibre tombe, BFD passe à `down` en quelques secondes sur biz-internet. Les routes OMP dont le next hop est ce TLOC ne sont plus résolues, et le trafic des caisses emprunte les tunnels lte vers les datacenters.

Pour les caisses, on peut réduire le délai de détection : hello 500 ms et multiplicateur 4 donnent environ 2 secondes. Il faut alors valider la stabilité du lien (risque de faux positifs).

### 8.3 Besoin 2 : une sauvegarde qui dépasse la simple panne

Une liaison internet peut ne pas tomber mais se dégrader (perte de paquets en soirée, congestion). Les caisses sont sensibles à la latence et à la perte (timeouts de paiement). La classe SLA-CAISSES (perte 1 %, latence 100 ms, gigue 30 ms) permet de déplacer le trafic des caisses vers un tunnel conforme, par exemple le MPLS ou la 4G, dès que le tunnel nominal dépasse les seuils.

### 8.4 Besoin 3 : limiter la consommation 4G

Chaque magasin de proximité entretient des sessions BFD sur le lien cellulaire vers les deux datacenters (2 sessions, 1 par datacenter, si chaque datacenter expose une couleur lte ou compatible). Avec un hello de 1 s, la consommation de données de supervision seule serait inutilement élevée sur un parc de plusieurs centaines de magasins. Le choix d'un hello à 10 s sur `lte` divise cette consommation par dix.

### 8.5 Besoin 4 : dimensionner les hubs

Calcul pédagogique pour un routeur hub du datacenter de Paris :

- 1000 magasins.
- Chaque magasin a 2 couleurs vers ce hub.
- Le hub expose 2 couleurs (mpls, biz-internet).

Nombre de tunnels compatibles par magasin vers ce hub : jusqu'à 2 (les paires de même couleur), soit 2000 sessions BFD pour ce hub, ou jusqu'à 4000 si toutes les combinaisons de couleurs sont autorisées. À 1 hello par seconde et par session, cela représente de l'ordre de 2000 à 4000 paquets par seconde en émission et autant en réception, uniquement pour BFD.

En topologie full mesh, le nombre de sessions croîtrait de façon quadratique : plusieurs millions de tunnels pour 1000 sites. C'est une des raisons pour lesquelles on impose un hub and spoke pour les magasins : la politique de contrôle OMP limite les tunnels, donc les sessions BFD.

### 8.5.1 Conséquence pour la conception

- Répartir les magasins sur plusieurs routeurs hub (par région ou par tranche de site-id).
- Vérifier les limites de tunnels et de sessions BFD de la plateforme choisie (documentées par Cisco, dépendantes du modèle et de la version).
- Éviter de descendre les timers BFD des hubs sans avoir mesuré l'impact CPU.

### 8.6 Scénarios de panne

| Événement | Rôle de BFD | Résultat |
|---|---|---|
| Coupure franche de la fibre d'un magasin | Détection en quelques secondes (selon timers) | Bascule vers lte |
| Lien internet dégradé (perte 4 %) | Mesure de perte via BFD, SLA violé | Le trafic caisses passe sur un tunnel conforme |
| Panne du MPLS entre le magasin et Paris | Session BFD down sur ces tunnels | Trafic via internet vers Paris, ou vers Lyon selon préférences OMP |
| Datacenter de Paris injoignable | Toutes les sessions BFD vers Paris down | Les routes de Paris ne sont plus résolues, trafic vers Lyon |
| Les deux vSmart tombent | Aucun effet sur BFD, qui est dans le plan de données | Les tunnels et la détection de panne continuent de fonctionner |

Ce dernier point est important : BFD continue à fonctionner sans contrôleur. Le magasin peut donc toujours basculer entre les chemins déjà connus pendant une panne du plan de contrôle.

---

## 9. Configuration pas à pas

### 9.1 Paramètres BFD sur un cEdge (IOS XE SD-WAN)

```
sdwan
 bfd default-dscp 48
 bfd app-route multiplier 6
 bfd app-route poll-interval 600000
 bfd color biz-internet
  hello-interval 1000
  multiplier 7
  pmtu-discovery
 exit
 bfd color lte
  hello-interval 10000
  multiplier 7
  no pmtu-discovery
 exit
!
```

### 9.2 Paramètres BFD sur un vEdge (Viptela OS)

```
bfd app-route multiplier 6
bfd app-route poll-interval 600000
bfd color biz-internet
 hello-interval 1000
 multiplier 7
 pmtu-discovery
!
bfd color lte
 hello-interval 10000
 multiplier 7
 no pmtu-discovery
!
```

Dans une exploitation réelle, ces paramètres sont généralement portés par un feature template BFD dans vManage.

### 9.3 Cohérence entre les deux extrémités

Les paramètres sont définis localement par couleur. Pour éviter les comportements asymétriques, harmonisez les valeurs sur les couleurs qui se font face. Le délai de détection effectif dépend des paquets reçus : un hello très espacé d'un côté retarde la détection de l'autre.

### 9.4 Interface de transport

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
!
```

---

## 10. Vérification et dépannage

### 10.1 Commandes principales (cEdge)

| Commande | Utilité |
|---|---|
| `show sdwan bfd summary` | Nombre de sessions up, down, flaps |
| `show sdwan bfd sessions` | Détail de chaque session BFD |
| `show sdwan bfd history` | Historique des transitions |
| `show sdwan app-route stats` | Mesures de perte, latence, gigue par tunnel |
| `show sdwan app-route sla-class` | Classes SLA et tunnels conformes |
| `show sdwan tunnel statistics` | Compteurs de tunnels IPsec |
| `show sdwan omp tlocs` | État BFD de chaque TLOC |
| `show sdwan policy app-route-policy-filter` | Politique AAR appliquée |

Sur vEdge : `show bfd summary`, `show bfd sessions`, `show app-route stats`.

### 10.2 Exemple de sortie : sessions BFD (illustratif)

```
Magasin-2750# show sdwan bfd sessions

                                SOURCE TLOC   REMOTE TLOC                    DST PUBLIC       DST PUBLIC
SYSTEM IP     SITE ID  STATE    COLOR         COLOR         SOURCE IP        IP               PORT   ENCAP  MULT  TX INT  UPTIME     TRANS
---------------------------------------------------------------------------------------------------------------------------------------
10.255.0.1    1000     up       biz-internet  biz-internet  203.0.113.10     198.51.100.20    12346  ipsec  7     1000    12:03:11   0
10.255.0.1    1000     up       lte           lte           100.64.12.5      198.51.100.30    12346  ipsec  7     10000   12:03:09   1
10.255.0.2    1001     up       biz-internet  biz-internet  203.0.113.10     198.51.100.40    12346  ipsec  7     1000    12:03:10   0
10.255.0.2    1001     up       lte           lte           100.64.12.5      198.51.100.50    12346  ipsec  7     10000   12:03:08   1
```

Lecture :

- Quatre sessions : deux couleurs vers deux datacenters.
- Intervalle de transmission 1000 ms sur biz-internet, 10000 ms sur lte (réglage allégé).
- La colonne `TRANS` compte les transitions (flaps) : une valeur qui augmente rapidement indique un lien instable.

### 10.3 Exemple de sortie : statistiques de qualité (illustratif)

```
Magasin-2750# show sdwan app-route stats

app-route statistics 203.0.113.10:12346 198.51.100.20:12346 ipsec
 remote-system-ip         10.255.0.1
 local-color              biz-internet
 remote-color             biz-internet
 mean-loss                0
 mean-latency             18
 mean-jitter              2

app-route statistics 203.0.113.10:12346 198.51.100.40:12346 ipsec
 remote-system-ip         10.255.0.2
 local-color              biz-internet
 remote-color             biz-internet
 mean-loss                0
 mean-latency             27
 mean-jitter              3
```

Lecture : latence 18 ms vers Paris, 27 ms vers Lyon, perte nulle, gigue très faible, le tunnel respecte la classe SLA-CAISSES.

### 10.4 Méthodologie de dépannage

1. Session BFD down : vérifier que le tunnel IPsec est bien monté (`show sdwan tunnel statistics`, `show sdwan ipsec outbound-connections`), que les TLOC distants sont connus (`show sdwan omp tlocs`).
2. Session qui ne monte pas : couleurs incompatibles, `restrict`, NAT symétrique, pare-feu bloquant UDP 12346 et la plage de port hopping, désynchronisation d'horloge si des certificats sont impliqués.
3. Session instable (flaps) : consulter `show sdwan bfd history`, vérifier la qualité du transport, envisager d'augmenter hello ou multiplicateur.
4. Mauvais choix de chemin applicatif : vérifier les statistiques de `app-route`, la classe SLA et les couleurs préférées.
5. MTU : un problème de fragmentation peut dégrader ou casser certains flux alors que BFD reste up. Vérifier `pmtu-discovery` et le MTU de tunnel.

### 10.5 Causes fréquentes d'incident

| Symptôme | Cause probable |
|---|---|
| BFD down alors que le lien physique est up | Pare-feu ou opérateur bloquant UDP, NAT problématique, TLOC distant down |
| Flaps répétés sur internet | Lien congestionné, timers trop agressifs |
| Trafic caisses sur la 4G sans raison | SLA trop strict, couleur préférée mal définie, mesures bruitées |
| Aucune bascule malgré dégradation | Poll interval long, SLA trop permissif, option de secours mal configurée |
| Sessions BFD élevées au hub | Politique de contrôle trop permissive (full mesh involontaire), absence de segmentation des hubs |
| Consommation 4G excessive | Hello à 1 s sur la couleur lte |

---

## 11. Dimensionnement et passage à l'échelle

### 11.1 Formule de base

Pour un hub, le nombre de sessions BFD est approximativement :

```
Sessions = (nombre de sites) x (nombre de paires de couleurs compatibles par site vers ce hub)
```

Pour un réseau en full mesh entre N sites ayant chacun C tunnels par paire de sites :

```
Sessions totales (approx.) = C x N x (N - 1) / 2
```

Exemple pédagogique : N = 1000 et C = 2 donnent environ 999 000 sessions, ce qui n'est pas réaliste sur des routeurs de magasin. Le hub and spoke ramène ce chiffre à N x C environ par hub.

### 11.2 Leviers de réduction

- Imposer la topologie par politique de contrôle OMP.
- Limiter le nombre de couleurs exposées (par exemple ne pas monter de tunnels sur la couleur lte vers tous les hubs).
- Utiliser l'option `restrict` pour empêcher les combinaisons de couleurs inutiles.
- Répartir les sites sur plusieurs hubs.
- Adapter les timers au transport.

### 11.3 Charge CPU et bande passante

Chaque session consomme un peu de CPU et de bande passante. Sur un petit routeur de magasin avec quelques sessions, l'impact est négligeable. Sur un hub avec plusieurs milliers de sessions, il faut :

- Vérifier les limites de la plateforme.
- Éviter les timers très agressifs (par exemple moins de 1 s) sauf besoin avéré.
- Superviser le CPU du plan de données.

---

## 12. BFD côté LAN (BGP, OSPF) : ne pas confondre

Les cEdge peuvent aussi utiliser BFD avec des voisins LAN (OSPF, BGP, EIGRP selon plateforme) pour accélérer la détection de panne d'un voisin sur le réseau de site. Ce BFD est :

- Configuré dans le VPN de service, sur l'interface LAN.
- Indépendant du BFD du plan de données SD-WAN.
- Un BFD standard (RFC 5880 et 5881), interopérable avec d'autres équipements.

Exemple : dans un hypermarché avec deux routeurs WAN Edge et des commutateurs de coeur de réseau, un BFD entre le WAN Edge et le routeur LAN permet de détecter rapidement la perte d'un voisin et de laisser OSPF ou BGP recalculer.

Résumé de la distinction :

| | BFD SD-WAN (overlay) | BFD LAN |
|---|---|---|
| Portée | Tunnels IPsec entre WAN Edge | Voisins de routage du site |
| Activation | Automatique, non désactivable | À configurer |
| Rôle | Liveness et mesure de qualité | Accélération de détection de voisin |
| Interopérabilité | Propre à la solution | Standard |

---

## 13. Bonnes pratiques

1. Garder les valeurs par défaut tant qu'un besoin mesuré ne justifie pas de les modifier.
2. Espacer les hello sur les transports cellulaires et à facturation au volume.
3. Aligner les paramètres des couleurs qui se font face.
4. Définir des classes SLA à partir de mesures réelles, avec une marge.
5. Éviter les politiques AAR strictes sur les flux critiques sans chemin de dernier recours.
6. Surveiller le nombre de transitions BFD : un pic signale un transport instable avant les plaintes des utilisateurs.
7. Limiter le nombre de tunnels par politique de contrôle OMP et par `restrict`.
8. Valider la découverte du MTU de chemin et le MTU de tunnel sur chaque type de lien.
9. Tester les scénarios de bascule en préproduction : coupure franche, dégradation progressive, perte du contrôleur.
10. Conserver un historique des mesures de qualité pour documenter les incidents opérateur.

---

## 14. Travaux pratiques

### TP 1 : Observer BFD

1. Dans un laboratoire à deux WAN Edge et des contrôleurs virtuels, vérifier `show bfd sessions`.
2. Relever le hello, le multiplicateur et l'uptime.
3. Interrompre le transport (shutdown de l'interface WAN) et mesurer le délai de détection.

### TP 2 : Régler les timers

1. Modifier le hello et le multiplicateur sur une couleur.
2. Mesurer de nouveau le délai de détection.
3. Introduire de la perte avec un émulateur de réseau (netem, par exemple) et observer l'apparition de flaps avec un réglage agressif.

### TP 3 : Mesure de qualité

1. Introduire une latence et une perte contrôlées sur un transport.
2. Consulter `show app-route stats` et comparer aux seuils d'une classe SLA.
3. Observer la bascule du trafic quand le SLA est violé.

### TP 4 : Politique AAR

1. Créer une classe SLA stricte pour un flux de test (VPN 10).
2. Définir une couleur préférée et un secours.
3. Dégrader la couleur préférée et vérifier le changement de chemin.
4. Rétablir la qualité et observer le retour (ou l'absence de retour) sur la couleur préférée.

### TP 5 : Lien avec OMP

1. Couper le tunnel d'une couleur.
2. Relever l'état BFD, le statut du TLOC dans `show omp tlocs` et le statut des routes dans `show omp routes`.
3. Expliquer chaque changement de statut (C,I,R vers U).

### TP 6 : Dimensionnement

1. Pour un réseau de 200 sites avec 2 couleurs chacun, calculer le nombre de sessions BFD en full mesh puis en hub and spoke avec 2 hubs.
2. Comparer et discuter les choix de conception.

---

## 15. Questionnaire d'auto-évaluation

**Q1. Combien de sessions BFD existe-t-il par tunnel IPsec entre deux WAN Edge ?**
Réponse : une session par tunnel, c'est-à-dire par paire de TLOC.

**Q2. Peut-on désactiver BFD sur les tunnels du plan de données ?**
Réponse : non, il est actif par défaut et ne peut pas être désactivé.

**Q3. Avec un hello de 1000 ms et un multiplicateur de 7, quel est le délai de détection ?**
Réponse : environ 7 secondes.

**Q4. À quoi sert le poll interval de app-route ?**
Réponse : il définit la fenêtre d'agrégation des mesures de perte, latence et gigue utilisées pour évaluer les classes SLA.

**Q5. Que devient une route OMP dont toutes les sessions BFD vers le TLOC sont down ?**
Réponse : le TLOC n'est plus résolu, la route n'est plus installée et le trafic utilise un autre chemin.

**Q6. Pourquoi espacer le hello sur une liaison 4G ?**
Réponse : pour réduire la consommation de données du forfait, le tunnel de secours n'ayant pas besoin d'une détection ultra rapide.

**Q7. Pourquoi BFD continue-t-il à fonctionner si les vSmart tombent ?**
Réponse : il appartient au plan de données, indépendant du plan de contrôle. Les tunnels déjà établis restent surveillés.

**Q8. En quoi une politique hub and spoke réduit-elle le nombre de sessions BFD ?**
Réponse : elle empêche les magasins de recevoir les TLOC des autres magasins, donc aucun tunnel ni session BFD magasin vers magasin n'est créé.

**Q9. Quelle différence entre BFD SD-WAN et BFD pour BGP ou OSPF sur le LAN ?**
Réponse : le premier surveille les tunnels de l'overlay et mesure la qualité, il est automatique et non standard. Le second est un BFD standard configuré sur le LAN pour accélérer la détection de voisins de routage.

**Q10. Quel risque présente un réglage trop agressif des timers BFD ?**
Réponse : faux positifs, flaps de tunnels, charge CPU accrue, instabilité du routage.

---

## 16. Glossaire

| Terme | Définition |
|---|---|
| AAR | Application-Aware Routing, choix du chemin selon la qualité mesurée |
| App-route poll interval | Fenêtre d'agrégation des mesures de qualité |
| BFD | Bidirectional Forwarding Detection |
| Color | Étiquette qualifiant un transport |
| DSCP | Marquage de priorité IP |
| Flap | Alternance répétée up et down d'une session |
| Gigue | Variation de la latence entre paquets |
| Hello interval | Intervalle d'envoi des paquets BFD |
| Multiplier | Nombre de hello manqués avant déclaration de panne |
| PMTU | Path MTU, plus grande taille de paquet sans fragmentation sur un chemin |
| SLA class | Seuils de perte, latence et gigue acceptables pour une application |
| TLOC | Transport Locator, point d'attache au transport |

---

## 17. Références

- RFC 5880 : Bidirectional Forwarding Detection (BFD).
- RFC 5881 : BFD for IPv4 and IPv6 (Single Hop).
- Documentation officielle Cisco SD-WAN : guides de configuration Policies et Systems and Interfaces (sections BFD et Application-Aware Routing).
- Cisco SD-WAN Design Guide (Cisco Validated Design).
- Cours associé : Cours complet SD-WAN et protocole OMP.

Les valeurs par défaut, les bornes de configuration, les limites de sessions et les syntaxes de commandes évoluent selon les versions et les plateformes : validez-les dans la documentation correspondant à votre version avant toute mise en production.
