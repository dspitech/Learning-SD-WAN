# SW02. Transports, overlay et underlay

| | |
|---|---|
| **Partie** | A. Fondamentaux du SD-WAN |
| **Durée indicative** | 1 semaine (8 h : cours, lab WireGuard, mesures) |
| **Prérequis** | SW01 ; bases IPsec, NAT, MTU ; Linux en ligne de commande |
| **Niveau** | Intermédiaire |
| **Lien examen** | ENCOR (SD-WAN), ENSDWI (transports, NAT, BFD) |
| **Labs associés** | T1 (overlay WireGuard avec mesure de qualité) |

> Module précédent : [SW01](../SW01-du-wan-au-sd-wan/SW01-du-wan-au-sd-wan.md) | Module suivant : [SW03](../SW03-fonctions-cles/SW03-fonctions-cles.md)

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [L'underlay : les transports](#2-lunderlay--les-transports)
3. [Les métriques de qualité](#3-les-métriques-de-qualité)
4. [L'overlay : IPsec, GRE, VXLAN, WireGuard](#4-loverlay--ipsec-gre-vxlan-wireguard)
5. [Traversée de NAT](#5-traversée-de-nat)
6. [MTU et MSS](#6-mtu-et-mss)
7. [Mesure de qualité : BFD et sondes](#7-mesure-de-qualité--bfd-et-sondes)
8. [Haute disponibilité des liens](#8-haute-disponibilité-des-liens)
9. [Interconnexion avec l'existant](#9-interconnexion-avec-lexistant)
10. [Lab guidé T1 : overlay WireGuard et mesures](#10-lab-guidé-t1--overlay-wireguard-et-mesures)
11. [Exercices](#11-exercices)
12. [Quiz](#12-quiz)
13. [Aide-mémoire](#13-aide-mémoire)
14. [Pour aller plus loin](#14-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Comparer les transports (MPLS, DIA, haut débit grand public, 4G/5G, satellite, Metro Ethernet).
- Interpréter débit, délai, gigue, perte, disponibilité.
- Expliquer le fonctionnement des tunnels IPsec, GRE, VXLAN et WireGuard.
- Prévoir l'effet d'un NAT et d'une MTU insuffisante sur un tunnel.
- Mesurer la qualité d'un chemin avec BFD et des sondes actives.
- Réaliser un overlay chiffré entre deux sites et mesurer son comportement.

---

## 2. L'underlay : les transports

### 2.1 Tableau comparatif

| Transport | Délai typique | Gigue | Perte | Disponibilité contractuelle | Délai de livraison | Coût relatif | Remarques |
|---|---|---|---|---|---|---|---|
| **MPLS** | 10 à 40 ms (national) | Faible | < 0,1 % | 99,9 à 99,99 % avec GTR | 30 à 90 jours | Élevé | Classes de service (CoS) possibles |
| **DIA** (accès Internet dédié) | 10 à 40 ms | Faible à moyenne | < 0,5 % | 99,5 à 99,9 % avec GTR | 15 à 60 jours | Moyen | Débit symétrique, IP fixe |
| **Fibre grand public pro** | 8 à 30 ms | Moyenne | Variable | Meilleur effort | 7 à 30 jours | Faible | Pas de GTR, asymétrie possible |
| **4G / 5G** | 30 à 80 ms (4G), 15 à 40 ms (5G) | Moyenne à élevée | 0,5 à 2 % | Meilleur effort | Immédiat | Moyen (volume) | NAT opérateur (CGNAT) fréquent |
| **Satellite orbite basse (LEO)** | 25 à 60 ms | Moyenne | Variable | Meilleur effort | Quelques jours | Moyen | Coupures brèves lors des changements de satellite |
| **Satellite géostationnaire (GEO)** | 550 à 700 ms | Élevée | Variable | Variable | Semaines | Élevé | Mal adapté à la voix interactive |
| **Metro Ethernet** | 1 à 10 ms | Très faible | Très faible | 99,9 % et plus | 30 à 90 jours | Élevé | Local ou régional |

> Les valeurs sont des ordres de grandeur pour un raisonnement de conception. Toujours mesurer sur le site réel (voir la section 10 et le module SW18).

### 2.2 Le DIA contre la fibre grand public

| Critère | DIA | Fibre grand public pro |
|---|---|---|
| Garantie de rétablissement (GTR) | Oui (4 h typiquement) | Non |
| Adresse IP publique fixe | Oui | Souvent dynamique ou en option |
| Débit garanti | Oui (contractuel) | Non (jusqu'à) |
| Support | 24 h sur 24 | Heures ouvrées |
| Usage SD-WAN conseillé | Transport principal des sites importants | Transport secondaire ou petits sites |

### 2.3 Combinaisons usuelles par type de site

| Type de site | Transport 1 | Transport 2 | Secours |
|---|---|---|---|
| Siège, centre de données | DIA opérateur A | DIA opérateur B | MPLS conservé ou 5G |
| Agence importante | DIA ou MPLS | Fibre pro | 5G |
| Petite agence | Fibre pro | 4G/5G | — |
| Magasin | Fibre pro | 4G/5G (en secours seulement) | — |
| Site éphémère, véhicule | 5G | Satellite LEO | — |

---

## 3. Les métriques de qualité

| Métrique | Définition | Seuil usuel voix | Seuil usuel visioconférence | Impact |
|---|---|---|---|---|
| **Débit** | Quantité de données par seconde | 100 kbit/s par appel | 1 à 4 Mbit/s par flux | Lenteur, saturation |
| **Délai** (latence) | Temps de transit (aller ou aller-retour) | < 150 ms sens unique | < 200 ms | Conversations hachées |
| **Gigue** (jitter) | Variation du délai entre paquets | < 30 ms | < 50 ms | Voix saccadée, nécessite des tampons |
| **Perte** | Pourcentage de paquets perdus | < 1 % | < 2 % | Coupures, images dégradées, retransmissions TCP |
| **Disponibilité** | Temps où le lien est utilisable | 99,9 % et plus | idem | Interruption de service |

### Débit TCP et perte : la formule de Mathis

Le débit maximum d'une connexion TCP sur un chemin avec perte est approximé par :

```text
Débit (bit/s) ≈ (MSS × 8 / RTT) × (1,22 / √p)
```

Exemple : MSS = 1 360 octets, RTT = 40 ms, perte p = 1 % (0,01).
`(1360 × 8 / 0,04) × (1,22 / 0,1) = 272 000 × 12,2 ≈ 3,3 Mbit/s` par connexion.

Avec p = 0,1 % : `272 000 × 38,6 ≈ 10,5 Mbit/s`. **Un lien de 1 Gbit/s avec 1 % de perte offre à une connexion TCP unique à peine plus de 3 Mbit/s.** C'est pourquoi la perte est la métrique la plus destructrice, et pourquoi le SD-WAN cherche à l'éviter (choix de chemin, FEC, duplication).

---

## 4. L'overlay : IPsec, GRE, VXLAN, WireGuard

### 4.1 Comparaison

| Protocole | Couche | Chiffrement | Surcoût (IPv4) | Usage |
|---|---|---|---|---|
| **GRE** | Encapsulation IP (proto 47) | Non | 24 octets | Tunnels simples, protocoles de routage |
| **IPsec ESP (mode tunnel)** | IP (proto 50) ou UDP 4500 | Oui | 50 à 80 octets selon algorithmes | Standard des SD-WAN |
| **GRE sur IPsec** | GRE puis IPsec | Oui | 70 à 100 octets | DMVPN |
| **VXLAN** | UDP 4789 | Non | 50 octets | Réseaux de centre de données, certains SD-WAN |
| **WireGuard** | UDP (port configurable) | Oui (ChaCha20-Poly1305) | 60 octets (IPv4), 80 (IPv6) | Overlay moderne, simple |

### 4.2 IPsec en bref (détails dans CP03)

- **IKE** (IKEv2, RFC 7296) négocie les associations de sécurité (SA) et les clés.
- **ESP** (RFC 4303) chiffre et authentifie les paquets.
- En SD-WAN, les clés sont **distribuées par le plan de contrôle** : pas d'IKE entre sites dans Cisco Catalyst SD-WAN (les équipements reçoivent les clés de chiffrement via OMP et construisent des SA ESP directement), ce qui simplifie le maillage.
- **Mode tunnel** : le paquet d'origine est entièrement encapsulé (le seul mode utile en SD-WAN).

### 4.3 Calcul du surcoût IPsec ESP

Exemple : ESP AES-256-GCM, mode tunnel, avec encapsulation UDP (NAT-T), IPv4.

| Élément | Octets |
|---|---|
| En-tête IP externe | 20 |
| En-tête UDP (NAT-T) | 8 |
| En-tête ESP (SPI + numéro de séquence) | 8 |
| Vecteur d'initialisation (IV) GCM | 8 |
| Bourrage + longueur + en-tête suivant | 2 à 5 |
| Marque d'intégrité (ICV) GCM | 16 |
| **Total** | **environ 62 à 65** |

Avec une MTU de 1 500 sur l'underlay, le paquet interne maximal est d'environ **1 435 octets**, d'où un MSS TCP de `1435 − 40 = 1395` au mieux. Les implémentations réservent une marge (par exemple MSS 1 350 en pratique chez Cisco). Retenez : **prévoyez 70 à 80 octets de surcoût** par sécurité.

---

## 5. Traversée de NAT

La plupart des sites ont un accès Internet derrière un NAT (box opérateur, CGNAT 4G). Les tunnels doivent s'établir **quand même**.

### 5.1 Types de NAT (classification historique RFC 3489)

| Type | Comportement | Impact sur un tunnel |
|---|---|---|
| **Full cone** | Une fois la correspondance (IP interne:port) créée, tout hôte externe peut y envoyer | Facile |
| **Restricted cone** | Seules les IP déjà contactées peuvent répondre | Facile |
| **Port restricted cone** | Idem avec contrôle du port source | Acceptable |
| **Symétrique** | Une correspondance différente par destination (port source change) | **Difficile** : l'adresse découverte pour un pair n'est pas valide pour un autre |

### 5.2 Mécanismes

- **NAT-T** : encapsulation ESP dans UDP (4500) pour traverser le NAT ; détection du NAT pendant IKE.
- **Keepalive** : paquets périodiques pour maintenir la correspondance NAT ouverte (typiquement toutes les 15 à 30 s ; WireGuard `PersistentKeepalive = 25`).
- **STUN** (RFC 8489) : un serveur public indique l'adresse et le port tels que vus de l'extérieur. Cisco SD-WAN utilise les Validators pour cette découverte.
- **Perforation (hole punching)** : les deux extrémités émettent simultanément pour ouvrir les correspondances. Échoue le plus souvent entre **deux NAT symétriques**.

```text
Site A (192.168.1.10) -> NAT A (203.0.113.5:40000)  ---+
                                                       +--- Internet --- Site B derrière NAT B (198.51.100.7:51000)
Site B (192.168.2.10) -> NAT B                      ---+
Cas dangereux : NAT A symétrique ET NAT B symétrique -> pas de tunnel direct -> passer par un concentrateur (hub) avec IP publique
```

### 5.3 Bonnes pratiques

1. Placer les **concentrateurs** (hubs) sur des adresses publiques **sans NAT**, ou avec NAT statique 1:1.
2. Éviter le **CGNAT** sur les sites concentrateurs.
3. Pour les sites 4G en CGNAT : tunnels sortants uniquement vers les hubs, pas de tunnel direct entre rayons.
4. Tester le type de NAT avant de choisir un transport (outils : `stunclient`, `natmap`, ou le mode de test du produit).

---

## 6. MTU et MSS

### 6.1 Définitions

- **MTU** (Maximum Transmission Unit) : taille maximale d'un paquet IP sur une interface (1 500 sur Ethernet).
- **MSS** (Maximum Segment Size) : taille maximale des données TCP dans un segment : `MSS = MTU − 20 (IP) − 20 (TCP) = 1460`.
- **PMTUD** : découverte de la MTU du chemin par ICMP « fragmentation needed » (type 3, code 4). Souvent cassée par des pare-feu qui bloquent l'ICMP.

### 6.2 Le symptôme classique

« Le ping fonctionne, le site web s'ouvre à moitié, le transfert de gros fichiers se bloque. » Cause : les petits paquets passent, les gros sont perdus ou fragmentés car le tunnel a une MTU plus petite que celle des postes et l'ICMP est filtré (« trou noir PMTUD »).

### 6.3 Solutions

| Solution | Principe | Exemple |
|---|---|---|
| **Réduire la MTU du tunnel** | Les postes émettent des paquets plus petits | `mtu 1400` sur l'interface tunnel |
| **MSS clamping** | Le routeur réécrit l'option MSS des SYN TCP | `ip tcp adjust-mss 1350` (IOS XE) |
| **Autoriser l'ICMP** type 3 code 4 | PMTUD fonctionne | ACL sur pare-feu |
| **Fragmentation** | À éviter : coûteuse, parfois impossible (DF=1) | |

### 6.4 Commandes

Cisco IOS XE :

```text
interface GigabitEthernet3
 ip mtu 1500
 ip tcp adjust-mss 1350
!
show interfaces Tunnel100001 | include MTU
show sdwan tunnel statistics | include mtu
```

Linux (test du chemin avec le bit DF) :

```bash
# Taille de la charge utile ICMP = MTU - 28 (20 IP + 8 ICMP)
ping -M do -s 1472 198.51.100.7     # teste une MTU de 1500
ping -M do -s 1372 10.200.0.2       # teste une MTU de 1400 à travers l'overlay

# Découverte automatique du chemin
tracepath -n 198.51.100.7

# MSS clamping sous Linux (routeur)
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

---

## 7. Mesure de qualité : BFD et sondes

### 7.1 BFD (RFC 5880)

**BFD** (Bidirectional Forwarding Detection) est un protocole de détection de panne rapide, indépendant du média. Deux extrémités échangent des paquets de contrôle périodiques.

Paramètres essentiels :

| Paramètre | Rôle | Valeur par défaut Cisco SD-WAN |
|---|---|---|
| **Hello interval** | Intervalle entre paquets | 1 000 ms |
| **Multiplier** | Nombre de paquets manqués avant déclaration de panne | 7 |
| **Temps de détection** | `hello × multiplier` | 7 s |

En SD-WAN, BFD circule **dans le tunnel IPsec**, sur chaque tunnel de données. Il sert à :

1. détecter la mort d'un tunnel (vivacité) ;
2. mesurer **perte, délai, gigue** par chemin (les paquets BFD portent des horodatages) ;
3. alimenter le routage selon l'application (SW03, SW07).

### 7.2 Compromis

- Intervalle court = détection rapide, mais plus de trafic de contrôle et risque de faux positifs sur un lien instable (4G).
- Exemple : 200 sites en maillage complet à 2 transports : 200 × 199 / 2 × 2 = 39 800 tunnels (voir SW13), chacun émettant ~1 paquet/s dans chaque sens → ~80 000 paquets/s de contrôle au total ; il faut dimensionner et parfois **restreindre** le maillage (hubs, politiques).

### 7.3 Sondes actives

En plus de BFD, on peut mesurer avec :

| Outil | Mesure |
|---|---|
| `ping` / ICMP | Délai, perte |
| **IP SLA** (Cisco) | Délai, gigue, perte, UDP jitter (voix) |
| `mtr` | Perte et délai par saut |
| `iperf3` | Débit TCP et UDP, gigue UDP |
| Sondes HTTP | Qualité d'expérience vers SaaS (Cloud OnRamp) |

---

## 8. Haute disponibilité des liens

```text
Site avec haute disponibilité des liens
                   +-------------+
   LAN ------------|  Équipement |--- Lien 1 : DIA opérateur A  ---> Internet
                   |   SD-WAN    |--- Lien 2 : fibre opérateur B ---> Internet
                   |             |--- Lien 3 : 5G (secours)     ---> Internet
                   +-------------+
```

Principes :

1. **Opérateurs distincts** (pas seulement des liens distincts) : un incident sur le réseau de l'opérateur A n'affecte pas B.
2. **Parcours physiques distincts** quand c'est possible (deux arrivées de fibre).
3. **4G/5G en secours** : utilisée uniquement quand les autres échouent (option « last resort ») pour limiter les coûts en volume.
4. **Bascule** : décision par la détection (BFD) + politiques (SW07). Temps de bascule = temps de détection + temps de reconvergence.

Exemple de calcul de disponibilité : deux liens indépendants de disponibilité 99,5 % chacun :
`1 − (0,005 × 0,005) = 99,9975 %`, soit environ 13 minutes d'indisponibilité par an au lieu de 44 heures. Cette formule suppose l'**indépendance** des pannes (opérateurs et chemins distincts).

---

## 9. Interconnexion avec l'existant

| Cas | Technique | Remarque |
|---|---|---|
| Opérateur MPLS | **eBGP CE-PE** (ou OSPF, statique) | Voir SW01 et SW14 |
| Coexistence MPLS et SD-WAN | MPLS comme transport de l'overlay (couleur « mpls ») | Voir SW14 |
| Internet | Accès par défaut + NAT | Souvent DHCP ou PPPoE |
| LAN du site | OSPF, BGP, EIGRP, statique | Voir SW06 |
| Centre de données | Concentrateur SD-WAN + routage vers le cœur | Voir SW13, SW16 |

Exemple de routage minimal sur un équipement de site avec deux transports :

```text
! Table de routage de transport (underlay)
ip route 0.0.0.0 0.0.0.0 198.51.100.1        ! Transport 1 (Internet)
ip route 0.0.0.0 0.0.0.0 192.0.2.1 10        ! Transport 2 (MPLS) en secours (distance administrative 10)
```

> En SD-WAN, un « default » par transport est standard ; l'équipement établit des tunnels sur **chaque interface de transport**, indépendamment de la route par défaut préférée. Les tunnels vers les autres sites utilisent la route qui permet d'atteindre l'adresse de transport distante.

---

## 10. Lab guidé T1 : overlay WireGuard et mesures

### 10.1 Objectif

Construire un overlay chiffré entre deux « sites » Linux, injecter des dégradations dans l'underlay avec `tc netem`, mesurer délai, perte, gigue et débit, et observer l'effet de la MTU.

### 10.2 Topologie

```text
  LAN A 192.168.10.0/24                                       LAN B 192.168.20.0/24
  +---------+   eth1: 192.168.10.1                      eth1: 192.168.20.1   +---------+
  |  Hôte A |---+                                                          +---|  Hôte B |
  +---------+   |   +--------+  underlay 198.51.100.0/24  +--------+       |   +---------+
                +---| Site A |--------- eth0 --- eth0 ----| Site B |-------+
                    +--------+  .1                     .2 +--------+
                    wg0: 10.200.0.1/24               wg0: 10.200.0.2/24
```

Ressources : 2 VM Ubuntu (routeurs) + 2 hôtes clients (ou simples espaces de noms réseau). Vous pouvez aussi utiliser un seul hôte avec des `ip netns`.

### 10.3 Installation et clés (sur chaque site)

```bash
sudo apt update && sudo apt install -y wireguard iperf3 mtr-tiny tcpdump
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-forward.conf

umask 077
wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key
cat /etc/wireguard/public.key        # à communiquer à l'autre site
```

### 10.4 Configuration du site A (`/etc/wireguard/wg0.conf`)

```ini
[Interface]
Address = 10.200.0.1/24
PrivateKey = <CLE_PRIVEE_A>
ListenPort = 51820
MTU = 1420

[Peer]
PublicKey = <CLE_PUBLIQUE_B>
Endpoint = 198.51.100.2:51820
AllowedIPs = 10.200.0.2/32, 192.168.20.0/24
PersistentKeepalive = 25
```

Site B (symétrique) :

```ini
[Interface]
Address = 10.200.0.2/24
PrivateKey = <CLE_PRIVEE_B>
ListenPort = 51820
MTU = 1420

[Peer]
PublicKey = <CLE_PUBLIQUE_A>
Endpoint = 198.51.100.1:51820
AllowedIPs = 10.200.0.1/32, 192.168.10.0/24
PersistentKeepalive = 25
```

Démarrage et vérification :

```bash
sudo wg-quick up wg0
sudo wg show
ping -c 3 10.200.0.2
```

La commande `wg show` doit indiquer un `latest handshake` récent et des compteurs `transfer` non nuls.

### 10.5 Mesures de référence (sans dégradation)

```bash
# Depuis l'hôte A vers l'hôte B (ou depuis le site A vers 10.200.0.2)
ping -c 100 -i 0.2 10.200.0.2 | tail -3
# Débit TCP
iperf3 -s                          # sur B
iperf3 -c 10.200.0.2 -t 20         # sur A
# Gigue et perte UDP
iperf3 -c 10.200.0.2 -u -b 5M -t 20
```

### 10.6 Injection de dégradations sur l'underlay

```bash
# Sur Site A, interface d'underlay eth0
sudo tc qdisc add dev eth0 root netem delay 40ms 10ms distribution normal loss 1%
tc qdisc show dev eth0

# Modifier
sudo tc qdisc change dev eth0 root netem delay 80ms 20ms loss 3%

# Retirer
sudo tc qdisc del dev eth0 root
```

### 10.7 Script de mesure automatisé

```bash
#!/usr/bin/env bash
# mesure-overlay.sh : mesure délai, perte, gigue et débit vers un pair
# Usage : ./mesure-overlay.sh 10.200.0.2 etiquette
set -euo pipefail
CIBLE="${1:?adresse cible}"
ETIQ="${2:-test}"
FICHIER="resultats.csv"

PING=$(ping -c 100 -i 0.2 -q "$CIBLE" | tail -2)
PERTE=$(echo "$PING" | grep -oP '\d+(\.\d+)?(?=% packet loss)')
RTT=$(echo "$PING" | grep -oP 'rtt min/avg/max/mdev = \K[^ ]+')
IFS='/' read -r MIN AVG MAX MDEV <<< "$RTT"

BW=$(iperf3 -c "$CIBLE" -t 10 -J | python3 -c 'import sys,json;print(round(json.load(sys.stdin)["end"]["sum_received"]["bits_per_second"]/1e6,2))')

[ -f "$FICHIER" ] || echo "etiquette,perte_pct,rtt_min,rtt_moy,rtt_max,gigue_mdev,debit_mbps" > "$FICHIER"
echo "$ETIQ,$PERTE,$MIN,$AVG,$MAX,$MDEV,$BW" >> "$FICHIER"
column -s, -t "$FICHIER"
```

Lancez-le pour chaque scénario : `reference`, `delai40_perte1`, `delai80_perte3`.

### 10.8 Effet de la MTU

```bash
# Mesurer la plus grande taille qui passe sans fragmentation
for s in 1472 1452 1432 1412 1392 1372; do
  printf "taille %s : " "$s"
  ping -M do -c 2 -W 1 -s "$s" 10.200.0.2 >/dev/null 2>&1 && echo OK || echo ECHEC
done
```

Avec `MTU = 1420` sur wg0, la charge utile maximale du ping est `1420 − 28 = 1392`. Réduisez l'MTU de l'underlay pour simuler un opérateur qui encapsule en plus :

```bash
sudo ip link set dev eth0 mtu 1400
# Refaire les pings avec -M do : les tailles supérieures échouent (ou "Frag needed")
sudo ip link set dev eth0 mtu 1500    # retour à la normale
```

Corrigez ensuite avec le MSS clamping et retestez un téléchargement TCP volumineux.

### 10.9 Capture pour observation

```bash
sudo tcpdump -ni eth0 udp port 51820 -c 10     # underlay : paquets chiffrés
sudo tcpdump -ni wg0 icmp -c 5                  # overlay : paquets en clair
```

### 10.10 Livrable et grille

| Élément | Points |
|---|---|
| Tunnel établi (capture de `wg show`) | 3 |
| Tableau de mesures (référence + 2 dégradations) | 5 |
| Analyse de l'effet de la perte sur le débit TCP (comparaison à la formule de Mathis) | 4 |
| Test MTU, symptôme et correction | 4 |
| Limites de l'overlay maison (voir ci-dessous) | 4 |

**À rédiger** : en quoi cet overlay se distingue d'un SD-WAN ? (absence de plan de contrôle central, de mesure par chemin, de politiques applicatives, de ZTP, de supervision.)

---

## 11. Exercices

1. Un opérateur 4G place vos sites en CGNAT. Pourquoi deux sites 4G ne peuvent-ils pas toujours établir un tunnel direct ? Quelle architecture adopter ?
2. Calculez le MSS à configurer pour un tunnel dont le surcoût est de 78 octets sur un underlay en MTU 1 500.
3. Un lien présente 2 % de perte et 50 ms de RTT. Estimez le débit TCP maximal d'une connexion (MSS 1 350).
4. Avec un BFD à 500 ms et un multiplicateur de 5, quel est le temps de détection ?
5. Deux liens de disponibilité respective 99,5 % et 99,0 % sont indépendants. Quelle est la disponibilité combinée ?

<details>
<summary>Corrigés</summary>

1. Les NAT symétriques attribuent un port différent par destination : l'adresse découverte par STUN n'est pas valable pour le pair. Architecture : tunnels sortants vers des **hubs publics** ; trafic rayon à rayon via le hub.
2. MTU interne = 1 500 − 78 = 1 422 ; MSS = 1 422 − 40 = **1 382** (on arrondit prudemment à 1 350 en pratique).
3. `(1350×8/0,05) × (1,22/√0,02) = 216 000 × 8,63 ≈ 1,86 Mbit/s`.
4. 500 ms × 5 = **2,5 s**.
5. `1 − (0,005 × 0,01) = 99,995 %`.

</details>

---

## 12. Quiz

1. Quelle métrique réduit le plus fortement le débit TCP ?
   a) Le débit du lien ; b) La perte ; c) La couleur ; d) Le DSCP.
2. Pourquoi mettre un hub SD-WAN sur une IP publique sans NAT ?
3. WireGuard offre-t-il un plan de contrôle central ? (oui / non)
4. Quel est le MSS d'un tunnel avec MTU interne de 1 400 ?
   a) 1 400 ; b) 1 360 ; c) 1 460 ; d) 1 340.
5. À quoi sert `PersistentKeepalive` ?
6. Pourquoi bloquer l'ICMP peut casser un tunnel ?
7. Citez trois mesures de qualité qu'apporte BFD en SD-WAN.

<details>
<summary>Réponses</summary>

1. b. 2. Pour que les rayons puissent initier des tunnels vers lui même derrière des NAT, sans négociation difficile. 3. Non. 4. b (1 400 − 40). 5. Maintenir ouvertes les correspondances NAT. 6. L'ICMP type 3 code 4 est nécessaire à PMTUD : sans lui, les gros paquets disparaissent (trou noir). 7. Perte, délai, gigue.

</details>

---

## 13. Aide-mémoire

- Délais : MPLS < DIA < fibre pro < 5G < 4G < satellite GEO.
- **Surcoût IPsec** : prévoir 70 à 80 octets ; **MSS** = MTU interne − 40.
- **NAT symétrique** des deux côtés = pas de tunnel direct.
- **BFD** : 1 s × 7 = 7 s par défaut en Cisco SD-WAN.
- **Disponibilité parallèle** : `1 − (1−a)(1−b)`.
- **Mathis** : `débit ≈ MSS×8/RTT × 1,22/√p`.
- Commandes : `ping -M do -s N`, `tc qdisc add dev X root netem ...`, `wg show`, `iperf3`.

## 14. Pour aller plus loin

- RFC 5880 (BFD) : <https://www.rfc-editor.org/rfc/rfc5880>
- RFC 7296 (IKEv2) : <https://www.rfc-editor.org/rfc/rfc7296>
- RFC 8489 (STUN) : <https://www.rfc-editor.org/rfc/rfc8489>
- Documentation WireGuard : <https://www.wireguard.com/quickstart/>

---

## Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **Gigue (jitter)** | Variation du délai d'un paquet à l'autre |
| **RTT** | Temps d'aller-retour d'un paquet |
| **CGNAT** | NAT à grande échelle chez l'opérateur : plusieurs abonnés partagent une adresse publique |
| **NAT-T** | Traversée de NAT : encapsulation d'ESP dans UDP 4500 |
| **STUN** | Protocole permettant de découvrir son adresse publique et le comportement du NAT |
| **PMTUD** | Découverte de la MTU du chemin par ICMP |
| **MSS clamping** | Réécriture de l'option MSS des SYN pour éviter la fragmentation |
| **SA (Security Association)** | Contexte de sécurité IPsec unidirectionnel (algorithmes, clés, SPI) |
| **ESP** | Protocole IPsec qui chiffre et authentifie (numéro de protocole 50) |
| **AEAD** | Chiffrement authentifié (AES-GCM, ChaCha20-Poly1305) |
| **BFD** | Détection de panne bidirectionnelle rapide (RFC 5880) |
| **Dead peer detection** | Détection d'un pair IPsec silencieux |
| **LEO / GEO** | Satellites en orbite basse / géostationnaire |
| **Dual-WAN / dual-homing** | Deux accès Internet ou deux opérateurs sur un même site |

### B. Cas d'usage détaillés

#### Cas 1. Agence derrière un CGNAT 4G (site isolé)

| Rubrique | Détail |
|---|---|
| **Situation** | Un chantier ou une agence rurale n'a que la 4G, partagée en CGNAT |
| **Problème** | Aucune adresse publique : pas de tunnel entrant ; deux sites 4G ne peuvent pas se joindre directement |
| **Solution** | Tunnels **sortants** vers deux hubs publics ; trafic entre rayons via le hub ; keepalives courts (15 à 25 s) |
| **Pièges** | Les adresses changent à chaque reconnexion : l'identité du site ne doit pas reposer sur l'IP |
| **Leçon** | Un hub avec **IP publique fixe et sans NAT** est le pivot de toute architecture contenant des sites en CGNAT |

#### Cas 2. « Ça marche en ping mais pas en web » (trou noir PMTUD)

| Étape | Observation |
|---|---|
| Symptôme | Les petites pages s'ouvrent, les gros fichiers et certains sites bloquent |
| Test | `ping -M do -s 1472` échoue sans message ; `-s 1372` réussit |
| Cause | Le tunnel réduit la MTU effective à ~1420 ; un pare-feu intermédiaire bloque l'ICMP « fragmentation needed » |
| Correctif | MSS clamping à 1350 sur les interfaces LAN, puis autoriser l'ICMP type 3 code 4 |
| Leçon | Dans ce métier, **la MTU est responsable de 20 % des pannes mystérieuses** ; elle doit figurer dans la check-list de chaque recette |

#### Cas 3. Visioconférence hachée à 17 h

| Étape | Observation |
|---|---|
| Symptôme | Teams saccade tous les jours entre 17 h et 18 h |
| Mesure | `mtr` et sondes montrent 2 % de perte sur le lien fibre grand public à ces heures |
| Cause | Saturation du réseau de collecte de l'opérateur en soirée |
| Solution | Ne pas changer d'application : ajouter un second accès chez un autre opérateur et laisser la sélection de chemin trancher |
| Leçon | La **mesure historique** (SW18) est la seule preuve opposable à un opérateur |

#### Cas 4. Site mobile (véhicule de secours, événementiel)

5G principale, satellite LEO en secours, tunnels reconstruits à chaque changement de réseau ; l'identité repose sur le certificat de l'équipement et non sur l'adresse IP.

#### Cas 5. Choix entre DIA et fibre pro pour 50 agences

| Critère | DIA | Fibre pro |
|---|---|---|
| Coût mensuel | 250 € | 60 € |
| GTR | 4 h | aucune |
| Verdict | Hubs, grandes agences | Petites agences **avec un second lien d'un autre opérateur** |

Principe de décision : **la redondance d'opérateurs vaut mieux qu'un seul lien premium**. Deux liens à 99,5 % indépendants donnent 99,9975 % ; un lien premium à 99,95 % reste inférieur.

### C. Retours d'expérience

1. **Les opérateurs partagent des infrastructures.** Deux « opérateurs différents » peuvent passer dans la même gaine ou sur le même réseau de collecte. Demandez les **parcours physiques** (diversité de chemins).
2. **Le débit annoncé n'est pas le débit utile.** Mesurez avec `iperf3` dans les deux sens, à des heures différentes.
3. **La box grand public change tout.** Mises à jour, NAT par défaut, ALG SIP, filtrage UDP : en cas de panne inexpliquée, mettez la box en mode pont ou testez en direct.
4. **IPv6.** De plus en plus d'accès sont natifs en IPv6 avec NAT64/464XLAT sur mobile : vérifiez la prise en charge de votre solution.
5. **Un keepalive trop long tue les tunnels.** Les NAT de certains opérateurs mobiles expirent les correspondances UDP en 30 s.
6. **Documenter le plan d'adressage de l'underlay.** Sans lui, aucun dépannage n'est possible à 3 h du matin.

### D. Exercices supplémentaires

1. Un site dispose d'un DIA 100 Mbit/s (99,9 %) et d'une 4G (98 %, indépendante). Calculez la disponibilité combinée et le temps d'indisponibilité annuel attendu.
2. Vous devez choisir des transports pour un hôpital de campagne temporaire. Proposez une architecture (transports, NAT, hub) et justifiez.
3. Écrivez la procédure en 8 étapes de diagnostic d'un site dont le tunnel monte mais qui n'ouvre pas les gros transferts.
4. Mesurez, sur votre propre accès Internet, perte, délai et gigue pendant 24 h (script `ping` toutes les 5 s) et tracez les résultats.

<details>
<summary>Corrigés</summary>

1. `1 − (0,001 × 0,02) = 99,998 %` → 0,002 % de 8 760 h = **environ 10,5 minutes** par an.
2. 5G principale + satellite LEO ; tunnels sortants vers deux hubs publics ; keepalive 20 s ; chiffrement ; QoS pour les flux médicaux ; équipement léger pré-configuré (ZTP).
3. Vérifier MTU (`ping -M do`), MSS, ICMP type 3 code 4, compteurs de fragmentation, capture `tcpdump` sur LAN et WAN, test avec MSS réduit, test avec un autre transport, vérification du pare-feu intermédiaire, correction et recette.
4. Exemple : `while true; do ping -c1 -W1 1.1.1.1 | awk -F'time=' '/time=/{print strftime("%F %T"),$2}' >> rtt.log; sleep 5; done` puis traçage avec Grafana ou un tableur.

</details>

### E. Mini-projet (4 heures)

Réalisez une **fiche de qualification d'un site** : collecte du type d'accès, débit mesuré dans les deux sens, type de NAT, MTU effective, perte et gigue sur 1 h, adresse publique (fixe ou non), décision argumentée (transport principal, secours, rôle dans la topologie).

---

> Module précédent : [SW01](../SW01-du-wan-au-sd-wan/SW01-du-wan-au-sd-wan.md) | Module suivant : [SW03](../SW03-fonctions-cles/SW03-fonctions-cles.md)
