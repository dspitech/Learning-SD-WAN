# Zero Trust, SASE, SSE : trois notions distinctes

Cours professionnel complet, rédigé pour être compris par un public non spécialiste tout en restant utile à un architecte ou à un responsable sécurité.

Avertissement : Carrefour est utilisé uniquement comme exemple pédagogique, parce que tout le monde connaît le fonctionnement d'un grand distributeur et de ses formats (hypermarchés, supermarchés, proximité, drive, entrepôts, siège). Je ne connais pas l'architecture réelle de cette enseigne. Tous les scénarios, noms de segments, numéros de magasins, applications, outils, incidents et choix techniques sont **inventés à but pédagogique**. Les noms de fonctions varient selon les constructeurs et les versions, il faut toujours se référer à la documentation officielle de la plateforme utilisée.

---

## Table des matières

1. La réponse en une page
2. D'où viennent ces notions : la fin du château fort
3. Zero Trust : un principe de sécurité
4. SSE : un ensemble de services de sécurité dans le cloud
5. SASE : réseau et sécurité réunis
6. Comparer les trois notions
7. Comment elles s'articulent
8. Le parcours d'une demande d'accès
9. Liens avec la PKI et le Secure Internet Breakout
10. Le cas d'étude : un grand distributeur fictif
11. Scénarios détaillés
12. Matrice de politiques d'accès
13. Feuille de route de mise en oeuvre
14. Gouvernance et responsabilités
15. Indicateurs de suivi
16. Critères de choix d'une solution
17. Risques et erreurs classiques
18. FAQ
19. Glossaire
20. Exercices et questions de révision
21. À retenir

---

## 1. La réponse en une page

Ces trois termes sont souvent confondus, y compris dans les documents commerciaux. Ils ne sont pas du même ordre.

| Notion | Nature | Question à laquelle elle répond | Exemple de phrase |
|---|---|---|---|
| **Zero Trust** | Principe, philosophie, modèle de sécurité | Comment décider de faire confiance ? | "Ne jamais faire confiance par défaut, toujours vérifier." |
| **SSE** (Security Service Edge) | Ensemble de services de sécurité fournis depuis le cloud | Quels outils protègent l'accès au web, au cloud et aux applications ? | "Notre SSE filtre le web, contrôle les applications cloud et donne l'accès aux applications privées." |
| **SASE** (Secure Access Service Edge) | Architecture qui réunit le réseau étendu (SD-WAN) et la sécurité cloud (SSE) | Comment connecter et protéger les utilisateurs et les sites, partout, de façon unifiée ? | "Notre SASE combine SD-WAN et SSE." |

Formule à retenir :

```
SASE = SD-WAN (le réseau) + SSE (la sécurité dans le cloud)
Zero Trust = le principe qui guide la façon de configurer l'ensemble
```

Image simple, celle d'un grand immeuble de bureaux :
- le **Zero Trust**, c'est la règle de l'immeuble : chacun montre son badge à chaque porte, même s'il est déjà entré, et l'accès dépend de qui il est, de ce qu'il porte et de ce qu'il va faire ;
- le **SSE**, c'est l'équipe de sécurité et ses équipements : gardiens, portiques, caméras, contrôle des colis ;
- le **SASE**, c'est l'immeuble complet avec ses couloirs et ses accès (le réseau) ainsi que son équipe de sécurité intégrée, accessible depuis n'importe quel site de l'entreprise.

---

## 2. D'où viennent ces notions : la fin du château fort

### 2.1 Le modèle historique du périmètre

Pendant des décennies, la sécurité reposait sur l'idée d'un **château fort** : un périmètre protégé par des pare-feux, avec "l'intérieur" considéré comme sûr et "l'extérieur" comme dangereux. Une fois à l'intérieur (réseau d'entreprise ou VPN), on avait accès à beaucoup de choses.

```
        Internet (hostile)
              |
        [ Pare-feu central ]
              |
   Réseau interne (considéré sûr)
   Postes, serveurs, applications
```

### 2.2 Pourquoi ce modèle ne tient plus

| Changement | Conséquence |
|---|---|
| Applications dans le cloud (SaaS, IaaS) | Les données ne sont plus "dans le château" |
| Télétravail et mobilité | Les utilisateurs sont partout, sur des réseaux non maîtrisés |
| Appareils personnels et objets connectés | Beaucoup d'équipements non gérés |
| Prestataires, partenaires, franchisés | Beaucoup d'acteurs externes ont besoin d'accéder à certaines ressources |
| Attaques par vol d'identifiants et rançongiciels | Une fois à l'intérieur, l'attaquant se déplace librement |
| Réseaux étendus avec accès Internet local | Le périmètre unique n'existe plus |

Constat central : **être "dans le réseau" ne prouve plus rien**. Un identifiant volé ou un poste infecté suffisent pour être "à l'intérieur".

### 2.3 Les trois réponses qui en découlent

1. Une **philosophie** : ne plus accorder de confiance implicite (Zero Trust).
2. Des **services de sécurité modernes** livrés depuis le cloud, au plus près des utilisateurs (SSE).
3. Une **architecture** qui réunit réseau et sécurité (SASE).

Repères historiques (à titre indicatif) : le terme Zero Trust a été popularisé au début des années 2010 et formalisé par des cadres de référence comme la publication SP 800-207 du NIST aux États-Unis. Les termes SASE et SSE ont été introduits par le cabinet d'analyse Gartner, respectivement en 2019 et en 2021.

---

## 3. Zero Trust : un principe de sécurité

### 3.1 Définition

Le Zero Trust (confiance zéro) est un **modèle de sécurité** selon lequel aucune personne, aucun appareil, aucun réseau ni aucune application n'est considéré comme fiable par défaut, quelle que soit sa position (à l'intérieur ou à l'extérieur du réseau). Chaque demande d'accès est **vérifiée explicitement**, avec le **minimum de droits**, en supposant que **la compromission est possible**.

Ce n'est **pas** un produit. On ne "achète pas du Zero Trust" : on le construit avec plusieurs technologies, des processus et une organisation.

### 3.2 Les trois idées fondatrices

| Principe | Signification | Exemple dans un magasin |
|---|---|---|
| Vérifier explicitement | Authentifier et autoriser avec toutes les informations disponibles : identité, appareil, lieu, risque, application demandée | Un responsable de rayon doit s'authentifier avec un second facteur, depuis un poste conforme, pour consulter les stocks |
| Moindre privilège | Donner uniquement l'accès nécessaire, pour le temps nécessaire | Un prestataire du froid accède à l'application du froid, pas au réseau du magasin, et seulement pendant son intervention |
| Supposer la compromission | Concevoir en pensant qu'un attaquant est peut-être déjà là, pour limiter les dégâts | Segmentation stricte : un poste infecté ne peut pas atteindre les caisses |

### 3.3 Les composants d'une architecture Zero Trust (modèle de référence)

Le cadre du NIST décrit des éléments logiques que l'on retrouve, sous des noms variables, dans toutes les solutions :

| Élément | Rôle | Analogie |
|---|---|---|
| Moteur de politique (Policy Engine) | Décide : autoriser ou refuser, à partir de la politique et des signaux | Le juge |
| Administrateur de politique (Policy Administrator) | Met en oeuvre la décision : ouvre ou ferme la session | Le greffier qui exécute |
| Point d'application (Policy Enforcement Point) | Se place sur le chemin de la connexion et applique la décision | Le portier |
| Sources de signaux | Fournissent le contexte : annuaire d'identités, état de l'appareil, renseignement sur les menaces, journaux, classification des données | Les informateurs |

Flux simplifié :

```
Utilisateur + Appareil
        |
  [ Point d'application ] <---- décision ---- [ Moteur de politique ]
        |                                             ^
        v                                             |
  Ressource (application, données)        Signaux : identité, appareil,
                                          localisation, risque, menaces
```

### 3.4 Les piliers d'un programme Zero Trust

Un modèle de maturité publié par l'agence américaine CISA organise le Zero Trust autour de cinq piliers, avec des fonctions transverses (visibilité, automatisation, gouvernance) :

| Pilier | Ce qu'il couvre | Exemples de mesures |
|---|---|---|
| Identité | Utilisateurs, comptes, services | Authentification multifacteur, fédération, gestion des accès privilégiés, désactivation automatique des comptes inutilisés |
| Appareils | Postes, mobiles, serveurs, objets | Inventaire, état de conformité, chiffrement, détection sur les terminaux |
| Réseaux | Segmentation, trafic | Microsegmentation, chiffrement des flux, contrôle des sorties |
| Applications et charges de travail | Applications, API, conteneurs | Accès par application, authentification forte, contrôle des API |
| Données | Classification et protection | Étiquetage, chiffrement, prévention des fuites |
| Transverse : visibilité et analytique, automatisation, gouvernance | Voir, décider vite, piloter | Journaux centralisés, réponse automatisée, revues d'accès |

### 3.5 Les niveaux de maturité (vue simplifiée)

| Niveau | Description | Exemple |
|---|---|---|
| Traditionnel | Confiance par le réseau, règles statiques | VPN donnant accès à tout le réseau |
| Initial | Premières vérifications explicites, automatisation partielle | MFA généralisé, début de segmentation |
| Avancé | Décisions contextuelles, intégration entre outils | Accès conditionnel selon l'état de l'appareil |
| Optimal | Décisions dynamiques, automatisées et continues | Accès adapté en temps réel au risque |

### 3.6 Ce que le Zero Trust n'est pas

| Idée reçue | Réalité |
|---|---|
| "C'est un produit" | C'est un modèle qui s'implémente avec plusieurs technologies et processus |
| "C'est ne faire confiance à personne dans l'entreprise" | C'est ne pas accorder de confiance implicite, tout en facilitant l'accès légitime |
| "C'est supprimer le pare-feu" | Le pare-feu reste utile, mais n'est plus l'unique rempart |
| "C'est seulement un VPN amélioré" | Le ZTNA remplace avantageusement le VPN pour l'accès aux applications, mais le Zero Trust est bien plus large |
| "C'est un projet de quelques mois" | C'est une démarche progressive qui dure plusieurs années |
| "C'est uniquement une affaire de réseau" | Identité, appareils, données et applications sont tout aussi centraux |

### 3.7 Les technologies qui servent le Zero Trust

| Technologie | Rôle |
|---|---|
| Gestion des identités et des accès (IAM) | Qui êtes-vous, que pouvez-vous faire |
| Authentification multifacteur (MFA) | Preuve renforcée de l'identité |
| Accès conditionnel | Autoriser selon le contexte (appareil, lieu, risque) |
| ZTNA (Zero Trust Network Access) | Accès aux applications privées, application par application |
| Détection et réponse sur les terminaux (EDR, XDR) | Voir et contenir les menaces sur les appareils |
| Microsegmentation | Limiter les déplacements entre systèmes |
| Gestion des appareils (MDM, gestion des postes) | Savoir et contrôler l'état des appareils |
| DLP | Prévenir les fuites de données |
| Journalisation, SIEM, SOAR | Voir, corréler, réagir |
| PKI | Identité forte des équipements et des services |

---

## 4. SSE : un ensemble de services de sécurité dans le cloud

### 4.1 Définition

Le SSE (Security Service Edge) est l'ensemble des **services de sécurité** livrés depuis le **cloud**, au plus près des utilisateurs, pour protéger l'accès au web, aux applications cloud et aux applications privées. Il correspond à la partie sécurité du SASE, sans la partie réseau étendu.

### 4.2 Les composants usuels

| Composant | Nom complet | Rôle | Exemple |
|---|---|---|---|
| SWG | Secure Web Gateway | Filtrer la navigation web, bloquer les sites dangereux, analyser les fichiers | Bloquer un site d'hameçonnage |
| CASB | Cloud Access Security Broker | Contrôler l'usage des applications cloud, détecter l'usage non autorisé, protéger les données | Interdire le dépôt de fichiers sensibles dans un service de partage personnel |
| ZTNA | Zero Trust Network Access | Donner accès à des applications privées sans exposer le réseau | Un prestataire accède à une seule application |
| FWaaS | Firewall as a Service | Pare-feu fourni en service cloud, pour les flux non web | Filtrer des protocoles variés sortant d'un site |
| DLP | Data Loss Prevention | Détecter et empêcher la sortie de données sensibles | Bloquer l'envoi d'un fichier contenant des numéros de cartes |
| Isolation de navigation (RBI) | Remote Browser Isolation | Exécuter la navigation dans un environnement isolé, l'utilisateur ne reçoit qu'un rendu | Ouvrir un lien suspect sans risque pour le poste |
| Sécurité DNS | | Bloquer les noms de domaine dangereux | Refuser la résolution d'un domaine malveillant |
| Surveillance de l'expérience numérique (DEM) | Digital Experience Monitoring | Mesurer la qualité perçue par l'utilisateur | Détecter une lenteur de la messagerie |

Selon les éditeurs, le périmètre exact du SSE varie : certains y incluent la DLP, l'isolation de navigation, la gestion de la posture de sécurité du cloud ou l'analyse du comportement des utilisateurs.

### 4.3 Comment le trafic arrive au SSE

| Mode | Description | Usage typique |
|---|---|---|
| Agent sur l'appareil | Un logiciel dirige le trafic vers le service | Postes nomades, télétravail |
| Tunnel depuis un site | Le routeur SD-WAN envoie le trafic par un tunnel IPsec ou GRE | Magasins, entrepôts |
| Configuration de proxy ou de navigateur | Le navigateur est dirigé vers le service | Cas simples |
| Connexion sans agent (navigateur) | Accès à une application via un portail | Prestataires, partenaires |
| Connecteur applicatif | Un connecteur sortant côté applications privées | ZTNA vers les applications internes |
| Intégration par API | Le service se branche aux applications cloud pour analyser les contenus | CASB hors ligne |

### 4.4 Où se situe le SSE par rapport à l'entreprise

```
Utilisateurs, sites, appareils
        |
        v
 [ Points de présence du service SSE, répartis dans le monde ]
   SWG | CASB | ZTNA | FWaaS | DLP
        |
        v
 Internet, applications cloud, applications privées
```

Avantages : politique unique, puissance élastique, mises à jour automatiques, protection partout. Limites : dépendance au fournisseur, coût par utilisateur ou par volume, besoin de superviser les tunnels et la disponibilité.

### 4.5 Zoom sur le ZTNA, le mot qui fait le lien

Le ZTNA est la technologie la plus emblématique du Zero Trust côté accès aux applications.

| | VPN traditionnel | ZTNA |
|---|---|---|
| Ce que l'utilisateur obtient | Un accès au réseau (souvent large) | Un accès à une application précise |
| Confiance | Accordée une fois à la connexion | Vérifiée en continu, selon le contexte |
| Visibilité des applications | Le réseau est visible de l'utilisateur | Les applications sont invisibles tant que l'accès n'est pas autorisé |
| Exposition | Passerelle VPN exposée sur Internet | Connexions sortantes uniquement, aucune porte ouverte |
| Mouvement latéral | Facile une fois connecté | Très limité |
| Expérience | Parfois lourde | Transparente pour l'utilisateur |

---

## 5. SASE : réseau et sécurité réunis

### 5.1 Définition

Le SASE (Secure Access Service Edge, prononcé "sassy") est une **architecture** qui réunit, dans un modèle livré largement depuis le cloud :
- les fonctions de **réseau étendu** (SD-WAN, optimisation, qualité de service),
- les fonctions de **sécurité** (SWG, CASB, ZTNA, FWaaS, DLP),
- avec une **politique unique** fondée sur l'identité, le contexte et l'application, appliquée partout.

### 5.2 Composition

| Couche | Fonctions | Éléments typiques |
|---|---|---|
| Réseau (WAN edge) | Connexion des sites, choix du meilleur chemin, qualité de service | Routeurs SD-WAN, overlay, liens fibre, 4G/5G |
| Sécurité (SSE) | Filtrage web, contrôle cloud, accès aux applications privées, prévention des fuites | SWG, CASB, ZTNA, FWaaS, DLP |
| Gestion et visibilité | Console, politiques, journaux, supervision de l'expérience | Gestionnaire central, tableaux de bord |
| Identité | Qui accède | Annuaire, MFA, accès conditionnel |

### 5.3 Modèles de mise en oeuvre

| Modèle | Description | Avantage | Inconvénient |
|---|---|---|---|
| SASE d'un seul fournisseur | Réseau et sécurité chez le même éditeur | Intégration, console unique, un seul contrat | Dépendance, forces inégales selon les domaines |
| SASE avec deux fournisseurs | SD-WAN d'un côté, SSE de l'autre, intégrés par tunnels et API | Meilleur de chaque domaine, flexibilité | Intégration à gérer, deux consoles |
| SASE géré | Un opérateur ou un intégrateur exploite la solution | Peu d'effort interne | Moins de maîtrise, dépendance au prestataire |
| Approche progressive | On commence par le SSE ou par le SD-WAN, on complète ensuite | Réduction du risque de projet | Période de transition hybride |

### 5.4 Rôle du SD-WAN dans le SASE

Le SD-WAN apporte :
- la connexion des sites avec plusieurs liens et la bascule automatique,
- l'identification des applications et le choix du chemin,
- la sortie Internet locale (breakout) et l'envoi du trafic vers le SSE,
- la qualité de service et la segmentation,
- l'authentification des équipements de site (voir le cours sur la PKI du SD-WAN).

Sans SD-WAN, le SSE protège bien les utilisateurs nomades, mais les sites (magasins, entrepôts) sont moins bien connectés. Sans SSE, le SD-WAN relie bien les sites mais la sécurité reste à assembler séparément.

---

## 6. Comparer les trois notions

### 6.1 Tableau de comparaison

| Critère | Zero Trust | SSE | SASE |
|---|---|---|---|
| Nature | Modèle, principe | Ensemble de services de sécurité cloud | Architecture réseau et sécurité |
| S'achète ? | Non | Oui (service) | Oui (service ou combinaison) |
| Couvre le réseau étendu ? | Pas en tant que tel | Non | Oui |
| Couvre la sécurité cloud ? | C'est son esprit | Oui, c'est son sujet | Oui, via le SSE |
| Dépend d'un éditeur ? | Non | Oui | Oui ou combinaison |
| Question posée | Comment décider d'accorder l'accès ? | Quels services protègent l'accès ? | Comment connecter et protéger de façon unifiée ? |
| Horizon | Démarche pluriannuelle | Projet technique | Programme de transformation |

### 6.2 Les confusions fréquentes

| Confusion | Précision |
|---|---|
| "SASE et Zero Trust, c'est la même chose" | Non : le SASE est une architecture, le Zero Trust est un principe que le SASE peut aider à appliquer |
| "SSE est une version allégée du SASE" | Presque : c'est la partie sécurité du SASE, sans le réseau |
| "Acheter du SASE, c'est être Zero Trust" | Non : il faut aussi les identités, les appareils, les données, les processus |
| "ZTNA = SASE" | Non : le ZTNA est un composant du SSE |
| "Zero Trust = ne plus avoir de pare-feu" | Non : on ajoute des contrôles et on réduit la confiance implicite |
| "Le SASE remplace le SD-WAN" | Non : il l'intègre ou s'appuie dessus |

### 6.3 Une métaphore commune : l'aéroport

- **Zero Trust** : la règle "chacun est contrôlé à chaque étape" (enregistrement, sûreté, embarquement), même un pilote ou un employé de l'aéroport.
- **SSE** : les équipements de contrôle (portiques, scanners, vérification des cartes d'embarquement, brigade cynophile).
- **SASE** : l'aéroport entier avec ses couloirs, ses portes et ses contrôles intégrés, utilisable depuis n'importe quel terminal.

---

## 7. Comment elles s'articulent

### 7.1 Vue d'ensemble

```
                        ZERO TRUST (principes et politique)
   +------------------------------------------------------------------+
   |                              SASE                                |
   |                                                                  |
   |   Réseau (SD-WAN)               +      Sécurité (SSE)            |
   |   - Sites, liens multiples            - SWG (web)               |
   |   - Choix du chemin                   - CASB (cloud)            |
   |   - Qualité de service                - ZTNA (applications)     |
   |   - Segmentation                      - FWaaS, DLP, RBI         |
   |                                                                  |
   +------------------------------------------------------------------+
        Identité, Appareil, Données, Applications, Visibilité
```

### 7.2 Ce que chacun apporte au Zero Trust

| Principe Zero Trust | Contribution du SD-WAN | Contribution du SSE |
|---|---|---|
| Vérifier explicitement | Authentification des équipements de site par certificat | Authentification de l'utilisateur et contrôle de la posture de l'appareil |
| Moindre privilège | Segmentation et politiques par flux | Accès par application (ZTNA), contrôle des applications cloud |
| Supposer la compromission | Confinement entre segments, isolement du wifi invité | Détection, isolation de navigation, prévention des fuites |
| Visibilité | Télémétrie des liens et des flux | Journaux de navigation, d'accès, de données |

### 7.3 Ce qui reste hors du SASE

Un programme Zero Trust complet comprend aussi : gestion des identités, gestion des appareils, détection sur les terminaux, sécurité des applications et du développement, sauvegardes, réponse aux incidents, formation des utilisateurs. Le SASE n'en est qu'une partie.

---

## 8. Le parcours d'une demande d'accès

Exemple : un responsable de rayon veut ouvrir une application interne de gestion des commandes depuis un poste du magasin.

| Étape | Action | Principe appliqué |
|---|---|---|
| 1 | L'utilisateur lance l'application | |
| 2 | Son identité est vérifiée (mot de passe plus second facteur) | Vérifier explicitement |
| 3 | L'état de l'appareil est contrôlé (système à jour, protection active, chiffrement) | Vérifier explicitement |
| 4 | Le contexte est évalué (heure, lieu, historique, niveau de risque) | Vérifier explicitement |
| 5 | Le moteur de politique décide : autoriser, restreindre, demander une preuve supplémentaire, refuser | Moindre privilège |
| 6 | Si autorisé, une connexion est établie vers cette seule application, via un connecteur sortant | Moindre privilège, aucune porte ouverte |
| 7 | La session est surveillée et enregistrée | Supposer la compromission |
| 8 | Si le risque change (appareil devenu non conforme, comportement anormal), la session est réévaluée ou coupée | Vérification continue |
| 9 | À la fin, la session est fermée, les journaux conservés | Traçabilité |

---

## 9. Liens avec la PKI et le Secure Internet Breakout

Cours précédents du même parcours :

| Cours | Lien avec Zero Trust, SSE, SASE |
|---|---|
| PKI du SD-WAN | Fournit l'**identité forte des équipements** de site (certificats, allowed list, révocation). C'est le pilier "appareils" et "réseaux" du Zero Trust côté site. |
| Secure Internet Breakout | Décrit comment un site sort localement sur Internet en restant protégé. Le **SSE** est l'un des moyens d'assurer cette protection, notamment par le tunnel du routeur vers le service cloud. |

Chaîne de confiance complète :

```
Équipement authentifié par certificat (PKI)
      -> reçoit la politique de sortie locale (breakout)
      -> envoie le trafic web vers le SSE par un tunnel
      -> l'utilisateur est authentifié, l'appareil évalué (Zero Trust)
      -> l'accès est accordé application par application (ZTNA)
```

---

## 10. Le cas d'étude : un grand distributeur fictif

### 10.1 Contexte

Un groupe de distribution inspiré d'une grande enseigne alimentaire comprend :
- un **siège** et des services centraux (finance, achats, marketing, informatique),
- des **hypermarchés, supermarchés et magasins de proximité**,
- des **drives** et des **entrepôts logistiques**,
- des **magasins franchisés** ou en location-gérance,
- des **prestataires** (maintenance du froid, informatique de caisse, nettoyage, sécurité),
- des **fournisseurs** et partenaires,
- des **clients** qui utilisent le wifi et l'application mobile.

### 10.2 Populations à protéger

| Population | Ordre de grandeur (fictif) | Besoins d'accès |
|---|---|---|
| Employés du siège | Plusieurs milliers | Messagerie, ERP, finance, outils collaboratifs, données de ventes |
| Personnel de magasin | Dizaines de milliers | Gestion des stocks, planning, messagerie, formation en ligne |
| Prestataires | Milliers de personnes | Accès ciblés à des systèmes précis |
| Franchisés | Centaines | Accès aux outils communs, séparation des données |
| Fournisseurs et partenaires | Milliers | Portails de commande, factures, catalogues |
| Développeurs et administrateurs | Centaines | Accès privilégiés aux environnements techniques |
| Clients | Millions | Wifi, application, site de commerce en ligne |

### 10.3 Applications et ressources

| Catégorie | Exemples fictifs |
|---|---|
| SaaS | Messagerie, collaboration, ressources humaines, formation |
| Applications internes | ERP, gestion des stocks, prix, planification, portails fournisseurs |
| Systèmes sensibles | Paiement, base clients, données de fidélité |
| Opérationnel magasin | Balances, étiquettes électroniques, froid, caméras |
| Environnements techniques | Serveurs, cloud public, outils de développement |

### 10.4 Appareils

| Type | Particularités |
|---|---|
| Postes du siège et portables | Gérés, chiffrés, agent de sécurité installé |
| Postes de magasin | Gérés, parfois partagés entre plusieurs personnes |
| Terminaux mobiles professionnels | Gérés par une plateforme de gestion |
| Appareils personnels | Non gérés, accès limité au web et à des applications ciblées |
| Caisses, terminaux de paiement | Environnement fermé, segmenté, sans agent |
| Objets connectés | Pas d'agent possible, contrôle par le réseau |

### 10.5 Cible d'architecture

| Brique | Rôle dans l'exemple |
|---|---|
| SD-WAN | Relier les sites, plusieurs liens, segmentation, sortie locale |
| PKI | Identité des Edges et des équipements |
| SSE | SWG pour le web, CASB pour les applications cloud, ZTNA pour les applications privées, DLP, RBI |
| Identité | Annuaire central, MFA, accès conditionnel |
| Gestion des appareils | Conformité des postes et mobiles |
| Détection sur les terminaux | Surveillance et réponse |
| Supervision | Journaux centralisés, corrélation, réponse |

---

## 11. Scénarios détaillés

Chaque scénario suit le plan : contexte, ce qui se passe, notions mobilisées (Zero Trust, SSE, SASE), enseignement.

### Scénario 1 : l'employé du siège qui télétravaille

**Contexte.** Une analyste des ventes travaille depuis son domicile, via un réseau personnel.

**Ce qui se passe.**
1. Elle ouvre son portable géré : l'agent du service de sécurité démarre automatiquement.
2. Elle s'authentifie avec un second facteur.
3. L'état du poste est contrôlé : chiffrement actif, système à jour, protection active.
4. Elle accède à la messagerie en ligne (SaaS) via le SWG et le CASB.
5. Elle ouvre l'application interne des ventes via le ZTNA : aucun accès au réseau, uniquement à cette application.
6. Une tentative d'ouvrir un outil de gestion de fichiers personnel est limitée en lecture seule par la politique CASB.

**Notions.** Zero Trust (vérification explicite, moindre privilège), SSE (SWG, CASB, ZTNA).

**Enseignement.** L'emplacement de l'utilisateur n'a plus d'importance : ce sont l'identité, l'appareil et le contexte qui décident.

### Scénario 2 : le responsable de rayon sur un poste partagé en magasin

**Contexte.** Magasin Super 2210, un poste est utilisé par plusieurs chefs de rayon dans la journée.

**Ce qui se passe.**
1. Chaque personne s'authentifie avec son propre compte et un second facteur adapté (badge ou application sur mobile).
2. Les droits sont ceux de son rôle : gestion des stocks de son rayon, pas des finances.
3. Les sessions se ferment automatiquement après inactivité.
4. Le trafic web passe par le tunnel du routeur vers le SSE, la messagerie en sortie locale optimisée.

**Notions.** Zero Trust (identité nominative, moindre privilège), SASE (SD-WAN local et SSE combinés).

**Enseignement.** Un poste partagé impose des identités individuelles fortes et des sessions courtes.

### Scénario 3 : le prestataire du froid

**Contexte.** Un technicien d'une société extérieure doit diagnostiquer un groupe froid à distance au Hyper 0142.

**Ce qui se passe.**
1. Le prestataire s'identifie avec son compte nominatif et un second facteur.
2. Une demande d'accès temporaire est validée par un responsable du magasin ou par un workflow.
3. Il ne voit qu'une seule application, la supervision du froid, via le ZTNA et un portail sans agent.
4. Sa session est enregistrée (actions, durée).
5. À la fin de la fenêtre, l'accès est coupé automatiquement.
6. Une revue mensuelle confirme les accès des prestataires.

**Notions.** Zero Trust (moindre privilège, accès limité dans le temps), SSE (ZTNA sans agent, enregistrement de session).

**Enseignement.** Ouvrir un VPN complet à un prestataire est le contraire du Zero Trust. On lui donne uniquement l'application, pour la durée nécessaire.

### Scénario 4 : un poste qui n'est plus conforme

**Contexte.** Un poste du siège perd sa protection (agent désactivé) à la suite d'une mise à jour défectueuse.

**Ce qui se passe.**
1. L'évaluation continue détecte que l'état du poste n'est plus conforme.
2. L'accès aux applications sensibles est suspendu, l'accès à une page d'aide est maintenu.
3. L'utilisateur reçoit un message expliquant la marche à suivre.
4. Après remédiation, l'accès est rétabli automatiquement.

**Notions.** Zero Trust (vérification continue), SSE (contrôle de posture).

**Enseignement.** La confiance n'est pas acquise une fois pour toutes : elle dépend de l'état à l'instant de la demande.

### Scénario 5 : compte compromis et connexion improbable

**Contexte.** À 9 h, un compte est utilisé depuis Paris. Vingt minutes plus tard, une connexion apparaît depuis un autre continent.

**Ce qui se passe.**
1. Le système détecte un déplacement impossible.
2. Le risque de la session monte : demande de preuve supplémentaire, puis blocage du compte.
3. L'équipe de sécurité reçoit l'alerte, les sessions actives sont révoquées.
4. Le mot de passe est réinitialisé, les journaux sont analysés pour mesurer l'impact.

**Notions.** Zero Trust (décision selon le contexte, supposer la compromission), SSE (analyse de comportement, journaux).

**Enseignement.** Un mot de passe volé ne suffit plus : le contexte et le comportement comptent.

### Scénario 6 : la fuite de données évitée

**Contexte.** Un acheteur tente d'envoyer, depuis sa messagerie personnelle, un fichier contenant la liste des prix négociés avec des fournisseurs.

**Ce qui se passe.**
1. Le CASB et la DLP détectent que le fichier est classé "confidentiel".
2. L'envoi vers un service de messagerie personnel est bloqué.
3. L'utilisateur est informé de la règle et de l'alternative autorisée (partage via l'outil de l'entreprise).
4. Un événement est journalisé pour revue.

**Notions.** Zero Trust (pilier données), SSE (CASB, DLP).

**Enseignement.** On protège la donnée elle-même, pas seulement le chemin qu'elle emprunte.

### Scénario 7 : l'application non autorisée découverte (shadow IT)

**Contexte.** Une équipe marketing utilise un outil de collaboration gratuit, inconnu du service informatique.

**Ce qui se passe.**
1. Le CASB détecte l'application dans les journaux de navigation et évalue son niveau de risque.
2. Le service sécurité contacte l'équipe pour comprendre le besoin.
3. Deux issues : proposer une application équivalente autorisée, ou autoriser celle-ci après évaluation, avec des règles (pas de données sensibles).

**Notions.** SSE (CASB, découverte des usages).

**Enseignement.** On ne peut pas protéger ce qu'on ne voit pas. La découverte précède la décision.

### Scénario 8 : le fournisseur qui accède à un portail

**Contexte.** Des milliers de fournisseurs consultent des commandes et déposent des factures.

**Ce qui se passe.**
1. Les fournisseurs accèdent au portail par un compte de partenaire avec authentification renforcée.
2. Chaque fournisseur ne voit que ses propres données (séparation par locataire).
3. Le portail est publié derrière le service de sécurité : pas d'exposition directe des serveurs.
4. Les dépôts de fichiers sont analysés (malware, contenu).

**Notions.** Zero Trust (moindre privilège), SSE (accès sans agent, analyse de fichiers).

**Enseignement.** Les partenaires ont un chemin dédié, jamais un accès au réseau interne.

### Scénario 9 : le magasin franchisé

**Contexte.** Le magasin City 8002 est exploité par un franchisé indépendant.

**Ce qui se passe.**
1. Les employés du franchisé disposent de comptes dans un annuaire séparé, fédéré avec celui du groupe.
2. Ils accèdent aux outils communs (catalogue, prix, commandes) via le ZTNA, application par application.
3. Les données du franchisé sont séparées de celles du groupe.
4. Le réseau du magasin applique la politique minimale commune, avec le wifi invité isolé.
5. Les journaux de sécurité sont partagés selon le contrat.

**Notions.** Zero Trust (séparation, moindre privilège), SASE (réseau et sécurité harmonisés).

**Enseignement.** La frontière juridique de l'entreprise ne coïncide pas avec la frontière technique : le Zero Trust gère ce type de cas naturellement.

### Scénario 10 : les caisses, un cas où l'agent n'est pas la solution

**Contexte.** Les caisses et terminaux de paiement sont des environnements fermés qui ne peuvent pas recevoir d'agent de sécurité classique.

**Ce qui se passe.**
1. Les caisses sont dans un segment dédié (PAIEMENT), sans sortie Internet.
2. Seules les destinations nécessaires sont autorisées, via l'overlay SD-WAN.
3. L'identité du routeur du site (certificat, allowed list) conditionne la réception de la politique.
4. Toute tentative de sortie vers Internet depuis ce segment déclenche une alerte de gravité élevée.

**Notions.** Zero Trust (microsegmentation, moindre privilège), SD-WAN et PKI. Le SSE n'est pas la protection principale ici.

**Enseignement.** Le Zero Trust ne se réduit pas à un agent ou à un service cloud : pour les environnements fermés, la segmentation et le contrôle strict des flux sont les outils adaptés.

### Scénario 11 : tentative de rançongiciel et déplacement latéral

**Contexte.** Un poste d'un magasin est infecté par un logiciel qui cherche à chiffrer des partages et à se propager.

**Ce qui se passe.**
1. L'agent de détection sur le poste repère un comportement de chiffrement massif et isole le poste du réseau.
2. Les règles entre segments empêchent l'accès au segment de paiement et aux autres magasins.
3. Le SWG bloque la communication vers le serveur de commande de l'attaquant.
4. Les sessions ZTNA de l'utilisateur sont révoquées, le compte est suspendu.
5. L'équipe de réponse enquête, restaure depuis les sauvegardes si nécessaire.

**Notions.** Zero Trust (supposer la compromission), SSE (SWG, révocation d'accès), SD-WAN (segmentation).

**Enseignement.** Les couches se complètent : chaque barrière réduit la zone d'impact.

### Scénario 12 : le fournisseur du service de sécurité cloud est en panne

**Contexte.** Un incident touche un point de présence du fournisseur du SSE.

**Ce qui se passe.**
1. Les routeurs détectent la perte du tunnel principal et basculent sur le tunnel secondaire vers un autre point de présence.
2. Les utilisateurs nomades se reconnectent automatiquement à un point voisin.
3. Si le service est entièrement indisponible, la politique par segment s'applique : fail-close pour la navigation web du personnel avec message clair, sortie locale conservée pour la messagerie critique si cela a été décidé, paiement non concerné.
4. Un plan de crise est activé, la communication aux magasins est assurée.

**Notions.** SASE (continuité), Zero Trust (décisions de repli explicites).

**Enseignement.** Un service cloud est une dépendance : elle se prépare par la redondance et des règles de repli définies à l'avance.

### Scénario 13 : le départ d'un collaborateur

**Contexte.** Un responsable achats quitte l'entreprise.

**Ce qui se passe.**
1. Le processus des ressources humaines déclenche la désactivation du compte à la date prévue.
2. Les sessions actives, les jetons et les accès aux applications sont révoqués.
3. L'appareil est récupéré, remis à zéro ou bloqué.
4. Les partages de fichiers sont transférés à son successeur.
5. Une revue vérifie qu'aucun accès résiduel ne subsiste, notamment dans les applications cloud.

**Notions.** Zero Trust (pilier identité, cycle de vie des accès), SSE (révocation, journaux).

**Enseignement.** Les accès orphelins sont une source fréquente de risque. La désactivation doit être rapide et automatique.

### Scénario 14 : l'usage d'outils d'IA générative

**Contexte.** Des collaborateurs utilisent des assistants d'IA en ligne pour rédiger des documents.

**Ce qui se passe.**
1. La politique définit les outils autorisés (version d'entreprise avec garanties de confidentialité) et les outils déconseillés ou bloqués.
2. Le CASB identifie les applications d'IA utilisées et mesure les volumes.
3. La DLP empêche d'y coller des données sensibles (données clients, contrats, informations de paiement).
4. Une formation explique les bons usages.

**Notions.** SSE (CASB, DLP), Zero Trust (pilier données).

**Enseignement.** Les nouveaux usages se gèrent par la même méthode : découvrir, évaluer, autoriser avec garde-fous, former.

### Scénario 15 : le développeur et l'environnement technique

**Contexte.** Un administrateur doit intervenir sur un serveur de production hébergé dans le cloud public.

**Ce qui se passe.**
1. Il demande un accès privilégié à une ressource précise, pour une durée définie, avec justification.
2. La demande est approuvée, une identité temporaire est créée.
3. L'accès passe par un bastion ou un ZTNA, avec enregistrement de la session.
4. À l'expiration, tous les droits disparaissent.

**Notions.** Zero Trust (accès juste à temps, moindre privilège), SSE (ZTNA, enregistrement).

**Enseignement.** Les comptes à privilèges permanents sont la cible préférée des attaquants. On privilégie les droits temporaires.

### Scénario 16 : la migration du VPN vers le ZTNA

**Contexte.** Le groupe utilise depuis longtemps un VPN qui donne accès à de larges plages réseau.

**Étapes.**
1. Inventorier les applications accessibles par VPN et les personnes qui les utilisent.
2. Classer les applications : web simples, applications métier, protocoles particuliers.
3. Publier d'abord les applications web via le ZTNA pour un groupe pilote.
4. Mesurer l'expérience utilisateur et corriger.
5. Étendre aux autres groupes et applications par vagues.
6. Réduire progressivement les plages accessibles par le VPN, puis le retirer.
7. Conserver un accès de secours documenté pendant la transition.

**Notions.** Zero Trust (moindre privilège), SSE (ZTNA).

**Enseignement.** Le VPN ne disparaît pas du jour au lendemain. La migration progressive limite le risque et l'impact utilisateur.

### Scénario 17 : ouverture d'un nouveau magasin dans l'architecture SASE

**Chronologie.**
1. J-30 : le routeur est commandé, son numéro de série est ajouté à l'allowed list.
2. J-15 : le modèle de configuration est associé (segments, sortie locale, tunnels vers le SSE).
3. J-10 : les comptes du personnel sont créés dans l'annuaire avec les rôles adaptés.
4. J-7 : le matériel est expédié, les terminaux mobiles sont préparés par la gestion des appareils.
5. Jour J : le routeur s'enrôle seul, établit les tunnels, reçoit la politique.
6. Jour J : tests guidés (paiement via l'overlay, messagerie en sortie locale, accès ZTNA aux applications, wifi invité isolé, bascule de lien).
7. J+7 : revue des journaux et ajustements.

**Notions.** SASE (réseau et sécurité déployés ensemble), PKI (identité du routeur), Zero Trust (identités et rôles).

**Enseignement.** L'ouverture d'un site doit être un processus reproductible, où la sécurité fait partie du déploiement dès le premier jour.

### Scénario 18 : l'intégration rapide d'une enseigne acquise

**Contexte.** Le groupe rachète une petite enseigne régionale de 80 magasins.

**Ce qui se passe.**
1. Au lieu de fusionner les réseaux, on connecte chaque magasin au SASE du groupe par un nouveau routeur ou un tunnel.
2. Les employés reçoivent des identités fédérées dans l'annuaire du groupe, avec des accès limités.
3. Les anciennes applications sont publiées par ZTNA, sans interconnexion réseau globale.
4. Les systèmes sont harmonisés progressivement, avec un calendrier de retrait des anciens accès.

**Notions.** Zero Trust (pas de confiance implicite entre réseaux), SASE (déploiement rapide).

**Enseignement.** Dans une acquisition, relier les réseaux entre eux est le contraire du Zero Trust. On connecte les utilisateurs aux applications, pas les réseaux entre eux.

### Scénario 19 : l'audit de conformité

**Contexte.** Un auditeur vérifie l'isolement du paiement et la maîtrise des accès.

**Ce qui est montré.**
1. Les politiques écrites, les schémas de segmentation, la gestion des identités et des accès.
2. Un test en direct : une caisse tente d'accéder à Internet, l'accès est bloqué et journalisé.
3. Un rapport des accès privilégiés et des revues périodiques.
4. La preuve d'une réponse à incident testée.
5. Les journaux centralisés avec durée de conservation conforme.

**Notions.** Zero Trust (gouvernance, visibilité).

**Enseignement.** Selon le périmètre, des cadres comme PCI DSS et le RGPD s'appliquent, et d'autres textes (par exemple de cybersécurité européenne) peuvent aussi être concernés, à vérifier avec les équipes conformité et juridique. Dans tous les cas, la sécurité doit être démontrable.

---

## 12. Matrice de politiques d'accès

Exemple fictif de politique, exprimée en langage simple.

| Population | Application | Condition | Décision |
|---|---|---|---|
| Employé du siège | Messagerie en ligne | Poste géré, MFA | Autoriser |
| Employé du siège | ERP finance | Poste géré, MFA, réseau de confiance ou ZTNA, rôle finance | Autoriser |
| Employé du siège | ERP finance | Poste non géré | Refuser, ou accès web en lecture seule via isolation |
| Chef de rayon | Gestion des stocks de son rayon | Poste de magasin géré, MFA | Autoriser |
| Chef de rayon | Données de paie | N'importe quelle condition | Refuser (hors rôle) |
| Prestataire froid | Supervision du froid | Compte nominatif, MFA, fenêtre d'intervention validée | Autoriser, session enregistrée |
| Prestataire froid | Autre application | N'importe quelle condition | Refuser |
| Fournisseur | Portail fournisseurs | Compte partenaire, MFA, ses données seulement | Autoriser |
| Franchisé | Catalogue et commandes | Compte fédéré, MFA | Autoriser |
| Administrateur | Serveur de production | Demande approuvée, accès temporaire, session enregistrée | Autoriser pour la durée |
| Appareil personnel | Messagerie | Navigateur uniquement, pas de téléchargement | Autoriser limité |
| Appareil personnel | Applications internes sensibles | N'importe quelle condition | Refuser |
| Caisse | Serveurs de paiement | Via overlay, destinations listées | Autoriser |
| Caisse | Internet | N'importe quelle condition | Refuser, alerte |
| Wifi invité | Réseau interne | N'importe quelle condition | Refuser |
| Wifi invité | Internet | Filtrage de contenu, débit plafonné | Autoriser |
| Compte avec risque élevé détecté | Toute application | Risque élevé | Preuve supplémentaire ou blocage |

Règle de lecture : **refuser par défaut**, n'autoriser que ce qui est écrit.

---

## 13. Feuille de route de mise en oeuvre

Programme progressif sur environ 24 mois (exemple), à adapter.

| Phase | Durée indicative | Objectifs | Livrables |
|---|---|---|---|
| 0. Cadrage | 1 à 2 mois | Définir la vision, les priorités, la gouvernance | Document de stratégie, sponsor exécutif, équipe projet |
| 1. Fondations identité et appareils | 3 à 6 mois | MFA généralisé, annuaire propre, inventaire des appareils, gestion des postes | Taux de MFA, inventaire à jour |
| 2. Visibilité | 2 à 4 mois (en parallèle) | Centraliser les journaux, cartographier les flux et applications | Cartographie, tableaux de bord |
| 3. SSE de base | 3 à 6 mois | Déployer SWG et sécurité DNS, CASB en découverte, tunnels depuis les sites | Politique web unifiée, découverte des usages cloud |
| 4. ZTNA par vagues | 6 à 12 mois | Remplacer progressivement le VPN, publier les applications privées | Applications migrées, VPN réduit |
| 5. Segmentation et microsegmentation | 6 à 12 mois (en parallèle) | Renforcer l'isolement des segments, des environnements et des charges de travail | Règles par segment, tests |
| 6. Protection des données | 4 à 8 mois | Classification, DLP, politiques CASB avancées | Règles de protection par type de donnée |
| 7. Automatisation et amélioration continue | Continu | Réponse automatisée, revues d'accès, tests d'intrusion | Playbooks, indicateurs de maturité |

Principes de conduite du projet :
- Commencer par un **pilote** représentatif avant de généraliser.
- Mesurer l'**expérience utilisateur** à chaque étape.
- Documenter un **plan de retour arrière** par vague.
- Associer les **métiers** autant que l'informatique.
- Communiquer et **former** les utilisateurs.

---

## 14. Gouvernance et responsabilités

| Rôle | Responsabilités principales |
|---|---|
| Sponsor exécutif | Soutien, arbitrages, budget |
| Responsable de la sécurité (RSSI) | Stratégie, politique, risques, conformité |
| Équipe réseau | SD-WAN, tunnels, liens, résilience |
| Équipe sécurité opérationnelle (SOC) | Surveillance, détection, réponse |
| Équipe identité et accès | Annuaire, MFA, cycle de vie des comptes |
| Équipe postes et mobilité | Gestion et conformité des appareils |
| Responsables d'applications | Propriétaires des applications et des droits |
| Métiers (magasins, logistique, siège) | Expression des besoins, validation des usages, relais auprès des utilisateurs |
| Conformité et juridique | Cadres réglementaires, protection des données, relations contractuelles |
| Achats et gestion des fournisseurs | Contrats, niveaux de service, clauses de sécurité |

Instances recommandées :
- comité de pilotage trimestriel,
- revue mensuelle des incidents et indicateurs,
- revue semestrielle des accès à privilèges et des accès de tiers,
- exercice annuel de gestion de crise.

---

## 15. Indicateurs de suivi

| Domaine | Indicateur | Cible indicative |
|---|---|---|
| Identité | Taux de comptes protégés par MFA | Proche de 100 % |
| Identité | Délai de désactivation d'un compte après départ | Moins de 24 heures, idéalement immédiat |
| Appareils | Taux d'appareils gérés et conformes | Plus de 95 % |
| Accès | Part d'applications privées accessibles via ZTNA | Croissance jusqu'à la quasi totalité |
| Accès | Nombre de comptes à privilèges permanents | Tendance à la baisse |
| Réseau | Part des flux soumis au filtrage de sécurité | Quasi 100 % pour le web |
| Réseau | Nombre de segments avec politique écrite | 100 % |
| Données | Part des données sensibles classifiées | Croissance continue |
| Détection | Délai moyen de détection d'un incident | Tendance à la baisse |
| Réponse | Délai moyen de confinement | Tendance à la baisse |
| Expérience | Temps de connexion, taux de plaintes, qualité des appels | Stables ou en amélioration |
| Disponibilité | Disponibilité mesurée des tunnels et du service cloud | Selon les engagements de niveau de service |
| Conformité | Écarts relevés en audit | Tendance à la baisse |

Les cibles sont indicatives, elles doivent être fixées selon le contexte de l'entreprise.

---

## 16. Critères de choix d'une solution

Questions à poser à tout fournisseur de SASE ou de SSE :

| Domaine | Questions |
|---|---|
| Couverture fonctionnelle | Quels composants sont natifs (SWG, CASB, ZTNA, FWaaS, DLP, RBI) et lesquels sont tiers ? |
| Présence mondiale | Quels points de présence, quelle latence mesurée depuis nos pays d'implantation ? |
| Disponibilité | Quels engagements de niveau de service, quel historique d'incidents publié ? |
| Intégration SD-WAN | Intégration native, tunnels automatisés, bascule entre points de présence ? |
| Identité | Compatibilité avec notre annuaire, MFA, accès conditionnel ? |
| Appareils | Intégration avec nos outils de gestion et de détection ? |
| Performance | Débits avec toutes les fonctions activées, inspection du trafic chiffré ? |
| Visibilité | Qualité des journaux, export vers notre outil de supervision, API ? |
| Conformité et données | Où sont stockés les journaux, localisation, certifications, conformité aux exigences locales ? |
| Souveraineté | Options de localisation des données et de contrôle, selon nos exigences ? |
| Exploitation | Console unique ? Automatisation par API ? Coût d'exploitation ? |
| Modèle de coûts | Par utilisateur, par volume, par site ? Évolution prévisible ? |
| Sortie du contrat | Réversibilité, export de la configuration, durée d'engagement ? |
| Feuille de route | Orientation produit, innovations, stabilité financière du fournisseur ? |

Méthode recommandée : établir une grille de critères pondérés, réaliser un **pilote comparatif** sur des cas réels (par exemple trois magasins et un groupe d'utilisateurs du siège), et mesurer performance, sécurité et expérience avant de décider.

---

## 17. Risques et erreurs classiques

| Erreur | Conséquence | Remède |
|---|---|---|
| Croire qu'acheter un produit suffit | Sécurité partielle, fausse assurance | Voir le Zero Trust comme un programme |
| Négliger l'identité | Le maillon le plus attaqué reste faible | Commencer par MFA, annuaire propre, cycle de vie des comptes |
| Tout déployer d'un coup | Rejet par les utilisateurs, incidents | Pilote puis vagues |
| Oublier les appareils non gérables (caisses, objets) | Trou dans la protection | Segmentation et contrôle réseau pour ces environnements |
| Politiques trop permissives "pour que ça marche" | Zero Trust de façade | Refuser par défaut, ajuster avec les métiers |
| Politiques trop strictes sans concertation | Contournements, shadow IT | Co-construire avec les métiers |
| Ne pas préparer la panne du service cloud | Arrêt d'activité | Redondance, repli par segment, exercices |
| Inspection du trafic chiffré sans cadrage | Problèmes de vie privée et de performance | Exclusions, information, dimensionnement |
| Ne pas supprimer le VPN | Le risque ancien subsiste | Plan de retrait daté |
| Ne pas mesurer l'expérience utilisateur | Perte de productivité, plaintes | Surveillance de l'expérience numérique |
| Pas de gouvernance | Dérive des règles | Comités, revues d'accès |
| Négliger la formation | Erreurs humaines | Sensibilisation continue |
| Journaux non exploités | Détection tardive | SIEM, cas d'usage, équipe dédiée |

---

## 18. FAQ

**Le Zero Trust veut-il dire que l'on se méfie des employés ?**
Non. Il s'agit de ne plus accorder de confiance par défaut selon la position dans le réseau, et de vérifier l'identité, l'appareil et le contexte. Pour l'utilisateur légitime, l'expérience peut même s'améliorer.

**Peut-on faire du Zero Trust sans SASE ?**
Oui. Le SASE est un moyen, pas une condition. On peut avancer avec l'identité, la gestion des appareils, la segmentation et le ZTNA seul.

**Peut-on faire du SASE sans Zero Trust ?**
Techniquement oui, mais on perd l'essentiel : sans politique fondée sur l'identité et le contexte, on déplace simplement l'ancien modèle dans le cloud.

**Quelle différence entre SSE et SASE ?**
Le SSE est la partie sécurité. Le SASE ajoute le réseau étendu (SD-WAN). Un SSE seul protège les utilisateurs et les applications, un SASE couvre aussi la connexion des sites.

**Le SASE remplace-t-il tout le matériel de sécurité ?**
Une grande part, mais pas toujours tout. Certains environnements (caisses, usines, sites très contraints) conservent une protection locale.

**Le ZTNA est-il un VPN ?**
Non, même si on l'utilise pour des cas d'usage proches. Le ZTNA donne accès à une application précise, sans exposer le réseau, avec vérification continue.

**Quel est le principal avantage du SASE pour un réseau de magasins ?**
Une politique unique appliquée partout, un déploiement rapide de nouveaux sites, de meilleures performances vers les applications cloud et une visibilité centralisée.

**Quel est son principal inconvénient ?**
La dépendance à un ou plusieurs fournisseurs et à la disponibilité d'un service cloud, qui se gère par la redondance et un plan de repli.

**Par où commencer ?**
Par l'identité (MFA, annuaire propre), l'inventaire des appareils et la visibilité sur les flux. Ce sont les fondations de toute la suite.

**Comment mesurer le succès ?**
Par la baisse des accès excessifs, du temps de détection et de confinement, la hausse de la couverture MFA et ZTNA, tout en préservant l'expérience utilisateur.

---

## 19. Glossaire

| Terme | Définition simple |
|---|---|
| Accès conditionnel | Règles d'accès qui dépendent du contexte (appareil, lieu, risque) |
| Allowed list | Liste des équipements autorisés |
| API | Interface permettant à des logiciels de communiquer |
| CASB | Contrôle de l'usage des applications cloud |
| DEM | Mesure de l'expérience numérique de l'utilisateur |
| DLP | Prévention de la perte de données |
| EDR / XDR | Détection et réponse sur les terminaux, ou sur plusieurs sources |
| FWaaS | Pare-feu fourni sous forme de service cloud |
| IAM | Gestion des identités et des accès |
| MFA | Authentification avec plusieurs preuves d'identité |
| Microsegmentation | Découpage fin du réseau en zones isolées |
| Moindre privilège | N'accorder que les droits strictement nécessaires |
| PKI | Infrastructure de gestion des certificats |
| Point de présence (PoP) | Site d'un fournisseur où se connectent les clients |
| RBI | Isolation de navigation |
| SASE | Architecture réunissant réseau étendu et sécurité cloud |
| SD-WAN | Réseau étendu piloté par logiciel |
| SIEM | Plateforme de collecte et de corrélation des journaux de sécurité |
| SOAR | Automatisation de la réponse aux incidents |
| SSE | Ensemble de services de sécurité fournis depuis le cloud |
| SWG | Passerelle de sécurité web |
| Shadow IT | Usage d'outils non approuvés par le service informatique |
| Zero Trust | Modèle de sécurité sans confiance implicite |
| ZTNA | Accès aux applications privées fondé sur le Zero Trust |

---

## 20. Exercices et questions de révision

### 20.1 Questions

1. En une phrase, quelle est la nature du Zero Trust ?
2. Que signifie "SASE = SD-WAN + SSE" ?
3. Citez quatre composants usuels d'un SSE.
4. Quelle différence entre un VPN et un ZTNA ?
5. Pourquoi les caisses ne reçoivent-elles pas un agent de sécurité, et comment les protège-t-on ?
6. Quel est le rôle de la PKI dans une architecture SASE ?
7. Pourquoi faut-il définir à l'avance le comportement en cas de panne du service cloud ?
8. Que signifie "supposer la compromission" ?
9. Pourquoi un prestataire ne doit-il pas disposer d'un accès réseau complet ?
10. Quelles sont les premières actions d'un programme Zero Trust ?

**Réponses courtes.**
1. Un modèle de sécurité qui n'accorde aucune confiance par défaut et vérifie chaque demande.
2. Le SASE réunit le réseau étendu et la sécurité fournie dans le cloud.
3. SWG, CASB, ZTNA, FWaaS (ou DLP, RBI).
4. Le VPN donne un accès au réseau, le ZTNA un accès à une application précise avec vérification continue.
5. Ce sont des environnements fermés. On les protège par segmentation, sortie Internet interdite et destinations listées.
6. Elle fournit l'identité forte des équipements de site, condition pour qu'ils reçoivent la politique et établissent les tunnels.
7. Pour éviter l'improvisation et protéger les segments sensibles pendant l'incident.
8. Concevoir la sécurité en admettant qu'un attaquant est peut-être déjà présent, pour limiter l'impact.
9. Pour limiter les dégâts en cas de compromission et respecter le moindre privilège.
10. Identité (MFA, annuaire), inventaire des appareils, visibilité sur les flux.

### 20.2 Exercice de classement

Classez chacune de ces phrases en Zero Trust, SSE ou SASE :
1. "Nous filtrons le web et contrôlons les applications cloud depuis le cloud."
2. "Nous ne faisons plus confiance à un poste parce qu'il est branché dans un magasin."
3. "Nous avons un routeur de site qui envoie son trafic vers un service de sécurité cloud avec une politique commune."
4. "Chaque accès est vérifié selon l'identité, l'appareil et le contexte, avec les droits minimums."

**Réponses.** 1) SSE. 2) Zero Trust. 3) SASE. 4) Zero Trust.

### 20.3 Exercice d'analyse de cas

Un hypermarché doit permettre à un prestataire informatique d'intervenir sur le serveur de gestion locale, un samedi, pendant deux heures.

Questions :
1. Quelle approche d'accès proposez-vous ?
2. Quelles preuves exigez-vous ?
3. Que se passe-t-il à la fin des deux heures ?
4. Quel contrôle a posteriori ?

**Éléments de réponse.**
1. Un accès ZTNA limité à l'application ou au serveur concerné, sans accès réseau global.
2. Compte nominatif, second facteur, validation de la demande par un responsable, éventuellement état de l'appareil du prestataire.
3. L'accès est coupé automatiquement et les sessions sont révoquées.
4. Revue de l'enregistrement de session et des journaux, confirmation qu'aucun accès résiduel ne subsiste.

### 20.4 Exercice de conception

Concevez l'accès pour un nouveau groupe d'utilisateurs : 300 conseillers du service client d'un site externalisé, travaillant à distance sur des ordinateurs fournis par le sous-traitant, avec accès à l'application de gestion des commandes et à la messagerie.

**Proposition.**
- Identités dans un annuaire dédié fédéré, MFA obligatoire.
- Appareils : conformité exigée (chiffrement, protection active), à défaut accès via un poste virtuel ou un navigateur isolé.
- Accès à l'application via ZTNA, sans accès réseau.
- Messagerie via SWG et CASB, avec DLP pour empêcher l'export massif de données clients.
- Pas de téléchargement local de données sensibles.
- Journalisation détaillée et revue mensuelle.
- Clause contractuelle de sécurité et audit annuel du sous-traitant.

### 20.5 Questions de réflexion

1. Quels sont les risques d'un projet SASE mené uniquement par l'équipe réseau ?
2. Comment mesurer qu'un programme Zero Trust avance vraiment ?
3. Dans quels cas un SASE d'un seul fournisseur est-il préférable à une solution à deux fournisseurs ?

---

## 21. À retenir

- **Zero Trust** est un principe : ne jamais faire confiance par défaut, vérifier explicitement, accorder le moindre privilège, supposer la compromission. Ce n'est pas un produit.
- **SSE** est la partie sécurité livrée depuis le cloud : SWG, CASB, ZTNA, FWaaS, souvent DLP et isolation de navigation.
- **SASE** réunit le réseau étendu (SD-WAN) et le SSE dans une architecture unifiée, avec une politique unique.
- Formule : SASE = SD-WAN + SSE, et le Zero Trust guide la façon de configurer l'ensemble.
- Le ZTNA est la technologie emblématique du Zero Trust pour l'accès aux applications privées : on donne accès à une application, pas au réseau.
- Les environnements fermés (caisses, objets connectés) se protègent par segmentation et contrôle strict des flux, pas par un agent.
- La PKI du SD-WAN fournit l'identité des équipements de site, le Secure Internet Breakout organise la sortie locale, le SSE protège l'accès : ces trois cours forment une chaîne cohérente.
- Un programme Zero Trust commence par l'identité, les appareils et la visibilité, et s'étend par vagues sur plusieurs années.
- Les décisions de repli en cas de panne du service cloud se prennent à l'avance, par segment.
- La sécurité doit être démontrable : politiques écrites, tests, journaux, revues.
- Tous les scénarios de ce document sont fictifs et pédagogiques, ils ne décrivent pas l'architecture réelle d'une enseigne.
