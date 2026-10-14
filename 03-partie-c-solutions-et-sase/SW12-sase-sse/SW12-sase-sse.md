# SW12. SASE et SSE

| | |
|---|---|
| **Partie** | C. Autres solutions, SASE et SSE |
| **Durée indicative** | 1,5 semaine (12 h) |
| **Prérequis** | SW01 à SW08 ; notions de proxy web, DNS, identité (SSO, SAML), Zero Trust |
| **Niveau** | Avancé |
| **Lien examen** | NIST SP 800-207 ; certifications SASE/SSE des éditeurs |
| **Lab associé** | T10 (architecture SASE d'une entreprise multi-sites et télétravailleurs) |

> Module précédent : [SW11](../SW11-panorama-solutions/SW11-panorama-solutions.md) | Module suivant : [SW13](../../04-partie-d-conception-migration-exploitation/SW13-conception-architecture/SW13-conception-architecture.md)

> **Avertissement.** Les offres SASE/SSE se regroupent et se renomment fréquemment (consolidation du marché). Les noms de produits cités sont des **exemples** et doivent être revérifiés. Concentrez-vous sur les **concepts** et les **fonctions**, plus stables que les noms commerciaux.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Pourquoi le SASE](#2-pourquoi-le-sase)
3. [Définitions : SASE, SSE et leurs composants](#3-définitions--sase-sse-et-leurs-composants)
4. [Les fonctions de sécurité du SSE en détail](#4-les-fonctions-de-sécurité-du-sse-en-détail)
5. [ZTNA contre VPN](#5-ztna-contre-vpn)
6. [Architecture Zero Trust (NIST SP 800-207)](#6-architecture-zero-trust-nist-sp-800-207)
7. [Modèles d'intégration SD-WAN et SSE](#7-modèles-dintégration-sd-wan-et-sse)
8. [Orientation du trafic vers le service](#8-orientation-du-trafic-vers-le-service)
9. [Identités, postures et politiques unifiées](#9-identités-postures-et-politiques-unifiées)
10. [Considérations de conception](#10-considérations-de-conception)
11. [Feuille de route de migration vers le SASE](#11-feuille-de-route-de-migration-vers-le-sase)
12. [Lab guidé T10](#12-lab-guidé-t10)
13. [Approfondissements](#13-approfondissements)
14. [Quiz](#14-quiz)
15. [Aide-mémoire](#15-aide-mémoire)
16. [Pour aller plus loin](#16-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Définir SASE et SSE, et expliquer pourquoi ces concepts sont apparus.
- Décrire chaque fonction du SSE : SWG, CASB, ZTNA, FWaaS, DLP.
- Comparer ZTNA et VPN d'accès à distance.
- Appliquer les principes de NIST SP 800-207 à une architecture réelle.
- Concevoir l'intégration d'un SD-WAN avec un service de sécurité cloud.
- Planifier une migration progressive, en maîtrisant risques et coûts.

---

## 2. Pourquoi le SASE

### 2.1 Le périmètre a disparu

```text
Avant : utilisateurs, applications et données DANS le périmètre -> pare-feu au siège = protection suffisante
Après : utilisateurs partout (domicile, agences, déplacement), applications dans le SaaS et l'IaaS, données partout
        -> le "périmètre" n'est plus un lieu, c'est l'identité, l'appareil et le contexte
```

### 2.2 Les problèmes de l'ancien modèle

| Problème | Conséquence |
|---|---|
| **Backhauling** : tout le trafic ramené au siège pour être filtré | Latence, goulot, coûts |
| **VPN d'accès à distance** en accès réseau large | Un poste compromis voit tout le réseau |
| **Empilement d'appliances** (proxy, pare-feu, IPS, DLP) par site | Coût, complexité, politiques incohérentes |
| **Angle mort du SaaS** | Données sensibles hors de la visibilité de l'entreprise |
| **Politiques multiples** | Une règle à écrire à cinq endroits |

### 2.3 Idée directrice

> Déplacer les services de sécurité **près des utilisateurs et des applications**, sous forme de **services cloud mutualisés**, avec **une politique unique** fondée sur l'identité et le contexte.

---

## 3. Définitions : SASE, SSE et leurs composants

| Terme | Définition |
|---|---|
| **SASE** (Secure Access Service Edge) | Convergence des **fonctions réseau** (SD-WAN) et des **fonctions de sécurité** (SSE) fournies comme un **service cloud** (terme proposé par Gartner en 2019) |
| **SSE** (Security Service Edge) | La **partie sécurité** du SASE : SWG, CASB, ZTNA, FWaaS, DLP… |
| **SD-WAN** | La **partie réseau** : connexion des sites, choix de chemin, QoS |
| **SWG** (Secure Web Gateway) | Proxy web sécurisé : filtrage d'URL, inspection, antimalware |
| **CASB** (Cloud Access Security Broker) | Visibilité et contrôle des **applications SaaS** (usage, données, conformité) |
| **ZTNA** (Zero Trust Network Access) | Accès **par application**, après vérification de l'identité et de la posture |
| **FWaaS** (Firewall as a Service) | Pare-feu dans le cloud (L3 à L7) pour tout le trafic |
| **DLP** (Data Loss Prevention) | Détection et blocage des fuites de données |
| **RBI** (Remote Browser Isolation) | Navigation isolée dans un conteneur distant |
| **DEM** (Digital Experience Monitoring) | Mesure de l'expérience de bout en bout |

Équation simplifiée :

```text
SASE  =  SD-WAN  +  SSE
SSE   =  SWG + CASB + ZTNA (+ FWaaS + DLP + RBI + DEM)
```

> **Précision utile** : « SASE » est souvent employé comme **terme marketing**. Examinez précisément **quelles fonctions** sont fournies, **sur quelle plate-forme**, avec **quelle politique unifiée** et **quelle couverture géographique**.

---

## 4. Les fonctions de sécurité du SSE en détail

### 4.1 SWG : passerelle web sécurisée

| Fonction | Détail |
|---|---|
| Filtrage d'URL et de catégories | Autorisation/blocage |
| Antimalware | Analyse des téléchargements |
| Inspection TLS | Déchiffrement sélectif |
| Contrôle des applications web | Autoriser la consultation, interdire le téléversement |
| Isolation de navigation (RBI) | Pour les sites à risque |
| Protection DNS | Blocage de domaines malveillants |

### 4.2 CASB

| Mode | Principe | Exemple |
|---|---|---|
| **Inline** (dans le flux) | Contrôle en temps réel du trafic vers le SaaS | Interdire le téléversement de fichiers vers un stockage personnel |
| **API** (hors flux) | Analyse des données déjà stockées dans le SaaS | Découvrir les fichiers partagés publiquement |
| **Shadow IT** | Découverte des applications utilisées | Rapport des 300 applications utilisées, dont 40 à risque |

### 4.3 ZTNA

Accès à une **application précise** (et non au réseau), après vérification de l'**identité**, de l'**état du terminal** et du **contexte**. Voir la section 5.

### 4.4 FWaaS

Pare-feu dans le cloud : filtrage sur tous les ports et protocoles (pas seulement le web), utile pour les sites sans pare-feu local et pour les flux non web.

### 4.5 DLP

| Élément | Exemple |
|---|---|
| Détecteurs | Numéros de carte bancaire, numéros de sécurité sociale, mots-clés |
| Actions | Bloquer, alerter, chiffrer, mettre en quarantaine |
| Lieux | Web, e-mail, SaaS, terminaux |
| Précaution | Très sensible aux **faux positifs** : démarrer en mode observation |

---

## 5. ZTNA contre VPN

### 5.1 Le VPN d'accès à distance classique

L'utilisateur se connecte au réseau de l'entreprise comme s'il était dans les locaux : **accès réseau large**. Un poste compromis peut scanner et attaquer tout ce qui est joignable.

### 5.2 ZTNA

```text
Utilisateur + terminal vérifiés --> Courtier ZTNA (cloud) --> connecteur dans le réseau --> application X UNIQUEMENT
                                    (politique : qui, quoi, quelle posture)
```

| Critère | VPN d'accès distant | ZTNA |
|---|---|---|
| **Accès accordé** | Au **réseau** | À **l'application** |
| **Mouvement latéral** | Facile | Très limité |
| **Visibilité des applications** | Faible | Précise (journaux par application) |
| **Vérification de la posture** | Optionnelle | Native (chiffrement disque, antivirus, correctifs) |
| **Exposition** | Concentrateur VPN exposé sur Internet | Connecteurs en **sortie uniquement** (pas de port entrant) |
| **Expérience** | Tunnel complet, parfois lent | Accès direct à l'application |
| **Applications non web** | Oui | Selon l'offre (souvent oui, avec agent) |
| **Migration** | — | Progressive, application par application |

### 5.3 Démarche de migration VPN → ZTNA

1. Inventorier les applications accessibles par VPN et leurs utilisateurs.
2. Commencer par les applications **web internes** (les plus simples).
3. Faire coexister VPN et ZTNA ; basculer par groupes d'utilisateurs.
4. Mesurer, ajuster, puis retirer l'accès VPN large.

---

## 6. Architecture Zero Trust (NIST SP 800-207)

### 6.1 Les sept principes (tenets)

| # | Principe (reformulé) |
|---|---|
| 1 | Toutes les sources de données et services de calcul sont des **ressources** |
| 2 | Toutes les communications sont **sécurisées** quelle que soit la localisation réseau |
| 3 | L'accès à une ressource est accordé **par session** |
| 4 | L'accès est déterminé par une **politique dynamique** (identité, état du terminal, comportement) |
| 5 | L'entreprise surveille et mesure l'**intégrité** et la **posture** de tous ses actifs |
| 6 | L'authentification et l'autorisation sont **dynamiques et strictement appliquées** avant l'accès |
| 7 | L'entreprise collecte un maximum d'informations pour **améliorer** sa posture |

### 6.2 Composants logiques

| Composant | Rôle | Correspondance SASE |
|---|---|---|
| **Policy Engine (PE)** | Décide d'accorder ou non l'accès | Moteur de politique du fournisseur SSE |
| **Policy Administrator (PA)** | Établit ou coupe la session | Courtier ZTNA, orchestrateur |
| **Policy Enforcement Point (PEP)** | Applique la décision | Passerelle SSE, connecteur ZTNA, WAN Edge |
| **Sources de données** | Identités, conformité des terminaux, menaces, journaux | IdP, MDM/EDR, SIEM |

### 6.3 Ce que le SD-WAN apporte au Zero Trust

Canaux chiffrés, segmentation, politique centralisée, journalisation et visibilité des flux. Il **ne remplace pas** l'identité forte et la vérification des terminaux.

---

## 7. Modèles d'intégration SD-WAN et SSE

| Modèle | Description | Avantages | Inconvénients |
|---|---|---|---|
| **Même éditeur (intégré)** | SD-WAN et SSE du même fournisseur, politiques et console communes | Simplicité, tunnels automatisés, support unique | Dépendance, parfois moins bon dans une des fonctions |
| **Éditeurs différents (« best of breed »)** | SD-WAN d'un éditeur, SSE d'un autre, connectés par tunnels IPsec/GRE | Meilleure fonction dans chaque domaine, négociation | Deux consoles, intégration à maintenir |
| **SASE nativement unifié** | Un seul service pour réseau et sécurité (équipement léger sur site) | Simplicité extrême | Dépendance forte, moins de flexibilité |
| **Hybride** | Pare-feu local pour certains flux (OT, inter-segments) et SSE pour le reste | Adapté aux contraintes réelles | Politique à synchroniser |

```text
                    +---------------------+
Utilisateurs        |  Service SSE (cloud)|        Internet / SaaS / IaaS
mobiles (agent) ----|  SWG CASB ZTNA FW   |------> applications
                    +----------+----------+
                               |  tunnels IPsec/GRE doubles
Sites (WAN Edge) --------------+
        |
        +--- overlay SD-WAN ---> Centres de données et applications privées (connecteurs ZTNA)
```

---

## 8. Orientation du trafic vers le service

| Méthode | Principe | Usage |
|---|---|---|
| **Tunnels IPsec ou GRE** depuis le WAN Edge | Tout le trafic Internet d'un VPN part vers le service | Sites |
| **Proxy explicite (PAC/WPAD)** | Le navigateur est configuré pour utiliser un proxy | Postes, sans changement réseau |
| **Agent sur le terminal** | Redirige le trafic (web, ZTNA) | Télétravailleurs, mobiles |
| **Redirection DNS** | Résolveur de sécurité dans le cloud | Complément de protection |
| **Connecteur ZTNA** | Tunnel sortant depuis le réseau privé vers le courtier | Applications privées |

### 8.1 Exemple de fichier PAC (proxy auto-config)

```javascript
// proxy.pac : web vers le SWG cloud, exceptions pour l'interne
function FindProxyForURL(url, host) {
  // Réseau interne et domaines d'entreprise : accès direct
  if (isPlainHostName(host) ||
      dnsDomainIs(host, ".entreprise.local") ||
      shExpMatch(host, "*.intranet.entreprise.fr") ||
      isInNet(dnsResolve(host), "10.0.0.0", "255.0.0.0") ||
      isInNet(dnsResolve(host), "192.168.0.0", "255.255.0.0")) {
    return "DIRECT";
  }
  // Applications incompatibles avec l'inspection : accès direct
  if (shExpMatch(host, "*.bancaire-exemple.fr")) {
    return "DIRECT";
  }
  // Tout le reste : proxy principal puis secondaire, sinon bloquer (pas de repli en direct)
  return "PROXY swg1.exemple-sse.net:80; PROXY swg2.exemple-sse.net:80";
}
```

> Avec ce PAC, l'absence de repli « DIRECT » à la fin évite le contournement du filtrage en cas de panne du proxy ; à l'inverse, elle bloque la navigation si le service est inaccessible : à choisir selon la tolérance de l'entreprise.

### 8.2 Principe d'une politique de redirection d'un VPN SD-WAN

```text
data-policy DP-SIG
 vpn-list CORP
  sequence 10
   match
    destination-data-prefix-list INTERNES     <- adresses d'entreprise et applications privées
   action accept                              <- trafic interne : overlay normal
  sequence 20
   match
    source-ip 0.0.0.0/0
   action accept
    sig                                       <- trafic Internet : vers le service SSE par tunnel
    sig-fallback                              <- (selon version) repli sur la sortie locale si les tunnels sont down
```

> La syntaxe exacte (mots-clés `sig`, repli) dépend de la version ; dans le Manager, on configure cela par **Cloud OnRamp for SSE** ou par un **modèle SIG** et une politique de redirection.

---

## 9. Identités, postures et politiques unifiées

### 9.1 Identité

| Élément | Exemple |
|---|---|
| **Fournisseur d'identité (IdP)** | Entra ID (Azure AD), Okta, Ping, Keycloak |
| **Protocoles** | SAML 2.0, OIDC pour le SSO ; SCIM pour le provisionnement |
| **MFA** | Obligatoire pour l'accès aux applications sensibles |
| **Groupes** | Les politiques utilisent des groupes de l'IdP |

### 9.2 Posture du terminal

Conditions vérifiées avant l'accès : disque chiffré, système à jour, antivirus/EDR actif, certificat d'appareil, absence de jailbreak. Remontées par un agent, un MDM ou un EDR.

### 9.3 Politique unifiée : exemple de règles

| # | Qui | Quoi | Condition | Action |
|---|---|---|---|---|
| 1 | Groupe `Finance` | Application ERP (privée) | Terminal conforme + MFA | Autoriser (ZTNA) |
| 2 | Tout utilisateur | Web catégorie « malveillant » | — | Bloquer |
| 3 | Tout utilisateur | SaaS non approuvé | — | Autoriser en lecture seule |
| 4 | Groupe `RH` | Stockage cloud personnel | — | Bloquer le téléversement |
| 5 | Prestataires | Application de support | Terminal non géré | Autoriser via isolation de navigateur |
| 6 | Tout | Transfert de numéros de carte vers l'extérieur | — | Bloquer + alerte (DLP) |

---

## 10. Considérations de conception

| Sujet | Questions et recommandations |
|---|---|
| **Latence** | Distance entre l'utilisateur et le point de présence (PoP) du service ; chiffrer + inspecter ajoute quelques ms ; viser un PoP à moins de 20 ms |
| **Couverture régionale** | Le service a-t-il des PoP dans **tous vos pays** (certaines régions sont mal couvertes) ? |
| **Résilience** | Tunnels vers **deux PoP** ; repli défini (bloquer ou sortie locale) |
| **Intégration des identités** | IdP unique, groupes cohérents, cycle de vie (arrivées, départs) |
| **Applications sensibles à l'inspection** | Exceptions documentées (visioconférence, mise à jour d'OS, applications bancaires) |
| **Journalisation et SIEM** | Export des journaux du service vers le SIEM |
| **Conformité et souveraineté** | Localisation du traitement et du stockage, certifications |
| **Coûts** | Par utilisateur et/ou par volume ; coût des tunnels et des licences SD-WAN |
| **Gestion unifiée** | Une console, une politique : évaluer la réalité derrière l'argument commercial |
| **Sortie de contrat** | Exports de politique, possibilité de migrer |

### 10.1 Calcul de latence ajoutée (exemple)

| Segment | Sans SSE | Avec SSE |
|---|---|---|
| Poste → site (LAN) | 1 ms | 1 ms |
| Site → Internet (accès) | 8 ms | 8 ms |
| Détour par le PoP SSE (aller-retour) | — | + 6 ms |
| Inspection (traitement) | — | + 4 ms |
| **Total supplémentaire** | | **≈ 10 ms** |

Comparé à un backhauling vers un siège distant de 40 ms, le SSE **réduit** la latence ; comparé à une sortie locale directe sans inspection, il l'**augmente de 10 ms**.

---

## 11. Feuille de route de migration vers le SASE

| Phase | Contenu | Durée indicative |
|---|---|---|
| **0. Cadrage** | Inventaire des flux, des applications, des utilisateurs ; objectifs ; choix de modèle | 4 à 6 semaines |
| **1. Fondations** | IdP unique, MFA, inventaire des terminaux, journalisation | 6 à 8 semaines |
| **2. SWG et DNS** | Redirection du trafic web (agents et tunnels) ; filtrage ; observation | 8 semaines |
| **3. ZTNA** | Applications web privées d'abord, puis applications non web | 12 semaines |
| **4. CASB et DLP** | Découverte SaaS, politiques, DLP en mode observation puis blocage | 8 à 12 semaines |
| **5. Consolidation** | Retrait des appliances redondantes, politique unifiée | 12 semaines |
| **6. Amélioration continue** | Revue trimestrielle, indicateurs | continu |

Indicateurs de succès : réduction du trafic de backhauling, nombre d'applications derrière ZTNA, taux de terminaux conformes, incidents évités, coût par utilisateur.

---

## 12. Lab guidé T10

### 12.1 Objectif

Concevoir l'architecture SASE d'une entreprise multi-sites avec télétravailleurs et applications cloud, et justifier le choix entre pare-feu local et services cloud.

### 12.2 Énoncé

Entreprise « Delta Services » : 45 agences (10 à 80 utilisateurs), 1 siège et 1 centre de données, 600 télétravailleurs, ERP privé dans le centre de données, Microsoft 365, Salesforce, un site industriel avec réseau OT, 4 pays (France, Allemagne, Espagne, Maroc).

### 12.3 Travail à produire

1. **Schéma d'architecture** : agences (WAN Edge), siège, DC, télétravailleurs (agent), service SSE (2 PoP par région), IdP, SIEM.
2. **Matrice de redirection** : pour chaque type de flux (web, SaaS, applications privées, OT, voix), choix du chemin et de la fonction de sécurité.
3. **Tableau pare-feu local ou cloud** par type de site, avec justification.
4. **Politique unifiée** : 8 règles types (modèle section 9.3).
5. **Plan de migration** en phases avec durées et indicateurs.
6. **Risques et parades** (au moins 6), dont panne du service SSE, couverture au Maroc, application incompatible avec l'inspection.

### 12.4 Éléments de correction

| Flux | Chemin | Fonction |
|---|---|---|
| Web des agences | Tunnel vers PoP SSE | SWG + DNS |
| Web des télétravailleurs | Agent vers PoP SSE | SWG + DNS |
| Microsoft 365, Salesforce | Sortie locale ou SSE, avec optimisation | CASB (API) + inline pour téléversements |
| ERP privé | ZTNA (connecteurs au DC) pour télétravailleurs ; overlay SD-WAN pour agences | ZTNA / segmentation |
| OT | Segment isolé, pare-feu **local**, aucune sortie Internet | Pare-feu local, pas de SSE |
| Voix | Overlay SD-WAN avec AAR, pas d'inspection | QoS, SLA |

| Type de site | Sécurité | Justification |
|---|---|---|
| Petites agences | SSE (cloud) + pare-feu simple local pour inter-segments | Pas d'équipe locale, simplicité |
| Site industriel | Pare-feu local | Contrainte OT, faible tolérance à la latence |
| Maroc | Pare-feu local + SSE si PoP disponible, sinon pare-feu local | Couverture à vérifier |
| Siège / DC | Pare-feu de centre de données + SSE pour l'accès Internet | Exigences fortes |

### 12.5 Grille

| Critère | Points |
|---|---|
| Architecture cohérente | 5 |
| Matrice de redirection argumentée | 4 |
| Choix local/cloud justifié par site | 4 |
| Politique unifiée réaliste | 3 |
| Plan de migration | 2 |
| Risques et parades | 2 |

---

## 13. Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **PoP** | Point de présence : centre de traitement du service cloud |
| **IdP** | Fournisseur d'identité |
| **SAML / OIDC** | Protocoles de fédération d'identité |
| **SCIM** | Protocole de provisionnement automatique des comptes |
| **MDM / EDR** | Gestion des terminaux mobiles / détection et réponse sur les terminaux |
| **PAC / WPAD** | Fichier de configuration automatique de proxy / découverte automatique |
| **PEP / PDP / PE / PA** | Points d'application et de décision de politique (NIST 800-207) |
| **Microsegmentation** | Segmentation fine par charge de travail ou application |
| **Shadow IT** | Applications utilisées sans validation de l'entreprise |
| **Posture** | État de conformité d'un terminal |
| **RBI** | Isolation de navigation à distance |
| **DEM** | Mesure de l'expérience numérique |
| **SD-WAN as a service** | Service géré incluant équipements et contrôleurs |
| **Hairpin** | Détour inutile du trafic |

### B. Cas d'usage détaillés

#### Cas 1. Remplacer 45 proxys et pare-feu d'agence

Chaque agence disposait d'un petit pare-feu avec filtrage web sous licence, mises à jour disparates. **Décision** : tunnels vers le SSE, pare-feu simple pour l'inter-segments. **Résultat** : politique de filtrage unique, 40 % de réduction du coût de licences, délai de correction d'une règle : de 3 semaines à 15 minutes. **Leçon** : la **politique unique** est le vrai bénéfice, avant le coût.

#### Cas 2. Fin du VPN pour les prestataires

Prestataires accédant à une application de support par VPN, avec accès réseau large (incident constaté : un poste de prestataire infecté). **Décision** : ZTNA avec isolation de navigateur, terminal non géré autorisé **sans** accès réseau. **Résultat** : surface d'attaque réduite, pas de client VPN à installer. **Leçon** : ZTNA est particulièrement adapté aux **tiers**.

#### Cas 3. Fuite de données via un stockage personnel

Un salarié téléverse un fichier de clients vers son stockage cloud personnel. **Détection** : le CASB signale le volume anormal. **Réponse** : blocage des téléversements vers les stockages non approuvés, sensibilisation. **Leçon** : le **shadow IT** est un risque majeur et invisible sans CASB.

#### Cas 4. Panne du service SSE un lundi matin

Le fournisseur SSE connaît une panne régionale de 90 minutes. L'entreprise, avec tunnels vers deux PoP et un **repli défini** (bascule vers la sortie locale avec pare-feu embarqué pour les petites agences), limite l'impact à 4 minutes. **Leçon** : décidez à l'avance **fail-open ou fail-close** et testez-le. Fail-close protège mais interrompt ; fail-open maintient l'activité mais réduit la sécurité.

#### Cas 5. Application bancaire qui refuse l'inspection TLS

L'application des paiements rejette tout certificat intermédiaire. **Solution** : exception d'inspection pour ce domaine, avec les autres contrôles maintenus (filtrage DNS, restriction de destination). **Leçon** : tenez un **registre des exceptions** avec propriétaire et date de révision.

#### Cas 6. Pays sans PoP proche

La filiale marocaine a le PoP le plus proche à 80 ms. **Décision** : pare-feu local pour ce pays, avec politique synchronisée et journaux vers le SIEM central. **Leçon** : un cahier des charges doit contenir la **couverture par pays**, vérifiée par des mesures.

### C. Retours d'expérience

1. **Commencez par l'identité.** Un SSE sans identité propre (comptes partagés, pas de MFA) est une façade.
2. **Le déchiffrement TLS est un projet à part entière** : exceptions, certificats déployés, information des salariés, droit du travail.
3. **Méfiez-vous du « tout-en-un »** vendu sans démonstration d'une politique réellement unifiée.
4. **Mesurez la latence réelle** vers les PoP depuis vos sites. Les cartes de couverture des fournisseurs sont optimistes.
5. **Définissez fail-open ou fail-close** par type de flux, et **testez** : une panne du service est une question de **quand**, non de **si**.
6. **ZTNA application par application.** Vouloir tout migrer d'un coup crée des blocages et des refus de la part des métiers.
7. **Impliquez les métiers et les RH** : visibilité sur les usages = sujet de confiance et de conformité.
8. **Le SASE n'est pas un produit, c'est une trajectoire.** Visez trois à cinq ans, avec des étapes de valeur tous les trimestres.

### D. Exercices supplémentaires

1. Rédigez 6 règles d'une politique unifiée pour un cabinet comptable (données clients sensibles, télétravail, stockage cloud).
2. Comparez en un tableau deux options pour 30 agences : « pare-feu local avec abonnement de sécurité » et « SSE avec tunnels ». Critères : coût, délai de correction de règle, couverture télétravail, latence, résilience, souveraineté.
3. Rédigez le PAC d'une entreprise dont les applications `*.cloud-rh.example` ne supportent pas l'inspection et qui exige un repli bloquant.
4. Votre direction pense que « le SASE remplace le SD-WAN ». Rédigez une réponse de 8 lignes.

<details>
<summary>Corrigés</summary>

4. Le SASE **inclut** le SD-WAN comme composant réseau : il ne le remplace pas. Le SD-WAN relie les sites et gère les chemins ; le SSE protège l'accès (web, SaaS, applications privées). Certains fournisseurs fusionnent les deux dans un seul service, mais fonctionnellement, les deux briques subsistent, y compris la connectivité des sites, la qualité de service et la résilience des liens.

</details>

### E. Mini-projet (8 heures)

Produisez le **dossier d'architecture SASE** d'une entreprise fictive : modèle de menace, flux, choix d'intégration (même éditeur ou non), politique unifiée de 15 règles, plan de migration à 6 phases avec indicateurs, analyse de risques (dont fail-open/fail-close), coûts sur 3 ans. Présentez-le à un comité de direction fictif en 10 minutes.

---

## 14. Quiz

1. Quelle est la différence entre SASE et SSE ?
2. Citez les cinq fonctions typiques d'un SSE.
3. Pourquoi le ZTNA limite-t-il le mouvement latéral ?
4. Qu'est-ce qu'un PEP dans NIST SP 800-207 ?
5. Que signifie fail-close ?
6. Quel est le risque principal du déchiffrement TLS généralisé ?
7. Pourquoi commencer par l'identité ?

<details>
<summary>Réponses</summary>

1. Le SASE combine SD-WAN et sécurité cloud ; le SSE est la partie sécurité. 2. SWG, CASB, ZTNA, FWaaS, DLP (et éventuellement RBI, DEM). 3. Il n'ouvre l'accès qu'à des applications précises, après vérification, et non au réseau. 4. Le point d'application de la politique (Policy Enforcement Point) qui autorise ou coupe la session. 5. En cas de panne du service, le trafic est bloqué plutôt que de contourner la sécurité. 6. Charge de calcul, incompatibilités d'applications, aspects légaux et de vie privée. 7. Toutes les politiques reposent sur « qui » demande l'accès ; sans identité fiable, la politique est inefficace.

</details>

---

## 15. Aide-mémoire

```text
SASE = SD-WAN + SSE        SSE = SWG + CASB + ZTNA (+ FWaaS + DLP + RBI + DEM)
ZTNA : accès par application, connecteurs en sortie, posture vérifiée    VPN : accès au réseau
NIST 800-207 : PE (décide), PA (orchestre), PEP (applique) ; accès par session, politique dynamique
Orientation : tunnels IPsec/GRE (sites) | agent (mobiles) | PAC (navigateurs) | connecteurs (apps privées)
Conception : latence PoP < 20 ms, 2 PoP, fail-open/close décidé et testé, couverture par pays, exceptions TLS documentées
Migration : identité -> SWG/DNS -> ZTNA -> CASB/DLP -> consolidation
```

## 16. Pour aller plus loin

- NIST SP 800-207, Zero Trust Architecture : <https://csrc.nist.gov/publications/detail/sp/800-207/final>
- Cisco (Catalyst SD-WAN avec Secure Access / SIG), Fortinet (FortiSASE), Palo Alto Networks (Prisma Access), Zscaler, Cato, Netskope : documentation des éditeurs.
- CISA, Zero Trust Maturity Model : <https://www.cisa.gov/zero-trust-maturity-model>

> Module précédent : [SW11](../SW11-panorama-solutions/SW11-panorama-solutions.md) | Module suivant : [SW13](../../04-partie-d-conception-migration-exploitation/SW13-conception-architecture/SW13-conception-architecture.md)
