# Cours complet SD-WAN : blackout et brownout

Ce cours fait suite aux cours sur OMP, BFD et SLA. Il traite les deux grandes familles de pannes de transport que rencontre un réseau SD-WAN :

- Le blackout : une coupure franche, le chemin ne transporte plus rien.
- Le brownout : une dégradation partielle, le chemin fonctionne mais mal (perte, latence, gigue).

La distinction est capitale, car ces deux pannes ne se détectent pas avec les mêmes mécanismes, ne se traitent pas avec les mêmes outils et n'ont pas le même coût opérationnel. Le fil rouge reste le réseau d'une enseigne de grande distribution de type Carrefour.

> Note importante : le cas Carrefour est un scénario pédagogique fictif. Les plans d'adressage, les incidents, les durées et les volumes sont inventés à des fins d'illustration et ne décrivent ni l'architecture réelle de l'enseigne ni ses incidents réels.

> Note de vocabulaire : Cisco a rebaptisé sa solution Catalyst SD-WAN (vManage devient SD-WAN Manager, vSmart devient SD-WAN Controller, vBond devient SD-WAN Validator). Ce cours conserve les noms historiques, plus répandus.

---

## Table des matières

1. Objectifs et prérequis
2. Définitions : blackout, brownout et pannes grises
3. Causes typiques
4. Détection du blackout : BFD et le plan de données
5. Détection du brownout : mesures de qualité et classes SLA
6. Pourquoi un brownout échappe à la détection de panne
7. Les mécanismes de réaction
8. Correction de perte : FEC et duplication de paquets
9. Stabilité : éviter les oscillations et gérer le retour arrière
10. Blackout et brownout du plan de contrôle
11. Cas d'étude fil rouge : le réseau d'une enseigne de distribution
12. Configuration pas à pas
13. Supervision et détection complémentaire
14. Vérification et dépannage
15. Tests de résilience en laboratoire
16. Pièges classiques
17. Bonnes pratiques
18. Travaux pratiques
19. Questionnaire d'auto-évaluation
20. Glossaire
21. Références

---

## 1. Objectifs et prérequis

### Objectifs pédagogiques

À la fin de ce cours, vous serez capable de :

- Définir blackout, brownout et panne grise, et les distinguer en situation réelle.
- Expliquer pourquoi BFD détecte le blackout mais laisse passer de nombreux brownouts.
- Calculer les délais de détection d'un blackout et d'un brownout selon les timers.
- Choisir le bon mécanisme de réaction : bascule, routage applicatif, correction de perte.
- Configurer FEC et duplication de paquets, et en mesurer le coût.
- Éviter les oscillations de chemin et organiser le retour arrière.
- Reproduire ces pannes en laboratoire et construire un guide d'exploitation.

### Prérequis

- Les cours sur OMP, BFD et SLA (TLOC, color, sessions BFD, classes SLA, AAR).
- Notions de QoS et de TCP (retransmission, timeout).

---

## 2. Définitions : blackout, brownout et pannes grises

### 2.1 Origine des termes

Ces mots viennent du monde de l'électricité :

- Blackout : coupure totale du courant.
- Brownout : baisse de tension, les appareils fonctionnent mal sans être éteints.

Les réseaux ont repris la métaphore.

### 2.2 Définitions réseau

| Terme | Définition | Exemple |
|---|---|---|
| Blackout | Le chemin ne transporte plus aucun trafic utile | Coupure de fibre, routeur opérateur en panne, SIM suspendue |
| Brownout | Le chemin reste ouvert mais sa qualité est dégradée au point de gêner les applications | Congestion du soir, perte de paquets de 3 %, latence multipliée par 5 |
| Panne grise (gray failure) | Panne partielle ou sélective, que certains observateurs voient et d'autres non | Un seul sens du lien défaillant, certains ports ou tailles de paquets bloqués, trou noir de MTU |

### 2.3 Tableau comparatif

| | Blackout | Brownout |
|---|---|---|
| État du lien | Down | Up, mais dégradé |
| Symptôme | Tout échoue | Lenteurs, coupures intermittentes, qualité de voix dégradée |
| Détection | Rapide et binaire | Plus lente, fondée sur des seuils |
| Mécanisme principal | BFD (liveness) | Mesures BFD agrégées, classes SLA, AAR |
| Délai de détection typique | Secondes (hello x multiplicateur) | Dizaines de secondes à minutes (poll interval x multiplicateur) |
| Réaction | Bascule sur un autre tunnel | Bascule sélective par application, ou correction de perte |
| Difficulté d'exploitation | Faible, la panne est évidente | Élevée, la panne est ambiguë et difficile à prouver |
| Fréquence | Plus rare | Plus fréquente |

Idée à retenir : en exploitation, les brownouts sont souvent plus coûteux que les blackouts, car ils sont plus fréquents, plus longs à détecter et plus difficiles à attribuer à une cause.

---

## 3. Causes typiques

### 3.1 Causes de blackout

- Coupure physique de la fibre ou du câble (travaux de voirie).
- Panne d'alimentation électrique du local technique ou du routeur.
- Panne matérielle du WAN Edge ou de l'équipement de l'opérateur.
- Incident de routage chez l'opérateur (erreur de configuration, fuite ou détournement de préfixes BGP).
- Attaque par déni de service saturant totalement l'accès.
- Lien cellulaire : panne d'antenne, SIM suspendue, forfait de données épuisé.
- Pare-feu ou opérateur qui bloque soudainement UDP 12346 (port des tunnels et des connexions de contrôle).
- Certificat expiré, qui coupe les connexions de contrôle (blackout du plan de contrôle).

### 3.2 Causes de brownout

- Congestion de l'accès ou du coeur de réseau (pic du soir, peering saturé).
- Bufferbloat : files d'attente trop longues qui gonflent la latence.
- Qualité de signal fluctuante sur 4G et 5G ou sur une liaison radio.
- Erreurs physiques (module optique vieillissant, câble abîmé, erreurs CRC).
- Désaccord de duplex ou de vitesse sur une interface (cause classique et sournoise).
- QoS mal configurée, shaping absent ou mal calibré.
- Saturation CPU du routeur WAN Edge, notamment en pic d'activité.
- Fragmentation et problèmes de MTU.
- Attaque partielle de déni de service.
- Chemin de routage de l'opérateur plus long que d'habitude après un incident en amont.

### 3.3 Causes de pannes grises

- Panne unidirectionnelle : un sens passe, l'autre non.
- Trou noir de MTU : les petits paquets passent, les gros disparaissent.
- Filtrage sélectif d'un protocole ou d'un port.
- Panne d'un seul des chemins ECMP de l'opérateur.
- Mauvais fonctionnement d'un équipement qui répond aux paquets de test mais perd les paquets de données.

---

## 4. Détection du blackout : BFD et le plan de données

### 4.1 Rappel

Chaque tunnel IPsec porte une session BFD qui émet un hello à intervalle régulier. Si aucun paquet n'est reçu pendant hello x multiplicateur, la session passe à down.

### 4.2 Calcul du délai de détection

```
Délai de détection du blackout = hello-interval x multiplier
```

| Réglage | Délai |
|---|---|
| Hello 1000 ms, multiplicateur 7 (valeurs usuelles par défaut) | 7 s |
| Hello 500 ms, multiplicateur 4 | 2 s |
| Hello 200 ms, multiplicateur 3 | 0,6 s |
| Hello 10 000 ms, multiplicateur 7 (réglage allégé pour la 4G) | 70 s |

### 4.3 Conséquences d'une session BFD down

1. Le tunnel est considéré inutilisable.
2. Le TLOC associé n'est plus résolu pour ce chemin.
3. Les routes OMP qui s'appuient sur ce TLOC ne sont plus installées.
4. Le trafic est acheminé sur les chemins restants (ECMP ou chemin de secours).
5. La décision est locale au WAN Edge, sans intervention du contrôleur. Elle est donc très rapide.

### 4.4 Chronologie d'un blackout (exemple avec hello 1 s et multiplicateur 7)

| Instant | Événement |
|---|---|
| t = 0 s | La fibre est coupée |
| t = 0 à 7 s | Les paquets envoyés sur ce tunnel sont perdus, TCP retransmet |
| t = 7 s | Les sessions BFD de ce lien passent à down des deux côtés |
| t = 7 s environ | Le trafic emprunte un autre tunnel |
| t = 8 à 10 s | Les nouvelles transactions aboutissent, les connexions applicatives les plus sensibles ont pu expirer |

Une coupure de 7 secondes est invisible pour un téléchargement mais peut faire échouer une transaction de paiement. Le réglage des timers est donc une décision de niveau de service (voir le cours sur le SLA).

### 4.5 Coût d'une détection plus rapide

- Davantage de paquets BFD et donc de bande passante consommée.
- Plus de charge CPU, surtout sur les hubs qui portent des milliers de sessions.
- Plus de risque de faux positifs : un simple incident de quelques centaines de millisecondes sur un lien instable provoque un down.

---

## 5. Détection du brownout : mesures de qualité et classes SLA

### 5.1 Principe

BFD ne sert pas seulement à détecter la panne : il mesure aussi la perte, la latence et la gigue de chaque tunnel. Ces mesures sont agrégées sur des fenêtres :

- `app-route poll-interval` : durée d'une fenêtre (par défaut 600000 ms).
- `app-route multiplier` : nombre de fenêtres retenues (par défaut 6).

Elles sont comparées aux seuils des classes SLA. Si un tunnel dépasse un seuil, il n'est plus conforme à cette classe et l'AAR dirige les flux concernés vers un tunnel conforme.

### 5.2 Délai de détection du brownout

L'ordre de grandeur est :

```
Délai de détection du brownout = fenêtre d'évaluation, de l'ordre de poll-interval x multiplier
```

Selon le mécanisme de moyenne propre à la version, la réaction peut être plus rapide que ce produit, mais il donne l'échelle. Avec les valeurs par défaut, elle se compte en dizaines de minutes. Avec un poll interval de 60 s et un multiplicateur de 5, en dizaines de secondes à quelques minutes.

### 5.3 Tableau des échelles de temps

| Situation | Mécanisme | Échelle de temps |
|---|---|---|
| Coupure franche | BFD liveness | Secondes |
| Dégradation de qualité | Mesures agrégées et classes SLA | Dizaines de secondes à dizaines de minutes |
| Dégradation brève (microcoupure, pic de latence de quelques secondes) | Rien, sauf correction de perte | Non détectée par l'AAR |

Conséquence : l'AAR n'est pas un mécanisme de temps réel. Pour les dégradations de quelques secondes, il faut un mécanisme complémentaire : la correction de perte (FEC, duplication) et une bonne QoS.

---

## 6. Pourquoi un brownout échappe à la détection de panne

### 6.1 Le calcul qui le prouve

Le BFD déclare un lien down quand il manque plusieurs hello consécutifs. Avec un multiplicateur de 7, il faut 7 pertes de suite. Pour une perte aléatoire de probabilité p, la probabilité que 7 paquets consécutifs soient perdus est p à la puissance 7.

| Perte aléatoire | Probabilité de 7 pertes consécutives |
|---|---|
| 1 % | 10 puissance moins 14 (négligeable) |
| 10 % | 10 puissance moins 7 (1 chance sur 10 millions) |
| 30 % | environ 0,0002 (soit 0,02 %) |
| 50 % | environ 0,008 (soit 0,8 %) |

Un lien qui perd 30 % de ses paquets reste donc presque toujours déclaré up par BFD, alors qu'il est inutilisable pour une application. TCP, avec 30 % de perte, s'effondre quasiment.

### 6.2 Conséquences

- Un brownout important ne déclenche aucune bascule de liveness.
- Seule la comparaison des mesures aux seuils SLA permet de le détecter.
- Sans classe SLA bien réglée, le SD-WAN se comporte comme un réseau classique face à ce type de panne : il ne bascule pas.

### 6.3 Pannes grises et BFD

| Panne grise | Détectée par BFD ? |
|---|---|
| Panne unidirectionnelle | Oui, généralement, car la session est bidirectionnelle et l'autre côté cesse de recevoir |
| Trou noir de MTU | Non si les paquets BFD sont petits : le tunnel reste up alors que les gros paquets sont perdus |
| Filtrage d'un port applicatif | Non |
| Perte sélective sur un chemin ECMP de l'opérateur | Partiellement, selon que les paquets BFD empruntent ce chemin |

D'où l'intérêt de la découverte du MTU de chemin (`pmtu-discovery`) et d'une supervision applicative complémentaire.

---

## 7. Les mécanismes de réaction

### 7.1 Panorama

| Mécanisme | Réagit à | Rapidité | Effet |
|---|---|---|---|
| Bascule BFD (liveness) | Blackout | Secondes | Retrait du tunnel, trafic sur les chemins restants |
| AAR avec classes SLA | Brownout durable | Dizaines de secondes à minutes | Déplacement des flux vers un tunnel conforme |
| FEC | Perte de paquets | Immédiat (paquet par paquet) | Reconstruction des paquets perdus |
| Duplication de paquets | Perte ou microcoupure | Immédiat | Envoi de chaque paquet sur deux tunnels, le premier arrivé est retenu |
| QoS et shaping | Congestion locale | Immédiat | Priorité des flux critiques, limitation du débit au débit souscrit |
| Optimisation TCP | Latence et perte sur de longues distances | Immédiat | Meilleure tenue des flux TCP |

### 7.2 Arbre de décision simplifié

1. Le lien est-il totalement coupé ? Si oui, le blackout est géré par BFD et la bascule.
2. Le lien est-il dégradé de façon durable ? Si oui, l'AAR avec classe SLA déplace les flux.
3. La dégradation est-elle brève ou intermittente ? Si oui, utiliser la correction de perte (FEC ou duplication) sur les flux sensibles.
4. La dégradation vient-elle de notre propre site (congestion locale, duplex) ? Si oui, corriger à la source : QoS, shaping, câblage, interface.
5. Aucun chemin n'est conforme ? Utiliser le repli sur le meilleur tunnel, jamais le mode strict pour un flux critique.

---

## 8. Correction de perte : FEC et duplication de paquets

### 8.1 Forward Error Correction (FEC)

Le FEC ajoute des paquets de redondance (de parité) à un groupe de paquets de données. Si l'un des paquets du groupe est perdu, le récepteur le reconstruit sans retransmission.

| Mode | Comportement |
|---|---|
| FEC toujours actif (`fec-always`) | Redondance en permanence, surcoût de bande passante constant |
| FEC adaptatif (`fec-adaptive`) | La redondance s'active seulement quand la perte mesurée dépasse un seuil |

Avantages : surcoût limité, adapté à la voix et à la vidéo en temps réel, protège contre les pertes isolées.
Limites : ne protège pas contre une perte en rafale importante ni contre une coupure.

### 8.2 Duplication de paquets

Chaque paquet est envoyé simultanément sur deux tunnels (idéalement de couleurs différentes). Le récepteur conserve la première copie reçue et supprime la seconde.

Avantages : protection forte contre les pertes et les microcoupures d'un des deux chemins, transparente pour l'application.
Limites : double la bande passante consommée pour le flux protégé, nécessite deux tunnels disponibles, et ne s'applique qu'aux flux sélectionnés.

### 8.3 Choix selon le type de flux

| Flux | Débit | Sensibilité | Protection conseillée |
|---|---|---|---|
| Transactions de paiement | Très faible | Très forte à la perte et aux coupures | Duplication de paquets (le surcoût est négligeable) |
| Voix et visioconférence | Faible à moyen | Forte à la perte et à la gigue | FEC adaptatif, éventuellement duplication |
| Applications métier interactives | Moyen | Moyenne | FEC adaptatif ou AAR seul |
| Vidéosurveillance, sauvegardes | Élevé | Faible | Aucune, un surcoût serait injustifié |

Point clé pour la distribution : les flux de paiement sont très légers. Dupliquer chaque paquet d'une transaction ne coûte pratiquement rien en bande passante, alors que cela réduit fortement le risque d'échec pendant un brownout ou une microcoupure.

### 8.4 Coût et précautions

- Éviter la duplication vers un lien cellulaire facturé au volume sauf pour des flux très légers.
- Vérifier que les deux tunnels utilisés pour la duplication ne partagent pas la même cause de panne (même opérateur, même boucle locale).
- Vérifier la compatibilité de la plateforme et de la version logicielle avec les fonctions de correction de perte.
- Évaluer l'impact CPU sur les routeurs de bordure.

---

## 9. Stabilité : éviter les oscillations et gérer le retour arrière

### 9.1 Le problème des oscillations

Un lien instable qui passe alternativement conforme et non conforme provoque des bascules répétées, appelées flaps. Chaque bascule perturbe les flux et peut produire un trafic désordonné.

### 9.2 Leviers de stabilisation

- Moyenne sur plusieurs fenêtres de mesure (multiplicateur de app-route) : une dégradation brève ne déclenche pas de bascule.
- Variances de `fallback-best-tunnel` : on ne change de tunnel que si l'écart est significatif.
- Seuils SLA fixés avec une marge par rapport aux valeurs normales.
- Couleur préférée explicite : elle évite les choix alternés entre deux tunnels équivalents.
- Surveillance du nombre de transitions BFD, avec alarme au-delà d'un seuil.

### 9.3 Retour arrière (failback)

Quand le lien d'origine est rétabli, deux stratégies existent :

| Stratégie | Avantage | Inconvénient |
|---|---|---|
| Retour automatique sur la couleur préférée | Retrouve le chemin nominal, libère le lien de secours | Risque de retour sur un lien encore fragile |
| Rester sur le lien de secours tant qu'il est conforme | Stabilité | Peut maintenir longtemps le trafic sur un lien coûteux |

Une combinaison courante : retour automatique, mais avec des mesures lissées sur plusieurs fenêtres, ce qui impose une période de stabilité avant le retour.

### 9.4 Cas particulier : lien cellulaire de secours

Le retour vers la fibre doit être rapide pour libérer le forfait de données, mais pas au prix d'oscillations. Une alarme sur la durée de séjour sur le lien de secours est indispensable.

---

## 10. Blackout et brownout du plan de contrôle

Le plan de contrôle (connexions DTLS ou TLS vers vSmart, vBond, vManage, sessions OMP) peut aussi subir ces deux types de pannes.

### 10.1 Blackout du plan de contrôle

| Panne | Effet | Protection |
|---|---|---|
| Perte d'un vSmart | Le WAN Edge garde sa session avec l'autre vSmart | Au moins deux vSmart, répartis géographiquement |
| Perte de tous les vSmart | Plus de mise à jour de routes ni de clés | Graceful restart d'OMP (routes et clés conservées, 43200 s par défaut) : le trafic existant continue |
| Perte de vManage | Plus de supervision ni de modification de configuration, le plan de données n'est pas touché | Cluster vManage, sauvegardes |
| Certificats expirés | Les connexions de contrôle tombent | Surveillance des échéances de certificats, renouvellement anticipé |

À la fin du graceful restart, si le contrôle n'est pas revenu, le plan de données finit par se dégrader (routes retirées, clés non renouvelées). Il faut donc considérer le graceful restart comme un répit, pas comme une solution.

### 10.2 Brownout du plan de contrôle

Une connexion de contrôle dégradée (perte, latence) peut provoquer des flaps de sessions OMP. Les symptômes : routes qui disparaissent et réapparaissent, mises à jour de politiques lentes, alarmes de connexion de contrôle. Les remèdes : diversifier les chemins vers les contrôleurs (plusieurs couleurs de transport autorisées à porter les connexions de contrôle), dimensionner les contrôleurs et placer au moins un vSmart accessible par un chemin indépendant.

### 10.3 Lien avec le plan de données

Une panne du plan de contrôle n'est pas une panne du plan de données. Dans un réseau bien conçu, les magasins continuent d'encaisser pendant la panne d'un contrôleur : l'interruption concerne les modifications, pas l'exploitation courante.

---

## 11. Cas d'étude fil rouge : le réseau d'une enseigne de distribution

> Rappel : scénario fictif à vocation pédagogique.

### 11.1 Rappel du contexte

- Deux datacenters (Paris, site-id 1000 et Lyon, site-id 1001).
- Environ 1000 magasins : hypermarchés (MPLS + 2 fibres), supermarchés (MPLS + fibre), proximité (fibre + 4G ou 5G).
- VPN 10 caisses, VPN 20 back office, VPN 30 vidéosurveillance, VPN 40 invités, VPN 50 IoT.
- Hub and spoke, BFD sur chaque tunnel, classes SLA et AAR en place.

### 11.2 Scénario 1 : blackout d'un magasin de proximité

Un chantier de voirie coupe la fibre d'un magasin de proximité à 10 h 12.

1. Les sessions BFD sur la couleur biz-internet tombent après 7 secondes (réglage par défaut) ou 2 secondes (réglage resserré pour les caisses).
2. Le magasin bascule sur la couleur lte.
3. Les transactions en cours peuvent échouer pendant la fenêtre de détection, les suivantes passent.
4. Une alarme prévient le centre d'exploitation que le site fonctionne sur son lien de secours.
5. Les flux volumineux (sauvegardes, mises à jour) ne sont pas autorisés sur lte et attendent.
6. Au rétablissement de la fibre, le trafic revient sur la couleur préférée après une période de stabilité.

Enseignement : le temps de détection (2 à 7 s) est le principal levier pour réduire l'impact. Des timers resserrés se justifient pour les caisses, à condition de valider la stabilité.

### 11.3 Scénario 2 : brownout du soir sur une liaison ADSL ou fibre mutualisée

Chaque soir à 19 h, le lien d'un magasin affiche 3 % de perte et 120 ms de latence, à cause de la congestion chez l'opérateur. Aucun BFD ne tombe.

1. La mesure dépasse les seuils de SLA-CAISSES (1 %, 100 ms, 30 ms).
2. Après la fenêtre d'évaluation, les flux des caisses sont déplacés sur un tunnel conforme (le MPLS d'un supermarché, la 4G d'une proximité).
3. Pendant la fenêtre d'évaluation, les caisses subissent la dégradation : c'est là que la duplication de paquets est utile, car elle protège dès le premier paquet.
4. Sans classe SLA, la panne aurait été invisible pour le SD-WAN et seuls les caissiers l'auraient signalée.

### 11.4 Scénario 3 : désaccord de duplex après un changement de commutateur

Après le remplacement d'un commutateur dans un supermarché, une interface WAN négocie un duplex incorrect. Le lien reste up avec 0,5 à 2 % d'erreurs et des retransmissions importantes.

- BFD ne tombe pas.
- Les mesures de perte approchent le seuil SLA, avec des oscillations de conformité.
- Le diagnostic nécessite l'examen des compteurs d'erreurs de l'interface (CRC, collisions), pas des mesures SD-WAN seules.
- Leçon : un brownout peut avoir une cause locale. La supervision des interfaces fait partie de la détection.

### 11.5 Scénario 4 : saturation CPU d'un hub pendant un pic d'activité

Un jour de forte affluence (période de promotions, fêtes de fin d'année), le routeur hub de Paris traite un volume exceptionnel de transactions et de sessions.

- Le CPU du hub atteint un niveau élevé, les paquets BFD sont traités avec retard, la gigue augmente.
- Les magasins mesurent une dégradation vers Paris et certains basculent vers Lyon, ce qui surcharge à son tour Lyon.
- Leçon : un brownout peut se propager (effet domino). Il faut dimensionner les hubs pour le pic, répartir les magasins sur plusieurs hubs, prévoir une marge de capacité sur le datacenter de secours et surveiller le CPU et le nombre de sessions BFD.

### 11.6 Scénario 5 : cause commune dans un hypermarché

Les deux fibres d'un hypermarché, souscrites auprès de deux opérateurs distincts, empruntent le même fourreau et sont coupées ensemble par un engin de chantier. Il reste le MPLS (si son tracé est différent) et, selon l'équipement, une 4G de secours.

- Leçon : deux opérateurs ne signifient pas deux chemins physiques. La diversité des tracés se vérifie contractuellement et sur le terrain.
- Un hypermarché doit disposer d'au moins un accès réellement indépendant (cellulaire ou autre technologie) pour le scénario où tous les accès filaires tombent.

### 11.7 Scénario 6 : blackout par forfait de données épuisé

Un magasin de proximité fonctionne plusieurs jours sur sa 4G après un incident de fibre prolongé. Le forfait est épuisé, l'opérateur suspend ou bride la ligne. Le dernier lien disparaît.

- Leçon : prévoir des alarmes de consommation, un forfait dimensionné pour un fonctionnement prolongé sur le secours, et une procédure d'escalade quand un site opère trop longtemps sur son lien de secours.

### 11.8 Scénario 7 : blackout du plan de contrôle par certificat expiré

Les certificats d'un lot de routeurs expirent un dimanche. Leurs connexions de contrôle tombent, les sessions OMP aussi.

- Le plan de données continue pendant la durée du graceful restart, les magasins ouvrent normalement.
- Au-delà, le trafic se dégrade.
- Leçon : suivre les dates d'expiration des certificats, planifier les renouvellements bien avant l'échéance et superviser l'état des connexions de contrôle.

### 11.9 Synthèse des politiques de protection par type de flux

| Flux | BFD (blackout) | Classe SLA (brownout durable) | Correction de perte (brownout bref) |
|---|---|---|---|
| Caisses (VPN 10) | Hello 500 ms, multiplicateur 4 sur fibre et MPLS | SLA-CAISSES avec repli sur le meilleur tunnel | Duplication de paquets |
| Voix et visio | Idem | SLA-VOIX-VISIO | FEC adaptatif |
| Back office, ERP | Valeurs par défaut | SLA-BUREAUTIQUE | Aucune |
| Vidéosurveillance, sauvegardes | Valeurs par défaut | Classe souple, hors lte | Aucune |
| Invités (VPN 40) | Valeurs par défaut | Aucune | Aucune |

### 11.10 Indicateurs de pilotage (exemple fictif sur un mois)

| Indicateur | Valeur |
|---|---|
| Blackouts de transport (sessions BFD tombées plus de 30 s) | 187 |
| Brownouts (violations SLA persistantes) | 1 450 |
| Durée moyenne de détection d'un blackout | 3,1 s |
| Durée moyenne de détection d'un brownout | 4 min 20 s |
| Magasins ayant fonctionné plus de 24 h sur lien de secours | 9 |
| Transactions échouées imputables à un incident de transport | 0,004 % |

---

## 12. Configuration pas à pas

### 12.1 Timers BFD pour la détection de blackout

```
sdwan
 bfd color mpls
  hello-interval 500
  multiplier 4
  pmtu-discovery
 exit
 bfd color biz-internet
  hello-interval 500
  multiplier 4
  pmtu-discovery
 exit
 bfd color lte
  hello-interval 10000
  multiplier 7
  no pmtu-discovery
 exit
!
```

Hello de 500 ms et multiplicateur de 4 donnent une détection d'environ 2 secondes. Validez la stabilité sur un site pilote avant déploiement général. Les bornes autorisées dépendent de la version et de la plateforme.

### 12.2 Fenêtre de mesure pour la détection de brownout

```
sdwan
 bfd app-route multiplier 5
 bfd app-route poll-interval 60000
!
```

### 12.3 Classes SLA avec repli

```
policy
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
 sla-class SLA-VOIX-VISIO
  loss    1
  latency 150
  jitter  30
 !
!
```

### 12.4 Correction de perte sur les flux sensibles

L'activation du FEC et de la duplication de paquets se définit dans une politique de données, appliquée aux flux sélectionnés. La syntaxe ci-dessous est indicative : dans vManage, l'action s'appelle correction de perte (Loss Correction) et propose FEC adaptatif, FEC toujours actif et duplication de paquets. La disponibilité dépend de la plateforme et de la version.

```
policy
 lists
  vpn-list VPN-CAISSES
   vpn 10
  !
  vpn-list VPN-VOIX
   vpn 20
  !
 !
 data-policy PROTECTION-PERTE
  vpn-list VPN-CAISSES
   sequence 10
    match
     dscp 46
    !
    action accept
     loss-protect pkt-dup
    !
   !
   default-action accept
  !
  vpn-list VPN-VOIX
   sequence 10
    match
     dscp 34
    !
    action accept
     loss-protect fec-adaptive
    !
   !
   default-action accept
  !
 !
!
```

### 12.5 Politique de routage applicatif associée

```
policy
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
  !
 !
!
```

### 12.5.1 À ne pas faire

Ne pas utiliser le mot clé `strict` pour les flux de caisses : en l'absence de tunnel conforme, les caisses seraient coupées.

---

## 13. Supervision et détection complémentaire

### 13.1 Pourquoi compléter la mesure SD-WAN

La mesure BFD ne voit que les paquets BFD. Une supervision complémentaire permet de détecter les pannes grises, les problèmes applicatifs et les dégradations de bout en bout vers le cloud.

### 13.2 Sources de détection

| Source | Ce qu'elle détecte |
|---|---|
| Sessions et historique BFD | Blackouts, flaps |
| Mesures app-route (perte, latence, gigue) | Brownouts durables |
| Compteurs d'interface (erreurs, CRC, drops, duplex) | Causes locales de brownout |
| Utilisation CPU et mémoire du WAN Edge | Saturation du routeur |
| Sondes applicatives (HTTP, DNS, tests synthétiques) | Pannes grises et problèmes applicatifs |
| Mesure de qualité vers les applications SaaS (par exemple les fonctions de type Cloud OnRamp for SaaS) | Dégradation du chemin vers un service cloud |
| Outils de visibilité internet, selon l'intégration (par exemple ThousandEyes avec Catalyst SD-WAN) | Incidents chez les opérateurs et sur internet |
| Métriques métier (taux d'échec des transactions, temps de réponse des terminaux de paiement) | Impact réel sur le métier |

### 13.3 Alarmes recommandées

- Session BFD down plus de N secondes.
- Nombre de transitions BFD au-delà d'un seuil par heure.
- Violation persistante d'une classe SLA sur une couleur.
- Séjour prolongé d'un site sur son lien de secours.
- Consommation de données d'un lien cellulaire au-delà d'un pourcentage du forfait.
- CPU d'un hub au-delà d'un seuil.
- Expiration prochaine de certificats.
- Perte de connexion de contrôle vers un contrôleur.

### 13.4 Indicateurs de pilotage

| Indicateur | Intérêt |
|---|---|
| MTTD (délai moyen de détection) | Efficacité de la supervision |
| MTTR (délai moyen de rétablissement) | Efficacité des procédures |
| Nombre de blackouts et de brownouts par opérateur | Objectiver les relations avec les opérateurs |
| Temps passé en brownout par site | Identifier les sites à problème chronique |
| Taux de réussite des bascules | Valider la conception |

---

## 14. Vérification et dépannage

### 14.1 Commandes principales (cEdge)

| Commande | Utilité |
|---|---|
| `show sdwan bfd sessions` | État des sessions, détection d'un blackout |
| `show sdwan bfd history` | Historique des transitions, détection de flaps |
| `show sdwan app-route stats` | Mesures de perte, latence, gigue, détection d'un brownout |
| `show sdwan app-route sla-class` | Classes SLA et tunnels conformes |
| `show sdwan omp tlocs` | Résolution des TLOC |
| `show sdwan control connections` | Blackout ou brownout du plan de contrôle |
| `show sdwan policy service-path ...` | Chemin retenu pour un flux donné |
| `show interfaces` | Erreurs, duplex, drops, saturation d'une interface |
| `show platform hardware ...` ou équivalent | Charge CPU et ressources du routeur |

### 14.2 Exemple de sortie : un blackout (illustratif)

```
Magasin-2750# show sdwan bfd sessions

SYSTEM IP     SITE ID  STATE  SOURCE TLOC COLOR  REMOTE TLOC COLOR  TX INT  UPTIME     TRANS
------------------------------------------------------------------------------------------
10.255.0.1    1000     down   biz-internet       biz-internet       500     0:00:00:00 3
10.255.0.2    1001     down   biz-internet       biz-internet       500     0:00:00:00 3
10.255.0.1    1000     up     lte                lte                10000   12:03:09   1
10.255.0.2    1001     up     lte                lte                10000   12:03:08   1
```

Lecture : les sessions biz-internet sont down, le magasin fonctionne sur lte.

### 14.3 Exemple de sortie : un brownout (illustratif)

```
Magasin-2750# show sdwan app-route stats summary

REMOTE SYSTEM IP  LOCAL COLOR    REMOTE COLOR   LOSS  LATENCY  JITTER  SLA CLASSES MET
-----------------------------------------------------------------------------------------
10.255.0.1        mpls           mpls           0     19       2       CAISSES,VOIX,BUREAUTIQUE
10.255.0.1        biz-internet   biz-internet   3     118      41      BUREAUTIQUE
```

Lecture : le tunnel biz-internet est up (donc aucun blackout) mais dégradé (3 % de perte, 118 ms, 41 ms de gigue). Il n'est conforme qu'à SLA-BUREAUTIQUE.

### 14.4 Méthodologie de classification d'un incident

1. Les sessions BFD sont-elles down ? Oui : blackout, passer à l'identification de la cause (physique, opérateur, alimentation, blocage de port).
2. Sinon, les mesures de perte, latence, gigue dépassent-elles les seuils ? Oui : brownout durable.
3. Sinon, les utilisateurs se plaignent-ils malgré des mesures normales ? Suspecter une panne grise : MTU, filtrage, port applicatif, chemin ECMP partiel, problème applicatif.
4. Vérifier les compteurs d'interface locale : erreurs, duplex, drops.
5. Vérifier la charge CPU du routeur et du hub distant.
6. Comparer avec d'autres sites utilisant le même opérateur : un incident commun désigne une cause opérateur.
7. Tester le chemin avec une sonde de bout en bout et des tailles de paquets variées (détection d'un trou noir de MTU).

### 14.5 Causes fréquentes et diagnostics

| Symptôme | Hypothèse | Vérification |
|---|---|---|
| Sessions BFD down sur toutes les couleurs d'un site | Panne du routeur, de l'alimentation, ou du LAN du site | Joindre le site hors bande, vérifier l'alimentation |
| Session BFD down sur une couleur uniquement | Coupure ou blocage sur ce transport | Vérifier l'accès, tester UDP 12346, contacter l'opérateur |
| Flaps répétés | Lien instable, timers trop agressifs | `bfd history`, relâcher les timers, vérifier le lien |
| Lenteurs sans alarme BFD | Brownout | `app-route stats`, congestion, compteurs d'interface |
| Perte de gros paquets seulement | Trou noir de MTU | Test avec paquets de tailles croissantes, `pmtu-discovery` |
| Bascule permanente sur la 4G | Seuils SLA irréalistes | Comparer seuils et mesures normales |
| Incident simultané sur beaucoup de sites d'un même opérateur | Cause opérateur | Corréler les sites par opérateur |

---

## 15. Tests de résilience en laboratoire

### 15.1 Simuler un blackout

Sur un équipement Linux placé sur le chemin entre deux WAN Edge virtuels :

```
# Coupure complète du trafic sur l'interface
sudo ip link set dev eth1 down

# Ou blocage ciblé du port des tunnels
sudo iptables -A FORWARD -p udp --dport 12346 -j DROP
sudo iptables -A FORWARD -p udp --sport 12346 -j DROP
```

Mesurer le délai entre la coupure et le passage de la session BFD à down, puis le délai de rétablissement du trafic applicatif.

### 15.2 Simuler un brownout

```
# Ajout de 3 % de perte, 80 ms de latence et 20 ms de gigue
sudo tc qdisc add dev eth1 root netem delay 80ms 20ms loss 3%

# Modification des paramètres
sudo tc qdisc change dev eth1 root netem delay 120ms 40ms loss 5%

# Suppression de la dégradation
sudo tc qdisc del dev eth1 root
```

Observer l'évolution des mesures (`show sdwan app-route stats`), le moment où un tunnel cesse d'être conforme et le déplacement des flux.

### 15.3 Simuler une panne grise

```
# Réduction du MTU pour créer un trou noir de MTU (interface du chemin)
sudo ip link set dev eth1 mtu 1300

# Perte unidirectionnelle sur un seul sens (règle sur un seul sens du flux)
sudo iptables -A FORWARD -s 198.51.100.20 -p udp --dport 12346 -j DROP
```

Les commandes ci-dessus sont des exemples de laboratoire, à adapter à votre topologie. Ne les exécutez jamais sur un équipement de production.

### 15.4 Grille de résultats à relever

| Test | Délai de détection | Délai de rétablissement applicatif | Flux affectés | Observations |
|---|---|---|---|---|
| Coupure franche | | | | |
| Perte 3 % | | | | |
| Perte 30 % | | | | |
| Latence 200 ms | | | | |
| Trou noir de MTU | | | | |
| Perte unidirectionnelle | | | | |

---

## 16. Pièges classiques

1. Croire que BFD protège contre tous les types de pannes : il ne voit pas les brownouts importants ni les pannes grises.
2. Laisser les valeurs par défaut de app-route sans analyser le délai de réaction réel.
3. Fixer des seuils SLA irréalistes qui provoquent des bascules permanentes.
4. Utiliser `strict` sur des flux critiques.
5. Resserrer les timers BFD partout sans test, au risque de flaps et de charge CPU sur les hubs.
6. Oublier que deux opérateurs ne garantissent pas deux chemins physiques.
7. Ne pas surveiller le séjour sur lien de secours et la consommation de données.
8. Négliger la cause locale d'un brownout (duplex, câble, QoS, CPU).
9. Dimensionner le datacenter de secours pour l'état nominal et non pour la charge reportée en cas de panne.
10. Oublier les certificats : un blackout de contrôle peut être entièrement évitable.

---

## 17. Bonnes pratiques

1. Traiter blackout et brownout comme deux problèmes distincts, avec des mécanismes et des indicateurs distincts.
2. Régler les timers BFD selon le type de lien et la criticité du flux.
3. Définir les classes SLA à partir de mesures réelles et valider avec un site pilote.
4. Protéger les flux critiques légers par duplication de paquets et la voix par FEC adaptatif.
5. Utiliser le repli sur le meilleur tunnel pour les flux critiques.
6. Vérifier la diversité physique des accès, pas seulement la diversité des opérateurs.
7. Surveiller les compteurs d'interface, le CPU et le nombre de sessions BFD.
8. Mettre en place des alarmes sur les flaps, le séjour sur lien de secours et la consommation des forfaits.
9. Tester régulièrement les scénarios de panne en laboratoire et, de façon contrôlée, sur des sites pilotes.
10. Documenter un guide d'exploitation avec arbre de décision pour classer rapidement un incident.
11. Dimensionner les hubs et les datacenters pour le pic d'activité et pour la charge reportée lors d'une bascule.
12. Suivre les dates d'expiration des certificats.

---

## 18. Travaux pratiques

### TP 1 : Mesurer un blackout

1. Établir deux tunnels entre deux WAN Edge virtuels.
2. Lancer un ping continu et une connexion TCP longue.
3. Couper un transport et mesurer le délai de détection BFD et le temps d'interruption du ping.
4. Recommencer avec trois jeux de timers et comparer.

### TP 2 : Montrer qu'un brownout échappe à BFD

1. Injecter 30 % de perte avec netem sur un transport.
2. Vérifier que la session BFD reste up.
3. Observer les mesures de perte, la non-conformité SLA et le déplacement des flux.
4. Comparer au calcul de la section 6.

### TP 3 : Délai de réaction de l'AAR

1. Régler le poll interval à 600 s, puis 120 s, puis 30 s.
2. Pour chaque réglage, introduire une dégradation et mesurer le délai avant le changement de chemin.
3. Observer l'apparition d'oscillations avec les valeurs les plus courtes.

### TP 4 : Correction de perte

1. Activer FEC adaptatif sur un flux voix de test.
2. Introduire 3 % de perte et comparer la qualité (perte résiduelle) avec et sans FEC.
3. Activer la duplication de paquets sur un flux transactionnel et mesurer le taux d'échec pendant une microcoupure d'un des deux transports.
4. Mesurer le surcoût de bande passante.

### TP 5 : Panne grise

1. Réduire le MTU d'un chemin pour créer un trou noir de MTU.
2. Constater que BFD reste up alors que certains transferts échouent.
3. Activer la découverte du MTU de chemin et comparer.

### TP 6 : Guide d'exploitation

1. À partir de la méthodologie de la section 14.4, rédiger un arbre de décision d'une page.
2. Le tester sur trois pannes simulées par un collègue sans connaître leur nature.
3. Relever les écarts et améliorer le guide.

---

## 19. Questionnaire d'auto-évaluation

**Q1. Quelle différence entre blackout et brownout ?**
Réponse : le blackout est une coupure franche, le chemin ne transporte plus rien. Le brownout est une dégradation partielle : le chemin reste ouvert mais perd des paquets, ajoute de la latence ou de la gigue.

**Q2. Quel mécanisme détecte principalement un blackout ?**
Réponse : BFD en mode liveness, avec un délai égal à hello x multiplicateur.

**Q3. Avec hello 500 ms et multiplicateur 4, quel est le délai de détection d'un blackout ?**
Réponse : environ 2 secondes.

**Q4. Pourquoi un lien avec 30 % de perte peut-il rester up pour BFD ?**
Réponse : BFD exige 7 hello consécutifs perdus (avec un multiplicateur de 7). Avec 30 % de perte aléatoire, la probabilité est d'environ 0,02 % par fenêtre, donc le lien reste presque toujours déclaré up.

**Q5. Quel mécanisme détecte un brownout durable ?**
Réponse : les mesures de perte, latence et gigue de BFD agrégées sur les fenêtres de app-route, comparées aux seuils des classes SLA.

**Q6. Pourquoi le FEC et la duplication sont-ils utiles en plus de l'AAR ?**
Réponse : l'AAR réagit après la fenêtre d'évaluation, donc lentement. La correction de perte protège les flux paquet par paquet, dès la première perte.

**Q7. Pourquoi la duplication de paquets convient-elle bien aux flux de paiement ?**
Réponse : ces flux sont très légers, le doublement de leur bande passante est négligeable, et la protection contre la perte et les microcoupures est forte.

**Q8. Quel est le risque du mode strict pour les caisses ?**
Réponse : si aucun tunnel n'est conforme, le trafic est rejeté, ce qui coupe l'encaissement. Il faut préférer le repli sur le meilleur tunnel.

**Q9. Que se passe-t-il pour le plan de données si tous les vSmart tombent ?**
Réponse : grâce au graceful restart, les routes et clés sont conservées pendant la durée configurée (43200 s par défaut) : le trafic existant continue. Au-delà, il se dégrade.

**Q10. Pourquoi deux accès de deux opérateurs ne garantissent-ils pas la résilience ?**
Réponse : ils peuvent partager le même fourreau, la même boucle locale ou la même infrastructure amont. Une cause commune coupe les deux en même temps.

---

## 20. Glossaire

| Terme | Définition |
|---|---|
| AAR | Application-Aware Routing, choix du chemin selon la qualité mesurée |
| BFD | Bidirectional Forwarding Detection |
| Blackout | Coupure franche d'un chemin |
| Brownout | Dégradation partielle de la qualité d'un chemin |
| Bufferbloat | Gonflement de la latence causé par des files d'attente trop longues |
| Duplication de paquets | Envoi de chaque paquet sur deux tunnels, la première copie reçue est retenue |
| FEC | Forward Error Correction, redondance permettant de reconstruire des paquets perdus |
| Flap | Alternance répétée entre deux états (up et down, conforme et non conforme) |
| Gigue | Variation de la latence entre paquets |
| Graceful restart | Conservation temporaire des routes et clés pendant une panne du plan de contrôle |
| Panne grise | Panne partielle ou sélective, vue par certains observateurs seulement |
| MTTD, MTTR | Délai moyen de détection, délai moyen de rétablissement |
| Trou noir de MTU | Perte silencieuse des paquets dépassant une taille donnée |

---

## 21. Références

- Documentation officielle Cisco SD-WAN : guides de configuration Policies (Application-Aware Routing, Forward Error Correction, Packet Duplication) et Systems and Interfaces (BFD).
- Cisco SD-WAN Design Guide (Cisco Validated Design).
- RFC 5880 : Bidirectional Forwarding Detection (BFD).
- Huang et al., "Gray Failure: The Achilles' Heel of Cloud-Scale Systems", HotOS 2017 : article de référence sur les pannes partielles et grises.
- Cours associés : Cours complet SD-WAN et protocole OMP, Cours complet SD-WAN et protocole BFD, Cours complet SD-WAN et SLA.

Les valeurs par défaut, les bornes de configuration, la disponibilité des fonctions de correction de perte, les limites de plateforme et les syntaxes de commandes évoluent selon les versions : validez-les dans la documentation correspondant à votre version avant toute mise en production.
