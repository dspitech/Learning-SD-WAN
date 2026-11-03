# Cours ZTP : Zero Touch Provisioning pour SD-WAN

Ce cours est entièrement consacré au Zero Touch Provisioning (ZTP) appliqué au SD-WAN. Il suppose que vous maîtrisez déjà l'architecture SD-WAN, les plans de contrôle et de données, l'overlay, les politiques et la sécurité. Le fil rouge est le déploiement d'un parc de magasins pour un distributeur de type Carrefour.

> Note : le scénario Carrefour est un cas d'école. Chiffres, noms de sites, adresses IP et processus sont fictifs et ne décrivent pas l'organisation réelle de l'enseigne.

---

## Table des matières

1. Définition et enjeux du ZTP
2. Les modèles de ZTP
3. Prérequis côté siège (jour 0)
4. Séquence détaillée de ZTP
5. Identité, certificats et confiance
6. Découverte : DHCP, DNS, redirection cloud
7. Templates, variables et profils de site
8. ZTP sans lien fixe : USB, bootstrap, 4G et 5G
9. Cas pratique : ouverture d'un magasin de proximité
10. Cas pratique : rénovation d'un hypermarché en haute disponibilité
11. Cas pratique : remplacement matériel (RMA) par le gérant
12. Industrialisation : staging, logistique, automatisation
13. Sécurité du ZTP
14. Dépannage du ZTP
15. Indicateurs de réussite
16. Bonnes pratiques et pièges
17. Exercices et corrigés
18. Quiz de révision
19. Glossaire

---

## 1. Définition et enjeux du ZTP

### 1.1 Définition

Le **Zero Touch Provisioning** est la capacité pour un équipement neuf (ou remis à zéro) de :

1. obtenir une connectivité IP sans configuration préalable ;
2. découvrir de façon automatique son orchestrateur ou son service de redirection ;
3. s'authentifier de manière sûre ;
4. récupérer sa configuration, son logiciel et ses politiques ;
5. rejoindre le réseau en production,

le tout **sans qu'un technicien réseau intervienne sur place**. Le personnel du site se limite à des gestes physiques simples : brancher l'alimentation et les câbles.

### 1.2 Pourquoi c'est critique en distribution

| Contrainte retail | Apport du ZTP |
|---|---|
| Milliers de sites répartis sur tout le territoire | Pas de déplacement d'ingénieur pour chaque ouverture |
| Gérants ou franchisés sans compétence réseau | Procédure en trois gestes |
| Ouvertures, rénovations et fermetures saisonnières | Rythme de déploiement élevé et prévisible |
| Disponibilité exigée des caisses et du paiement | Remplacement matériel rapide, sans attente d'un technicien |
| Parc homogène et conforme | Même template appliqué partout, sans écart manuel |
| Coût du déploiement | Suppression des interventions on-site |

### 1.3 Ce que le ZTP n'est pas

- Ce n'est pas un simple DHCP avec un fichier de configuration : il inclut l'**authentification mutuelle** et l'orchestration du plan de contrôle.
- Ce n'est pas la livraison du lien WAN : le ZTP a besoin d'un lien fonctionnel pour démarrer.
- Ce n'est pas la fin de la préparation : la rigueur se déplace du terrain vers le siège (inventaire, templates, variables).

### 1.4 Vocabulaire des "jours"

| Terme | Signification |
|---|---|
| Day 0 | Mise en service initiale : connexion au plan de contrôle et configuration de base |
| Day 1 | Configuration complète du site : VPN de service, interfaces, politiques |
| Day 2 | Exploitation : supervision, changements, mises à jour |

Le ZTP couvre le Day 0 et le Day 1, et prépare le Day 2.

---

## 2. Les modèles de ZTP

### 2.1 Redirection cloud (modèle dominant)

L'équipement contacte un service cloud de l'éditeur. Ce service connaît son numéro de série, l'associe au compte du client et renvoie l'adresse de l'orchestrateur du client.

```
Edge  --->  Portail de redirection éditeur  --->  Orchestrateur du client
```

Avantage : aucune infrastructure à exposer pour la découverte initiale, aucune option DHCP spécifique à configurer sur le site.

### 2.2 Découverte locale (DHCP, DNS)

Le réseau local ou le FAI fournit l'adresse du serveur de découverte via une option DHCP ou une entrée DNS spéciale. Utile en entreprise avec un serveur de provisioning interne (exemple : serveur Plug and Play interne chez Cisco).

### 2.3 Bootstrap par fichier (USB, carte, console)

Un fichier de configuration minimale (identité, orchestrateur, organisation, certificats racine) est fourni localement. L'équipement s'en sert pour se connecter à l'orchestrateur.

Cas d'usage : absence de DHCP, réseau très filtré, pré-staging chez un intégrateur.

### 2.4 Activation par SIM intégrée (ZTP cellulaire)

Sur les modèles avec modem 4G ou 5G, l'équipement démarre via le réseau mobile lorsque le lien fixe n'est pas encore livré. Très utile pour ouvrir un magasin avant l'arrivée de la fibre.

### 2.5 Comparatif

| Modèle | Intervention locale | Prérequis | Cas d'usage retail |
|---|---|---|---|
| Redirection cloud | Brancher l'équipement | DHCP et Internet | Cas standard proximité et supermarchés |
| Découverte DHCP/DNS | Brancher l'équipement | Serveur interne et options configurées | Sites reliés à un réseau d'entreprise existant |
| Bootstrap USB | Insérer une clé | Fichier préparé | Réseau filtré, absence de DHCP |
| SIM intégrée | Brancher l'équipement | Couverture mobile, forfait actif | Ouverture avant fibre, secours |

---

## 3. Prérequis côté siège (jour 0)

### 3.1 Inventaire

Pour chaque équipement :

- numéro de série (et référence matérielle) ;
- modèle et licence associée ;
- site cible (identifiant, adresse, type de format) ;
- date de livraison prévue.

Cet inventaire doit être **à jour et fiable** : une erreur de numéro de série bloque le ZTP.

### 3.2 Compte et portail de l'éditeur

- Création du compte client (par exemple Smart Account et Virtual Account chez Cisco).
- Déclaration des équipements dans le portail de redirection (Plug and Play Connect chez Cisco).
- Association de chaque équipement à l'orchestrateur du client.

### 3.3 Infrastructure du plan de contrôle

- Orchestrateur, contrôleurs et manager accessibles depuis Internet sur les ports requis (ou via des points d'entrée adaptés).
- Haute disponibilité des composants : si l'orchestrateur est indisponible, aucun nouvel équipement ne peut rejoindre le réseau.
- Certificats des composants valides et chaîne de confiance prête.

### 3.4 Modèles de configuration

- Un template par type de site (hyper, super, proximité, entrepôt).
- Un jeu de variables par site.
- Une politique centralisée associée aux listes de sites.

### 3.5 Contrat opérateur

- Information du FAI sur le mode de livraison : DHCP attendu, pas de VLAN particulier, port Ethernet disponible.
- Paramètres éventuels : VLAN, PPPoE, adressage statique. Le ZTP est plus simple avec DHCP.

### 3.6 Processus et responsabilités

| Rôle | Responsabilité |
|---|---|
| Équipe réseau | Templates, variables, politiques, supervision |
| Achats et logistique | Commande, déclaration des numéros de série, expédition |
| Intégrateur (si présent) | Staging, tests, expédition |
| Gérant ou site | Branchements physiques |
| Support N1 | Assistance téléphonique au gérant |

---

## 4. Séquence détaillée de ZTP

### 4.1 Vue globale (modèle Cisco Catalyst SD-WAN)

```
Edge             DHCP/DNS     Portail PnP      Validator     Manager/Controller
 |                  |             |               |                 |
 | 1 DHCP --------->|             |               |                 |
 |<---- IP, GW, DNS-|             |               |                 |
 | 2 Résolution + requête avec n° de série ->     |                 |
 |<---- Adresse Validator + organisation ---------|                 |
 | 3 DTLS/TLS + certificats ------------------>   |                 |
 |<---- Adresses Controller et Manager -----------|                 |
 | 4 Connexion Controller/Manager ------------------------------>   |
 |<---- Config template + version logicielle -------------------     |
 | 5 Tunnels IPsec, BFD, OMP -------------------------------------> |
```

### 4.2 Étape 1 : connectivité IP

- L'équipement détecte une interface WAN utilisable (souvent un port dédié ou le premier port disponible).
- Il envoie une requête DHCP.
- Il reçoit adresse, passerelle, DNS.

Points d'attention : port non branché au bon endroit, box FAI sans DHCP actif, VLAN imposé par l'opérateur.

### 4.3 Étape 2 : découverte du point d'entrée

L'équipement tente, dans un ordre défini par l'éditeur :

1. un serveur de découverte local (option DHCP ou entrée DNS) ;
2. le service cloud de redirection (un nom DNS public connu de l'éditeur).

Il présente son **numéro de série** et sa **référence matérielle**. Le service répond avec l'adresse de l'orchestrateur et le nom de l'organisation.

### 4.4 Étape 3 : authentification mutuelle

- L'équipement ouvre une session DTLS ou TLS vers l'orchestrateur.
- Chacun présente son certificat.
- L'orchestrateur contrôle que le numéro de série fait partie de la liste autorisée.
- L'équipement vérifie le certificat de l'orchestrateur grâce à la chaîne racine reçue (bootstrap ou pré-installée).
- Si tout est valide, l'orchestrateur transmet l'adresse des contrôleurs et du manager.

### 4.5 Étape 4 : configuration et logiciel

- L'équipement se connecte au manager.
- Le manager repère le template associé au numéro de série et injecte les variables du site.
- Selon la politique, une mise à jour logicielle est imposée avant mise en production.
- L'équipement applique la configuration, redémarre si nécessaire.

### 4.6 Étape 5 : construction de l'overlay

- Connexion de contrôle vers les contrôleurs (échange de routes, de TLOC, de clés).
- Tunnels IPsec vers les hubs et éventuellement les autres sites selon la topologie.
- Sessions BFD.
- Application des politiques centralisées.

### 4.7 Durées indicatives

| Phase | Durée typique |
|---|---|
| DHCP et découverte | Quelques secondes à 1 minute |
| Authentification | Moins de 1 minute |
| Téléchargement et installation logicielle | 5 à 20 minutes selon la taille et la bande passante |
| Application de la configuration | 1 à 5 minutes |
| Établissement de l'overlay | 1 à 3 minutes |

Une mise en service complète se compte donc en minutes, avec un pic lié à la mise à jour logicielle. Ces valeurs sont indicatives.

---

## 5. Identité, certificats et confiance

### 5.1 Pourquoi l'identité est centrale

Sans identité forte, n'importe quel équipement pourrait se faire passer pour un magasin et rejoindre l'overlay. Le ZTP repose donc sur **trois piliers** :

1. **Identité matérielle** : certificat installé en usine (par exemple un certificat de type SUDI chez Cisco) lié au numéro de série.
2. **Liste blanche** : seuls les numéros de série déclarés par le client sont acceptés.
3. **Chaîne de confiance** : l'équipement vérifie les certificats des composants du plan de contrôle.

### 5.2 Certificats du plan de contrôle

- Les composants (orchestrateur, contrôleurs, manager) possèdent des certificats signés par une autorité de certification (ACs éditeur ou autorité d'entreprise).
- Les équipements d'extrémité obtiennent ensuite, après authentification initiale, un certificat opérationnel selon le modèle choisi (autorité de l'éditeur, autorité d'entreprise, ou méthode manuelle).

### 5.3 Cycle de vie des certificats

| Événement | Action |
|---|---|
| Expiration proche | Alerte et renouvellement planifié |
| Compromission présumée | Révocation, blocage du numéro de série |
| Remplacement matériel | Désactivation de l'ancien équipement, déclaration du nouveau |
| Changement d'autorité | Plan de migration par vagues avec fenêtre de coexistence |

### 5.4 L'horloge, un point sensible

La validation d'un certificat dépend de l'heure. Un équipement avec une horloge très fausse rejette les certificats comme "non encore valides" ou "expirés". Parades : NTP joignable, mécanisme de synchronisation précoce, horloge matérielle fiable.

---

## 6. Découverte : DHCP, DNS, redirection cloud

### 6.1 DHCP

Le DHCP fournit : adresse, masque, passerelle, serveurs DNS, éventuellement des options de découverte.

Contrainte importante : sur la plupart des sites de proximité, le DHCP est fourni par la **box du FAI**. Le ZTP doit fonctionner avec cette box "telle quelle", sans réglage spécifique.

### 6.2 DNS

Le DNS permet :

- de résoudre le service de redirection de l'éditeur ;
- de résoudre l'orchestrateur du client (nom de domaine qui doit être public ou accessible).

Si le DNS du FAI filtre ou ne répond pas correctement, le ZTP échoue. Certains équipements permettent de repli vers des DNS publics connus.

### 6.3 Méthodes de découverte Cisco PnP (à titre d'exemple)

| Méthode | Mécanisme |
|---|---|
| DHCP | Option dédiée donnant l'adresse du serveur PnP |
| DNS | Résolution d'un nom conventionnel basé sur le domaine reçu |
| Cloud | Redirection par le service de l'éditeur |

Les détails (noms d'options, noms DNS, ports) varient selon la version : référez-vous à la documentation de votre plateforme.

### 6.4 Flux réseau à ouvrir

Selon la solution, les flux typiques à autoriser en sortie depuis le site :

| Flux | Usage |
|---|---|
| UDP ou TCP vers le service de redirection | Découverte |
| DTLS (UDP) ou TLS (TCP) vers l'orchestrateur | Authentification |
| DTLS ou TLS vers contrôleurs et manager | Contrôle et gestion |
| UDP pour IPsec et NAT-T | Tunnels de données |
| NTP | Synchronisation d'horloge |
| HTTPS vers un serveur de mise à jour logicielle | Images |

Dans un contexte retail, la box FAI laisse en général sortir ces flux. Les difficultés surviennent surtout lorsque des pare-feux intermédiaires ou des services de filtrage sont présents.

### 6.5 NAT et double NAT

Quand l'équipement est derrière la box FAI (NAT), il est souvent sur une adresse privée. Les solutions SD-WAN gèrent bien le NAT, mais :

- le **double NAT** (box FAI plus routeur opérateur) complique la traversée pour certains flux entrants entre sites ;
- le NAT de type symétrique peut empêcher l'établissement direct de tunnels entre deux sites derrière NAT. Dans le modèle hub and spoke, le hub avec adresse publique fixe résout le problème.

---

## 7. Templates, variables et profils de site

### 7.1 Principe

Un **template** est un modèle de configuration avec des **variables** (hostname, adressage, VLAN, paramètres du lien). Au moment du ZTP, le manager remplit les variables avec les valeurs propres au site et pousse la configuration.

### 7.2 Profils de site (retail)

| Profil | Équipement | Liens | VPN de service |
|---|---|---|---|
| Proximité | Edge compact | Fibre plus 4G/5G | Monétique, caisses, voix, invités |
| Supermarché | Edge moyen | Fibre plus MPLS ou 2e fibre | Plus vidéosurveillance et IoT |
| Hypermarché | 2 edges en HA | MPLS ou fibre dédiée, fibre Internet, 4G/5G | Tous, plus drive et logistique |
| Entrepôt | 2 edges | Liens à fort débit | Logistique, scanners, voix |

### 7.3 Exemple de variables

| Variable | Exemple | Règle |
|---|---|---|
| `site_id` | 3412 | Identifiant unique du magasin |
| `hostname` | PRX-3412-R1 | Format `TYPE-ID-Rn` |
| `system_ip` | 10.255.34.12 | Dérivé de `site_id` |
| `lan_caisses` | 10.20.34.0/24 | `10.20.<A>.0/24` avec A = site_id divisé par 100 |
| `lan_monetique` | 10.10.34.0/24 | Idem sur 10.10 |
| `wan1_type` | dhcp | dhcp, static ou pppoe |
| `wan2_type` | lte | lte, dhcp |
| `region` | IDF | Pour regrouper les politiques |
| `timezone` | Europe/Paris | Pour logs et fenêtres de mise à jour |

### 7.4 Bonnes pratiques de conception des templates

1. **Modulaires** : séparer identité, transport, VPN de service, sécurité, routage.
2. **Peu de variables** : plus il y en a, plus les erreurs se multiplient. Dériver au maximum de l'identifiant de site.
3. **Valeurs par défaut sûres** : pas d'ouverture de service par accident.
4. **Validation** : contrôle de format (adresses IP, masques) avant ZTP.
5. **Versionnement** : chaque changement de template est tracé et testé.

### 7.4.1 Exemple de dérivation d'adressage

Pour le site `N` :

- A = partie entière de N divisé par 100 ;
- B = N modulo 100 ;
- `system_ip = 10.255.A.B` ;
- `lan_caisses = 10.20.A.0/24`.

Pour N = 3412 : A = 34, B = 12, donc `10.255.34.12` et `10.20.34.0/24`.

Attention : cette règle est limitée (A ne dépasse pas 99). Prévoir un plan d'adressage documenté et vérifié par script avant déploiement.

### 7.5 Mise à jour logicielle dans le ZTP

Deux approches :

- **Version cible imposée** : l'équipement installe la version standard avant de passer en production. Plus long, mais garantit l'homogénéité.
- **Version d'usine tolérée** : l'équipement démarre avec sa version, la mise à jour est planifiée ensuite. Plus rapide, mais crée de l'hétérogénéité.

En retail, la première approche est préférable pour les formats critiques. Pour limiter la durée, prévoir un **staging** (voir section 12) où le logiciel est déjà à la bonne version.

---

## 8. ZTP sans lien fixe : USB, bootstrap, 4G et 5G

### 8.1 Bootstrap par USB

Un fichier de configuration minimal est placé sur une clé USB insérée dans l'équipement avant ou pendant le démarrage. Il contient typiquement :

- identifiant d'organisation ;
- adresse de l'orchestrateur ;
- numéro de série ou identifiant ;
- certificat racine ;
- paramètres du lien WAN (statique, VLAN, PPPoE).

Le nom exact du fichier et son format dépendent de l'éditeur et de la version (exemple Cisco : un fichier de bootstrap généré par le manager). Référez-vous à la documentation de la plateforme.

Cas d'usage :

- lien WAN nécessitant une configuration statique ;
- réseau d'accès sans DHCP ;
- pré-staging en dépôt.

### 8.2 ZTP via 4G ou 5G

Principe : l'équipement dispose d'un modem mobile avec SIM (ou eSIM). À l'allumage, s'il n'y a pas de lien fixe fonctionnel, il établit une connexion mobile et lance la séquence ZTP.

Points d'attention :

| Sujet | Recommandation |
|---|---|
| Forfait de données | Volume suffisant pour la mise à jour logicielle initiale |
| Couverture | Test en amont, antenne déportée si nécessaire (arrière-boutique, sous-sol) |
| APN | Préconfigurer l'APN du forfait professionnel |
| Taille de la mise à jour | Éviter une mise à jour lourde sur cellulaire ; staging préalable |
| Basculement ensuite | Une fois la fibre livrée, bascule automatique du trafic et limitation du cellulaire à un rôle de secours |

### 8.3 Exemple d'usage : magasin ouvert avant la fibre

1. Le magasin ouvre le lundi, la fibre est annoncée pour trois semaines plus tard.
2. L'équipement est expédié avec une SIM professionnelle.
3. Le ZTP s'effectue par 5G.
4. Une politique "mode dégradé" réserve le lien cellulaire aux caisses, au paiement et à la voix.
5. À l'arrivée de la fibre, le gérant branche le second port WAN, la politique normale reprend automatiquement.

---

## 9. Cas pratique : ouverture d'un magasin de proximité

### 9.1 Contexte

Le groupe fictif "Groupe Distri" ouvre le magasin "Proximité Villejuif" (site 3412). Format : 3 caisses, un terminal de paiement par caisse, deux téléphones IP, une caméra à l'entrée, un Wi-Fi invités. Liens : fibre FAI plus 5G.

### 9.2 Chronologie

| Délai | Action | Responsable |
|---|---|---|
| J-30 | Création du site dans l'outil d'inventaire, numéro de site attribué | Équipe réseau |
| J-28 | Commande du lien fibre | Achats |
| J-21 | Commande de l'edge, déclaration du numéro de série dans le portail éditeur | Achats |
| J-14 | Création du site dans le manager, affectation du template `TPL-PROXIMITE-V3`, saisie des variables | Équipe réseau |
| J-10 | Staging chez l'intégrateur : ZTP de test, mise à jour logicielle, vérification, remise à zéro de l'état "site" | Intégrateur |
| J-5 | Expédition du colis avec guide visuel | Logisticien |
| J-1 | Contrôle que la fibre est livrée et active (test FAI) | Équipe réseau |
| J | Branchement par le gérant, ZTP automatique | Gérant |
| J+0 (15 minutes) | Site "vert" dans le manager, tests de caisse | Support N1 |

### 9.3 Guide en trois gestes remis au gérant

1. **Brancher** le câble d'alimentation.
2. **Relier** la prise fibre (ONT ou box) au port repéré "WAN 1" (bleu).
3. **Relier** le switch du magasin au port repéré "LAN 1" (vert).

Puis attendre que la LED passe au vert fixe. En cas de problème, appeler le numéro du support indiqué sur l'étiquette.

### 9.4 Ce qui se passe techniquement

1. L'équipement obtient une IP par DHCP auprès de la box fibre.
2. Il contacte le service de redirection, présente son numéro de série.
3. Il est dirigé vers l'orchestrateur du groupe, s'authentifie par certificat.
4. Il récupère les adresses des contrôleurs et du manager.
5. Le manager applique `TPL-PROXIMITE-V3` avec les variables du site 3412.
6. Les tunnels IPsec vers les hubs Paris et Lyon s'établissent.
7. OMP annonce les préfixes du magasin, reçoit les routes des autres sites.
8. Les politiques applicatives, la QoS et la segmentation s'appliquent.
9. Le monitoring s'active, le site apparaît dans les tableaux de bord.

### 9.5 Validation après ZTP

| Contrôle | Résultat attendu | Outil |
|---|---|---|
| Statut du site | Opérationnel | Manager |
| Connexions de contrôle | Établies vers au moins deux contrôleurs | `show sdwan control connections` |
| Sessions BFD | Actives vers les deux hubs | `show sdwan bfd sessions` |
| Version logicielle | Conforme au standard | Manager |
| Routes monétique | Reçues et annoncées dans le VPN 10 | `show sdwan omp routes` |
| Test de transaction | Succès | Terminal de paiement de test |
| Test de bascule | Fibre coupée, bascule sur 5G, flux critiques maintenus | Test planifié avec le support |

### 9.6 Cas d'échec et résolution

| Constat | Diagnostic | Résolution |
|---|---|---|
| LED "recherche IP" indéfiniment | Câble branché sur un port LAN, ou fibre non active | Support N1 guide le gérant, vérification FAI |
| LED "contact orchestrateur" bloquée | DNS filtré par la box ou pare-feu | Changement du DNS de la box, ou ZTP par 5G |
| Site "orphelin" dans le manager | Numéro de série non déclaré | Déclaration dans le portail, relance du ZTP |
| Rejet pour certificat | Horloge fausse ou certificat expiré | Vérifier l'accès NTP, régénérer le certificat |
| Configuration partielle | Variable manquante dans le template | Compléter les variables, re-pousser la configuration |

---

## 10. Cas pratique : rénovation d'un hypermarché en haute disponibilité

### 10.1 Contexte

Hypermarché "Hyper Lyon Part-Dieu" (site fictif 1208). 60 caisses, vidéosurveillance dense, drive, voix, étiquettes électroniques. Liens : MPLS conservé, fibre Internet dédiée, 5G de secours. Deux edges en haute disponibilité.

### 10.2 Particularités du ZTP en HA

- Les deux edges sont déclarés et reçoivent des variables propres (`hostname` différent, `system_ip` différent), tout en partageant le même `site_id`.
- Ils doivent partager des liens WAN et un LAN commun (HSRP, VRRP selon design).
- Le ZTP se fait **un edge à la fois** pour éviter des conflits de rôle.
- Les politiques doivent préciser l'edge principal et l'edge de secours.

### 10.3 Séquence recommandée

1. Pré-staging en dépôt : ZTP complet de chaque edge, tests de bascule entre eux.
2. Livraison sur site, installation par un technicien (le gérant ne suffit pas ici, mais le ZTP supprime la phase de configuration).
3. Connexion du premier edge, ZTP, validation.
4. Connexion du second edge, ZTP, validation de la HA.
5. Bascule progressive des flux : d'abord Wi-Fi invités et IoT, puis voix, puis caisses, puis paiement.
6. Fenêtre de bascule planifiée hors horaires d'ouverture.
7. Maintien du MPLS en parallèle quelques semaines, puis décommissionnement selon les indicateurs.

### 10.4 Retour arrière

- Conservation de la configuration de l'ancien routeur.
- Bascule inverse possible en moins d'une heure grâce à la double connectivité.
- Critères de déclenchement du retour arrière définis à l'avance (par exemple perte supérieure à un seuil sur le paiement pendant plus de 5 minutes).

---

## 11. Cas pratique : remplacement matériel (RMA) par le gérant

### 11.1 Scénario

L'edge de "Proximité Villejuif" tombe en panne matérielle un vendredi soir. Le magasin n'a plus de caisses connectées.

### 11.2 Processus cible

1. Le support N1 identifie la panne et déclenche un remplacement.
2. Un équipement de remplacement est expédié en urgence (dépôt régional ou chronopost).
3. Côté siège :
   - l'ancien numéro de série est marqué "retiré" dans le manager ;
   - le nouveau numéro de série est déclaré dans le portail éditeur ;
   - le nouvel équipement est rattaché au **même site logique** avec les mêmes variables.
4. Le gérant reçoit le nouvel équipement avec le même guide en trois gestes.
5. Le ZTP reconstruit la configuration automatiquement, retrouve la même identité réseau.

### 11.3 Points importants

- **Les variables ne changent pas** : l'adressage et la configuration du site sont préservés.
- **L'ancien certificat est révoqué** pour éviter toute réutilisation.
- **Le délai cible** se mesure en heures, non en jours.
- Un **stock tampon** d'équipements pré-stagés (déjà à la bonne version logicielle) accélère le processus.

### 11.4 Mode dégradé pendant l'attente

Prévoir un mode de repli (terminaux de paiement autonomes par 4G, caisses en mode déconnecté) pour couvrir le délai de remplacement. Le SD-WAN n'élimine pas le besoin de plan de continuité.

---

## 12. Industrialisation : staging, logistique, automatisation

### 12.1 Staging

Le **staging** consiste à préparer l'équipement avant livraison, chez l'intégrateur ou en dépôt :

| Activité | Bénéfice |
|---|---|
| Mise à jour vers la version standard | Réduit la durée du ZTP sur site |
| Test de ZTP de bout en bout | Détecte les défauts de lot ou de variables |
| Pose d'étiquettes de repérage des ports | Évite les erreurs de branchement |
| Emballage avec guide visuel et accessoires | Autonomie du gérant |

Après test, l'équipement est remis à l'état "neuf" pour que le ZTP final s'exécute sur site.

### 12.2 Logistique

- Livraison directe de l'intégrateur vers le magasin.
- Suivi des colis relié à l'outil d'inventaire.
- Confirmation de réception par le gérant (scan ou formulaire).
- Gestion des retours (équipements remplacés).

### 12.3 Automatisation par API

Le manager expose des API REST. Une chaîne typique :

```
Outil d'inventaire (source de vérité)
      |
      v
Script d'orchestration (Python, Ansible, Terraform)
      |
      +--> Création du site dans le manager
      +--> Affectation du template et des variables
      +--> Déclaration du numéro de série dans le portail éditeur
      +--> Création du ticket de livraison
      |
      v
ZTP sur site -> validation automatique -> notification
```

### 12.4 Exemple de script (illustratif)

```python
import csv
import requests

MANAGER = "https://manager.groupedistri.example"
TEMPLATES = {
    "proximite": "TPL-PROXIMITE-V3",
    "supermarche": "TPL-SUPER-V3",
    "hyper": "TPL-HYPER-V3",
}

session = requests.Session()
session.headers.update({"Authorization": "Bearer <TOKEN>"})

def prepare_site(row):
    payload = {
        "serial": row["serial"],
        "template": TEMPLATES[row["site_type"]],
        "variables": {
            "hostname": row["hostname"],
            "system_ip": row["system_ip"],
            "site_id": row["site_id"],
            "lan_caisses": row["lan_caisses"],
            "lan_monetique": row["lan_monetique"],
        },
    }
    r = session.post(f"{MANAGER}/api/v1/sites", json=payload, timeout=30)
    r.raise_for_status()
    return r.json()

with open("sites_a_deployer.csv", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        result = prepare_site(row)
        print(f"Site {row['site_id']} prêt pour ZTP : {result.get('status', 'ok')}")
```

Les routes d'API, l'authentification et les champs réels dépendent de l'éditeur et de la version.

### 12.5 Validation automatique après ZTP

Un script interroge le manager à intervalles réguliers après l'expédition :

1. Le site est-il apparu dans l'inventaire ?
2. Les connexions de contrôle sont-elles établies ?
3. Les sessions BFD sont-elles actives ?
4. La version logicielle est-elle conforme ?
5. Les tests de connectivité applicative sont-ils bons ?

Si un contrôle échoue au bout d'un délai défini, un ticket est créé automatiquement pour le support.

### 12.6 Déploiement par vagues

| Vague | Périmètre | Objectif |
|---|---|---|
| Pilote | 5 à 10 sites variés | Valider processus et templates |
| Vague 1 | 1 % du parc | Mesurer taux de succès et durée |
| Vague 2 | 10 % | Ajuster le support et la logistique |
| Vague 3 | Reste du parc par région | Déploiement à rythme soutenu |

Critère de passage à la vague suivante : taux de ZTP réussi au premier essai supérieur à un seuil défini (par exemple 95 %).

---

## 13. Sécurité du ZTP

### 13.1 Menaces

| Menace | Description |
|---|---|
| Équipement volé | Un voleur tente de connecter un équipement dérobé |
| Usurpation de numéro de série | Tentative de faire passer un faux équipement |
| Interception du trafic d'amorçage | Attaque sur le premier lien (Wi-Fi public, réseau non maîtrisé) |
| Compromission du portail de redirection | Détournement vers un faux orchestrateur |
| Fuite d'un fichier de bootstrap | Exposition de paramètres d'organisation |
| Équipement revendu ou non réinitialisé | Réutilisation d'un équipement hors parc |

### 13.2 Parades

- **Authentification mutuelle par certificats** : l'équipement vérifie l'orchestrateur et réciproquement.
- **Liste blanche de numéros de série** : un équipement inconnu est refusé.
- **Désactivation rapide** d'un équipement volé : retrait de la liste blanche et révocation du certificat.
- **Fichiers de bootstrap** : ne contenant pas de secrets réutilisables, à durée de validité limitée, stockés de façon sécurisée.
- **Chiffrement** de toutes les sessions de contrôle et de gestion.
- **Journalisation** de chaque tentative de ZTP (réussie ou non), avec alertes sur les échecs répétés.
- **Contrôles de géolocalisation ou d'adresse source** si pertinent : alerte si un équipement apparaît depuis un pays ou un FAI inattendu.
- **Segmentation du réseau de gestion** et accès administrateur protégé (MFA).

### 13.3 Équipement volé : procédure

1. Détection ou signalement (colis non reçu, vol en magasin).
2. Retrait immédiat du numéro de série de la liste autorisée.
3. Révocation du certificat de l'équipement.
4. Surveillance des tentatives de connexion avec cette identité.
5. Déclaration de remplacement.

### 13.4 Conformité

Pour les environnements PCI DSS :

- le ZTP ne doit pas affaiblir la segmentation du paiement ;
- les journaux de provisioning sont conservés ;
- les accès au manager sont contrôlés et audités.

---

## 14. Dépannage du ZTP

### 14.1 Méthodologie

Raisonner **dans l'ordre de la séquence** :

1. Connectivité IP (DHCP, passerelle, DNS).
2. Découverte (redirection, résolution).
3. Authentification (certificat, numéro de série, heure).
4. Contrôle et gestion (connexions aux contrôleurs et au manager).
5. Configuration (template, variables).
6. Overlay (tunnels, BFD, OMP).

### 14.2 Tableau de diagnostic

| Étape bloquée | Symptômes | Causes fréquentes | Vérifications |
|---|---|---|---|
| IP | Pas d'adresse | Mauvais port, DHCP absent, VLAN imposé | Voyants, test d'un PC sur le même câble |
| Découverte | Ne trouve pas l'orchestrateur | DNS filtré, pare-feu, numéro de série inconnu | Résolution DNS, portail de redirection |
| Authentification | Session rejetée | Certificat expiré, heure fausse, numéro non autorisé | Logs de l'orchestrateur, NTP |
| Contrôle | Connexion qui oscille | NAT restrictif, filtrage UDP | Capture, logs, test sur un autre lien |
| Configuration | Site actif sans services | Variable manquante, template incompatible | Vue de configuration dans le manager |
| Overlay | Pas de tunnels | Politique bloquante, UDP bloqué, MTU | BFD, `show sdwan ...` |

### 14.3 Commandes utiles (Cisco cEdge, à titre d'exemple)

```
show sdwan control connections
show sdwan control connection-history
show sdwan control local-properties
show sdwan certificate installed
show sdwan certificate serial
show sdwan bfd sessions
show sdwan omp summary
show clock
show ip interface brief
show platform software ...
```

Selon la version, certaines commandes diffèrent. Les journaux de l'orchestrateur et du manager sont souvent plus informatifs que ceux de l'edge pour comprendre un rejet.

### 14.4 Rôle du support N1 avec le gérant

Prévoir une **grille de questions simples** :

1. Quelle est la couleur des voyants de l'équipement ?
2. Les câbles sont-ils bien branchés sur les ports repérés WAN 1 et LAN 1 ?
3. Le voyant de la box ou de l'ONT est-il vert ?
4. Un redémarrage de l'équipement (débrancher 30 secondes) a-t-il été essayé ?

Le support N1 s'appuie sur un tableau de bord montrant l'état du ZTP en temps réel (étape atteinte, dernière erreur).

### 14.5 Escalade

| Niveau | Intervenant | Cas |
|---|---|---|
| N1 | Support | Branchement, voyants, redémarrage |
| N2 | Équipe réseau | Rejets, certificats, templates |
| N3 | Éditeur ou intégrateur | Anomalie logicielle, matériel défectueux |

---

## 15. Indicateurs de réussite

| Indicateur | Définition | Cible indicative |
|---|---|---|
| Taux de ZTP réussi au premier essai | Sites actifs sans intervention humaine | Supérieur à 95 % |
| Durée de ZTP médiane | Du branchement à l'état opérationnel | Inférieure à 30 minutes |
| Délai de remplacement (RMA) | De la panne au retour en production | Inférieur à 24 heures |
| Taux de recours à une intervention sur site | Interventions physiques | Inférieur à 2 % |
| Taux d'erreurs de variables | Sites nécessitant une correction | Inférieur à 3 % |
| Conformité de version | Sites sur la version cible | Supérieur à 98 % |
| Temps de création d'un site côté siège | Inventaire complet à site prêt | Inférieur à 1 heure (avec automatisation) |

Ces cibles sont indicatives et à adapter à votre contexte.

---

## 16. Bonnes pratiques et pièges

### Bonnes pratiques

1. **Fiabiliser l'inventaire** : une seule source de vérité, validée par script.
2. **Standardiser** : peu de profils de site, templates versionnés.
3. **Stager** les équipements critiques.
4. **Surveiller** l'état du ZTP en temps réel avec un tableau de bord.
5. **Tester** régulièrement la procédure complète sur un site pilote.
6. **Documenter** le guide du gérant avec visuels, une page maximum.
7. **Prévoir** une procédure RMA et un stock tampon.
8. **Contrôler les certificats** avec alertes d'expiration.
9. **Garder l'orchestrateur disponible** : haute disponibilité et supervision dédiée.
10. **Mesurer** les indicateurs et améliorer le processus à chaque vague.

### Pièges

- Numéros de série mal saisis ou non déclarés.
- Variables incohérentes (chevauchement de sous-réseaux entre sites).
- Mise à jour logicielle trop lourde pour un lien cellulaire.
- Horloge non synchronisée.
- Oubli de la révocation d'un équipement remplacé.
- Box FAI configurée en mode inhabituel (bridge, VLAN imposé, PPPoE).
- Guide du gérant trop technique ou ambigu sur les ports.
- Absence de mode dégradé pendant l'attente d'un remplacement.

---

## 17. Exercices et corrigés

### Exercice 1 : séquence

**Énoncé.** Classez dans l'ordre : a) application du template, b) DHCP, c) authentification par certificat, d) redirection par numéro de série, e) construction des tunnels IPsec, f) mise à jour logicielle.

**Corrigé.** b, d, c, puis f et a (la mise à jour peut précéder ou suivre l'application du template selon la politique, mais elle intervient avant la mise en production), puis e.

### Exercice 2 : identification d'un blocage

**Énoncé.** Un edge obtient une IP, mais n'atteint jamais l'orchestrateur. Le même réseau permet à un PC de naviguer. Donnez trois hypothèses et un test pour chacune.

**Corrigé.**

1. Numéro de série non déclaré dans le portail : vérifier l'inventaire et la redirection.
2. Flux sortant spécifique bloqué (UDP DTLS) : tester depuis un autre lien (hotspot 5G).
3. DNS ne résolvant pas le nom de l'orchestrateur : tester la résolution depuis un PC sur le même réseau.

### Exercice 3 : certificat

**Énoncé.** Un edge est rejeté pour "certificat non valide". L'horloge affiche l'année 2015. Quelle est la cause probable et comment prévenir ?

**Corrigé.** L'heure erronée rend les certificats du plan de contrôle "non encore valides". Prévention : synchronisation NTP très tôt dans le processus, horloge matérielle fiable, staging qui met l'horloge à jour.

### Exercice 4 : variables

**Énoncé.** Le site 4905 doit recevoir `system_ip = 10.255.A.B`, avec A et B définis comme dans le cours. Calculez les valeurs et la plage `lan_caisses`.

**Corrigé.** N = 4905 donne A = 49 et B = 5. `system_ip = 10.255.49.5`, `lan_caisses = 10.20.49.0/24`.

### Exercice 5 : plan de déploiement

**Énoncé.** Vous devez déployer 300 magasins de proximité en 3 mois. Proposez un plan en vagues avec critères de passage.

**Corrigé (exemple).**

| Semaine | Vague | Volume | Critère de passage |
|---|---|---|---|
| 1 à 2 | Pilote | 10 | Taux de ZTP réussi de 100 %, retour du gérant favorable |
| 3 à 4 | Vague 1 | 30 | Au moins 95 % de réussite au premier essai |
| 5 à 8 | Vague 2 | 100 | Durée médiane inférieure à 30 minutes, support N1 maîtrisé |
| 9 à 12 | Vague 3 | 160 | Indicateurs stables, stock de RMA constitué |

### Exercice 6 : sécurité

**Énoncé.** Un colis contenant 5 edges disparaît pendant le transport. Quelles actions immédiates ?

**Corrigé.** Retirer les cinq numéros de série de la liste autorisée, révoquer d'éventuels certificats opérationnels, surveiller les tentatives de connexion avec ces identités, déclarer le sinistre au transporteur, expédier des équipements de remplacement et mettre à jour l'inventaire.

---

## 18. Quiz de révision

1. Qu'est-ce qui distingue le ZTP d'un simple DHCP avec fichier de configuration ?
2. Quel est le rôle de la liste blanche de numéros de série ?
3. Pourquoi l'horloge est-elle sensible dans le ZTP ?
4. Citez deux avantages du staging.
5. Quelle est la différence entre Day 0 et Day 1 ?
6. Pourquoi privilégier un DHCP fourni par la box FAI plutôt qu'une configuration statique ?
7. Comment gérer l'ouverture d'un magasin avant la livraison de la fibre ?
8. Que faire lorsqu'un edge est remplacé suite à une panne ?
9. Citez trois indicateurs de suivi d'un programme ZTP.
10. Pourquoi le ZTP en haute disponibilité se fait-il un équipement à la fois ?

**Éléments de réponse**

1. Le ZTP inclut une authentification mutuelle, une orchestration du plan de contrôle et l'application de politiques, pas seulement de la configuration.
2. Empêcher qu'un équipement inconnu ou volé rejoigne le réseau.
3. La validation des certificats dépend de l'heure.
4. Gain de temps sur site (version logicielle déjà à jour), détection précoce des défauts.
5. Day 0 : connexion au plan de contrôle et configuration initiale. Day 1 : configuration complète des services du site.
6. Parce que le ZTP est conçu pour fonctionner sans configuration préalable, et que la box FAI est le cas standard en proximité.
7. ZTP via 4G ou 5G avec une politique de mode dégradé, puis bascule vers la fibre à son arrivée.
8. Retirer l'ancien numéro de série, révoquer son certificat, déclarer le nouveau, rattacher au même site logique avec les mêmes variables.
9. Taux de ZTP réussi au premier essai, durée médiane, délai de RMA (entre autres).
10. Pour éviter les conflits de rôle et valider la bascule entre les deux équipements de façon contrôlée.

---

## 19. Glossaire

| Terme | Définition |
|---|---|
| Bootstrap | Configuration minimale permettant à un équipement de joindre le plan de contrôle |
| Day 0, Day 1, Day 2 | Phases de vie : mise en service initiale, configuration complète, exploitation |
| Liste blanche | Liste des numéros de série autorisés à rejoindre le réseau |
| Orchestrateur | Composant qui authentifie les équipements et les dirige vers les contrôleurs |
| PnP | Plug and Play, mécanisme de découverte et de redirection (terme utilisé chez Cisco) |
| Portail de redirection | Service cloud de l'éditeur qui associe un numéro de série à l'orchestrateur d'un client |
| RMA | Return Merchandise Authorization, procédure de remplacement d'un équipement défectueux |
| Staging | Préparation d'un équipement (mise à jour, tests) avant livraison sur site |
| SUDI | Secure Unique Device Identifier, identité matérielle installée en usine (terminologie Cisco) |
| Template | Modèle de configuration à variables appliqué par le manager |
| Variable | Valeur propre à un site injectée dans un template |
| ZTP | Zero Touch Provisioning, mise en service automatique sans intervention technique locale |

---

## Pour aller plus loin

- Documentation officielle de votre éditeur sur le provisioning (Plug and Play, FortiDeploy, activation Aruba Central, etc.).
- Guides de bonnes pratiques de déploiement à grande échelle.
- Laboratoires virtuels permettant de rejouer la séquence de ZTP sans matériel.

---

*Fin du cours.*
