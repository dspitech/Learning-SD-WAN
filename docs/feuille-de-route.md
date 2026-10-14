# Feuille de route SD-WAN : concepts, Cisco Catalyst SD-WAN, SASE, conception et migration

> **Parcours complet** pour comprendre, concevoir, déployer, sécuriser et exploiter un réseau étendu défini par logiciel : principes du SD-WAN, **Cisco Catalyst SD-WAN** en profondeur (préparation à la concentration CCNP **ENSDWI 300-415**), panorama des autres solutions (Meraki, Fortinet, Palo Alto, Aruba EdgeConnect, VeloCloud), **SASE et SSE**, conception, migration depuis MPLS, automatisation et exploitation, avec labs, projets et préparation aux certifications.
> Ce document est une **feuille de route** accompagnée d'un **aide-mémoire** : il liste et décrit les modules, il ne les développe pas. Chaque module fera l'objet d'un support dédié (cours, démonstrations, labs corrigés).
> Il prolonge les feuilles de route CCNA 3 (module WAN) et CCNP (concentration ENSDWI), et s'applique à un homelab : un tunnel WireGuard entre deux sites est un premier « overlay » à comprendre avant de passer aux solutions commerciales.

> **Actualité et vocabulaire à connaître (vérifié en octobre 2026, à revérifier avant de figer un cours)** :
> - Les composants de **Cisco Catalyst SD-WAN** ont été renommés : **SD-WAN Manager** (anciennement vManage), **SD-WAN Controller** (anciennement vSmart), **SD-WAN Validator** (anciennement vBond). Les équipements de bordure s'appellent **WAN Edge** (routeurs Catalyst 8000 en IOS XE SD-WAN ; les anciens vEdge sont en fin de vie). Les supports et les commandes anciens utilisent encore les anciens noms.
> - **VeloCloud**, longtemps rattaché à VMware puis à Broadcom, a été **racheté par Arista en juillet 2025** et se nomme désormais Arista VeloCloud.
> - Le marché du SD-WAN converge avec la **sécurité dans le cloud (SASE et SSE)** : la plupart des éditeurs (Cisco, Fortinet, Palo Alto Networks, HPE Aruba Networking, Arista, Versa) proposent une offre SD-WAN et SASE combinée. Les rachats et regroupements sont fréquents : vérifier l'état du marché au moment de choisir.
> - Le plan d'examen de la concentration CCNP **ENSDWI 300-415** est publié par Cisco : vérifier la version en vigueur.

---

## Table des matières

1. [Objectifs et public cible](#1-objectifs-et-public-cible)
2. [Prérequis](#2-prérequis)
3. [Place du parcours dans le cursus](#3-place-du-parcours-dans-le-cursus)
4. [Organisation du parcours](#4-organisation-du-parcours)
5. [Vue d'ensemble des modules](#5-vue-densemble-des-modules)
6. [Partie A : Fondamentaux du SD-WAN (SW01 à SW03)](#6-partie-a--fondamentaux-du-sd-wan-sw01-à-sw03)
7. [Partie B : Cisco Catalyst SD-WAN (SW04 à SW10)](#7-partie-b--cisco-catalyst-sd-wan-sw04-à-sw10)
8. [Partie C : Autres solutions, SASE et SSE (SW11 et SW12)](#8-partie-c--autres-solutions-sase-et-sse-sw11-et-sw12)
9. [Partie D : Conception, migration, automatisation et exploitation (SW13 à SW18)](#9-partie-d--conception-migration-automatisation-et-exploitation-sw13-à-sw18)
10. [Aide-mémoire pratique : Catalyst SD-WAN et dépannage](#10-aide-mémoire-pratique--catalyst-sd-wan-et-dépannage)
11. [Modules complémentaires (hors programme)](#11-modules-complémentaires-hors-programme)
12. [Environnement de laboratoire](#12-environnement-de-laboratoire)
13. [Labs transverses](#13-labs-transverses)
14. [Projets fil rouge](#14-projets-fil-rouge)
15. [Projets de synthèse](#15-projets-de-synthèse)
16. [Évaluations et certifications](#16-évaluations-et-certifications)
17. [Correspondance avec les référentiels](#17-correspondance-avec-les-référentiels)
18. [Guide pédagogique (pour l'enseignant)](#18-guide-pédagogique-pour-lenseignant)
19. [Structure recommandée du dépôt](#19-structure-recommandée-du-dépôt)
20. [Ressources](#20-ressources)
21. [Suivi de progression](#21-suivi-de-progression)

---

## 1. Objectifs et public cible

### Objectifs

À l'issue du parcours, l'apprenant sera capable de :

- Expliquer pourquoi le **SD-WAN** est apparu (coût et rigidité de MPLS, applications dans le cloud, accès Internet direct des sites) et ce qu'il apporte : transport indépendant, politiques centralisées, choix de chemin selon l'application, déploiement simplifié.
- Décrire l'architecture générique d'un SD-WAN : **plan d'orchestration, de gestion, de contrôle et de données**, **overlay** et **underlay**.
- Distinguer les transports (MPLS, accès Internet dédié, 4G et 5G, satellite) et leurs effets sur les performances et la disponibilité.
- Décrire les mécanismes clés : **tunnels chiffrés**, **mesure de qualité de chemin (BFD, sondes)**, **routage selon l'application**, correction d'erreurs, duplication de paquets, segmentation, QoS.
- Déployer et configurer **Cisco Catalyst SD-WAN** : contrôleurs, certificats, intégration des équipements, **modèles et groupes de configuration**, **politiques centralisées et localisées**, segmentation par VPN, accès direct à Internet.
- Mettre en œuvre la sécurité intégrée (pare-feu, IPS, filtrage d'URL, protection DNS, accès sécurisé à Internet) et connecter les sites aux clouds publics et aux applications SaaS (**Cloud OnRamp**).
- Superviser, dépanner et exploiter un réseau SD-WAN avec une méthode structurée, en commençant par le plan de contrôle.
- Comparer les principales solutions du marché et choisir selon des critères explicites.
- Situer le **SASE** et le **SSE** (passerelle web sécurisée, ZTNA, CASB, pare-feu en tant que service) et leur intégration avec le SD-WAN.
- **Concevoir** une architecture SD-WAN (topologie, haute disponibilité, adressage, segmentation, sécurité, dimensionnement) et **planifier la migration** depuis un réseau MPLS.
- **Automatiser** le SD-WAN par API (REST, Python, Terraform, Ansible) et l'intégrer à une chaîne CI/CD.

### Public cible

| Profil | Remarque |
|---|---|
| Étudiant en réseaux ou cybersécurité | Public principal |
| Ingénieur réseau d'entreprise | Passage d'un WAN MPLS et routé à un WAN piloté par politiques |
| Candidat CCNP Enterprise (concentration ENSDWI) | Préparation ciblée à l'examen 300-415 |
| Architecte réseau ou avant-vente | Choix, conception et justification d'une solution |
| Ingénieur sécurité | Intégration du SD-WAN et du SASE à une stratégie Zero Trust |

---

## 2. Prérequis

**Connaissances**

- Réseaux : modèles OSI et TCP/IP, adressage, VLAN, routage statique et dynamique (**OSPF**, **BGP**), NAT, ACL (niveau CCNA 1 à 3).
- WAN : MPLS, VPN, IPsec et GRE (voir les modules WAN et VPN du CCNA 3).
- Qualité de service : classification, marquage DSCP, files d'attente.
- Sécurité : chiffrement, certificats et PKI, pare-feu (voir la feuille de route cryptographie).
- Notions d'API REST et de Python (utiles à partir de la partie D).

**Matériel et logiciels**

- Un poste avec **32 Go de RAM** conseillés pour un laboratoire de simulation (contrôleurs et équipements virtuels gourmands).
- Accès à un laboratoire : **Cisco Modeling Labs (CML)** avec images SD-WAN, bac à sable **Cisco DevNet** (Catalyst SD-WAN), EVE-NG ou GNS3 selon les licences.
- Pour la comparaison : machines virtuelles de test d'autres éditeurs, **FlexiWAN** (SD-WAN open source), WireGuard et FRRouting pour un overlay « fait maison ».
- Voir la section 12 pour la liste détaillée.

---

## 3. Place du parcours dans le cursus

| Étape | Contenu | Lien avec ce parcours |
|---|---|---|
| CCNA 3 | Concepts WAN, VPN, IPsec, QoS | Introduction aux technologies WAN et aux tunnels |
| CCNP Enterprise : ENCOR | Architecture, SD-WAN et SD-Access (vue d'ensemble), virtualisation, QoS | Bases de l'architecture SD-WAN |
| CCNP Enterprise : concentration ENSDWI | Déploiement de Catalyst SD-WAN | Parties B et D de ce parcours |
| Cryptographie | IPsec, TLS, PKI | Sécurité des tunnels et des contrôleurs |
| Ports réseau | Filtrage, NAT, dépannage | Ouvertures nécessaires aux contrôleurs et aux tunnels |
| CI/CD et GitOps | Pipelines, API | Automatisation du SD-WAN (SW15) |
| **SD-WAN (ce parcours)** | **Concepts, solutions, conception, migration, exploitation** | **WAN d'entreprise moderne** |

---

## 4. Organisation du parcours

### Légende

| Terme | Signification |
|---|---|
| Sujets | Contenu du module |
| Compétences visées | Résultat attendu à la fin du module |
| Lab | Travaux pratiques (simulation, bac à sable, scripts) |
| Projet | Réalisation de plus grande ampleur combinant plusieurs modules |

### Rythme estimé (indicatif)

| Partie | Modules | Durée estimée |
|---|---|---|
| A. Fondamentaux du SD-WAN | SW01 à SW03 | 3 semaines |
| B. Cisco Catalyst SD-WAN | SW04 à SW10 | 8 à 10 semaines |
| C. Autres solutions, SASE et SSE | SW11, SW12 | 3 semaines |
| D. Conception, migration, automatisation et exploitation | SW13 à SW18 | 5 à 6 semaines |
| Compléments et révisions | Section 11 | 2 semaines |

Durée totale indicative : 21 à 24 semaines (à adapter selon l'accès au laboratoire et à l'expérience préalable en WAN).

### Principes directeurs

1. **Le transport est un fournisseur d'accès, pas une architecture** : l'overlay doit fonctionner sur n'importe quel transport.
2. **La politique est centralisée, l'exécution est distribuée** : on décrit l'intention une fois, les équipements l'appliquent localement.
3. **Les applications décident du chemin** : on choisit le meilleur chemin selon la qualité mesurée, pas seulement selon le coût de la route.
4. **La sécurité est intégrée au design** : chiffrement des tunnels, segmentation, accès Internet local protégé.
5. **Commencer par le plan de contrôle** : quand un site ne fonctionne pas, vérifier d'abord l'authentification et les connexions aux contrôleurs.
6. **Migrer par vagues** : pilote, coexistence avec l'ancien WAN, retour arrière prévu.
7. **Automatiser** : modèles, politiques et inventaire gérés comme du code.

---

## 5. Vue d'ensemble des modules

| # | Module | Partie |
|---|---|---|
| SW01 | Du WAN traditionnel au SD-WAN | A |
| SW02 | Transports, overlay et underlay | A |
| SW03 | Fonctions clés du SD-WAN | A |
| SW04 | Catalyst SD-WAN : architecture et composants | B |
| SW05 | Déploiement des contrôleurs et intégration des équipements | B |
| SW06 | Modèles, groupes de configuration et segmentation | B |
| SW07 | Politiques centralisées et localisées | B |
| SW08 | Sécurité intégrée | B |
| SW09 | Cloud OnRamp et accès aux applications SaaS et IaaS | B |
| SW10 | Supervision, exploitation et dépannage | B |
| SW11 | Panorama des solutions et critères de choix | C |
| SW12 | SASE et SSE | C |
| SW13 | Conception d'une architecture SD-WAN | D |
| SW14 | Migration depuis un réseau MPLS | D |
| SW15 | Automatisation et API | D |
| SW16 | Intégration avec le LAN, le centre de données et les usages métier | D |
| SW17 | Sécurité opérationnelle et conformité | D |
| SW18 | Indicateurs, SLA et amélioration continue | D |

---

## 6. Partie A : Fondamentaux du SD-WAN (SW01 à SW03)

### SW01. Du WAN traditionnel au SD-WAN

- **Sujets** :
  - **WAN traditionnel** : sites reliés par MPLS et lignes dédiées, topologie concentrateur et rayons, trafic Internet ramené par le siège, configuration équipement par équipement.
  - **Limites** : coût de la bande passante MPLS, délais de déploiement, faible agilité, expérience dégradée pour les applications SaaS et cloud, administration complexe.
  - **Moteurs du SD-WAN** : applications dans le cloud, sites distants nombreux, accès Internet bon marché, exigences d'agilité et de sécurité.
  - **Définition** : séparation du **plan de contrôle** (décisions) et du **plan de données** (transfert), politiques centralisées, **overlay** indépendant du transport.
  - **Architecture générique** : plan d'orchestration (admission des équipements), plan de gestion (configuration, supervision), plan de contrôle (échange de routes et de politiques), plan de données (tunnels et transfert).
  - **Bénéfices et limites** : réduction de coûts, performance applicative, déploiement sans intervention, mais complexité de transition, dépendance à l'éditeur, qualité variable des accès Internet.
  - Rappel : **SD-WAN n'est pas MPLS** ni un VPN classique : il en est une couche de pilotage.
- **Compétences visées** : expliquer le besoin, décrire l'architecture générique, comparer un WAN MPLS classique et un SD-WAN.
- **Lab** : schémas comparatifs d'un WAN MPLS et d'un SD-WAN pour un même cahier des charges, liste des gains et des risques.

### SW02. Transports, overlay et underlay

- **Sujets** :
  - **Underlay** : **MPLS**, accès Internet professionnel (DIA), haut débit grand public, **4G et 5G**, **satellite** (orbite basse), Metro Ethernet ; débit, délai, gigue, perte, disponibilité, coût, délais de déploiement.
  - **Overlay** : tunnels **IPsec**, GRE, VXLAN selon les éditeurs ; chiffrement, authentification, intégrité.
  - **Traversée de NAT** : NAT-T, types de NAT (cône, symétrique) et effets sur l'établissement de tunnels.
  - **MTU et MSS** : surcoût d'encapsulation, fragmentation, ajustement du MSS TCP.
  - **Mesure de qualité** : **BFD** (RFC 5880) et sondes applicatives pour mesurer perte, délai et gigue en continu.
  - **Haute disponibilité des liens** : double accès par site, opérateurs distincts, 4G ou 5G en secours, bascule.
  - Interconnexion avec l'existant : routage avec les opérateurs (**BGP**), coexistence avec MPLS.
- **Compétences visées** : choisir des transports adaptés à un site, prévoir l'effet d'un NAT et de la MTU sur un tunnel.
- **Lab** : mise en place d'un overlay IPsec ou WireGuard entre deux sites, mesure du délai et de la perte, effet d'une réduction de MTU.

### SW03. Fonctions clés du SD-WAN

- **Sujets** :
  - **Routage selon l'application** (application-aware routing) : classification des applications (inspection de paquets, signatures), **classes de SLA** (perte, délai, gigue), choix du chemin conforme.
  - **Sélection dynamique de chemin** et répartition de charge (par flux, par paquet), bascule rapide en cas de dégradation.
  - **Techniques de fiabilisation** : correction d'erreurs par anticipation (FEC), **duplication de paquets**, optimisation de TCP (selon les éditeurs).
  - **Segmentation** : séparation des flux en segments (VPN) de bout en bout.
  - **QoS** : marquage, files d'attente, lissage à la sortie WAN (rappel du module QoS).
  - **Déploiement sans intervention** (ZTP) : l'équipement se connecte, s'authentifie et reçoit sa configuration.
  - **Politiques centralisées** : une intention décrite une fois, appliquée à tous les sites.
  - **Analytique** : visibilité sur les applications, les chemins et les performances.
  - **Accès direct à Internet** (DIA, local breakout) et sécurité associée.
  - Optimisation WAN : compression, déduplication, mise en cache (selon les solutions).
- **Compétences visées** : décrire chaque fonction et le problème qu'elle résout, associer une fonction à un besoin métier.
- **Lab** : étude de scénarios (visioconférence dégradée, SaaS lent, panne d'un accès) et fonction du SD-WAN appropriée.

---

## 7. Partie B : Cisco Catalyst SD-WAN (SW04 à SW10)

### SW04. Catalyst SD-WAN : architecture et composants

- **Sujets** :
  - **Plans** : orchestration (**SD-WAN Validator**), gestion (**SD-WAN Manager**), contrôle (**SD-WAN Controller**), données (**WAN Edge**) ; correspondance avec les anciens noms vBond, vManage, vSmart.
  - **Équipements de bordure** : plateformes **Catalyst 8000** (matérielles et virtuelles) en IOS XE SD-WAN ; anciens vEdge (fin de vie) ; critères de choix (débit chiffré, fonctions de sécurité).
  - **Protocole OMP** (Overlay Management Protocol) : échange des routes, des TLOC, des services et des politiques entre les équipements et les contrôleurs.
  - **TLOC** (point de terminaison de transport) : identifié par l'adresse système, la **couleur** et l'encapsulation ; couleurs publiques et restreintes (mpls, biz-internet, public-internet, lte, etc.).
  - **Tunnels de données** : IPsec entre TLOC, **BFD** pour la vivacité et les mesures de qualité.
  - **Identifiants** : adresse système (system IP), identifiant de site, nom d'organisation, certificats.
  - **Connexions de contrôle** : DTLS ou TLS entre les équipements et les contrôleurs, ports usuels (UDP 12346 par défaut avec décalages possibles, TCP 23456 pour TLS).
  - **VPN et segmentation** : VPN 0 (transport), VPN 512 (gestion), VPN de service ; correspondance avec les VRF.
  - **Modèles de déploiement** : contrôleurs hébergés dans le cloud par Cisco ou auto-hébergés ; haute disponibilité des contrôleurs.
- **Compétences visées** : décrire le rôle de chaque composant et le chemin d'établissement d'un site (orchestration, contrôle, données).
- **Lab** : schéma annoté d'un site qui rejoint la fabrique (Validator, Manager, Controller, tunnels), exploration de l'interface du bac à sable DevNet.

### SW05. Déploiement des contrôleurs et intégration des équipements

- **Sujets** :
  - **Déploiement des contrôleurs** : tailles, ressources, adresses, ports ouverts, haute disponibilité, **cluster** de gestionnaires.
  - **PKI et certificats** : autorité de certification (Cisco ou d'entreprise), génération et installation des certificats, durée de validité, renouvellement ; **liste des équipements autorisés** (numéro de série, châssis).
  - **Synchronisation horaire** : NTP indispensable à la validité des certificats.
  - **Intégration des WAN Edge** : configuration minimale d'amorçage (adresse IP, VPN 0, interface de transport, adresse du Validator, adresse système, identifiant de site, organisation), **Plug and Play** et provisionnement sans intervention, intégration manuelle.
  - **Mises à jour logicielles** : ordre de mise à niveau (contrôleurs puis équipements), compatibilité de versions, retour arrière.
  - **Sauvegardes** de la configuration et du plan de gestion.
- **Compétences visées** : déployer les contrôleurs, installer les certificats et intégrer un premier équipement.
- **Lab** : mise en place d'un environnement de laboratoire (contrôleurs et deux sites), installation des certificats, intégration d'un WAN Edge avec connexions de contrôle actives.

### SW06. Modèles, groupes de configuration et segmentation

- **Sujets** :
  - **Modèles de fonctionnalité** et **modèles d'équipement** : variables, héritage, modèles CLI complémentaires.
  - **Groupes de configuration et groupes de politiques** (approche récente) : profils, variables par équipement, déploiement par site ; comparaison avec l'approche par modèles.
  - **Segmentation par VPN** : service-side VPN, correspondance avec des VRF, isolation des flux.
  - **Routage côté service** : OSPF, BGP, EIGRP, routes statiques, redistribution entre OMP et les protocoles du réseau local, prévention des boucles.
  - **Accès direct à Internet** : NAT local, itinéraires par défaut, choix de la sortie.
  - **Haute disponibilité côté LAN** : VRRP, double équipement de bordure.
  - **Interfaces de transport** : couleurs, restrictions, bande passante, lissage, tunnels de secours.
  - Gestion des **variables** et des fichiers de paramètres, nommage cohérent.
- **Compétences visées** : configurer un site complet par modèle ou groupe de configuration, avec segmentation et routage vers le LAN.
- **Lab** : déploiement de trois sites avec deux segments, OSPF côté LAN, accès direct à Internet pour un segment, VRRP sur un site.

### SW07. Politiques centralisées et localisées

- **Sujets** :
  - **Politiques centralisées**, exécutées sur le SD-WAN Controller : **politique de contrôle** (topologie, filtrage de routes, préférence de TLOC), **politique de données** (règles de transfert selon l'adresse, l'application), **politique de routage applicatif** (classes de SLA et chemins préférés).
  - **Topologies** : maillage complet, concentrateur et rayons, maillage partiel, **hubs régionaux**, multi-régions ; fonctions de chaînage de services.
  - **Classes de SLA** : seuils de perte, délai et gigue (par exemple voix : perte inférieure à 1 %, délai à sens unique inférieur à 150 ms, gigue inférieure à 30 ms), choix des transports de repli.
  - **Politiques localisées**, exécutées sur chaque équipement : **QoS** locale, ACL, politiques de routage, mise en miroir.
  - **Visibilité applicative** : classification (inspection de paquets), listes d'applications.
  - **Ordre et application des politiques** : sens (entrant, sortant), sites et VPN ciblés, vérification de l'effet.
  - Dépannage : politique non reçue, correspondance incorrecte, effets de bord sur la topologie.
- **Compétences visées** : concevoir et appliquer des politiques de topologie et de chemin, vérifier leur effet.
- **Lab** : topologie concentrateur et rayons par politique de contrôle, routage applicatif de la voix vers l'accès le plus performant, politique de QoS localisée.

### SW08. Sécurité intégrée

- **Sujets** :
  - **Chiffrement des tunnels** : IPsec, gestion des clés, rotation, algorithmes.
  - **Segmentation** comme premier niveau de sécurité.
  - **Fonctions de sécurité sur le WAN Edge** : pare-feu d'entreprise avec reconnaissance des applications, **IPS et IDS**, **filtrage d'URL**, protection contre les logiciels malveillants (AMP), protection DNS, proxy TLS (selon les cas).
  - **Accès sécurisé à Internet** : redirection du trafic vers une **passerelle web sécurisée dans le cloud** (tunnels vers un service SIG ou SSE), intégration à une solution de sécurité cloud.
  - **Principes Zero Trust** appliqués : identités, segmentation, vérification continue.
  - **Sécurité du plan de gestion** : authentification forte, RBAC, journalisation, accès restreint aux contrôleurs.
  - Compromis : fonctions de sécurité sur le site, dans le cloud ou hybrides, impact sur le débit.
- **Compétences visées** : choisir où appliquer la sécurité (site ou cloud), configurer les principales fonctions de sécurité.
- **Lab** : activation du pare-feu et du filtrage d'URL sur un site, redirection de l'accès Internet d'un segment vers un service de sécurité cloud (si disponible en laboratoire).

### SW09. Cloud OnRamp et accès aux applications SaaS et IaaS

- **Sujets** :
  - **Cloud OnRamp pour le SaaS** : mesure de qualité vers les applications SaaS (par exemple messagerie et collaboration), choix du meilleur chemin (accès direct à Internet ou via un concentrateur).
  - **Cloud OnRamp pour l'IaaS et le multicloud** : connexion de la fabrique SD-WAN aux clouds publics (AWS, Azure, Google Cloud), passerelles cloud, routage entre branches et cloud.
  - **Interconnexion et colocation** : points de présence régionaux, liens privés vers les clouds, concentrateurs régionaux.
  - **Optimisation des applications critiques** : visioconférence, voix, applications d'entreprise.
  - **Conception de la sortie Internet** : locale, régionale ou centralisée, avec sécurité adaptée.
  - **Coûts et performances** : trafic sortant des clouds, choix des régions.
- **Compétences visées** : connecter la fabrique SD-WAN à un cloud public et optimiser l'accès à une application SaaS.
- **Lab** : étude de conception de la connexion d'un cloud public à un SD-WAN, comparaison de trois options de sortie Internet pour une même application.

### SW10. Supervision, exploitation et dépannage

- **Sujets** :
  - **Supervision** dans le SD-WAN Manager : tableaux de bord, état des équipements, alarmes, événements, performances des tunnels, applications.
  - **Analytique et outils complémentaires** : analyse de performances, **Network-Wide Path Insight**, intégration avec des outils de supervision de l'expérience applicative (par exemple ThousandEyes).
  - **Exploitation** : cycle de vie logiciel, mises à jour, sauvegardes, sécurité des accès, gestion des changements.
  - **Méthode de dépannage** : (1) plan de contrôle : certificats, horloge, connexions aux contrôleurs ; (2) OMP : routes et TLOC reçus ; (3) plan de données : tunnels et BFD ; (4) politiques : reçues et appliquées ; (5) application : chemin réel du flux.
  - **Pannes fréquentes** : certificat invalide ou expiré, décalage d'horloge, ports bloqués par un pare-feu, NAT symétrique, couleur incorrecte (tunnels non établis entre couleurs restreintes), identifiants dupliqués, mauvaise MTU, politique non appliquée.
  - **Commandes de diagnostic** de l'IOS XE SD-WAN (voir section 10).
  - Documentation d'incident et escalade au support de l'éditeur.
- **Compétences visées** : localiser rapidement l'origine d'une panne en suivant la méthode par plan, utiliser les outils de supervision.
- **Lab** : série de pannes injectées (certificat, horloge, pare-feu, couleur, politique), diagnostic chronométré avec fiche d'incident.

---

## 8. Partie C : Autres solutions, SASE et SSE (SW11 et SW12)

### SW11. Panorama des solutions et critères de choix

- **Sujets** :
  - **Cisco** : Catalyst SD-WAN (entreprise, fonctions riches) et **Meraki** (gestion par le cloud, simplicité, VPN automatique) ; positionnement et cas d'usage.
  - **Fortinet Secure SD-WAN** : SD-WAN intégré aux pare-feu FortiGate, règles de sélection de chemin, tunnels dynamiques, sécurité native.
  - **Palo Alto Networks Prisma SD-WAN** : fabrique pilotée par les applications, intégration avec Prisma Access.
  - **HPE Aruba Networking EdgeConnect** (issu de Silver Peak) : routage et optimisation WAN, intégration avec la gamme Aruba ; contexte du rapprochement de HPE et de Juniper (solution Session Smart Router de Juniper).
  - **Arista VeloCloud** : passerelles dans le cloud, optimisation dynamique multi-chemins ; rachat en juillet 2025.
  - **Versa**, **Cato Networks**, **Aryaka** et services gérés par des opérateurs : plateformes SASE nativement cloud et SD-WAN en service géré.
  - **Solutions ouvertes** : **FlexiWAN** (SD-WAN open source), constructions maison avec WireGuard et FRRouting (usage pédagogique ou petites structures).
  - **Critères de choix** : cohérence avec l'existant (LAN, pare-feu, cloud), sécurité intégrée, simplicité d'exploitation, modèle de gestion (autonome, cloud, géré par un opérateur), couverture géographique, coûts et licences, écosystème d'API, feuille de route et pérennité de l'éditeur, souveraineté des données.
  - **Service géré ou autogéré** : avantages et risques de chaque modèle.
- **Compétences visées** : comparer des solutions selon une grille de critères pondérés, justifier un choix pour un cas donné.
- **Lab** : matrice de décision pondérée pour trois scénarios (PME multi-sites, entreprise internationale, réseau de commerces), avec recommandations argumentées.

### SW12. SASE et SSE

- **Sujets** :
  - **SASE** (Secure Access Service Edge) : convergence du réseau (SD-WAN) et de la sécurité fournie dans le cloud.
  - **SSE** (Security Service Edge) : la partie sécurité du SASE : **passerelle web sécurisée (SWG)**, **courtier de sécurité d'accès au cloud (CASB)**, **accès réseau Zero Trust (ZTNA)**, pare-feu en tant que service (FWaaS), prévention des fuites de données (DLP).
  - **ZTNA et VPN** : accès par application plutôt que par réseau, vérification de l'identité et de l'état du terminal.
  - **Modèles d'intégration** : SD-WAN d'un éditeur avec SSE du même éditeur ou d'un autre, tunnels vers le service, redirection du trafic.
  - **Architecture Zero Trust** (NIST SP 800-207) et place du SASE dans une stratégie de sécurité.
  - **Considérations** : latence, présence régionale des points d'accès, intégration des identités, gestion unifiée des politiques, couverture des utilisateurs distants.
  - Évolution : convergence des offres et rachats.
- **Compétences visées** : expliquer le SASE et le SSE, concevoir l'intégration d'un SD-WAN avec un service de sécurité cloud.
- **Lab** : schéma d'architecture SASE d'une entreprise avec sites, télétravailleurs et applications cloud, et choix argumenté entre pare-feu local et services cloud.

---

## 9. Partie D : Conception, migration, automatisation et exploitation (SW13 à SW18)

### SW13. Conception d'une architecture SD-WAN

- **Sujets** :
  - **Recueil des besoins** : sites, applications critiques, volumes, contraintes réglementaires, disponibilité attendue, compétences disponibles.
  - **Topologie** : maillage complet, concentrateur et rayons, **hubs régionaux**, accès cloud, rôle des centres de données et des points de colocation.
  - **Haute disponibilité** : double équipement par site, double accès, opérateurs distincts, redondance des contrôleurs.
  - **Plan d'adressage et de routage** : adresses système, plan IP des sites, résumé, routage côté LAN, intégration BGP avec les opérateurs.
  - **Plan de segmentation** : segments (utilisateurs, invités, voix, IoT et OT, serveurs), politiques entre segments.
  - **Sécurité** : fonctions sur le site ou dans le cloud, Zero Trust, gestion des accès aux contrôleurs.
  - **QoS et SLA** : classes de trafic, seuils, transports de repli.
  - **Dimensionnement** : débit chiffré, nombre de tunnels, fonctions activées, choix des plateformes, licences.
  - **Modèle de contrôleurs** : hébergé par l'éditeur ou auto-hébergé, emplacement, souveraineté.
  - **Conventions de nommage**, documentation, plan d'exploitation.
- **Compétences visées** : produire un dossier de conception complet et justifié à partir d'un cahier des charges.
- **Lab** : dossier de conception d'un réseau de 40 sites avec deux centres de données et un accès cloud, comprenant schémas, tableaux d'adressage, choix des plateformes et justification.

### SW14. Migration depuis un réseau MPLS

- **Sujets** :
  - **Évaluation de l'existant** : inventaire des sites, applications, liens, contrats opérateurs, politiques de routage et de sécurité.
  - **Analyse de coûts** (TCO) : coût des liens, des équipements, des licences, de l'exploitation, gains attendus.
  - **Stratégie de migration** : **pilote**, vagues successives, priorisation des sites, calendrier, fenêtres de maintenance.
  - **Coexistence** MPLS et SD-WAN : transport MPLS comme une couleur de l'overlay, routage entre les deux mondes, **prévention des boucles** et des routes asymétriques (balises, filtres).
  - **Intégration au routage existant** : BGP avec les opérateurs, redistribution contrôlée, résumé.
  - **Bascule et retour arrière** : critères de réussite, plan de retour, tests avant et après.
  - **Préparation des équipes** : formation, procédures d'exploitation, support.
  - **Communication** avec les utilisateurs et les opérateurs.
- **Compétences visées** : planifier une migration par vagues avec coexistence, gérer le routage entre l'ancien et le nouveau réseau, préparer un retour arrière.
- **Lab** : plan de migration d'un réseau fictif de 25 sites avec calendrier, matrice de risques et plan de retour arrière ; simulation d'une coexistence MPLS et SD-WAN avec redistribution contrôlée.

### SW15. Automatisation et API

- **Sujets** :
  - **API REST** du SD-WAN Manager : authentification (session et jeton anti-falsification de requête selon la version), inventaire des équipements, état des tunnels, modèles et politiques ; documentation interactive.
  - **Python** : requêtes HTTP, bibliothèques et kits de développement (par exemple la bibliothèque Python pour Catalyst SD-WAN), scripts d'inventaire et de rapport.
  - **Infrastructure as code** : fournisseur **Terraform** et collections **Ansible** pour Catalyst SD-WAN (vérifier la disponibilité et les versions), gestion des modèles et des politiques dans Git.
  - **CI/CD** : revue des changements par demande de fusion, validation, déploiement contrôlé, retour arrière (voir la feuille de route CI/CD et GitOps).
  - **Cas d'usage** : intégration automatique d'un site, génération de rapports de conformité, extraction de mesures vers un tableau de bord, alertes.
  - **Bonnes pratiques** : comptes de service au moindre privilège, secrets hors du code, vérification des certificats TLS, limitation de débit.
- **Compétences visées** : automatiser des tâches d'inventaire et de supervision par API, gérer la configuration comme du code.
- **Lab** : script Python listant les équipements et l'état de leurs tunnels, extraction des mesures de qualité vers un tableau de bord, dépôt Git de modèles avec validation automatique.

### SW16. Intégration avec le LAN, le centre de données et les usages métier

- **Sujets** :
  - **Campus et succursales** : intégration avec les commutateurs et les points d'accès, segmentation de bout en bout, relation avec **SD-Access** (association des étiquettes de groupe de sécurité).
  - **Centres de données** : concentrateurs, résilience, accès aux applications hébergées.
  - **Voix et vidéo** : QoS de bout en bout, classes de SLA, bascule sans coupure.
  - **IoT et technologies opérationnelles** : segments isolés, politiques restrictives, passerelles.
  - **Sites distants et mobiles** : 4G et 5G en secours ou en principal, sites éphémères, véhicules.
  - **Télétravail** : options (équipement léger, accès ZTNA).
  - **Collaboration et applications SaaS** : optimisation du chemin.
  - Intégration avec la **supervision** et les journaux (SIEM) de l'entreprise.
- **Compétences visées** : intégrer le SD-WAN dans l'environnement existant (LAN, centre de données, usages métier) sans rupture.
- **Lab** : conception de l'intégration d'un site industriel (OT isolé, voix, 5G en secours) avec politiques entre segments.

### SW17. Sécurité opérationnelle et conformité

- **Sujets** :
  - **Durcissement des contrôleurs** : accès restreints, authentification multifacteur, RBAC, journalisation, correctifs.
  - **Cycle de vie des certificats** : suivi des échéances, renouvellement, révocation.
  - **Gestion des identifiants et des secrets** : comptes de service, rotation.
  - **Journalisation et SIEM** : export des journaux et événements, règles de détection (échecs d'authentification, changements de configuration, perte de tunnels), intégration avec une plateforme comme Wazuh.
  - **Conformité** : cartographie des exigences (par exemple isolation des flux de paiement), preuves d'audit, séparation des tâches.
  - **Réponse à incident** : isolement d'un site, révocation d'un équipement compromis, restauration.
  - **Vulnérabilités** : suivi des avis de sécurité de l'éditeur, plan de correction.
  - Sécurité de la chaîne d'approvisionnement du matériel et des logiciels.
- **Compétences visées** : exploiter le SD-WAN de façon sûre et auditable, détecter et traiter un incident touchant la fabrique.
- **Lab** : envoi des journaux du SD-WAN vers un SIEM de laboratoire, règles de détection, exercice de révocation d'un équipement compromis.

### SW18. Indicateurs, SLA et amélioration continue

- **Sujets** :
  - **Indicateurs** : disponibilité par site et par transport, taux de bascule, perte, délai et gigue par chemin, **conformité aux classes de SLA**, scores d'expérience applicative, débit et saturation.
  - **Capacité** : prévision de croissance, dimensionnement des liens et des équipements.
  - **Suivi des opérateurs** : engagements de service, pénalités, revues périodiques.
  - **Tableaux de bord** et rapports pour la direction et l'exploitation.
  - **Amélioration continue** : revue des politiques, des couleurs et des chemins, tests de bascule, retours d'expérience.
  - **Coûts** : suivi des liens, des licences, des économies réalisées.
  - Gestion du cycle de vie des équipements et des versions.
- **Compétences visées** : piloter un SD-WAN par les indicateurs, planifier la capacité et conduire des revues de performance.
- **Lab** : tableau de bord de conformité aux SLA et revue mensuelle simulée avec plan d'actions.

---

## 10. Aide-mémoire pratique : Catalyst SD-WAN et dépannage

Exemples à utiliser en laboratoire ou sur un environnement autorisé. Les commandes ci-dessous concernent l'**IOS XE SD-WAN** (WAN Edge Catalyst 8000) ; les anciens vEdge utilisent des commandes sans le préfixe `sdwan`.

**Commandes de diagnostic sur un WAN Edge**

```text
show sdwan control connections            # connexions DTLS ou TLS vers Controller, Manager et Validator
show sdwan control local-properties       # identifiants, certificat, couleur, ports
show sdwan certificate installed          # certificats installés et leur validité
show sdwan omp peers                      # sessions OMP avec les contrôleurs
show sdwan omp routes                     # routes reçues et annoncées
show sdwan omp tlocs                      # TLOC connus
show sdwan bfd sessions                   # tunnels de données et BFD (état, délai, perte, gigue)
show sdwan app-route stats                # statistiques de qualité par chemin
show sdwan policy from-vsmart             # politiques centralisées reçues du Controller
show sdwan running-config                 # configuration SD-WAN en vigueur
show ip route vrf <N>                     # table de routage d'un VPN de service
```

**Méthode de dépannage par plan (ordre conseillé)**

| Étape | Question | Vérification |
|---|---|---|
| 1 | L'équipement est-il authentifié et joint-il les contrôleurs ? | Connexions de contrôle, certificats, horloge, accès au Validator |
| 2 | Les routes et les TLOC sont-ils échangés ? | Sessions OMP, routes et TLOC reçus |
| 3 | Les tunnels de données sont-ils établis et sains ? | Sessions BFD, couleurs, NAT, pare-feu intermédiaires, MTU |
| 4 | Les politiques sont-elles reçues et appliquées ? | Politiques reçues, correspondance des listes de sites et de VPN |
| 5 | Le flux emprunte-t-il le chemin attendu ? | Trace du chemin, statistiques de routage applicatif, classification |

**Symptômes fréquents**

| Symptôme | Causes probables | Vérifications |
|---|---|---|
| Connexion de contrôle absente | Certificat invalide ou expiré, horloge décalée, port filtré, nom d'organisation incorrect | Certificat installé, NTP, ports DTLS et TLS, configuration d'amorçage |
| Tunnels de données absents | Couleurs restreintes non compatibles, NAT symétrique, pare-feu bloquant IPsec et UDP, routes de transport absentes | TLOC, couleurs, NAT, routes du VPN 0 |
| Tunnels instables | Perte ou gigue sur le transport, MTU, saturation du lien | Statistiques BFD, mesures de chemin, MTU et MSS |
| Application lente | Mauvaise classification, politique non appliquée, chemin non conforme au SLA | Routage applicatif, statistiques, QoS |
| Routes manquantes côté LAN | Redistribution absente, filtre de politique, VPN de service incorrect | Routes OMP, redistribution, politique de contrôle |
| Politique sans effet | Liste de sites ou de VPN incorrecte, sens d'application, politique non activée | Politique reçue du Controller, activation |

**API REST du SD-WAN Manager (Python, laboratoire uniquement)**

```python
import requests

base = "https://<ADRESSE_DU_MANAGER>"
s = requests.Session()
s.verify = False          # laboratoire seulement : en production, vérifier le certificat

# Ouverture de session puis jeton anti-falsification de requête (selon la version)
s.post(f"{base}/j_security_check",
       data={"j_username": "<UTILISATEUR>", "j_password": "<MOT_DE_PASSE>"})
s.headers["X-XSRF-TOKEN"] = s.get(f"{base}/dataservice/client/token").text

equipements = s.get(f"{base}/dataservice/device").json()["data"]
for e in equipements:
    print(e.get("host-name"), e.get("system-ip"), e.get("reachability"))
```

> Les chemins d'API et le mode d'authentification varient selon la version du logiciel : consulter la documentation de la version déployée. Ne jamais consigner d'identifiants dans un dépôt.

**Pour comprendre l'overlay avec un tunnel WireGuard (pédagogique)**

```ini
# Site A : /etc/wireguard/wg0.conf (le site B est symétrique)
[Interface]
Address = 10.200.0.1/24
PrivateKey = <CLE_PRIVEE_A>
ListenPort = 51820

[Peer]
PublicKey = <CLE_PUBLIQUE_B>
Endpoint = <ADRESSE_PUBLIQUE_B>:51820
AllowedIPs = 10.200.0.2/32, 192.168.20.0/24
PersistentKeepalive = 25
```

> Cet exemple illustre un overlay chiffré point à point. Il n'offre pas de plan de contrôle centralisé, de mesure de qualité de chemin ni de politiques applicatives : c'est précisément ce qu'ajoute une solution SD-WAN.

---

## 11. Modules complémentaires (hors programme)

### CP01. Routage avancé pour le SD-WAN
- BGP avec les opérateurs (communautés, préférence locale, filtrage), redistribution et prévention des boucles, résumé et équilibrage.
- **Lab** : coexistence de BGP avec deux opérateurs et un overlay, politique de préférence.

### CP02. Qualité de service en détail pour le WAN
- Modèles de files, lissage et contrôle de débit, marquage de bout en bout, traitement de la voix et de la vidéo.
- **Lab** : politique QoS complète avec mesures avant et après.

### CP03. Fonctionnement d'IPsec en profondeur
- IKEv2, associations de sécurité, mode tunnel et transport, chiffrement et intégrité, dépannage de tunnels.
- **Lab** : analyse d'un échange IKE et ESP avec Wireshark.

### CP04. FlexiWAN et SD-WAN ouvert
- Architecture, déploiement d'un orchestrateur et d'équipements, politiques de chemin.
- **Lab** : déploiement de trois sites avec un orchestrateur open source et comparaison avec une solution commerciale.

### CP05. Télémétrie et observabilité du WAN
- Collecte de mesures (SNMP, flux, API), tableaux de bord Grafana, alertes, corrélation avec les événements.
- **Lab** : tableau de bord du WAN avec indicateurs de qualité par chemin.

### CP06. Préparation aux certifications
- Questions à choix multiples, études de cas, simulations, fiches de révision (voir section 16).
- **Lab** : examen blanc chronométré.

---

## 12. Environnement de laboratoire

| Besoin | Outils proposés |
|---|---|
| Simulation Cisco | **Cisco Modeling Labs** (images SD-WAN selon licences), bac à sable **DevNet** pour Catalyst SD-WAN, EVE-NG ou GNS3 (selon licences) |
| Contrôleurs et équipements | SD-WAN Manager, Controller, Validator virtuels ; WAN Edge Catalyst 8000V |
| Simulation du transport | Routeurs virtuels (MPLS et Internet simulés), émulation de perte, de délai et de gigue (`tc netem` sous Linux) |
| Overlay maison et open source | WireGuard, FRRouting, **FlexiWAN** |
| Autres éditeurs | Versions d'évaluation ou machines virtuelles de test (Fortinet, Palo Alto, Aruba EdgeConnect, VeloCloud) selon disponibilité |
| Sécurité | Pare-feu virtuel, serveur de journaux, SIEM de laboratoire (Wazuh) |
| Automatisation | Python, Postman, Git, Terraform, Ansible |
| Supervision | Prometheus, Grafana |
| Analyse de trafic | Wireshark, `tcpdump`, `iperf3`, `mtr` |
| Poste | 32 Go de RAM conseillés, SSD rapide |

> Le simulateur ne remplace pas le matériel pour les performances de chiffrement : les mesures de débit en laboratoire ne sont pas représentatives d'une production.

---

## 13. Labs transverses

| Lab | Thème | Associé à |
|---|---|---|
| T1 | Overlay WireGuard entre deux sites avec mesure de qualité (délai, perte, MTU) | SW02, SW03 |
| T2 | Intégration d'un site : contrôleurs, certificats, WAN Edge, connexions de contrôle | SW04, SW05 |
| T3 | Trois sites avec modèles ou groupes de configuration, deux segments et OSPF côté LAN | SW06 |
| T4 | Topologie concentrateur et rayons par politique de contrôle, puis maillage partiel | SW07 |
| T5 | Routage applicatif de la voix avec classes de SLA et dégradation injectée sur un transport | SW03, SW07 |
| T6 | Pare-feu, filtrage d'URL et accès direct à Internet sécurisé pour un segment | SW08 |
| T7 | Conception de la connexion à un cloud public et choix de la sortie Internet | SW09 |
| T8 | Série de pannes (certificat, horloge, port, couleur, politique) à diagnostiquer | SW10 |
| T9 | Matrice de décision comparant trois solutions pour trois scénarios | SW11 |
| T10 | Architecture SASE d'une entreprise multi-sites et télétravailleurs | SW12 |
| T11 | Dossier de conception d'un réseau de 40 sites | SW13 |
| T12 | Plan de migration MPLS vers SD-WAN avec coexistence et retour arrière | SW14 |
| T13 | Script Python d'inventaire et de suivi des tunnels, modèles dans Git | SW15 |
| T14 | Journaux du SD-WAN vers un SIEM, règles de détection, révocation d'un équipement | SW17 |
| T15 | Tableau de bord de conformité aux SLA et revue mensuelle | SW18 |

---

## 14. Projets fil rouge

Un projet évolutif : **le WAN d'une entreprise multi-sites, de l'étude à l'exploitation**.

| Étape | Projet | Contenu |
|---|---|---|
| Partie A | **Étude de l'existant et cadrage** | Cahier des charges d'une entreprise fictive de 30 sites, analyse des limites du WAN actuel, objectifs et indicateurs cibles |
| Partie B | **Fabrique SD-WAN de laboratoire** | Contrôleurs, certificats, trois sites, segmentation, politiques de topologie et de routage applicatif, sécurité intégrée |
| Partie C | **Choix de solution et SASE** | Matrice de décision argumentée, architecture de sécurité avec services cloud |
| Partie D | **Conception complète** | Dossier de conception (topologie, adressage, haute disponibilité, QoS, sécurité, dimensionnement) |
| Partie D | **Migration et exploitation** | Plan de migration par vagues, automatisation par API, journaux vers un SIEM, tableau de bord de SLA |

### Idées de projets complémentaires

- Comparaison chiffrée d'une solution commerciale et d'un SD-WAN ouvert sur un même scénario (coûts, fonctionnalités, effort d'exploitation).
- Générateur de rapports hebdomadaires de qualité de chemin et de conformité aux SLA.
- Intégration du SD-WAN avec la supervision d'un homelab (Prometheus, Grafana) et les alertes.
- Plan de continuité d'activité d'un réseau de magasins avec accès 4G ou 5G en secours.
- Audit de sécurité d'une fabrique SD-WAN avec plan de remédiation.

---

## 15. Projets de synthèse

| Projet | Description | Modules |
|---|---|---|
| **Déploiement SD-WAN complet** | Contrôleurs, sites, modèles, segmentation, politiques, sécurité, Cloud OnRamp, supervision et dépannage | SW04 à SW10 |
| **Dossier de conception et de migration** | Conception d'une architecture pour un client fictif avec analyse de coûts, plan de migration et retour arrière | SW11 à SW14 |
| **Automatisation et exploitation du SD-WAN** | Scripts d'API, gestion de la configuration comme du code, intégration SIEM, indicateurs de SLA | SW15, SW17, SW18 |
| **Architecture SASE et Zero Trust** | Intégration SD-WAN et SSE, politiques d'accès par application, modèle de menace | SW08, SW12, SW17 |

**Livrables attendus** : schémas d'architecture, tableaux d'adressage et de segmentation, configurations et politiques, scripts, dossier de conception, plan de migration, rapport de tests et de dépannage, présentation orale.

---

## 16. Évaluations et certifications

### Évaluations du parcours

- Contrôles de connaissances par module.
- Labs notés avec grille.
- Exercice de dépannage chronométré.
- Évaluation d'un dossier de conception.
- Soutenance du projet de synthèse.

### Certifications associées

| Certification | Lien avec le parcours |
|---|---|
| **CCNP Enterprise** : ENCOR 350-401 puis concentration **ENSDWI 300-415** (Implementing Cisco SD-WAN Solutions) | Parties A, B et D |
| **CCIE Enterprise Infrastructure** | Le plan de l'examen comprend le SD-WAN |
| **Fortinet NSE 7 (spécialisation SD-WAN)** | Partie C : SD-WAN de Fortinet (vérifier l'intitulé en vigueur) |
| **MEF (certifications SD-WAN et Carrier Ethernet)** | Concepts indépendants des éditeurs (vérifier l'offre en vigueur) |
| Certifications des autres éditeurs (Palo Alto Networks, HPE Aruba Networking, Arista) | Selon la solution déployée |

> Vérifier sur les sites officiels les objectifs d'examen en vigueur, la durée, le format, le tarif et les conditions d'inscription : les programmes de certification et les noms de produits évoluent régulièrement.

---

## 17. Correspondance avec les référentiels

| Référentiel | Éléments concernés | Modules |
|---|---|---|
| Cisco **ENCOR 350-401** | Vue d'ensemble de SD-WAN et SD-Access, QoS, virtualisation | SW01 à SW04 |
| Cisco **ENSDWI 300-415** | Architecture, déploiement des contrôleurs et des équipements, politiques, sécurité et QoS, gestion et exploitation (vérifier la version du plan d'examen) | SW04 à SW10 |
| **RFC 5880** | BFD | SW02 |
| **RFC 4271** | BGP-4 | SW02, SW13, SW14 |
| **RFC 4364** | VPN BGP et MPLS | SW01, SW14 |
| **RFC 7296** et **RFC 4301** | IKEv2 et architecture IPsec | SW02, SW08 |
| **RFC 2475** | Architecture des services différenciés | SW03, SW07 |
| **NIST SP 800-207** | Architecture Zero Trust | SW08, SW12 |
| **NIST SP 800-77** | Guide des VPN IPsec | SW08 |

> Vérifier les versions en vigueur de chaque référentiel avant de figer le contenu du cours.

---

## 18. Guide pédagogique (pour l'enseignant)

- **Format type d'un module** : objectifs, théorie courte, démonstration, lab guidé, lab libre, quiz, débrief.
- **Alternance recommandée** : environ 35 % de théorie et 65 % de pratique ; chaque notion est illustrée par un schéma de flux et une vérification en laboratoire.
- **Approche par pannes** : dès SW05, proposer des laboratoires avec pannes injectées (certificat, horloge, pare-feu, couleur, politique) et imposer la méthode par plan.
- **Évaluation** : contrôles, labs notés, dépannage chronométré, dossier de conception, projet de synthèse.
- **Grille de notation d'un lab** : exactitude technique, preuves fournies (sorties de commandes, captures), respect de la méthode, justification des choix, documentation.
- **Points de vigilance fréquents** : confondre overlay et underlay ; confondre couleur et VPN ; oublier l'horloge et les certificats ; ne pas tenir compte du NAT et de la MTU ; appliquer une politique centralisée sans vérifier la liste de sites ; croire que le SD-WAN supprime le besoin de routage ; ne pas prévoir le retour arrière en migration ; confondre SD-WAN, SASE et SSE.
- **Accès au matériel et aux licences** : prévoir un laboratoire partagé ou un bac à sable ; les images commerciales peuvent exiger des licences, à anticiper.
- **Neutralité** : présenter le SD-WAN comme un ensemble de concepts, puis étudier une solution (Catalyst SD-WAN) en profondeur sans perdre le regard critique sur les alternatives.
- **Différenciation** : exercices guidés pour les apprenants en difficulté ; défis de conception, d'automatisation et d'architecture SASE pour les plus à l'aise.
- **Mise à jour du contenu** : suivre les changements de noms de produits, les rachats, les plans d'examen et les nouveautés de sécurité des éditeurs.

---

## 19. Structure recommandée du dépôt

```text
sd-wan-roadmap/
|-- README.md
|-- 00-prerequis/
|-- 01-partie-a-fondamentaux/
|   |-- SW01-du-wan-au-sd-wan/
|   |   |-- cours.md
|   |   `-- lab/
|   |-- SW02-transports-overlay-underlay/
|   `-- SW03-fonctions-cles/
|-- 02-partie-b-catalyst-sd-wan/
|-- 03-partie-c-solutions-et-sase/
|-- 04-partie-d-conception-migration-exploitation/
|-- complements/
|-- labs-transverses/
|-- projets/
|   |-- fil-rouge/
|   `-- synthese/
|-- fichiers-laboratoires/       (topologies CML, EVE-NG, GNS3)
|-- configs/                     (modeles, politiques, configurations de reference)
|-- scripts/                     (API, inventaire, rapports de SLA)
|-- dossiers-de-conception/      (modeles de dossier, tableaux d'adressage)
|-- captures/                    (fichiers pcap : IKE, ESP, BFD)
`-- docs/
    |-- glossaire.md
    |-- fiche-commandes-sdwan.md
    |-- fiche-depannage-par-plan.md
    |-- grille-de-choix-des-solutions.md
    |-- modele-plan-de-migration.md
    `-- grille-evaluation.md
```

---

## 20. Ressources

| Ressource | Lien |
|---|---|
| Cisco Catalyst SD-WAN (documentation et guides de conception) | <https://www.cisco.com/c/en/us/solutions/enterprise-networks/sd-wan/index.html> |
| Cisco DevNet : bacs à sable et API | <https://developer.cisco.com> |
| Cisco Modeling Labs | <https://developer.cisco.com/modeling-labs/> |
| Cisco Learning Network (certifications) | <https://learningnetwork.cisco.com> |
| Documentation Fortinet | <https://docs.fortinet.com> |
| FlexiWAN (SD-WAN open source) | <https://flexiwan.com> |
| MEF (normalisation du SD-WAN et des services Carrier Ethernet) | <https://www.mef.net> |
| NIST SP 800-207 (Zero Trust) | <https://csrc.nist.gov/publications/detail/sp/800-207/final> |
| RFC 5880 (BFD) | <https://www.rfc-editor.org/rfc/rfc5880> |
| RFC 4364 (VPN BGP et MPLS) | <https://www.rfc-editor.org/rfc/rfc4364> |
| RFC 7296 (IKEv2) | <https://www.rfc-editor.org/rfc/rfc7296> |
| Documentation Wireshark | <https://www.wireshark.org/docs/> |

> Les liens, les noms de produits, les versions et les plans d'examen évoluent : vérifier les versions en vigueur avant de figer le contenu du cours.

---

## 21. Suivi de progression

### Partie A : Fondamentaux du SD-WAN
- [ ] SW01 Du WAN traditionnel au SD-WAN
- [ ] SW02 Transports, overlay et underlay
- [ ] SW03 Fonctions clés du SD-WAN

### Partie B : Cisco Catalyst SD-WAN
- [ ] SW04 Catalyst SD-WAN : architecture et composants
- [ ] SW05 Déploiement des contrôleurs et intégration des équipements
- [ ] SW06 Modèles, groupes de configuration et segmentation
- [ ] SW07 Politiques centralisées et localisées
- [ ] SW08 Sécurité intégrée
- [ ] SW09 Cloud OnRamp et accès aux applications SaaS et IaaS
- [ ] SW10 Supervision, exploitation et dépannage

### Partie C : Autres solutions, SASE et SSE
- [ ] SW11 Panorama des solutions et critères de choix
- [ ] SW12 SASE et SSE

### Partie D : Conception, migration, automatisation et exploitation
- [ ] SW13 Conception d'une architecture SD-WAN
- [ ] SW14 Migration depuis un réseau MPLS
- [ ] SW15 Automatisation et API
- [ ] SW16 Intégration avec le LAN, le centre de données et les usages métier
- [ ] SW17 Sécurité opérationnelle et conformité
- [ ] SW18 Indicateurs, SLA et amélioration continue

### Compléments
- [ ] CP01 Routage avancé pour le SD-WAN
- [ ] CP02 Qualité de service en détail pour le WAN
- [ ] CP03 Fonctionnement d'IPsec en profondeur
- [ ] CP04 FlexiWAN et SD-WAN ouvert
- [ ] CP05 Télémétrie et observabilité du WAN
- [ ] CP06 Préparation aux certifications

### Projets et évaluations
- [ ] Projet fil rouge (étapes A à D)
- [ ] Au moins un projet de synthèse
- [ ] Exercice de dépannage chronométré
- [ ] Dossier de conception évalué
- [ ] Examen blanc de certification

---

## Licence et contributions

Ce document est une feuille de route évolutive. Les contributions (corrections, ajouts de labs, retours d'expérience) sont les bienvenues via issues et pull requests.

Étapes suivantes possibles : certification CCNP Enterprise (ENCOR et ENSDWI), architectures SASE et Zero Trust, ou automatisation réseau avancée.
