# La sélection dynamique de chemin, pas à pas

Niveau : étudiants en réseaux et télécoms
Notions clés : brownout, blackout, BFD, SLA, politique de routage applicatif (application-aware routing)

---

## Table des matières

1. Objectifs d'apprentissage
2. Principe général
3. Les deux familles de pannes : brownout et blackout
4. Les outils de mesure : BFD et SLA
5. La politique : relier application, SLA et chemin préféré
6. Analyse pas à pas du graphique de cours
7. Chronologie détaillée
8. Différence entre brownout et blackout
9. Exemples Carrefour
10. Mise en oeuvre Cisco et Fortinet
11. Points d'attention et nuances
12. Questions de révision

---

## 1. Objectifs d'apprentissage

- Expliquer comment un SD-WAN choisit un chemin pour chaque application.
- Distinguer une dégradation (brownout) d'une coupure (blackout) et dire quel mécanisme détecte chacune.
- Lire et commenter un graphique de perte de paquets avec des seuils SLA.
- Écrire une classe SLA et une politique applicative simples.

---

## 2. Principe général

Dans un WAN classique, le routage choisit le chemin d'après la topologie (le lien est-il up ou down ?). Il ne voit pas qu'un lien "up" perd 3 % des paquets et ruine la voix.

Le SD-WAN ajoute une couche de décision fondée sur la qualité réelle mesurée en continu, application par application :

```
1. Mesurer     sondes sur chaque tunnel (perte, latence, gigue)
2. Comparer    avec les seuils (SLA) de chaque classe d'application
3. Décider     choisir le chemin conforme et préféré
4. Basculer    déplacer uniquement les flux concernés
5. Revenir     quand le chemin préféré est de nouveau stable
```

---

## 3. Les deux familles de pannes

### 3.1 Brownout (dégradation)

Le brownout est une dégradation de la qualité du réseau sans coupure totale.

- Le réseau fonctionne toujours.
- La latence augmente.
- La perte de paquets apparaît.
- Les applications deviennent lentes.

Détection : les mesures SLA. Le lien est vivant, donc BFD reste up, mais les valeurs mesurées sortent des seuils.

Exemple Carrefour : une connexion Orange subit une forte latence. Les caisses fonctionnent encore, mais les paiements deviennent lents. Le SD-WAN détecte la dégradation grâce aux SLA et bascule les flux concernés vers SFR.

### 3.2 Blackout (coupure)

Le blackout correspond à une perte totale de connexion.

- Le lien WAN est complètement indisponible.
- Les communications sont interrompues.
- Les applications ne sont plus accessibles sur ce lien.
- Le SD-WAN bascule vers un autre lien disponible.

Détection : les sondes BFD. Elles ne reçoivent plus de réponse et déclarent le tunnel down.

Exemple Carrefour : la connexion Orange d'un magasin tombe complètement. Le SD-WAN détecte la coupure grâce à BFD et bascule automatiquement sur SFR.

---

## 4. Les outils de mesure

### 4.1 BFD (Bidirectional Forwarding Detection)

BFD envoie des paquets très légers et réguliers à travers chaque tunnel. Il sert à deux choses :

1. Détection de panne : si plusieurs paquets consécutifs manquent, le tunnel est déclaré down.
2. Mesure de qualité : à partir des mêmes sondes, le SD-WAN calcule perte, latence et gigue.

Réglages par défaut chez Cisco : un paquet par seconde (hello 1000 ms) et un multiplicateur de 7. Il faut donc 7 réponses manquantes pour conclure à la panne, d'où une détection en environ 7 secondes. C'est exactement la valeur du graphique de cours ("détecté par BFD en 7 s").

### 4.2 SLA (Service Level Agreement) mesuré

Trois critères :

- Latence : temps de réponse (ms).
- Gigue (jitter) : variation de la latence (ms).
- Perte de paquets : pourcentage de paquets perdus.

Une classe SLA est un jeu de seuils. Un chemin est dit conforme si ses trois mesures sont dans les seuils.

Exemple de calcul :

| Critère | Orange | SFR | Seuil SLA |
|---|---|---|---|
| Latence | 80 ms | 25 ms | 50 ms maximum |
| Gigue | 30 ms | 5 ms | 10 ms maximum |
| Perte | 4 % | 0 % | 1 % maximum |
| Verdict | Non conforme (3 critères dépassés) | Conforme | |

---

## 5. La politique : application, SLA et chemin préféré

Une politique de routage applicatif relie trois éléments :

```
Quelle application ?  -->  Quel SLA exiger ?  -->  Quel chemin préférer ?
 (DSCP, liste d'apps)       (classe SLA)           (couleur MPLS, Internet)
```

Exemple de classes pour un magasin :

| Classe | Applications | Perte max | Chemin préféré | Secours |
|---|---|---|---|---|
| VOICE | Téléphonie IP | 1 % | MPLS | Fibre Internet |
| TRANSACTIONAL | Caisses, paiements | 1 % à 2 % | Fibre | MPLS |
| VIDEO | Visioconférence, caméras | 2 % | Fibre | MPLS |
| BULK | Sauvegardes, mises à jour | non exigeant | Câble | Tout lien |

Selon la politique, quand le chemin préféré est non conforme, le trafic passe sur un chemin conforme. Si aucun chemin n'est conforme, on choisit en général le meilleur disponible.

---

## 6. Analyse pas à pas du graphique de cours

Le graphique montre la perte mesurée (en pourcent) en fonction de l'heure, de 10:30 à 11:30, pour trois transports :

- Fibre FAI B (courbe bleue)
- MPLS (courbe sombre, avec un blackout de 11:05 à 11:16, BFD down)
- Câble FAI C (pointillés orange, perte stable et faible, environ 0,4 %)
- Une ligne pointillée horizontale à 1 % représente le seuil VOICE et TRANSACTIONAL.

Lecture :

1. Avant 10:42, tous les liens sont conformes. La fibre est quasi à 0 %.
2. À 10:42, la fibre passe à environ 2,5 % de perte : c'est un brownout. Le lien n'est pas coupé, mais il dépasse le seuil de 1 %, et aussi le seuil de 2 % annoncé dans la légende du schéma.
3. Après la fenêtre de mesure, les classes TRANSACTIONAL et VIDEO basculent vers le MPLS, qui est conforme.
4. Vers 10:57, la fibre redevient bonne (la courbe retombe presque à zéro).
5. À 11:05, le MPLS subit un blackout. BFD le détecte en 7 secondes et déclare le tunnel down.
6. VOICE et TRANSACTIONAL, qui étaient sur le MPLS, basculent vers la fibre, redevenue conforme.
7. À 11:16, le MPLS revient. La politique pourra y renvoyer le trafic une fois la stabilité confirmée (voir le document sur l'hystérisation).

Remarque d'examen : dans la figure, la ligne de seuil est à 1 % et l'annotation du brownout évoque 2 %. Les deux valeurs sont dépassées par une perte de 2,5 %, donc la conclusion est identique. Dans votre copie, indiquez la valeur de seuil que vous retenez.

Autre remarque : le schéma ne précise pas ce que fait VOICE à 10:42. Une lecture cohérente est que la voix se trouvait déjà sur le MPLS (chemin préféré de la voix), donc elle n'avait pas à bouger.

---

## 7. Chronologie détaillée

| Heure | Événement | Mécanisme | Action du SD-WAN |
|---|---|---|---|
| 10:30 | Situation nominale | SLA mesuré, tous conformes | Chaque classe sur son chemin préféré |
| 10:42 | Brownout de la fibre, perte de 2,5 % | Mesure SLA, seuil dépassé | Après la fenêtre de mesure, TRANSACTIONAL et VIDEO vers MPLS |
| 10:57 environ | Fibre rétablie | Mesure SLA conforme | Conformité confirmée, pas de retour immédiat |
| 11:05 | Blackout du MPLS | BFD, détection en 7 s | Tunnel MPLS down, VOICE et TRANSACTIONAL vers la fibre |
| 11:16 | MPLS rétabli | BFD up | Retour possible après stabilisation |

Point important : à 11:05, la déclaration down de BFD retire le tunnel pour toutes les classes, pas seulement celles qui ont une classe SLA. Un blackout est donc plus brutal qu'un brownout, mais beaucoup plus simple à détecter.

---

## 8. Différence entre brownout et blackout

| | Brownout | Blackout |
|---|---|---|
| État du lien | Dégradé | Coupé |
| Ressenti | Connexion lente, paiements lents | Aucune connexion |
| Détection | SLA (perte, latence, gigue) | BFD (perte du lien) |
| Délai de détection | Dépend de la fenêtre de mesure (secondes à minutes) | Environ 7 s par défaut |
| Difficulté | Plus difficile : le lien semble vivant | Plus simple : réponse absente |

À retenir : le brownout, le réseau fonctionne mal. Le blackout, le réseau ne fonctionne plus.

---

## 9. Exemples Carrefour

1. Brownout : la fibre Orange du magasin perd 2,5 % des paquets entre 10:42 et 10:57. Les paiements par carte sont ralentis. Le SD-WAN déplace uniquement les flux transactionnels vers le MPLS. Les sauvegardes, moins sensibles, restent où elles sont.
2. Blackout : à 11:05, le MPLS est coupé. En 7 secondes les flux reviennent sur la fibre, désormais saine. Les caissiers ne remarquent qu'un très bref ralentissement.

---

## 10. Mise en oeuvre Cisco et Fortinet

### Cisco : classes SLA et politique applicative

```
policy
 sla-class VOICE_SLA
  loss 1
  latency 150
  jitter 30
 !
 sla-class TRANSACTIONAL_SLA
  loss 1
  latency 100
  jitter 50
 !
 app-route-policy AAR_MAGASIN
  vpn-list VPN_MAGASIN
   sequence 10
    match
     dscp 46
    !
    action
     sla-class VOICE_SLA preferred-color mpls
    !
   !
   sequence 20
    match
     dscp 26
    !
    action
     sla-class TRANSACTIONAL_SLA preferred-color biz-internet
```

Paramètres par défaut utiles : BFD hello 1 s et multiplicateur 7 ; intervalle de sondage app-route de 600 s et multiplicateur 6, que l'on réduit souvent dans les labs ou pour les sites critiques afin de réagir plus vite.

### Fortinet : health check et règle SD-WAN

```
config system sdwan
    config health-check
        edit "HC_DC1"
            set server "10.100.0.10"
            set protocol ping
            set interval 500
            set failtime 5
            set recoverytime 5
            config sla
                edit 1
                    set link-cost-factor latency jitter packet-loss
                    set latency-threshold 50
                    set jitter-threshold 10
                    set packetloss-threshold 1
                next
            end
        next
    end
    config service
        edit 1
            set name "CAISSES"
            set mode sla
            set dst "DC1_SUBNETS"
            config sla
                edit "HC_DC1"
                    set id 1
                next
            end
            set priority-members 1 2
        next
    end
end
```

Lecture : la sonde interroge le serveur de DC1 toutes les 500 ms. Si les seuils (50 ms, 10 ms, 1 %) sont dépassés, le membre est jugé non conforme et le service choisit un autre membre dans l'ordre de priorité.

---

## 11. Points d'attention et nuances

1. La fenêtre de mesure est un compromis : courte, elle réagit vite mais risque des faux positifs ; longue, elle est stable mais lente.
2. Les valeurs des seuils (1 %, 2 %, 50 ms) sont des exemples pédagogiques. Elles se règlent selon l'application et le contrat.
3. La bascule est par flux ou par classe, pas pour tout le site. C'est ce qui distingue le SD-WAN d'un simple failover de lien.
4. Les sessions déjà établies peuvent être coupées lors d'un déplacement de chemin (changement d'adresse source NAT, par exemple). Les applications modernes se reconnectent, mais ce point doit être testé.
5. Les sondes consomment un peu de bande passante, surtout sur les liens 4G ou 5G facturés au volume.

---

## 12. Questions de révision

1. Quel mécanisme détecte un brownout et lequel détecte un blackout ?
   Réponse : le brownout est détecté par la mesure SLA, le blackout par BFD.

2. Pourquoi BFD met-il environ 7 secondes à déclarer un tunnel down par défaut ?
   Réponse : hello de 1 seconde multiplié par un multiplicateur de 7.

3. Pourquoi le SD-WAN est-il plus fin qu'un failover classique ?
   Réponse : il décide par application selon la qualité mesurée, et non uniquement selon l'état up ou down du lien.

4. Que se passe-t-il pour les classes sans SLA quand BFD déclare un tunnel down ?
   Réponse : le tunnel est retiré de la table de transfert, donc tout le trafic est redistribué sur les tunnels restants.

5. Pourquoi ne revient-on pas immédiatement sur le lien préféré dès qu'il redevient bon ?
   Réponse : pour éviter les battements (flapping). C'est l'objet de l'hystérisation.
