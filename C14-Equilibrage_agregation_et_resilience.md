# Équilibrage, agrégation et résilience dans un SD-WAN

Niveau : étudiants en réseaux et télécoms
Fil rouge : un magasin Carrefour avec deux accès, Orange et SFR

---

## Table des matières

1. Objectifs d'apprentissage
2. Définitions
3. Importance
4. L'équilibrage en pratique
5. L'agrégation : ce qu'elle apporte et ses limites
6. La résilience
7. Dimensionnement : le calcul qui évite les mauvaises surprises
8. Exemples Carrefour
9. Mise en oeuvre Cisco et Fortinet
10. Erreurs fréquentes
11. Questions de révision

---

## 1. Objectifs d'apprentissage

- Distinguer équilibrage, agrégation et résilience.
- Expliquer pourquoi une session unique n'utilise normalement qu'un seul lien.
- Dimensionner des liens pour qu'une panne ne détruise pas les applications critiques.
- Choisir un algorithme de répartition adapté.

---

## 2. Définitions

| Notion | Définition | Question à laquelle elle répond |
|---|---|---|
| Équilibrage (load balancing) | Répartir le trafic entre plusieurs connexions | Comment ne pas surcharger un seul lien ? |
| Agrégation | Utiliser plusieurs connexions pour augmenter la capacité réseau disponible | Comment disposer de plus de bande passante au total ? |
| Résilience | Maintenir le réseau en cas de panne | Que se passe-t-il quand un lien tombe ? |

Ces trois notions sont liées, mais différentes : on peut équilibrer sans être résilient (deux liens saturés qui tombent ensemble), et on peut être résilient sans agrégation (un lien de secours qui ne sert qu'en panne).

---

## 3. Importance

- Améliorer les performances.
- Optimiser l'usage de la bande passante achetée.
- Éviter les interruptions de service.
- Assurer la continuité des activités et garantir la disponibilité du réseau.

---

## 4. L'équilibrage en pratique

### 4.1 Répartition par flux, pas par paquet

Le SD-WAN répartit en général par flux (une session TCP ou UDP identifiée par adresses et ports) et non paquet par paquet. Raison : envoyer les paquets d'une même session sur deux liens de latences différentes les ferait arriver dans le désordre, ce qui dégrade TCP et la voix.

Conséquence : un seul flux ne dépasse pas la capacité d'un seul lien. En revanche, avec beaucoup de flux (un magasin en compte des centaines), la répartition remplit bien les deux liens.

### 4.2 Algorithmes courants

| Algorithme | Principe | Usage |
|---|---|---|
| ECMP (Cisco) | Répartition équitable par hachage de flux entre chemins de même coût, jusqu'à 4 par défaut | Liens de capacités proches |
| Pondéré (weight) | Proportion selon un poids, par exemple 70 % et 30 % | Liens de capacités différentes |
| Source IP | Tous les flux d'un même poste sur le même lien | Éviter les changements d'adresse source NAT |
| Selon l'usage | Envoie vers le lien le moins chargé | Maximiser l'utilisation |
| Selon la qualité (SLA) | Choisit le lien le plus conforme ou le meilleur | Applications sensibles |

### 4.3 Exemple de pondération

Deux liens sortants : MPLS 50 Mbit/s et câble 20 Mbit/s.

```
Part du MPLS  = 50 / (50 + 20) = 71 %
Part du câble = 20 / (50 + 20) = 29 %
```

Avec un poids de 5 contre 2, on obtient à peu près ces proportions.

---

## 5. L'agrégation : ce qu'elle apporte et ses limites

L'agrégation additionne les capacités : 50 plus 20 donne 70 Mbit/s de capacité totale pour l'ensemble des flux.

Points d'attention honnêtes :

1. L'agrégation est globale, pas par session. Un transfert unique reste limité à la vitesse du lien choisi pour ce flux (sauf technique de répartition par paquet avec remise en ordre, utilisée dans certaines solutions mais qui ajoute de la complexité).
2. Les liens ne sont pas symétriques : un câble peut offrir 500 Mbit/s en réception mais 20 Mbit/s en émission. Pour le trafic montant vers le datacenter, c'est l'émission qui compte.
3. Latences différentes : mélanger un lien à 10 ms et un à 80 ms nuit aux applications sensibles. L'agrégation s'applique surtout aux flux tolérants.

Techniques complémentaires de fiabilité :

- Duplication de paquets : le même paquet est envoyé sur deux liens, le premier arrivé est gardé. Utile pour la voix, au prix de la bande passante.
- FEC (Forward Error Correction) : envoi de paquets de redondance pour reconstruire les paquets perdus sans retransmission.

---

## 6. La résilience

La résilience est la capacité à continuer de servir malgré une panne. En SD-WAN, elle repose sur :

1. la détection rapide (BFD pour les coupures, SLA pour les dégradations) ;
2. la redistribution automatique des flux sur les liens restants ;
3. des politiques de priorité pour que les applications critiques passent d'abord.

Séquence typique lors de la perte du lien Orange :

```
1. BFD détecte la perte de tunnel (environ 7 s)
2. Le tunnel Orange est retiré des chemins disponibles
3. Les nouveaux flux et les flux existants sont répartis sur SFR
4. La QoS donne la priorité aux caisses et aux paiements
5. À la reprise d'Orange, le trafic est progressivement rééquilibré
```

---

## 7. Dimensionnement : le calcul qui évite les mauvaises surprises

Piège courant : croire que deux liens de 50 et 20 Mbit/s donnent un réseau de 70 Mbit/s résilient. Ce n'est vrai qu'en situation normale.

| Situation | Capacité disponible | Commentaire |
|---|---|---|
| Normale, deux liens | 70 Mbit/s | Tous les flux passent |
| Panne du MPLS | 20 Mbit/s | Le câble doit porter caisses et paiements |
| Panne du câble | 50 Mbit/s | Confortable |

Règle de conception : la somme des trafics critiques doit tenir sur le lien le plus petit. Si caisses, paiements et voix demandent 12 Mbit/s, un lien de 20 Mbit/s convient. Mais la vidéosurveillance (par exemple 15 Mbit/s) devra être limitée ou suspendue pendant la panne, d'où la nécessité de la QoS et de politiques de secours.

Évaluation simple à faire pour chaque site :

```
Besoin critique (voix + caisses + paiements)  <=  capacité du plus petit lien x 0,8
```

Le facteur 0,8 garde une marge d'environ 20 % pour les pics.

---

## 8. Exemples Carrefour

Un magasin Carrefour utilise deux connexions Internet, Orange et SFR. Le SD-WAN répartit le trafic entre les deux liens. Si Orange tombe en panne, le trafic passe automatiquement par SFR. Les caisses continuent ainsi de fonctionner.

Variante avec politique :

- Trafic critique (caisses, paiements) : toujours sur le lien le plus sain, avec secours automatique.
- Trafic de masse (mises à jour, sauvegardes) : réparti sur les deux liens, pondéré selon les capacités.
- Vidéosurveillance : sur le lien le moins cher, suspendue ou bridée en cas de panne.

---

## 9. Mise en oeuvre Cisco et Fortinet

### Cisco

Par défaut, un Edge installe jusqu'à 4 chemins de même coût (ECMP) vers un préfixe. Pour répartir de façon inégale, on règle un poids sur l'interface de transport. La syntaxe ci-dessous est indicative et varie selon la plateforme et la version.

```
vpn 0
 interface ge0/0
  tunnel-interface
   color mpls
   weight 5
 interface ge0/1
  tunnel-interface
   color biz-internet
   weight 2
```

### Fortinet

La règle SD-WAN peut utiliser un mode de répartition de charge.

```
config system sdwan
    config service
        edit 2
            set name "TRAFIC_MASSE"
            set mode load-balance
            set dst "all"
            set hash-mode round-robin
            set priority-members 1 2
        next
    end
end
```

Selon la version, les modes disponibles incluent la répartition par IP source, par poids, par usage de bande passante ou par volume. Consultez la documentation de votre version.

---

## 10. Erreurs fréquentes

1. Confondre capacité totale et capacité résiliente.
2. Penser qu'une grosse copie de fichier utilisera les deux liens en même temps.
3. Répartir la voix sur deux liens de qualité différente.
4. Ignorer l'asymétrie montante et descendante.
5. Oublier que la répartition peut changer l'adresse IP publique de sortie, ce qui perturbe certains services qui lient la session à l'adresse source.

---

## 11. Questions de révision

1. Pourquoi répartit-on par flux et non par paquet ?
   Réponse : pour éviter le désordre des paquets qui nuit à TCP et à la voix.

2. Deux liens de 100 et 30 Mbit/s, quelle pondération approximative ?
   Réponse : environ 77 % et 23 %, soit un poids proche de 10 contre 3.

3. Quelle est la différence entre agrégation et résilience ?
   Réponse : l'agrégation augmente la capacité en situation normale, la résilience maintient le service en cas de panne.

4. Que faut-il vérifier avant de déclarer un site résilient ?
   Réponse : que le trafic critique tient sur le lien survivant le plus petit.

5. Citez deux techniques pour fiabiliser un flux sensible sur un lien instable.
   Réponse : la duplication de paquets et la correction d'erreur (FEC).
