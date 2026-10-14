# SW09. Cloud OnRamp et accès aux applications SaaS et IaaS

| | |
|---|---|
| **Partie** | B. Cisco Catalyst SD-WAN |
| **Durée indicative** | 1 semaine (10 h) |
| **Prérequis** | SW04 à SW08 ; notions de cloud public (AWS, Azure, GCP) ; DNS ; BGP |
| **Niveau** | Avancé |
| **Lien examen** | ENSDWI 300-415 (Cloud OnRamp) ; ENCOR |
| **Lab associé** | T7 (conception de la connexion à un cloud public et choix de la sortie Internet) |

> Module précédent : [SW08](../SW08-securite-integree/SW08-securite-integree.md) | Module suivant : [SW10](../SW10-supervision-exploitation-depannage/SW10-supervision-exploitation-depannage.md)

> **Avertissement.** Les offres cloud (noms de services, tarifs, limites) évoluent très vite. Les tarifs ci-dessous sont des **ordres de grandeur pédagogiques** : consultez les grilles officielles des fournisseurs avant tout chiffrage client.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Pourquoi Cloud OnRamp](#2-pourquoi-cloud-onramp)
3. [Cloud OnRamp pour le SaaS](#3-cloud-onramp-pour-le-saas)
4. [Cloud OnRamp pour l'IaaS et le multicloud](#4-cloud-onramp-pour-liaas-et-le-multicloud)
5. [Interconnexion et colocation](#5-interconnexion-et-colocation)
6. [Conception de la sortie Internet](#6-conception-de-la-sortie-internet)
7. [Optimisation des applications critiques](#7-optimisation-des-applications-critiques)
8. [Coûts et performances](#8-coûts-et-performances)
9. [Mesurer l'expérience d'une application SaaS](#9-mesurer-lexpérience-dune-application-saas)
10. [Vérifications et dépannage](#10-vérifications-et-dépannage)
11. [Lab guidé T7](#11-lab-guidé-t7)
12. [Approfondissements](#12-approfondissements)
13. [Quiz](#13-quiz)
14. [Aide-mémoire](#14-aide-mémoire)
15. [Pour aller plus loin](#15-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Expliquer le problème que résout Cloud OnRamp et ses trois grandes familles (SaaS, IaaS/multicloud, interconnexion).
- Configurer et interpréter la mesure de qualité vers une application SaaS.
- Concevoir la connexion d'une fabrique SD-WAN à un cloud public (passerelles, segments, routage).
- Comparer trois modèles de sortie Internet (locale, régionale, centralisée) avec des critères chiffrés.
- Évaluer les coûts de transit et de sortie de données du cloud.

---

## 2. Pourquoi Cloud OnRamp

```text
Avant : agence --> WAN privé --> siège --> Internet --> SaaS / cloud           (chemin long, goulot au siège)
Après : agence --> meilleur chemin mesuré (local ou régional) --> SaaS / cloud  (chemin court, adapté à la qualité mesurée)
```

**Cloud OnRamp** est le nom donné par Cisco à l'ensemble des fonctions qui connectent la fabrique SD-WAN aux clouds et en optimisent l'accès :

| Famille | Cible | Fonction |
|---|---|---|
| **Cloud OnRamp pour SaaS** | Microsoft 365, Webex, Salesforce, Google Workspace, Zoom et autres | Mesurer la qualité vers l'application et choisir le meilleur chemin de sortie |
| **Cloud OnRamp pour Multicloud** | AWS, Azure, Google Cloud | Déployer et connecter automatiquement des passerelles SD-WAN dans les clouds, relier les VPC/VNet |
| **Interconnect** | Colocation (par exemple Equinix, Megaport) | Interconnexions privées avec les clouds via des points de présence |
| **Cloud OnRamp pour Colocation** | Centres de colocation | Pile de services déployée en colocation |
| **Cloud OnRamp pour SSE/SIG** | Services de sécurité cloud | Tunnels automatisés vers le service (SW08, SW12) |

---

## 3. Cloud OnRamp pour le SaaS

### 3.1 Principe de fonctionnement

1. Le Manager active la fonction pour des **applications** (par exemple Microsoft 365) et des **sites** (clients).
2. Chaque site mesure, par des **sondes HTTP** régulières, la qualité vers l'application **par chaque sortie possible**.
3. Un **score de qualité d'expérience** (par exemple de 0 à 10) est calculé à partir de la perte et du délai.
4. Le routeur choisit la **meilleure sortie** et bascule si la qualité d'une autre devient meilleure.

```text
Agence --(sortie locale DIA 1)---------------------------> Internet --> Microsoft 365   score 8,5
Agence --(sortie locale DIA 2)---------------------------> Internet --> Microsoft 365   score 7,2
Agence --(overlay)--> Hub régional --(DIA hub)-----------> Internet --> Microsoft 365   score 9,1
Choix : hub régional (score le plus élevé)
```

### 3.2 Modes de sortie

| Mode | Description | Cas d'usage |
|---|---|---|
| **Sortie locale directe** (DIA) | Chaque site sort par ses liens Internet | Sites bien dotés en accès Internet |
| **Via une passerelle (concentrateur ou régional)** | Le trafic SaaS passe par un hub qui sort sur Internet | Petits sites à accès Internet médiocre |
| **Mixte** (meilleur chemin) | Le système choisit entre locale et passerelle | Cas général |

### 3.3 Mise en œuvre (interface du Manager)

1. **Configuration > Cloud OnRamp for SaaS** : activer ; choisir les **applications**.
2. Sélectionner les **sites clients** (agences) et, le cas échéant, les **passerelles**.
3. Définir le **VPN** concerné (par exemple le VPN 10).
4. Valider : les routeurs commencent à sonder.
5. Superviser : **Monitor > Applications > Cloud OnRamp for SaaS** (scores, chemin retenu).

### 3.3.1 Extrait de configuration (indicatif)

```text
! Sur le WAN Edge : les paramètres de mesure sont poussés par le Manager.
! Les sorties locales exigent NAT DIA actif sur le VPN concerné (voir SW06).
```

### 3.4 Commandes de vérification

```text
show sdwan cloudexpress applications
show sdwan cloudexpress gateway-exits
show sdwan cloudexpress local-exits
```

Informations lues : application, score de qualité, sortie choisie (locale ou via passerelle), état.

### 3.5 Limites et précautions

| Point | Remarque |
|---|---|
| La mesure porte sur **le chemin**, pas sur **le service interne** de l'éditeur | Une panne côté SaaS n'est pas corrigée par le SD-WAN |
| Les sondes **consomment peu** mais ne sont pas gratuites | Surveillez le volume sur 4G |
| Les sorties doivent être **sécurisées** | Pare-feu local, filtrage, ou SSE |
| Les applications SaaS changent d'adresses | Utiliser les mécanismes de l'éditeur (noms, listes officielles) |
| Plusieurs applications = plus de sondes | Limiter aux applications critiques |

---

## 4. Cloud OnRamp pour l'IaaS et le multicloud

### 4.1 Objectif

Étendre la fabrique SD-WAN **dans** les clouds publics : les sites accèdent aux charges de travail (VPC, VNet) comme à un site de plus, avec segmentation, politiques et supervision.

### 4.2 Architecture type

```text
Agences ---(overlay SD-WAN)---+
Hub / DC  ---(overlay)--------+---> Passerelle SD-WAN dans le cloud (paire de 8000V)
                                         |
                              +----------+----------+
                              |                     |
                          VPC / VNet 1          VPC / VNet 2   (applications)
```

| Cloud | Mécanismes de connexion aux réseaux applicatifs | Remarque |
|---|---|---|
| **AWS** | Transit Gateway, VPC de transit, connexion à Cloud WAN | Passerelles déployées dans un VPC de transit |
| **Azure** | Virtual WAN, VNet de transit | Passerelles dans un hub ou VNet |
| **Google Cloud** | Network Connectivity Center, VPC | Intégration à des étapes pilotes selon la version |
| **Multicloud** | Passerelles dans chaque cloud + overlay commun | Politiques de segmentation homogènes |

### 4.3 Étapes générales

1. **Préparer le compte cloud** : droits d'automatisation (rôle IAM ou principal de service) ; quotas.
2. **Associer le compte** dans le Manager (Cloud OnRamp for Multicloud > Associate Cloud Account).
3. **Créer les passerelles** (Cloud Gateway) : région, taille, nombre (haute disponibilité par paire), licences.
4. **Découvrir les réseaux hôtes** (VPC/VNet) et les **associer** à des **segments** (VPN) par étiquettes (tags).
5. **Associer les passerelles** aux VPN de la fabrique ; définir le **routage** (OMP côté SD-WAN, BGP ou tables de routage côté cloud).
6. **Politiques** : AAR, sécurité, QoS.
7. **Tester** : latence, débit, bascule d'une passerelle.

### 4.4 Choix d'architecture

| Option | Avantages | Inconvénients |
|---|---|---|
| **Passerelles SD-WAN dans un VPC de transit** | Contrôle fin, fonctions SD-WAN complètes | Coût d'instances et de licences, exploitation |
| **Intégration avec le service réseau natif du cloud** (Cloud WAN, vWAN) | Simplicité, évolutivité du fournisseur | Dépendance, limites fonctionnelles |
| **VPN IPsec directs vers les passerelles du cloud** | Simple, peu coûteux | Pas de fonctions SD-WAN, supervision limitée |
| **Interconnexion privée** (Direct Connect, ExpressRoute) | Débit et latence stables | Coût fixe, délai de livraison |

### 4.5 Exemple d'étiquetage (AWS)

Un VPC reçoit l'étiquette `SDWAN-VPN=10` : le système l'associe au VPN 10 (CORP). Convention : `SDWAN-VPN=<numéro>` ; documentée et appliquée par modèle d'infrastructure (Terraform).

```hcl
# Exemple Terraform (AWS) : étiquettes sur un VPC pour l'association à un segment
resource "aws_vpc" "app_prod" {
  cidr_block = "10.50.0.0/16"
  tags = {
    Name         = "app-prod"
    "SDWAN-VPN"  = "10"
    "Environment" = "prod"
  }
}
```

> Respectez les conventions du produit pour les noms d'étiquettes : consultez le guide Cloud OnRamp for Multicloud de votre version.

### 4.6 Résilience

- **Deux passerelles** dans deux zones de disponibilité.
- **Deux transports** depuis l'entreprise vers le cloud (overlay et lien privé).
- **Supervision** : état des passerelles, santé des tunnels, trafic.

---

## 5. Interconnexion et colocation

### 5.1 Pourquoi un point de colocation

Un **point d'interconnexion** (centre de colocation) regroupe de nombreux opérateurs et clouds. En y plaçant des équipements SD-WAN (**passerelle d'interconnexion**), on obtient :

| Avantage | Détail |
|---|---|
| **Chemins courts** vers plusieurs clouds | Latence réduite |
| **Liens privés** vers les clouds | Débit prévisible |
| **Point d'agrégation régional** | Sortie Internet régionale sécurisée |
| **Indépendance** vis-à-vis d'un opérateur ou d'un cloud | Flexibilité |

### 5.2 Interconnect Gateways (principe)

```text
Agences --(overlay)--> Passerelle d'interconnexion (colocation) --(lien privé)--> AWS / Azure / Google Cloud
                                          |
                                 Fournisseur d'interconnexion (Equinix, Megaport, …)
```

### 5.3 Hubs régionaux

Un hub par région (Europe, Amérique, Asie) évite les détours intercontinentaux. Il porte : la sortie Internet régionale, la connexion cloud régionale, la sécurité régionale.

---

## 6. Conception de la sortie Internet

### 6.1 Trois modèles

| Critère | **Locale** (DIA par site) | **Régionale** (par hubs) | **Centralisée** (siège) |
|---|---|---|---|
| Latence vers le SaaS | La meilleure | Bonne | Mauvaise (détour) |
| Sécurité | À déployer sur chaque site ou via SSE | Pile de sécurité par région | Pile unique, simple |
| Coût d'accès Internet | Élevé (un accès par site) | Moyen | Faible mais **forte consommation du WAN** |
| Complexité | Moyenne (politique par site) | Moyenne | Faible |
| Résilience | Dépend du site | Bonne (hubs redondants) | Point de défaillance central |
| Conformité | Variable | Bonne | Facile à démontrer |
| Adresse IP source vue par le SaaS | Multiples | Quelques-unes | Une seule |

### 6.2 Grille de décision (pondérée)

| Critère | Poids | Locale | Régionale | Centralisée |
|---|---|---|---|---|
| Expérience utilisateur | 30 % | 5 | 4 | 2 |
| Sécurité et conformité | 25 % | 3 | 4 | 5 |
| Coût total | 20 % | 3 | 4 | 3 |
| Simplicité d'exploitation | 15 % | 3 | 3 | 5 |
| Résilience | 10 % | 4 | 5 | 2 |
| **Score pondéré** | | **3,70** | **3,95** | **3,40** |

Calcul pour « Locale » : `0,30×5 + 0,25×3 + 0,20×3 + 0,15×3 + 0,10×4 = 1,50 + 0,75 + 0,60 + 0,45 + 0,40 = 3,70`. Recalculez toujours vos scores (voir le script ci-dessous) : une erreur d'addition dans une grille a déjà fait choisir la mauvaise architecture.

### 6.3 Calcul automatisé

```python
#!/usr/bin/env python3
"""Calcule les scores pondérés de plusieurs options (sortie Internet, solutions, etc.)."""
CRITERES = {
    "Expérience utilisateur": 0.30,
    "Sécurité et conformité": 0.25,
    "Coût total": 0.20,
    "Simplicité d'exploitation": 0.15,
    "Résilience": 0.10,
}
OPTIONS = {
    "Locale":      [5, 3, 3, 3, 4],
    "Régionale":   [4, 4, 4, 3, 5],
    "Centralisée": [2, 5, 3, 5, 2],
}
assert abs(sum(CRITERES.values()) - 1.0) < 1e-9, "les poids doivent totaliser 100 %"
for nom, notes in OPTIONS.items():
    score = sum(p * n for p, n in zip(CRITERES.values(), notes))
    print(f"{nom:12s} {score:.2f}")
```

Résultat : Locale 3,70 ; Régionale 3,95 ; Centralisée 3,40 (avec les notes ci-dessus). La **régionale** l'emporte ; mais ce résultat dépend entièrement des notes et des poids, qu'il faut **faire valider par les parties prenantes**.

### 6.4 Règle d'or

> Plus l'entreprise a de petits sites et d'utilisateurs mobiles, plus la **sécurité dans le cloud (SSE)** avec sortie locale se justifie. Plus elle a de contraintes réglementaires et peu de sites, plus la sortie **centralisée ou régionale** reste raisonnable.

---

## 7. Optimisation des applications critiques

| Application | Exigence | Mesure SD-WAN |
|---|---|---|
| **Voix** | Perte ≤ 1 %, délai ≤ 150 ms, gigue ≤ 30 ms | AAR, QoS prioritaire, duplication si disponible |
| **Visioconférence** | Perte ≤ 2 %, 200 ms, 50 ms de gigue | AAR, QoS, DIA |
| **SaaS de collaboration** | Faible latence vers l'éditeur | Cloud OnRamp SaaS |
| **ERP hébergé** | Disponibilité, sessions longues | Chemin stable, éviter les bascules fréquentes |
| **Virtualisation de poste (VDI)** | Faible gigue, 100-150 ms max | QoS spécifique, AAR |
| **Transfert de fichiers** | Débit | QoS (classe de masse), lissage |

> L'**expérience** ne se résume pas à la **qualité du réseau**. Les causes de lenteur sont aussi dans le poste, le Wi-Fi, le DNS, le serveur. Mesurez par **couche** (SW10, SW18).

---

## 8. Coûts et performances

### 8.1 Coûts du cloud à anticiper

| Poste | Observation |
|---|---|
| **Instances de passerelle** | Deux instances par région et par cloud, 24 h/24 |
| **Licences SD-WAN** | Par débit ou par instance |
| **Trafic sortant** (« egress ») | Facturé au Go par le fournisseur |
| **Trafic inter-régions et inter-zones** | Facturé |
| **Liaisons privées** | Frais mensuels et par Go |
| **Services de transit** (Transit Gateway, Virtual WAN) | Facturation par attachement et par Go |

### 8.2 Exemple de chiffrage (ordre de grandeur, pédagogique)

Hypothèses : 20 To/mois sortent du cloud vers les sites ; tarif de sortie moyen 0,07 €/Go ; deux passerelles à 700 €/mois chacune ; services de transit 400 €/mois.

| Poste | Calcul | Mensuel |
|---|---|---|
| Trafic sortant | 20 000 Go × 0,07 € | 1 400 € |
| Passerelles | 2 × 700 € | 1 400 € |
| Services de transit | forfait | 400 € |
| **Total** | | **3 200 €** |

**Leçon** : le **trafic sortant** est souvent le premier poste ; réduire le volume (cache, compression, placement des données) a plus d'effet que négocier le prix des instances.

### 8.3 Choix de région

| Critère | Remarque |
|---|---|
| Proximité des utilisateurs | Latence |
| Conformité (localisation des données) | RGPD, souveraineté |
| Coûts | Varient selon les régions |
| Disponibilité des services | Toutes les fonctions ne sont pas partout |

---

## 9. Mesurer l'expérience d'une application SaaS

Même sans fonction Cloud OnRamp, on peut **comparer des sorties** avec des mesures simples. Script de laboratoire à lancer depuis un poste de chaque site (ou depuis le routeur-hôte Linux d'un lab) :

```bash
#!/usr/bin/env bash
# mesure-saas.sh : temps de réponse HTTP vers une application, via l'interface de sortie choisie
# Usage : ./mesure-saas.sh https://outlook.office.com/ eth0 20
URL="${1:?url}"; IFACE="${2:-}"; N="${3:-20}"
printf "%-4s %-10s %-10s %-10s %-10s\n" "n" "dns_ms" "tcp_ms" "tls_ms" "total_ms"
for i in $(seq 1 "$N"); do
  OPT=()
  [ -n "$IFACE" ] && OPT=(--interface "$IFACE")
  curl -s -o /dev/null "${OPT[@]}" -w "%{time_namelookup} %{time_connect} %{time_appconnect} %{time_total}\n" "$URL" \
  | awk -v n="$i" '{printf "%-4d %-10.0f %-10.0f %-10.0f %-10.0f\n", n, $1*1000, $2*1000, $3*1000, $4*1000}'
  sleep 1
done | tee mesures.txt
echo "--- moyenne (total_ms) ---"
awk 'NR>1{s+=$5;n++} END{if(n>0) printf "%.0f ms sur %d mesures\n", s/n, n}' mesures.txt
```

À exécuter :

1. depuis un site avec sortie **locale** ;
2. depuis le même site avec sortie **via hub** (en modifiant la route ou la politique) ;
3. à différentes heures.

Comparez **DNS, TCP, TLS et total** : le décalage entre sorties montre où se perd le temps (DNS lent, détour, saturation).

---

## 10. Vérifications et dépannage

| Symptôme | Causes probables | Vérifications |
|---|---|---|
| Aucune sonde SaaS | Fonction non activée pour le VPN/site, DIA non configuré | `show sdwan cloudexpress applications`, NAT DIA |
| Score très bas partout | Problème côté application ou DNS | Test depuis un poste hors SD-WAN, tests par `curl` |
| Bascule permanente entre sorties | Écarts de score faibles, instabilité | Ajouter hystérésis, vérifier les liens |
| Passerelle cloud non joignable | Groupes de sécurité, tables de routage, licence | Console du cloud, `show sdwan control connections` |
| VPC non joint à un segment | Étiquette incorrecte, découverte non rafraîchie | Étiquettes, rafraîchir |
| Latence élevée vers le cloud | Région éloignée, hairpin | `mtr`, traceroute par VPN, choix de région |
| Facture élevée | Trafic sortant, trafic inter-régions | Rapports de coûts du fournisseur |

---

## 11. Lab guidé T7

### 11.1 Objectif

Concevoir (sur papier et avec des mesures réelles) la connexion d'une fabrique à un cloud public et choisir le modèle de sortie Internet pour une application SaaS.

### 11.2 Énoncé

Entreprise : 30 agences en France, 2 hubs régionaux (Paris et Lyon), 1 centre de données, un compte AWS (région `eu-west-3`) hébergeant l'ERP, Microsoft 365 pour tous.

À produire :

1. **Schéma d'architecture** : agences, hubs, DC, passerelles cloud (deux zones de disponibilité), VPC (prod, test), segments (CORP, SERVEURS).
2. **Tableau d'adressage cloud** : plages des VPC non chevauchantes avec le plan interne.
3. **Choix du modèle de sortie** pour Microsoft 365 parmi locale, régionale et centralisée, avec la grille pondérée (section 6) adaptée à l'entreprise.
4. **Mesures** : exécuter `mesure-saas.sh` depuis deux points (un site avec bon accès, un avec accès médiocre) vers deux URL SaaS ; comparer.
5. **Chiffrage mensuel** du cloud (instances, transit, sortie) avec hypothèses.
6. **Risques** (au moins 5) et parades.

### 11.3 Corrigé (extrait)

| Élément | Proposition |
|---|---|
| Passerelles | 2 × 8000V dans 2 AZ du VPC de transit |
| Segments | VPC prod → VPN 10 (CORP) ; VPC test → VPN 50 (TEST) |
| Adressage | VPC prod 10.50.0.0/16 ; test 10.51.0.0/16 ; (internes en 10.1.0.0/16 à 10.40.0.0/16, sans chevauchement) |
| Sortie M365 | **Régionale** pour les petites agences (hubs Paris et Lyon), **locale** pour les grandes agences avec DIA |
| Résilience | Deux passerelles, deux hubs, liens redondants |
| Risques | Chevauchement d'adresses (contrôle IPAM), coût de sortie (alertes budgétaires), dépendance à une région (plan de reprise), quotas cloud (demande préalable), erreur d'étiquette (tests) |

### 11.4 Grille

| Critère | Points |
|---|---|
| Architecture cohérente et justifiée | 5 |
| Adressage sans conflit | 3 |
| Grille pondérée et mesures réelles | 5 |
| Chiffrage argumenté | 3 |
| Risques et parades | 2 |
| Qualité de présentation | 2 |

---

## 12. Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **SaaS / IaaS / PaaS** | Logiciel / infrastructure / plate-forme en tant que service |
| **VPC / VNet** | Réseau virtuel privé dans AWS/GCP / Azure |
| **Transit Gateway / Virtual WAN** | Services de transit natifs d'AWS / d'Azure |
| **Egress** | Données sortant du cloud, facturées |
| **Hairpin** | Détour inutile par un point intermédiaire |
| **Colocation** | Hébergement d'équipements dans un centre partagé |
| **Direct Connect / ExpressRoute** | Liaisons privées vers AWS / Azure |
| **Region / AZ** | Région géographique / zone de disponibilité |
| **vQoE** | Score de qualité d'expérience d'application (terminologie Cloud OnRamp) |
| **Landing zone** | Cadre standardisé d'un environnement cloud (comptes, réseau, sécurité) |
| **IPAM** | Gestion centralisée des adresses IP |
| **Tag** | Étiquette de ressource cloud |
| **Shared responsibility** | Modèle de responsabilité partagée entre fournisseur et client |

### B. Cas d'usage détaillés

#### Cas 1. Microsoft 365 lent dans 30 agences

| Rubrique | Détail |
|---|---|
| **Symptôme** | Outlook et Teams lents ; temps de chargement de 6 s |
| **Diagnostic** | Trafic ramené au siège, proxy saturé ; mesures par `curl` : TCP 55 ms depuis le siège, 18 ms en sortie locale |
| **Solution** | Cloud OnRamp SaaS ; sortie locale pour les agences à bon DIA, sortie régionale pour les autres |
| **Résultat** | Chargement en 1,5 s ; proxy central déchargé de 60 % du trafic |
| **Leçon** | **Mesurez avant** et **après** ; la preuve chiffrée convainc la direction |

#### Cas 2. ERP migré dans le cloud

Migration d'un ERP vers AWS. Constats : latence agence-cloud passée de 12 ms à 38 ms, d'où des écrans plus lents. Solutions : passerelles dans la région la plus proche, AAR privilégiant le chemin le plus court, optimisation du cache côté application. **Leçon** : la migration d'une application **modifie la géographie du réseau** ; replanifiez le WAN avec elle.

#### Cas 3. Multicloud après une acquisition

Une société utilise AWS et rachète une entreprise sous Azure. Solution : passerelles SD-WAN dans chaque cloud, segments communs, politiques de sécurité homogènes. **Pièges** : plages d'adresses qui se recouvrent, quotas, responsabilités partagées. **Leçon** : un **plan d'adressage global (IPAM)** évite des mois de NAT.

#### Cas 4. Interconnexion régionale en colocation

Une entreprise présente en Europe et en Asie installe deux passerelles d'interconnexion (Francfort, Singapour) avec liens privés vers les clouds et sortie Internet régionale. Gains : latence divisée par 2 pour les applications cloud asiatiques, coût du transit international maîtrisé. **Leçon** : la **colocation** est l'étape logique à partir d'une certaine taille.

#### Cas 5. Facture cloud multipliée par 3

L'ajout de sites déclenche un trafic inter-zones et inter-régions mal compris. Remèdes : placer les passerelles et les applications dans les mêmes zones, mettre des budgets et alertes, optimiser le chemin de données. **Leçon** : la **gouvernance financière** (FinOps) fait partie de la conception réseau.

### C. Retours d'expérience

1. **Chiffrez le trafic sortant avant de migrer.** C'est le coût le plus sous-estimé.
2. **Évitez les chevauchements d'adresses** : tenez un IPAM unique pour l'entreprise et les clouds.
3. **Deux passerelles ne suffisent pas si elles sont dans la même zone** : répartissez sur deux zones de disponibilité.
4. **Le cloud n'est pas un site comme les autres.** Quotas, groupes de sécurité, tables de routage : une erreur de table de routage cloud est un classique.
5. **Testez la bascule** d'une passerelle : en plein jour, avec supervision.
6. **Respectez le modèle de responsabilité partagée.** Le fournisseur sécurise l'infrastructure ; vous sécurisez vos flux et vos configurations.
7. **Ne multipliez pas les clouds sans raison.** Chaque cloud supplémentaire multiplie les compétences, la sécurité et les coûts.
8. **Documentez les conventions d'étiquetage** et automatisez-les en Terraform.

### D. Exercices supplémentaires

1. Chiffrez le coût mensuel d'une architecture à 2 passerelles (900 €/mois chacune), 35 To de sortie à 0,08 €/Go, et 600 € de transit.
2. Votre ERP passe de votre DC à un cloud dont le **temps d'aller-retour** est 25 ms plus élevé. Une transaction fait 40 échanges aller-retour. Quel surcoût de latence par transaction ?
3. Vous avez 3 options de sortie Internet pour une application critique. Construisez une grille avec 5 critères, des poids et des notes, et calculez les scores.
4. Proposez un plan d'adressage pour 3 VPC et 2 VNet qui ne chevauche pas un réseau interne en 10.0.0.0/12.

<details>
<summary>Corrigés</summary>

1. `2 × 900 + 35 000 × 0,08 + 600 = 1 800 + 2 800 + 600 = 5 200 €`.
2. 25 ms de plus par aller-retour × 40 échanges = **1 000 ms** (1 s) de plus par transaction.
4. Interne : 10.0.0.0/12 (10.0 à 10.15). Cloud : VPC 10.32.0.0/16, 10.33.0.0/16, 10.34.0.0/16 ; VNet 10.40.0.0/16, 10.41.0.0/16.

</details>

### E. Mini-projet (6 heures)

Rédigez le **dossier d'architecture cloud** d'une entreprise de votre choix : choix du cloud, modèle de connexion (passerelles, interconnexion), segmentation, adressage, sécurité, coûts (hypothèses explicites), plan de test de bascule et indicateurs de suivi. Présentez-le en comité fictif d'architecture.

---

## 13. Quiz

1. Que mesure Cloud OnRamp pour le SaaS ?
2. Pourquoi mettre deux passerelles cloud dans deux zones de disponibilité ?
3. Quel est le premier poste de coûts à surveiller dans un déploiement cloud ?
4. Citez deux modèles de sortie Internet et un inconvénient de chacun.
5. Pourquoi un chevauchement d'adresses entre VPC et réseau interne est-il problématique ?
6. Quelle commande affiche les applications mesurées par Cloud OnRamp SaaS ?

<details>
<summary>Réponses</summary>

1. La qualité (perte et délai, traduits en score) du chemin vers l'application par chaque sortie possible. 2. Résilience face à la panne d'une zone. 3. Le trafic sortant (egress). 4. Locale : sécurité à déployer partout ; centralisée : détour et goulot. 5. Il impose du NAT ou rend certains chemins impossibles. 6. `show sdwan cloudexpress applications`.

</details>

---

## 14. Aide-mémoire

```text
Cloud OnRamp : SaaS (sondes + score) | Multicloud (passerelles, VPC/VNet <-> segments) | Interconnect | SSE
Sortie Internet : locale (perf) | régionale (compromis) | centralisée (contrôle)
Coûts : egress > passerelles > transit ; deux AZ ; IPAM unique ; étiquettes standardisées
Commandes : show sdwan cloudexpress applications | gateway-exits | local-exits
Mesure : curl -w "%{time_namelookup} %{time_connect} %{time_appconnect} %{time_total}"
```

## 15. Pour aller plus loin

- Cisco Catalyst SD-WAN Cloud OnRamp Configuration Guide (votre version) : <https://www.cisco.com/c/en/us/solutions/enterprise-networks/sd-wan/index.html>
- Documentation AWS Transit Gateway, Azure Virtual WAN, Google Network Connectivity Center (sites officiels des fournisseurs).
- Lecture : « Well-Architected Framework » (piliers fiabilité, coûts, sécurité) des fournisseurs de cloud.

> Module précédent : [SW08](../SW08-securite-integree/SW08-securite-integree.md) | Module suivant : [SW10](../SW10-supervision-exploitation-depannage/SW10-supervision-exploitation-depannage.md)
