# CP01. Routage avancé pour le SD-WAN : BGP, OSPF et redistribution

| | |
|---|---|
| **Partie** | Compléments (hors parcours principal) |
| **Durée indicative** | 1,5 semaine (12 h) |
| **Prérequis** | CCNA (OSPF, BGP de base) ; SW04, SW06 |
| **Niveau** | Avancé |
| **Lien examen** | ENARSI 300-410, ENCOR 350-401, ENSDWI 300-415 |
| **Lab associé** | Lab CP01 (maquette BGP/OSPF avec FRRouting et containerlab) |

> Module précédent : [SW18](../../04-partie-d-conception-migration-exploitation/SW18-indicateurs-sla/SW18-indicateurs-sla.md) | Module suivant : [CP02](../CP02-qos-wan/CP02-qos-wan.md)

> **Pourquoi ce complément.** Le SD-WAN ne supprime pas le routage : il en déplace la frontière. Chaque projet réel se heurte à BGP avec l'opérateur, OSPF dans le LAN et la redistribution avec OMP. Les erreurs de routage sont la première cause de pannes de migration (SW14). Ce module donne la maîtrise du routage dont un architecte SD-WAN a besoin.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Rappels : distance administrative et table de routage](#2-rappels--distance-administrative-et-table-de-routage)
3. [BGP en profondeur](#3-bgp-en-profondeur)
4. [Filtrage et manipulation : prefix-lists, route-maps, communautés](#4-filtrage-et-manipulation--prefix-lists-route-maps-communautés)
5. [Doubles opérateurs : entrée et sortie du trafic](#5-doubles-opérateurs--entrée-et-sortie-du-trafic)
6. [OSPF en profondeur](#6-ospf-en-profondeur)
7. [Redistribution sûre](#7-redistribution-sûre)
8. [BFD et convergence](#8-bfd-et-convergence)
9. [ECMP et répartition de charge](#9-ecmp-et-répartition-de-charge)
10. [Routage et SD-WAN : les points de jonction](#10-routage-et-sd-wan--les-points-de-jonction)
11. [Dépannage du routage](#11-dépannage-du-routage)
12. [Outil : simulateur de choix de route BGP](#12-outil--simulateur-de-choix-de-route-bgp)
13. [Lab guidé CP01](#13-lab-guidé-cp01)
14. [Approfondissements](#14-approfondissements)
15. [Quiz](#15-quiz)
16. [Aide-mémoire](#16-aide-mémoire)
17. [Pour aller plus loin](#17-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Prévoir la sélection de route BGP et la manipuler avec des attributs.
- Écrire des filtres propres (prefix-lists, route-maps, communautés).
- Concevoir l'entrée et la sortie de trafic avec deux opérateurs.
- Choisir les types de zones OSPF, résumer et réduire les LSA.
- Redistribuer sans boucle avec balises et filtres.
- Dépanner un problème de routage avec une méthode.

---

## 2. Rappels : distance administrative et table de routage

### 2.1 Distances administratives (Cisco IOS XE, valeurs par défaut)

| Source | Distance |
|---|---|
| Connecté | 0 |
| Statique | 1 |
| eBGP | 20 |
| EIGRP interne | 90 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP externe | 170 |
| iBGP | 200 |
| OMP (SD-WAN) | 251 |

### 2.2 Règle de choix dans la table de routage

1. **Préfixe le plus long** (longest match) l'emporte, quelle que soit la distance.
2. À préfixe identique : **distance administrative** la plus basse.
3. À distance égale : **métrique** du protocole.
4. À métrique égale : **partage de charge** (ECMP).

```text
10.0.0.0/8 via OSPF (110), 10.1.0.0/16 via OMP (251)  -> pour 10.1.5.5 : /16 gagne (longest match) malgré 251 > 110
```

C'est la source d'un piège classique : une route OMP plus spécifique l'emporte sur une route OSPF plus générale, quelle que soit sa distance.

---

## 3. BGP en profondeur

### 3.1 eBGP et iBGP

| Critère | eBGP | iBGP |
|---|---|---|
| Entre | AS différents | Même AS |
| Distance | 20 | 200 |
| TTL par défaut | 1 (voisins directement connectés) | 255 |
| Prochain saut | Modifié à l'émission | **Non modifié** par défaut |
| Règle d'annonce | — | **Split-horizon iBGP** : une route apprise en iBGP n'est pas ré-annoncée à un autre pair iBGP (d'où maillage complet, réflecteurs ou confédérations) |

### 3.2 Sélection de la meilleure route (ordre Cisco)

| # | Critère | Préférence |
|---|---|---|
| 1 | **Weight** (local au routeur Cisco) | Plus haut |
| 2 | **Local preference** | Plus haute |
| 3 | Route **originée localement** | Préférée |
| 4 | **AS-path** | Plus court |
| 5 | **Origin** | IGP < EGP < incomplet |
| 6 | **MED** | Plus bas (entre routes d'un même AS voisin) |
| 7 | eBGP plutôt qu'iBGP | eBGP |
| 8 | **Métrique IGP** vers le prochain saut | Plus basse |
| 9 | Route la plus ancienne (eBGP), puis router-id le plus bas, puis adresse de voisin | |

> Mémo : « **W**eight, **L**ocal-pref, **L**ocal, **A**S-path, **O**rigin, **M**ED, **e**BGP, **I**GP metric, **R**outer-id ».

### 3.3 Attributs : qui influence quoi

| Attribut | Portée | Direction d'influence |
|---|---|---|
| **Local preference** | AS entier (iBGP) | **Sortie** du trafic (quel opérateur utiliser pour sortir) |
| **AS-path prepend** | Voisins externes | **Entrée** du trafic (rendre un chemin moins attractif) |
| **MED** | Voisin direct | Suggère l'entrée à un AS voisin |
| **Communautés** | Au gré des politiques | Étiquetage, demande de politique à l'opérateur |
| **Weight** | Routeur local | Sortie, usage local |

### 3.4 Configuration BGP de base (IOS XE)

```text
router bgp 65001
 bgp router-id 10.255.0.1
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 203.0.113.1 remote-as 64500
 neighbor 203.0.113.1 description OPERATEUR-A
 neighbor 203.0.113.1 password 7 <CHIFFRE>
 neighbor 203.0.113.1 ttl-security hops 1
 !
 address-family ipv4
  network 198.51.100.0 mask 255.255.255.0
  neighbor 203.0.113.1 activate
  neighbor 203.0.113.1 prefix-list OUT-ANNONCES out
  neighbor 203.0.113.1 prefix-list IN-DEFAUT in
  neighbor 203.0.113.1 maximum-prefix 100 80
 exit-address-family
```

| Bonne pratique | Raison |
|---|---|
| `no bgp default ipv4-unicast` | Activer explicitement par famille d'adresses |
| **Authentification** (MD5/TCP-AO) | Éviter l'injection de sessions |
| `ttl-security` (GTSM) | Protéger des attaques lointaines |
| **maximum-prefix** | Se protéger d'une fuite de routes |
| **Filtres entrants et sortants** | Ne pas devenir un opérateur de transit par erreur |

### 3.5 Vérifications

```text
show ip bgp summary
show ip bgp neighbors 203.0.113.1 advertised-routes
show ip bgp neighbors 203.0.113.1 received-routes
show ip bgp 198.51.100.0/24
show ip bgp regexp ^64500_
show ip route bgp
```

---

## 4. Filtrage et manipulation : prefix-lists, route-maps, communautés

### 4.1 Prefix-lists

```text
! Autoriser exactement 198.51.100.0/24 et ses sous-réseaux jusqu'à /26
ip prefix-list OUT-ANNONCES seq 10 permit 198.51.100.0/24
! Accepter une route par défaut seule
ip prefix-list IN-DEFAUT seq 10 permit 0.0.0.0/0
! Rejeter les préfixes privés et trop spécifiques reçus (liste « bogons » raccourcie)
ip prefix-list REJET-BOGONS seq 10 deny 10.0.0.0/8 le 32
ip prefix-list REJET-BOGONS seq 20 deny 172.16.0.0/12 le 32
ip prefix-list REJET-BOGONS seq 30 deny 192.168.0.0/16 le 32
ip prefix-list REJET-BOGONS seq 40 deny 0.0.0.0/0 ge 25
ip prefix-list REJET-BOGONS seq 50 permit 0.0.0.0/0 le 24
```

Opérateurs `ge` et `le` : `10.0.0.0/8 le 24` correspond à tout préfixe de 10.0.0.0/8 de longueur 8 à 24.

### 4.2 Route-maps

```text
route-map VERS-OP-A permit 10
 match ip address prefix-list OUT-ANNONCES
 set as-path prepend 65001 65001
route-map VERS-OP-A deny 100
!
route-map DEPUIS-OP-A permit 10
 match ip address prefix-list IN-DEFAUT
 set local-preference 200
route-map DEPUIS-OP-A deny 100
```

Une route-map est évaluée **séquence par séquence** (numéro croissant) ; **le refus implicite** termine la liste : ce qui ne correspond à aucune séquence est refusé.

### 4.3 Communautés

| Communauté | Signification |
|---|---|
| `no-export` (65535:65281) | Ne pas annoncer hors de l'AS (ou de la confédération) |
| `no-advertise` (65535:65282) | Ne pas annoncer à aucun pair |
| `local-AS` (65535:65283) | Ne pas annoncer hors de l'AS local (sous-AS) |
| **Valeurs `AS:valeur`** | Définies localement ou par l'opérateur (ex. 64500:80 = local-pref 80) |

```text
ip community-list standard SDWAN permit 65001:777
route-map MARQUER permit 10
 set community 65001:777 additive
neighbor 203.0.113.1 send-community both
```

> Sans `send-community`, les communautés ne sont **pas** transmises au voisin.

### 4.4 Listes d'AS-path

```text
ip as-path access-list 10 permit ^$            ! uniquement les routes originées dans l'AS local
ip as-path access-list 20 permit ^64500_       ! routes du voisin 64500
```

---

## 5. Doubles opérateurs : entrée et sortie du trafic

### 5.1 Sortie du trafic (vers Internet) : local-preference

```text
route-map DEPUIS-OP-A permit 10
 set local-preference 200       ! opérateur A préféré pour la sortie
route-map DEPUIS-OP-B permit 10
 set local-preference 100
```

### 5.2 Entrée du trafic (depuis Internet) : prepend et communautés

```text
! Rendre le chemin via l'opérateur B moins attractif pour les autres
route-map VERS-OP-B permit 10
 match ip address prefix-list OUT-ANNONCES
 set as-path prepend 65001 65001 65001
```

Le prepend n'est qu'une **suggestion** : un AS distant peut préférer une autre route selon sa propre politique. Pour un contrôle plus fin, utilisez les **communautés de l'opérateur** (par exemple pour baisser la préférence locale chez lui).

### 5.3 Répartition par préfixe

Annoncer deux préfixes /25 d'un /24 : préfixe A préféré par l'opérateur 1 (prepend sur 2), préfixe B préféré par l'opérateur 2 (prepend sur 1), tous deux aussi annoncés en /24 (secours). Attention aux **filtres des opérateurs** (souvent au plus /24 en IPv4).

### 5.4 Sécurité de l'interconnexion

| Risque | Parade |
|---|---|
| Devenir un transit involontaire entre A et B | N'annoncer **que vos préfixes** (AS-path vide), filtre sortant sur `^$` |
| Détournement de vos préfixes | **RPKI** (ROA) ; surveillance des annonces |
| Fuite de routes reçues | `maximum-prefix`, filtres entrants |

---

## 6. OSPF en profondeur

### 6.1 Types de zones

| Type | LSA admis | Utilité |
|---|---|---|
| **Backbone (0)** | Tous | Cœur |
| **Standard** | 1, 2, 3, 4, 5 | Cas général |
| **Stub** | 1, 2, 3 (+ route par défaut) | Pas de routes externes |
| **Totally stub** (Cisco) | 1, 2 (+ défaut) | Agences : tables minimales |
| **NSSA** | 1, 2, 3, **7** | Permet une redistribution locale (un site avec un routeur externe) |
| **Totally NSSA** | 1, 2, 7 (+ défaut) | NSSA minimal |

### 6.2 Les LSA essentiels

| LSA | Nom | Portée |
|---|---|---|
| 1 | Routeur | Zone |
| 2 | Réseau (DR) | Zone |
| 3 | Résumé inter-zones | Entre zones |
| 4 | Résumé ASBR | Entre zones |
| 5 | Externe | Domaine entier (sauf stub) |
| 7 | NSSA externe | Zone NSSA (converti en 5 à la frontière) |

### 6.3 Coût et bande passante de référence

```text
Coût = bande passante de référence / bande passante de l'interface
Par défaut : 100 Mbit/s -> un lien 1 Gbit/s et un lien 100 Mbit/s ont le MÊME coût (1) !
```

```text
router ospf 10
 auto-cost reference-bandwidth 100000     ! 100 Gbit/s, à harmoniser sur TOUS les routeurs
```

### 6.4 Configuration d'un site (zone totally stub, résumés)

```text
router ospf 10 vrf 10
 router-id 192.168.10.1
 area 10 stub no-summary
 network 192.168.10.0 0.0.0.255 area 10
 passive-interface default
 no passive-interface GigabitEthernet3
!
! Sur l'ABR : résumé des réseaux de la zone vers le backbone
router ospf 10
 area 10 range 10.10.0.0 255.255.0.0
```

### 6.5 Vérifications

```text
show ip ospf neighbor
show ip ospf database
show ip ospf interface brief
show ip route ospf
show ip ospf border-routers
```

### 6.6 Problèmes classiques de voisinage

| Symptôme (état du voisin) | Causes probables |
|---|---|
| `INIT` | Hello reçus mais le voisin ne nous voit pas (ACL, MTU, unicast/multicast) |
| `2-WAY` bloqué | Normal entre deux DROther ; vérifier DR/BDR |
| `EXSTART/EXCHANGE` | **MTU différente** entre les deux extrémités |
| Aucun voisin | Timers hello/dead différents, zone ou type de réseau différent, authentification, sous-réseau différent |

---

## 7. Redistribution sûre

### 7.1 Règles d'or

1. **Un seul point de redistribution** dans une direction, ou **balises** systématiques.
2. **Filtrer** ce qui est redistribué (prefix-list ou balise).
3. Fixer **métriques et types** explicitement (OSPF `metric-type 1` ou 2).
4. **Résumer** avant de redistribuer quand c'est possible.
5. **Tester** les scénarios de panne.

### 7.2 Exemple : OSPF ↔ BGP avec balises

```text
route-map OSPF-TO-BGP deny 10
 match tag 777            ! route déjà passée par BGP : ne pas la reprendre
route-map OSPF-TO-BGP permit 20
 set community 65001:100
!
route-map BGP-TO-OSPF permit 10
 set tag 777
 set metric-type type-1
 set metric 100
!
router bgp 65001
 address-family ipv4
  redistribute ospf 10 route-map OSPF-TO-BGP
!
router ospf 10
 redistribute bgp 65001 subnets route-map BGP-TO-OSPF
```

### 7.3 Chemins sous-optimaux et boucles : les mécanismes

| Phénomène | Cause | Parade |
|---|---|---|
| **Boucle de redistribution** | Deux points redistribuent mutuellement sans balise | Balises, un seul point |
| **Routage sous-optimal** | Distance administrative qui préfère le mauvais protocole | Ajuster les distances par préfixe (`distance`) |
| **Flapping** | Redistribution d'une route instable | Résumer, lissage, BFD |
| **Perte d'informations de métrique** | Les métriques ne se traduisent pas | Fixer la métrique explicitement |

---

## 8. BFD et convergence

### 8.1 Détection rapide

BFD détecte la perte d'un voisin en quelques centaines de millisecondes à quelques secondes (selon les timers), bien plus vite que les temporisateurs de protocole (OSPF dead 40 s, BGP hold 180 s).

```text
interface GigabitEthernet1
 bfd interval 300 min_rx 300 multiplier 3         ! détection en 900 ms
!
router ospf 10
 bfd all-interfaces
!
router bgp 65001
 neighbor 203.0.113.1 fall-over bfd
```

### 8.2 Calcul de convergence

```text
Temps de convergence ≈ détection + propagation + calcul + installation (FIB)
BGP sans BFD : jusqu'à 180 s (hold) ; avec BFD 300 ms x 3 : ~1 s + reconvergence
```

Exemple : une panne du lien opérateur A, avec BFD à 900 ms et traitement BGP de 2 s, rétablit le trafic par B en environ **3 secondes**.

### 8.3 Précautions

BFD très agressif (50 ms) sur des liens instables (4G) provoque des **flaps** ; choisissez des timers cohérents avec la qualité du lien.

---

## 9. ECMP et répartition de charge

| Sujet | Détails |
|---|---|
| **ECMP** | Plusieurs chemins de même coût sont utilisés ; hachage par flux (5-uplets) |
| **BGP multipath** | `maximum-paths 4` (eBGP) ou `maximum-paths ibgp 4` ; les attributs jusqu'à l'IGP metric doivent être égaux |
| **OSPF ECMP** | `maximum-paths 4` |
| **Répartition inégale** | `bandwidth` de liens (DMZ link-bandwidth) pour une pondération proportionnelle au débit |
| **Désordonnancement** | Hachage par flux évite le désordre par flux |

```text
router bgp 65001
 address-family ipv4
  maximum-paths 2
```

---

## 10. Routage et SD-WAN : les points de jonction

| Jonction | Protocole côté service | Précautions |
|---|---|---|
| **WAN Edge ↔ LAN** | OSPF, BGP, EIGRP, statique | `advertise <protocole>` dans OMP, `redistribute omp` dans le protocole, balises |
| **WAN Edge ↔ opérateur MPLS** | eBGP (ou OSPF, statique) | Filtres stricts, communautés, coexistence (SW14) |
| **WAN Edge ↔ DC** | BGP/OSPF | Résumés, ECMP, pare-feu |
| **Hub ↔ cloud** | BGP | Limites de préfixes, nombre de routes |
| **Underlay (VPN 0)** | Statique, DHCP, parfois BGP avec l'opérateur | Séparer du routage de service |

> L'underlay (transport) et les VRF de service sont **deux mondes de routage séparés**. Une route statique du VPN 0 n'a rien à voir avec les routes des VPN de service. Cette confusion est la cause de la moitié des questions des stagiaires.

---

## 11. Dépannage du routage

### 11.1 Méthode en six étapes

| # | Question | Commande |
|---|---|---|
| 1 | Le voisin est-il établi ? | `show ip bgp summary`, `show ip ospf neighbor` |
| 2 | La route est-elle reçue ? | `show ip bgp neighbors X received-routes` (nécessite `soft-reconfiguration inbound` ou `show ip bgp neighbors X routes`) |
| 3 | Est-elle filtrée ? | Prefix-list/route-map entrants ; `debug ip bgp updates` (avec prudence) |
| 4 | Est-elle la meilleure ? | `show ip bgp <préfixe>` : attributs et raison |
| 5 | Est-elle installée dans la RIB ? | `show ip route <préfixe>` (distance, concurrente) |
| 6 | Est-elle annoncée aux autres ? | `show ip bgp neighbors Y advertised-routes` |

### 11.2 Symptômes et causes

| Symptôme | Cause fréquente |
|---|---|
| Session BGP `Idle` / `Active` | Pas de route vers le voisin, ACL, mauvais AS, mot de passe, TTL |
| Route BGP reçue mais non installée | Prochain saut inaccessible, route plus spécifique ou distance moindre d'un autre protocole |
| Route annoncée non vue chez le voisin | Filtre sortant, communauté `no-export`, `send-community` manquant |
| OSPF bloqué en `EXSTART` | MTU différente |
| Chemin asymétrique | Local-pref ou prepend incohérents d'un site à l'autre |
| Boucle | Redistribution sans balise |

---

## 12. Outil : simulateur de choix de route BGP

```python
#!/usr/bin/env python3
"""bgp_best_path.py : applique l'ordre de sélection BGP (Cisco, simplifié) à un ensemble de routes candidates.
Utile pour s'entraîner et pour expliquer un choix à un collègue."""
from dataclasses import dataclass, field

ORIGINE = {"IGP": 0, "EGP": 1, "incomplete": 2}

@dataclass
class Route:
    nom: str
    weight: int = 0
    local_pref: int = 100
    locale: bool = False
    as_path: list = field(default_factory=list)
    origin: str = "IGP"
    med: int = 0
    ebgp: bool = True
    igp_metric: int = 0
    router_id: str = "0.0.0.0"

def cle(r: Route):
    # tuple de comparaison : plus petit = meilleur
    rid = tuple(int(x) for x in r.router_id.split("."))
    return (-r.weight, -r.local_pref, 0 if r.locale else 1, len(r.as_path), ORIGINE[r.origin],
            r.med, 0 if r.ebgp else 1, r.igp_metric, rid)

def meilleure(routes):
    classement = sorted(routes, key=cle)
    gagnante = classement[0]
    # explication : premier critère qui départage la première et la deuxième
    raisons = ["weight", "local-preference", "route locale", "longueur AS-path", "origin", "MED",
               "eBGP>iBGP", "métrique IGP", "router-id"]
    if len(classement) > 1:
        a, b = cle(classement[0]), cle(classement[1])
        for i, (x, y) in enumerate(zip(a, b)):
            if x != y:
                return gagnante, raisons[i]
    return gagnante, "aucun (égalité)"

if __name__ == "__main__":
    routes = [
        Route("via Opérateur A", local_pref=200, as_path=[64500, 64999], router_id="203.0.113.1"),
        Route("via Opérateur B", local_pref=100, as_path=[64501], router_id="198.51.100.1"),
        Route("via iBGP hub", local_pref=100, as_path=[64501], ebgp=False, igp_metric=20, router_id="10.255.0.2"),
    ]
    g, raison = meilleure(routes)
    print(f"Meilleure route : {g.nom}   (départage par : {raison})")
```

Exercice : modifiez les attributs pour faire gagner chaque route à son tour, et expliquez pourquoi.

---

## 13. Lab guidé CP01

### 13.1 Objectif

Construire une maquette avec FRRouting : un site avec deux opérateurs en eBGP, un cœur OSPF, redistribution avec balises, BFD, et vérifier les comportements (entrée/sortie, bascule, boucle évitée).

### 13.2 Topologie

```text
          AS 64500 (OP-A)                AS 64501 (OP-B)
              r-opa                         r-opb
                |  203.0.113.0/30              |  198.51.100.0/30
                +-------------+  +-------------+
                             [ edge ]  AS 65001  (BGP eBGP vers A et B)
                                 | 10.0.0.0/30  (OSPF area 0)
                              [ core ]  (OSPF) -- LAN 192.168.10.0/24
```

### 13.3 Déploiement avec containerlab

```yaml
# lab-cp01.clab.yml
name: cp01
topology:
  kinds:
    linux:
      image: quay.io/frrouting/frr:10.0.1
  nodes:
    r-opa:
      kind: linux
      binds: [opa/daemons:/etc/frr/daemons, opa/frr.conf:/etc/frr/frr.conf]
    r-opb:
      kind: linux
      binds: [opb/daemons:/etc/frr/daemons, opb/frr.conf:/etc/frr/frr.conf]
    edge:
      kind: linux
      binds: [edge/daemons:/etc/frr/daemons, edge/frr.conf:/etc/frr/frr.conf]
    core:
      kind: linux
      binds: [core/daemons:/etc/frr/daemons, core/frr.conf:/etc/frr/frr.conf]
  links:
    - endpoints: ["edge:eth1", "r-opa:eth1"]
    - endpoints: ["edge:eth2", "r-opb:eth1"]
    - endpoints: ["edge:eth3", "core:eth1"]
```

Fichier `daemons` commun (extrait) :

```text
bgpd=yes
ospfd=yes
bfdd=yes
zebra=yes
vtysh_enable=yes
zebra_options="  -A 127.0.0.1 -s 90000000"
bgpd_options="   -A 127.0.0.1"
ospfd_options="  -A 127.0.0.1"
bfdd_options="   -A 127.0.0.1"
```

Configuration de `edge/frr.conf` (extrait) :

```text
frr defaults traditional
hostname edge
!
interface eth1
 ip address 203.0.113.2/30
interface eth2
 ip address 198.51.100.2/30
interface eth3
 ip address 10.0.0.1/30
!
router bgp 65001
 bgp router-id 10.255.0.1
 no bgp ebgp-requires-policy
 neighbor 203.0.113.1 remote-as 64500
 neighbor 198.51.100.1 remote-as 64501
 neighbor 203.0.113.1 bfd
 neighbor 198.51.100.1 bfd
 !
 address-family ipv4 unicast
  network 192.168.10.0/24
  neighbor 203.0.113.1 route-map DEPUIS-A in
  neighbor 198.51.100.1 route-map DEPUIS-B in
  neighbor 198.51.100.1 route-map VERS-B out
 exit-address-family
!
route-map DEPUIS-A permit 10
 set local-preference 200
route-map DEPUIS-B permit 10
 set local-preference 100
route-map VERS-B permit 10
 set as-path prepend 65001 65001
!
router ospf
 ospf router-id 10.255.0.1
 network 10.0.0.0/30 area 0
 redistribute bgp route-map BGP-TO-OSPF
!
route-map BGP-TO-OSPF permit 10
 set metric 100
 set metric-type 1
```

### 13.4 Étapes

| # | Action | Vérification |
|---|---|---|
| 1 | Déployer : `sudo containerlab deploy -t lab-cp01.clab.yml` | `containerlab inspect` |
| 2 | Établir BGP avec A et B ; OSPF avec le cœur | `vtysh -c 'show ip bgp summary'` |
| 3 | OP-A et OP-B annoncent une route par défaut ; vérifier la préférence | `show ip bgp 0.0.0.0/0` : A (local-pref 200) |
| 4 | Vérifier la sortie du trafic depuis le LAN | `traceroute` via A |
| 5 | Couper le lien vers A (`ip link set eth1 down` dans le conteneur) | Bascule vers B ; mesurer le temps avec BFD |
| 6 | Rétablir A ; observer le retour | Préférence restaurée |
| 7 | Vérifier l'entrée : sur OP-B, observer le prepend de `192.168.10.0/24` | `show ip bgp 192.168.10.0/24` |
| 8 | Redistribuer des routes du cœur dans BGP **sans balise**, créer une boucle ; la détecter ; corriger avec balise | Captures avant/après |
| 9 | Introduire une **MTU différente** sur une extrémité OSPF, observer `EXSTART`, corriger | `show ip ospf neighbor` |
| 10 | Rédiger le rapport avec temps de convergence mesurés | Document |

### 13.5 Grille

| Critère | Points |
|---|---|
| Maquette fonctionnelle (BGP, OSPF, BFD) | 5 |
| Politique de sortie/entrée démontrée | 4 |
| Bascule chronométrée et analysée | 4 |
| Boucle de redistribution provoquée et corrigée | 4 |
| Dépannage de la MTU OSPF | 2 |
| Rapport | 1 |

---

## 14. Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **AS / ASN** | Système autonome / son numéro |
| **eBGP / iBGP** | BGP entre / à l'intérieur d'AS |
| **Local-preference** | Attribut BGP déterminant la sortie préférée dans un AS |
| **MED** | Indication à un AS voisin pour l'entrée préférée |
| **AS-path prepend** | Allongement artificiel de l'AS-path |
| **Communauté** | Étiquette attachée à une route |
| **Route-map / prefix-list** | Outils de filtrage et de modification de routes |
| **RIB / FIB** | Table de routage / table de transfert |
| **ABR / ASBR** | Routeur de bordure de zone / de système autonome (OSPF) |
| **LSA** | Annonce d'état de lien OSPF |
| **DR / BDR** | Routeur désigné / de secours (OSPF) |
| **BFD** | Détection de défaillance bidirectionnelle |
| **ECMP** | Chemins multiples de coût égal |
| **RPKI / ROA** | Infrastructure de validation d'origine des préfixes |
| **GTSM** | Sécurité par TTL |
| **Route reflector** | Réflecteur de routes iBGP |

### B. Cas d'usage détaillés

#### Cas 1. Entreprise multi-homée : 70 % du trafic sur un seul opérateur

Deux opérateurs, mais tout le trafic entrant passe par A. Cause : l'opérateur B reçoit les mêmes préfixes mais A est favori chez plusieurs transitaires. Solution : annoncer deux /25 avec prepend croisé et utiliser les **communautés de l'opérateur B** pour relever la préférence locale. Résultat : 55/45. **Leçon** : le trafic entrant se **négocie** ; on le suggère, on ne le décide pas.

#### Cas 2. L'incident du transit involontaire

Faute de filtre sortant, un site annonce à B les routes reçues de A et devient un transit : saturation et facture. **Parade** : filtre `^$` sur l'AS-path sortant, `maximum-prefix`. **Leçon** : « n'annoncez que ce qui est à vous » est la première règle de BGP.

#### Cas 3. OSPF : coût identique sur 100 Mbit/s et 10 Gbit/s

Dans un réseau d'agences, tous les liens ont le coût 1, donc OSPF choisit parfois le lien lent. **Parade** : `auto-cost reference-bandwidth` harmonisé sur tous les routeurs. **Leçon** : un paramètre oublié coûte des mois de plaintes.

#### Cas 4. Route OMP plus spécifique que la route OSPF

Après migration, un serveur est joint par l'overlay alors qu'il se trouve dans le LAN local : une route OMP en /24 a été annoncée par un autre site, plus spécifique que la route OSPF /16 locale. **Leçon** : **le longest match prime sur la distance** ; maîtrisez les résumés et les annonces dans OMP.

#### Cas 5. BFD trop agressif sur la 4G

BFD à 100 ms sur une liaison 4G provoque des flaps toutes les 10 minutes. Solution : 1 s × 5 sur ce lien, 300 ms × 3 sur la fibre. **Leçon** : **adaptez les timers à la qualité du support**.

#### Cas 6. L'iBGP qui ne propage pas

Un hub reçoit une route par iBGP d'un voisin et ne la ré-annonce pas à un autre voisin iBGP. Cause : split-horizon iBGP. Solution : réflecteur de routes ou maillage complet. **Leçon** : sachez **où se trouvent vos réflecteurs**.

### C. Retours d'expérience

1. **Filtrez toujours, des deux côtés.** La plupart des incidents BGP publics viennent de filtres absents.
2. **Documentez la politique de routage** : tableau « qui annonce quoi à qui, avec quels attributs ».
3. **Préférez la simplicité** : peu d'attributs, des conventions de communautés claires, des route-maps lisibles.
4. **Testez les pannes**, pas seulement le nominal.
5. **Commentez vos route-maps** (descriptions) : dans deux ans, personne ne se souviendra des raisons.
6. **Surveillez les sessions et le nombre de préfixes** : une chute ou une explosion est un signal.
7. **Évitez la redistribution mutuelle** à plus d'un point, sinon balisez.
8. **Gardez un lab** avec la même logique de routage que la production.

### D. Exercices supplémentaires

1. Dans le simulateur, créez trois routes où le critère décisif est tour à tour : local-preference, longueur d'AS-path, MED.
2. Écrivez la route-map qui fait préférer l'opérateur B pour la sortie vers 8.8.8.0/24 et l'opérateur A pour le reste.
3. Calculez le coût OSPF d'un lien 10 Gbit/s avec bande passante de référence 100 Mbit/s puis 100 Gbit/s.
4. Rédigez les filtres d'entrée et de sortie d'un CE connecté à un opérateur (annonces autorisées, réponses acceptées).
5. Un site reçoit la route par défaut de A (local-pref 200) et de B (100). A tombe : combien de temps pour la bascule avec BFD 300 ms × 3 et sans BFD (hold 90 s) ?

<details>
<summary>Corrigés</summary>

3. Référence 100 Mbit/s : `100 / 10 000 = 0,01` → arrondi à **1** ; référence 100 Gbit/s : `100 000 / 10 000 = 10`.
5. Avec BFD : environ **0,9 s** de détection + traitement ; sans BFD : jusqu'à **90 s** (hold timer) si le lien reste « up » physiquement.
2. `ip prefix-list GOOGLE-DNS permit 8.8.8.0/24` ; `route-map DEPUIS-B permit 10` + `match ip address prefix-list GOOGLE-DNS` + `set local-preference 300` ; `route-map DEPUIS-B permit 20` (sans modification, local-pref 100) ; A garde 200 pour le reste.

</details>

### E. Mini-projet (8 heures)

Documentez et validez en maquette la **politique de routage complète** d'un site d'entreprise multi-homé relié à un LAN OSPF et à un SD-WAN (simulé) : plan d'AS, prefix-lists, route-maps, communautés, BFD, résumés, redistribution avec balises, plan de test (10 scénarios) et résultats chronométrés.

---

## 15. Quiz

1. Quel attribut BGP influence la **sortie** du trafic dans un AS ?
2. Pourquoi le prepend n'est-il pas un contrôle absolu de l'entrée du trafic ?
3. Dans quelle condition OSPF reste-t-il bloqué à `EXSTART` ?
4. Quel est le piège de la bande passante de référence par défaut d'OSPF ?
5. Quelle règle gagne : le préfixe le plus long ou la distance administrative la plus basse ?
6. Pourquoi une communauté n'arrive-t-elle pas chez le voisin ?
7. À quoi sert `maximum-prefix` ?

<details>
<summary>Réponses</summary>

1. La local-preference. 2. Les AS distants appliquent leurs propres politiques. 3. Une MTU différente des deux côtés. 4. Les liens à 100 Mbit/s et plus ont tous le coût 1. 5. Le préfixe le plus long. 6. Parce que `send-community` n'est pas configuré. 7. À se protéger d'une fuite massive de routes en coupant la session au-delà d'un seuil.

</details>

---

## 16. Aide-mémoire

```text
Table de routage : longest match > distance administrative > métrique ; AD : conn 0, stat 1, eBGP 20, OSPF 110, iBGP 200, OMP 251
BGP best path : Weight > Local-pref > Local > AS-path > Origin > MED > eBGP > IGP metric > router-id
Sortie : local-pref | Entrée : prepend, communautés opérateur, /25 | Filtrer : prefix-list + route-map, maximum-prefix, AS-path ^$
OSPF : cost = ref-bw / bw (harmoniser auto-cost) ; stub/totally stub pour agences ; MTU = EXSTART ; area range pour résumer
Redistribution : balises, un point, filtres, métriques explicites ; BFD : 300 ms x 3 fibre, 1 s x 5 sur 4G
```

## 17. Pour aller plus loin

- RFC 4271 (BGP-4), RFC 1997 (communautés), RFC 2328 (OSPFv2), RFC 5880 (BFD) : <https://www.rfc-editor.org>
- FRRouting : <https://docs.frrouting.org>
- containerlab : <https://containerlab.dev>
- Livre : « Internet Routing Architectures » (Halabi, Cisco Press) et « Routing TCP/IP » (Doyle).

> Module précédent : [SW18](../../04-partie-d-conception-migration-exploitation/SW18-indicateurs-sla/SW18-indicateurs-sla.md) | Module suivant : [CP02](../CP02-qos-wan/CP02-qos-wan.md)
