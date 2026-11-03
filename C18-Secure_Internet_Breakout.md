# SD-WAN : Secure Internet Breakout, sortir localement sans s'exposer

Cours complet, pour tout public. Objectif : comprendre comment un site (magasin, agence, entrepôt) peut accéder à Internet directement depuis sa propre connexion, tout en restant protégé.

Avertissement : les exemples de distributeur sont pédagogiques et fictifs. Ils ne décrivent pas l'architecture réelle d'une entreprise existante. Les noms de fonctions et de produits varient selon les constructeurs, il faut toujours consulter la documentation officielle de la plateforme utilisée.

---

## 1. L'idée en une minute

Avant le SD-WAN, presque tout le trafic Internet d'un site remontait vers le siège (le centre de données), traversait les pare-feux centraux, puis sortait vers Internet. C'était sûr mais lent et coûteux.

Le **Secure Internet Breakout** (sortie Internet sécurisée locale) change cela : le site sort directement sur Internet par son propre accès, mais avec une protection de sécurité appliquée sur place ou dans le cloud.

Image simple : un magasin qui, au lieu de faire livrer toutes ses marchandises par le siège, se fait livrer directement par les fournisseurs, mais avec un contrôle qualité à la porte d'entrée.

| Approche | Chemin du trafic Internet | Avantage | Inconvénient |
|---|---|---|---|
| Centralisée (backhaul) | Site, siège, Internet | Sécurité centralisée | Latence, saturation des liens, coût |
| Sortie locale non protégée | Site, Internet | Rapide | Dangereux, site exposé |
| Secure Internet Breakout | Site, contrôle de sécurité, Internet | Rapide et protégé | Demande de bien concevoir la sécurité |

---

## 2. Vocabulaire de base

| Terme | Sens simple |
|---|---|
| Breakout | Point de sortie vers Internet |
| Backhaul | Faire remonter le trafic vers un site central avant de sortir |
| DIA (Direct Internet Access) | Accès direct à Internet depuis le site |
| NAT | Traduction des adresses privées en adresse publique pour sortir |
| Pare-feu | Filtre qui autorise ou bloque les flux |
| IPS / IDS | Détection ou blocage des attaques connues |
| SWG (Secure Web Gateway) | Passerelle de sécurité web (filtrage des sites, analyse des fichiers) |
| CASB | Contrôle de l'usage des applications cloud |
| FWaaS | Pare-feu fourni sous forme de service cloud |
| ZTNA | Accès aux applications selon l'identité, sans confiance implicite |
| SASE | Combinaison réseau SD-WAN et sécurité cloud |
| SSE | Partie sécurité du SASE (SWG, CASB, ZTNA, FWaaS) |
| Segment / VRF | Réseau virtuel isolé sur le même équipement |
| SaaS | Application en ligne (messagerie, collaboration, etc.) |
| Fail-open / Fail-close | Que fait la sécurité si elle tombe : laisser passer ou tout bloquer |

---

## 3. Pourquoi sortir localement

### Les limites du backhaul

1. **Latence** : un appel vidéo depuis un magasin du sud qui passe par un siège au nord puis revient est moins fluide.
2. **Saturation** : les liens vers le siège transportent aussi du trafic Internet banal (vidéos, mises à jour), ce qui gêne les applications métier.
3. **Coût** : les liens privés et MPLS coûtent cher par rapport à un accès Internet.
4. **Applications cloud** : avec Microsoft 365 ou Salesforce, faire un détour par le siège n'a plus de sens, l'application est déjà sur Internet.

### Ce que le breakout local apporte

| Bénéfice | Explication |
|---|---|
| Meilleure performance | Chemin direct vers les applications cloud |
| Économie | Le trafic web ne charge plus les liens vers le siège |
| Résilience | Chaque site garde un accès même si le siège a un souci |
| Évolutivité | Ajouter un site ne surcharge pas les pare-feux centraux |

---

## 4. Le risque : ce qui change quand on sort localement

Quand un site sort directement sur Internet, il devient **visible depuis Internet** et **joignable par des attaquants**. Ce que le siège protégeait de façon centralisée doit maintenant l'être sur chaque site, ou dans le cloud.

| Menace | Exemple concret | Conséquence possible |
|---|---|---|
| Malware téléchargé | Un employé télécharge un fichier piégé | Infection du réseau du magasin |
| Phishing | Faux site qui vole des identifiants | Compromission de comptes |
| Exfiltration de données | Un logiciel malveillant envoie des données vers l'extérieur | Fuite d'informations |
| Scan et attaque entrante | Un robot teste les ports ouverts du site | Intrusion |
| Mouvement latéral | Un poste infecté attaque les caisses | Atteinte aux flux de paiement |
| DNS détourné | Résolution vers un faux serveur | Redirection vers un site malveillant |

Principe directeur : **sortir ne doit jamais signifier ouvrir**. Le site peut initier des connexions vers l'extérieur, mais ne doit accepter aucune connexion non sollicitée depuis Internet.

---

## 5. Les briques de sécurité d'un breakout sécurisé

### 5.1 Segmentation avant tout

Tous les flux n'ont pas le droit de sortir. On commence par décider **quels segments** ont accès à Internet.

| Segment | Accès Internet direct ? | Remarque |
|---|---|---|
| Paiement (caisses, terminaux) | Non | Uniquement vers les serveurs de paiement autorisés |
| Gestion magasin | Limité | Seulement certaines applications autorisées |
| Vidéosurveillance | Non, ou très limité | Flux vers l'enregistreur central |
| Wifi invité | Oui, isolé | Aucun accès au reste du réseau |
| Objets connectés (balances, étiquettes) | Non, ou vers le fournisseur uniquement | Liste blanche de destinations |

### 5.2 Traduction d'adresse (NAT)

Les équipements du site utilisent des adresses privées. Pour sortir, le routeur les traduit en adresse publique de l'accès Internet local. Le NAT, par défaut, bloque aussi les connexions entrantes non sollicitées, ce qui est une première protection, mais **insuffisante seule**.

### 5.3 Pare-feu à état et par zones

Un pare-feu à état suit les conversations : il autorise les réponses aux connexions initiées de l'intérieur, et bloque le reste. Un pare-feu **par zones** associe des règles entre zones (par exemple de la zone "magasin" vers la zone "Internet").

### 5.4 Prévention d'intrusion (IPS)

L'IPS analyse le trafic pour reconnaître des schémas d'attaque connus et les bloquer.

### 5.5 Filtrage web et DNS

- Filtrage par catégories : bloquer sites malveillants, jeux d'argent, contenu adulte selon la politique.
- Sécurité DNS : refuser la résolution de domaines connus comme dangereux, avant même la connexion.

### 5.6 Protection contre les logiciels malveillants

Analyse des fichiers téléchargés, réputation des sites, bac à sable (analyse dans un environnement isolé).

### 5.7 Inspection du trafic chiffré

La plupart du trafic web est chiffré (HTTPS). Sans inspection, la sécurité ne voit pas ce qui passe. L'inspection déchiffre, analyse, puis rechiffre. À manier avec précaution :
- consomme beaucoup de puissance de calcul,
- pose des questions de vie privée et de conformité (ne pas déchiffrer les sites bancaires ou de santé personnels des employés, par exemple),
- demande un certificat d'inspection déployé sur les postes.

### 5.8 Contrôle des applications cloud (CASB)

Permet d'autoriser une application d'entreprise et de bloquer ou limiter une version personnelle équivalente, pour éviter les fuites de données.

### 5.9 Journalisation et supervision

Tout flux autorisé ou bloqué doit laisser une trace exploitable : source, destination, application, décision, heure.

---

## 6. Les grandes architectures possibles

### Architecture A : breakout local avec sécurité sur le routeur

La sécurité (pare-feu, IPS, filtrage) tourne **sur l'Edge lui-même** ou sur un équipement à côté.

```
Poste du magasin --> Edge (pare-feu, IPS, filtrage) --> Internet
```

Avantages : chemin le plus court, indépendant du cloud.
Limites : puissance de l'Edge limitée, politique à maintenir sur chaque site, visibilité répartie.

### Architecture B : breakout local avec sécurité dans le cloud (SASE/SSE)

L'Edge envoie le trafic Internet par un **tunnel chiffré** vers un service de sécurité cloud, qui filtre puis relaie vers Internet.

```
Poste --> Edge --(tunnel IPsec ou GRE)--> Service de sécurité cloud --> Internet
```

Avantages : politique unique et centralisée, puissance quasi illimitée, protection homogène, mises à jour automatiques.
Limites : dépendance au fournisseur, coût par utilisateur ou par volume, tunnel à superviser.

### Architecture C : hybride (la plus courante)

On combine selon le type de flux :

| Flux | Chemin |
|---|---|
| Applications SaaS de confiance (messagerie, collaboration) | Sortie locale directe, optimisée |
| Navigation web générale | Tunnel vers la sécurité cloud (SWG) |
| Wifi invité | Sortie locale isolée avec filtrage |
| Applications internes | Overlay SD-WAN vers le siège ou le cloud privé |
| Paiement | Overlay uniquement, jamais Internet direct |

### Architecture D : sortie régionale (hub de breakout)

Au lieu de sortir sur chaque site, on regroupe la sortie dans quelques points régionaux sécurisés. Compromis entre centralisation et local, utile pour de très petits sites sans équipement de sécurité suffisant.

### Tableau de choix

| Critère | A : sécurité sur l'Edge | B : sécurité cloud | C : hybride | D : hubs régionaux |
|---|---|---|---|---|
| Simplicité de gestion | Moyenne | Bonne | Moyenne | Bonne |
| Performance | Très bonne | Bonne | Très bonne | Moyenne |
| Coût par site | Élevé (matériel plus puissant) | Variable (abonnement) | Variable | Faible |
| Cohérence de la politique | Moyenne | Très bonne | Bonne | Bonne |
| Adapté aux petits sites | Moyen | Très bon | Bon | Bon |

---

## 7. Comment le SD-WAN décide quoi faire sortir

Le SD-WAN permet de choisir, **flux par flux**, le chemin à emprunter. Cela repose sur :

1. **L'identification de l'application** : le routeur reconnaît le trafic (messagerie, vidéo, paiement) grâce à l'analyse des paquets, des noms de domaine et de bases d'applications.
2. **Les politiques de trafic** : règles du type "si application = collaboration en ligne, sortir localement", "si segment = paiement, interdire la sortie Internet".
3. **La mesure de la qualité des chemins** : le routeur teste en continu latence, perte et gigue sur chaque lien, et choisit le meilleur.
4. **Les routes par défaut et les segments** : chaque segment a sa propre table de routage, ce qui permet que le segment "invité" ait une sortie Internet et que le segment "paiement" n'en ait pas.

### Optimisation des applications SaaS

Certaines plateformes mesurent la qualité du chemin vers une application SaaS depuis chaque lien du site (fibre, 4G, etc.) et choisissent automatiquement la meilleure sortie. Si la fibre se dégrade, le trafic bascule vers l'autre lien, sans intervention.

---

## 8. Lien avec la PKI vue précédemment

La confiance dans les équipements est un prérequis du breakout :

- seul un Edge authentifié (certificat valide, numéro de série dans l'allowed list) reçoit les politiques de sécurité et les règles de breakout ;
- un Edge volé et invalidé ne reçoit plus de politique et ne peut plus rejoindre l'overlay ;
- les tunnels vers la sécurité cloud utilisent aussi des identités (certificats, clés partagées ou secrets) à gérer et à renouveler.

Un breakout sécurisé suppose donc une identité d'équipement solide.

---

## 9. Cas pratique : un distributeur fictif (scénario pédagogique)

### Contexte

Un grand distributeur de plusieurs milliers de magasins veut :
- améliorer les performances de sa messagerie et de ses outils collaboratifs en ligne,
- offrir un wifi aux clients sans risque pour les caisses,
- réduire la charge sur les liens vers le siège,
- rester conforme aux exigences du paiement par carte.

### Segments d'un magasin type

| Segment | Contenu | Politique Internet |
|---|---|---|
| Paiement | Caisses, terminaux | Aucune sortie Internet, seulement overlay vers serveurs de paiement |
| Gestion | Postes du personnel, balances | Sortie locale filtrée, liste d'applications autorisées |
| Caméras | Enregistreurs et caméras | Aucune sortie, flux vers stockage central |
| Wifi invité | Smartphones des clients | Sortie locale isolée, filtrage de contenu, débit limité |
| Objets connectés | Étiquettes, capteurs | Seulement vers le fournisseur concerné |

### Choix d'architecture

Architecture hybride (C) :

| Flux | Décision |
|---|---|
| Messagerie et collaboration en ligne | Sortie locale directe avec optimisation du chemin |
| Navigation web du personnel | Tunnel vers le service de sécurité cloud |
| Wifi invité | Sortie locale, pare-feu et filtrage de contenu sur place |
| Paiement | Overlay uniquement |
| Mises à jour des équipements | Sortie locale vers des destinations listées |

### Scénario : une journée de test

1. À 9 h, une employée ouvre un lien suspect. Le filtrage DNS bloque le domaine, aucune connexion n'est établie.
2. À 11 h, un client connecté au wifi invité tente de scanner le réseau du magasin. Le segment invité est isolé, le scan échoue et l'événement est journalisé.
3. À 14 h, la fibre du magasin tombe. Les applications SaaS et la navigation basculent sur le lien 4G/5G sans coupure notable. Les caisses, qui passent par l'overlay, basculent aussi.
4. À 16 h, le tunnel vers la sécurité cloud devient indisponible. Selon la politique, deux comportements possibles :
   - fail-close : la navigation web est bloquée jusqu'au rétablissement (plus sûr),
   - fail-open : la navigation passe en direct avec la protection locale minimale (plus disponible, plus risqué).
   Pour le segment de gestion, la décision retenue ici est fail-close avec un message clair à l'utilisateur.
5. À 18 h, l'équipe sécurité consulte les journaux : sites bloqués, tentatives d'accès, volumes par segment.

---

## 10. Politique de sécurité de référence (modèle)

1. Aucune connexion entrante non sollicitée depuis Internet vers un site.
2. Chaque segment possède une politique de sortie Internet écrite et validée.
3. Le segment paiement n'a jamais d'accès Internet direct.
4. Le wifi invité est isolé de tous les autres segments.
5. Le trafic web du personnel est filtré (catégories, DNS, malwares).
6. L'inspection du trafic chiffré exclut certaines catégories (banque, santé) selon la politique de conformité.
7. Les journaux de sécurité sont centralisés et conservés selon les obligations légales.
8. Les tunnels vers la sécurité cloud sont supervisés, avec alertes en cas de coupure.
9. Le comportement en cas de panne (fail-open ou fail-close) est défini par segment.
10. La politique est revue au moins une fois par an et après chaque incident majeur.

---

## 11. Exemple de politique exprimée en langage simple

À titre illustratif, indépendamment d'un constructeur précis :

```
Règle 1 : source = segment Paiement, destination = Internet        -> BLOQUER
Règle 2 : source = segment Paiement, destination = serveurs paiement
          via overlay                                               -> AUTORISER
Règle 3 : source = segment Gestion, application = messagerie cloud  -> SORTIE LOCALE
Règle 4 : source = segment Gestion, web général                     -> TUNNEL SECURITE CLOUD
Règle 5 : source = segment Invité, destination = segments internes  -> BLOQUER
Règle 6 : source = segment Invité, destination = Internet           -> SORTIE LOCALE FILTREE
Règle 7 : source = Internet, destination = tout segment             -> BLOQUER (sauf retours)
Règle 8 : tout le reste                                             -> BLOQUER ET JOURNALISER
```

Principe fondamental : **refuser par défaut**, n'autoriser que ce qui est nécessaire.

---

## 12. Performance et dimensionnement

| Fonction de sécurité | Impact sur le débit | À savoir |
|---|---|---|
| NAT et pare-feu simple | Faible | Souvent matériel dédié |
| IPS | Moyen | Réduit le débit maximal |
| Filtrage web et DNS | Faible à moyen | Souvent déporté dans le cloud |
| Inspection TLS | Élevé | Peut diviser le débit par plusieurs fois |
| Antimalware avec bac à sable | Moyen à élevé | Souvent en cloud |

Règles pratiques :
- Dimensionner le routeur en fonction du débit **avec toutes les fonctions de sécurité activées**, pas du débit annoncé de base.
- Prévoir une marge de croissance (les applications cloud consomment de plus en plus).
- Surveiller l'usage réel avant de généraliser un modèle à tous les sites.

---

## 13. Résilience

| Panne | Conséquence | Mesure |
|---|---|---|
| Accès Internet principal tombe | Perte de sortie locale | Deuxième lien (4G/5G ou autre opérateur) |
| Tunnel vers sécurité cloud tombe | Plus de filtrage web | Deuxième tunnel vers un autre point de présence, politique fail-open ou fail-close définie |
| Service de sécurité cloud indisponible | Idem | Basculer vers un autre fournisseur ou point d'entrée, plan de repli documenté |
| Surcharge du lien local | Dégradation | Qualité de service, limitation du wifi invité |
| Panne du Edge | Magasin isolé | Edge redondant pour les gros sites, remplacement rapide avec zero touch |

---

## 14. Supervision et conformité

### Indicateurs à suivre

| Indicateur | Pourquoi |
|---|---|
| Volume et débit par segment | Détecter une anomalie |
| Sites et catégories bloqués | Mesurer l'efficacité du filtrage |
| Tentatives de connexion entrantes | Détecter des scans |
| État des tunnels de sécurité | Prévenir les coupures |
| Basculements de lien | Évaluer la qualité des accès |
| Alertes de malware | Réagir vite |

### Conformité

- **Paiement par carte (PCI DSS)** : isoler strictement les segments qui traitent des données de cartes. Pas de sortie Internet directe depuis ces segments, journalisation et revues régulières.
- **Protection des données personnelles (RGPD)** : l'inspection du trafic chiffré et la conservation des journaux doivent respecter la vie privée des employés et des clients, avec une base légale, une information claire et une durée de conservation limitée.
- **Vidéosurveillance** : les flux et les enregistrements restent confinés au réseau interne et aux serveurs autorisés.

---

## 15. Erreurs classiques

| Erreur | Conséquence |
|---|---|
| Ouvrir la sortie Internet sans segmentation | Un poste infecté atteint les caisses |
| Se fier au NAT seul | Aucune protection contre les attaques initiées de l'intérieur |
| Oublier de borner le wifi invité | Saturation des liens, risques d'abus |
| Activer l'inspection TLS sans dimensionner le matériel | Débit effondré |
| Ne pas définir fail-open ou fail-close | Comportement imprévisible en cas de panne |
| Politique différente d'un site à l'autre | Failles et incompréhension |
| Ne pas superviser les tunnels de sécurité | Coupure découverte par les utilisateurs |
| Oublier de déployer le certificat d'inspection | Erreurs de sécurité dans les navigateurs |
| Autoriser par défaut | Le contraire du principe de moindre privilège |

---

## 16. Plan de déploiement progressif

| Phase | Action | Objectif |
|---|---|---|
| 1 | Cartographier les flux et segments | Savoir ce qui doit sortir |
| 2 | Définir les politiques par segment | Cadre de sécurité clair |
| 3 | Choisir l'architecture (A, B, C ou D) | Modèle cible |
| 4 | Pilote sur 5 à 10 sites variés | Valider performance et sécurité |
| 5 | Mesurer (débit, incidents, retours utilisateurs) | Ajuster |
| 6 | Déploiement par vagues | Réduire le risque |
| 7 | Exploitation et revues régulières | Maintenir la sécurité dans la durée |

---

## 17. Checklist avant ouverture d'un breakout sur un site

- Les segments sont définis et isolés.
- La politique de sortie par segment est validée.
- Le NAT est configuré et aucune connexion entrante n'est autorisée.
- Le pare-feu et les protections activées sont dimensionnés.
- Le filtrage web et DNS est actif.
- Les tunnels vers la sécurité cloud sont redondants et supervisés.
- Le comportement en cas de panne est défini et testé.
- Le lien de secours est fonctionnel.
- Les journaux remontent vers la supervision centrale.
- Un test d'intrusion ou d'audit est réalisé sur un site pilote.
- L'Edge est authentifié, présent dans l'allowed list, avec une identité valide.

---

## 18. FAQ

**Le NAT suffit-il à protéger un site ?**
Non. Il bloque les connexions entrantes non sollicitées, mais ne protège pas contre un malware téléchargé volontairement ou un site malveillant visité.

**Pourquoi ne pas tout faire passer par le siège pour être plus sûr ?**
Cela fonctionne, mais crée de la latence, un goulot d'étranglement et un coût élevé. Un breakout bien conçu offre une sécurité équivalente avec de meilleures performances.

**Que signifie SASE ?**
C'est l'association du réseau SD-WAN et d'une sécurité fournie dans le cloud, avec une politique unique appliquée partout.

**Faut-il déchiffrer tout le trafic ?**
Non. L'inspection du trafic chiffré est puissante mais coûteuse et sensible. On l'applique de façon ciblée, avec des exclusions (banque, santé).

**Fail-open ou fail-close ?**
Cela dépend du risque. Pour un segment sensible, fail-close. Pour un usage où la disponibilité prime, fail-open avec des protections locales de base. L'important est de décider et de documenter.

**Un petit site a-t-il besoin du même niveau de sécurité qu'un gros ?**
Le niveau de protection doit être cohérent. Un petit site peut s'appuyer davantage sur la sécurité cloud pour éviter du matériel coûteux.

**Un breakout local rend-il le réseau interne plus vulnérable ?**
Pas si les segments sont isolés et si le principe "refuser par défaut" est appliqué. Le risque vient d'une mauvaise conception, pas du breakout en lui même.

---

## 19. Glossaire complémentaire

| Terme | Définition simple |
|---|---|
| Bac à sable (sandbox) | Environnement isolé où l'on exécute un fichier suspect pour l'observer |
| Exfiltration | Sortie illégitime de données vers l'extérieur |
| Gigue | Variation de la latence, gênante pour la voix et la vidéo |
| IPsec | Protocole de tunnel chiffré couramment utilisé vers les services cloud |
| GRE | Protocole de tunnel simple, souvent associé à un chiffrement séparé |
| Liste blanche | Liste des destinations explicitement autorisées |
| Moindre privilège | N'accorder que les droits strictement nécessaires |
| Point de présence (PoP) | Site d'un fournisseur cloud où se connectent les clients |
| Qualité de service (QoS) | Priorisation de certains flux sur d'autres |
| Zero trust | Principe selon lequel aucune connexion n'est fiable par défaut |

---

## 20. Questions de révision

1. Quelle différence entre backhaul et breakout local ?
2. Pourquoi le NAT seul est-il insuffisant ?
3. Quel segment ne doit jamais avoir d'accès Internet direct dans un magasin ?
4. Quelle différence entre fail-open et fail-close ?
5. Pourquoi l'inspection du trafic chiffré demande-t-elle de la prudence ?
6. Quel est l'intérêt de l'architecture hybride ?
7. Quel lien existe entre la PKI et le breakout sécurisé ?

Réponses courtes :
1. Le backhaul fait remonter le trafic vers le siège avant de sortir, le breakout sort directement depuis le site.
2. Il bloque les connexions entrantes non sollicitées mais ne détecte ni malware ni site malveillant.
3. Le segment paiement.
4. Fail-open laisse passer le trafic quand la sécurité est indisponible, fail-close le bloque.
5. Elle consomme beaucoup de puissance et pose des enjeux de vie privée et de conformité.
6. Elle adapte le chemin et la protection à chaque type de flux.
7. Seuls des équipements authentifiés reçoivent les politiques et peuvent établir les tunnels de sécurité.

---

## 21. Exercice final

**Énoncé.** Un magasin de proximité dispose d'une fibre et d'une carte 4G/5G de secours. Il a quatre segments : paiement, gestion, caméras, wifi invité. Le groupe veut que la messagerie en ligne sorte localement, que la navigation web soit filtrée dans le cloud, et que les caisses n'aient aucun accès Internet.

**Questions.**
1. Quelle décision pour chaque segment ?
2. Que se passe-t-il si la fibre tombe ?
3. Que faire si le tunnel vers la sécurité cloud tombe pour le segment gestion ?

**Proposition de réponse.**
1. Paiement : overlay uniquement. Gestion : messagerie en sortie locale, web via tunnel cloud. Caméras : flux interne vers stockage central, pas d'Internet. Wifi invité : sortie locale isolée et filtrée.
2. Le trafic bascule sur la 4G/5G, avec priorité aux flux métier et limitation du wifi invité.
3. Appliquer la politique prévue : fail-close pour la navigation de gestion avec message clair, et basculer vers un second tunnel si disponible.

---

## 22. À retenir

- Sortir localement améliore performance et coût, mais déplace la responsabilité de la sécurité vers chaque site ou vers le cloud.
- La segmentation est la première barrière : chaque segment a sa propre politique de sortie.
- Le paiement n'a jamais d'accès Internet direct.
- Principe central : refuser par défaut, autoriser uniquement le nécessaire.
- L'architecture hybride est la plus courante : sortie directe pour les applications de confiance, sécurité cloud pour le web, isolation pour les invités.
- Définir à l'avance le comportement en cas de panne (fail-open ou fail-close).
- Superviser, journaliser et revoir régulièrement la politique.
- Un breakout sûr repose sur des équipements authentifiés, d'où le lien avec la PKI du SD-WAN.

---
---

# Partie 2 : Scénarios concrets inspirés d'un grand distributeur (Carrefour comme exemple pédagogique)

Avertissement important : Carrefour est utilisé ici uniquement comme exemple parlant, parce que tout le monde connaît le fonctionnement d'un grand distributeur et de ses différents formats (hypermarchés, supermarchés, magasins de proximité, drive, entrepôts). Je ne connais pas l'architecture réseau réelle de Carrefour. Tous les scénarios, noms de segments, numéros de magasins, horaires, incidents et choix techniques ci-dessous sont **inventés à but pédagogique**.

---

## A. Les formats de magasins et leurs besoins Internet

| Format (exemple) | Taille | Équipements réseau (hypothèse) | Besoins Internet principaux |
|---|---|---|---|
| Hypermarché | Très grand, 30 à 80 caisses | 2 Edges redondants, fibre principale, lien de secours | Beaucoup d'applications, wifi invité, back office, drive, objets connectés |
| Supermarché | Moyen | 1 ou 2 Edges, fibre plus 4G/5G | Caisses, gestion, wifi invité, quelques objets connectés |
| Proximité (type City ou Express) | Petit | 1 Edge, fibre plus 5G | Peu de postes, caisses, messagerie, wifi invité limité |
| Drive | Entrepôt de préparation plus zone de retrait | 1 ou 2 Edges, terminaux mobiles | Application de préparation, retrait clients |
| Entrepôt logistique | Très grand | 2 Edges, lien dédié | Scanners, système d'entrepôt, camions, transporteurs |

Idée clé : un seul modèle de sécurité ne convient pas à tous les formats. La **politique** est commune, les **moyens** s'adaptent.

---

## B. Magasins de référence utilisés dans les scénarios

| Magasin (fictif) | Format | Particularités |
|---|---|---|
| Hyper 0142 | Hypermarché | 48 caisses, 12 caisses libre-service, drive attenant, wifi invité, balances connectées |
| Super 2210 | Supermarché | 14 caisses, galerie marchande voisine |
| City 5873 | Proximité urbaine | 2 caisses, un seul Edge, fibre plus 5G, ouvert tard le soir |
| Drive 7005 | Drive isolé | Terminaux mobiles de préparation, bornes de retrait |
| Entrepôt Nord | Logistique | 120 scanners, système d'entrepôt, camions |

Segments communs à tous les magasins :

| Segment | Contenu | Sortie Internet |
|---|---|---|
| PAIEMENT | Caisses, terminaux de paiement, caisses libre-service | Aucune, overlay uniquement |
| GESTION | Postes du personnel, imprimantes, balances | Limitée et filtrée |
| MOBILE_PRO | Terminaux mobiles du personnel et du drive | Limitée, applications métier uniquement |
| CAMERAS | Caméras et enregistreurs | Aucune |
| OBJETS | Étiquettes électroniques, capteurs de froid | Seulement vers le fournisseur concerné |
| INVITE | Wifi clients | Sortie locale isolée et filtrée |

---

## C. Matrice flux et décision (modèle commun)

| Flux | Segment | Chemin | Remarque |
|---|---|---|---|
| Paiement par carte | PAIEMENT | Overlay vers serveurs de paiement | Jamais Internet direct |
| Messagerie et collaboration en ligne | GESTION | Sortie locale directe optimisée | Meilleure qualité d'appels |
| Navigation web du personnel | GESTION | Tunnel vers sécurité cloud | Filtrage des catégories |
| Application de préparation de commandes | MOBILE_PRO | Overlay vers les serveurs applicatifs | Pas de passage par le web général |
| Mises à jour des étiquettes électroniques | OBJETS | Sortie locale vers le fournisseur uniquement | Liste blanche de destinations |
| Alarmes de température des frigos | OBJETS | Sortie locale chiffrée vers le service de supervision | Alertes prioritaires |
| Vidéo des caméras | CAMERAS | Overlay vers stockage central | Aucun accès Internet |
| Wifi des clients | INVITE | Sortie locale isolée, filtrage de contenu, débit limité | Aucun accès aux autres segments |
| Applications du personnel hors magasin | GESTION | ZTNA vers applications autorisées | Authentification renforcée |

---

## D. Scénarios détaillés

Chaque scénario suit le même plan : contexte, ce qui se passe, ce que fait le SD-WAN, leçon.

### Scénario 1 : la matinée normale d'un hypermarché (Hyper 0142)

**Contexte.** 7 h 30, ouverture du personnel. Les postes de gestion démarrent, les caisses se préparent, les balances se synchronisent.

**Ce qui se passe.**
1. Les postes de gestion ouvrent la messagerie en ligne et les outils collaboratifs.
2. Les caisses chargent les prix du jour depuis le siège.
3. Les étiquettes électroniques reçoivent les nouveaux prix.
4. Un employé consulte un site d'actualités.

**Ce que fait le SD-WAN.**
- Messagerie et collaboration : sortie locale directe, chemin le plus rapide mesuré en continu.
- Prix pour les caisses : overlay, jamais Internet.
- Étiquettes : sortie locale vers le seul domaine du fournisseur, rien d'autre.
- Site d'actualités : tunnel vers le service de sécurité cloud, catégorie autorisée.

**Leçon.** Un même site, quatre flux, quatre chemins différents. La décision se fait par segment et par application, pas par site.

### Scénario 2 : le lien frauduleux ouvert par un employé (Super 2210)

**Contexte.** 10 h 15, un responsable de rayon reçoit un message qui ressemble à un avis de livraison et clique sur un lien.

**Ce qui se passe.**
1. Le poste tente de résoudre le nom de domaine du lien.
2. Le service de sécurité DNS reconnaît ce domaine comme malveillant (hameçonnage) et refuse la résolution.
3. Le navigateur affiche une page de blocage.
4. Un événement est journalisé : poste, heure, domaine, catégorie.

**Ce que fait le SD-WAN.** Le trafic web du segment GESTION passe par la sécurité cloud, qui bloque avant toute connexion. L'équipe sécurité reçoit une alerte de niveau moyen.

**Si le domaine n'était pas connu.** L'analyse de réputation et le bac à sable peuvent ralentir ou bloquer le téléchargement du fichier. Une alerte demande une vérification manuelle du poste.

**Leçon.** La sécurité DNS et web est la première barrière. Elle protège même si l'utilisateur fait une erreur.

### Scénario 3 : le client curieux sur le wifi invité (City 5873)

**Contexte.** 21 h, un client connecté au wifi du magasin lance un outil de balayage réseau pour voir ce qui est joignable.

**Ce qui se passe.**
1. Le client ne trouve que sa propre adresse et la passerelle du wifi invité.
2. Toute tentative vers les autres réseaux est bloquée (règle INVITE vers segments internes : bloquer).
3. Chaque tentative est journalisée avec la source.
4. Après un seuil de tentatives, la session invité est automatiquement limitée puis coupée.

**Ce que fait le SD-WAN.** Le segment INVITE est un réseau virtuel séparé. Les autres segments n'existent tout simplement pas pour lui.

**Leçon.** La segmentation rend l'attaque inutile : on ne peut pas attaquer ce qu'on ne voit pas.

### Scénario 4 : le poste infecté qui tente d'atteindre les caisses (Hyper 0142)

**Contexte.** Un poste de gestion est infecté après l'ouverture d'une pièce jointe piégée.

**Ce qui se passe.**
1. Le logiciel malveillant tente de contacter son serveur de commande à l'extérieur : le filtrage web et le contrôle des destinations bloquent la connexion sortante.
2. Il tente de scanner le réseau local pour atteindre les caisses : les règles entre segments interdisent GESTION vers PAIEMENT.
3. L'outil de détection signale un comportement anormal (nombreuses tentatives refusées).
4. L'équipe sécurité isole le poste via une règle de quarantaine.

**Ce que fait le SD-WAN.** Les segments sont des îlots. Même infecté, le poste ne peut atteindre ni Internet de manière libre ni le segment de paiement.

**Leçon.** Le but n'est pas d'éviter toute infection, mais de limiter les dégâts (principe de confinement).

### Scénario 5 : la fibre coupée un samedi à 11 h (City 5873)

**Contexte.** Chantier voisin : la fibre du magasin est sectionnée en pleine affluence.

**Ce qui se passe.**
1. Le routeur détecte la perte du lien principal en quelques secondes.
2. Il bascule sur la 5G.
3. Les paiements, qui passent par l'overlay, continuent via la 5G.
4. Le wifi invité est automatiquement limité pour préserver la bande passante des flux métier.
5. Une alerte est envoyée au centre de supervision, un ticket est ouvert chez l'opérateur.

**Ce que fait le SD-WAN.** Surveillance continue des liens, politique de priorité (paiement et gestion d'abord, invités ensuite), retour automatique sur la fibre quand elle est rétablie, avec un délai de stabilisation pour éviter les allers retours.

**Leçon.** Un magasin avec un seul lien est un magasin fragile. Un lien de secours est un investissement rentable.

### Scénario 6 : le tunnel vers la sécurité cloud tombe (Super 2210)

**Contexte.** 16 h, le tunnel principal vers le service de sécurité cloud devient indisponible à cause d'un incident chez le fournisseur.

**Ce qui se passe.**
1. L'Edge détecte la perte du tunnel.
2. Il bascule vers un deuxième tunnel vers un autre point de présence du fournisseur.
3. Si les deux sont indisponibles, la politique par segment s'applique :

| Segment | Comportement si plus de sécurité cloud |
|---|---|
| GESTION | Fail-close pour le web général, messagerie en sortie locale conservée |
| INVITE | Fail-close, sortie locale limitée à des destinations de base ou coupure temporaire |
| OBJETS | Maintien vers le fournisseur concerné uniquement |
| PAIEMENT et CAMERAS | Non concernés, ils ne passent pas par Internet |

4. Un message clair apparaît aux utilisateurs : "Accès web temporairement indisponible".
5. Un incident est ouvert, avec suivi jusqu'au rétablissement.

**Leçon.** Décider à l'avance du comportement en cas de panne évite les improvisations et protège les segments sensibles.

### Scénario 7 : le pic du Black Friday (tous magasins)

**Contexte.** Forte affluence, beaucoup de clients sur le wifi, trafic de vidéos et de réseaux sociaux.

**Ce qui se passe.**
1. Le débit du wifi invité grimpe en quelques minutes.
2. La politique de qualité de service plafonne chaque session invité et limite le total du segment INVITE.
3. Les flux métier (paiement, messagerie, préparation de commandes) conservent leur priorité.
4. Un tableau de bord de supervision montre l'utilisation par segment.

**Ce que fait le SD-WAN.** Plafonnement du segment INVITE à une part maximale du lien, priorité absolue aux flux de paiement.

**Leçon.** Le wifi invité est un service de confort, jamais au détriment des caisses.

### Scénario 8 : un objet connecté compromis (Hyper 0142)

**Contexte.** Un capteur de température de chambre froide, mal mis à jour, est exploité par un attaquant.

**Ce qui se passe.**
1. Le capteur tente de joindre une adresse inconnue : le segment OBJETS autorise seulement le domaine du fournisseur, la tentative est bloquée.
2. Il tente de scanner d'autres segments : bloqué.
3. L'équipe voit une rafale de refus venant de cette adresse et isole l'équipement.
4. Une mise à jour correctrice est planifiée.

**Ce que fait le SD-WAN.** Liste blanche de destinations pour le segment OBJETS, confinement strict, alerte sur les comportements inhabituels.

**Leçon.** Les objets connectés sont souvent le maillon faible : on leur donne le strict minimum de droits.

### Scénario 9 : la mise à jour de milliers de caisses (tous magasins)

**Contexte.** Une nouvelle version du logiciel de caisse doit être déployée dans des milliers de magasins.

**Ce qui se passe.**
1. Le déploiement se fait par vagues (pilote, puis régions, puis le reste), jamais d'un seul coup.
2. Les fichiers sont téléchargés par l'overlay depuis un point de distribution central, pas depuis Internet.
3. Les téléchargements sont étalés dans la nuit pour ne pas saturer les liens.
4. Chaque magasin remonte son statut, avec retour arrière possible.

**Ce que fait le SD-WAN.** Politique de qualité de service nocturne, limitation de débit par magasin, supervision de l'avancement.

**Leçon.** Les mises à jour massives se planifient comme une opération de sécurité et de capacité, pas comme un simple téléchargement.

### Scénario 10 : la maintenance du froid par un prestataire (Hyper 0142)

**Contexte.** Un prestataire doit se connecter à distance au système de gestion du froid pour un diagnostic.

**Ce qui se passe.**
1. Le prestataire s'authentifie avec une identité nominative et une authentification à plusieurs facteurs.
2. Il obtient un accès limité dans le temps, uniquement à l'application de supervision du froid (principe ZTNA), jamais au réseau entier.
3. Sa session est enregistrée et journalisée.
4. À la fin de l'intervention, l'accès est automatiquement fermé.

**Ce que fait le SD-WAN.** L'accès tiers n'ouvre pas de port à l'extérieur. C'est la connexion sortante vers un service d'accès sécurisé qui met en relation prestataire et application.

**Leçon.** Les accès de tiers sont une porte d'entrée classique des attaques. On les limite en durée, en périmètre et on les trace.

### Scénario 11 : le drive et ses terminaux mobiles (Drive 7005)

**Contexte.** Les préparateurs utilisent des terminaux mobiles pour collecter les commandes dans le magasin.

**Ce qui se passe.**
1. Les terminaux se connectent à un réseau wifi du segment MOBILE_PRO.
2. Ils ne peuvent atteindre que l'application de préparation, via l'overlay.
3. Le web général leur est interdit.
4. Si un terminal est perdu, il est invalidé dans la gestion des appareils et ne peut plus se connecter.

**Ce que fait le SD-WAN.** Segment dédié, liste d'applications autorisées, aucune navigation libre.

**Leçon.** Un terminal professionnel ne sert qu'à ce pour quoi il est prévu.

### Scénario 12 : l'Edge volé lors d'un cambriolage (City 5873)

**Contexte.** Une nuit, un cambriolage : le routeur est emporté.

**Ce qui se passe.**
1. Le centre de supervision détecte la perte de contact du magasin.
2. Une vérification confirme le vol (appel du gérant, alerte de l'alarme).
3. Le numéro de série du routeur est invalidé dans le Manager, la liste d'autorisation est mise à jour.
4. Le certificat du routeur est révoqué côté autorité de certification.
5. Un routeur de remplacement est expédié et s'enrôle automatiquement à son branchement.
6. Les tunnels vers la sécurité cloud associés à ce routeur sont désactivés et leurs secrets renouvelés.

**Ce que fait le SD-WAN.** L'identité de l'équipement est retirée : même avec son matériel, le voleur ne peut plus rejoindre l'overlay ni utiliser les tunnels de sécurité.

**Leçon.** La PKI et le breakout sont liés : un Edge perdu doit perdre toute légitimité, y compris pour les tunnels de sécurité.

### Scénario 13 : l'audit de conformité au paiement (siège)

**Contexte.** Un auditeur vérifie que les flux de paiement sont bien isolés.

**Ce qui est montré.**
1. Un schéma des segments avec la règle "PAIEMENT vers Internet : bloquer".
2. Un test en direct : depuis une caisse, tentative d'accès à un site web, résultat refusé et journalisé.
3. Les journaux de plusieurs mois, centralisés, avec conservation conforme.
4. La preuve que seuls des Edges authentifiés reçoivent la politique.
5. Les procédures de gestion des changements et de révision annuelle.

**Leçon.** La sécurité doit être démontrable : politiques écrites, tests, traces, preuves d'application.

### Scénario 14 : l'entrepôt logistique (Entrepôt Nord)

**Contexte.** 120 scanners sans fil, un système de gestion d'entrepôt, des camions qui arrivent avec des documents numériques.

**Ce qui se passe.**
1. Les scanners n'accèdent qu'au système d'entrepôt via l'overlay.
2. Les transporteurs utilisent un portail dédié, accessible depuis un segment partenaire isolé.
3. Les postes administratifs ont une sortie locale filtrée pour la messagerie et le web.
4. Les scanners ne peuvent pas naviguer sur Internet.

**Ce que fait le SD-WAN.** Segments dédiés (SCANNERS, PARTENAIRES, ADMIN), liens redondants, qualité de service pour les flux temps réel de l'entrepôt.

**Leçon.** Les partenaires externes ont leur segment, jamais un accès au réseau interne.

### Scénario 15 : le magasin qui passe en franchise (Super 2210)

**Contexte.** Le magasin passe sous la gestion d'un exploitant indépendant, avec un contrat de service réseau.

**Ce qui se passe.**
1. Le groupe décide qui reste propriétaire de l'Edge et de la politique de sécurité.
2. Les segments du franchisé sont séparés de ceux gérés par le groupe.
3. Les flux vers les systèmes centraux (paiement, prix, stocks) passent par l'overlay, sous contrôle du groupe.
4. Le wifi invité et la navigation du personnel suivent une politique minimale commune définie au contrat.
5. Les journaux de sécurité sont partagés selon les termes du contrat.

**Leçon.** Un changement d'organisation est un événement de sécurité : il faut redéfinir qui contrôle quoi, avant et pas après.

### Scénario 16 : ouverture d'un nouveau magasin avec breakout sécurisé (City 8002)

**Chronologie.**
1. J-30 : commande du routeur et de la carte 5G, numéro de série ajouté à la liste d'autorisation.
2. J-15 : modèle de configuration "Proximité urbaine" associé, avec segments et politique de sortie.
3. J-10 : tunnels vers la sécurité cloud préparés pour ce site (identifiants, point de présence principal et secondaire).
4. J-7 : le routeur est expédié au magasin.
5. Jour J : branchement, enrôlement automatique, application de la politique.
6. Jour J, vérifications guidées : test paiement (overlay), test messagerie (sortie locale), test wifi invité (isolé), test coupure fibre (bascule 5G).
7. J+7 : revue des journaux et des volumes, ajustement des limites.

**Leçon.** L'ouverture est un projet de bout en bout où réseau, sécurité et exploitation travaillent ensemble.

### Scénario 17 : l'employé et sa banque en ligne (Hyper 0142)

**Contexte.** Un employé consulte son compte bancaire personnel depuis le poste de pause.

**Ce qui se passe.**
1. La politique d'inspection du trafic chiffré exclut la catégorie "banque et finance personnelle".
2. Le trafic est autorisé et chiffré de bout en bout, sans déchiffrement par l'entreprise.
3. Il est tout de même journalisé au niveau minimal (destination, volume), sans contenu.

**Leçon.** Inspecter tout n'est ni possible ni souhaitable. Respecter la vie privée fait partie de la conformité.

### Scénario 18 : l'application mobile des clients sur le wifi

**Contexte.** Des clients utilisent l'application du groupe pour consulter leur carte de fidélité et scanner leurs articles.

**Ce qui se passe.**
1. L'application appelle les services du groupe via Internet public, comme n'importe quel autre site.
2. Le wifi invité permet cette sortie vers Internet, mais sans accès au réseau interne du magasin.
3. Les services du groupe sont protégés de leur côté (authentification, limitation de débit).

**Leçon.** Le wifi invité est un simple accès Internet. Les services destinés aux clients se protègent côté serveur, pas en comptant sur le réseau du magasin.

### Scénario 19 : tentative d'intrusion depuis Internet sur l'adresse publique d'un magasin

**Contexte.** Un robot scanne des plages d'adresses publiques et trouve celle du magasin.

**Ce qui se passe.**
1. Le robot teste de nombreux ports.
2. Aucun service n'écoute sur l'adresse publique, et la règle "Internet vers tout segment : bloquer" s'applique.
3. Les tentatives sont comptées et journalisées. Au delà d'un seuil, l'adresse source est bloquée plus longtemps.

**Ce que fait le SD-WAN.** Le site sort sur Internet mais ne publie aucun service : le robot ne trouve rien à attaquer.

**Leçon.** Sortir sans exposer, c'est ne rien rendre joignable depuis l'extérieur.

---

## E. Tableau récapitulatif des scénarios

| N° | Scénario | Mécanisme clé | Principe illustré |
|---|---|---|---|
| 1 | Matinée normale | Politique par segment et application | Chemins différents selon le flux |
| 2 | Lien frauduleux | Sécurité DNS et web | Première barrière |
| 3 | Client curieux sur wifi | Isolation du segment invité | On n'attaque pas ce qu'on ne voit pas |
| 4 | Poste infecté | Confinement entre segments | Limiter les dégâts |
| 5 | Fibre coupée | Bascule sur lien de secours | Résilience |
| 6 | Tunnel cloud tombé | Fail-open ou fail-close par segment | Décider à l'avance |
| 7 | Black Friday | Qualité de service et plafonds | Priorité aux flux métier |
| 8 | Objet connecté compromis | Liste blanche de destinations | Moindre privilège |
| 9 | Mise à jour des caisses | Déploiement par vagues et étalement | Capacité et sécurité |
| 10 | Prestataire distant | Accès ZTNA limité dans le temps | Contrôle des tiers |
| 11 | Drive et terminaux | Segment mobile restreint | Un outil, un usage |
| 12 | Edge volé | Invalidation et révocation | Lien avec la PKI |
| 13 | Audit de conformité | Preuves et journaux | Sécurité démontrable |
| 14 | Entrepôt | Segments dédiés dont partenaires | Isoler les externes |
| 15 | Passage en franchise | Redéfinition des responsabilités | Changement organisationnel |
| 16 | Ouverture de magasin | Zero touch et vérifications guidées | Projet de bout en bout |
| 17 | Banque en ligne d'un employé | Exclusions d'inspection | Vie privée |
| 18 | Application mobile clients | Wifi invité = Internet simple | Protéger côté serveur |
| 19 | Intrusion depuis Internet | Aucun service exposé | Sortir sans exposer |

---

## F. Exemple de journal d'événements (fictif)

Extrait pour une journée sur un magasin, tel qu'une équipe de supervision pourrait le lire :

| Heure | Segment | Événement | Décision | Action |
|---|---|---|---|---|
| 07:42 | OBJETS | Mise à jour étiquettes | Autorisé (fournisseur listé) | Aucune |
| 09:12 | GESTION | Domaine d'hameçonnage demandé | Bloqué (DNS) | Alerte, sensibilisation du poste |
| 10:03 | INVITE | Balayage de ports vers segments internes | Bloqué | Limitation de la session |
| 11:20 | PAIEMENT | Tentative d'accès à un site web depuis une caisse | Bloqué | Alerte élevée, vérification du poste |
| 13:47 | Liens | Perte de la fibre | Bascule 5G | Ticket opérateur |
| 14:05 | INVITE | Plafond de débit atteint | Limitation appliquée | Aucune |
| 16:30 | Tunnel sécurité | Tunnel principal indisponible | Bascule tunnel secondaire | Surveillance |
| 18:15 | Liens | Retour de la fibre, stabilisation 10 min | Retour sur fibre | Clôture du ticket |
| 22:30 | Internet | Scan entrant sur adresse publique | Bloqué | Source bloquée temporairement |

Observation : la ligne de 11:20 est la plus grave. Une caisse qui essaie d'aller sur Internet est un signal fort de compromission ou de mauvaise configuration.

---

## G. Exercices sur les scénarios

### Exercice 1 : choisir le chemin
Pour chaque flux du magasin City 5873, indiquer le chemin (overlay, sortie locale directe, tunnel sécurité cloud, bloqué) :

1. Une caisse envoie une transaction de paiement.
2. Un employé ouvre sa messagerie en ligne.
3. Un employé consulte un site de comparaison de prix.
4. Un client regarde une vidéo sur le wifi invité.
5. Une étiquette électronique télécharge un nouveau prix.
6. Une caméra envoie son flux vers le stockage central.

**Réponses.** 1) overlay. 2) sortie locale directe optimisée. 3) tunnel vers la sécurité cloud. 4) sortie locale isolée, filtrée et plafonnée. 5) sortie locale vers le fournisseur listé uniquement. 6) overlay.

### Exercice 2 : diagnostic
Le magasin Super 2210 se plaint de lenteur du web depuis 15 h. Les paiements fonctionnent normalement. Les journaux montrent le tunnel principal vers la sécurité cloud indisponible depuis 14 h 55 et le tunnel secondaire saturé.

Questions :
1. Quelle partie du réseau est touchée, quelle partie est épargnée ?
2. Quelle action immédiate ?
3. Quelle amélioration à moyen terme ?

**Réponses.** 1) Seule la navigation web est touchée, le paiement passe par l'overlay. 2) Ouvrir un incident chez le fournisseur et vérifier si un autre point de présence peut être utilisé. 3) Dimensionner le tunnel secondaire pour absorber la charge d'un tunnel principal défaillant, et tester ce scénario régulièrement.

### Exercice 3 : analyse d'incident
Une caisse libre-service du Hyper 0142 apparaît dans les journaux avec des tentatives de connexion vers une adresse Internet inconnue. Que faire ?

**Éléments de réponse.**
1. Vérifier que la règle "PAIEMENT vers Internet : bloquer" a bien empêché la connexion.
2. Isoler la caisse concernée du réseau.
3. Analyser l'équipement (logiciel, mise à jour, intrusion physique éventuelle).
4. Rechercher d'autres caisses au comportement similaire.
5. Documenter l'incident, corriger la cause, informer les responsables conformité si nécessaire.

### Exercice 4 : conception
Vous devez ouvrir un mini-drive dans un village, avec une seule fibre et une 5G de secours, deux caisses, un poste de gestion, quatre terminaux mobiles et un wifi invité. Proposez :
1. Les segments.
2. La politique de sortie Internet de chacun.
3. Le comportement en cas de panne de fibre.

**Proposition.** Segments PAIEMENT, GESTION, MOBILE_PRO, INVITE. PAIEMENT : overlay uniquement. GESTION : messagerie en sortie locale, web via sécurité cloud. MOBILE_PRO : application de préparation via overlay uniquement. INVITE : sortie locale isolée, limitée. En cas de panne de fibre : bascule 5G, plafond du wifi invité réduit, priorité au paiement et à la préparation.

---

## H. Questions de révision sur les scénarios

1. Pourquoi une caisse qui tente d'aller sur Internet est elle un signal d'alerte fort ?
2. Que signifie "fail-close" pour la navigation du segment GESTION ?
3. Comment un prestataire peut-il intervenir sans qu'on ouvre un port sur Internet ?
4. Pourquoi plafonner le wifi invité un jour de forte affluence ?
5. Que doit-on faire des tunnels de sécurité quand un Edge est volé ?
6. Pourquoi exclure certaines catégories de l'inspection du trafic chiffré ?
7. Quel est l'intérêt d'avoir des segments différents pour les scanners d'entrepôt et pour les transporteurs ?

Réponses courtes : 1) Elle ne devrait jamais le faire, c'est le signe d'une compromission ou d'une erreur. 2) Le web général est bloqué tant que la sécurité cloud est indisponible, avec un message clair. 3) Par une connexion sortante vers un service d'accès qui relie le prestataire à l'application autorisée. 4) Pour préserver la bande passante des flux métier comme le paiement. 5) Les désactiver et renouveler leurs identifiants, en plus de l'invalidation et de la révocation du certificat. 6) Pour respecter la vie privée et les obligations légales. 7) Les partenaires externes ne doivent jamais avoir accès au réseau interne.

---

## I. À retenir de cette partie

- Un même magasin utilise plusieurs chemins Internet différents selon le segment et l'application.
- Les caisses et caméras n'ont jamais d'accès Internet direct.
- Le wifi invité est un service isolé, plafonné et filtré.
- Les objets connectés et les terminaux mobiles ont des droits strictement limités.
- Le comportement en cas de panne (lien, tunnel, sécurité cloud) se décide à l'avance, par segment.
- Les accès de tiers sont limités, temporaires et tracés.
- Un Edge perdu perd toute légitimité : invalidation, révocation et renouvellement des secrets de tunnel.
- La sécurité se prouve : politiques écrites, tests, journaux.
- Tous les scénarios de cette partie sont fictifs et pédagogiques, ils ne décrivent pas le réseau réel d'une enseigne.
