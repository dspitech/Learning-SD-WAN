# La redondance des transports dans un SD-WAN

Niveau : étudiants en réseaux et télécoms
Fil rouge : un magasin Carrefour relié à un datacenter par deux opérateurs

---

## Table des matières

1. Objectifs d'apprentissage
2. Définition
3. Pourquoi la redondance est vitale
4. Les niveaux de redondance
5. Diversité des opérateurs et des chemins physiques
6. Modes de fonctionnement : actif/actif et actif/secours
7. Calcul de disponibilité
8. Exemples Carrefour : magasin, petit magasin, cloud
9. Mise en oeuvre Cisco et Fortinet
10. Pièges fréquents
11. Questions de révision

---

## 1. Objectifs d'apprentissage

- Définir la redondance des transports et expliquer son intérêt métier.
- Distinguer redondance des liens, des équipements et des chemins physiques.
- Calculer la disponibilité d'une architecture à deux liens.
- Choisir un mode actif/actif ou actif/secours selon le contexte.
- Identifier les points de défaillance communs qui annulent la redondance.

---

## 2. Définition

La redondance des transports consiste à relier un site à son réseau par plusieurs connexions, de préférence chez des fournisseurs différents, afin de garantir la continuité du réseau en cas de panne d'un lien.

Elle assure la disponibilité du réseau et évite les coupures.

Le mot "transport" désigne ici l'underlay : MPLS, fibre Internet, câble, xDSL, 4G, 5G. Le SD-WAN construit par dessus un overlay de tunnels IPsec et sait utiliser plusieurs transports en même temps, sans que les applications le remarquent.

---

## 3. Pourquoi la redondance est vitale

Dans la distribution, une caisse qui ne peut plus valider un paiement est une perte de chiffre d'affaires immédiate. Un magasin de taille moyenne peut perdre plusieurs milliers d'euros par heure de panne totale. Les liens WAN ne sont pas fiables à 100 % : travaux de voirie, coupure de fibre, panne de l'opérateur, saturation, erreur de configuration chez le fournisseur.

Avant le SD-WAN, on utilisait un lien MPLS principal avec un lien de secours souvent inutilisé et coûteux. Le SD-WAN permet d'utiliser tous les liens, y compris des accès Internet bon marché, avec une bascule automatique.

---

## 4. Les niveaux de redondance

```
Niveau 1 : le lien            deux accès (Orange et SFR)
Niveau 2 : le chemin physique deux itinéraires de fibre distincts
Niveau 3 : l'équipement       deux Edges en haute disponibilité
Niveau 4 : l'énergie          onduleur, double alimentation
Niveau 5 : le datacenter      deux hubs ou deux régions cloud
```

| Niveau | Panne couverte | Mécanisme typique |
|---|---|---|
| Lien | Panne d'un opérateur | Deux transports SD-WAN, bascule par BFD et SLA |
| Chemin physique | Fibre coupée par des travaux | Collectes distinctes, pas de fourreau partagé |
| Équipement | Panne matérielle de l'Edge | Paire HA, VRRP côté LAN |
| Énergie | Coupure électrique | Onduleur, double alimentation |
| Datacenter | Perte du site central | Deux hubs, ou deux FortiGate-VM dans le cloud |

---

## 5. Diversité des opérateurs et des chemins physiques

Avoir deux contrats n'est pas suffisant. Il faut une vraie diversité :

- Opérateurs différents : par exemple Orange Business et SFR Business. Autres exemples de fournisseurs d'entreprise : Bouygues Telecom Entreprises, Colt, Vodafone.
- Chemins physiques différents : deux opérateurs peuvent en réalité louer la même fibre ou passer dans le même fourreau. Une pelleteuse coupe alors les deux d'un coup. On parle de risque partagé (shared risk, SRLG).
- Technologies différentes : fibre et câble, ou fibre et 5G. Elles ne tombent pas pour les mêmes raisons.
- Points d'entrée différents : éviter que les deux liens arrivent par la même baie et le même switch.

Question à poser à l'opérateur : "Quel est l'itinéraire de collecte de mon accès et partage-t-il des infrastructures avec l'autre ?"

---

## 6. Modes de fonctionnement

### Actif/secours

Un lien porte tout le trafic, l'autre attend. Simple, mais le lien de secours est payé pour rien et sa qualité n'est pas vérifiée en continu si on ne le sonde pas.

Cas typique : une connexion 4G ou 5G pour un petit magasin. Elle est lente et facturée au volume, on ne l'utilise qu'en cas de panne.

### Actif/actif

Les deux liens portent du trafic en même temps. Le SD-WAN répartit selon l'application, la qualité mesurée ou la charge. Avantages : on rentabilise les deux accès, et on sait que les deux fonctionnent. Condition : dimensionner chaque lien pour que le trafic critique tienne sur un seul si l'autre tombe (voir le document sur l'équilibrage).

---

## 7. Calcul de disponibilité

Pour deux liens indépendants de disponibilités A1 et A2 :

```
A_total = 1 - (1 - A1) x (1 - A2)
```

Exemple : deux accès à 99,5 % chacun.

```
Indisponibilité d'un lien : 1 - 0,995 = 0,005
A_total = 1 - (0,005 x 0,005) = 1 - 0,000025 = 0,999975 = 99,9975 %
```

| Architecture | Disponibilité | Indisponibilité annuelle approximative |
|---|---|---|
| Un seul lien à 99,5 % | 99,5 % | 8760 h x 0,005 = environ 43,8 heures |
| Deux liens indépendants à 99,5 % | 99,9975 % | 525 600 min x 0,000025 = environ 13 minutes |

Limites de ce calcul :

- Il suppose l'indépendance des pannes. Si les deux liens partagent un fourreau, la vraie disponibilité est bien inférieure.
- Il ne tient pas compte du temps de bascule. Avec BFD à 7 secondes de détection, chaque panne coûte quelques secondes, ce qui reste négligeable devant les heures économisées.
- L'équipement Edge, s'il est unique, devient le nouveau maillon faible.

---

## 8. Exemples Carrefour

### Magasin standard

Un magasin utilise deux connexions Internet : Orange et SFR, avec des chemins physiques différents. Si la première connexion tombe en panne, le SD-WAN bascule automatiquement sur la deuxième. Les caisses et les paiements continuent sans intervention humaine.

### Petit magasin de proximité

Deux accès filaires coûtent cher pour un petit point de vente. Solution fréquente : un accès fibre principal plus un accès 4G ou 5G de secours. Le SD-WAN envoie les flux critiques (caisses, paiements) sur le secours en cas de panne et suspend les flux volumineux (vidéo, sauvegardes) pour ne pas consommer le forfait de données.

### Dans le cloud

Carrefour peut déployer deux équipements SD-WAN virtuels (par exemple FortiGate-VM) dans le cloud AWS ou Azure, idéalement dans deux zones de disponibilité. Si l'un tombe en panne, le deuxième prend le relais pour assurer la continuité des services. Les magasins établissent des tunnels vers les deux et choisissent selon la qualité mesurée.

```
Magasin  ===(Orange)===+===> Hub cloud A (zone 1)  <--- actif
   |                   |
   +=====(SFR)=========+===> Hub cloud B (zone 2)  <--- relais
```

---

## 9. Mise en oeuvre Cisco et Fortinet

### Cisco : les couleurs de TLOC

Chaque lien est identifié par une couleur : mpls, biz-internet, public-internet, lte, etc. Les couleurs privées (mpls) ne forment des tunnels qu'avec la même couleur, les couleurs publiques peuvent se connecter entre elles. Pour un lien de secours, on peut utiliser l'option restrict ou une préférence basse afin qu'il ne porte du trafic qu'en dernier recours.

```
vpn 0
 interface ge0/0
  ip address 203.0.113.2/30
  tunnel-interface
   encapsulation ipsec
   color mpls restrict
 !
 interface ge0/1
  ip address 198.51.100.2/30
  tunnel-interface
   encapsulation ipsec
   color biz-internet
```

### Fortinet : membres SD-WAN

Les deux interfaces WAN sont ajoutées comme membres de la zone SD-WAN, chacune avec une sonde de santé (health check).

```
config system sdwan
    set status enable
    config zone
        edit "virtual-wan-link"
        next
    end
    config members
        edit 1
            set interface "wan1"
            set gateway 203.0.113.1
        next
        edit 2
            set interface "wan2"
            set gateway 198.51.100.1
            set cost 10
        next
    end
end
```

Ici l'attribut cost sert à exprimer une préférence : le membre au coût plus élevé n'est choisi que si les règles de qualité ou de disponibilité l'exigent.

---

## 10. Pièges fréquents

1. Deux opérateurs, un seul fourreau : la redondance n'existe que sur le papier.
2. Un seul Edge : le routeur devient le point unique de défaillance. Prévoir une paire HA pour les sites critiques.
3. Ne pas sonder le lien de secours : on découvre qu'il est mort le jour de la panne.
4. Secours sous-dimensionné : la 4G à 20 Mbit/s ne porte pas la vidéosurveillance. Il faut une politique qui réserve le secours aux flux critiques.
5. Facturation au volume du lien de secours non maîtrisée.
6. Oublier la redondance côté datacenter : des magasins redondants vers un hub unique restent dépendants de ce hub.

---

## 11. Questions de révision

1. Pourquoi deux liens d'un même opérateur ne constituent-ils pas une vraie redondance ?
   Réponse : une panne chez l'opérateur, ou un risque partagé sur le réseau, affecte les deux. La diversité d'opérateur et de chemin est la condition.

2. Calculez la disponibilité de deux liens indépendants à 99 % chacun.
   Réponse : 1 - (0,01 x 0,01) = 0,9999 soit 99,99 %.

3. Quel mode choisir pour un petit magasin avec accès 5G de secours ?
   Réponse : actif/secours, avec une politique qui réserve la 5G aux flux critiques.

4. Que doit-on faire dans le cloud pour la redondance ?
   Réponse : déployer deux Edges virtuels, de préférence dans deux zones de disponibilité, avec des tunnels depuis les sites vers les deux.

5. Quel mécanisme détecte la coupure franche d'un lien ?
   Réponse : les sondes BFD, qui détectent une perte de lien en quelques secondes.
