# L'hystérisation dans le SD-WAN

Niveau : étudiants en réseaux et télécoms
Notions liées : SLA, BFD, bascule (failover), retour au lien principal (failback), battement de lien (flapping)

---

## Table des matières

1. Objectifs d'apprentissage
2. Le problème : le battement de lien
3. Définition
4. Principe : deux seuils et du temps
5. Les mécanismes d'hystérisation
6. Simulation chiffrée
7. Les compromis à régler
8. Exemple de seuils et scénario Carrefour
9. Mise en oeuvre Cisco et Fortinet
10. Erreurs fréquentes
11. Questions techniques et réponses
12. Questions de révision

---

## 1. Objectifs d'apprentissage

- Expliquer pourquoi un SD-WAN sans hystérisation devient instable.
- Décrire les mécanismes : seuils différents, durées de confirmation, fenêtres de mesure.
- Réaliser une simulation de bascule avec et sans hystérisation.
- Régler les compromis entre réactivité et stabilité.

---

## 2. Le problème : le battement de lien

Supposons un seuil de latence unique à 50 ms. Le lien principal oscille autour de cette valeur : 48 ms, 52 ms, 49 ms, 53 ms. Sans précaution, le SD-WAN bascule à chaque franchissement. Les flux changent de lien en permanence.

Conséquences :

- sessions coupées ou perturbées à chaque changement ;
- changement fréquent d'adresse IP publique de sortie à cause du NAT ;
- latences variables pour les applications, dont l'expérience est pire que sur un lien légèrement dégradé mais stable ;
- charge inutile sur les équipements et logs saturés.

Ce phénomène s'appelle le battement de lien (flapping).

---

## 3. Définition

L'hystérisation permet d'éviter les basculements répétés entre deux liens lorsque la qualité du réseau varie légèrement.

Le mot vient de la physique : l'état d'un système dépend aussi de son histoire récente, pas seulement de la valeur actuelle. Comme un thermostat qui allume le chauffage à 18 degrés et l'éteint à 20 degrés, et non à 19 degrés dans les deux sens.

Importance :

- stabiliser les connexions ;
- éviter les changements fréquents de liens ;
- maintenir les performances des applications.

---

## 4. Principe : deux seuils et du temps

```
Latence (ms)
  60 |
  50 |-------------------- seuil de dégradation : on quitte le lien principal
  40 |-------------------- seuil de retour : on y revient
  30 |  ........ latence normale
     +--------------------------------------> temps
```

Fonctionnement : le SD-WAN surveille la qualité des liens grâce aux mesures SLA. Il attend que le seuil de dégradation soit dépassé avant de déclencher un basculement. Il n'y revient que lorsque la qualité est nettement meilleure et stable.

La zone entre 40 et 50 ms est la zone d'hystérésis : dans cet intervalle, on ne change pas d'état.

---

## 5. Les mécanismes d'hystérisation

| Mécanisme | Description | Effet |
|---|---|---|
| Deux seuils distincts | Seuil de sortie (50 ms) supérieur au seuil de retour (40 ms) | Crée la zone de non changement |
| Confirmation sur plusieurs échantillons | Il faut N mesures consécutives hors seuil (ou conformes) | Ignore les pics isolés |
| Temps de maintien (hold-down) | Une fois basculé, on reste X secondes sur le nouveau lien | Empêche le retour immédiat |
| Fenêtre glissante | On évalue une moyenne sur une période | Lisse les mesures |
| Tolérance (variance) | Deux chemins à moins de X ms d'écart sont considérés équivalents | Évite de changer pour un gain minime |
| Retour manuel ou différé | Failback soumis à validation ou à une plage horaire | Évite les ruptures en pleine pointe |

---

## 6. Simulation chiffrée

Hypothèse : la latence d'Orange est mesurée régulièrement (un échantillon par ligne du tableau).

Deux politiques :

- Politique A, sans hystérisation : bascule dès qu'un échantillon dépasse 50 ms, retour dès qu'un échantillon est de 50 ms ou moins.
- Politique B, avec hystérisation : bascule après 3 échantillons consécutifs au dessus de 50 ms, retour après 3 échantillons consécutifs en dessous de 40 ms.

| Échantillon | Latence Orange | Lien utilisé (A) | Lien utilisé (B) |
|---|---|---|---|
| 0 | 30 ms | Orange | Orange |
| 1 | 52 ms | SFR | Orange (compteur 1) |
| 2 | 48 ms | Orange | Orange (compteur remis à zéro) |
| 3 | 55 ms | SFR | Orange (compteur 1) |
| 4 | 47 ms | Orange | Orange (compteur remis à zéro) |
| 5 | 53 ms | SFR | Orange (compteur 1) |
| 6 | 51 ms | SFR | Orange (compteur 2) |
| 7 | 52 ms | SFR | SFR (compteur 3, bascule) |
| 8 | 54 ms | SFR | SFR |
| 9 | 45 ms | Orange | SFR (45 n'est pas sous 40) |
| 10 | 38 ms | Orange | SFR (retour, compteur 1) |
| 11 | 36 ms | Orange | SFR (retour, compteur 2) |
| 12 | 35 ms | Orange | Orange (compteur 3, retour) |
| 13 | 34 ms | Orange | Orange |

Résultat :

- Politique A : 6 changements de lien en 13 échantillons (un battement net).
- Politique B : 2 changements (une bascule, un retour).

Coût de la politique B : la bascule intervient 6 échantillons plus tard que A (échantillon 7 contre 1), et le retour 2 échantillons plus tard (12 contre 9). C'est le prix de la stabilité. Si un échantillon représente une seconde, on gagne de la stabilité au prix de quelques secondes de réaction.

À retenir : l'hystérisation échange un peu de réactivité contre beaucoup de stabilité.

---

## 7. Les compromis à régler

| Si on... | Avantage | Inconvénient |
|---|---|---|
| Raccourcit les délais de confirmation | Réagit vite à une vraie dégradation | Plus de battements |
| Allonge les délais | Très stable | Garde longtemps un lien dégradé |
| Éloigne les deux seuils | Peu de changements | Tolère une qualité moyenne plus longtemps |
| Rapproche les deux seuils | Revient vite sur le lien préféré | Risque de rebasculer |

Règle pratique : l'application la plus sensible (voix) demande des délais courts pour la bascule, mais un retour prudent. On bascule vite, on revient lentement.

Les blackouts (BFD) sont traités à part : la détection est rapide (environ 7 secondes) et la bascule immédiate. L'hystérisation concerne surtout les brownouts mesurés par le SLA et le retour sur le lien après une panne.

---

## 8. Exemple de seuils et scénario Carrefour

Exemple de seuils fictifs, donnés pour illustrer le fonctionnement :

- latence normale : 30 ms ;
- seuil de dégradation : 50 ms ;
- retour au lien principal : lorsque la latence redescend sous 40 ms.

Scénario : un magasin Carrefour utilise deux connexions Internet, Orange et SFR. Si la latence d'Orange dépasse 50 ms, le SD-WAN bascule vers SFR. Il attend ensuite que la qualité d'Orange redevienne stable avant de revenir sur ce lien.

Application aux paiements : si Orange oscille entre 45 et 55 ms à cause de congestion, sans hystérisation les terminaux de paiement verraient leur connexion changer toutes les quelques secondes. Avec hystérisation, ils basculent une fois sur SFR et y restent jusqu'à ce qu'Orange soit stable sous 40 ms.

---

## 9. Mise en oeuvre Cisco et Fortinet

### Cisco

La stabilité repose sur les paramètres de sondage de l'application-aware routing : intervalle de mesure et multiplicateur définissent la fenêtre glissante sur laquelle les mesures sont moyennées. Par défaut, l'intervalle est de 600 secondes et le multiplicateur de 6, ce qui donne une fenêtre très stable mais lente ; on les réduit pour les sites critiques.

```
bfd app-route multiplier 6
bfd app-route poll-interval 600000
```

La valeur de poll-interval est en millisecondes. Les classes SLA peuvent aussi tolérer une variance de latence entre chemins afin de ne pas changer pour une différence minime.

### Fortinet

```
config system sdwan
    config health-check
        edit "HC_DC1"
            set server "10.100.0.10"
            set interval 1000
            set failtime 5
            set recoverytime 10
            config sla
                edit 1
                    set latency-threshold 50
                    set jitter-threshold 10
                    set packetloss-threshold 1
                next
            end
        next
    end
end
```

Lecture :

- failtime 5 : il faut 5 échecs consécutifs de la sonde pour déclarer le membre en échec.
- recoverytime 10 : il faut 10 réussites consécutives avant de le déclarer de nouveau sain.

C'est un mécanisme de confirmation asymétrique : on sort vite, on revient prudemment. Selon la version, d'autres paramètres comme un temps de maintien (hold-down) sur la règle SD-WAN complètent le dispositif.

---

## 10. Erreurs fréquentes

1. Utiliser un seul seuil pour la sortie et le retour.
2. Régler des délais trop longs : on garde un lien dégradé pendant que les utilisateurs se plaignent.
3. Régler des délais trop courts : l'hystérisation existe mais ne sert à rien.
4. Oublier de tester avec un lien réellement instable (par exemple par injection de perte avec un outil de simulation).
5. Confondre hystérisation et lenteur de détection d'une panne franche : BFD reste rapide.

---

## 11. Questions techniques et réponses

Ces questions proviennent de la réflexion du cours et sont utiles pour une soutenance.

Question 1 : comment le SD-WAN détermine-t-il les seuils de dégradation (latence, gigue, perte) ainsi que le délai de retour au lien principal pour éviter les battements ?

Réponse : les seuils ne sont pas calculés automatiquement, ils sont définis par l'administrateur dans des classes SLA, selon les besoins de chaque application (par exemple 150 ms pour la voix, perte de 1 %). Ils se déduisent de la tolérance de l'application et des contrats avec les opérateurs. Le délai de retour se règle par des paramètres de confirmation (échantillons consécutifs, fenêtre de mesure, temps de maintien). On valide ensuite par des tests en conditions dégradées.

Question 2 : comment le SD-WAN gère-t-il les priorités DSCP lorsque plusieurs applications critiques utilisent simultanément un lien saturé ? La QoS garantit-elle réellement la bande passante ?

Réponse : la QoS garantit un minimum de bande passante à chaque classe en cas de congestion (par exemple 25 % à TRANSACTIONAL), et une priorité stricte plafonnée pour la voix. Ce n'est pas une garantie absolue : si la somme des besoins critiques dépasse la capacité du lien, la QoS ne peut pas tout satisfaire. Elle répartit la pénurie selon les priorités. Le dimensionnement reste indispensable.

Question 3 : comment le SD-WAN combine-t-il la QoS, la sélection des TLOC, BFD et le SLA pour maintenir les performances en cas de dégradation ?

Réponse : BFD détecte les coupures et mesure la qualité, le SLA compare ces mesures aux seuils de chaque application, la politique choisit le TLOC conforme et préféré, et la QoS organise le passage des paquets sur le lien choisi. L'hystérisation stabilise les décisions. Chaque brique traite un problème différent : détecter, évaluer, décider, prioriser, stabiliser.

---

## 12. Questions de révision

1. Définissez l'hystérisation en une phrase.
   Réponse : mécanisme qui évite les basculements répétés entre liens en utilisant des seuils et des délais différents à l'aller et au retour.

2. Pourquoi le seuil de retour est-il inférieur au seuil de bascule ?
   Réponse : pour créer une zone intermédiaire où l'état ne change pas, ce qui empêche les va-et-vient.

3. Quel est le coût de l'hystérisation ?
   Réponse : une réaction un peu plus lente à une dégradation réelle.

4. L'hystérisation ralentit-elle la détection d'une panne franche ?
   Réponse : non, BFD détecte les coupures en quelques secondes, indépendamment de ces réglages.

5. Dans l'exemple, combien de changements de lien évite-t-on ?
   Réponse : 6 changements sans hystérisation contre 2 avec, donc 4 évités.
