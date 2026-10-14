# CP06. Préparation aux certifications SD-WAN

| | |
|---|---|
| **Partie** | Compléments (hors parcours principal) |
| **Durée indicative** | 8 à 12 semaines de préparation (3 à 6 h par semaine) |
| **Prérequis** | Avoir suivi tout ou partie de SW01 à SW18 ; expérience de laboratoire |
| **Niveau** | Tous niveaux (voir parcours) |
| **Lien examen** | Cisco ENSDWI 300-415 et ENCOR 350-401 ; Fortinet NSE ; Cisco Meraki ; culture SASE |
| **Lab associé** | Examens blancs et révision par labs (section 9) |

> Module précédent : [CP05](../CP05-telemetrie-observabilite/CP05-telemetrie-observabilite.md) | Retour à l'index : [Cours complet](../../README.md)

> **Avertissement important.** Les **plans d'examen, intitulés, pondérations et codes** des certifications changent régulièrement. Les informations ci-dessous sont des **repères pédagogiques** : **consultez toujours la page officielle de l'examen** (objectifs détaillés, durée, nombre de questions, langues, prérequis, tarifs, politique de reprise) avant de planifier votre préparation. Aucune question de ce document n'est issue d'un examen réel : ce sont des questions **originales d'entraînement**. Évitez les « dumps » : ils sont contraires aux règles de certification et pédagogiquement inutiles.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Panorama des certifications](#2-panorama-des-certifications)
3. [Cisco ENSDWI 300-415 : domaines et correspondance avec le cours](#3-cisco-ensdwi-300-415--domaines-et-correspondance-avec-le-cours)
4. [Plans de préparation](#4-plans-de-préparation)
5. [Méthodes de révision efficaces](#5-méthodes-de-révision-efficaces)
6. [Fiches de révision](#6-fiches-de-révision)
7. [Questions d'entraînement corrigées](#7-questions-dentraînement-corrigées)
8. [Questions de scénario (type étude de cas)](#8-questions-de-scénario-type-étude-de-cas)
9. [Labs de révision](#9-labs-de-révision)
10. [Générateur de cartes de révision](#10-générateur-de-cartes-de-révision)
11. [Le jour de l'examen](#11-le-jour-de-lexamen)
12. [Au-delà de la certification](#12-au-delà-de-la-certification)
13. [Approfondissements](#13-approfondissements)
14. [Quiz final](#14-quiz-final)
15. [Aide-mémoire](#15-aide-mémoire)
16. [Pour aller plus loin](#16-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Choisir la certification adaptée à son profil et à son objectif professionnel.
- Faire correspondre les objectifs d'examen avec les modules du cours.
- Construire un plan de préparation réaliste et le suivre.
- Mobiliser des techniques de révision actives (laboratoire, répétition espacée, auto-évaluation).
- S'entraîner avec des questions originales et des scénarios.
- Aborder l'examen avec méthode.

---

## 2. Panorama des certifications

### 2.1 Pistes principales

| Certification | Éditeur | Contenu | Pour qui |
|---|---|---|---|
| **CCNA** | Cisco | Fondamentaux réseau | Débutants (prérequis conseillé) |
| **ENCOR 350-401** (CCNP Enterprise, examen principal) | Cisco | Réseaux d'entreprise : routage, QoS, sécurité, automatisation, notions de SD-WAN | Ingénieurs réseau |
| **ENSDWI 300-415** (concentration CCNP Enterprise) | Cisco | **Conception, déploiement et exploitation de Cisco Catalyst SD-WAN** | Spécialistes SD-WAN |
| **ENAUTO 300-435** | Cisco | Automatisation et programmabilité | Ingénieurs d'automatisation |
| **CCIE Enterprise Infrastructure** | Cisco | Expert : inclut SD-WAN dans le laboratoire | Experts |
| **Cisco Meraki (certifications Meraki)** | Cisco | Gestion des solutions Meraki | Exploitants Meraki |
| **Fortinet NSE** (niveaux 4 à 7, dont SD-WAN) | Fortinet | Fortinet Secure SD-WAN | Ingénieurs Fortinet |
| **Palo Alto Networks** (parcours Prisma SD-WAN) | Palo Alto | Prisma SD-WAN | Ingénieurs Palo Alto |
| **Certifications SASE/SSE** (éditeurs : Zscaler, Netskope, Cato, etc.) | Éditeurs | SSE, ZTNA, SWG | Ingénieurs sécurité |
| **Sécurité générale** : CISSP, Security+, SCOR 350-701 | (ISC)², CompTIA, Cisco | Sécurité | Profils sécurité |

### 2.2 Quel choix selon votre profil

| Profil | Objectif | Piste conseillée |
|---|---|---|
| **Étudiant en réseaux** | Premier emploi | CCNA, puis ENCOR ; labs de ce cours |
| **Ingénieur réseau en entreprise Cisco** | Spécialisation SD-WAN | ENCOR puis **ENSDWI** |
| **Intégrateur multi-éditeurs** | Polyvalence | ENSDWI + Fortinet NSE (SD-WAN) + une certification SASE |
| **Profil sécurité** | SASE/ZTNA | SCOR + certification SSE d'un éditeur |
| **Exploitant PME** | Gestion Meraki ou Fortinet | Parcours de l'éditeur retenu |
| **Architecte** | Vision globale | ENSDWI + CCIE (long terme) + connaissance du marché (SW11) |

---

## 3. Cisco ENSDWI 300-415 : domaines et correspondance avec le cours

### 3.1 Domaines (répartition indicative : à vérifier sur la page officielle)

| Domaine (intitulé simplifié) | Poids indicatif | Contenu principal |
|---|---|---|
| **Architecture** | ≈ 20 % | Composants, OMP, TLOC, couleurs, plans, connexions de contrôle |
| **Déploiement des contrôleurs** | ≈ 15 % | PKI, certificats, intégration des contrôleurs, haute disponibilité |
| **Déploiement des routeurs** | ≈ 20 % | Intégration des WAN Edge, modèles/groupes, routage de service, DIA |
| **Politiques** | ≈ 20 % | Politiques de contrôle, de données, AAR, topologies, localisées |
| **Sécurité et QoS** | ≈ 15 % | Pare-feu, IPS, filtrage, SIG, QoS |
| **Gestion et opérations** | ≈ 10 % | Supervision, dépannage, mises à jour, API |

### 3.2 Correspondance avec ce cours

| Domaine | Modules à travailler | Labs |
|---|---|---|
| Architecture | SW01 à SW04 | T1, T2 |
| Contrôleurs | SW05 | T2 |
| Routeurs | SW06, CP01 | T3 |
| Politiques | SW07 | T4, T5 |
| Sécurité et QoS | SW08, CP02, CP03 | T6 |
| Gestion et opérations | SW10, SW15, SW17, SW18, CP05 | T8, T13, T14, T15 |
| Cloud et SaaS | SW09 | T7 |
| Conception et migration | SW13, SW14 | T11, T12 |

### 3.3 Compétences pratiques attendues (savoir-faire)

| Compétence | À savoir faire sans aide |
|---|---|
| Lire `show sdwan control connections`, `omp`, `bfd` | Diagnostiquer un état en 2 minutes |
| Distinguer couleurs publiques et privées, `restrict` | Prédire quels tunnels se forment |
| Construire un modèle/groupe avec variables | Déployer un site |
| Écrire une politique de contrôle hub-and-spoke | Prédire le comportement |
| Écrire une politique AAR et une classe de SLA | Prévoir la bascule |
| Configurer DIA avec NAT | Vérifier avec `show ip nat translations` |
| Diagnostiquer les erreurs de connexion de contrôle | Associer code d'erreur et cause |
| Planifier une mise à jour | Ordre et retour arrière |

---

## 4. Plans de préparation

### 4.1 Plan de 10 semaines (rythme de 4 à 6 h par semaine, profil ingénieur réseau)

| Semaine | Thème | Modules | Travaux |
|---|---|---|---|
| 1 | Rappels et architecture | SW01 à SW03 | Lire, faire les quiz |
| 2 | Architecture Cisco | SW04 | Annoter le schéma, lire les sorties |
| 3 | Contrôleurs et PKI | SW05 | Lab T2 (ou DevNet) |
| 4 | Modèles, segmentation, routage | SW06, CP01 | Lab T3 |
| 5 | Politiques (1) : contrôle | SW07 | Lab T4 |
| 6 | Politiques (2) : AAR, QoS, données | SW07, CP02 | Lab T5 |
| 7 | Sécurité | SW08, CP03 | Lab T6 |
| 8 | Cloud et opérations | SW09, SW10 | Lab T7, T8 |
| 9 | Automatisation et exploitation | SW15, SW17, SW18 | Lab T13 à T15 (au moins un) |
| 10 | Révision et examens blancs | Tout | 2 examens blancs, revue des erreurs |

### 4.2 Plan accéléré de 6 semaines (déjà expérimenté en SD-WAN)

Semaines 1-2 : architecture, contrôleurs, routeurs ; semaines 3-4 : politiques, sécurité et QoS ; semaine 5 : opérations, API, cloud ; semaine 6 : révision et examens blancs.

### 4.3 Plan de 12 semaines (étudiant, 3 h par semaine)

Réutilisez le plan de 10 semaines en étalant : une semaine pour SW04 seul, une pour SW05 seul, deux pour SW07.

### 4.4 Suivi

Tableau de suivi (à tenir par le candidat) :

| Semaine | Heures prévues | Heures réalisées | Labs réalisés | Score d'auto-évaluation (%) | Points faibles |
|---|---|---|---|---|---|
| 1 | | | | | |

Un score régulièrement **supérieur à 85 %** aux questions d'entraînement **et** une capacité à refaire les labs **sans aide** sont les meilleurs indicateurs de préparation.

---

## 5. Méthodes de révision efficaces

| Méthode | Principe | Application |
|---|---|---|
| **Pratique en laboratoire** | On retient ce qu'on a manipulé | Refaire chaque lab en moins de 60 % du temps initial |
| **Rappel actif** | Se tester plutôt que relire | Cartes de révision, questions |
| **Répétition espacée** | Revoir à J+1, J+3, J+7, J+21 | Logiciel de cartes (Anki) |
| **Explication à un tiers** | Expliquer clarifie | Enseigner à un collègue, écrire un billet |
| **Cartes mentales** | Relier les notions | Un schéma par domaine |
| **Analyse des erreurs** | Chaque erreur = une leçon | Cahier d'erreurs : question, mauvaise réponse, cause, règle |
| **Mélange** | Alterner les sujets | Séances de questions mixtes |
| **Simulation de conditions** | Durée limitée, pas d'aide | Examens blancs chronométrés |

> Le piège le plus fréquent : **relire passivement** et confondre familiarité et maîtrise.

---

## 6. Fiches de révision

### 6.1 Ports et protocoles

| Élément | Valeur |
|---|---|
| Connexion de contrôle DTLS | UDP 12346 (+ décalages 12366, 12386, 12406, 12426) |
| Connexion de contrôle TLS | TCP 23456 (+ décalages) |
| Interface web / API | TCP 443 |
| NETCONF | TCP 830 |
| IPsec NAT-T | UDP 4500 ; IKE UDP 500 |
| BFD (par défaut) | Hello 1 000 ms, multiplicateur 7 |
| AAR (par défaut) | Poll-interval 600 000 ms, multiplicateur 6 |
| Graceful restart OMP | 43 200 s (12 h) |

### 6.2 Identifiants et couleurs

| Notion | À retenir |
|---|---|
| **System IP** | Unique, identifiant de l'équipement, non routable nécessairement |
| **Site ID** | Identique pour les équipements d'un même site |
| **TLOC** | (system-ip, couleur, encapsulation) |
| **Couleurs privées** | mpls, metro-ethernet, private1 à private6 |
| **Couleurs publiques** | default, biz-internet, public-internet, lte, 3g, red, green, blue, gold, silver, bronze, custom1-3 |
| **restrict** | Tunnels uniquement avec la même couleur |
| **VPN 0 / 512** | Transport / gestion |

### 6.3 Ordre de mise à jour et de déploiement

```text
Mise à jour : Manager -> Validator -> Controller -> WAN Edge (par vagues)
Intégration d'un WAN Edge : DHCP/DNS/NTP -> Validator (DTLS) -> Manager -> Controller (OMP) -> BFD
Diagnostic : Contrôle -> OMP -> Données (BFD) -> Politiques -> Application
```

### 6.4 Politiques

| Type | S'exécute | Sens | Exemple |
|---|---|---|---|
| Contrôle | Controller | in / out | hub-and-spoke |
| Données | WAN Edge | from-service / from-tunnel / all | DIA (nat use-vpn 0) |
| AAR | WAN Edge | — | sla-class VOICE preferred-color mpls |
| Localisée | WAN Edge | — | QoS, ACL |

### 6.5 Commandes essentielles

```text
show sdwan control connections | connections-history | local-properties
show sdwan omp summary | peers | routes | tlocs
show sdwan bfd sessions | summary
show sdwan policy from-vsmart | service-path ... | app-route-policy-filter
show sdwan app-route stats | sla-class
show sdwan certificate installed | root-ca-cert | serial
request platform software sdwan vedge_cloud activate chassis-number <UUID> token <OTP>
```

---

## 7. Questions d'entraînement corrigées

> Questions **originales** d'entraînement. Réponses et explications en fin de chaque bloc.

### 7.1 Architecture (8 questions)

1. Quel composant authentifie un nouvel équipement et lui indique les adresses des autres contrôleurs ?
   a) SD-WAN Manager ; b) SD-WAN Controller ; c) SD-WAN Validator ; d) WAN Edge.
2. Quel protocole annonce les routes, les TLOC et les clés de chiffrement dans l'overlay ?
   a) BGP ; b) OSPF ; c) OMP ; d) BFD.
3. Un WAN Edge possède deux interfaces de transport (mpls et biz-internet). Combien de TLOC possède-t-il ?
   a) 1 ; b) 2 ; c) 3 ; d) 4.
4. Quelle affirmation sur le VPN 512 est correcte ?
   a) Il transporte le trafic utilisateur ; b) Il sert à la gestion hors bande ; c) Il est le transport ; d) Il n'existe pas.
5. Avec `color mpls restrict` côté site A et `color biz-internet` côté site B, que se passe-t-il entre les deux interfaces ?
   a) Un tunnel se forme ; b) Aucun tunnel ne se forme ; c) Un tunnel GRE se forme ; d) Dépend du Controller.
6. Que se passe-t-il si tous les Controllers deviennent injoignables pendant 2 heures (graceful restart par défaut) ?
   a) Tout le trafic s'arrête ; b) Les tunnels et routes existants restent actifs ; c) Les équipements redémarrent ; d) Les tunnels passent en GRE.
7. Quel est le rôle de BFD dans les tunnels de données ?
   a) Chiffrer ; b) Détecter les pannes et mesurer perte, délai, gigue ; c) Distribuer les routes ; d) Authentifier.
8. Quelles informations identifient de façon unique un équipement dans l'overlay ?
   a) Le site ID ; b) La system IP ; c) La couleur ; d) Le numéro de VPN.

<details>
<summary>Réponses et explications</summary>

1. **c** : le Validator (orchestration). 2. **c** : OMP. 3. **b** : un TLOC par couleur/interface de transport. 4. **b**. 5. **b** : `restrict` interdit les tunnels vers des couleurs différentes. 6. **b** : le graceful restart conserve routes et tunnels (aucun changement n'est possible). 7. **b**. 8. **b** : la system IP identifie l'équipement ; le site ID identifie le site.

</details>

### 7.2 Contrôleurs et déploiement (6 questions)

9. Quelle est la conséquence d'une horloge décalée de plusieurs années sur un WAN Edge ?
   a) Aucune ; b) Échec de vérification des certificats ; c) Changement de couleur ; d) Perte de la configuration de VPN.
10. Quelle commande installe la racine sur un WAN Edge IOS XE ?
    a) `request root-cert-chain install` ; b) `request platform software sdwan root-cert-chain install bootflash:<fichier>` ; c) `crypto pki import` ; d) `install root`.
11. Quel élément doit être identique sur tous les composants de la fabrique ?
    a) La system IP ; b) Le site ID ; c) L'organization name ; d) Le nom d'hôte.
12. Dans quel ordre met-on à jour les composants ?
    a) WAN Edge, Controller, Validator, Manager ; b) Manager, Validator, Controller, WAN Edge ; c) Controller, Manager, WAN Edge, Validator ; d) Peu importe.
13. Un WAN Edge neuf est absent de la WAN Edge List « Valid » envoyée aux Controllers. Que constate-t-on ?
    a) Il rejoint la fabrique normalement ; b) Sa connexion de contrôle échoue ; c) Il rejoint en mode dégradé ; d) Il devient un Validator.
14. Que contient le fichier d'amorçage de type cloud-init d'un routeur virtuel ?
    a) La configuration complète des politiques ; b) UUID, jeton (OTP), Validator, organisation ; c) La liste des VPN ; d) Les certificats des contrôleurs.

<details>
<summary>Réponses et explications</summary>

9. **b**. 10. **b**. 11. **c**. 12. **b**. 13. **b** (rejet : motif de type `SERNTPRES`/`CRTREJSER`). 14. **b**.

</details>

### 7.3 Routeurs, modèles et routage de service (5 questions)

15. Quel code porte une route OMP dans la table de routage d'une VRF IOS XE ?
    a) O ; b) B ; c) m ; d) S.
16. Pour qu'un site apprenne par OSPF les réseaux d'autres sites, que configurer dans OSPF ?
    a) `redistribute omp subnets` ; b) `redistribute bgp` ; c) `network 0.0.0.0` ; d) `default-information originate`.
17. Quel mécanisme évite une boucle lors d'une redistribution mutuelle entre OMP et OSPF ?
    a) NTP ; b) Une balise (tag) posée à la redistribution et filtrée à l'entrée ; c) VRRP ; d) NAT.
18. Quel mot-clé active une interface 4G seulement quand les autres transports sont indisponibles ?
    a) `backup` ; b) `last-resort-circuit` ; c) `restrict` ; d) `port-hop`.
19. Pour sortir directement sur Internet depuis le VPN 20 avec NAT (sans politique de données), que faut-il ?
    a) Une route par défaut du VPN 20 vers la table globale et un NAT sur l'interface de transport ; b) Rien ; c) Un second Validator ; d) Un VPN 0 sur le LAN.

<details>
<summary>Réponses et explications</summary>

15. **c**. 16. **a**. 17. **b**. 18. **b**. 19. **a** (`ip route vrf 20 0.0.0.0 0.0.0.0 <interface> <passerelle> global` + `ip nat inside source list ... interface ... vrf 20 overload`).

</details>

### 7.4 Politiques (8 questions)

20. Quelle politique est exécutée sur le Controller ?
    a) AAR ; b) Données ; c) Contrôle ; d) QoS.
21. Combien de politiques centralisées peuvent être actives simultanément ?
    a) 1 ; b) 2 ; c) 10 ; d) Autant que de sites.
22. Dans une politique de contrôle hub-and-spoke appliquée aux rayons, quel est le comportement attendu du `default-action reject` ?
    a) Les rayons reçoivent tout ; b) Les rayons ne reçoivent que les TLOC et routes explicitement acceptées (hubs) ; c) Les hubs perdent leurs routes ; d) Les tunnels passent en GRE.
23. Quelle est la signification de `from-service` dans une politique de données ?
    a) Trafic venant de l'overlay ; b) Trafic venant du LAN ; c) Trafic de contrôle ; d) Trafic de gestion.
24. Quelle commande indique quel chemin une politique choisirait pour un flux donné ?
    a) `show sdwan omp routes` ; b) `show sdwan policy service-path` ; c) `show sdwan bfd sessions` ; d) `show ip route`.
25. Quel est le DSCP de la voix (EF) et son octet ToS ?
    a) 34 et 136 ; b) 46 et 184 ; c) 26 et 104 ; d) 48 et 192.
26. Une AAR bascule la voix sur Internet quand MPLS dépasse 150 ms. Quel élément définit cette limite ?
    a) La couleur ; b) La classe de SLA ; c) Le site ID ; d) Le VPN.
27. Les rayons ne communiquent plus entre eux après une politique hub-and-spoke. Quelle est la cause la plus probable ?
    a) Le hub n'annonce pas de résumé ou de route par défaut couvrant les rayons ; b) Le NTP ; c) Le DNS ; d) La licence.

<details>
<summary>Réponses et explications</summary>

20. **c**. 21. **a**. 22. **b**. 23. **b**. 24. **b**. 25. **b**. 26. **b**. 27. **a**.

</details>

### 7.5 Sécurité et QoS (6 questions)

28. Pourquoi n'y a-t-il pas d'IKE entre WAN Edge dans Catalyst SD-WAN ?
    a) Les clés sont distribuées par le plan de contrôle (OMP) ; b) IKE est interdit ; c) Les tunnels ne sont pas chiffrés ; d) Les clés sont saisies manuellement.
29. Quelle fonction de sécurité protège contre les domaines malveillants dès la résolution ?
    a) IPS ; b) Protection DNS ; c) ZBFW ; d) AMP.
30. Pourquoi lisser le débit sortant au débit contractuel d'un lien ?
    a) Pour que la file se forme chez soi où la QoS agit ; b) Pour économiser l'énergie ; c) Pour chiffrer ; d) Pour réduire le MTU.
31. Quel risque présente un `allow-service all` sur une interface exposée à Internet ?
    a) Aucun ; b) Exposition de services de gestion à Internet ; c) Perte de BFD ; d) Changement de couleur.
32. Dans une file LLQ, pourquoi borne-t-on la file prioritaire ?
    a) Pour éviter qu'elle n'affame les autres classes ; b) Pour économiser la mémoire ; c) Pour améliorer le chiffrement ; d) Pour réduire la MTU.
33. Quelle est la différence entre IDS et IPS ?
    a) Aucune ; b) IDS détecte, IPS bloque ; c) IDS chiffre ; d) IPS journalise seulement.

<details>
<summary>Réponses et explications</summary>

28. **a**. 29. **b**. 30. **a**. 31. **b**. 32. **a**. 33. **b**.

</details>

### 7.6 Gestion et opérations (6 questions)

34. Quelle est la première étape de la méthode de dépannage par plan ?
    a) Politiques ; b) Contrôle ; c) Application ; d) QoS.
35. Quelle commande montre la raison de la dernière défaillance d'une connexion de contrôle ?
    a) `show sdwan control connections-history` ; b) `show sdwan omp tlocs` ; c) `show sdwan bfd sessions` ; d) `show sdwan version`.
36. Que signifie le motif `CRTVERFL` ?
    a) Échec de vérification du certificat ; b) Couleur incorrecte ; c) Politique absente ; d) NAT symétrique.
37. À quoi sert l'authentification de l'API par jeton CSRF (`X-XSRF-TOKEN`) ?
    a) Pour les requêtes qui modifient l'état ; b) Pour chiffrer ; c) Pour la pagination ; d) Pour le DNS.
38. Quel outil du Manager trace un flux de bout en bout à travers plusieurs équipements ?
    a) Network-Wide Path Insight ; b) Ping ; c) Admin Tech ; d) Banner.
39. Pourquoi conserver l'historique des mesures de tunnel pendant au moins 12 mois ?
    a) Pour la prévision de capacité et la preuve face à un opérateur ; b) Pour économiser du disque ; c) Pour le chiffrement ; d) Pour les licences.

<details>
<summary>Réponses et explications</summary>

34. **b**. 35. **a**. 36. **a**. 37. **a**. 38. **a**. 39. **a**.

</details>

---

## 8. Questions de scénario (type étude de cas)

### Scénario 1. Nouveau site qui n'apparaît pas

Un routeur 8000V est installé dans une agence, branché sur la box de l'opérateur. Aucun appel au Manager. Sorties : `show sdwan control connections` vide ; `ping` du Validator OK ; `show clock` indique l'année 2019 ; certificat racine installé.

**Questions** : (a) Quelle est la cause probable ? (b) Quelle commande la confirme ? (c) Que corriger ?

<details>
<summary>Réponse</summary>

(a) Horloge incorrecte : le certificat est jugé non valide (date antérieure). (b) `show clock`, `show sdwan control connections-history` (motif `CRTVERFL`). (c) Rétablir NTP ou `clock set`, vérifier la synchronisation, relancer la connexion.

</details>

### Scénario 2. Voix de mauvaise qualité

La voix entre deux agences passe par Internet alors que MPLS est sain. Une politique AAR existe avec `sla-class VOICE preferred-color mpls`. `show sdwan app-route sla-class` montre MPLS « non conforme » : gigue de 35 ms (limite 30).

**Questions** : (a) Pourquoi la voix ne suit-elle pas MPLS ? (b) Que faire ?

<details>
<summary>Réponse</summary>

(a) MPLS ne respecte pas la classe de SLA (gigue 35 ms > 30 ms) : l'AAR bascule vers un chemin conforme. (b) Analyser la gigue de MPLS (opérateur, congestion), ajuster les seuils si réalistes (en sachant les conséquences) ou corriger la cause ; vérifier que les seuils reflètent un besoin réel.

</details>

### Scénario 3. Les rayons ne se joignent plus

Après activation d'une politique de contrôle hub-and-spoke appliquée aux rayons en `out`, les agences ne communiquent plus entre elles ; l'accès au DC fonctionne.

**Questions** : (a) Pourquoi ? (b) Quelle solution ?

<details>
<summary>Réponse</summary>

(a) Les rayons ne reçoivent que les TLOC et routes des hubs ; ils n'ont plus les préfixes des autres agences. (b) Faire annoncer par les hubs un résumé couvrant les LAN des agences (ou une route par défaut) ; le trafic rayon à rayon passera par le hub.

</details>

### Scénario 4. Internet des invités

Les invités doivent sortir directement sur Internet (VPN 20), sans accès au VPN 10. Le DIA n'apporte pas de connectivité : `show ip nat translations vrf 20` est vide.

**Questions** : (a) Citez trois causes. (b) Quelles vérifications ?

<details>
<summary>Réponse</summary>

(a) Route par défaut du VPN 20 vers la table globale absente, ACL de NAT incorrecte, `ip nat inside/outside` manquants sur les interfaces, NAT non appliqué sur la bonne VRF, ou politique de données absente. (b) `show ip route vrf 20 0.0.0.0`, `show run | section ip nat`, `show ip nat statistics`, capture sur l'interface de transport.

</details>

### Scénario 5. Mise à jour

Vous devez passer d'une version N à une version N+1 d'un parc de 120 sites. Décrivez l'ordre, la stratégie de vagues, les contrôles et le retour arrière.

<details>
<summary>Réponse</summary>

Sauvegarde et snapshots ; lecture des notes de version et de la matrice de compatibilité ; mise à jour Manager, puis Validator, puis Controllers ; vérification de la santé ; pilote de 3 sites (48 h) ; vagues croissantes (10 %, 25 %, reste) avec critères de passage ; utilisation du minuteur de retour automatique et de la version par défaut ; plan de retour arrière documenté ; contrôle après chaque vague.

</details>

---

## 9. Labs de révision

| Lab | Objectif | Temps cible (2e passage) |
|---|---|---|
| T2 | Intégrer contrôleurs et WAN Edge, certificats | 90 min |
| T3 | 3 sites, 2 segments, OSPF, DIA, VRRP | 90 min |
| T4 | Hub-and-spoke et maillage partiel | 40 min |
| T5 | AAR de la voix et bascule | 45 min |
| T6 | Pare-feu, filtrage, durcissement | 60 min |
| T8 | 6 pannes injectées | 90 min |
| T13 | Inventaire et sauvegarde par API | 60 min |

**Défi final** : en 4 heures, à partir d'un lab vierge, intégrer deux sites avec deux segments, une politique hub-and-spoke, une AAR de la voix, un DIA pour les invités, puis diagnostiquer deux pannes tirées au sort.

---

## 10. Générateur de cartes de révision

Convertit un fichier de questions/réponses en CSV importable dans Anki.

```python
#!/usr/bin/env python3
"""cartes_anki.py : génère un CSV (recto;verso;étiquettes) pour Anki.

Fichier d'entrée cartes.md au format :
Q: Quel port utilise DTLS pour la connexion de contrôle ?
R: UDP 12346 (et décalages)
T: ports

Les blocs sont séparés par une ligne vide.
"""
import csv
import sys
from pathlib import Path

blocs = Path(sys.argv[1] if len(sys.argv) > 1 else "cartes.md").read_text(encoding="utf-8").strip().split("\n\n")
with open("cartes_anki.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.writer(f, delimiter=";")
    n = 0
    for b in blocs:
        champs = {l[:1]: l[2:].strip() for l in b.splitlines() if len(l) > 2 and l[1] == ":"}
        if "Q" in champs and "R" in champs:
            w.writerow([champs["Q"], champs["R"], champs.get("T", "sdwan")])
            n += 1
print(n, "cartes générées dans cartes_anki.csv")
```

Exemple de contenu de `cartes.md` :

```text
Q: Que signifie le statut C,I,R d'un TLOC ?
R: Chosen, Installed, Resolved
T: omp

Q: Quel est l'ordre de mise à jour des composants ?
R: Manager, Validator, Controller, WAN Edge
T: operations
```

Règle d'or : **écrire ses propres cartes** (une idée par carte) est une révision en soi.

---

## 11. Le jour de l'examen

### 11.1 Avant

| Élément | Conseil |
|---|---|
| **Veille** | Pas de révision intense ; relire les fiches (section 6) ; dormir |
| **Pièces** | Pièce d'identité, confirmation d'inscription ; test technique si examen en ligne |
| **Lieu** | Centre agréé ou environnement conforme (en ligne) : bureau dégagé, connexion fiable |
| **Repas** | Léger, hydratation |

### 11.2 Pendant

| Technique | Détail |
|---|---|
| **Gestion du temps** | Calculer un temps moyen par question (durée / nombre de questions) ; ne pas s'attarder |
| **Lecture attentive** | Mots clés (« **sauf** », « **le plus** », « **deux** réponses ») |
| **Élimination** | Écarter les options manifestement fausses |
| **Marquer et revenir** | Passer les questions longues, y revenir |
| **Simulations et questions de type pratique** | Lire toute la topologie avant d'agir |
| **Ne pas changer sans raison** | La première intuition fondée est souvent la bonne |
| **Gérer le stress** | Respiration, pause d'une minute |

### 11.3 Après

Analysez le rapport de score par domaine ; en cas d'échec, planifiez la reprise (délais de ré-inscription), concentrez-vous sur les domaines faibles ; en cas de réussite, valorisez (profil professionnel) et planifiez la suite.

---

## 12. Au-delà de la certification

| Étape | Action |
|---|---|
| **Pratiquer** | Mettre en œuvre en vrai (projets, lab domestique) |
| **Partager** | Écrire des billets, présenter en réunion, animer un atelier |
| **Contribuer** | Scripts, modèles et runbooks open source (sans données sensibles) |
| **Se spécialiser** | Sécurité, automatisation, conception, cloud |
| **Rester à jour** | Notes de version, avis de sécurité, conférences, communautés |
| **Mentorer** | Aider un débutant : la meilleure façon d'apprendre |

Une certification **atteste** d'un niveau à un instant donné ; l'expérience le **démontre** dans la durée.

---

## 13. Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **Concentration** | Examen de spécialisation d'une certification Cisco (CCNP) |
| **Core exam** | Examen principal d'un parcours |
| **Blueprint** | Plan détaillé des objectifs d'un examen |
| **Simlet / simulation** | Question pratique avec simulation de CLI |
| **Drag-and-drop** | Question d'association |
| **Répétition espacée** | Technique de révision aux intervalles croissants |
| **Dump** | Fuite de questions réelles : interdit |
| **NDA** | Accord de confidentialité accepté lors de l'examen |
| **Recertification** | Renouvellement périodique d'une certification |
| **CCIE** | Cisco Certified Internetwork Expert |
| **DevNet** | Programme Cisco pour l'automatisation et le développement |
| **Lab domestique** | Environnement personnel d'entraînement |
| **PDCA** | Cycle d'amélioration continue appliqué à l'étude |

### B. Cas d'usage (parcours types)

#### Cas 1. Étudiante en licence professionnelle

Objectif : premier poste d'ingénieure réseau. Parcours : CCNA pendant les études, SW01 à SW07 et labs T1 à T5, puis ENCOR ; projet de fin d'études sur la conception d'un SD-WAN pour une PME (SW13). **Résultat** : un portfolio de labs et rapports, un entretien d'embauche appuyé par des réalisations. **Leçon** : le **portfolio** compte autant que le diplôme.

#### Cas 2. Ingénieur réseau en reconversion vers le SD-WAN

Dix ans d'expérience en routage ; ENCOR déjà obtenu. Plan accéléré de 6 semaines, focus sur politiques et sécurité. Réussite à l'ENSDWI au premier passage. **Leçon** : l'expérience accélère, mais **la pratique du produit** reste décisive.

#### Cas 3. Architecte d'intégrateur

Objectif : crédibilité face aux clients. Parcours : ENSDWI + une certification Fortinet SD-WAN + lecture de SW11 et SW12 pour comparer les solutions. **Leçon** : un architecte qui connaît **plusieurs solutions** conseille mieux.

#### Cas 4. Administrateur en PME

Environnement Meraki : formation éditeur, mise en place de supervision (CP05), pas de certification CCNP. **Leçon** : la **bonne certification** est celle qui sert **votre contexte**.

#### Cas 5. Équipe d'exploitation

Un responsable organise des séances hebdomadaires de 1 h : un lab de panne injectée (SW10) par semaine, une carte de révision rédigée par personne. En 3 mois, temps de dépannage divisé par 2. **Leçon** : la révision collective **diffuse la compétence**.

### C. Retours d'expérience de formateurs

1. **Les candidats qui échouent ont souvent beaucoup lu et peu pratiqué.**
2. **Les échecs viennent des détails** : défauts (timers, ports), ordres, syntaxes ; fiches et rappel actif.
3. **Lire l'énoncé en entier** : les pièges sont dans les négations et les exceptions.
4. **Ne pas apprendre de « dumps »** : ils ne servent pas le métier et exposent à des sanctions.
5. **Un examen blanc ne vaut que s'il est corrigé en profondeur.**
6. **Planifier la date d'examen tôt** crée un engagement et un rythme.
7. **Dormir avant l'examen** vaut plus qu'une dernière heure de révision.
8. **Une certification n'est pas une fin** : c'est un jalon.

### D. Exercices supplémentaires

1. Construisez votre plan de préparation personnalisé avec dates, heures hebdomadaires et jalons ; faites-le valider par un pair.
2. Rédigez 30 cartes de révision sur les domaines où vous vous sentez le plus faible.
3. Écrivez 10 questions à choix multiples **originales** sur la sécurité et corrigez-les avec explications.
4. Chronométrez-vous sur le défi final de la section 9 ; analysez les écarts.
5. Préparez un exposé de 10 minutes « Comment fonctionne OMP » pour un public de non-spécialistes.

### E. Mini-projet (12 heures)

Constituez votre **dossier de préparation** : plan, 100 cartes de révision, cahier d'erreurs, 3 comptes rendus de labs, 1 examen blanc chronométré avec analyse, fiche de synthèse d'une page par domaine, et un **retour d'expérience** à partager avec votre groupe.

---

## 14. Quiz final

1. Pourquoi privilégier la pratique en laboratoire plutôt que la relecture ?
2. Que signifie la répétition espacée ?
3. Pourquoi éviter les « dumps » ?
4. Quel est l'ordre de la méthode de dépannage par plan ?
5. Quelle est la différence entre un core exam et une concentration ?
6. Citez trois compétences pratiques d'un candidat à l'ENSDWI.
7. Que faire immédiatement après un échec à l'examen ?

<details>
<summary>Réponses</summary>

1. Parce que la mémoire procédurale et la compréhension se construisent en manipulant. 2. Revoir une notion à des intervalles croissants. 3. Ils sont contraires aux règles de certification, n'attestent pas de compétence réelle et exposent à des sanctions. 4. Contrôle, OMP, données (BFD), politiques, application. 5. Le core exam est l'examen principal d'un parcours ; la concentration est un examen de spécialisation. 6. Par exemple : lire les sorties de connexions de contrôle/OMP/BFD, écrire une politique de contrôle, configurer un DIA. 7. Analyser le rapport de score par domaine, identifier les causes, replanifier en respectant les délais de reprise.

</details>

---

## 15. Aide-mémoire

```text
Choisir : CCNA -> ENCOR -> ENSDWI (SD-WAN Cisco) | NSE Fortinet SD-WAN | Meraki | SSE/SASE éditeurs | sécurité (SCOR, CISSP)
Vérifier : plan d'examen officiel (domaines, poids, durée), prérequis, tarifs, reprise
Plan : 10 semaines = architecture -> contrôleurs -> routeurs -> politiques -> sécurité/QoS -> opérations -> révision + 2 examens blancs
Méthodes : labs refaits sans aide, cartes (répétition espacée), cahier d'erreurs, explication à un tiers, examens blancs chronométrés
Jour J : sommeil, pièces, temps par question, lire les mots clés, éliminer, marquer, ne pas changer sans raison
Après : valoriser, pratiquer, partager, spécialiser
```

## 16. Pour aller plus loin

- Cisco Learning Network, pages des examens et certifications : <https://www.cisco.com/c/en/us/training-events/training-certifications/certifications.html>
- Cisco DevNet (laboratoires et bacs à sable) : <https://developer.cisco.com>
- Fortinet Training Institute : <https://training.fortinet.com>
- Anki (cartes de révision) : <https://apps.ankiweb.net>

> Module précédent : [CP05](../CP05-telemetrie-observabilite/CP05-telemetrie-observabilite.md) | Retour à l'index : [Cours complet](../../README.md)
