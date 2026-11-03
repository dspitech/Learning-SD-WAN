# Cours complet SD-WAN et SLA (Service Level Agreement et classes SLA)

Ce cours fait suite aux cours sur OMP et BFD. Le terme SLA recouvre en réalité deux notions complémentaires que l'on rencontre dans tout projet SD-WAN :

- Le SLA contractuel : l'engagement pris par un opérateur ou un prestataire (disponibilité, délais de rétablissement, performances).
- Les classes SLA techniques : les seuils de perte, de latence et de gigue configurés dans la politique SD-WAN pour choisir le meilleur chemin pour chaque application.

Le fil rouge reste le réseau d'une enseigne de grande distribution de type Carrefour.

> Note importante : le cas Carrefour est un scénario pédagogique fictif. Les plans d'adressage, les contrats, les taux de disponibilité, les mesures et les volumes sont inventés à des fins d'illustration et ne décrivent ni l'architecture réelle de l'enseigne ni ses contrats réels.

> Note de vocabulaire : Cisco a rebaptisé sa solution Catalyst SD-WAN. Selon les versions de la documentation, vManage devient SD-WAN Manager, vSmart devient SD-WAN Controller et vBond devient SD-WAN Validator. Ce cours conserve les noms historiques, plus répandus.

---

## Table des matières

1. Objectifs et prérequis
2. SLA, SLO, SLI : clarifier le vocabulaire
3. Le SLA contractuel des opérateurs
4. Calculer et composer la disponibilité
5. Le SLA technique dans Cisco SD-WAN : la classe SLA
6. Comment BFD alimente les classes SLA
7. Construire les classes SLA : méthode
8. Politiques de routage applicatif (AAR) et comportement de repli
9. SLA et QoS : deux mécanismes complémentaires
10. Cas d'étude fil rouge : le réseau d'une enseigne de distribution
11. Configuration pas à pas
12. Supervision, rapports et preuve auprès de l'opérateur
13. Vérification et dépannage
14. Pièges classiques
15. Bonnes pratiques
16. Travaux pratiques
17. Questionnaire d'auto-évaluation
18. Glossaire
19. Références

---

## 1. Objectifs et prérequis

### Objectifs pédagogiques

À la fin de ce cours, vous serez capable de :

- Distinguer SLA contractuel, SLO et SLI, et SLA technique (classe SLA).
- Lire un contrat opérateur (GTI, GTR, disponibilité, classes de service, exclusions).
- Calculer une disponibilité théorique et la composer sur des liens redondants.
- Définir des classes SLA réalistes à partir de mesures et de besoins applicatifs.
- Écrire une politique de routage applicatif avec classes SLA, couleurs préférées et repli.
- Distinguer le rôle de l'AAR et celui de la QoS.
- Produire un rapport de conformité et constituer une preuve pour une réclamation opérateur.

### Prérequis

- Le cours sur OMP (TLOC, color, politiques de contrôle).
- Le cours sur BFD (hello, multiplicateur, poll interval, mesures de qualité).
- Notions de QoS (DSCP, files d'attente).

---

## 2. SLA, SLO, SLI : clarifier le vocabulaire

| Sigle | Signification | Nature | Exemple |
|---|---|---|---|
| SLI | Service Level Indicator | Une mesure | Latence mesurée entre un magasin et le datacenter : 22 ms |
| SLO | Service Level Objective | Un objectif interne | 99,95 % de disponibilité de l'encaissement pendant les heures d'ouverture |
| SLA | Service Level Agreement | Un engagement contractuel, assorti de pénalités | L'opérateur garantit 99,9 % de disponibilité du lien MPLS |
| Classe SLA (SD-WAN) | Seuils techniques d'une politique | Un seuil de décision pour le routage | Perte maximale 1 %, latence maximale 100 ms, gigue maximale 30 ms |

Remarque importante : la classe SLA du SD-WAN n'est pas un contrat. C'est un critère de décision de routage. Elle peut être plus stricte ou plus souple que le contrat de l'opérateur, et son rôle est de détecter les dégradations que le contrat ne couvre pas.

### 2.1 Indicateurs de disponibilité et de rétablissement

| Terme | Définition |
|---|---|
| Disponibilité | Part du temps pendant laquelle le service est opérationnel sur une période donnée |
| GTI | Garantie de Temps d'Intervention : délai maximal avant la prise en charge de l'incident |
| GTR | Garantie de Temps de Rétablissement : délai maximal pour rétablir le service |
| MTBF | Mean Time Between Failures : temps moyen entre pannes |
| MTTR | Mean Time To Repair : temps moyen de réparation |
| RTO | Recovery Time Objective : durée maximale acceptable d'interruption |

---

## 3. Le SLA contractuel des opérateurs

### 3.1 Contenu typique d'un contrat

- Disponibilité garantie (par exemple 99,5 %, 99,9 %, 99,95 %), mesurée sur une période mensuelle ou annuelle.
- GTI et GTR, avec plage horaire de couverture : heures ouvrées (HO), 24 heures sur 24 et 7 jours sur 7.
- Performances : perte, latence, gigue, parfois garanties par classe de service sur les réseaux MPLS.
- Pénalités : crédits de facturation, généralement plafonnés.
- Exclusions : maintenances planifiées, force majeure, défaut côté client, coupure de l'alimentation électrique du site.
- Périmètre de mesure : point à point entre routeurs d'opérateur (PE vers PE) ou de bout en bout jusqu'au routeur client (CE vers CE).

### 3.2 Niveaux d'engagement selon le type de transport

| Transport | Engagement typique | Remarque |
|---|---|---|
| MPLS opérateur | Disponibilité élevée, GTR courte, classes de service (temps réel, critique, standard) | Le plus cher, performances garanties sur le coeur de réseau |
| Fibre professionnelle (accès internet dédié) | Disponibilité et GTR contractuelles, mais pas de garantie de performance de bout en bout au-delà du réseau de l'opérateur | Bon compromis |
| Internet grand public (ADSL, fibre mutualisée, câble) | Meilleur effort, GTR longue | Faible coût, qualité variable |
| 4G et 5G | Meilleur effort, pas de GTR significative | Lien de secours |

Les chiffres exacts varient fortement selon les opérateurs et les offres. Ils sont à lire dans chaque contrat.

### 3.3 Un point essentiel : le périmètre de mesure

Le SLA de l'opérateur est souvent mesuré sur son propre coeur de réseau. La mesure BFD du SD-WAN, elle, couvre le chemin complet entre les deux WAN Edge : boucle locale, accès, coeur de réseau, chiffrement IPsec et traitement des routeurs. Il peut donc y avoir un écart entre :

- Un réseau opérateur conforme à son SLA.
- Un chemin de bout en bout dégradé pour l'application (par exemple saturation de la boucle locale du magasin).

Cet écart explique pourquoi un SLA contractuel respecté ne garantit pas la satisfaction des utilisateurs, et pourquoi le SD-WAN ajoute une mesure de bout en bout.

---

## 4. Calculer et composer la disponibilité

### 4.1 Conversion d'une disponibilité en indisponibilité annuelle

Une année compte 8760 heures, soit 525 600 minutes.

| Disponibilité | Indisponibilité annuelle (24 h sur 24) |
|---|---|
| 99 % | environ 87,6 heures |
| 99,5 % | environ 43,8 heures |
| 99,9 % | environ 8,76 heures |
| 99,95 % | environ 4,38 heures |
| 99,99 % | environ 52,6 minutes |
| 99,999 % | environ 5,3 minutes |

### 4.2 Composition de liens redondants

Pour deux liens indépendants de disponibilités A1 et A2 utilisés en redondance, la disponibilité combinée théorique est :

```
A = 1 - (1 - A1) x (1 - A2)
```

Exemple pour un magasin de proximité (scénario fictif) :

- Lien fibre : 99,5 %.
- Lien 4G : 99,0 %.
- Disponibilité combinée théorique : 1 - (0,005 x 0,01) = 99,995 %, soit environ 26 minutes d'indisponibilité par an.

Exemple pour un hypermarché (scénario fictif) :

- MPLS : 99,9 %.
- Fibre opérateur A : 99,5 %.
- Fibre opérateur B : 99,5 %.
- Disponibilité combinée théorique : 1 - (0,001 x 0,005 x 0,005) = 99,9999975 %. Ce chiffre n'a pas de sens pratique, car d'autres pannes dominent.

### 4.3 Limites de ce calcul

Le calcul suppose l'indépendance des pannes. Dans la réalité, des causes communes limitent le gain :

- Deux accès qui partagent le même fourreau, la même boucle locale ou le même point d'entrée du bâtiment.
- Deux opérateurs qui s'appuient sur la même infrastructure physique en amont.
- Une panne d'alimentation électrique du site ou du local technique.
- Une panne du routeur WAN Edge lui-même (d'où la redondance matérielle sur les gros sites).
- Une erreur de configuration commune (une politique poussée par erreur sur tous les sites).
- Le temps de bascule lui-même.

### 4.4 Prise en compte du temps de bascule

Chaque panne de lien provoque une interruption de la durée de détection BFD avant la bascule.

Exemple : 20 pannes de lien nominal par an et par magasin.

| Réglage BFD | Détection | Interruption cumulée annuelle |
|---|---|---|
| Hello 1 s, multiplicateur 7 | 7 s | 140 s (environ 2,3 min) |
| Hello 500 ms, multiplicateur 4 | 2 s | 40 s |

Ce temps est faible par rapport aux minutes d'indisponibilité du calcul précédent, mais il compte pour les caisses : une interruption de 7 secondes peut provoquer des échecs de transaction. Le réglage des timers est donc aussi une décision de niveau de service.

### 4.5 Disponibilité sur les heures d'ouverture

Un magasin n'a pas besoin du même niveau de service la nuit. L'indicateur le plus pertinent pour l'enseigne est souvent la disponibilité de l'encaissement pendant les heures d'ouverture. Les maintenances planifiées la nuit ne comptent donc pas contre l'objectif interne, même si elles figurent dans les exclusions contractuelles de l'opérateur.

---

## 5. Le SLA technique dans Cisco SD-WAN : la classe SLA

### 5.1 Définition

Une classe SLA (sla-class) est un jeu de seuils maximaux acceptables pour trois critères mesurés par BFD :

| Critère | Unité | Signification |
|---|---|---|
| `loss` | pourcentage | Taux maximal de paquets perdus |
| `latency` | millisecondes | Délai maximal |
| `jitter` | millisecondes | Variation maximale du délai |

Un tunnel est dit conforme à une classe SLA si ses mesures respectent les trois seuils (selon la configuration, un seul critère peut aussi être défini). Les unités de latence (aller simple ou aller-retour) peuvent dépendre de la version : à vérifier dans la documentation de votre release.

### 5.2 Exemple de classes SLA

```
policy
 sla-class SLA-CAISSES
  loss    1
  latency 100
  jitter  30
 !
 sla-class SLA-VOIX-VISIO
  loss    1
  latency 150
  jitter  30
 !
 sla-class SLA-BUREAUTIQUE
  loss    5
  latency 300
  jitter  100
 !
 sla-class SLA-BEST-EFFORT
  loss    10
  latency 500
  jitter  200
 !
!
```

Le nombre de classes SLA par politique est limité. Vérifiez la limite de votre version.

### 5.3 Repli : que faire si aucun tunnel n'est conforme

Deux cas sont à anticiper :

| Mode | Comportement | Usage |
|---|---|---|
| Par défaut | Si aucun tunnel n'est conforme, le trafic est acheminé sur le chemin habituel de routage (ECMP sur les tunnels disponibles) | Cas courant, évite de couper le service |
| `strict` | Si aucun tunnel n'est conforme, le trafic est rejeté | À réserver à des cas précis, car il peut couper un service critique |
| `fallback-best-tunnel` | Si aucun tunnel n'est conforme, utilise le meilleur tunnel selon des critères de comparaison | Compromis recommandé pour les flux critiques |

Exemple avec repli sur le meilleur tunnel (la syntaxe peut varier selon la version) :

```
sla-class SLA-CAISSES
 loss    1
 latency 100
 jitter  30
 fallback-best-tunnel
  criteria loss latency jitter
  loss-variance    10
  latency-variance 10
  jitter-variance  10
 !
!
```

Les variances définissent l'écart toléré (en pourcentage) entre le meilleur tunnel et les autres pour considérer que plusieurs tunnels sont équivalents. Elles évitent de basculer sur de petites différences.

---

## 6. Comment BFD alimente les classes SLA

### 6.1 Principe de la mesure

BFD circule dans chaque tunnel IPsec et sert de sonde. Le WAN Edge calcule, pour chaque tunnel, la perte, la latence et la gigue.

### 6.2 Fenêtre de mesure

- `app-route poll-interval` : durée d'une fenêtre de mesure (par défaut 600000 ms, soit 10 minutes).
- `app-route multiplier` : nombre de fenêtres prises en compte (par défaut 6).

Le tunnel est évalué sur la base de ces mesures agrégées. Le temps de réaction à une dégradation de qualité dépend donc du produit poll interval x multiplicateur, selon le mécanisme de moyenne propre à la version.

### 6.3 Compromis de réglage

| Réglage | Effet |
|---|---|
| Poll interval long (valeur par défaut) | Mesures lissées, réaction lente, peu d'oscillations |
| Poll interval court (de l'ordre de 10 à 120 secondes) | Réaction rapide à une dégradation, mais mesures plus bruitées et risque d'oscillation |

Les caisses réclament une réaction relativement rapide, les flux de sauvegarde n'en ont pas besoin. Le poll interval est cependant global à l'équipement, il faut donc choisir une valeur de compromis.

### 6.4 Remarque sur la représentativité

Les paquets BFD sont petits et marqués avec un DSCP prioritaire. Ils peuvent donc ne pas refléter exactement ce que vivent des gros paquets de données dans une file standard. Par exemple, sur un lien saturé en sortie, une QoS mal conçue peut dégrader les flux applicatifs plus que les paquets BFD. Cela justifie de combiner la mesure BFD avec une QoS correcte et la supervision applicative.

---

## 7. Construire les classes SLA : méthode

### 7.1 Démarche en cinq étapes

1. Inventorier les applications critiques et leur sensibilité (transactionnelle, temps réel, volumétrique).
2. Mesurer l'état normal : relever la perte, la latence et la gigue par couleur sur plusieurs semaines (valeurs moyennes, P95, P99).
3. Fixer les seuils au-dessus des valeurs normales, avec une marge, et au-dessous du seuil de gêne applicative.
4. Valider sur un site pilote et ajuster.
5. Revoir périodiquement les seuils avec les nouveaux relevés.

### 7.2 Repères de seuils couramment cités

Pour la voix et la visioconférence interactive, les recommandations de conception Cisco QoS et l'UIT-T G.114 convergent vers :

| Critère | Valeur recommandée (voix) |
|---|---|
| Latence aller simple | 150 ms ou moins |
| Gigue | 30 ms ou moins |
| Perte | 1 % ou moins |

Ce sont des repères à adapter, pas des obligations. Pour des applications transactionnelles (encaissement), ce sont surtout la perte et les retransmissions TCP qui provoquent des temps de réponse dégradés.

### 7.3 Un piège : des seuils irréalistes

Des seuils plus stricts que la qualité normale d'un lien entraînent des violations permanentes du SLA. Le trafic oscille alors entre les tunnels, ou bascule systématiquement sur le chemin de secours coûteux (la 4G par exemple). Il faut toujours partir des mesures réelles.

---

## 8. Politiques de routage applicatif (AAR) et comportement de repli

### 8.1 Rappel de la structure

Une politique de routage applicatif (app-route-policy) associe, par VPN et par séquence, des critères de reconnaissance du trafic (application, DSCP, préfixe, port) à une classe SLA et à des préférences de couleur.

### 8.2 Options d'action

| Option | Rôle |
|---|---|
| `sla-class` | Classe SLA à respecter |
| `preferred-color` | Couleur(s) préférée(s) parmi les tunnels conformes |
| `backup-sla-preferred-color` | Couleur(s) de secours utilisées si aucun tunnel de la couleur préférée n'est conforme |
| `strict` | Interdit le repli par défaut : le trafic est rejeté si aucun tunnel n'est conforme |
| `count`, `log` | Compteurs et journalisation des violations |

### 8.3 Logique de sélection

1. Le WAN Edge identifie le trafic (application, DSCP).
2. Il liste les tunnels conformes à la classe SLA.
3. Parmi ces tunnels, il choisit la couleur préférée si elle est présente.
4. Sinon, il choisit une couleur de secours, puis tout tunnel conforme.
5. Si aucun tunnel n'est conforme, le repli dépend de la configuration : chemin par défaut, meilleur tunnel, ou rejet en mode strict.

### 8.4 Exemple de politique complète

```
policy
 lists
  vpn-list VPN-CAISSES
   vpn 10
  !
  vpn-list VPN-BACKOFFICE
   vpn 20
  !
  site-list MAGASINS
   site-id 2000-2999
  !
 !
 sla-class SLA-CAISSES
  loss    1
  latency 100
  jitter  30
 !
 sla-class SLA-BUREAUTIQUE
  loss    5
  latency 300
  jitter  100
 !
 app-route-policy AAR-MAGASINS
  vpn-list VPN-CAISSES
   sequence 10
    match
     dscp 46
    !
    action
     sla-class SLA-CAISSES
      preferred-color mpls biz-internet
      backup-sla-preferred-color lte
     !
    !
   !
   default-action
    sla-class SLA-CAISSES
   !
  !
  vpn-list VPN-BACKOFFICE
   sequence 10
    match
     dscp 18
    !
    action
     sla-class SLA-BUREAUTIQUE
      preferred-color biz-internet
     !
    !
   !
  !
 !
!
apply-policy
 site-list MAGASINS
  app-route-policy AAR-MAGASINS
 !
!
```

La syntaxe exacte varie selon la version. En pratique, on construit ces politiques dans l'interface de vManage.

---

## 9. SLA et QoS : deux mécanismes complémentaires

| | Routage applicatif (AAR) | QoS |
|---|---|---|
| Question traitée | Sur quel tunnel envoyer ce flux ? | Dans quel ordre servir les flux sur un lien ? |
| Moment d'action | Choix du chemin selon qualité mesurée | Mise en file, priorité, mise en forme (shaping) au moment de l'émission |
| Dépend de BFD | Oui | Non |
| Limite | Ne crée pas de bande passante | Ne change pas de chemin |

Les deux sont nécessaires :

- Si un lien est saturé, l'AAR peut déplacer des flux vers un autre tunnel, mais une QoS absente laissera la visioconférence souffrir à côté d'une sauvegarde volumineuse.
- Si une QoS est bien conçue mais que le lien tombe en perte importante, seule l'AAR peut basculer vers un meilleur chemin.

Bonne pratique : marquer le trafic (DSCP) à l'entrée du site, appliquer des files d'attente cohérentes sur chaque interface WAN avec mise en forme (shaping) au débit souscrit, et utiliser ces mêmes marquages comme critères de correspondance dans l'AAR.

---

## 10. Cas d'étude fil rouge : le réseau d'une enseigne de distribution

> Rappel : scénario fictif à vocation pédagogique.

### 10.1 Rappel du contexte

- Deux datacenters (Paris, site-id 1000 et Lyon, site-id 1001).
- Environ 1000 magasins : hypermarchés, supermarchés, proximité.
- VPN 10 caisses, VPN 20 back office, VPN 30 vidéosurveillance, VPN 40 invités, VPN 50 IoT.
- Topologie hub and spoke imposée par OMP, BFD sur chaque tunnel.

### 10.2 Niveaux de service par type de magasin

| Type de magasin | Transports | Engagements opérateur (fictifs) | Objectif interne (SLO) |
|---|---|---|---|
| Hypermarché | MPLS + 2 fibres de deux opérateurs | MPLS 99,9 % GTR 4 h 24 h sur 24, fibres 99,5 % GTR 4 h | Encaissement 99,99 % en heures d'ouverture |
| Supermarché | MPLS + fibre | MPLS 99,9 %, fibre 99,5 % | Encaissement 99,95 % |
| Proximité | Fibre + 4G ou 5G | Fibre 99,5 % GTR 8 h, 4G meilleur effort | Encaissement 99,9 % |

Les SLO internes sont définis par le métier, ce sont eux qui guident la conception, pas les contrats seuls.

### 10.3 Classes SLA par type de flux

| Flux | VPN | Marquage | Classe SLA | Couleur préférée | Secours |
|---|---|---|---|---|---|
| Transactions de paiement et caisses | 10 | DSCP 46 | SLA-CAISSES (1 %, 100 ms, 30 ms) | mpls puis biz-internet | lte |
| Visioconférence et voix magasin | 20 | DSCP 34 ou 46 | SLA-VOIX-VISIO (1 %, 150 ms, 30 ms) | mpls puis biz-internet | lte déconseillé selon forfait |
| ERP, gestion de stock | 20 | DSCP 18 | SLA-BUREAUTIQUE (5 %, 300 ms, 100 ms) | biz-internet | mpls |
| Vidéosurveillance | 30 | DSCP 18 ou 10 | Classe adaptée (3 %, 200 ms, 80 ms) | biz-internet | mpls |
| Sauvegardes et mises à jour | 20 | DSCP 8 | SLA-BEST-EFFORT | biz-internet | aucun, jamais sur lte |
| Wi-Fi invités | 40 | DSCP 0 | Aucune | Sortie internet locale | Aucun |

Remarque : la sauvegarde ne doit jamais basculer sur la 4G, sous peine de consommer le forfait. L'AAR ne s'applique donc pas ou la couleur lte est exclue pour ce flux.

### 10.4 Scénario 1 : dégradation sans coupure

Un soir, la fibre d'un supermarché subit 3 % de perte à cause d'une congestion chez l'opérateur. Aucun BFD ne tombe, mais :

1. La perte mesurée dépasse le seuil de 1 % de SLA-CAISSES.
2. Le tunnel fibre n'est plus conforme pour les caisses.
3. Le WAN Edge bascule les flux des caisses vers le tunnel MPLS (conforme).
4. Le trafic de sauvegarde, dont le seuil est de 10 %, reste sur la fibre.
5. Quand la perte redescend, les caisses reviennent sur la fibre ou restent sur le MPLS, selon la couleur préférée.

Sans AAR, les caisses seraient restées sur le lien dégradé alors que le lien est techniquement "up".

### 10.5 Scénario 2 : le piège de la 4G

Dans un magasin de proximité, la fibre est dégradée. La 4G, secours des caisses, présente 80 ms de latence et 0,5 % de perte, donc conforme à SLA-CAISSES.

- Les flux des caisses basculent sur la 4G.
- Le trafic de sauvegarde et de mise à jour, lui, n'est pas autorisé sur lte : il ne bascule pas.
- Une alarme prévient le centre d'exploitation qu'un site opère sur son lien de secours.

Sans cette sélectivité, un lot de mises à jour pourrait épuiser le forfait de données en quelques heures.

### 10.6 Scénario 3 : aucun tunnel conforme

Panne partielle à Paris et dégradation de l'opérateur : tous les tunnels d'un magasin dépassent le seuil de latence de SLA-CAISSES. Selon le mode :

- Mode par défaut : le trafic reste acheminé sur les chemins existants. Les caisses fonctionnent, mais lentement.
- Mode `fallback-best-tunnel` : le trafic emprunte le tunnel le moins dégradé. C'est le choix recommandé pour les caisses.
- Mode `strict` : les caisses seraient coupées. C'est inacceptable pour ce flux.

### 10.7 Rapport mensuel de conformité (exemple fictif)

| Indicateur | Mesure |
|---|---|
| Magasins mesurés | 1000 |
| Disponibilité de l'encaissement en heures d'ouverture | 99,96 % (objectif 99,95 %) |
| Basculements automatiques sur lien de secours | 312 |
| Durée cumulée sur lien de secours | 47 heures sur l'ensemble du parc |
| Violations SLA-CAISSES | 1 450, dont 62 % sur trois opérateurs d'accès |
| Latence médiane vers Paris | 21 ms |
| Latence P95 vers Paris | 48 ms |
| Perte moyenne | 0,08 % |

Ce type de rapport sert à piloter l'objectif interne, à identifier les opérateurs ou les sites problématiques et à fonder les réclamations.

---

## 11. Configuration pas à pas

### 11.1 Paramètres de mesure BFD

```
sdwan
 bfd app-route multiplier 5
 bfd app-route poll-interval 60000
 bfd color biz-internet
  hello-interval 1000
  multiplier 7
 exit
 bfd color lte
  hello-interval 10000
  multiplier 7
 exit
!
```

Un poll interval de 60 secondes avec un multiplicateur de 5 conduit à une fenêtre d'évaluation de l'ordre de cinq minutes, sensiblement plus réactive que les valeurs par défaut. Validez l'impact (oscillations, charge) avant déploiement général.

### 11.2 Marquage du trafic à l'entrée (exemple de politique de données)

```
policy
 lists
  vpn-list VPN-CAISSES
   vpn 10
  !
 !
 data-policy MARQUAGE-CAISSES
  vpn-list VPN-CAISSES
   sequence 10
    match
     source-data-prefix-list CAISSES
    !
    action accept
     set
      dscp 46
     !
    !
   !
   default-action accept
  !
 !
!
```

La liste de préfixes `CAISSES` est à définir selon le plan d'adressage (par exemple les réseaux des terminaux de paiement).

### 11.3 Séquence de déploiement conseillée

1. Marquer le trafic (DSCP) et configurer la QoS sur les interfaces WAN.
2. Définir les classes SLA à partir des mesures.
3. Déployer l'AAR en mode non strict sur un site pilote.
4. Surveiller les violations, ajuster les seuils.
5. Étendre progressivement par vagues (hypermarchés, supermarchés, proximité).
6. Activer `fallback-best-tunnel` pour les flux critiques.

---

## 12. Supervision, rapports et preuve auprès de l'opérateur

### 12.1 Ce qu'il faut superviser

- Disponibilité des sessions BFD par site et par couleur.
- Perte, latence, gigue par tunnel (moyenne, percentiles P95 et P99).
- Nombre de violations de classes SLA par site, par couleur, par heure.
- Durée de présence sur les liens de secours.
- Nombre de basculements (une valeur anormale signale un lien instable).
- Consommation de données sur les liens cellulaires.

### 12.2 Outils

- Tableaux de bord de vManage (qualité des tunnels, historique des mesures).
- Plateformes d'analyse associées à Catalyst SD-WAN, selon licence.
- Fonctions de qualité d'expérience pour les applications SaaS (par exemple les services de type Cloud OnRamp for SaaS qui évaluent la qualité du chemin vers une application cloud), selon version et licence.
- Export des données vers des outils de supervision tiers par API.

### 12.3 Constituer une réclamation opérateur

1. Dater précisément l'incident (début, fin, couleur concernée).
2. Extraire les mesures de perte, latence et gigue sur la période et le nombre de sessions BFD tombées.
3. Comparer au périmètre de mesure du contrat (PE vers PE ou CE vers CE).
4. Vérifier qu'aucune exclusion ne s'applique (maintenance planifiée, coupure d'alimentation, défaut côté client).
5. Rapprocher le ticket ouvert et les horodatages de GTI et de GTR.
6. Soumettre les preuves dans le canal prévu au contrat.

Les mesures BFD sont des indices solides, mais l'opérateur peut contester leur périmètre : elles incluent la boucle locale et le routeur client. Il faut donc corréler avec des relevés du côté de l'opérateur lorsque c'est possible.

---

## 13. Vérification et dépannage

### 13.1 Commandes principales (cEdge)

| Commande | Utilité |
|---|---|
| `show sdwan app-route stats` | Mesures de perte, latence, gigue par tunnel |
| `show sdwan app-route sla-class` | Classes SLA et tunnels conformes |
| `show sdwan policy from-vsmart` | Politique reçue du contrôleur |
| `show sdwan policy app-route-policy-filter` | Politique AAR appliquée et compteurs |
| `show sdwan policy service-path vpn 10 interface <nom> source-ip <ip> dest-ip <ip> protocol <n> app all` | Chemin retenu pour un flux donné |
| `show sdwan bfd sessions` | État des sessions BFD |
| `show sdwan bfd history` | Historique des transitions |

Sur vEdge, les commandes sont équivalentes sans le mot `sdwan`.

### 13.2 Exemple de sortie : classes SLA (illustratif)

```
Magasin-2750# show sdwan app-route sla-class

                                  SLA CLASS
INDEX  NAME             LOSS  LATENCY  JITTER  FALLBACK
-------------------------------------------------------
0      SLA-CAISSES      1     100      30      yes
1      SLA-VOIX-VISIO   1     150      30      no
2      SLA-BUREAUTIQUE  5     300      100     no
```

### 13.3 Exemple de sortie : tunnels conformes (illustratif)

```
Magasin-2750# show sdwan app-route stats summary

REMOTE SYSTEM IP  LOCAL COLOR    REMOTE COLOR   LOSS  LATENCY  JITTER  SLA CLASSES MET
-----------------------------------------------------------------------------------------
10.255.0.1        mpls           mpls           0     18       2       CAISSES,VOIX,BUREAUTIQUE
10.255.0.1        biz-internet   biz-internet   3     64       12      BUREAUTIQUE
10.255.0.1        lte            lte            0     82       15      CAISSES,VOIX,BUREAUTIQUE
```

Lecture : le tunnel biz-internet présente 3 % de perte. Il n'est conforme qu'à la classe SLA-BUREAUTIQUE. Les flux des caisses utilisent donc le MPLS ou la 4G.

### 13.4 Méthodologie de dépannage

1. Vérifier que la politique est bien reçue et appliquée au site (`show sdwan policy from-vsmart`).
2. Vérifier les mesures par tunnel (`show sdwan app-route stats`).
3. Vérifier à quelle classe SLA chaque tunnel est conforme.
4. Utiliser `service-path` pour savoir quel chemin un flux donné emprunte.
5. Vérifier le marquage DSCP du flux à l'entrée du routeur (sinon l'AAR n'identifie pas le trafic).
6. Contrôler la QoS et la saturation de la bande passante sur le lien.
7. Examiner l'historique des changements de chemin pour détecter une oscillation.

### 13.5 Causes fréquentes d'incident

| Symptôme | Cause probable |
|---|---|
| Aucun flux ne respecte l'AAR | Politique non appliquée à la site-list, mauvais VPN, marquage DSCP absent |
| Bascule permanente vers la 4G | Seuils irréalistes, couleur préférée mal définie |
| Oscillations entre deux tunnels | Seuils proches des mesures normales, poll interval trop court, absence de variance |
| Caisses coupées pendant une dégradation | Mode `strict` activé sur un flux critique |
| Mesures SLA très différentes de l'opérateur | Périmètre de mesure différent (boucle locale, chiffrement) |
| Forfait 4G épuisé | Trafic volumineux autorisé sur lte |
| Applications lentes sans violation SLA | Saturation du lien, QoS absente ou mal calibrée, problème applicatif |

---

## 14. Pièges classiques

1. Confondre le SLA contractuel et la classe SLA technique.
2. Fixer des seuils sans avoir mesuré l'état normal.
3. Activer le mode strict sur des flux critiques.
4. Oublier la QoS et croire que l'AAR suffit.
5. Autoriser les flux volumineux sur les liens à forfait.
6. Compter sur le calcul de disponibilité de deux liens "indépendants" qui partagent la même boucle locale.
7. Négliger le temps de bascule dans l'évaluation du service rendu.
8. Ne pas aligner le périmètre de mesure BFD et celui du contrat lors d'une réclamation.
9. Changer les timers globalement sans test sur un site pilote.
10. Ne pas revoir les seuils après un changement d'opérateur ou de débit.

---

## 15. Bonnes pratiques

1. Définir d'abord les SLO métier, puis en déduire les contrats et les classes SLA.
2. Mesurer pendant plusieurs semaines avant de fixer les seuils.
3. Commencer par peu de classes SLA, cohérentes avec les types de flux.
4. Utiliser un repli sur le meilleur tunnel pour les flux critiques.
5. Combiner AAR, marquage DSCP et QoS avec mise en forme au débit souscrit.
6. Diversifier les opérateurs et les chemins physiques pour des liens réellement indépendants.
7. Réserver les liens cellulaires aux flux essentiels.
8. Automatiser un rapport mensuel de conformité par site, par couleur et par opérateur.
9. Conserver les preuves (mesures horodatées) pour les réclamations.
10. Revoir régulièrement les seuils et les contrats.

---

## 16. Travaux pratiques

### TP 1 : Calcul de disponibilité

1. Calculer la disponibilité combinée pour un magasin avec fibre 99,5 % et 4G 99 %.
2. Calculer l'indisponibilité annuelle en minutes.
3. Ajouter l'effet de 25 basculements par an avec une détection de 7 s, puis de 2 s.
4. Discuter l'effet d'une cause commune (même boucle locale) sur le résultat.

### TP 2 : Définir des classes SLA

1. Dans un laboratoire, relever pendant quelques jours la perte, la latence et la gigue de deux transports.
2. Calculer moyenne, P95 et P99.
3. Proposer des seuils pour trois classes SLA.
4. Justifier chaque seuil par rapport aux mesures et aux besoins applicatifs.

### TP 3 : Politique AAR

1. Définir deux classes SLA (critique et bureautique).
2. Créer une politique avec couleurs préférées et couleur de secours.
3. Dégrader un transport à l'aide d'un émulateur de réseau (netem par exemple).
4. Vérifier le changement de chemin avec `show sdwan policy service-path`.

### TP 4 : Comparer les modes de repli

1. Dégrader tous les tunnels au-delà du seuil.
2. Observer le comportement par défaut, avec `fallback-best-tunnel`, puis avec `strict`.
3. Conclure sur le mode adapté à chaque type de flux.

### TP 5 : Oscillations

1. Fixer un seuil très proche de la latence normale d'un transport.
2. Observer les changements de chemin.
3. Corriger par un réglage des seuils et des variances.

### TP 6 : Rapport de conformité

1. Extraire les mesures de qualité et les transitions BFD d'un laboratoire sur une journée.
2. Construire un rapport avec disponibilité, violations SLA, durée sur lien de secours.
3. Rédiger l'argumentaire d'une réclamation fictive adressée à un opérateur.

---

## 17. Questionnaire d'auto-évaluation

**Q1. Quelle différence entre un SLA contractuel et une classe SLA SD-WAN ?**
Réponse : le premier est un engagement contractuel de l'opérateur. La seconde est un jeu de seuils techniques de décision de routage, sans valeur contractuelle.

**Q2. Quels sont les trois critères d'une classe SLA ?**
Réponse : la perte, la latence et la gigue.

**Q3. Quelle est l'indisponibilité annuelle d'un lien à 99,9 % ?**
Réponse : environ 8,76 heures.

**Q4. Quelle est la disponibilité combinée théorique de deux liens indépendants à 99,5 % chacun ?**
Réponse : 1 - (0,005 x 0,005) = 99,9975 %.

**Q5. Pourquoi ce calcul est-il optimiste ?**
Réponse : il suppose l'indépendance des pannes, alors que boucle locale, alimentation, routeur et erreurs de configuration créent des causes communes.

**Q6. Que fait le mode `strict` d'une politique AAR ?**
Réponse : si aucun tunnel n'est conforme, le trafic est rejeté. Il est dangereux pour les flux critiques.

**Q7. Quel mécanisme mesure la qualité des tunnels pour l'AAR ?**
Réponse : BFD, qui mesure perte, latence et gigue sur chaque tunnel.

**Q8. Pourquoi l'AAR ne remplace-t-il pas la QoS ?**
Réponse : l'AAR choisit un chemin mais ne hiérarchise pas les flux dans les files d'attente ni ne crée de bande passante. La QoS gère la priorité sur un lien.

**Q9. Pourquoi les mesures BFD peuvent-elles différer du SLA opérateur ?**
Réponse : le périmètre de mesure est différent. BFD couvre la boucle locale, le chiffrement et les routeurs clients, alors que l'opérateur mesure souvent sur son propre coeur de réseau.

**Q10. Quelle précaution pour les flux volumineux sur un lien 4G ?**
Réponse : les exclure de la couleur lte (ou ne les inclure dans aucune couleur de secours cellulaire) pour protéger le forfait de données.

---

## 18. Glossaire

| Terme | Définition |
|---|---|
| AAR | Application-Aware Routing, choix du chemin selon la qualité mesurée |
| BFD | Bidirectional Forwarding Detection, sonde de qualité et détection de panne |
| Classe SLA | Seuils de perte, latence et gigue d'une politique SD-WAN |
| CoS | Class of Service, classes de service d'un réseau MPLS opérateur |
| DSCP | Marquage de priorité IP |
| GTI | Garantie de Temps d'Intervention |
| GTR | Garantie de Temps de Rétablissement |
| Gigue | Variation de la latence entre paquets |
| MTTR | Temps moyen de réparation |
| P95, P99 | Percentiles de mesure : 95 % ou 99 % des mesures sont en dessous de cette valeur |
| PE, CE | Routeur de bordure de l'opérateur, routeur de bordure du client |
| QoE | Quality of Experience, qualité perçue par l'utilisateur |
| RTO | Recovery Time Objective |
| SLA | Service Level Agreement, engagement contractuel |
| SLI | Service Level Indicator, mesure |
| SLO | Service Level Objective, objectif interne |

---

## 19. Références

- Documentation officielle Cisco SD-WAN : guides de configuration Policies (sections Application-Aware Routing et SLA classes) et Systems and Interfaces (BFD).
- Cisco SD-WAN Design Guide (Cisco Validated Design).
- UIT-T G.114 : temps de transmission unidirectionnel.
- UIT-T Y.1541 : objectifs de performance réseau pour les services IP.
- RFC 4594 : recommandations de configuration des classes de service DiffServ.
- RFC 5880 : Bidirectional Forwarding Detection (BFD).
- Cours associés : Cours complet SD-WAN et protocole OMP, Cours complet SD-WAN et protocole BFD.

Les valeurs par défaut, les limites de configuration, la syntaxe des politiques et les niveaux de service des contrats évoluent selon les versions et les opérateurs : validez-les dans la documentation correspondant à votre version et dans vos contrats avant toute mise en production.
