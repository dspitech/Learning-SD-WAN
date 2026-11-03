# La QoS (Qualité de Service) dans le SD-WAN

Niveau : étudiants en réseaux et télécoms
Support : schéma de cours "Classification, Marquage, Files d'attente, Shaping, Interface WAN"

---

## Table des matières

1. Objectifs d'apprentissage
2. Définition et raison d'être
3. Vue d'ensemble de la chaîne QoS
4. Étape 1 : la classification
5. Étape 2 : le marquage DSCP
6. Étape 3 : les files d'attente et l'ordonnancement
7. Étape 4 : le shaping
8. Lecture complète du schéma de cours
9. Pourquoi la QoS doit s'appliquer à la sortie WAN
10. Table DSCP d'exemple pour Carrefour
11. Scénario Carrefour
12. Mise en oeuvre Cisco et Fortinet
13. QoS, TLOC et sélection de chemin
14. Erreurs fréquentes
15. Questions de révision

---

## 1. Objectifs d'apprentissage

- Définir la QoS et expliquer pourquoi elle est nécessaire sur un lien WAN.
- Décrire les quatre étapes : classification, marquage, files d'attente, shaping.
- Distinguer policing et shaping.
- Lire une politique de files d'attente exprimée en pourcentages.
- Expliquer pourquoi la QoS ne crée pas de bande passante.

---

## 2. Définition et raison d'être

La QoS permet de donner la priorité aux applications importantes pour éviter les ralentissements du réseau.

Idée centrale : la QoS n'augmente pas la capacité d'un lien. Elle décide qui passe en premier et qui attend ou perd des paquets quand le lien est saturé. Sur un lien jamais saturé, elle ne change rien. Elle prend tout son sens aux heures de pointe.

Dans un magasin, un lien WAN à 20 Mbit/s partagé entre les caisses (quelques Mbit/s), la voix, les caméras (plusieurs Mbit/s) et une mise à jour logicielle (tout ce qu'elle trouve) sature vite. Sans QoS, la mise à jour peut étouffer les paiements.

---

## 3. Vue d'ensemble de la chaîne QoS

```
Classification --> Marquage --> Files d'attente --> Shaping --> Interface WAN
 classe applicative  DSCP interne   LLQ : VOICE            50 Mbit/s MPLS
 DSCP                + copie dans   VIDEO, TRANSACTIONAL   20 Mbit/s montant câble
                     l'en-tête      BUSINESS-DEFAULT, BULK
                     externe
```

---

## 4. Étape 1 : la classification

Classifier, c'est identifier le type de trafic. Critères possibles :

- adresses et ports (les caisses vers le serveur de ventes) ;
- DSCP déjà positionné par l'application ou un équipement amont ;
- reconnaissance d'application (DPI) : visioconférence, messagerie, sauvegarde ;
- segment ou VRF d'origine.

Exemple Carrefour : distinguer le trafic des caisses de celui des caméras.

---

## 5. Étape 2 : le marquage DSCP

Le DSCP (Differentiated Services Code Point) est un champ de 6 bits de l'en-tête IP. Il exprime la classe de service. Les équipements suivants lisent cette valeur pour appliquer la priorité.

Particularité du SD-WAN : comme le paquet d'origine est chiffré dans un tunnel IPsec, le DSCP interne n'est plus visible du réseau de transport. Le schéma de cours le précise : marquage DSCP interne et copie dans l'en-tête externe. Ainsi, l'opérateur (s'il propose de la QoS, c'est le cas du MPLS) peut lire le marquage sur l'enveloppe externe.

| Classe | Valeur DSCP | Usage typique |
|---|---|---|
| EF (Expedited Forwarding) | 46 | Voix, paiements critiques |
| AF31 | 26 | Applications transactionnelles |
| AF21 | 18 | Applications administratives |
| CS1 | 8 | Trafic de masse, faible priorité |

Rappel de la table de cours pour Carrefour : paiements bancaires (EF 46, priorité très élevée), caisses (AF31 26, élevée), applications administratives (AF21 18, moyenne), vidéosurveillance (CS1 8, normale). Ces valeurs sont données à titre d'exemple pédagogique et doivent être adaptées aux politiques réseau réelles.

---

## 6. Étape 3 : les files d'attente et l'ordonnancement

Une fois classé et marqué, chaque paquet est placé dans une file. L'ordonnanceur choisit quelle file sert ses paquets en premier.

### 6.1 Le schéma de cours

| File | Mode | Allocation | Rôle |
|---|---|---|---|
| VOICE | LLQ, priorité stricte | limitée à 5 Mbit/s par un policer | Voix : très sensible au délai |
| VIDEO | Garantie de bande passante | 30 % | Visioconférence |
| TRANSACTIONAL | Garantie de bande passante | 25 % | Caisses et paiements |
| BUSINESS-DEFAULT | Garantie de bande passante | 30 % | Applications métier courantes |
| BULK | Garantie minimale | 5 % (minimum) | Sauvegardes, mises à jour |

### 6.2 Comprendre LLQ

LLQ (Low Latency Queuing) est une file à priorité stricte : dès qu'un paquet y est présent, il est servi avant tout autre. Danger : si la voix déborde, elle affamerait les autres files. C'est pourquoi on lui associe un policer qui plafonne son débit (ici 5 Mbit/s). Au delà, les paquets en excès sont éliminés.

### 6.3 Comprendre les pourcentages

Pour le MPLS à 50 Mbit/s :

```
VOICE (LLQ, plafonnée)     5 Mbit/s    soit 10 %
VIDEO                      30 %       soit 15 Mbit/s
TRANSACTIONAL              25 %       soit 12,5 Mbit/s
BUSINESS-DEFAULT           30 %       soit 15 Mbit/s
BULK                        5 %       soit 2,5 Mbit/s
Total                     100 % = 50 Mbit/s
```

Ces pourcentages sont des garanties minimales en cas de congestion. Si une classe n'utilise pas sa part, les autres peuvent la récupérer. Ainsi BULK, avec 5 % garantis, peut utiliser presque tout le lien la nuit.

Remarque : sur le lien câble à 20 Mbit/s, un plafond de 5 Mbit/s pour la voix représente 25 % du lien. En pratique, on exprime souvent la politique en pourcentage pour qu'elle s'adapte à chaque lien.

---

## 7. Étape 4 : le shaping

Le shaping (mise en forme du trafic) limite le débit de sortie à une valeur choisie, en retardant les paquets excédentaires dans une file plutôt qu'en les jetant.

Pourquoi shaper ?

L'opérateur vend un débit contractuel (50 Mbit/s MPLS, 20 Mbit/s montant câble). Si l'Edge émet plus vite, c'est l'équipement de l'opérateur qui jette des paquets, sans tenir compte de vos priorités. En shapant à la valeur du contrat (ou un peu en dessous), c'est votre Edge qui forme la file et applique vos priorités. La QoS reprend le contrôle de ce qui est jeté.

### Policing et shaping

| | Policing | Shaping |
|---|---|---|
| Réaction à l'excès | Jette ou re-marque | Retarde dans une file |
| Effet sur le trafic | Rafales dures, pertes | Lissage |
| Usage dans le schéma | Plafond de la file voix (5 Mbit/s) | Débit de sortie de l'interface WAN |

---

## 8. Lecture complète du schéma de cours

1. Les paquets arrivent et sont classés par classe applicative ou DSCP.
2. Le DSCP interne est positionné et copié dans l'en-tête externe du tunnel.
3. Chaque paquet entre dans sa file. La voix en priorité stricte (plafonnée), les autres selon leur part.
4. Le shaping limite le débit global : 50 Mbit/s sur le MPLS, 20 Mbit/s en montant sur le câble.
5. Les paquets sortent par l'interface WAN.

---

## 9. Pourquoi la QoS doit s'appliquer à la sortie WAN

Le goulot d'étranglement se trouve à la sortie vers le WAN, là où le débit chute, par exemple de 1 Gbit/s côté LAN à 20 Mbit/s côté WAN. La QoS doit donc être appliquée sur l'interface de sortie, dans le sens où le trafic quitte le site. Elle est aussi nécessaire au datacenter pour le sens retour.

---

## 10. Table DSCP d'exemple pour Carrefour

| Application | Priorité | DSCP |
|---|---|---|
| Paiements bancaires | Très élevée | EF (46) |
| Caisses | Élevée | AF31 (26) |
| Applications administratives | Moyenne | AF21 (18) |
| Vidéosurveillance | Normale | CS1 (8) |

Valeurs proposées à titre d'exemple pédagogique. Dans un vrai déploiement, la voix serait aussi en EF, et l'on distinguerait paiements et voix par des files différentes ou des plafonds différents.

---

## 11. Scénario Carrefour

Un magasin Carrefour utilise le réseau SD-WAN pour connecter ses caisses, ses terminaux de paiement et ses caméras de surveillance au datacenter central. Pendant les heures de forte affluence, le réseau peut être saturé. Le SD-WAN donne alors la priorité aux caisses et aux paiements pour éviter les ralentissements des transactions. En revanche, il limite le débit des vidéos de surveillance, car elles sont moins prioritaires. Ainsi, les clients peuvent continuer à payer et les caisses restent opérationnelles malgré la saturation.

Chronologie d'une saturation :

```
17h30  Affluence : 300 transactions par minute, caméras en plein débit
17h31  Le lien WAN est saturé
17h31  Les files se remplissent. L'ordonnanceur sert TRANSACTIONAL avant BULK
17h31  Les paquets des caméras sont retardés ou jetés selon leur part
17h32  Les paiements restent rapides, la vidéo est dégradée (acceptable)
```

---

## 12. Mise en oeuvre Cisco et Fortinet

### Cisco (principe)

On définit des planificateurs (schedulers) par file, une table de correspondance DSCP vers file, et on applique le tout à l'interface avec un débit de shaping.

```
policy
 qos-scheduler Q0
  class Queue0
  bandwidth-percent 10
  buffer-percent 10
  scheduling llq
  drops tail-drop
 !
 qos-scheduler Q1
  class Queue1
  bandwidth-percent 25
  buffer-percent 25
  scheduling wrr
  drops red-drop
 !
 qos-map QOS_MAGASIN
  qos-scheduler Q0
  qos-scheduler Q1
 !
vpn 0
 interface ge0/0
  qos-map QOS_MAGASIN
  shaping-rate 50000
```

La syntaxe exacte varie selon la version. Le point à retenir : un scheduler en llq pour la voix, des files pondérées pour le reste, et un shaping-rate (en kbit/s) sur l'interface de transport.

### Fortinet (principe)

```
config firewall shaping-profile
    edit "PROFIL_MAGASIN"
        set default-class-id 30
        config shaping-entries
            edit 1
                set class-id 10
                set priority high
                set guaranteed-bandwidth-percentage 20
                set maximum-bandwidth-percentage 100
            next
            edit 2
                set class-id 20
                set priority medium
                set guaranteed-bandwidth-percentage 25
                set maximum-bandwidth-percentage 100
            next
        end
    next
end

config system interface
    edit "wan1"
        set outbandwidth 50000
        set egress-shaping-profile "PROFIL_MAGASIN"
    next
end
```

Ici, outbandwidth fixe le débit de shaping de l'interface, et le profil répartit la bande passante entre les classes.

---

## 13. QoS, TLOC et sélection de chemin

La QoS et les TLOC sont complémentaires :

- la QoS gère les priorités du trafic (DSCP, files, shaping) sur chaque lien ;
- les TLOC représentent les différents chemins de transport disponibles ;
- la sélection de chemin (SLA, BFD) choisit le lien, puis la QoS organise le passage sur ce lien.

```
Application --> classification --> DSCP
                                     |
                 sélection de chemin (SLA + BFD)  : quel TLOC ?
                                     |
                      QoS sur le lien choisi      : quelle file, quel débit ?
```

Une politique cohérente combine les deux : l'application critique est dirigée vers le chemin le plus sain, puis protégée par sa file prioritaire. Sans QoS, un bon chemin peut quand même être saturé. Sans sélection de chemin, une bonne file ne protège pas contre un lien dégradé.

---

## 14. Erreurs fréquentes

1. Croire que la QoS crée de la bande passante : elle ne fait que répartir.
2. Mettre trop de trafic en priorité stricte : on retombe sur un réseau sans priorité et on affame les autres classes.
3. Ne pas shaper à la valeur du contrat : l'opérateur jette alors vos paquets sans respecter vos priorités.
4. Oublier de copier le DSCP sur l'en-tête externe : le marquage devient invisible pour l'opérateur.
5. Faire confiance au DSCP venant des postes utilisateurs : un utilisateur peut marquer son trafic en EF. On re-marque en entrée de site.
6. Appliquer la QoS dans un seul sens : la congestion existe aussi au retour.

---

## 15. Questions de révision

1. Quelles sont les étapes principales de la QoS ?
   Réponse : classification, marquage DSCP, files d'attente avec priorités, shaping.

2. Pourquoi plafonne-t-on la file LLQ de la voix ?
   Réponse : pour qu'une rafale de voix ne prive pas les autres files de bande passante.

3. Quelle est la différence entre policing et shaping ?
   Réponse : le policing jette ou re-marque l'excès, le shaping le retarde pour lisser le débit.

4. Pourquoi copier le DSCP dans l'en-tête externe ?
   Réponse : parce que le paquet interne est chiffré par IPsec et illisible par le réseau de transport.

5. Sur un lien de 50 Mbit/s avec VIDEO à 30 %, combien de Mbit/s sont garantis à VIDEO ?
   Réponse : 15 Mbit/s en cas de congestion.
