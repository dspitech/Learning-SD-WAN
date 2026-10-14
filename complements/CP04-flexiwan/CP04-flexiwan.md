# CP04. FlexiWAN : un SD-WAN open source pour apprendre et expérimenter

| | |
|---|---|
| **Partie** | Compléments (hors parcours principal) |
| **Durée indicative** | 1 semaine (8 à 10 h) |
| **Prérequis** | SW01 à SW03 ; Linux (Ubuntu), réseau (VLAN, NAT, routage), virtualisation |
| **Niveau** | Intermédiaire |
| **Lien examen** | Culture technique ; utile pour comprendre les concepts indépendamment d'un éditeur |
| **Lab associé** | Lab CP04 (SD-WAN à trois sites avec flexiManage et flexiEdge) |

> Module précédent : [CP03](../CP03-ipsec-en-profondeur/CP03-ipsec-en-profondeur.md) | Module suivant : [CP05](../CP05-telemetrie-observabilite/CP05-telemetrie-observabilite.md)

> **Avertissement.** FlexiWAN évolue : interface, noms de menus, versions d'Ubuntu prises en charge et commandes de l'agent peuvent changer. Les étapes suivent la **logique générale** ; suivez la documentation officielle (<https://docs.flexiwan.com>) pour les commandes exactes de votre version. Aucune syntaxe d'agent n'est garantie ici : vérifiez-les avec `--help`.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Présentation de FlexiWAN](#2-présentation-de-flexiwan)
3. [Architecture](#3-architecture)
4. [Fonctions principales](#4-fonctions-principales)
5. [Préparer le laboratoire](#5-préparer-le-laboratoire)
6. [Mise en route pas à pas](#6-mise-en-route-pas-à-pas)
7. [Tunnels, étiquettes de chemin et politiques](#7-tunnels-étiquettes-de-chemin-et-politiques)
8. [Supervision et dépannage](#8-supervision-et-dépannage)
9. [Comparaison avec Cisco Catalyst SD-WAN](#9-comparaison-avec-cisco-catalyst-sd-wan)
10. [Cas d'usage et limites](#10-cas-dusage-et-limites)
11. [Lab guidé CP04](#11-lab-guidé-cp04)
12. [Approfondissements](#12-approfondissements)
13. [Quiz](#13-quiz)
14. [Aide-mémoire](#14-aide-mémoire)
15. [Pour aller plus loin](#15-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Situer FlexiWAN dans le paysage des solutions SD-WAN.
- Identifier ses composants (flexiManage, flexiEdge) et les relier aux plans d'un SD-WAN.
- Monter un laboratoire de trois sites : enregistrement, interfaces, tunnels, étiquettes de chemin, politiques.
- Observer, mesurer et dépanner la solution.
- Comparer les concepts avec Cisco Catalyst SD-WAN et argumenter les cas d'usage.

---

## 2. Présentation de FlexiWAN

FlexiWAN est une plateforme SD-WAN **ouverte** : un système de gestion (**flexiManage**) et un agent de routage (**flexiEdge**) fonctionnant sur Linux. Son code est largement ouvert, avec une offre commerciale optionnelle (hébergement du gestionnaire, support).

| Aspect | Détail |
|---|---|
| **Licence** | Open source pour les composants principaux ; offres commerciales complémentaires |
| **Public** | Intégrateurs, fournisseurs de services, laboratoires, petites et moyennes structures avec compétences Linux |
| **Atout pédagogique** | Concepts SD-WAN visibles, code consultable, déploiement sur n'importe quelle machine virtuelle |
| **Moteur de transfert** | Basé sur un routeur logiciel rapide (**VPP**, Vector Packet Processing) |
| **Gestion** | **Hébergée** (service du fournisseur) ou **auto-hébergée** |

---

## 3. Architecture

```text
                       +---------------------------+
                       |        flexiManage        |   gestion : organisations, équipements, tunnels,
                       |  (SaaS ou auto-hébergé)   |   politiques, supervision, API
                       +-------------+-------------+
                                     | connexion sécurisée (WebSocket/TLS), configuration poussée
        +----------------------------+----------------------------+
        |                            |                            |
  +-----+------+              +------+-----+              +-------+-----+
  | flexiEdge  |==== tunnel ==| flexiEdge  |==== tunnel ==| flexiEdge   |
  | (agent +   |  (chiffré)   | (agent +   |  (chiffré)   | (agent +    |
  |  VPP)      |              |  VPP)      |              |  VPP)       |
  +------------+              +------------+              +-------------+
     Site A                      Site B                      Site C
```

| Composant | Rôle | Équivalent Cisco |
|---|---|---|
| **flexiManage** | Gestion et orchestration centralisées | Manager (et en partie Validator) |
| **flexiEdge (agent)** | Applique la configuration, supervise, communique avec flexiManage | Agent de gestion du WAN Edge |
| **VPP** | Plan de données rapide (routage, tunnels) | Moteur de transfert (QFP) |
| **Tunnels** | Entre flexiEdge, chiffrés | Tunnels IPsec entre TLOC |
| **Étiquettes de chemin** (path labels) | Nom d'un type de lien (MPLS, Internet…) | Couleurs |

> Le plan de contrôle est plus **centralisé** que dans Cisco : flexiManage calcule et pousse la configuration des tunnels ; il n'y a pas d'équivalent exact d'OMP.

---

## 4. Fonctions principales

| Fonction | Description |
|---|---|
| **Enregistrement par jeton** | Un équipement s'enregistre avec un jeton d'organisation |
| **Multi-liens (WAN)** | Plusieurs interfaces WAN par équipement |
| **Tunnels automatiques** | Création de tunnels entre équipements selon les étiquettes de chemin |
| **Politiques de chemin** | Sélection par application, étiquette de lien, ordre de préférence, repli |
| **Qualité de lien** | Mesures de perte, délai, gigue par tunnel |
| **QoS** | Classes de trafic, files, lissage |
| **Pare-feu** | Règles d'entrée/sortie par interface |
| **Routage** | Statique, OSPF, BGP |
| **NAT et traversée** | Prise en charge des équipements derrière NAT |
| **Applications** | Bibliothèque d'applications reconnues, classification personnalisée |
| **API REST** | Automatisation (organisations, équipements, tunnels) |
| **Notifications** | Alertes e-mail, webhooks |

---

## 5. Préparer le laboratoire

### 5.1 Ressources

| Élément | Minimum |
|---|---|
| **3 machines Linux** (flexiEdge) | Ubuntu Server (version recommandée dans la documentation actuelle), 2 vCPU, 4 Go de RAM, 2 interfaces réseau minimum (WAN et LAN) |
| **flexiManage** | Service hébergé (compte gratuit) ou installation auto-hébergée |
| **Hôtes LAN** | 3 petites machines ou conteneurs |
| **Réseau « Internet » simulé** | Commutateur virtuel/routeur de transport entre les sites |

### 5.2 Topologie du lab

```text
 LAN A 192.168.10.0/24         LAN B 192.168.20.0/24         LAN C 192.168.30.0/24
        |                              |                              |
   [ flexiEdge A ]               [ flexiEdge B ]               [ flexiEdge C ]
    WAN1 203.0.113.11            WAN1 203.0.113.12            WAN1 203.0.113.13
    WAN2 198.51.100.11           WAN2 198.51.100.12           WAN2 198.51.100.13
        |____________________ réseau de transport (2 chemins : "INTERNET" et "LTE") ____________________|
```

### 5.3 Prérequis système

```bash
sudo apt update && sudo apt -y upgrade
sudo apt install -y curl net-tools tcpdump iperf3 mtr-tiny
# Désactiver les services qui monopolisent les interfaces utilisées par VPP (voir documentation)
# Vérifier l'horloge
timedatectl status
```

---

## 6. Mise en route pas à pas

> Chaque étape est **indicative** : la documentation officielle reste la référence pour les commandes exactes.

### 6.1 Créer l'organisation et le jeton

1. Se connecter à flexiManage (service hébergé ou auto-hébergé).
2. Créer une **organisation** et un **compte** administrateur.
3. Créer un **jeton d'accès** (menu *Inventory > Tokens*).

### 6.2 Installer l'agent sur chaque site

La documentation propose un **script d'installation** (dépôt de paquets de FlexiWAN) puis l'installation de l'agent :

```bash
# Principe (à remplacer par les commandes de la documentation officielle de votre version)
curl -sL https://deb.flexiwan.com/setup | sudo -E bash -
sudo apt install -y flexiwan-router
```

Puis l'**enregistrement** avec le jeton :

```bash
sudo fwagent configure        # ou l'assistant : saisie du jeton, nom de l'équipement (voir fwagent --help)
sudo systemctl status flexiwan-router
```

### 6.3 Approuver les équipements

Dans flexiManage, les équipements apparaissent en **« Unapproved »** : les **approuver** (contrôle d'identité : l'équivalent de la liste d'autorisation Cisco).

### 6.4 Configurer les interfaces

Pour chaque équipement (menu *Devices > Interfaces*) :

| Interface | Type | Adresse | Passerelle | Étiquette de chemin |
|---|---|---|---|---|
| eth1 | WAN | 203.0.113.11/24 | 203.0.113.1 | INTERNET |
| eth2 | WAN | 198.51.100.11/24 | 198.51.100.1 | LTE |
| eth3 | LAN | 192.168.10.1/24 | — | — |

Activer l'interface, le routage de LAN, DHCP si besoin ; **démarrer le routeur** sur chaque équipement (bouton *Start*).

### 6.5 Vérifier la connexion

Les équipements doivent apparaître **Connected** et **Running** dans flexiManage.

---

## 7. Tunnels, étiquettes de chemin et politiques

### 7.1 Étiquettes de chemin (path labels)

Créer **INTERNET** et **LTE** (*Inventory > Path Labels*), les associer aux interfaces WAN correspondantes. C'est l'équivalent des **couleurs** : elles servent à créer des tunnels cohérents et à exprimer les politiques.

### 7.2 Tunnels

*Network > Tunnels* : sélectionner les équipements, flexiManage crée les tunnels **entre interfaces compatibles** (mêmes étiquettes ou toutes combinaisons). Chaque tunnel est **chiffré**. Vérifier leur état et leurs mesures (perte, délai, gigue).

Nombre de tunnels : pour N équipements à 2 liens et étiquettes appariées : `N(N−1)/2 × 2`. Pour 3 sites : `3 × 2 = 6` tunnels.

### 7.3 Politiques de chemin (multi-liens)

*Services > Path selection* (« Multi-Link policy ») : règle = **application ou classe de trafic** + **ordre de préférence des étiquettes** + **mode** (par flux, répartition) + **repli**.

| Règle | Trafic | Préférence | Repli |
|---|---|---|---|
| VOIX | Voix et visioconférence (classification par application) | INTERNET puis LTE | Oui |
| SAUVEGARDE | Port 873/tcp et 22/tcp vers `192.168.30.50` | INTERNET seulement | Non |
| DEFAUT | Tout le reste | INTERNET, LTE en secours | Oui |

### 7.4 Routage entre sites

Ajouter des **routes statiques** (ou BGP/OSPF) pour joindre les LAN distants via les tunnels ; flexiWAN peut les propager automatiquement selon la configuration (voir la documentation).

### 7.5 Pare-feu et QoS

Règles d'entrée par interface WAN (autoriser seulement ce qui est nécessaire), classes de QoS avec priorité pour la voix (concepts de CP02).

---

## 8. Supervision et dépannage

### 8.1 Dans flexiManage

| Écran | Informations |
|---|---|
| **Devices** | État, version, CPU, mémoire, interfaces |
| **Tunnels** | État, qualité, trafic |
| **Statistics** | Débit, applications, perte/délai/gigue historiques |
| **Notifications** | Alertes et événements |
| **Logs** | Journaux des actions |

### 8.2 Sur l'équipement

```bash
sudo systemctl status flexiwan-router
sudo journalctl -u flexiwan-router -f
sudo vppctl show interface            # interfaces VPP et compteurs
sudo vppctl show ip fib               # table de transfert
sudo vppctl show ipsec sa             # associations IPsec
fwagent --help                        # sous-commandes de l'agent (états, configuration, diagnostic)
```

### 8.3 Problèmes courants

| Symptôme | Causes probables | Vérifications |
|---|---|---|
| Équipement jamais « Connected » | Jeton erroné, pas d'accès sortant HTTPS/WebSocket, horloge fausse | Journaux de l'agent, `curl` vers flexiManage |
| Routeur ne démarre pas | Interface réservée par le système, ressources insuffisantes | `journalctl`, mémoire, interfaces |
| Tunnel absent | Étiquettes incompatibles, NAT, pare-feu, ports | Vue des tunnels, capture |
| Tunnel up sans trafic | Routes manquantes, pare-feu LAN | `vppctl show ip fib`, ping |
| Qualité mauvaise | Lien d'underlay dégradé | Mesures de tunnel, `mtr` underlay |
| Politique ignorée | Application mal identifiée, règle d'ordre | Vérifier la classification et l'ordre |

---

## 9. Comparaison avec Cisco Catalyst SD-WAN

| Critère | FlexiWAN | Cisco Catalyst SD-WAN |
|---|---|---|
| **Licence** | Ouverte, services commerciaux optionnels | Commerciale |
| **Plan d'orchestration/authentification** | Jeton + approbation | Certificats + liste d'autorisation + Validator |
| **Plan de contrôle** | Centralisé dans flexiManage | OMP distribué via Controllers |
| **Concept de transport** | Étiquettes de chemin | Couleurs (TLOC) |
| **Tunnels** | Créés et gérés par flexiManage | Tunnels IPsec entre TLOC, clés via OMP |
| **Segmentation** | Plus limitée (segments logiques selon version) | VPN/VRF riches |
| **Politiques** | Multi-liens, QoS, pare-feu | Politiques centralisées très riches (contrôle, données, AAR) |
| **Sécurité intégrée** | Pare-feu de base | Pare-feu, IPS, URL, DNS, SIG |
| **Échelle** | Petites et moyennes déploiements | Très grandes fabriques |
| **Écosystème** | Communauté, intégrateurs | Large (support, partenaires, formation) |
| **Courbe d'apprentissage** | Moyenne (Linux) | Élevée |
| **Usage idéal** | Laboratoire, PME, fournisseurs de services | Entreprises moyennes à très grandes |

---

## 10. Cas d'usage et limites

| Cas d'usage | Adapté ? |
|---|---|
| **Laboratoire d'apprentissage** | Excellent |
| **PME de 10 à 50 sites avec compétences Linux** | Oui, avec support et prudence |
| **Fournisseur de services gérés** | Oui (multi-organisations) |
| **Site isolé à faible coût** | Oui |
| **Grande entreprise régulée avec exigences de sécurité intégrées complexes** | Peu |
| **Besoin de segmentation avancée et d'intégration DC/cloud étendue** | Limité |

**Limites** : fonctions de sécurité moins riches, écosystème et support plus réduits, besoin de compétences Linux et VPP, évolution rapide de la plateforme (validez la version).

---

## 11. Lab guidé CP04

### 11.1 Objectif

Mettre en place un SD-WAN à trois sites avec FlexiWAN, démontrer le multi-liens, la bascule et une politique de chemin, et mesurer la qualité.

### 11.2 Étapes

| # | Action | Vérification |
|---|---|---|
| 1 | Préparer 3 VM Ubuntu et un réseau de transport simulé (2 chemins) | Ping entre les WAN |
| 2 | Créer l'organisation et le jeton dans flexiManage | Jeton |
| 3 | Installer l'agent sur les 3 VM et les enregistrer | Équipements « Unapproved » |
| 4 | Approuver, configurer interfaces WAN/LAN, étiquettes INTERNET et LTE | Interfaces actives |
| 5 | Démarrer les routeurs | « Running » |
| 6 | Créer les tunnels (6) | Tunnels « up », qualité visible |
| 7 | Ping et `iperf3` entre LAN A et LAN B | Connectivité |
| 8 | Dégrader l'underlay INTERNET (`tc netem delay 150ms loss 5%`) | Mesures de tunnel dégradées |
| 9 | Créer une politique « VOIX » préférant INTERNET puis LTE | Politique active |
| 10 | Observer la bascule de la voix sur LTE ; mesurer le temps | Chronométrage |
| 11 | Couper totalement INTERNET sur un site | Bascule du trafic par défaut |
| 12 | Ajouter une règle de pare-feu et vérifier son effet | Test |
| 13 | Comparer avec le scénario équivalent Cisco (tableau) | Document |

### 11.3 Script de mesure

```bash
#!/usr/bin/env bash
# mesure-flexiwan.sh : délai/perte/débit entre LAN A et LAN B pendant les scénarios
CIBLE=${1:-192.168.20.10}
echo "horodatage;perte_pct;rtt_moy_ms"
while true; do
  R=$(ping -c 10 -i 0.2 -q "$CIBLE" | tail -2)
  PERTE=$(echo "$R" | grep -oP '\d+(\.\d+)?(?=% packet loss)')
  RTT=$(echo "$R" | grep -oP 'rtt min/avg/max/mdev = [^/]+/\K[^/]+')
  echo "$(date +%T);${PERTE:-100};${RTT:-NA}"
  sleep 5
done
```

### 11.4 Grille

| Critère | Points |
|---|---|
| Mise en place (3 sites, tunnels up) | 5 |
| Politique de chemin et bascule démontrée | 5 |
| Mesures et analyse | 4 |
| Dépannage d'une panne injectée | 3 |
| Comparaison argumentée avec Cisco | 3 |

---

## 12. Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **flexiManage** | Plateforme de gestion de FlexiWAN |
| **flexiEdge** | Agent/routeur FlexiWAN |
| **VPP** | Vector Packet Processing : moteur de transfert logiciel haute performance |
| **Path label** | Étiquette d'un type de lien (équivalent des couleurs) |
| **Jeton (token)** | Secret d'enregistrement d'un équipement |
| **Multi-link policy** | Politique de sélection de lien selon l'application |
| **Approbation** | Validation d'un équipement enregistré |
| **Open source** | Logiciel dont le code est consultable et modifiable |
| **DPDK** | Bibliothèque d'accélération réseau utilisée avec VPP |
| **MSP** | Prestataire de services gérés |
| **NFV / VNF** | Virtualisation des fonctions réseau |
| **Self-hosted** | Auto-hébergé |

### B. Cas d'usage détaillés

#### Cas 1. Laboratoire de formation d'une école

Une école de réseaux déploie FlexiWAN sur des VM pour que chaque étudiant monte son propre SD-WAN à 3 sites, sans licence. Gains : liberté d'expérimentation, injection de pannes, lecture des journaux. **Leçon** : un SD-WAN ouvert est un **excellent outil pédagogique** ; il prépare à comprendre les produits commerciaux.

#### Cas 2. PME de 20 agences avec un administrateur Linux

L'entreprise remplace des VPN IPsec manuels par FlexiWAN sur des mini-PC. Bénéfices : tunnels automatiques, bascule automatique entre fibre et 4G, supervision centralisée. Précautions : un second administrateur formé, supervision des mises à jour, sauvegardes de flexiManage. **Leçon** : l'**absence d'éditeur « à appeler »** impose une rigueur d'exploitation interne.

#### Cas 3. Intégrateur local

Un intégrateur propose du SD-WAN géré à des artisans et commerces : multi-organisations dans flexiManage, mini-PC fournis. **Leçon** : le modèle économique du fournisseur de services s'appuie sur l'**automatisation** et la **standardisation**.

#### Cas 4. Un site de chantier en 4G + satellite

Mini-PC flexiEdge avec routeur 4G et terminal satellite, politiques de chemin préférant la 4G, satellite en secours pour les applications critiques. **Leçon** : la flexibilité du matériel (machine Linux générique) est un avantage pour les sites **atypiques**.

#### Cas 5. Preuve de concept avant un achat

Une entreprise valide ses scénarios (bascule, qualité par lien, flux) avec FlexiWAN en laboratoire avant de lancer un appel d'offres : cela précise ses exigences. **Leçon** : un **SD-WAN de laboratoire** réduit le risque de cahier des charges mal conçu.

#### Cas 6. Quand FlexiWAN ne suffit pas

Une banque regarde FlexiWAN pour 80 agences, mais exige segmentation avancée, preuves d'audit, intégration SIEM complète et support 24 h : la solution est écartée au profit d'un produit commercial. **Leçon** : le choix dépend des **exigences de conformité et de support**, pas de la technologie seule.

### C. Retours d'expérience

1. **Un lab FlexiWAN en une journée** vaut dix présentations commerciales pour comprendre les concepts.
2. **Pour un usage réel**, vérifiez la maturité de la version, la communauté et le support.
3. **Documentez tout** : l'ouverture implique une responsabilité d'intégration plus grande.
4. **Sauvegardez flexiManage** (si auto-hébergé) comme n'importe quel contrôleur.
5. **Testez les mises à jour de l'agent en lab** avant la production.
6. **Surveillez les ressources des mini-PC** : VPP est gourmand en CPU et mémoire si mal dimensionné.
7. **Ne confondez pas « open source » et « gratuit »** : le coût se déplace vers le temps et les compétences.
8. **Utilisez-le pour enseigner** la différence entre plans : approuver, configurer, mesurer, appliquer.

### D. Exercices supplémentaires

1. Calculez le nombre de tunnels pour 8 sites à 2 liens avec étiquettes appariées.
2. Rédigez le tableau de correspondance des termes FlexiWAN ↔ Cisco (au moins 8 lignes).
3. Proposez une politique de chemin pour 3 classes (voix, ERP, sauvegarde) avec repli.
4. Rédigez la check-list de passage d'un lab FlexiWAN à une production pour 15 sites.

<details>
<summary>Corrigés</summary>

1. `8×7/2 × 2 = 56` tunnels.
2. flexiManage ↔ Manager ; flexiEdge ↔ WAN Edge ; path label ↔ couleur ; jeton + approbation ↔ certificat + liste d'autorisation ; tunnel ↔ tunnel IPsec entre TLOC ; multi-link policy ↔ politique de routage applicatif ; VPP ↔ moteur de transfert ; notifications ↔ alarmes.
4. Version stable validée, sauvegardes, supervision, documentation, formation de deux personnes, plan de support, tests de bascule, sécurité du gestionnaire (MFA, accès restreint), plan de mise à jour, plan de sortie.

</details>

### E. Mini-projet (8 heures)

Montez un SD-WAN FlexiWAN à **4 sites** (dont un site 4G simulé), documentez l'architecture, écrivez 3 politiques, injectez 3 pannes, mesurez les temps de bascule, et produisez une **note comparative d'une page** « FlexiWAN ou solution commerciale pour une PME de 25 sites ».

---

## 13. Quiz

1. Quels sont les deux composants principaux de FlexiWAN ?
2. À quoi correspond une étiquette de chemin dans Cisco ?
3. Quel moteur de transfert utilise flexiEdge ?
4. Pourquoi approuver un équipement ?
5. Citez deux limites de FlexiWAN face à un produit commercial.
6. Dans quel cas FlexiWAN est-il particulièrement adapté ?

<details>
<summary>Réponses</summary>

1. flexiManage et flexiEdge. 2. À une couleur. 3. VPP. 4. Pour contrôler quels équipements rejoignent l'organisation (équivalent de la liste d'autorisation). 5. Sécurité intégrée moins riche, segmentation et écosystème/support moins étendus. 6. Laboratoire, formation, PME avec compétences Linux, fournisseurs de services, sites atypiques.

</details>

---

## 14. Aide-mémoire

```text
FlexiWAN = flexiManage (gestion SaaS/auto-hébergé) + flexiEdge (agent Linux + VPP) ; open source
Flux : organisation + jeton -> installer l'agent -> approuver -> interfaces (WAN/LAN) + path labels -> démarrer -> tunnels -> politiques -> supervision
Équivalences : path label = couleur ; approbation = liste d'autorisation ; multi-link policy = AAR ; tunnels gérés par flexiManage
Dépannage : journalctl -u flexiwan-router | vppctl show interface/ip fib/ipsec sa | fwagent --help
Usage : labo, formation, PME, MSP, sites atypiques ; moins adapté aux exigences fortes de sécurité/conformité
```

## 15. Pour aller plus loin

- Documentation FlexiWAN : <https://docs.flexiwan.com>
- Code et communauté : <https://github.com/flexiWAN>
- VPP : <https://fd.io>

> Module précédent : [CP03](../CP03-ipsec-en-profondeur/CP03-ipsec-en-profondeur.md) | Module suivant : [CP05](../CP05-telemetrie-observabilite/CP05-telemetrie-observabilite.md)
