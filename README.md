# Cours complet SD-WAN : de la théorie à l'exploitation

> Formation en 18 modules et 6 compléments, avec laboratoires, scripts, études de cas, exercices corrigés et mini-projets. Elle s'appuie sur la feuille de route fournie (voir [docs/feuille-de-route.md](docs/feuille-de-route.md)).

---

## 1. À qui s'adresse ce cours

| Public | Ce que vous y trouverez |
|---|---|
| **Étudiants** (BTS, BUT, licence pro, école d'ingénieurs) | Les bases expliquées pas à pas, des définitions, des schémas, des quiz corrigés et des labs réalisables sur un poste personnel |
| **Techniciens et ingénieurs réseau** | Architecture, configuration, politiques, dépannage, automatisation, avec des commandes et des scripts réutilisables |
| **Architectes et chefs de projet** | Conception, dimensionnement, choix de solution, migration depuis MPLS, sécurité, indicateurs, dossiers types |
| **Formateurs** | Un programme complet, des grilles d'évaluation, des corrigés et des mini-projets prêts à l'emploi |
| **Candidats aux certifications** | Correspondance avec les objectifs d'examen, plans de révision et questions d'entraînement originales |

## 2. Comment utiliser ce cours

Chaque module est **un seul fichier**, nommé d'après le module, et suit la même trame :

1. **En-tête** : partie, durée, prérequis, niveau, lien avec les examens, lab associé.
2. **Table des matières** et **objectifs pédagogiques**.
3. **Cours** : définitions, schémas, tableaux, formules, commandes, configurations, scripts.
4. **Lab guidé** avec étapes, vérifications, livrables et **grille d'évaluation**.
5. **Approfondissements** : glossaire du module, **cas d'usage détaillés**, **retours d'expérience de terrain**, exercices supplémentaires corrigés, mini-projet.
6. **Quiz corrigé**, **aide-mémoire**, ressources pour aller plus loin, navigation précédent/suivant.

> **Conseil de méthode** : lisez le cours, **refaites les labs sans copier-coller**, répondez au quiz, puis traitez le mini-projet. Ce qui n'a pas été manipulé n'est pas acquis.

### Conventions et avertissements

- Les **configurations** (Cisco IOS XE SD-WAN, FortiOS, FRRouting, Linux…) sont **indicatives** : la syntaxe varie selon les versions. Vérifiez avec `?`, les notes de version et la documentation officielle avant toute utilisation en production.
- Les **chiffres** (coûts, délais, débits, seuils) sont des **ordres de grandeur pédagogiques** ou des **hypothèses d'exemple** : refaites vos propres calculs.
- Les **cas d'usage** sont des **scénarios réalistes mais fictifs**.
- Les **noms de produits et d'éditeurs** évoluent (rachats, renommages) : recoupez avec les sources officielles. Le cours ne constitue ni un classement ni une recommandation d'achat.
- Les exercices d'injection de pannes se font **uniquement en laboratoire**.
- Aucun secret réel n'apparaît dans ce cours : les valeurs entre chevrons (`<...>`) sont des **marqueurs à remplacer**.

---

## 3. Programme

### Partie A. Fondamentaux du SD-WAN

| Module | Titre | Lab |
|---|---|---|
| [SW01](01-partie-a-fondamentaux/SW01-du-wan-au-sd-wan/SW01-du-wan-au-sd-wan.md) | Du WAN traditionnel au SD-WAN | Schémas comparatifs MPLS/SD-WAN |
| [SW02](01-partie-a-fondamentaux/SW02-transports-overlay-underlay/SW02-transports-overlay-underlay.md) | Transports, overlay et underlay | T1 : overlay WireGuard et mesures |
| [SW03](01-partie-a-fondamentaux/SW03-fonctions-cles/SW03-fonctions-cles.md) | Fonctions clés du SD-WAN | Scénarios et simulation AAR |

### Partie B. Cisco Catalyst SD-WAN

| Module | Titre | Lab |
|---|---|---|
| [SW04](02-partie-b-catalyst-sd-wan/SW04-architecture-composants/SW04-architecture-composants.md) | Architecture et composants | Schéma annoté, bac à sable DevNet |
| [SW05](02-partie-b-catalyst-sd-wan/SW05-deploiement-controleurs-equipements/SW05-deploiement-controleurs-equipements.md) | Déploiement des contrôleurs et des équipements | T2 : contrôleurs, PKI, WAN Edge |
| [SW06](02-partie-b-catalyst-sd-wan/SW06-modeles-groupes-segmentation/SW06-modeles-groupes-segmentation.md) | Modèles, groupes de configuration et segmentation | T3 : 3 sites, 2 segments, DIA, VRRP |
| [SW07](02-partie-b-catalyst-sd-wan/SW07-politiques-centralisees-localisees/SW07-politiques-centralisees-localisees.md) | Politiques centralisées et localisées | T4 : topologies ; T5 : AAR de la voix |
| [SW08](02-partie-b-catalyst-sd-wan/SW08-securite-integree/SW08-securite-integree.md) | Sécurité intégrée | T6 : pare-feu, URL, durcissement |
| [SW09](02-partie-b-catalyst-sd-wan/SW09-cloud-onramp/SW09-cloud-onramp.md) | Cloud OnRamp | T7 : connexion cloud et sortie Internet |
| [SW10](02-partie-b-catalyst-sd-wan/SW10-supervision-exploitation-depannage/SW10-supervision-exploitation-depannage.md) | Supervision, exploitation et dépannage | T8 : pannes injectées |

### Partie C. Autres solutions, SASE et SSE

| Module | Titre | Lab |
|---|---|---|
| [SW11](03-partie-c-solutions-et-sase/SW11-panorama-solutions/SW11-panorama-solutions.md) | Panorama des solutions et critères de choix | T9 : matrice de décision |
| [SW12](03-partie-c-solutions-et-sase/SW12-sase-sse/SW12-sase-sse.md) | SASE et SSE | T10 : architecture SASE |

### Partie D. Conception, migration et exploitation

| Module | Titre | Lab |
|---|---|---|
| [SW13](04-partie-d-conception-migration-exploitation/SW13-conception-architecture/SW13-conception-architecture.md) | Conception d'architecture | T11 : dossier d'architecture (40 sites) |
| [SW14](04-partie-d-conception-migration-exploitation/SW14-migration-mpls/SW14-migration-mpls.md) | Migration depuis MPLS et coexistence | T12 : coexistence BGP/OMP |
| [SW15](04-partie-d-conception-migration-exploitation/SW15-automatisation-api/SW15-automatisation-api.md) | Automatisation, API et Infrastructure as Code | T13 : inventaire, Git, dérive, pipeline |
| [SW16](04-partie-d-conception-migration-exploitation/SW16-integration-lan-dc-usages/SW16-integration-lan-dc-usages.md) | Intégration au LAN, au centre de données et aux usages | Lab SW16 : site industriel |
| [SW17](04-partie-d-conception-migration-exploitation/SW17-securite-operationnelle/SW17-securite-operationnelle.md) | Sécurité opérationnelle et conformité | T14 : SIEM, règles, révocation |
| [SW18](04-partie-d-conception-migration-exploitation/SW18-indicateurs-sla/SW18-indicateurs-sla.md) | Indicateurs, SLA et amélioration continue | T15 : conformité SLA et revue mensuelle |

### Compléments

| Module | Titre | Lab |
|---|---|---|
| [CP01](complements/CP01-routage-avance/CP01-routage-avance.md) | Routage avancé : BGP, OSPF et redistribution | Maquette FRRouting/containerlab |
| [CP02](complements/CP02-qos-wan/CP02-qos-wan.md) | QoS pour le WAN | QoS sous Linux avec tc |
| [CP03](complements/CP03-ipsec-en-profondeur/CP03-ipsec-en-profondeur.md) | IPsec en profondeur | strongSwan, capture, pannes |
| [CP04](complements/CP04-flexiwan/CP04-flexiwan.md) | FlexiWAN | SD-WAN ouvert à trois sites |
| [CP05](complements/CP05-telemetrie-observabilite/CP05-telemetrie-observabilite.md) | Télémétrie et observabilité | Pile Prometheus/Grafana |
| [CP06](complements/CP06-preparation-certifications/CP06-preparation-certifications.md) | Préparation aux certifications | Examens blancs et labs de révision |

### Ressources transverses

| Document | Contenu |
|---|---|
| [docs/glossaire.md](docs/glossaire.md) | Glossaire général (termes SD-WAN, réseau, sécurité) |
| [docs/fiche-commandes.md](docs/fiche-commandes.md) | Fiche de commandes et de ports pour l'exploitation |
| [docs/feuille-de-route.md](docs/feuille-de-route.md) | Feuille de route d'origine |

---

## 4. Parcours conseillés

| Parcours | Durée indicative | Modules |
|---|---|---|
| **Découverte** (étudiants, managers) | 4 à 6 semaines | SW01, SW02, SW03, SW04, SW11, SW12 |
| **Ingénieur Cisco SD-WAN** | 14 à 18 semaines | SW01 à SW10, SW13, SW15, SW17, SW18, CP01 à CP03 |
| **Architecte et chef de projet** | 12 à 16 semaines | SW01 à SW04, SW07, SW08, SW09, SW11 à SW14, SW16, SW18 |
| **Exploitation et sécurité** | 10 à 14 semaines | SW04, SW05, SW08, SW10, SW15, SW17, SW18, CP05 |
| **Préparation ENSDWI** | 10 semaines | Plan du module CP06, appuyé sur SW04 à SW10 |
| **Cours complet** | 27 à 30 semaines (parcours principal, 8 à 12 h par semaine) puis 7 à 8 semaines de compléments | Tout |

## 5. Environnement de laboratoire

| Besoin | Option |
|---|---|
| **Poste** | 16 à 32 Go de RAM, 8 cœurs, 200 Go de disque (pour les labs complets de Cisco) |
| **Virtualisation** | VirtualBox, VMware, KVM/libvirt, Proxmox |
| **Simulateurs** | Cisco CML, EVE-NG, GNS3 (images à obtenir légalement selon les licences) |
| **Bacs à sable en ligne** | Cisco DevNet (SD-WAN), Meraki, services d'essai des éditeurs |
| **Labs open source** | Linux, WireGuard, FRRouting, containerlab, strongSwan, FlexiWAN, Prometheus, Grafana, Wazuh |
| **Outils** | Wireshark, iperf3, mtr, Python 3, Git, Docker |

> Les labs Cisco exigent des images et des licences : consultez les offres officielles (DevNet, programmes éducatifs). Les modules SW01 à SW03, SW11 à SW14 et les compléments s'appuient surtout sur des outils libres.

## 6. Suivi de progression

| Module | Lu | Lab réalisé | Quiz (score) | Mini-projet |
|---|---|---|---|---|
| SW01 | ☐ | ☐ | | ☐ |
| SW02 | ☐ | ☐ | | ☐ |
| SW03 | ☐ | ☐ | | ☐ |
| SW04 | ☐ | ☐ | | ☐ |
| SW05 | ☐ | ☐ | | ☐ |
| SW06 | ☐ | ☐ | | ☐ |
| SW07 | ☐ | ☐ | | ☐ |
| SW08 | ☐ | ☐ | | ☐ |
| SW09 | ☐ | ☐ | | ☐ |
| SW10 | ☐ | ☐ | | ☐ |
| SW11 | ☐ | ☐ | | ☐ |
| SW12 | ☐ | ☐ | | ☐ |
| SW13 | ☐ | ☐ | | ☐ |
| SW14 | ☐ | ☐ | | ☐ |
| SW15 | ☐ | ☐ | | ☐ |
| SW16 | ☐ | ☐ | | ☐ |
| SW17 | ☐ | ☐ | | ☐ |
| SW18 | ☐ | ☐ | | ☐ |
| CP01 à CP06 | ☐ | ☐ | | ☐ |

## 7. Évaluation (pour les formateurs)

Chaque lab comporte une **grille de notation** sur 20 environ. Suggestion de pondération pour un cursus :

| Élément | Poids |
|---|---|
| Quiz de modules | 15 % |
| Labs (grilles) | 40 % |
| Mini-projets (dossier et soutenance) | 30 % |
| Projet de synthèse (T11 ou T12) | 15 % |

## 8. Licence d'usage et contribution

Ce support est destiné à la **formation**. Adaptez-le, complétez-le avec vos propres retours d'expérience et **vérifiez les versions** de chaque produit avant de dispenser un module. Toute contribution doit éviter les données sensibles (adresses réelles, secrets) et préciser les versions testées.
