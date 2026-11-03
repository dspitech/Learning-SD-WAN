# Cours complet SD-WAN : les 4 plans

Un cours structuré pour comprendre, concevoir et exploiter une architecture SD-WAN, illustré par un fil rouge inspiré d'une grande enseigne de distribution comme Carrefour.

> Avertissement : le scénario Carrefour de ce document est un cas pédagogique fictif. Les chiffres, noms de VPN, topologies et choix techniques sont inventés pour illustrer les concepts. Ils ne décrivent pas l'architecture réelle de l'entreprise.

---

## Table des matières

1. [Objectifs pédagogiques et prérequis](#1-objectifs-pédagogiques-et-prérequis)
2. [Rappels : du WAN traditionnel au SD-WAN](#2-rappels--du-wan-traditionnel-au-sd-wan)
3. [Définition et bénéfices du SD-WAN](#3-définition-et-bénéfices-du-sd-wan)
4. [Fil rouge : le groupe de distribution "Carrefour"](#4-fil-rouge--le-groupe-de-distribution-carrefour)
5. [Vue d'ensemble : les 4 plans du SD-WAN](#5-vue-densemble--les-4-plans-du-sd-wan)
6. [Plan 1 : le plan d'orchestration](#6-plan-1--le-plan-dorchestration)
7. [Plan 2 : le plan de gestion (management)](#7-plan-2--le-plan-de-gestion-management)
8. [Plan 3 : le plan de contrôle](#8-plan-3--le-plan-de-contrôle)
9. [Plan 4 : le plan de données (data plane)](#9-plan-4--le-plan-de-données-data-plane)
10. [Les 4 plans ensemble : scénarios de bout en bout](#10-les-4-plans-ensemble--scénarios-de-bout-en-bout)
11. [Correspondance de vocabulaire entre éditeurs](#11-correspondance-de-vocabulaire-entre-éditeurs)
12. [Sécurité, segmentation et SASE](#12-sécurité-segmentation-et-sase)
13. [Conception : topologies, haute disponibilité, dimensionnement](#13-conception--topologies-haute-disponibilité-dimensionnement)
14. [Déploiement et migration depuis MPLS](#14-déploiement-et-migration-depuis-mpls)
15. [Exploitation, supervision et dépannage](#15-exploitation-supervision-et-dépannage)
16. [Exemple de configuration (style Cisco)](#16-exemple-de-configuration-style-cisco)
17. [Exercices et corrigés](#17-exercices-et-corrigés)
18. [Bonnes pratiques et erreurs fréquentes](#18-bonnes-pratiques-et-erreurs-fréquentes)
19. [Glossaire](#19-glossaire)
20. [Pour aller plus loin](#20-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques et prérequis

### Objectifs

À la fin de ce cours, vous serez capable de :

- expliquer pourquoi le SD-WAN existe et quels problèmes il résout ;
- décrire les 4 plans du SD-WAN (orchestration, gestion, contrôle, données) et le rôle de chacun ;
- relier chaque plan à des composants concrets et à des protocoles ;
- raisonner sur un cas réel de distribution multi-sites (magasins, entrepôts, sièges, cloud) ;
- concevoir une architecture SD-WAN de base : segmentation, politiques, haute disponibilité ;
- diagnostiquer une panne en identifiant le plan concerné.

### Prérequis

- Notions de réseau IP : adressage, routage statique et dynamique (OSPF, BGP).
- Notions de VPN IPsec et de NAT.
- Idée générale de ce qu'est une liaison MPLS et un accès Internet (fibre, xDSL, 4G/5G).

---

## 2. Rappels : du WAN traditionnel au SD-WAN

### 2.1 Le WAN classique

Un WAN (Wide Area Network) relie des sites géographiquement distants : magasins, entrepôts, sièges, datacenters. Pendant des années, le modèle dominant a été :

- un **réseau MPLS** opéré par un opérateur, reliant chaque site à un ou deux datacenters ;
- un **routage statique ou BGP** configuré site par site ;
- un **trafic Internet centralisé** : tout le trafic vers Internet est renvoyé vers le datacenter, qui héberge les pare-feu et proxys.

```
        Magasin A ----\
        Magasin B -----\
        Magasin C ------>  [ Réseau MPLS ]  ---->  Datacenter  ---->  Internet
        Entrepôt  -----/                              (pare-feu, proxy)
```

### 2.2 Les limites de ce modèle

| Limite | Explication |
|---|---|
| Coût | La bande passante MPLS coûte cher au Mbit/s. |
| Délais de déploiement | Un nouveau site peut attendre plusieurs semaines ou mois pour un circuit opérateur. |
| Rigidité | Un changement de politique se fait site par site, souvent en CLI. |
| Trafic cloud inefficace | Un accès à Microsoft 365 ou à un SaaS fait un détour par le datacenter (effet "tromboning"). |
| Faible visibilité | Peu de visibilité applicative sur l'expérience réelle de l'utilisateur. |
| Dépendance à un opérateur | Peu de leviers pour négocier ou changer de transport. |

### 2.3 Ce qui a changé

- Les applications migrent vers le **SaaS et l'IaaS** (messagerie, ERP en cloud, e-commerce, supervision).
- Les accès **Internet haut débit** (fibre, 5G) sont devenus fiables et peu chers.
- La **virtualisation et l'automatisation** permettent de piloter un réseau comme un système logiciel.

Le SD-WAN répond à cette situation en ajoutant une couche logicielle intelligente au-dessus de n'importe quel transport.

---

## 3. Définition et bénéfices du SD-WAN

### 3.1 Définition

Le **SD-WAN (Software-Defined Wide Area Network)** est une architecture qui :

1. **sépare** l'intelligence (contrôle, politiques) du matériel de transfert des paquets ;
2. **abstrait** les transports (MPLS, Internet, 4G/5G) dans une **couche overlay** unifiée ;
3. **centralise** la configuration et la politique dans une console unique ;
4. **choisit dynamiquement** le meilleur chemin pour chaque application selon des mesures de qualité en temps réel.

### 3.2 Underlay et overlay

- **Underlay** : les transports physiques fournis par les opérateurs (MPLS, fibre Internet, 5G).
- **Overlay** : des tunnels chiffrés (en général IPsec) construits entre les équipements SD-WAN, par-dessus l'underlay.

```
   Overlay SD-WAN (tunnels IPsec, politiques, segmentation)
  ===========================================================
   Underlay :  [ MPLS ]     [ Internet fibre ]     [ 5G ]
```

Le réseau applicatif voit un seul réseau logique, quel que soit le nombre de transports réels.

### 3.3 Bénéfices attendus

- **Agilité** : ouverture d'un site en jours plutôt qu'en semaines (zero-touch provisioning).
- **Coûts** : usage de liens Internet standards, souvent en complément ou en remplacement partiel de MPLS.
- **Performance applicative** : choix du chemin selon la latence, la gigue et la perte mesurées.
- **Sécurité** : chiffrement systématique, segmentation, intégration avec des services de sécurité cloud.
- **Simplicité d'exploitation** : modèles de configuration et politiques centralisées.
- **Visibilité** : tableaux de bord par application, par site, par lien.

### 3.4 Ce que le SD-WAN n'est pas

- Ce n'est pas un simple "VPN over Internet" : il y a une couche de contrôle centralisée et une mesure de qualité continue.
- Ce n'est pas magique : une liaison Internet de mauvaise qualité reste mauvaise. Le SD-WAN aide à la contourner, pas à la corriger.
- Ce n'est pas forcément la fin du MPLS : beaucoup d'entreprises gardent un MPLS pour les flux critiques et ajoutent de l'Internet.

---

## 4. Fil rouge : le groupe de distribution "Carrefour"

> Rappel : scénario fictif et simplifié, inspiré du secteur de la grande distribution.

### 4.1 Le contexte

Imaginons un groupe de distribution présent dans plusieurs pays avec des milliers de points de vente :

| Type de site | Nombre (illustratif) | Besoins réseau |
|---|---|---|
| Hypermarchés | 300 | Forte bande passante, caisses, Wi-Fi clients, vidéosurveillance, drive |
| Supermarchés | 1 200 | Caisses, back-office, balances connectées |
| Magasins de proximité et franchisés | 3 500 | Connexion simple, paiement, remontée des ventes |
| Entrepôts et plateformes logistiques | 60 | Flux WMS, scanners RF, transport |
| Sièges et centres de services | 10 | Applications métier, visioconférence |
| Datacenters et cloud | 2 + 2 régions cloud | ERP, gestion des stocks, paiement, e-commerce |

### 4.2 Les flux à gérer

- **Paiement en caisse** : critique, faible volume, extrêmement sensible à la disponibilité et à la conformité PCI-DSS.
- **Remontée des ventes et stocks** : volume modéré, tolérant à un léger délai.
- **Vidéosurveillance** : volume élevé, peu sensible à la latence.
- **Wi-Fi clients et invités** : à isoler du reste, sortie directe vers Internet.
- **Applications SaaS** (messagerie, collaboration, RH) : à envoyer directement vers le cloud.
- **Étiquettes électroniques, balances, objets connectés** : réseau dédié.
- **Drive et e-commerce** : applications en ligne à forte exigence de disponibilité.
- **Mises à jour des caisses** : gros volumes, à planifier hors des heures de pointe.

### 4.3 Les problèmes de l'ancienne architecture

- Chaque magasin a un seul MPLS avec un 4G de secours manuel.
- Le trafic Internet des Wi-Fi invités passe par le datacenter, ce qui le sature.
- Une évolution de politique réseau demande des mois pour être déployée sur 5 000 sites.
- Un incident de lien est détecté par le magasin avant d'être détecté par le NOC.

### 4.4 L'architecture cible

Chaque site reçoit un équipement SD-WAN (edge) avec :

- un lien **MPLS** (si existant) et un lien **Internet** (fibre ou xDSL) ;
- un lien **4G/5G** de secours ;
- 5 **segments** logiques (voir plus loin) ;
- une connexion automatique vers le plan de contrôle et de gestion.

Nous allons maintenant voir comment les **4 plans** rendent cela possible.

---

## 5. Vue d'ensemble : les 4 plans du SD-WAN

### 5.1 Pourquoi parler de "plans"

Un plan est un ensemble de fonctions qui répondent à une question précise. Séparer les plans permet de faire évoluer et de faire monter en charge chaque fonction indépendamment, et de mieux comprendre où chercher en cas de problème.

### 5.2 Les 4 plans

| Plan | Question à laquelle il répond | Composants (vocabulaire Cisco) | Protocoles et mécanismes clés |
|---|---|---|---|
| 1. Orchestration | Qui est autorisé à rejoindre le réseau et comment se trouve-t-on ? | Validator (vBond) | Authentification par certificats, DTLS/TLS, NAT traversal, ZTP |
| 2. Gestion | Comment configure-t-on, supervise-t-on et automatise-t-on ? | Manager (vManage) | NETCONF, API REST, templates, SNMP, syslog, télémétrie |
| 3. Contrôle | Quelles routes, quelles clés et quelles politiques s'appliquent ? | Controller (vSmart) | OMP, TLOC, distribution de clés, politiques de contrôle |
| 4. Données | Comment les paquets utilisateurs circulent-ils réellement ? | Edge (vEdge, cEdge, Catalyst 8000) | IPsec/GRE, BFD, routage applicatif, QoS, FEC |

> Remarque : certains éditeurs ou livres décrivent seulement 3 plans (gestion, contrôle, données) et rangent l'orchestration dans la gestion ou le contrôle. Le modèle à 4 plans est celui popularisé par l'architecture Cisco SD-WAN (ex-Viptela). C'est celui que nous utilisons ici car il permet de bien comprendre la phase de "premier contact" d'un équipement.

### 5.3 Schéma global

```
                      +------------------------------------------+
                      |            PLAN DE GESTION               |
                      |   Manager : templates, supervision, API  |
                      +--------------------+---------------------+
                                           |
          +--------------------------------+---------------------------+
          |                                |                           |
+---------v-----------+        +-----------v-----------+               |
|  PLAN D'ORCHESTRATION|        |    PLAN DE CONTROLE   |               |
|  Validator : accueil,|        |  Controller : OMP,    |               |
|  authentification    |        |  routes, clés, policy |               |
+---------+------------+        +-----------+-----------+               |
          |                                 |                           |
          |  (sessions de contrôle DTLS/TLS depuis chaque edge)         |
          |                                 |                           |
+---------v---------------------------------v---------------------------v--+
|                        PLAN DE DONNEES                                   |
|   Edges  <====== tunnels IPsec (overlay) ======>  Edges                  |
|   Magasin ... Hypermarché ... Entrepôt ... Datacenter ... Cloud          |
+--------------------------------------------------------------------------+
```

### 5.4 Une image simple pour retenir

Imaginons un **aéroport** :

- Le **plan d'orchestration** est le contrôle d'identité et l'accueil : on vérifie qui vous êtes avant de vous laisser entrer.
- Le **plan de gestion** est la direction de l'aéroport : elle définit les règles, les horaires et surveille l'ensemble.
- Le **plan de contrôle** est la tour de contrôle : elle dit à chaque avion quelle route suivre.
- Le **plan de données** est le trafic des avions et des passagers eux-mêmes.

Chaque plan est expliqué en détail dans les sections suivantes.

---

## 6. Plan 1 : le plan d'orchestration

### 6.1 Rôle

Le plan d'orchestration gère le **premier contact** et l'**authentification mutuelle** entre les équipements et le réseau SD-WAN. C'est lui qui permet l'**intégration automatique** (zero-touch provisioning, ZTP) d'un nouvel équipement, et qui aide les équipements à se **découvrir** malgré les NAT.

Ses fonctions :

- vérifier l'identité d'un équipement (certificat, numéro de série, liste blanche) ;
- vérifier que les contrôleurs sont légitimes (authentification mutuelle) ;
- indiquer à l'équipement où se trouvent les contrôleurs du plan de gestion et de contrôle ;
- aider à la **traversée de NAT** en révélant l'adresse IP publique et le port vus de l'extérieur (rôle similaire à STUN) ;
- distribuer la liste des contrôleurs aux équipements.

### 6.2 Composant principal

Dans le vocabulaire Cisco : le **Validator** (anciennement **vBond**). Il doit avoir une **adresse IP publique joignable** par tous les équipements, car c'est le premier point de contact.

### 6.3 Identité et confiance

- Chaque équipement possède un **certificat** (installé en usine ou généré) et un identifiant unique (numéro de série, chassis number).
- L'entreprise gère une **liste d'équipements autorisés** (plug-and-play, "whitelist").
- Un **nom d'organisation** (organization-name) identique doit être configuré partout : un équipement dont l'organisation ne correspond pas est refusé.

### 6.4 Déroulement d'un ZTP

1. L'edge démarre sans configuration et obtient une adresse IP par DHCP sur son lien Internet.
2. Il contacte un service de redirection du constructeur (ou un serveur configuré) pour apprendre l'adresse du Validator de l'entreprise.
3. Il établit une session sécurisée (DTLS ou TLS) avec le **Validator**.
4. Le Validator **vérifie le certificat** et le numéro de série de l'edge face à la liste autorisée.
5. Le Validator communique à l'edge les adresses du **Controller** et du **Manager**.
6. Le Validator observe l'adresse source de l'edge : il peut ainsi lui indiquer son adresse publique et son port (utile derrière un NAT).
7. L'edge ferme la session avec le Validator et ouvre ses sessions de contrôle vers les autres composants.

```
 Edge (nouveau)                 Validator                Controller / Manager
      |                             |                              |
      |---- DTLS/TLS + certificat ->|                              |
      |<--- vérification + liste ---|                              |
      |   (adresses des controllers)|                              |
      |                                                            |
      |----------- session de contrôle (DTLS/TLS) ---------------->|
      |<---------- configuration et politiques --------------------|
```

### 6.5 Exemple Carrefour : ouverture d'un magasin de proximité

**Situation.** Un nouveau magasin de proximité ouvre en centre-ville, avec une box fibre grand public et un routeur 4G de secours.

**Sans SD-WAN.** Un technicien doit se déplacer pour configurer le routeur, ou le routeur est préconfiguré en atelier puis expédié. Le délai est de plusieurs jours à plusieurs semaines.

**Avec SD-WAN et ZTP.**

1. Le routeur SD-WAN est livré directement au magasin avec une simple étiquette de branchement.
2. Le gérant branche le port WAN sur la box fibre et met l'équipement sous tension.
3. L'équipement obtient une IP, contacte le Validator, s'authentifie, apprend où sont le Controller et le Manager.
4. Il télécharge sa configuration (template du type "magasin de proximité"), ses 5 segments, ses politiques.
5. Au bout de quelques minutes, les caisses fonctionnent et le magasin est visible dans la supervision.

**Résultat.** Aucun ingénieur réseau sur place, ouverture en jours, déploiement homogène sur des centaines de magasins.

### 6.6 Haute disponibilité et placement

- Prévoir **au moins 2 Validators** dans des datacenters ou régions cloud différents, avec **plusieurs adresses IP** ou un nom DNS à plusieurs entrées.
- Les placer dans une zone accessible depuis Internet (DMZ), protégés par pare-feu avec des règles strictes sur les ports requis.
- Surveiller les certificats : un certificat expiré peut empêcher l'accès au réseau.

### 6.7 Erreurs fréquentes liées à ce plan

| Symptôme | Cause probable |
|---|---|
| L'edge n'établit jamais de session | Pare-feu amont bloquant le port de contrôle, DNS du Validator non résolu |
| L'edge est rejeté après contact | Numéro de série absent de la liste autorisée, certificat invalide, organization-name différent |
| Session instable derrière un NAT | NAT symétrique agressif, timeouts NAT trop courts |
| Mauvaise date sur l'edge | Heure incorrecte, les certificats paraissent expirés ou non encore valides |

### 6.8 À retenir

- Le plan d'orchestration est le **portier** du réseau.
- Il garantit l'**authentification mutuelle** et rend possible le **zero-touch**.
- Il aide à franchir les **NAT**.

---

## 7. Plan 2 : le plan de gestion (management)

### 7.1 Rôle

Le plan de gestion est le **cockpit** de l'administrateur. Il permet de :

- **configurer** les équipements à grande échelle via des **modèles (templates)** ;
- **superviser** l'état des sites, des liens et des applications ;
- **déployer** des mises à jour logicielles ;
- **gérer les accès** des administrateurs (RBAC) ;
- **exposer des API** pour l'automatisation et l'intégration avec les outils d'entreprise (ITSM, CMDB, CI/CD).

### 7.2 Composant principal

Dans le vocabulaire Cisco : le **Manager** (anciennement **vManage**). Il offre une interface web, une API REST et communique avec les équipements en général via **NETCONF** sur la session de contrôle chiffrée.

### 7.3 Fonctions détaillées

**Configuration par modèles.**
- Un **template de fonctionnalité** décrit un bloc (interface, VPN, routage, QoS).
- Un **template d'équipement** assemble plusieurs blocs pour un profil de site.
- Des **variables** (adresse IP LAN, nom du site, identifiant) personnalisent le modèle pour chaque site, souvent importées depuis un fichier CSV ou une API.

**Supervision.**
- Tableau de bord global : nombre de sites actifs, liens dégradés, alarmes.
- Vue par application : latence, perte et gigue observées.
- Vue par lien : disponibilité des transports MPLS, Internet, 4G/5G.
- Journalisation (syslog), alertes (e-mail, webhook), export vers des outils externes (SIEM, outils de supervision).

**Cycle de vie logiciel.**
- Téléchargement des images, **déploiement par vagues** (pilote, puis groupes de sites), retour arrière.

**Gestion des identités et accès.**
- Rôles et périmètres : ingénieur réseau, support N1, auditeur sécurité, partenaire externe.
- Intégration SSO (SAML, LDAP, RADIUS).

**API et automatisation.**
- Création de sites, modification de politiques, extraction de métriques via REST.
- Intégration avec Ansible, Terraform, Python, ServiceNow.

### 7.4 Exemple Carrefour : modèles par type de site

Le groupe définit 4 **profils** :

| Profil | Transports | Segments actifs | Particularités |
|---|---|---|---|
| Hyper | MPLS + 2 Internet + 5G | Les 5 | Double edge en haute disponibilité, sécurité avancée locale |
| Super | MPLS + Internet + 5G | Les 5 | Un edge, Wi-Fi invités en sortie locale |
| Proximité | Internet + 4G | Caisses, back-office, invités | Edge compact, pas de MPLS |
| Entrepôt | MPLS + Internet + 5G | Back-office, IoT, scanners RF | QoS forte pour les flux WMS |

Pour ouvrir un nouveau magasin, l'équipe réseau renseigne quelques variables (nom, site-id, plan d'adressage LAN) et rattache l'équipement au profil. Le Manager pousse la configuration complète.

### 7.5 Exemple Carrefour : une opération à grande échelle

**Situation.** Il faut modifier le DNS interne et activer une nouvelle règle de sécurité sur les 5 000 sites.

**Approche.**
1. Modifier le template unique concerné dans le Manager.
2. Valider en **pilote** sur 20 magasins.
3. Déployer par **vagues** de 500 sites, avec contrôle automatique après chaque vague.
4. En cas d'anomalie, **retour arrière** à la version précédente du template.

**Gain.** Quelques heures au lieu de semaines, avec un résultat uniforme et traçable.

### 7.6 Exemple Carrefour : préparer le Black Friday

- Création d'un **tableau de bord** dédié aux sites e-commerce, drive et entrepôts.
- **Alertes** si la perte dépasse 1 % sur le lien principal d'un entrepôt.
- **Gel des changements** : verrouillage des modifications de template pendant la période sensible, sauf pour les administrateurs désignés.

### 7.7 Bonnes pratiques

- Versionner les templates (Git ou export régulier).
- Séparer les rôles : qui peut modifier, qui peut déployer, qui peut seulement lire.
- Sauvegarder régulièrement la base de configuration du Manager.
- Dimensionner le Manager (nombre d'équipements, rétention des statistiques).
- Prévoir un cluster pour la haute disponibilité à grande échelle.

### 7.8 À retenir

- Le plan de gestion est le **centre de pilotage humain et automatisé**.
- Il ne transporte pas le trafic utilisateur : une panne du Manager n'interrompt pas le trafic existant, mais empêche les changements et la visibilité.

---

## 8. Plan 3 : le plan de contrôle

### 8.1 Rôle

Le plan de contrôle est le **cerveau** du réseau. Il :

- **apprend** les routes de chaque site (préfixes LAN, routes connectées, routes apprises par OSPF ou BGP) ;
- **distribue** ces routes aux autres sites selon la politique ;
- **distribue les clés de chiffrement** utilisées par les tunnels du plan de données ;
- **applique les politiques centralisées** (qui peut parler à qui, par quel chemin) ;
- maintient la **topologie logique** (full mesh, hub-and-spoke, partielle).

### 8.2 Composant principal

Dans le vocabulaire Cisco : le **Controller** (anciennement **vSmart**). Chaque edge maintient une **session de contrôle** (DTLS ou TLS) avec un ou plusieurs Controllers.

### 8.3 Le protocole OMP

**OMP (Overlay Management Protocol)** est le protocole de routage de l'overlay, fonctionnant entre les edges et les Controllers. Il s'inspire de BGP mais est conçu pour le SD-WAN.

OMP transporte trois grands types d'informations :

| Type | Contenu | Utilité |
|---|---|---|
| **Routes OMP** | Préfixes appris dans un segment (VPN) | Savoir où se trouve un réseau de destination |
| **Routes TLOC** | Points de terminaison de transport : IP, couleur, encapsulation | Savoir par quel lien atteindre un site |
| **Routes de service** | Services disponibles (pare-feu, IDS, passerelle) | Chaîner des services dans le chemin |

Il transporte aussi les **clés de chiffrement** utilisées pour les tunnels IPsec, et les **politiques de contrôle**.

### 8.4 Le concept de TLOC

Un **TLOC (Transport Locator)** identifie un point d'accès au transport d'un site. Il est composé de :

- l'**adresse système** (system-IP) de l'équipement ;
- la **couleur** du transport (par exemple `mpls`, `biz-internet`, `public-internet`, `lte`) ;
- l'**encapsulation** (IPsec ou GRE).

Un site avec MPLS, Internet et 5G annonce **3 TLOC**. Les autres sites décident, selon la politique, quels TLOC utiliser pour joindre chaque destination.

### 8.5 Les couleurs

Les couleurs servent à **qualifier** le type de transport. Deux sites ne peuvent construire un tunnel direct que s'ils respectent les règles de **restriction** de couleur : par défaut, les couleurs "privées" (MPLS) ne se connectent qu'à la même couleur, tandis que les couleurs "publiques" peuvent se connecter entre elles.

### 8.6 La segmentation (VPN / VRF)

Le plan de contrôle gère la **segmentation** de bout en bout. Chaque **VPN** (équivalent d'un VRF) isole un type de trafic et possède sa propre table de routes.

Conventions courantes dans l'écosystème Cisco :

- **VPN 0** : transport (les liens WAN, la découverte des contrôleurs) ;
- **VPN 512** : gestion hors bande des équipements ;
- **VPN 1 à 511** : réseaux de service, côté LAN.

**Exemple Carrefour : 5 segments de service.**

| VPN | Usage | Exigences |
|---|---|---|
| 10 | Caisses et paiement | Conformité PCI-DSS, aucune sortie Internet directe |
| 20 | Back-office et applications métier | Accès aux applications du datacenter et SaaS autorisés |
| 30 | Wi-Fi clients et invités | Sortie locale Internet uniquement, pas d'accès au réseau interne |
| 40 | Vidéosurveillance | Faible priorité, flux vers un stockage centralisé |
| 50 | IoT : balances, étiquettes électroniques | Communication limitée à des serveurs précis |

Un client du VPN 30 ne peut techniquement pas atteindre le VPN 10, même en étant sur le même magasin.

### 8.7 Les politiques centralisées

Elles sont définies dans le Manager et appliquées via le Controller. On distingue :

**Politiques de contrôle (control policy).** Elles agissent sur les **routes** annoncées par OMP. Exemples :
- **Topologie hub-and-spoke** : les magasins ne voient que les hubs (datacenters), pas les autres magasins.
- **Topologie partielle** : les entrepôts d'une même région se voient entre eux.
- **Préférence de chemin** : préférer le TLOC MPLS d'un hub à son TLOC Internet.

**Politiques de données (data policy).** Elles agissent sur les **flux** : classement et redirection selon l'application, la source, la destination. Elles sont poussées vers les edges pour exécution.

**Politiques de routage applicatif (application-aware routing).** Elles définissent des **SLA** (latence, perte, gigue maximales) par classe d'application, et les chemins autorisés.

> Les politiques sont **décidées** dans le plan de contrôle et de gestion, mais **exécutées** dans le plan de données.

### 8.8 Exemple Carrefour : topologie et politiques

**Topologie.**
- Les **magasins** forment des **spokes** : ils sont connectés aux 2 **hubs** (datacenters) par des tunnels.
- Les **hypermarchés** d'une même région peuvent se parler en direct pour des services partagés (par exemple un centre de préparation de commandes drive).
- Les **entrepôts** sont connectés aux hubs et entre eux.

**Politique de segmentation.**
- Le VPN 10 (caisses) n'est routé **que vers** le datacenter de paiement.
- Le VPN 30 (Wi-Fi invités) n'est **pas annoncé** aux autres sites : il n'est visible que localement.

**Politique de chemin.**
- Le trafic de paiement privilégie MPLS, avec bascule automatique vers Internet si le SLA n'est plus respecté.
- Le trafic SaaS (messagerie, collaboration) privilégie la **sortie locale Internet** (DIA, Direct Internet Access).

### 8.9 Exemple Carrefour : la panne d'un lien

Un magasin perd son MPLS. Le plan de contrôle n'est pas "réécrit" pour autant :

1. Les tunnels sur ce lien tombent (détectés par le plan de données via BFD).
2. L'edge retire les TLOC concernés de ses annonces OMP.
3. Les autres sites mettent à jour leurs préférences et utilisent les TLOC Internet.
4. Les flux de paiement continuent via Internet, en quelques secondes.

### 8.10 Passage à l'échelle

- **Plusieurs Controllers** sont déployés (au moins 2, souvent 4 ou plus) pour la redondance et la répartition de charge. Chaque edge maintient des sessions avec plusieurs d'entre eux.
- Si **tous** les Controllers sont indisponibles, les edges **continuent** de fonctionner avec la dernière table de routes connue pendant une durée configurable (graceful restart OMP). Les changements de topologie ne sont toutefois plus appris.

### 8.11 Erreurs fréquentes liées à ce plan

| Symptôme | Cause probable |
|---|---|
| Un site n'est pas joignable depuis les autres | Préfixe non annoncé en OMP, route filtrée par politique de contrôle |
| Un site voit un autre site mais pas son LAN | Segment (VPN) manquant ou mal configuré d'un côté |
| Trafic qui passe par le mauvais chemin | Politique de préférence de TLOC mal écrite |
| Session de contrôle qui tombe régulièrement | Instabilité du lien, pare-feu amont, certificat proche de l'expiration |

### 8.12 À retenir

- Le plan de contrôle est le **cerveau** qui dit **où est quoi** et **comment y aller**.
- Il repose sur **OMP**, les **TLOC** et les **politiques**.
- Il assure la **segmentation** et la **distribution des clés**.

---

## 9. Plan 4 : le plan de données (data plane)

### 9.1 Rôle

Le plan de données transporte le **trafic utilisateur** réel : paiements, vidéos, e-mails, flux applicatifs. Il **exécute** les décisions prises par les autres plans.

Ses fonctions :

- **établir les tunnels** IPsec (ou GRE) entre les edges, avec les clés fournies par le plan de contrôle ;
- **chiffrer et déchiffrer** le trafic ;
- **mesurer en continu** la qualité de chaque tunnel ;
- **choisir le chemin** pour chaque flux selon les politiques ;
- appliquer la **QoS**, la mise en forme du trafic, la correction d'erreurs ;
- intégrer des fonctions **de sécurité** locales (pare-feu, filtrage, IPS) selon l'équipement.

### 9.2 Composant principal

Les **edges** (équipements de bordure) : routeurs physiques, équipements virtuels (VNF) dans un hyperviseur ou un cloud, ou instances logicielles dans les clouds publics.

### 9.3 Les tunnels

- Les edges construisent des tunnels **IPsec** entre leurs TLOC.
- Les clés sont **distribuées par le Controller** : les edges n'ont pas besoin de négocier des associations de sécurité par un protocole lourd comme IKE entre chaque paire de sites.
- Les tunnels peuvent traverser des **NAT** grâce aux informations apprises via le plan d'orchestration.

Selon la topologie de la politique de contrôle, les tunnels sont établis en **hub-and-spoke**, en **full mesh** ou en **maillage partiel**. La possibilité de créer des tunnels **à la demande** entre sites (dynamic tunnels) dépend de l'éditeur et de la version.

### 9.4 La mesure de qualité : BFD

**BFD (Bidirectional Forwarding Detection)** tourne à l'intérieur de chaque tunnel. Il envoie des sondes régulières pour :

- détecter rapidement une **panne** (liveness) ;
- mesurer **latence, gigue et perte** sur chaque chemin.

À titre indicatif (valeurs par défaut dans l'écosystème Cisco, à vérifier sur la version utilisée) :

- intervalle d'envoi de sondes : environ 1 seconde ;
- multiplicateur de détection : environ 7 (un tunnel est considéré hors service après plusieurs sondes perdues) ;
- fenêtre de mesure applicative : de l'ordre de plusieurs minutes, avec un historique glissant.

Ces valeurs se règlent pour trouver le bon compromis entre **réactivité** et **stabilité**.

### 9.5 Le routage applicatif (Application-Aware Routing)

L'edge **identifie l'application** (par inspection de paquets, par signatures ou par ports) et la classe selon la politique. Il choisit ensuite, parmi les tunnels qui **respectent le SLA**, celui prévu par la politique.

**Exemple d'une politique.**

| Classe d'application | SLA maximum | Chemin préféré | Chemin de secours |
|---|---|---|---|
| Paiement en caisse | Latence 100 ms, perte 1 %, gigue 30 ms | MPLS | Internet |
| Voix et visio | Latence 150 ms, perte 1 %, gigue 30 ms | Meilleur chemin disponible | Tous |
| SaaS (messagerie, collaboration) | Latence 200 ms, perte 3 % | Sortie locale Internet | MPLS vers datacenter |
| Vidéosurveillance | Pas de SLA strict | Internet | MPLS en dernier recours |
| Sauvegardes et mises à jour | Pas de SLA | Internet, heures creuses | Aucun |

Si le tunnel préféré ne respecte plus le SLA, le flux est **déplacé** vers un tunnel conforme, sans coupure perceptible pour des applications bien conçues.

### 9.6 Qualité de service (QoS)

- **Classification** : marquage DSCP des paquets selon l'application.
- **Files d'attente** : priorité stricte pour la voix, bande passante garantie pour les applications critiques, "best effort" pour le reste.
- **Mise en forme (shaping)** : adaptation du débit émis à la capacité réelle du lien, pour éviter la congestion chez l'opérateur.

### 9.7 Techniques d'amélioration de la qualité

- **FEC (Forward Error Correction)** : ajout de paquets de redondance pour reconstruire des paquets perdus sans retransmission.
- **Duplication de paquets** : envoi du même paquet sur deux liens pour les flux très critiques.
- **Optimisation TCP** et **compression** (selon les éditeurs).

### 9.8 Sortie directe vers Internet (DIA) et vers le cloud

L'edge peut envoyer certains flux **directement** vers Internet au lieu de les renvoyer vers un datacenter. Cela réduit la latence vers les SaaS et soulage les liens MPLS. La sécurité est alors assurée localement (pare-feu intégré) ou via un **service de sécurité cloud** (SWG, CASB, SASE).

Il est aussi possible de déployer des **edges virtuels** dans les clouds publics pour joindre des applications IaaS depuis l'overlay SD-WAN (cloud on-ramp).

### 9.9 Exemple Carrefour : une journée dans un supermarché

| Moment | Ce qui se passe dans le plan de données |
|---|---|
| 9 h | Les caisses envoient les paiements : tunnel MPLS, priorité haute, QoS garantie. |
| 10 h | Un employé consulte la messagerie : le trafic sort localement vers Internet (DIA). |
| 11 h | Un client utilise le Wi-Fi invité : le trafic sort par Internet via un segment isolé, sans toucher aux caisses. |
| 12 h | La perte sur le MPLS passe à 4 % (dégradation chez l'opérateur) : les flux de paiement basculent vers Internet automatiquement, car le SLA n'est plus respecté. |
| 14 h | La vidéosurveillance envoie ses enregistrements vers le stockage central : classe basse priorité, sur Internet. |
| 22 h | Mise à jour des logiciels de caisse : téléchargement en heures creuses, bande passante limitée. |

### 9.10 Exemple Carrefour : entrepôt et scanners RF

- Les scanners RF et le WMS exigent une **latence faible et stable** : politique de routage applicatif stricte, avec **duplication de paquets** sur MPLS et Internet pour les transactions critiques.
- Les flux de reporting, moins sensibles, utilisent le lien le moins coûteux.

### 9.11 Chiffrement et sécurité du plan de données

- Chiffrement des tunnels par **IPsec** avec des algorithmes modernes (AES-256-GCM par exemple).
- **Rotation régulière des clés**, gérée par le plan de contrôle.
- Les flux restent **segmentés** de bout en bout : chaque segment utilise ses propres identifiants de VPN dans l'overlay.

### 9.12 Erreurs fréquentes liées à ce plan

| Symptôme | Cause probable |
|---|---|
| Session de contrôle OK mais aucun tunnel de données | Ports IPsec bloqués par un pare-feu, problème de NAT, restriction de couleurs |
| Tunnels qui montent et descendent | Lien instable, MTU mal ajustée, seuils BFD trop agressifs |
| Lenteurs inexpliquées sur certaines applications | MTU/MSS inadaptés avec l'encapsulation IPsec, congestion sans shaping |
| Application sur un mauvais lien | Politique de routage applicatif absente ou mal ciblée |

### 9.13 À retenir

- Le plan de données est le **muscle** du réseau : il fait réellement passer les paquets.
- Il **exécute** les politiques, **mesure** la qualité et **bascule** les flux.
- Il continue de fonctionner même si les plans de gestion et d'orchestration sont temporairement indisponibles.

---

## 10. Les 4 plans ensemble : scénarios de bout en bout

### 10.1 Tableau comparatif

| Critère | Orchestration | Gestion | Contrôle | Données |
|---|---|---|---|---|
| Métaphore | Contrôle d'identité | Direction | Tour de contrôle | Avions et passagers |
| Composant Cisco | Validator | Manager | Controller | Edge |
| Protocoles | DTLS/TLS, certificats | NETCONF, REST, SNMP | OMP | IPsec, BFD |
| Transporte du trafic utilisateur | Non | Non | Non | Oui |
| Impact si indisponible | Pas de nouveaux équipements, pas de reconnexion | Plus de changements ni de supervision, trafic maintenu | Plus d'apprentissage de routes, le trafic existant continue un temps | Plus de trafic utilisateur |
| Exemple de panne | Certificat expiré | Base de données saturée | Controllers injoignables | Lien opérateur coupé |

### 10.2 Scénario A : ouverture d'un nouveau magasin de bout en bout

1. **Gestion.** L'équipe réseau déclare le site, ses variables et son profil dans le Manager.
2. **Orchestration.** L'edge arrive, contacte le Validator, s'authentifie, apprend l'adresse des contrôleurs.
3. **Gestion.** Le Manager envoie la configuration (templates).
4. **Contrôle.** L'edge établit sa session avec le Controller, annonce ses préfixes et ses TLOC, reçoit les routes et les clés.
5. **Données.** Les tunnels IPsec se montent vers les hubs, les mesures BFD démarrent, les caisses fonctionnent.

### 10.3 Scénario B : dégradation d'un lien MPLS

1. **Données.** BFD mesure une perte de 5 % sur le tunnel MPLS.
2. **Données.** La politique de routage applicatif détecte la violation de SLA et bascule le flux de paiement sur Internet.
3. **Gestion.** Le Manager remonte l'alarme au NOC, qui ouvre un ticket chez l'opérateur.
4. **Contrôle.** Si le tunnel tombe complètement, les TLOC associés sont retirés des annonces OMP.

### 10.4 Scénario C : perte de connectivité vers les contrôleurs

Un incident rend les Controllers injoignables pendant 30 minutes.

- **Données.** Les tunnels existants et le trafic **continuent** de fonctionner.
- **Contrôle.** Les edges utilisent les dernières routes connues (graceful restart).
- **Gestion.** La supervision n'est plus à jour, les changements sont impossibles.
- **Risque.** Si un lien tombe pendant la panne, la convergence peut être limitée car les nouvelles annonces ne sont plus échangées normalement.

C'est pourquoi on **déploie plusieurs Controllers** dans des zones indépendantes.

### 10.5 Scénario D : déploiement d'une nouvelle politique de sécurité

1. **Gestion.** L'équipe sécurité écrit la politique (par exemple, interdire certains flux inter-segments) dans le Manager.
2. **Contrôle.** Le Controller applique la politique de contrôle et la propage.
3. **Données.** Les edges appliquent la politique de données localement.
4. **Gestion.** Un tableau de bord vérifie que les flux interdits sont bien bloqués.

---

## 11. Correspondance de vocabulaire entre éditeurs

Les concepts sont proches d'un éditeur à l'autre, mais les noms changent. Le tableau est donné à titre indicatif, car les produits et leurs noms évoluent régulièrement.

| Fonction | Cisco (Catalyst SD-WAN) | VMware VeloCloud | Fortinet | Palo Alto Prisma SD-WAN | HPE Aruba EdgeConnect |
|---|---|---|---|---|---|
| Orchestration (accueil, ZTP) | Validator (vBond) | Orchestrator (avec ZTP) | FortiGate ZTP via FortiManager | Contrôleur cloud | Orchestrator |
| Gestion | Manager (vManage) | Orchestrator | FortiManager | Console cloud | Orchestrator |
| Contrôle | Controller (vSmart), OMP | Gateways et Orchestrator | Fonctions intégrées aux FortiGate | Contrôleur | Orchestrator |
| Données | Edge (cEdge, vEdge) | Edge | FortiGate | ION | EdgeConnect |

> Remarque : certains éditeurs fusionnent plusieurs plans dans un même composant (par exemple l'orchestrateur gère à la fois orchestration, gestion et une partie du contrôle). La **logique des 4 plans** reste utile pour raisonner, même quand l'implémentation diffère.

---

## 12. Sécurité, segmentation et SASE

### 12.1 Les couches de sécurité d'un SD-WAN

1. **Sécurité de l'accès au réseau** : authentification mutuelle par certificats, liste autorisée d'équipements.
2. **Sécurité du transport** : chiffrement IPsec, rotation des clés.
3. **Sécurité de la segmentation** : isolation des VPN, politiques inter-segments.
4. **Sécurité applicative et des flux sortants** : pare-feu, filtrage URL, IPS, anti-malware, DNS sécurisé.
5. **Sécurité de la gestion** : RBAC, journalisation, authentification forte pour les administrateurs.

### 12.2 Exemple Carrefour : conformité PCI-DSS

Les environnements de paiement doivent être isolés et protégés. Le SD-WAN y contribue par :

- un **segment dédié** (VPN 10) pour les caisses et terminaux de paiement ;
- une **absence de route** vers Internet pour ce segment ;
- un **chiffrement** systématique des flux entre le magasin et le datacenter de paiement ;
- une **traçabilité** des changements de configuration (audit du Manager).

> Le SD-WAN ne rend pas à lui seul une architecture conforme PCI-DSS. Il fournit des briques de segmentation et de chiffrement qui doivent s'intégrer dans une démarche globale de conformité.

### 12.3 SASE et sécurité dans le cloud

**SASE (Secure Access Service Edge)** combine fonctions réseau (SD-WAN) et fonctions de sécurité fournies dans le cloud : SWG (passerelle web sécurisée), CASB, ZTNA, pare-feu en tant que service (FWaaS).

**Exemple Carrefour.**
- Les magasins sortent en direct vers Internet pour les flux SaaS.
- Leur trafic web est **redirigé par tunnel** vers un service de sécurité cloud pour inspection.
- Les sièges adoptent le même modèle, ce qui évite de renvoyer tout le trafic vers le datacenter.

### 12.4 Zero Trust

Principe : ne jamais faire confiance par défaut, vérifier à chaque accès. En SD-WAN, cela se traduit par :

- une **identité** pour chaque équipement et chaque segment ;
- le **moindre privilège** dans les politiques inter-segments ;
- une **vérification continue** de la posture et du comportement.

---

## 13. Conception : topologies, haute disponibilité, dimensionnement

### 13.1 Choisir une topologie

| Topologie | Avantages | Inconvénients | Cas d'usage Carrefour |
|---|---|---|---|
| Hub-and-spoke | Simple, contrôle central, peu de tunnels | Les flux inter-sites passent par le hub | Magasins vers datacenters |
| Full mesh | Latence minimale entre sites | Beaucoup de tunnels, charge sur les edges | Quelques hypermarchés d'une région |
| Mesh partiel ou régional | Compromis | Plus complexe à décrire | Entrepôts d'une même zone |
| Tunnels dynamiques | Tunnels créés à la demande | Dépend du support de l'éditeur | Flux inter-magasins occasionnels |

### 13.2 Haute disponibilité

**Côté site.**
- **Double edge** pour les sites critiques (hypermarchés, entrepôts) avec protocole de redondance (VRRP, ou équivalent).
- **Double lien** au minimum, idéalement de **deux opérateurs** différents et avec des technologies différentes (fibre, 5G).
- **Alimentation** secourue pour les équipements de caisse et réseau.

**Côté contrôleurs.**
- **2 Validators minimum**, répartis dans des sites indépendants.
- **Cluster Manager** pour la tolérance aux pannes.
- **Plusieurs Controllers** répartis en deux régions.

**Côté hubs.**
- Au moins **2 hubs** dans des datacenters distincts, avec des politiques de préférence claires pour éviter les chemins asymétriques.

### 13.3 Dimensionnement

Points à analyser :

- **Nombre d'équipements** par contrôleur (limites constructeur).
- **Débit chiffré** réel des edges par type de site (le chiffrement IPsec consomme du CPU).
- **Nombre de tunnels** par edge selon la topologie.
- **Rétention des statistiques** dans le Manager.
- **Capacité des hubs** : un hub agrège les tunnels de milliers de spokes.

**Exemple Carrefour.** Un datacenter hub doit supporter des milliers de tunnels. On choisit des edges haut de gamme, redondés, avec un dimensionnement avec marge de 30 à 50 % sur les pics (Black Friday, soldes).

### 13.4 Adressage et plan d'identifiants

Définir à l'avance :

- un **plan d'adressage LAN** unique par site, résumable par région ;
- un **system-IP** unique par équipement ;
- un **site-id** structuré (par exemple pays, région, type, numéro) ;
- une **convention de nommage** (par exemple `FR-IDF-HYP-0042`).

### 13.5 MTU et fragmentation

L'encapsulation IPsec ajoute un en-tête (de l'ordre de 50 à 100 octets selon la configuration). Il faut :

- ajuster la **MTU des interfaces de tunnel** ;
- configurer le **TCP MSS adjust** ;
- tester avec de gros paquets sur des liens de différents opérateurs (4G/5G en particulier).

---

## 14. Déploiement et migration depuis MPLS

### 14.1 Étapes d'un projet

1. **Cadrage** : objectifs, périmètre, contraintes de sécurité et de conformité.
2. **Audit** : inventaire des sites, liens, applications, volumétrie, SLA existants.
3. **Conception** : architecture, topologie, segmentation, politiques, plan d'adressage.
4. **Pilote** : quelques sites représentatifs (un hyper, un super, un magasin de proximité, un entrepôt).
5. **Industrialisation** : templates, scripts, procédures de ZTP, formation des équipes.
6. **Déploiement par vagues** : par région ou par type de site.
7. **Optimisation** : réglage des politiques à partir des données réelles.
8. **Décommissionnement partiel du MPLS** si le business case le justifie.

### 14.2 Coexistence SD-WAN et MPLS

Pendant la migration :

- le MPLS peut rester le **transport principal** avec Internet en complément ;
- certains sites sont **encore sur l'ancien réseau**, d'autres déjà sur SD-WAN : il faut un **point d'interconnexion** (passerelle entre MPLS legacy et overlay SD-WAN) avec une redistribution de routes maîtrisée ;
- éviter les **boucles de routage** et les routes asymétriques lors de la redistribution.

### 14.3 Exemple Carrefour : stratégie de vagues

| Vague | Sites | Objectif |
|---|---|---|
| 0 | Laboratoire et 5 sites pilotes | Valider l'architecture et les politiques |
| 1 | 200 magasins de proximité d'une région | Valider le ZTP à grande échelle |
| 2 | Supermarchés | Valider la haute disponibilité et la segmentation |
| 3 | Hypermarchés | Sites critiques, double edge, fenêtres de bascule strictes |
| 4 | Entrepôts et sièges | Politiques applicatives avancées |

### 14.4 Gestion du changement

- **Communication** vers les magasins : une opération de bascule est planifiée hors des horaires d'ouverture.
- **Plan de retour arrière** pour chaque vague.
- **Support renforcé** (hypercare) dans les jours suivant la bascule.
- **Formation** des équipes NOC et support de niveau 1.

---

## 15. Exploitation, supervision et dépannage

### 15.1 Méthode de dépannage par plan

Face à un incident, on se pose les questions dans cet ordre :

1. **Orchestration** : l'équipement a-t-il pu s'authentifier et trouver les contrôleurs ?
2. **Gestion** : l'équipement est-il visible et configuré dans le Manager ?
3. **Contrôle** : la session de contrôle est-elle établie ? Les routes OMP sont-elles reçues ?
4. **Données** : les tunnels sont-ils montés ? BFD est-il stable ? Le trafic emprunte-t-il le chemin attendu ?

### 15.2 Commandes utiles (équipements Cisco IOS XE SD-WAN)

Les commandes exactes varient selon la version. Voici des exemples courants :

```
show sdwan control connections          ! état des sessions vers Validator, Controller, Manager
show sdwan control local-properties     ! identité locale, certificat, couleurs
show sdwan omp summary                  ! résumé OMP
show sdwan omp routes                   ! routes apprises par OMP
show sdwan omp tlocs                    ! TLOC annoncés et reçus
show sdwan bfd sessions                 ! état des tunnels et mesures BFD
show sdwan app-route stats              ! statistiques de routage applicatif
show sdwan running-config               ! configuration SD-WAN active
```

### 15.3 Tableau de diagnostic rapide

| Observation | Plan probable | Pistes |
|---|---|---|
| L'equipement n'apparaît jamais | Orchestration | Connectivité Internet, DNS, pare-feu, certificat, liste autorisée |
| Équipement visible mais configuration non appliquée | Gestion | Template non rattaché, variables manquantes, erreur de validation |
| Session de contrôle "connect" mais pas de routes | Contrôle | Politique de contrôle, VPN absent, filtrage OMP |
| Routes présentes mais pas de tunnel | Données | Ports bloqués, NAT, restriction de couleurs |
| Tunnels OK mais application lente | Données | QoS, MTU/MSS, congestion, SLA mal calibrés |

### 15.4 Indicateurs à suivre (KPI)

- **Disponibilité des sites** et des liens (en pourcentage).
- **Latence, gigue, perte** par lien et par application.
- **Temps de bascule** lors d'une panne de lien.
- **Taux d'utilisation** de chaque transport (pour optimiser les coûts).
- **Délai de mise en service** d'un nouveau site (objectif : quelques jours).
- **Nombre d'incidents** par plan sur une période.

### 15.5 Exemple Carrefour : organisation du NOC

- **Niveau 1** : surveille les tableaux de bord et traite les alarmes standards (lien down, site hors ligne).
- **Niveau 2** : analyse approfondie (OMP, BFD, politiques).
- **Niveau 3 et architecture** : modifications de politiques, évolution de topologie.
- **Astreinte** renforcée lors des pics d'activité commerciaux.

---

## 16. Exemple de configuration (style Cisco)

> Exemple pédagogique simplifié, à adapter. Les noms, adresses et valeurs sont fictifs. Une configuration réelle se pilote en général via les templates du Manager.

```
system
 host-name           FR-IDF-SUP-0123
 system-ip           10.255.10.123
 site-id             10123
 organization-name   "RETAIL-DEMO"
 sp-organization-name "RETAIL-DEMO"
 vbond vbond.retail-demo.example.com
!
vrf definition 10
 description CAISSES_PAIEMENT
!
vrf definition 20
 description BACK_OFFICE
!
vrf definition 30
 description WIFI_INVITES
!
interface GigabitEthernet0/0/0
 description WAN_FIBRE_INTERNET
 ip address dhcp
 tunnel-interface
  encapsulation ipsec
  color biz-internet
  allow-service dhcp
  allow-service dns
  allow-service icmp
!
interface GigabitEthernet0/0/1
 description WAN_MPLS
 ip address 192.0.2.2 255.255.255.252
 tunnel-interface
  encapsulation ipsec
  color mpls restrict
!
interface GigabitEthernet0/0/2
 description LAN_CAISSES
 vrf forwarding 10
 ip address 10.10.123.1 255.255.255.0
!
interface GigabitEthernet0/0/3
 description LAN_BACKOFFICE
 vrf forwarding 20
 ip address 10.20.123.1 255.255.255.0
!
omp
 no shutdown
 graceful-restart
 advertise connected
 advertise ospf external
```

Points à noter :

- la **couleur** et l'**encapsulation** définissent les TLOC annoncés ;
- `restrict` sur la couleur `mpls` limite les tunnels à des liens de même couleur ;
- chaque `vrf` correspond à un **segment** de l'overlay ;
- la section `omp` déclenche l'annonce des préfixes du site vers le plan de contrôle.

---

## 17. Exercices et corrigés

### Exercice 1 : associer plans et fonctions

Associez chaque fonction à son plan (orchestration, gestion, contrôle, données).

a. Distribuer les routes entre sites.
b. Valider le certificat d'un nouvel équipement.
c. Chiffrer les paquets d'une caisse.
d. Déployer un template sur 500 sites.
e. Mesurer la latence d'un tunnel.
f. Aider à la traversée de NAT.
g. Distribuer les clés de chiffrement des tunnels.
h. Fournir une API REST pour l'automatisation.

**Corrigé.**
a. Contrôle. b. Orchestration. c. Données. d. Gestion. e. Données. f. Orchestration. g. Contrôle. h. Gestion.

### Exercice 2 : choix de topologie

Le groupe veut que les 3 500 magasins de proximité accèdent aux applications du datacenter, mais ne communiquent pas entre eux. Quelle topologie et quel type de politique ?

**Corrigé.** Topologie **hub-and-spoke** : les magasins sont des spokes reliés aux hubs. On met en oeuvre une **politique de contrôle** qui filtre les routes pour que les spokes ne reçoivent que les routes des hubs.

### Exercice 3 : analyse de panne

Un magasin voit son état "contrôle connecté" dans le Manager, les routes OMP sont reçues, mais aucune session BFD n'est active et les caisses sont injoignables. Dans quel plan chercher en priorité et quelles causes explorer ?

**Corrigé.** Plan de **données**. Causes probables : ports IPsec bloqués par un pare-feu ou une box, problème de NAT, restriction de couleurs entre les TLOC, MTU. On vérifie `show sdwan bfd sessions` et les règles du pare-feu amont.

### Exercice 4 : écrire une politique applicative

Définissez en langage naturel une politique pour les flux de paiement et pour les mises à jour logicielles.

**Corrigé (exemple).**
- Paiement : classe "critique", SLA latence 100 ms, perte 1 %, gigue 30 ms, chemin préféré MPLS, bascule sur Internet si le SLA est violé.
- Mises à jour : classe "bulk", pas de SLA, chemin Internet, bande passante limitée à 20 % du lien, uniquement entre 22 h et 5 h.

### Exercice 5 : segmentation

Le Wi-Fi des clients d'un hypermarché doit rester isolé du réseau interne et sortir directement sur Internet. Quelles mesures prendre ?

**Corrigé.**
- Créer un **segment dédié** (VPN 30) pour le Wi-Fi invités.
- **Ne pas annoncer** ce VPN en OMP vers les autres sites.
- Configurer une **sortie locale Internet (DIA)** pour ce VPN, avec NAT.
- Appliquer un **filtrage** et une inspection de sécurité (pare-feu local ou service cloud).
- Interdire toute route entre VPN 30 et les VPN internes.

### Exercice 6 : QCM

1. Quel composant est le premier point de contact d'un edge en ZTP ?
   a) Manager b) Validator c) Controller d) Un autre edge
2. Quel protocole distribue les routes de l'overlay ?
   a) OSPF b) BGP uniquement c) OMP d) RIP
3. Que devient le trafic si le Manager devient indisponible ?
   a) Il est coupé b) Il continue c) Il est redirigé vers MPLS d) Il est chiffré en clair
4. À quoi sert BFD dans les tunnels ?
   a) Chiffrer b) Mesurer la qualité et détecter les pannes c) Distribuer les routes d) Authentifier les administrateurs
5. Combien de plans compte le modèle étudié ici ?
   a) 2 b) 3 c) 4 d) 5

**Corrigé.** 1b, 2c, 3b, 4b, 5c.

### Exercice 7 : étude de cas final

Le groupe prépare l'ouverture de 100 magasins de proximité en 3 mois, avec une équipe réseau de 4 personnes. Proposez une démarche s'appuyant sur les 4 plans.

**Corrigé (pistes).**
- **Orchestration** : liste des équipements importée à l'avance, ZTP, Validators redondants.
- **Gestion** : un template unique "Proximité" avec variables importées par CSV ou API ; pilotage par vagues de 10 à 20 sites.
- **Contrôle** : topologie hub-and-spoke standardisée, politiques de segmentation communes.
- **Données** : deux transports (fibre et 5G), politiques de SLA pour le paiement, DIA pour les SaaS.
- **Exploitation** : tableaux de bord, alertes, procédure standard pour le support N1.

---

## 18. Bonnes pratiques et erreurs fréquentes

### Bonnes pratiques

- Concevoir **par plan** : chaque plan a ses exigences de disponibilité, de sécurité et de dimensionnement.
- Standardiser : un nombre **limité de profils** de site, des templates réutilisables.
- Gérer les **certificats** avec un calendrier de renouvellement et des alertes.
- Mettre en place des **tests automatisés** après chaque changement de politique.
- Documenter la **convention de nommage** et le plan d'adressage dès le début.
- Superviser les **liens opérateurs** de manière indépendante du SD-WAN pour contrôler les SLA contractuels.
- Prévoir des **fenêtres de changement** et un gel avant les périodes de pointe commerciale.
- Tester régulièrement la **perte des contrôleurs** et la **perte d'un transport** (exercices de panne).

### Erreurs fréquentes

- Croire que le SD-WAN compense un **lien Internet de mauvaise qualité**.
- Oublier la **segmentation** et laisser tous les flux dans un seul VPN.
- Mettre en production des **seuils de SLA** trop stricts qui provoquent des bascules incessantes (flapping).
- Sous-dimensionner les **hubs** ou les **edges** de gros sites (charge de chiffrement).
- Négliger la **MTU** et le **MSS**.
- Déployer un **seul Controller** ou un seul Validator.
- Laisser des administrateurs avec des **droits excessifs** dans le Manager.
- Ne pas planifier la **coexistence** avec l'ancien WAN pendant la migration.

---

## 19. Glossaire

| Terme | Définition |
|---|---|
| **SD-WAN** | Réseau étendu défini par logiciel, avec contrôle centralisé et overlay chiffré. |
| **Underlay** | Réseau de transport physique fourni par les opérateurs. |
| **Overlay** | Réseau logique construit au-dessus de l'underlay (tunnels). |
| **Edge** | Équipement SD-WAN installé sur un site. |
| **Validator (vBond)** | Composant d'orchestration : authentification et découverte. |
| **Manager (vManage)** | Composant de gestion : configuration, supervision, API. |
| **Controller (vSmart)** | Composant de contrôle : routes, clés, politiques. |
| **ZTP** | Zero-Touch Provisioning : intégration automatique d'un équipement. |
| **OMP** | Overlay Management Protocol, protocole de routage de l'overlay. |
| **TLOC** | Transport Locator : point de terminaison de transport (IP système, couleur, encapsulation). |
| **Couleur** | Étiquette qualifiant un type de transport (mpls, biz-internet, lte, etc.). |
| **VPN / VRF** | Segment logique isolé avec sa propre table de routage. |
| **BFD** | Bidirectional Forwarding Detection : détection de panne et mesure de qualité. |
| **SLA** | Seuils de qualité (latence, perte, gigue) à respecter pour une application. |
| **AAR** | Application-Aware Routing : choix du chemin selon l'application et la qualité mesurée. |
| **DIA** | Direct Internet Access : sortie directe vers Internet depuis le site. |
| **FEC** | Forward Error Correction : correction d'erreurs par redondance. |
| **QoS** | Qualité de service : priorisation du trafic. |
| **SASE** | Secure Access Service Edge : convergence réseau et sécurité fournies depuis le cloud. |
| **ZTNA** | Zero Trust Network Access : accès basé sur l'identité et le moindre privilège. |
| **Hub** | Site central qui agrège les connexions des spokes. |
| **Spoke** | Site périphérique relié à un hub. |
| **RBAC** | Role-Based Access Control : droits d'accès selon les rôles. |
| **PCI-DSS** | Norme de sécurité des données des cartes de paiement. |
| **NOC** | Network Operations Center : centre de supervision du réseau. |

---

## 20. Pour aller plus loin

- Documentation officielle de l'éditeur choisi (Cisco Catalyst SD-WAN, VMware VeloCloud, Fortinet Secure SD-WAN, Palo Alto Prisma SD-WAN, HPE Aruba EdgeConnect).
- Les RFC et standards utiles : RFC 7348 (VXLAN, pour la culture overlay), RFC 5880 (BFD), RFC 4301 (architecture IPsec), RFC 4271 (BGP).
- Certifications : Cisco CCNP Enterprise avec spécialisation SD-WAN (examen ENSDWI 300-415), Fortinet NSE 7 SD-WAN, VMware (Broadcom) VeloCloud, Palo Alto Prisma SD-WAN.
- Pratique : construire un laboratoire avec un simulateur (Cisco CML, EVE-NG, GNS3) ou un environnement de test fourni par l'éditeur, et rejouer les scénarios du chapitre 10.
- Sujets avancés : automatisation par API et Infrastructure as Code, intégration SASE, multi-cloud, observabilité applicative, SD-WAN et 5G privée.

---

## Fiche de révision en une page

| Plan | Mot clé | Composant | Question clé |
|---|---|---|---|
| 1. Orchestration | Accueil et confiance | Validator | Qui es-tu et où sont les autres ? |
| 2. Gestion | Pilotage | Manager | Comment configurer et superviser ? |
| 3. Contrôle | Intelligence | Controller | Où est quoi, et par où passer ? |
| 4. Données | Transport | Edge | Comment les paquets circulent et avec quelle qualité ? |

**Fil rouge à retenir.** Un magasin Carrefour s'ouvre en quelques jours grâce à l'**orchestration** et à la **gestion**, ses règles de sécurité et de chemin viennent du **contrôle**, et ses encaissements sont protégés et mesurés en permanence par le **plan de données**.
