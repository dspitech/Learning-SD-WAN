# SW11. Panorama des solutions et critères de choix

| | |
|---|---|
| **Partie** | C. Autres solutions, SASE et SSE |
| **Durée indicative** | 1,5 semaine (12 h) |
| **Prérequis** | SW01 à SW10 (au moins SW04 pour la comparaison avec Cisco) |
| **Niveau** | Intermédiaire à avancé (conseil et avant-vente) |
| **Lien examen** | Certifications éditeurs ; culture d'architecte |
| **Lab associé** | T9 (matrice de décision pondérée pour trois scénarios) |

> Module précédent : [SW10](../../02-partie-b-catalyst-sd-wan/SW10-supervision-exploitation-depannage/SW10-supervision-exploitation-depannage.md) | Module suivant : [SW12](../SW12-sase-sse/SW12-sase-sse.md)

> **Avertissement important.** Le marché du SD-WAN et du SASE change **très vite** : rachats, fusions, renommages de produits, évolutions de licences. Les descriptions ci-dessous reflètent l'état du marché connu à la rédaction (octobre 2026, à revérifier) et **ne constituent pas un classement**. Aucune solution n'est « la meilleure » dans l'absolu : la bonne solution dépend du **contexte** (existant, compétences, budget, exigences de sécurité et de souveraineté). Vérifiez toujours les fiches produits, les guides de conception et les conditions de licence officiels avant de recommander.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Panorama du marché](#2-panorama-du-marché)
3. [Cisco : Catalyst SD-WAN et Meraki](#3-cisco--catalyst-sd-wan-et-meraki)
4. [Fortinet Secure SD-WAN](#4-fortinet-secure-sd-wan)
5. [Palo Alto Networks Prisma SD-WAN](#5-palo-alto-networks-prisma-sd-wan)
6. [HPE Aruba Networking EdgeConnect et HPE Juniper](#6-hpe-aruba-networking-edgeconnect-et-hpe-juniper)
7. [Arista VeloCloud](#7-arista-velocloud)
8. [Versa, Cato Networks, Aryaka et services gérés](#8-versa-cato-networks-aryaka-et-services-gérés)
9. [Solutions ouvertes : FlexiWAN et constructions maison](#9-solutions-ouvertes--flexiwan-et-constructions-maison)
10. [Tableau comparatif synthétique](#10-tableau-comparatif-synthétique)
11. [Critères de choix](#11-critères-de-choix)
12. [Service géré ou autogéré](#12-service-géré-ou-autogéré)
13. [Méthode de sélection : appel d'offres et preuve de concept](#13-méthode-de-sélection--appel-doffres-et-preuve-de-concept)
14. [Matrice de décision pondérée : outil de calcul](#14-matrice-de-décision-pondérée--outil-de-calcul)
15. [Lab guidé T9](#15-lab-guidé-t9)
16. [Approfondissements](#16-approfondissements)
17. [Quiz](#17-quiz)
18. [Aide-mémoire](#18-aide-mémoire)
19. [Pour aller plus loin](#19-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Situer les principales familles de solutions et leur philosophie.
- Comparer des solutions selon une grille de critères explicites et pondérés.
- Reconnaître les modèles de gestion (autonome, cloud, géré par un opérateur) et leurs implications.
- Conduire une démarche de sélection : cadrage, appel d'offres, preuve de concept, décision.
- Justifier un choix par écrit, avec ses risques et ses hypothèses.

---

## 2. Panorama du marché

### 2.1 Quatre familles d'acteurs

| Famille | Idée directrice | Exemples |
|---|---|---|
| **Réseau d'abord** | Le SD-WAN prolonge une gamme de routage/commutation d'entreprise | Cisco Catalyst SD-WAN, HPE Aruba EdgeConnect, Juniper (HPE) |
| **Sécurité d'abord** | Le SD-WAN est une fonction d'un pare-feu de nouvelle génération | Fortinet, Palo Alto Networks |
| **Cloud d'abord / SASE natif** | Réseau et sécurité fournis comme services depuis des points de présence | Cato Networks, Versa, Aryaka, Zscaler (SSE) |
| **Gestion par le cloud, simplicité** | Équipements pilotés par un tableau de bord cloud, peu de réglages | Cisco Meraki, certaines offres de petits éditeurs |

### 2.2 Tendances

1. **Convergence SD-WAN et SASE** : presque tous proposent une offre combinée.
2. **Consolidation** : rachats fréquents (ex. VeloCloud, passé de VMware à Broadcom puis racheté par Arista en 2025 ; rapprochement de HPE et Juniper).
3. **Sécurité intégrée** de plus en plus attendue par défaut.
4. **Automatisation et API** : critère de choix majeur pour les équipes d'exploitation.
5. **Modèles de services gérés** en croissance, notamment pour les entreprises moyennes.

> Conséquence pratique : **évaluez la pérennité de l'éditeur et de la gamme** autant que les fonctions. Demandez la feuille de route et les engagements de support.

---

## 3. Cisco : Catalyst SD-WAN et Meraki

| Critère | **Catalyst SD-WAN** | **Meraki SD-WAN** |
|---|---|---|
| **Cible** | Moyennes et grandes entreprises, exigences avancées | PME, réseaux d'agences à exploitation simple |
| **Gestion** | SD-WAN Manager (cloud Cisco ou auto-hébergé) | Tableau de bord cloud unique (Dashboard) |
| **Équipements** | Catalyst 8000 (et gammes ISR/ASR en mode contrôleur) | Appliances MX |
| **Personnalisation** | Très élevée (politiques, topologies, intégration au routage) | Limitée (volontairement) |
| **Segmentation** | VPN/VRF riches | VLAN et segments plus simples |
| **Sécurité intégrée** | Pare-feu, IPS, filtrage, intégration SSE | Pare-feu et protections intégrés, intégration cloud Cisco |
| **VPN site à site** | Overlay avec politiques | « Auto VPN » : tunnels automatiques en concentrateur et rayons ou maillage |
| **Courbe d'apprentissage** | Élevée | Faible |
| **Forces** | Flexibilité, écosystème Cisco, échelle | Simplicité, rapidité de déploiement |
| **Points d'attention** | Complexité, compétences nécessaires | Moins de contrôle fin, dépendance au cloud Meraki |

Exemple d'accès par API Meraki (Python, état des liaisons montantes d'une organisation) :

```python
#!/usr/bin/env python3
"""État des liaisons montantes des appliances MX d'une organisation Meraki.
Variables d'environnement : MERAKI_API_KEY, MERAKI_ORG_ID
Documentation : https://developer.cisco.com/meraki/api-v1/
"""
import os
import requests

url = f"https://api.meraki.com/api/v1/organizations/{os.environ['MERAKI_ORG_ID']}/appliance/uplink/statuses"
entetes = {"Authorization": f"Bearer {os.environ['MERAKI_API_KEY']}"}
reponse = requests.get(url, headers=entetes, timeout=30)
reponse.raise_for_status()
for appliance in reponse.json():
    for liaison in appliance.get("uplinks", []):
        print(f"{appliance.get('networkId')}  {appliance.get('serial')}  "
              f"{liaison.get('interface'):6s} {liaison.get('status')}")
```

> Les points d'accès de l'API (chemins, paramètres) évoluent : consultez la documentation officielle de votre version.

---

## 4. Fortinet Secure SD-WAN

### 4.1 Philosophie

Le SD-WAN est une **fonction native de FortiOS**, présente sur tout pare-feu FortiGate sans licence séparée (la sécurité est la valeur centrale). L'architecte raisonne en **membres** (interfaces WAN), **zones**, **contrôles de santé** (performance SLA) et **règles SD-WAN** (stratégies de sélection).

| Élément | Rôle |
|---|---|
| **Membres SD-WAN** | Interfaces de transport (physiques ou tunnels IPsec) |
| **Zones** | Regroupements de membres (par exemple `underlay`, `overlay`) |
| **Performance SLA (health-check)** | Sondes actives (ping, HTTP, DNS) vers des serveurs de référence : perte, délai, gigue |
| **Règles SD-WAN (service)** | Sélection du chemin selon l'application, les seuils de SLA, le coût, la charge |
| **ADVPN** | Tunnels dynamiques rayon à rayon à la demande |
| **Gestion** | FortiManager (centralisée), FortiGate Cloud, orchestration de séries |

### 4.2 Stratégies de sélection

| Mode | Description |
|---|---|
| **Manuel** | Ordre de préférence fixe |
| **Meilleure qualité** | Plus faible latence/gigue/perte |
| **Plus faible coût respectant le SLA** | Chemin le moins cher parmi ceux conformes |
| **Charge maximale (load balance)** | Répartition selon la bande passante |

### 4.3 Exemple de configuration FortiOS (indicatif)

```text
config system sdwan
    set status enable
    config zone
        edit "underlay"
        next
    end
    config members
        edit 1
            set interface "wan1"
            set zone "underlay"
            set gateway 203.0.113.1
        next
        edit 2
            set interface "wan2"
            set zone "underlay"
            set gateway 198.51.100.1
        next
    end
    config health-check
        edit "SLA-INTERNET"
            set server "1.1.1.1"
            set protocol ping
            set interval 500
            set failtime 5
            set recoverytime 5
            set members 1 2
            config sla
                edit 1
                    set link-cost-factor latency jitter packet-loss
                    set latency-threshold 150
                    set jitter-threshold 30
                    set packetloss-threshold 1
                next
            end
        next
    end
    config service
        edit 1
            set name "VOIX"
            set mode sla
            set dst "all"
            set protocol 17
            set priority-members 1 2
            config sla
                edit "SLA-INTERNET"
                    set id 1
                next
            end
        next
    end
end
```

### 4.4 Commandes de diagnostic Fortinet

```text
diagnose sys sdwan health-check
diagnose sys sdwan service
diagnose sys sdwan member
diagnose sys sdwan intf-sla-log wan1
get router info routing-table all
```

### 4.5 Forces et points d'attention

| Forces | Points d'attention |
|---|---|
| Sécurité et SD-WAN dans un seul équipement, bon rapport performance/prix (processeurs dédiés) | Gestion de grandes flottes demande FortiManager et de la rigueur |
| Écosystème de sécurité (FortiGate, FortiAnalyzer, FortiSASE) | Moins de sophistication de routage inter-domaines que des routeurs dédiés dans certains cas |
| Déploiements rapides pour agences | Version et fonctions variables selon les gammes |

---

## 5. Palo Alto Networks Prisma SD-WAN

| Critère | Détail |
|---|---|
| **Philosophie** | Fabrique pilotée par les **applications** (politiques par application), gestion cloud |
| **Équipements** | Appareils « ION » (matériels et virtuels) |
| **Gestion** | Console cloud, **CloudBlades** (intégrations avec des services tiers) |
| **Intégration sécurité** | Forte avec **Prisma Access** (SSE) : tunnels automatisés |
| **Forces** | Politique par application et par chemin, intégration SSE du même éditeur |
| **Points d'attention** | Écosystème surtout cohérent avec les autres produits Palo Alto ; vérifier l'état des gammes et des licences |

---

## 6. HPE Aruba Networking EdgeConnect et HPE Juniper

| Critère | EdgeConnect (ex-Silver Peak) | Juniper Session Smart Router (HPE Juniper) |
|---|---|---|
| **Philosophie** | Optimisation WAN, routage et QoS dynamique | Routage orienté **sessions**, sans tunnels par défaut (réduction du surcoût) |
| **Gestion** | **Orchestrator** (Business Intent Overlays) | Conductor / gestion cloud (Mist) |
| **Fonctions marquantes** | Boost (optimisation), bonding de tunnels, correction d'erreurs | Politiques par service, sécurité intégrée |
| **Intégration** | Gamme Aruba (commutation, Wi-Fi, ClearPass) | Gamme Juniper et Mist AI |
| **Points d'attention** | Le rapprochement HPE et Juniper conduit à des évolutions de gammes : suivre la feuille de route | Idem |

---

## 7. Arista VeloCloud

| Critère | Détail |
|---|---|
| **Historique** | Créé par VeloCloud, racheté par VMware (2017), passé chez Broadcom, puis **racheté par Arista en juillet 2025** |
| **Architecture** | **Edges** (sites), **Gateways** (points de présence cloud), **Orchestrator** |
| **Technologie marquante** | **DMPO** (Dynamic Multipath Optimization) : agrégation de liens, correction d'erreurs, duplication |
| **Forces** | Qualité de service sur liens Internet médiocres, passerelles cloud, déploiement simple |
| **Points d'attention** | Intégration à la gamme Arista en cours, évolutions de gamme et de support à suivre |

---

## 8. Versa, Cato Networks, Aryaka et services gérés

| Acteur | Positionnement | Remarques |
|---|---|---|
| **Versa** | SD-WAN et SASE, plateforme logicielle unifiée, forte présence chez opérateurs et fournisseurs de services | Flexibilité, multi-locataire |
| **Cato Networks** | **SASE natif cloud** : réseau privé de points de présence, sécurité unifiée, équipements « Socket » légers | Modèle de service clé en main ; dépendance à un fournisseur unique |
| **Aryaka** | SD-WAN et SASE en **service géré**, cœur privé mondial | Intéressant pour les entreprises internationales sans équipe réseau |
| **Services d'opérateurs** (par exemple Orange Business, BT, Colt, NTT, Telefónica) | SD-WAN géré, souvent basé sur un éditeur partenaire (Cisco, Fortinet, Versa, VeloCloud…) | Responsabilité unique, mais contrôle plus limité et délais de changement |

---

## 9. Solutions ouvertes : FlexiWAN et constructions maison

### 9.1 FlexiWAN

| Élément | Détail |
|---|---|
| **Nature** | SD-WAN **open source** avec une offre commerciale optionnelle |
| **Architecture** | **flexiManage** (orchestration) et **flexiEdge** (agent sur Linux ou équipement) |
| **Forces** | Coût, ouverture, apprentissage, personnalisation |
| **Points d'attention** | Moins de fonctions avancées de sécurité intégrée, écosystème plus réduit, besoin de compétences Linux |
| **Usage** | Pédagogie, petites structures, laboratoires (CP04) |

### 9.2 Constructions maison (WireGuard + FRRouting)

Overlay point à point chiffré avec routage dynamique : très bon pour **comprendre**, rarement adapté à une **production à grande échelle** : pas d'orchestration, de ZTP, de mesure de chemin ni de politiques applicatives intégrées (voir SW02).

---

## 10. Tableau comparatif synthétique

> Vue **simplifiée et indicative**. Les cases dépendent de la version, de la gamme et de la licence : à valider.

| Critère | Cisco Catalyst | Cisco Meraki | Fortinet | Palo Alto Prisma | Aruba EdgeConnect | Arista VeloCloud | Versa | Cato | FlexiWAN |
|---|---|---|---|---|---|---|---|---|---|
| **Maturité grandes flottes** | Élevée | Moyenne | Élevée | Moyenne à élevée | Élevée | Élevée | Élevée | Moyenne à élevée | Faible à moyenne |
| **Sécurité native** | Bonne | Bonne | Très forte | Forte (avec Prisma) | Moyenne | Moyenne | Forte | Forte (cloud) | Limitée |
| **Simplicité** | Faible | Très forte | Moyenne | Moyenne | Moyenne | Forte | Moyenne | Très forte | Moyenne |
| **Personnalisation** | Très forte | Faible | Forte | Moyenne | Forte | Moyenne | Très forte | Faible | Très forte |
| **API et automatisation** | Très riche | Riche | Riche | Riche | Riche | Riche | Riche | Moyenne | Riche (ouverte) |
| **Gestion** | Cloud ou auto-hébergé | Cloud | Cloud ou auto-hébergé | Cloud | Cloud ou auto-hébergé | Cloud ou auto-hébergé | Auto-hébergé ou cloud | Cloud (service) | Auto-hébergé ou cloud |
| **Intégration SASE** | Oui (Cisco) | Oui (Cisco) | Oui (FortiSASE) | Oui (Prisma Access) | Oui (partenaires) | Oui (partenaires) | Oui (natif) | Natif | Non |
| **Coût (indicatif)** | Moyen à élevé | Moyen | Faible à moyen | Moyen à élevé | Moyen | Moyen | Moyen | Moyen à élevé (service) | Faible |

---

## 11. Critères de choix

### 11.1 Les dix critères

| # | Critère | Questions à poser |
|---|---|---|
| 1 | **Cohérence avec l'existant** | LAN, Wi-Fi, pare-feu, cloud : quelle gamme déjà en place ? |
| 2 | **Sécurité intégrée** | Besoin de pare-feu, IPS, filtrage ? SSE du même éditeur ? |
| 3 | **Simplicité d'exploitation** | Taille et compétences de l'équipe ? |
| 4 | **Modèle de gestion** | Autonome, cloud, géré par un opérateur ? |
| 5 | **Couverture géographique** | Présence des points de présence, du support, de la livraison dans les pays concernés |
| 6 | **Coûts et licences** | Matériel, licences (par débit, par site, par fonction), support, durée d'engagement |
| 7 | **Écosystème d'API** | Automatisation, Terraform, intégrations ITSM/SIEM |
| 8 | **Pérennité** | Santé de l'éditeur, feuille de route, rachats récents |
| 9 | **Souveraineté et conformité** | Localisation des données, certifications (ISO 27001, SecNumCloud, etc.) |
| 10 | **Performance** | Débits avec sécurité activée, capacité de passage à l'échelle |

### 11.2 Comment fixer les poids

1. Faire un atelier avec les parties prenantes (réseau, sécurité, direction, métier).
2. Chaque participant répartit 100 points sur les critères.
3. Retenir la **moyenne**, discuter les écarts importants.
4. Verrouiller les poids **avant** de noter les solutions (pour éviter de « fabriquer » le résultat attendu).

### 11.3 Critères éliminatoires

À fixer **en premier** : certaines exigences ne se pondèrent pas, elles éliminent (par exemple hébergement des données en France ou en UE, certification obligatoire, compatibilité avec un équipement existant critique). Ne notez que les solutions qui passent ces filtres.

---

## 12. Service géré ou autogéré

| Critère | Autogéré | Géré par un opérateur/intégrateur |
|---|---|---|
| **Contrôle** | Total | Limité par le contrat |
| **Vitesse de changement** | Immédiate | Délais (demande, ticket, fenêtre) |
| **Compétences requises** | Élevées en interne | Faibles en interne |
| **Coût** | Équipe + outils | Abonnement mensuel (souvent plus élevé en cumulé) |
| **Responsabilité en cas d'incident** | Interne | Contractuelle (SLA, pénalités) |
| **Risque principal** | Manque de compétences, bus factor | Dépendance, opacité, difficultés de sortie |
| **Souplesse de sortie** | Haute | Faible (contrat, équipements propriétaires) |
| **Visibilité** | Totale | À négocier (accès lecture au tableau de bord, API) |

**Modèle intermédiaire (co-géré)** : le client garde l'accès au tableau de bord et à la politique, l'opérateur gère l'infrastructure, les mises à jour et l'astreinte.

Questions à poser à un prestataire de service géré :

1. Puis-je accéder aux **journaux, aux mesures et à l'API** ?
2. Quel est le **délai** de traitement d'un changement standard / urgent ?
3. Que se passe-t-il à **la fin du contrat** (propriété des équipements, configurations exportables) ?
4. Quels sont les **engagements** (disponibilité, GTR, remplacement matériel) et les pénalités ?
5. Où sont hébergés les **contrôleurs** et les **données** ?

---

## 13. Méthode de sélection : appel d'offres et preuve de concept

### 13.1 Étapes

```text
1. Cadrage      : besoins, contraintes, critères éliminatoires, poids
2. Consultation : 3 à 5 solutions présélectionnées (RFI puis RFP)
3. Démonstrations structurées : scénarios imposés (pas des démos génériques)
4. Preuve de concept (POC) : 2 solutions, 4 à 8 semaines, sites représentatifs
5. Notation et analyse de coût total sur 3 à 5 ans
6. Décision et négociation
7. Plan de déploiement
```

### 13.2 Scénarios de POC (exemples)

| # | Scénario | Critère d'acceptation |
|---|---|---|
| 1 | Déploiement sans intervention d'un site | En moins de 30 minutes, sans technicien réseau |
| 2 | Panne d'un transport | Bascule en moins de 10 s, sans coupure d'appel voix perceptible |
| 3 | Dégradation (perte 5 %) | Bascule selon politique en moins de 2 min |
| 4 | Segmentation et sécurité | Isolement de deux segments démontré par test |
| 5 | Sortie Internet locale sécurisée | Filtrage actif, journaux disponibles |
| 6 | API | Inventaire et création d'un site par script |
| 7 | Mise à jour logicielle | Sans interruption supérieure à 2 min par site |
| 8 | Exploitation | Un opérateur novice dépanne une panne avec le runbook |

### 13.3 Pièges de la POC

1. **Conditions non représentatives** : tester en laboratoire propre alors que la production a des NAT, des CGNAT et des pare-feu.
2. **Fonctions en préversion** : exiger que la POC n'utilise que des fonctions **en disponibilité générale**.
3. **Démonstration pilotée par le fournisseur** : demandez que **vos** équipes pilotent.
4. **Absence de critère d'acceptation** : chaque scénario doit avoir un résultat attendu mesurable.

---

## 14. Matrice de décision pondérée : outil de calcul

```python
#!/usr/bin/env python3
"""Matrice de décision pondérée avec analyse de sensibilité.

Format du fichier decision.csv (séparateur virgule) :
critere,poids,SolutionA,SolutionB,SolutionC
Cohérence avec l'existant,20,5,3,4
Sécurité intégrée,20,3,5,4
...
Notes de 1 (faible) à 5 (excellent). Poids en points (la somme est normalisée à 100 %).
"""
import csv
import sys
from itertools import product

def charger(chemin):
    with open(chemin, newline="", encoding="utf-8") as f:
        lecteur = csv.DictReader(f)
        solutions = [c for c in lecteur.fieldnames if c not in ("critere", "poids")]
        lignes = list(lecteur)
    poids = [float(l["poids"]) for l in lignes]
    total = sum(poids)
    poids = [p / total for p in poids]
    notes = {s: [float(l[s]) for l in lignes] for s in solutions}
    criteres = [l["critere"] for l in lignes]
    return criteres, poids, notes

def scores(poids, notes):
    return {s: sum(p * n for p, n in zip(poids, v)) for s, v in notes.items()}

def sensibilite(poids, notes, variation=0.25):
    """Fait varier chaque poids de +-25 % et compte combien de fois chaque solution arrive première."""
    victoires = {s: 0 for s in notes}
    essais = 0
    for facteurs in product((1 - variation, 1, 1 + variation), repeat=len(poids)):
        p = [a * b for a, b in zip(poids, facteurs)]
        t = sum(p)
        p = [x / t for x in p]
        sc = scores(p, notes)
        victoires[max(sc, key=sc.get)] += 1
        essais += 1
    return {s: v / essais for s, v in victoires.items()}

if __name__ == "__main__":
    criteres, poids, notes = charger(sys.argv[1] if len(sys.argv) > 1 else "decision.csv")
    print("Scores pondérés :")
    for s, v in sorted(scores(poids, notes).items(), key=lambda x: -x[1]):
        print(f"  {s:20s} {v:.2f} / 5")
    print("\nRobustesse (part des variations de poids où la solution arrive première) :")
    for s, v in sorted(sensibilite(poids, notes).items(), key=lambda x: -x[1]):
        print(f"  {s:20s} {v:.0%}")
```

Fichier d'exemple `decision.csv` (scénario « PME de 25 magasins », **notes purement illustratives**) :

```csv
critere,poids,Meraki,Fortinet,Catalyst,Cato
Cohérence avec l'existant,15,3,3,2,2
Sécurité intégrée,25,3,5,4,5
Simplicité d'exploitation,25,5,3,2,5
Coût total sur 3 ans,20,4,5,2,3
API et automatisation,5,4,4,5,3
Pérennité,5,4,4,5,4
Souveraineté et conformité,5,3,4,4,3
```

**Interprétation** : le score donne un classement ; la **robustesse** dit si ce classement résiste à des variations raisonnables des poids. Un résultat qui bascule à la moindre variation signifie que **deux solutions sont équivalentes pour votre besoin** : départagez-les par la POC et la négociation.

---

## 15. Lab guidé T9

### 15.1 Objectif

Construire une matrice de décision pondérée pour **trois scénarios**, avec justification écrite.

### 15.2 Scénarios

| Scénario | Description |
|---|---|
| **A. PME multi-sites** | 25 agences en France, 1 informaticien, Microsoft 365, pas de centre de données, budget limité |
| **B. Entreprise internationale** | 120 sites dans 40 pays, centres de données régionaux, équipe réseau de 8 personnes, exigences de conformité fortes |
| **C. Réseau de commerces** | 600 magasins, caisses (conformité de paiement), 4G de secours, déploiement de masse, peu de compétences locales |

### 15.3 Travail à produire

Pour chaque scénario :

1. critères **éliminatoires** et leur justification ;
2. critères pondérés (somme 100) avec la justification des poids ;
3. choix de **3 solutions** à comparer et notes (1 à 5) avec une phrase de justification par note ;
4. résultat du script (scores et robustesse) ;
5. recommandation argumentée : solution, modèle de gestion (autogéré, cloud, géré), risques et conditions ;
6. plan de POC (3 scénarios de test et critères d'acceptation).

### 15.4 Éléments de correction (attendus)

| Scénario | Orientation plausible | Raisonnement |
|---|---|---|
| **A** | Solution à gestion cloud simple (Meraki, Fortinet, ou service cloud) | Une seule personne : la simplicité et la sécurité intégrée pèsent plus que la personnalisation |
| **B** | Plateforme riche et extensible (Catalyst SD-WAN, Fortinet avec FortiManager, Versa, Aruba) ; SSE régional | Besoins d'échelle, de personnalisation, d'API, de conformité : accepter la complexité |
| **C** | Solution de déploiement de masse avec ZTP, sécurité de paiement et 4G (Fortinet, Meraki, Cato, service géré) | Le volume et le manque de compétences locales dominent |

> Plusieurs réponses sont défendables : **la qualité de l'argumentation prime sur le nom de la solution retenue**.

### 15.5 Grille

| Critère | Points |
|---|---|
| Critères éliminatoires pertinents | 3 |
| Pondération argumentée | 4 |
| Notes justifiées (pas d'adjectifs creux) | 4 |
| Utilisation correcte du script et lecture de la robustesse | 3 |
| Recommandation claire avec risques | 4 |
| Plan de POC | 2 |

---

## 16. Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **RFI / RFP / RFQ** | Demande d'information / de proposition / de prix |
| **POC** | Preuve de concept |
| **TCO** | Coût total de possession |
| **Lock-in** | Enfermement chez un fournisseur |
| **Bus factor** | Nombre de personnes dont le départ paralyserait l'exploitation |
| **MSP** | Prestataire de services gérés |
| **SASE / SSE** | Convergence réseau et sécurité cloud / sa partie sécurité |
| **DMPO** | Optimisation dynamique multi-chemins (VeloCloud) |
| **ADVPN** | Tunnels dynamiques rayon à rayon (Fortinet) |
| **BIO** | Business Intent Overlay (Aruba EdgeConnect) |
| **SSR** | Session Smart Router (Juniper) |
| **GA / Preview** | Disponibilité générale / préversion d'une fonction |
| **Critère éliminatoire** | Exigence binaire qui écarte une solution |
| **Analyse de sensibilité** | Étude de l'effet d'une variation des hypothèses sur le résultat |

### B. Cas d'usage détaillés

#### Cas 1. PME de 25 agences : le choix de la simplicité

| Rubrique | Détail |
|---|---|
| **Contexte** | 1 informaticien polyvalent, budget serré, Microsoft 365 |
| **Solutions examinées** | Meraki, Fortinet, Catalyst, service géré d'un opérateur |
| **Décision** | Fortinet avec gestion cloud simple (sécurité native, bon prix) ; Catalyst écarté pour la complexité ; service géré écarté pour le coût cumulé |
| **Résultat** | Déploiement de 25 sites en 8 semaines, sécurité unifiée |
| **Leçon** | Une solution **plus simple et moins puissante** vaut mieux qu'une solution riche **que l'équipe ne maîtrise pas** |

#### Cas 2. Industriel international : écarter le « tout cloud »

120 sites dans 40 pays, certains pays avec contraintes locales de transit de données et de filtrage. Le SSE d'un fournisseur unique n'a pas de point de présence dans 6 pays. Décision : plateforme SD-WAN extensible, SSE régional **plus** pare-feu local dans les pays non couverts. **Leçon** : la **couverture géographique** est un critère éliminatoire souvent oublié.

#### Cas 3. 600 magasins : le déploiement de masse décide

Le coût matériel est secondaire ; le facteur déterminant est la **capacité à déployer 600 sites en 6 mois sans technicien**. Le prestataire retenu fournit : étiquetage des colis, ZTP éprouvé, centre d'appels de déploiement, pièces de rechange. **Leçon** : évaluez **l'écosystème de déploiement** (logistique, support) autant que la technologie.

#### Cas 4. Rachat du fournisseur en cours de contrat

Un éditeur est racheté ; la gamme est fusionnée avec une autre. Le client constate des délais de support plus longs et une feuille de route incertaine. **Prévention** : clause de **continuité de support** (durée minimale), droit de sortie, exports de configuration, suivi des annonces de l'éditeur. **Leçon** : la **pérennité** est un risque réel, à traiter contractuellement.

#### Cas 5. POC qui change la décision

Sur le papier, la solution A est première. En POC, avec de vrais accès (CGNAT 4G, box grand public), la solution A échoue à établir des tunnels sur 30 % des sites alors que B réussit. **Leçon** : une POC **représentative** vaut cent fiches produits.

#### Cas 6. Coût caché : licences par débit

La solution C est moins chère à l'achat mais facture les licences **par débit**, ce qui multiplie le coût à chaque montée en débit des sites. Sur 5 ans, elle devient la plus chère. **Leçon** : modélisez le TCO **avec la croissance** attendue des débits.

### C. Retours d'expérience

1. **Ne choisissez pas sur une démonstration de 45 minutes.** Une démo est un spectacle ; une POC est un test.
2. **Gardez la neutralité.** Un architecte qui défend « sa » marque perd sa crédibilité. Rédigez la matrice avant de discuter des marques.
3. **Le meilleur produit est celui que l'équipe sait exploiter.** La complexité mal maîtrisée est la première cause d'échec.
4. **Demandez des références comparables** (même taille, même secteur, même pays) et **appelez-les**.
5. **Lisez les conditions de licence.** Par site, par débit, par fonction, par durée : les différences se chiffrent en dizaines de pour cent.
6. **Prévoyez la sortie.** Demandez les formats d'export des configurations et vérifiez la propriété des équipements.
7. **Évitez les engagements trop longs** sans clause de révision ; le marché change tous les 18 mois.
8. **Négociez après la POC**, pas avant : le fournisseur est alors le plus motivé.

### D. Exercices supplémentaires

1. Rédigez 6 critères éliminatoires pour une mutuelle de santé (hébergement des données de santé) souhaitant du SD-WAN.
2. Calculez le TCO sur 5 ans de deux solutions pour 50 sites : A = 4 000 € de matériel par site + 600 €/an de licence + 20 000 €/an de support ; B = 2 500 € de matériel par site + 1 200 €/an de licence + 10 000 €/an de support. À partir de combien d'années B devient-elle plus chère ?
3. Exécutez le script avec le fichier d'exemple et modifiez les poids pour que « Catalyst » arrive premier. Que concluez-vous sur la fragilité d'un résultat ?
4. Rédigez cinq questions pièges à poser lors d'une démonstration pour tester la maturité d'un produit.

<details>
<summary>Corrigés</summary>

2. A : `50×4000 = 200 000` + `50×600×n = 30 000 n` + `20 000 n` = `200 000 + 50 000 n`. B : `50×2500 = 125 000` + `50×1200×n = 60 000 n` + `10 000 n` = `125 000 + 70 000 n`. Égalité : `200 000 + 50 000 n = 125 000 + 70 000 n` → `n = 3,75` années. Au-delà de **3,75 ans**, B devient plus chère que A. Sur 5 ans : A = 450 000 € ; B = 475 000 €.
3. Il faut augmenter fortement les poids de l'API et de la pérennité, ce qui ne reflète pas le besoin initial : un résultat qui n'arrive à basculer qu'avec des poids extrêmes est robuste ; un résultat qui bascule à ±10 % est fragile.
4. Exemples : « Montrez la bascule d'un lien en direct », « Montrez la mise à jour d'un site sans coupure », « Montrez l'API pour créer un site », « Que se passe-t-il quand le contrôleur est injoignable ? », « Comment exportez-vous toute la configuration ? ».

</details>

### E. Mini-projet (8 heures)

Rédigez le **dossier de choix** pour un client fictif : cadrage, critères éliminatoires, grille pondérée (script), comparaison de trois solutions avec sources citées (fiches officielles), plan de POC en 8 scénarios, analyse de TCO sur 5 ans, risques et clauses contractuelles à négocier. Présentez-le en 10 minutes.

---

## 17. Quiz

1. Quelles sont les quatre familles d'acteurs du marché ?
2. Qu'est-ce qu'un critère éliminatoire et pourquoi le fixer avant la pondération ?
3. Quel est l'avantage principal d'un service géré ? Son risque principal ?
4. Pourquoi une POC doit-elle être pilotée par vos équipes ?
5. Qu'est-ce que l'analyse de sensibilité apporte à une matrice de décision ?
6. Citez deux critères souvent oubliés dans les choix de solution.

<details>
<summary>Réponses</summary>

1. Réseau d'abord, sécurité d'abord, cloud d'abord / SASE natif, gestion par le cloud et simplicité. 2. Une exigence binaire qui écarte une solution ; la fixer d'abord évite de noter des solutions inacceptables et de biaiser la pondération. 3. Avantage : peu de compétences internes et responsabilité contractuelle ; risque : dépendance et manque de contrôle. 4. Pour vérifier que l'exploitation réelle est à votre portée et éviter une démonstration « sur mesure ». 5. Elle montre si le classement résiste à des variations des poids. 6. Couverture géographique et logistique de déploiement ; conditions de sortie ; modèle de licence.

</details>

---

## 18. Aide-mémoire

```text
Familles : réseau d'abord | sécurité d'abord | SASE natif | gestion cloud simple
Méthode  : cadrage -> critères éliminatoires -> poids -> RFI/RFP -> démos structurées -> POC -> TCO -> décision
Piège n°1: choisir sur une démo ; n°2 : ignorer la pérennité ; n°3 : oublier la licence par débit ; n°4 : POC non représentative
Autogéré vs géré : contrôle et agilité <-> délégation et responsabilité contractuelle
Toujours : vérifier l'état du marché (rachats), les fonctions GA, les conditions de sortie
```

## 19. Pour aller plus loin

- Cisco : <https://www.cisco.com/c/en/us/solutions/enterprise-networks/sd-wan/index.html> et <https://meraki.cisco.com>
- Fortinet : <https://docs.fortinet.com>
- FlexiWAN : <https://flexiwan.com>
- MEF (définition normalisée du SD-WAN, MEF 70 et suivantes) : <https://www.mef.net>
- Fiches et guides des éditeurs (Palo Alto Networks, HPE Aruba Networking, Arista, Versa, Cato, Aryaka) : sites officiels.

> Module précédent : [SW10](../../02-partie-b-catalyst-sd-wan/SW10-supervision-exploitation-depannage/SW10-supervision-exploitation-depannage.md) | Module suivant : [SW12](../SW12-sase-sse/SW12-sase-sse.md)
