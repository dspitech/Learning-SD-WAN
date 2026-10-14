# SW03. Fonctions clés du SD-WAN

| | |
|---|---|
| **Partie** | A. Fondamentaux du SD-WAN |
| **Durée indicative** | 1 semaine (8 h) |
| **Prérequis** | SW01, SW02 ; notions de QoS (marquage DSCP, files d'attente) |
| **Niveau** | Intermédiaire |
| **Lien examen** | ENCOR, ENSDWI (routage applicatif, QoS, segmentation) |
| **Labs associés** | T1 (suite), T5 (voix avec classes de SLA) |

> Module précédent : [SW02](../SW02-transports-overlay-underlay/SW02-transports-overlay-underlay.md) | Module suivant : [SW04](../../02-partie-b-catalyst-sd-wan/SW04-architecture-composants/SW04-architecture-composants.md)

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Routage selon l'application (AAR)](#2-routage-selon-lapplication-aar)
3. [Sélection dynamique de chemin et répartition de charge](#3-sélection-dynamique-de-chemin-et-répartition-de-charge)
4. [Techniques de fiabilisation](#4-techniques-de-fiabilisation)
5. [Segmentation](#5-segmentation)
6. [QoS](#6-qos)
7. [Déploiement sans intervention (ZTP)](#7-déploiement-sans-intervention-ztp)
8. [Politiques centralisées](#8-politiques-centralisées)
9. [Analytique et visibilité](#9-analytique-et-visibilité)
10. [Accès direct à Internet (DIA)](#10-accès-direct-à-internet-dia)
11. [Optimisation WAN](#11-optimisation-wan)
12. [Lab : scénarios et simulations](#12-lab--scénarios-et-simulations)
13. [Exercices](#13-exercices)
14. [Quiz](#14-quiz)
15. [Aide-mémoire](#15-aide-mémoire)
16. [Pour aller plus loin](#16-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Décrire chaque fonction clé du SD-WAN et le problème qu'elle résout.
- Associer une fonction à un besoin métier (voix, SaaS, panne, coût).
- Définir une classe de SLA et simuler un choix de chemin.
- Chiffrer l'effet de la FEC et de la duplication de paquets.
- Appliquer correctement les notions de segmentation, QoS et DIA.

---

## 2. Routage selon l'application (AAR)

Le routage selon l'application (**Application-Aware Routing**, AAR) choisit, pour chaque flux, un chemin qui respecte les exigences de l'application, d'après des **mesures continues** sur les chemins.

### 2.1 Les trois ingrédients

| Ingrédient | Rôle | Exemple |
|---|---|---|
| **Classification** | Identifier l'application | Inspection de paquets (DPI), signatures, DSCP, ports |
| **Classe de SLA** | Définir « assez bon » | Voix : perte ≤ 1 %, délai ≤ 150 ms, gigue ≤ 30 ms |
| **Politique de chemin** | Choisir les transports | Voix : préférer MPLS, sinon Internet s'il respecte le SLA |

### 2.2 Algorithme simplifié

```text
Pour chaque flux classé dans la classe VOIX :
  1. Lister les chemins (tunnels) disponibles vers la destination.
  2. Ne garder que ceux dont les mesures respectent la classe de SLA VOIX.
  3. Parmi eux, choisir selon la préférence (couleur préférée, répartition…).
  4. Si aucun chemin ne respecte le SLA :
       - soit utiliser le meilleur chemin disponible (repli, comportement par défaut selon les éditeurs),
       - soit refuser (mode strict) selon la politique.
  5. Réévaluer à chaque nouvelle mesure.
```

### 2.3 Mesure de la qualité : fenêtres et réactivité

Les mesures de perte, délai et gigue sont calculées sur des fenêtres glissantes. Exemple (Cisco Catalyst SD-WAN, valeurs par défaut courantes à vérifier selon la version) :

| Paramètre | Valeur par défaut | Rôle |
|---|---|---|
| Intervalle BFD | 1 s | Fréquence des sondes |
| Intervalle de calcul (poll interval) | 600 s (10 min) | Longueur d'une période de mesure |
| Multiplicateur AAR | 6 | Nombre de périodes moyennées |

**Conséquence** : avec les valeurs par défaut, la moyenne porte sur 6 × 10 min ; une dégradation brève ne déclenche pas de bascule. On **raccourcit** l'intervalle pour la voix (réactivité accrue) en acceptant davantage de charge de contrôle et le risque d'oscillations. Exemple : intervalle 30 s × multiplicateur 6 = moyenne sur 3 minutes.

### 2.4 Exemple de décision

| Chemin | Perte | Délai | Gigue | Conforme VOIX ? |
|---|---|---|---|---|
| MPLS | 0,1 % | 30 ms | 3 ms | Oui |
| Internet fibre | 0,4 % | 25 ms | 8 ms | Oui |
| 5G | 1,8 % | 55 ms | 25 ms | Non (perte) |

Avec la préférence « MPLS » : le flux voix prend MPLS. Si MPLS dépasse 150 ms, il bascule sur Internet fibre, sans toucher la 5G.

---

## 3. Sélection dynamique de chemin et répartition de charge

### 3.1 Politiques de sélection

| Stratégie | Principe | Cas d'usage |
|---|---|---|
| **Manuelle** (ordre de préférence fixe) | Premier transport disponible | Simple et prévisible |
| **Meilleure qualité** | Chemin le plus performant selon une métrique | Applications sensibles |
| **Coût minimal respectant le SLA** | Chemin le moins cher parmi les conformes | Optimisation des coûts |
| **Répartition de charge** | Partage entre transports | Maximiser le débit utile |

### 3.2 Répartition de charge : par flux ou par paquet

| Mode | Fonctionnement | Avantage | Inconvénient |
|---|---|---|---|
| **Par flux** (hachage 5-uplet) | Un flux reste sur un chemin | Pas de désordonnancement | Un gros flux ne dépasse pas un lien |
| **Par paquet** | Chaque paquet alterne | Un flux peut agréger plusieurs liens | Réordonnancement, pénalise TCP et voix |

> Règle générale : **par flux** par défaut ; **par paquet** seulement avec un mécanisme de réordonnancement (agrégation de liens, tunnel bonding des solutions qui en offrent).

### 3.3 Bascule

Temps de bascule ≈ temps de détection + temps de reconvergence de la politique :

| Événement | Mécanisme | Ordre de grandeur |
|---|---|---|
| Coupure franche d'un lien | BFD (détection) | 1 à 7 s |
| Dégradation (perte, délai) | AAR (poll interval) | secondes à minutes selon réglage |
| Panne de l'accès (DHCP, câble) | Surveillance de l'interface | quelques secondes |

---

## 4. Techniques de fiabilisation

### 4.1 FEC (Forward Error Correction)

On ajoute des paquets de **redondance** pour reconstruire des paquets perdus sans retransmission.

Principe avec un code de parité simple : pour k paquets de données, on envoie 1 paquet de parité (XOR). La perte d'**un seul** des k+1 paquets est récupérable.

```text
D1 D2 D3 D4  | P = D1 ^ D2 ^ D3 ^ D4
Si D3 est perdu : D3 = P ^ D1 ^ D2 ^ D4
Surcoût en débit : 1/k  (k = 4 → 25 %)
```

Démonstration en Python :

```python
#!/usr/bin/env python3
"""Démonstration de FEC par parité XOR sur 4 paquets."""
import os
from functools import reduce

def xor_bytes(a: bytes, b: bytes) -> bytes:
    return bytes(x ^ y for x, y in zip(a, b))

paquets = [os.urandom(8) for _ in range(4)]            # D1..D4
parite = reduce(xor_bytes, paquets)                    # P

perdu = 2                                              # D3 est perdu
recus = [p for i, p in enumerate(paquets) if i != perdu] + [parite]
reconstruit = reduce(xor_bytes, recus)

print("D3 original   :", paquets[perdu].hex())
print("D3 reconstruit:", reconstruit.hex())
assert reconstruit == paquets[perdu]
print("Reconstruction réussie, surcoût =", f"{1/len(paquets):.0%}")
```

Effet sur la perte résiduelle (perte aléatoire indépendante p, groupe de k+1 paquets, 1 parité) : un groupe est **irrécupérable** si 2 paquets ou plus sont perdus.

Exemple : p = 2 %, k = 4 (5 paquets) :
`P(≥2 pertes) = 1 − (0,98⁵ + 5×0,02×0,98⁴) ≈ 1 − (0,9039 + 0,0922) = 0,0039` → **0,39 %** de groupes touchés (et seulement les paquets de ces groupes sont perdus) au lieu de 2 %.

### 4.2 Duplication de paquets

On envoie **chaque paquet sur deux chemins** ; le récepteur garde le premier arrivé et supprime le doublon.

| Critère | FEC | Duplication |
|---|---|---|
| Surcoût de débit | 10 à 50 % (réglable) | 100 % |
| Efficacité contre les pertes | Bonne | Excellente (tant que les deux chemins ne perdent pas le même paquet) |
| Délai ajouté | Petit (attente de la parité) | Nul (premier arrivé) |
| Usage conseillé | Voix et vidéo sur un seul lien dégradé | Flux critiques à faible débit (voix, contrôle) |

Perte résiduelle avec duplication sur deux chemins indépendants de pertes p₁ et p₂ : `p₁ × p₂`. Exemple : 2 % et 1 % → **0,02 %**.

> Ces fonctions dépendent de l'éditeur et de la plate-forme (disponibles notamment chez Fortinet, VeloCloud, Aruba EdgeConnect ; à vérifier pour Cisco Catalyst SD-WAN selon la version, qui propose notamment la duplication de paquets pour des flux sélectionnés).

### 4.3 Optimisation TCP

Sur liens à fort délai (satellite, intercontinental) : ajustement de fenêtre, accusés de réception locaux (proxy TCP), contrôle de congestion adapté. Fonction de plate-forme selon les éditeurs.

---

## 5. Segmentation

La segmentation sépare les flux en domaines de routage **de bout en bout** (VPN, VRF) : utilisateurs, invités, voix, IoT, serveurs, paiement.

```text
Site A                                          Site B
 VPN 10 (Corp)  ======= tunnel overlay =======  VPN 10 (Corp)
 VPN 20 (Invités) ===== (même tunnel) ========  VPN 20 (Invités)
 VPN 30 (IoT)   ======                           VPN 30 (IoT)
Les VPN restent isolés : aucune route ni trafic ne passe de l'un à l'autre
sauf politique explicite (fuite de routes, passage par un pare-feu).
```

Intérêts : sécurité (réduction de la surface d'attaque), conformité (PCI DSS), simplification (une seule infrastructure, plusieurs réseaux logiques).

---

## 6. QoS

La QoS priorise les flux **à la sortie du WAN** (le goulot est le lien, pas le cœur de l'overlay).

| Étape | Rôle |
|---|---|
| **Classification** | Identifier les flux (DSCP, application) |
| **Marquage** | Poser un DSCP cohérent (EF pour la voix, AF41 pour la vidéo…) |
| **Files d'attente** | Voix : file à priorité stricte (LLQ) ; autres : bande passante garantie |
| **Lissage** (shaping) | Limiter le débit au contrat du lien pour **éviter que la file se forme chez l'opérateur** |
| **Abandon** (WRED/tail drop) | Gérer la congestion |

Exemple de plan de classes :

| Classe | DSCP | Traitement |
|---|---|---|
| Voix | EF (46) | Priorité stricte, 15 à 20 % |
| Vidéo interactive | AF41 (34) | Bande passante garantie, 20 à 25 % |
| Applications métier | AF31 (26) | Bande passante garantie, 20 à 30 % |
| Contrôle réseau (BFD, routage) | CS6 (48) | Garantie minimale |
| Par défaut | 0 | Le reste |
| Arrière-plan | CS1 (8) | Minimum |

Le détail (MQC, hiérarchique, calculs) est dans [CP02](../../complements/CP02-qos-wan/CP02-qos-wan.md) et SW07.

---

## 7. Déploiement sans intervention (ZTP)

**ZTP** (Zero Touch Provisioning) : l'équipement est expédié sur le site, branché (alimentation, réseau). Sans technicien réseau :

```text
1. Démarrage -> DHCP : obtention IP, passerelle, DNS
2. Contact d'un service de redirection (cloud de l'éditeur) avec son numéro de série
3. Redirection vers l'orchestrateur du client
4. Authentification mutuelle (certificats)
5. Connexion aux contrôleurs, téléchargement de la configuration
6. Établissement des tunnels et mise en production
```

| Avantage | Précision |
|---|---|
| Déploiement en heures et non en semaines | Si le lien Internet est livré |
| Moins d'erreurs | Configuration issue d'un modèle |
| Pas de technicien spécialisé sur place | Un personnel local branche les câbles |

Prérequis : le site doit disposer d'un accès Internet DHCP et d'un équipement **préalablement déclaré** dans la plateforme du client.

---

## 8. Politiques centralisées

Une **intention** (« la voix passe par le transport le plus performant ; les invités sortent en direct ») est décrite **une fois** et appliquée à des **groupes de sites**.

```text
Intention : "Pour tous les sites régionaux, la visioconférence doit rester sous 200 ms et 2 % de perte"
   -> classe de SLA + politique de routage applicatif
   -> appliquée sur la liste de sites REGIONS
   -> les équipements reçoivent la politique et l'exécutent localement
```

C'est le principe directeur : **politique centralisée, exécution distribuée**.

---

## 9. Analytique et visibilité

| Visibilité | Exemples |
|---|---|
| Applications | Top applications par volume, par site |
| Chemins | Qualité par tunnel (perte, délai, gigue), historique |
| Disponibilité | État des transports, nombre de bascules |
| Expérience | Score de qualité par application SaaS |
| Capacité | Utilisation des liens, saturation |

L'analytique permet de **justifier** les décisions (changer d'opérateur, augmenter un débit) avec des données. Voir SW10 et SW18.

---

## 10. Accès direct à Internet (DIA)

| Modèle de sortie | Description | Avantage | Inconvénient |
|---|---|---|---|
| **Centralisée** | Tout passe par le siège | Contrôle total, une seule pile de sécurité | Détour, goulot, coût MPLS |
| **Locale (DIA)** | Chaque site sort en direct | Performance, simplicité | Sécurité à déployer partout |
| **Régionale** | Via un hub régional | Compromis | Conception plus fine |
| **Cloud** | Vers un service de sécurité cloud (SIG, SSE) | Sécurité centralisée sans détour | Dépend du fournisseur |

L'accès direct n'est acceptable qu'avec une **sécurité adaptée** (pare-feu local, filtrage DNS/URL, ou redirection vers un service cloud). Voir SW08 et SW12.

---

## 11. Optimisation WAN

| Technique | Principe | Bénéfice |
|---|---|---|
| **Compression** | Réduction de la taille des données | Gain de débit sur données compressibles |
| **Déduplication** | Remplacement des séquences déjà vues par des références | Fort gain sur transferts répétitifs |
| **Mise en cache** | Contenu local | Réduction du trafic WAN |
| **Optimisation de protocoles** | Réduction des échanges (SMB, HTTP) | Moins de latence perçue |

Avec le trafic massivement chiffré (TLS 1.3), l'efficacité de la compression et de la déduplication a fortement diminué. L'optimisation reste utile pour certains protocoles internes et les liens à forte latence.

---

## 12. Lab : scénarios et simulations

### 12.1 Étude de scénarios (T5, volet analyse)

| Scénario | Symptôme | Fonction du SD-WAN appropriée | Justification |
|---|---|---|---|
| Visioconférence saccadée aux heures de pointe | Perte 3 % sur le transport préféré | AAR + classe de SLA vidéo | Bascule vers le chemin conforme |
| Microsoft 365 lent dans une agence | Trafic ramené au siège | DIA + mesure de qualité SaaS | Raccourcit le chemin |
| Panne d'un accès Internet | Plus de connectivité | Multi-transport + BFD | Détection et bascule en secondes |
| Sauvegarde nocturne saturant le lien | Voix dégradée | QoS + lissage | Priorité à la voix |
| Ouverture de 15 magasins en un mois | Délais | ZTP + modèles | Déploiement sans technicien |
| Satellite : transferts lents | Fort RTT | Optimisation TCP / FEC | Récupération sans retransmission |

Complétez le tableau avec trois scénarios de votre invention.

### 12.2 Simulation Python du routage applicatif

```python
#!/usr/bin/env python3
"""Simulation d'un routage selon l'application (AAR) simplifié.

Trois chemins, mesures bruitées toutes les 10 s. La voix doit rester conforme au SLA
(perte <= 1 %, délai <= 150 ms, gigue <= 30 ms). On affiche le chemin choisi dans le temps.
"""
import random
from dataclasses import dataclass

@dataclass
class Sla:
    perte: float
    delai: float
    gigue: float

VOIX = Sla(perte=1.0, delai=150.0, gigue=30.0)

chemins = {
    "mpls":      {"perte": 0.1, "delai": 30.0, "gigue": 3.0,  "cout": 3},
    "internet":  {"perte": 0.4, "delai": 28.0, "gigue": 8.0,  "cout": 1},
    "lte":       {"perte": 1.5, "delai": 60.0, "gigue": 25.0, "cout": 2},
}
preference = ["mpls", "internet", "lte"]      # ordre de préférence

def mesure(base):
    return {
        "perte": max(0, random.gauss(base["perte"], 0.3)),
        "delai": max(1, random.gauss(base["delai"], 5)),
        "gigue": max(0, random.gauss(base["gigue"], 2)),
    }

def conforme(m, sla):
    return m["perte"] <= sla.perte and m["delai"] <= sla.delai and m["gigue"] <= sla.gigue

random.seed(7)
actuel = None
for t in range(0, 600, 10):
    # Dégradation injectée : MPLS dégradé entre 200 s et 350 s
    if 200 <= t < 350:
        chemins["mpls"]["perte"] = 3.0
        chemins["mpls"]["delai"] = 190.0
    else:
        chemins["mpls"]["perte"] = 0.1
        chemins["mpls"]["delai"] = 30.0

    mesures = {n: mesure(c) for n, c in chemins.items()}
    ok = [n for n in preference if conforme(mesures[n], VOIX)]
    choix = ok[0] if ok else min(mesures, key=lambda n: mesures[n]["perte"])  # repli : meilleur chemin
    if choix != actuel:
        print(f"t={t:4d}s  bascule {actuel} -> {choix}   (conformes : {ok or 'aucun'})")
        actuel = choix
```

Exécutez-le (`python3 simulation_aar.py`) et vérifiez que la voix bascule sur `internet` pendant la dégradation de MPLS puis revient sur `mpls`. Modifiez ensuite : ajoutez une hystérésis (ne revenir qu'après 3 mesures consécutives conformes) pour éviter les oscillations.

### 12.3 Reproduire une dégradation sur le lab WireGuard

```bash
# Sur le routeur A : dégrader l'underlay pendant 60 s puis restaurer
sudo tc qdisc add dev eth0 root netem delay 120ms 30ms loss 5%
sleep 60
sudo tc qdisc del dev eth0 root
```

Pendant ce temps, mesurez un flux UDP voix simulé :

```bash
iperf3 -c 10.200.0.2 -u -b 100k -l 160 -t 90 --get-server-output
```

Relevez gigue et perte avant, pendant et après.

### 12.4 Livrable

Tableau de scénarios complété, résultats de la simulation (capture de la sortie), analyse de 10 lignes : quelle fonction protège le mieux la voix, et à quel coût ?

---

## 13. Exercices

1. Un lien subit 4 % de perte. Avec une FEC (k = 5, 1 parité), estimez la fraction de groupes irrécupérables (perte indépendante).
2. Deux chemins : A (perte 3 %), B (perte 2 %). Quelle perte résiduelle avec duplication ?
3. Pour quels flux utiliseriez-vous la duplication de paquets ? Pour lesquels l'éviteriez-vous ?
4. Pourquoi lisser (shaper) le débit sortant à la valeur du contrat plutôt que de laisser le lien se saturer ?
5. Expliquez en trois lignes la différence entre classe de SLA et politique de QoS.

<details>
<summary>Corrigés</summary>

1. Groupe de 6 paquets, p = 0,04 : `P(0 perte) = 0,96⁶ = 0,7828 ; P(1 perte) = 6×0,04×0,96⁵ = 0,1957 ; P(≥2) = 1 − 0,9785 = 0,0215` soit **2,15 %** de groupes irrécupérables.
2. `0,03 × 0,02 = 0,0006` soit **0,06 %**.
3. Voix, signalisation, flux critiques à faible débit. À éviter pour les gros transferts (doublerait la consommation).
4. Si le lien est saturé côté opérateur, les paquets sont abandonnés dans **sa** file, sans respecter vos priorités. En lissant à la valeur du contrat, la file se forme chez vous, où la QoS la gère.
5. La classe de SLA définit des **seuils de qualité** (perte, délai, gigue) pour décider **du chemin** ; la QoS définit **le traitement local** des paquets (priorité, bande passante) sur une interface.

</details>

---

## 14. Quiz

1. Que signifie « politique centralisée, exécution distribuée » ?
2. Quelle fonction réduit le plus la perte résiduelle d'un flux voix critique ?
   a) Compression ; b) Duplication ; c) Cache ; d) Déduplication.
3. La répartition de charge par paquet est conseillée par défaut.
   a) Vrai ; b) Faux.
4. Quel est le surcoût de débit d'une duplication complète ?
5. Que mesure BFD en SD-WAN en plus de la vivacité ?
6. À quoi sert le ZTP ?
7. Pourquoi l'accès direct à Internet impose-t-il une sécurité locale ou cloud ?

<details>
<summary>Réponses</summary>

1. Les règles sont définies au centre, mais chaque équipement les applique localement sur son plan de données. 2. b. 3. b. 4. 100 %. 5. Perte, délai, gigue. 6. Mettre en service un équipement sans intervention locale, de façon sécurisée. 7. Parce que le trafic ne passe plus par le pare-feu central.

</details>

---

## 15. Aide-mémoire

| Fonction | Problème résolu | Mot-clé |
|---|---|---|
| AAR | Chemin adapté à l'application | Classe de SLA |
| FEC | Pertes sans retransmission | Parité |
| Duplication | Pertes critiques | 2 chemins |
| Segmentation | Isolation | VPN/VRF |
| QoS | Priorité à la sortie | DSCP, LLQ, shaping |
| ZTP | Déploiement | Redirection + certificats |
| DIA | Performance SaaS | Sortie locale |
| Analytique | Preuve et capacité | Tableaux de bord |

Formules : `perte résiduelle (duplication) = p₁ × p₂` ; `surcoût FEC = 1/k`.

## 16. Pour aller plus loin

- RFC 2475 (architecture DiffServ) : <https://www.rfc-editor.org/rfc/rfc2475>
- RFC 4594 (recommandations de configuration DiffServ) : <https://www.rfc-editor.org/rfc/rfc4594>
- Guides de conception Cisco SD-WAN (QoS et routage applicatif) : <https://www.cisco.com/c/en/us/solutions/enterprise-networks/sd-wan/index.html>

---

## Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **AAR** | Application-Aware Routing : choix de chemin selon l'application et la qualité mesurée |
| **Classe de SLA** | Seuils de perte, délai et gigue définissant un chemin conforme |
| **DPI** | Inspection approfondie de paquets pour identifier une application |
| **FEC** | Correction d'erreurs par redondance, sans retransmission |
| **Packet duplication** | Envoi du même paquet sur deux chemins |
| **LLQ** | Low Latency Queuing : file à priorité stricte pour la voix |
| **Shaping / policing** | Lissage (met en file) / écrêtage (jette l'excédent) |
| **DSCP** | Champ de marquage de priorité dans l'en-tête IP |
| **Flow-based / packet-based** | Répartition par flux / par paquet |
| **Hystérésis** | Délai ou marge avant de revenir à un chemin, pour éviter les oscillations |
| **Brownout** | Dégradation partielle (pertes, lenteurs) sans coupure franche ; plus difficile à détecter qu'une panne |
| **Blackout** | Coupure totale d'un lien |
| **Local breakout / DIA** | Sortie Internet locale |
| **Segment** | Réseau logique isolé (VPN, VRF) |

### B. Cas d'usage détaillés

#### Cas 1. Centre d'appels de 300 postes en voix sur IP

| Rubrique | Détail |
|---|---|
| **Exigence** | Voix de qualité, fonctionnement continu |
| **Mise en œuvre** | Classe de SLA VOICE, DSCP 46 conservé de bout en bout, file prioritaire (15 % du lien), deux accès, préférence MPLS |
| **Incident** | Brownout sur MPLS (3 % de perte) : bascule en quelques secondes sur Internet ; les appels en cours ont une brève dégradation |
| **Leçon** | Sans **hystérésis**, les appels oscillaient entre deux chemins : on impose 3 mesures conformes avant de revenir |

#### Cas 2. Application métier ancienne sensible à la latence

Une application client-serveur « bavarde » (200 échanges par écran) devient inutilisable au-delà de 60 ms. On l'identifie par adresse et port (pas de reconnaissance applicative), on lui attribue une classe `BUSINESS-LOWLAT` (délai ≤ 50 ms) et on la dirige sur MPLS. **Leçon** : la bonne mesure est parfois « le nombre d'échanges », pas le débit.

#### Cas 3. Sauvegarde nocturne qui écrase la voix

La sauvegarde commence à 19 h (équipes en poste de nuit). Solution : classe `BULK` limitée à 30 % du lien en journée, 80 % de 22 h à 5 h, et marquage CS1. **Leçon** : les politiques peuvent dépendre de l'**heure**, mais la QoS n'est pas conçue pour cela : la plupart des projets règlent cela par la planification des sauvegardes.

#### Cas 4. Lien satellite pour un site isolé

Perte faible mais RTT 600 ms : TCP se comporte mal. Mesures : optimisation TCP, taille de fenêtre adaptée, compression, interdiction de la visioconférence interactive sur ce lien. **Leçon** : il faut savoir dire à l'utilisateur ce que son lien ne permet pas.

#### Cas 5. Ouverture de 15 magasins en un mois

ZTP : l'équipement part du stock central, le gérant branche l'alimentation et le câble ; le site apparaît en 20 minutes. **Prérequis oubliés** : DHCP activé sur la box, port UDP 12346 sortant non filtré par la box du site, heure correcte. **Leçon** : préparer une fiche **« avant livraison »** envoyée à chaque site.

### C. Retours d'expérience

1. **Le DPI n'est pas parfait.** Les applications changent d'adresses et de signatures toutes les semaines ; l'**identification par DSCP ou par adresse** reste plus robuste pour les flux critiques.
2. **Une politique trop fine devient ingérable.** Trente classes d'applications et 80 règles : personne ne comprend plus le comportement. Visez **4 à 6 classes**.
3. **Mesurer avant de décider.** Beaucoup de problèmes « de WAN » viennent du Wi-Fi, d'un serveur ou d'un poste. Mesurez les trois : poste, WAN, application.
4. **L'AAR n'est pas instantané.** Il agit sur des moyennes ; il corrige les dégradations **durables**, pas les micro-coupures. Pour les micro-coupures : duplication de paquets ou FEC.
5. **La segmentation protège aussi la production.** Isoler les invités et l'IoT diminue les incidents autant que les attaques.
6. **Toute fonction a un coût CPU.** DPI, chiffrement et sécurité consomment du débit : dimensionnez avec **toutes les fonctions activées**.

### D. Exercices supplémentaires

1. Vous devez définir 4 classes de trafic pour un hôpital (voix, imagerie, dossiers patients, Internet). Fixez DSCP, classe de SLA et rôle de chaque transport.
2. Calculez le surcoût de la duplication de paquets pour 40 appels voix simultanés (G.711 : environ 90 kbit/s avec en-têtes).
3. Une politique envoie la visioconférence sur la 4G « pour économiser ». Quels risques ? Quelle alternative ?
4. Modifiez le script de simulation AAR pour ajouter une **hystérésis** (3 mesures consécutives) et comptez le nombre de bascules avant et après.

<details>
<summary>Corrigés</summary>

1. Voix : EF/46, VOICE, MPLS puis Internet ; imagerie : AF41/34, `VIDEO`-like avec débit garanti ; dossiers : AF31/26, `BUSINESS` ; Internet : 0, `DEFAULT`, sortie locale sécurisée.
2. 40 × 90 kbit/s = 3,6 Mbit/s ; la duplication double → **3,6 Mbit/s supplémentaires** sur le second chemin.
3. Perte et gigue élevées, plafond de données mobile, qualité dégradée ; alternative : second accès filaire, 4G en secours uniquement (`last-resort`).
4. Exemple : compteur `ok_consecutifs` ; ne revenir à `mpls` que si `ok_consecutifs >= 3`.

</details>

### E. Mini-projet (4 heures)

Rédigez le **catalogue de services réseau** d'une entreprise fictive : 5 classes de trafic, DSCP, classe de SLA, politique de sortie Internet, règles de segmentation, indicateurs de suivi. Une page par classe, avec un exemple d'application et la justification des seuils.

---

> Module précédent : [SW02](../SW02-transports-overlay-underlay/SW02-transports-overlay-underlay.md) | Module suivant : [SW04](../../02-partie-b-catalyst-sd-wan/SW04-architecture-composants/SW04-architecture-composants.md)
