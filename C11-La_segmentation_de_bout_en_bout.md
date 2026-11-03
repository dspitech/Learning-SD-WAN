# La segmentation de bout en bout dans un SD-WAN

Niveau : étudiants en réseaux et télécoms (licence, BTS, bachelor, master)
Fil rouge : les magasins Carrefour, le site de Berlin et le datacenter DC1
Technologies couvertes : Cisco SD-WAN (VPN, OMP, TLOC) et Fortinet Secure SD-WAN (VRF, vpn-id-ipip)

---

## Table des matières

1. Objectifs d'apprentissage
2. Le problème : le réseau plat
3. Définition
4. Vocabulaire indispensable
5. Analyse du schéma Berlin et DC1
6. Le trajet d'un paquet, étape par étape
7. Où se situe réellement l'isolation
8. Mise en oeuvre côté Cisco et côté Fortinet
9. Exemple complet : un magasin Carrefour
10. Matrice de flux
11. Idées reçues et erreurs fréquentes
12. Avantages
13. Résumé en deux lignes
14. Questions de révision

---

## 1. Objectifs d'apprentissage

À la fin de ce document, vous serez capable de :

- définir la segmentation de bout en bout et de la distinguer d'une simple segmentation VLAN ;
- expliquer comment plusieurs réseaux logiques traversent un seul tunnel IPsec sans se mélanger ;
- décrire le rôle des VRF, des identifiants de VPN, d'OMP et du pare-feu inter-VRF ;
- justifier le choix d'une sortie Internet locale pour un réseau invité ;
- construire une matrice de flux autorisés et interdits pour un site de distribution.

---

## 2. Le problème : le réseau plat

Sans segmentation, tous les équipements d'un magasin partagent le même réseau IP : les caisses, les terminaux de paiement, les caméras, les ordinateurs du personnel, les imprimantes et les bornes Wi-Fi.

Conséquence : si un seul équipement faible est compromis (une imprimante jamais mise à jour, un PC infecté, un point d'accès Wi-Fi mal configuré), l'attaquant se trouve dans le même domaine de diffusion et le même plan d'adressage que les systèmes sensibles. Il peut scanner, rebondir et atteindre les caisses ou le serveur de paiement. C'est le déplacement latéral (lateral movement).

```
Réseau plat dans un magasin

  [Caisse] [TPE] [Caméra] [PC] [Imprimante compromise] [Wi-Fi invité]
      \       \      |      /           /                  /
       +--------------------------------------------------+
                      Même réseau, mêmes routes
```

Le WAN aggrave le problème : si ce réseau plat est simplement prolongé jusqu'au datacenter, le mélange se propage à toute l'entreprise.

---

## 3. Définition

La segmentation de bout en bout consiste à séparer les différents types de trafic dans des réseaux logiques indépendants, depuis le site de départ jusqu'au site de destination, y compris pendant la traversée du WAN, même si tous ces réseaux utilisent la même infrastructure physique et parfois le même tunnel.

Trois idées à retenir :

- Indépendants : chaque segment a sa propre table de routage, son propre plan d'adressage possible (les plages peuvent même se chevaucher).
- De bout en bout : la séparation n'est pas rompue au passage du LAN vers le WAN, ni du WAN vers le datacenter.
- Même infrastructure : on ne déploie pas un lien physique ou un tunnel par service. On virtualise.

---

## 4. Vocabulaire indispensable

| Terme | Définition simple |
|---|---|
| VLAN | Séparation au niveau 2 (Ethernet) sur le réseau local. Ne traverse pas le WAN à lui seul. |
| VRF (Virtual Routing and Forwarding) | Table de routage indépendante sur un même équipement. Deux VRF ne se voient pas par défaut. |
| VPN de service (Cisco) | Nom Cisco SD-WAN d'un segment. Équivalent fonctionnel d'une VRF. Numéros de 1 à 65530 pour les services, avec VPN 0 pour le transport et VPN 512 pour la gestion. |
| Overlay | Réseau logique construit au dessus des liens physiques, ici les tunnels IPsec du SD-WAN. |
| Underlay (transport) | Les liens réels : MPLS, fibre Internet, câble, 4G, 5G. |
| Edge | Équipement SD-WAN d'un site (routeur Cisco, FortiGate). |
| Controller | Cerveau centralisé du SD-WAN. Dans Cisco : vSmart (contrôle), vBond (orchestration), vManage (gestion). |
| OMP (Overlay Management Protocol) | Protocole Cisco qui transporte routes, TLOC et clés entre les Edges et les Controllers. |
| TLOC (Transport Locator) | Point d'accès au transport : système IP, couleur (mpls, biz-internet), encapsulation. |
| Label de VPN | Identifiant du segment porté dans chaque paquet du tunnel pour que le site distant sache dans quelle VRF le déposer. |
| Pare-feu inter-VRF | Équipement qui, et seulement lui, autorise un échange contrôlé entre deux segments. |

---

## 5. Analyse du schéma Berlin et DC1

Le schéma de votre cours représente deux sites reliés par un seul tunnel IPsec.

```
   BERLIN                                                  DC1
 +-------------------+                          +------------------------+
 | Bureaux   VRF 10  |--+                    +--| ERP, AD        VRF 10  |
 | Automates VRF 20  |--+-[Edge]=============[Edge]-+ SCADA         VRF 20  |
 | Wi-Fi invité VRF 30|-+      un seul tunnel  +--| Pare-feu inter-VRF     |
 +-------------------+     |       IPsec         +------------------------+
                           |
                           +--> Internet (sortie locale, VRF 30 seulement)
```

Lecture détaillée :

1. Côté Berlin, trois VRF : bureaux (10), automates industriels (20), Wi-Fi invité (30).
2. Côté DC1, deux VRF correspondantes : 10 (ERP et annuaire Active Directory) et 20 (SCADA), plus un pare-feu inter-VRF.
3. Un seul tunnel IPsec entre les deux Edges transporte les VRF 10 et 20. Chaque paquet porte le label de sa VRF.
4. Le Controller n'annonce les routes de la VRF 20 qu'aux sites qui possèdent cette VRF. Un site sans VRF 20 ne reçoit même pas ces routes.
5. La VRF 30 (invités) ne traverse pas le WAN vers DC1. Elle sort directement vers Internet par une sortie locale (local breakout, ou DIA, Direct Internet Access). Ainsi un invité ne consomme ni la bande passante WAN ni l'accès au datacenter.

Pourquoi ce choix pour les invités : un visiteur n'a aucun besoin légitime des applications internes. Le sortir localement réduit la surface d'attaque et décharge le WAN.

---

## 6. Le trajet d'un paquet, étape par étape

Exemple : un automate de la VRF 20 de Berlin envoie une donnée vers un serveur SCADA de la VRF 20 de DC1.

```
Automate (VRF 20)
   |  1. paquet IP normal
   v
Edge Berlin
   |  2. consultation de la table de routage de la VRF 20 uniquement
   |  3. la route vers DC1 a été apprise par OMP pour le VPN 20
   |  4. ajout du label VPN 20, puis chiffrement IPsec
   v
Underlay (MPLS ou Internet), le contenu est chiffré
   v
Edge DC1
   |  5. déchiffrement IPsec, lecture du label VPN 20
   |  6. dépôt du paquet dans la VRF 20 de DC1
   v
Serveur SCADA (VRF 20)
```

Si ce même automate essayait d'atteindre un serveur de la VRF 10, l'Edge Berlin ne trouverait aucune route vers la VRF 10 dans la table de la VRF 20. Le paquet est abandonné. La seule voie légitime est de passer par le pare-feu inter-VRF qui applique une politique.

---

## 7. Où se situe réellement l'isolation

L'isolation repose sur plusieurs couches complémentaires. Il est important de ne pas les confondre.

| Couche | Mécanisme | Ce qu'il garantit |
|---|---|---|
| Équipement | VRF (ou VPN de service) | Tables de routage séparées : pas de route entre segments |
| Plan de contrôle | Identifiant de VPN dans OMP et politique de contrôle du Controller | Les routes d'un segment ne sont annoncées qu'aux sites autorisés |
| Plan de données | Label de VPN dans chaque paquet du tunnel | Le site distant dépose le paquet dans la bonne VRF |
| Transport | IPsec (clés et authentification) | Confidentialité et intégrité entre les Edges |
| Inter-segments | Pare-feu et politiques | Les échanges entre segments sont explicites et journalisés |

Précision utile pour un examen : IPsec protège le transport entre sites (chiffrement, authentification). La séparation entre segments est assurée par les VRF, le label de VPN et le filtrage des routes par OMP. Dans la formulation condensée du cours, on cite souvent trois mécanismes : les VRF, les identifiants de VPN dans OMP, et les clés et politiques IPsec. Vous pouvez reprendre cette formulation en ajoutant la nuance ci-dessus.

---

## 8. Mise en oeuvre côté Cisco et côté Fortinet

### Comparaison

| Besoin | Cisco SD-WAN | Fortinet Secure SD-WAN |
|---|---|---|
| Segment | VPN de service (numéro) | VRF (interface assignée à une VRF) |
| Séparation locale | Interface placée dans un VPN, sous-interface par VLAN | VLAN, interface avec attribut vrf |
| Distribution des routes | OMP via vSmart | BGP, y compris MP-BGP VPNv4 selon la conception |
| Transport du segment dans le tunnel | Label de VPN dans l'encapsulation | Encapsulation vpn-id-ipip dans l'overlay IPsec |
| Contrôle entre segments | Politiques centralisées et pare-feu | Politiques de pare-feu (firewall policies) |

Point technique : avec Fortinet, plusieurs VRF peuvent partager un même overlay IPsec grâce à l'encapsulation vpn-id-ipip. On évite ainsi de créer un tunnel par segment. La syntaxe exacte dépend de la version de FortiOS, consultez la documentation officielle Fortinet "Segmentation over single overlay" correspondant à votre version (7.6 pour le cours).

### Exemple Cisco (extrait de configuration d'un Edge)

```
vpn 10
 interface ge0/1
  ip address 10.11.10.1/24
  no shutdown
 !
!
vpn 20
 interface ge0/2
  ip address 10.11.20.1/24
  no shutdown
 !
!
vpn 30
 interface ge0/3
  ip address 192.168.30.1/24
  no shutdown
 !
 ip route 0.0.0.0/0 vpn 0
!
```

Explication : la dernière ligne de la VPN 30 envoie le trafic invité vers le VPN 0 (transport), donc directement vers Internet. Il faut en plus un NAT sur l'interface de transport pour que les adresses privées soient traduites. Les VPN 10 et 20 n'ont pas cette route : leur trafic suit les routes apprises par OMP.

### Exemple Fortinet (principe)

```
config system interface
    edit "vlan10"
        set vrf 10
    next
    edit "vlan20"
        set vrf 20
    next
end
```

Ensuite, on écrit des politiques de pare-feu explicites. Une politique relie une interface source à une interface de destination, et sans politique, le trafic entre deux VRF est refusé.

---

## 9. Exemple complet : un magasin Carrefour

Attention : il s'agit d'un exemple pédagogique, pas de l'architecture réelle de Carrefour.

| Segment | Contenu | Exemple de VLAN et VRF |
|---|---|---|
| 1. Caisses | Terminaux de caisse, remontée des transactions | VLAN 10, VRF CAISSES |
| 2. Paiement bancaire | Terminaux de paiement, échanges avec les services bancaires | VLAN 20, VRF PAIEMENT |
| 3. Vidéosurveillance | Caméras IP, serveurs d'enregistrement | VLAN 30, VRF VIDEO |
| 4. Administration | Postes des employés, gestion des stocks, applications internes | VLAN 40, VRF ADMIN |
| 5. Invités (optionnel) | Wi-Fi visiteurs | VLAN 50, VRF INVITES, sortie Internet locale |

Parcours d'une caisse vers le serveur de gestion des ventes :

1. La caisse appartient au VLAN 10, associé à la VRF CAISSES.
2. L'Edge (FortiGate ou routeur) identifie le segment et applique les règles de sécurité.
3. Le trafic entre dans un tunnel IPsec vers le datacenter. L'identité du segment est conservée.
4. Au datacenter, l'Edge distant reconnaît la VRF CAISSES et dirige le trafic vers le serveur de ventes.
5. Les réponses suivent le même principe de séparation logique.

Résultat : le trafic des caisses ne se mélange ni avec celui des caméras, ni avec celui des terminaux de paiement.

---

## 10. Matrice de flux

| Source | Destination | Décision |
|---|---|---|
| Caisses | Serveur de ventes | Autoriser |
| Paiement | Serveur bancaire | Autoriser |
| Caméras | Serveur vidéo | Autoriser |
| Caméras | Caisses | Bloquer |
| Wi-Fi visiteurs | Réseau administratif | Bloquer |
| Administration | Caisses (supervision) | Autoriser sous conditions (ports et machines précis, journalisé) |

Ces règles sont des exemples pédagogiques. Une bonne matrice suit le principe du moindre privilège : tout est interdit sauf ce qui est explicitement nécessaire.

---

## 11. Idées reçues et erreurs fréquentes

1. "Un VLAN suffit." Faux. Un VLAN sépare au niveau du LAN du magasin. Il ne dit rien sur le WAN. Sans VRF et sans transport du label, la séparation s'arrête à la porte du magasin.
2. "Il faut un tunnel IPsec par segment." Faux. Le principe est justement de transporter plusieurs segments dans un seul tunnel en étiquetant les paquets.
3. "Segmenter, c'est sécuriser." Incomplet. La segmentation limite la propagation d'un incident, mais il faut aussi un filtrage entre segments, de la supervision et des correctifs.
4. "Le réseau invité doit passer par le datacenter." Faux et coûteux. La sortie locale est la bonne pratique.
5. Oublier le plan d'adressage : deux segments peuvent légitimement utiliser la même plage IP, grâce aux VRF, mais cela complique le diagnostic. Documentez.
6. Fuite de routes (route leaking) mal contrôlée entre VRF : elle recrée un réseau plat sans qu'on s'en aperçoive.

---

## 12. Avantages

- Sécurité : un équipement compromis ne peut pas atteindre librement les autres segments.
- Disponibilité : un flot vidéo intense n'écrase pas les caisses, car on peut appliquer une politique par segment.
- Performance : la QoS s'applique par application ou par segment.
- Administration : les politiques sont définies une fois, de façon centralisée, et appliquées à des centaines de magasins.
- Conformité : isoler le segment de paiement facilite les audits (par exemple PCI DSS).

---

## 13. Résumé en deux lignes

Le site de Berlin et le datacenter DC1 utilisent un seul tunnel IPsec pour transporter plusieurs réseaux (VRF 10, 20 et 30), tout en gardant chaque trafic séparé. Chaque paquet conserve son identifiant de VRF, ce qui garantit l'isolation, tandis que le réseau invité (VRF 30) sort directement vers Internet.

---

## 14. Questions de révision

1. Quelle est la différence entre une segmentation VLAN et une segmentation de bout en bout ?
   Réponse : le VLAN sépare localement au niveau 2. La segmentation de bout en bout maintient la séparation à travers le WAN grâce aux VRF et au transport du label de segment dans l'overlay.

2. Pourquoi un seul tunnel IPsec suffit-il pour plusieurs VRF ?
   Réponse : chaque paquet est étiqueté avec l'identifiant de sa VRF. Le site distant lit ce label et dépose le paquet dans la bonne VRF.

3. Pourquoi la VRF invitée sort-elle localement ?
   Réponse : les invités n'ont pas besoin des applications internes. La sortie locale réduit le risque et économise la bande passante du WAN.

4. Que se passe-t-il si un automate de la VRF 20 envoie un paquet à une adresse de la VRF 10 ?
   Réponse : il n'existe pas de route correspondante dans la VRF 20. Le paquet est abandonné, sauf autorisation explicite via le pare-feu inter-VRF.

5. Citez trois mécanismes qui renforcent l'étanchéité.
   Réponse : VRF, identifiant de VPN dans OMP (avec filtrage des routes par le Controller), politiques et clés IPsec complétées par le pare-feu inter-VRF.
