# Questions et réponses : plan de contrôle, résilience et étanchéité

Niveau : étudiants en réseaux et télécoms
Support : Figure 3.3, échange de routes et de clés via OMP (architecture Cisco SD-WAN)

---

## Table des matières

1. Objectifs d'apprentissage
2. Préambule : les acteurs de l'architecture
3. Analyse de la Figure 3.3
4. Question 1 : l'adresse publique de DC1 et l'Edge de Londres
5. Question 2 : perte des deux Controllers
6. Question 3 : étanchéité entre le segment OT et le segment Corporate
7. Exemples Carrefour
8. Formulations courtes pour une copie d'examen
9. Questions complémentaires pour aller plus loin

---

## 1. Objectifs d'apprentissage

- Décrire la séparation entre plan de contrôle et plan de données.
- Expliquer comment un Edge découvre les autres sites sans configuration manuelle.
- Prévoir le comportement d'un réseau SD-WAN quand les Controllers sont injoignables.
- Justifier l'étanchéité de deux segments qui partagent un tunnel.

---

## 2. Préambule : les acteurs de l'architecture

| Composant | Rôle | Remarque |
|---|---|---|
| vBond (orchestrateur) | Premier contact. Authentifie les équipements, indique à chacun où sont les autres Controllers, et l'aide à découvrir sa propre adresse publique | Doit être joignable à une adresse publique |
| vSmart (Controller de contrôle) | Réflecteur de routes OMP. Applique les politiques de contrôle | Souvent deux instances pour la redondance |
| vManage | Gestion, configuration, supervision | Ne participe pas au routage |
| Edge (Londres, DC1) | Applique les politiques, établit les tunnels IPsec, transporte les données | Peut être un routeur Cisco ou un autre équipement selon l'architecture |

Principe fondamental : le plan de contrôle (échange de routes, de clés, de politiques) passe par les Controllers. Le plan de données (trafic utilisateur chiffré) circule directement d'Edge à Edge. Les Edges ne se parlent qu'à travers le Controller pour le contrôle, puis les données circulent directement entre eux.

---

## 3. Analyse de la Figure 3.3

La figure montre un Controller (réflecteur de routes OMP) au sommet et deux Edges : Londres (système 192.168.255.11) et DC1 (système 192.168.255.100). Chaque Edge a deux TLOC : mpls et biz-internet.

Étapes numérotées de la figure :

1. L'Edge de Londres annonce par OMP : son préfixe 10.11.0.0/16 (VPN 10), ses TLOC Londres/mpls et Londres/biz-internet avec leurs adresses privée et publique, et les clés IPsec de chaque TLOC. L'Edge DC1 fait la même chose (1') avec 10.100.0.0/16 (VPN 10).
2. Le Controller applique la politique de contrôle (filtres, préférences, TLOC visibles), puis réfléchit les routes et les TLOC à chaque Edge.
3. Les tunnels IPsec s'établissent par paire de TLOC compatibles (mpls avec mpls, biz-internet avec biz-internet). BFD fonctionne à l'intérieur de ces tunnels.
4. Londres installe 10.100.0.0/16 via les TLOC DC1/mpls et DC1/biz-internet (jusqu'à 4 chemins ECMP par défaut). Les routes sont conservées 12 heures si les Controllers deviennent injoignables (graceful restart OMP).

Légende de la figure : échange de routes et de clés via OMP. Les Edges ne se parlent qu'à travers le Controller pour le contrôle, les données circulent ensuite directement entre eux.

Schéma synthétique :

```
                 [ vSmart : réflecteur OMP ]
                   ^                    ^
      routes, TLOC |                    | routes, TLOC
      et clés      |                    | et clés
                   v                    v
              [Edge Londres]=======[Edge DC1]
                    plan de données (IPsec direct)
```

---

## 4. Question 1

Énoncé : pourquoi l'Edge de Londres n'a-t-il pas besoin de connaître l'adresse publique de l'Edge DC1 avant de parler au Controller ? Qui la lui fournit, et comment est-elle apprise ?

### Réponse développée

Londres n'a besoin de connaître qu'un seul point de départ : l'adresse (ou le nom) du vBond. Tout le reste est découvert dynamiquement.

Déroulement :

1. L'Edge DC1 contacte le vBond. Ce dernier l'authentifie et, en échange, lui apprend l'adresse IP et le port publics vus de l'extérieur, après un éventuel NAT. Le vBond joue donc un rôle proche de celui d'un serveur STUN. Il lui indique aussi l'adresse des vSmart et vManage.
2. DC1 établit une connexion de contrôle avec le vSmart et annonce, par OMP, ses routes, ses TLOC (système IP, couleur, encapsulation, adresses privée et publique) et ses clés de chiffrement.
3. Londres fait la même démarche de son côté, vers le vBond puis le vSmart.
4. Le vSmart, après avoir appliqué sa politique, réfléchit à Londres les TLOC de DC1 : Londres reçoit donc l'adresse publique de DC1, sans l'avoir jamais configurée.
5. Londres peut alors ouvrir un tunnel IPsec directement vers cette adresse publique et lancer BFD.

Réponse à "qui la lui fournit" : le Controller (vSmart), qui la reflète à partir de l'annonce OMP de DC1. Réponse à "comment est-elle apprise" : DC1 l'a découverte via le vBond puis l'a annoncée par OMP.

Avantages :

- aucune adresse de site distant à saisir manuellement (le plan est zéro configuration de topologie) ;
- fonctionne derrière un NAT, car l'adresse annoncée est l'adresse publique vue de l'extérieur ;
- l'ajout d'un nouveau site n'oblige pas à modifier les 500 autres.

### Remarque sur la formulation du cours

Votre réponse de cours dit que le contrôleur "récupère les informations grâce à OMP et les transmet". Elle est correcte dans l'esprit. Pour l'enrichir : précisez que l'adresse publique est apprise par l'Edge DC1 grâce au vBond, puis annoncée en OMP, puis réfléchie par le vSmart.

---

## 5. Question 2

Énoncé : un Edge voit ses deux Controllers injoignables pendant une heure. Listez ce qui continue de fonctionner et ce qui ne fonctionne plus. En quoi la réponse change-t-elle si la panne dure plus de 12 heures (valeur par défaut Cisco) ?

### Principe

Le plan de données est autonome du plan de contrôle une fois établi. Les routes OMP reçues sont conservées dans une mémoire grâce au graceful restart OMP. La durée par défaut est de 12 heures (43 200 secondes) et peut être augmentée, jusqu'à 7 jours.

### Pendant la première heure

Ce qui continue de fonctionner :

| Élément | Explication |
|---|---|
| Tunnels IPsec existants | Le chiffrement et les clés déjà échangées restent valides |
| Sessions BFD | Elles tournent entre Edges, sans Controller |
| Trafic inter-sites déjà routé | Les routes OMP en mémoire sont toujours utilisées |
| Sélection de chemin par SLA et bascule sur panne | Basée sur les mesures locales BFD |
| Politiques locales | QoS, listes de contrôle d'accès, NAT, sortie Internet locale |
| Routage local du LAN | Protocoles locaux comme OSPF ou BGP |
| Trafic invité vers Internet | Sortie locale, indépendante du WAN |

Ce qui ne fonctionne plus :

| Élément | Explication |
|---|---|
| Création de tunnels vers de nouveaux TLOC | Il faut des informations venant du Controller |
| Mises à jour des routes | Nouveaux sites, changements de préfixes, retraits, non reçus |
| Changement de politique centralisée | Les nouvelles politiques ne sont pas distribuées |
| Configuration et supervision depuis vManage | Pas de gestion ni de télémétrie |
| Intégration d'un nouvel équipement | L'authentification passe par les Controllers |
| Renouvellement des clés | Les nouvelles clés ne peuvent pas être distribuées |

Conséquence pratique : le réseau est figé. Il survit tel quel, y compris à une panne d'un lien (BFD continue), mais il ne peut plus évoluer.

### Si la panne dépasse 12 heures

Le délai de graceful restart OMP expire. L'Edge supprime les routes distantes et les clés associées reçues d'OMP. Le trafic vers les autres sites n'a plus de route et les tunnels tombent. Le site n'a plus que sa connectivité locale et sa sortie Internet locale.

Pour aller plus loin : sur un réseau où l'on redoute une panne prolongée des Controllers, on peut augmenter la valeur du graceful restart, et on déploie des Controllers redondants dans des lieux et des régions différents.

### Chronologie

```
t = 0           Perte des Controllers
t = 0 à 12 h    Réseau figé mais fonctionnel (graceful restart actif)
t > 12 h        Routes OMP et clés expirées, trafic inter-sites interrompu
retour Ctrl.    Reconnexion, nouvelles routes, tunnels rétablis
```

---

## 6. Question 3

Énoncé : pourquoi un paquet du segment OT de Berlin ne peut-il pas atteindre le segment Corporate de DC1, même si les deux sont transportés dans le même tunnel ? Citez les trois mécanismes qui garantissent l'étanchéité.

### Réponse du cours (formulation attendue)

Chaque réseau possède une séparation logique. Elle repose sur trois mécanismes :

1. Les VRF (ou VPN) séparent les tables de routage.
2. Les identifiants de VPN dans OMP permettent d'identifier chaque segment (chaque route est associée à son VPN et chaque paquet porte son label).
3. Les clés et politiques IPsec empêchent les communications non autorisées.

### Explication détaillée

1. Table de routage séparée : dans la VRF OT de Berlin, il n'existe pas de route vers les préfixes de la VRF Corporate de DC1. Un paquet sans route est abandonné.
2. Identifiant de VPN dans OMP : le vSmart sait que les routes OT appartiennent au VPN OT. Il peut ne les annoncer qu'aux sites qui possèdent ce VPN. À l'arrivée, le label du paquet impose de le déposer dans la VRF correspondante, et jamais dans une autre.
3. IPsec et politiques : IPsec garantit que seuls les Edges authentifiés participent à l'overlay et que le contenu est confidentiel et intègre. Les politiques ajoutent le filtrage explicite.

Précision pour approfondir : l'étanchéité entre deux segments vient surtout des VRF, du label de VPN et du filtrage des routes par le Controller. IPsec, lui, protège le transport (chiffrement, authentification) entre les Edges. C'est la combinaison qui forme la défense en profondeur. Si un échange entre OT et Corporate est nécessaire, il se fait par un pare-feu inter-VRF qui applique une politique explicite.

---

## 7. Exemples Carrefour

### Exemple 1 : communication Londres vers DC1

Un magasin Carrefour à Londres souhaite accéder au serveur central DC1 pour consulter les applications de gestion des stocks. Le Controller partage les routes de DC1 avec l'Edge de Londres grâce à OMP. L'Edge connaît ainsi le chemin pour joindre le datacenter sans avoir besoin de configurer manuellement son adresse IP.

### Exemple 2 : perte des Controllers

Un magasin Carrefour à Berlin perd la connexion avec ses deux Controllers. Pendant une heure, les caisses continuent de communiquer avec le datacenter DC1 grâce aux tunnels IPsec déjà établis. Le magasin ne peut plus créer de nouveaux tunnels ni recevoir de nouvelles routes. Si la panne dépasse 12 heures, les routes distantes sont supprimées et les communications avec DC1 peuvent être interrompues.

### Exemple 3 : isolation OT et Corporate

Un technicien du magasin de Berlin utilise le réseau OT pour gérer les équipements industriels et les systèmes de réfrigération. Il essaie d'accéder à un serveur du réseau Corporate de DC1. L'accès est bloqué car les deux réseaux sont séparés par des VRF, des identifiants de VPN dans OMP et des politiques IPsec. Les équipements industriels restent protégés contre les accès non autorisés.

---

## 8. Formulations courtes pour une copie d'examen

Question 1 : L'Edge de Londres n'a pas besoin de connaître l'adresse de DC1 car celle-ci lui est fournie par le Controller, qui la reçoit de DC1 par OMP (TLOC) et la réfléchit. DC1 a lui même appris son adresse publique grâce au vBond.

Question 2 : Pendant la panne, les tunnels IPsec, BFD et le routage existant continuent grâce au graceful restart OMP, mais plus aucune nouvelle route, politique, clé ou création de tunnel. Au delà de 12 heures, les routes distantes expirent et le trafic inter-sites s'arrête.

Question 3 : Les trois mécanismes sont les VRF (tables séparées), les identifiants de VPN dans OMP (routes et labels par segment) et IPsec avec ses politiques (confidentialité et filtrage).

---

## 9. Questions complémentaires pour aller plus loin

1. Pourquoi déploie-t-on au moins deux vSmart ?
   Réponse : pour éviter que la perte d'un Controller ne fige les mises à jour de contrôle. Chaque Edge se connecte à plusieurs Controllers.

2. Pourquoi les Edges ne se connectent-ils pas tous les uns aux autres en configuration manuelle ?
   Réponse : le Controller distribue les TLOC et les clés. Cela évite un maillage manuel de n au carré tunnels à configurer.

3. Que signifie ECMP dans l'étape 4 de la figure ?
   Réponse : Equal-Cost Multi-Path, la répartition du trafic sur plusieurs chemins de même coût, ici jusqu'à 4 par défaut.

4. Quel est l'intérêt de limiter les TLOC visibles dans la politique de contrôle ?
   Réponse : pour imposer une topologie (hub and spoke, par exemple) sans configurer chaque Edge.
