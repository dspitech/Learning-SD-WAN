# Fiche de commandes, de ports et de valeurs par défaut

> Fiche d'aide-mémoire à imprimer. **Vérifiez les commandes et valeurs avec la documentation de votre version.** Les valeurs par défaut sont celles couramment observées et peuvent évoluer.

---

## 1. Cisco Catalyst SD-WAN : WAN Edge (IOS XE)

### Plan de contrôle

```text
show sdwan control connections
show sdwan control connections-history
show sdwan control local-properties
show sdwan control affinity config
show sdwan certificate installed
show sdwan certificate root-ca-cert
show sdwan certificate serial
show clock
show ntp associations
```

### OMP

```text
show sdwan omp summary
show sdwan omp peers
show sdwan omp routes
show sdwan omp routes vpn <N> <prefixe>/<masque> detail
show sdwan omp tlocs
show sdwan omp services
```

### Plan de données

```text
show sdwan bfd sessions
show sdwan bfd summary
show sdwan tunnel statistics
show sdwan ipsec inbound-connections
show sdwan ipsec outbound-connections
show sdwan app-route stats
show sdwan app-route sla-class
```

### Politiques et sécurité

```text
show sdwan policy from-vsmart
show sdwan policy app-route-policy-filter
show sdwan policy data-policy-filter
show sdwan policy service-path vpn <N> interface <if> source-ip <ip> dest-ip <ip> protocol <n> all
show sdwan zbfw zonepair-statistics
show sdwan zbfw drop-statistics
show utd engine standard status
show app-hosting list
```

### Système et configuration

```text
show sdwan running-config
show sdwan system status
show sdwan version
show sdwan software
show logging
show platform resources
show processes cpu sorted | exclude 0.00
show ip route vrf <N>
show ip interface brief
```

### Cloud OnRamp

```text
show sdwan cloudexpress applications
show sdwan cloudexpress gateway-exits
show sdwan cloudexpress local-exits
```

### Opérations

```text
request platform software sdwan vedge_cloud activate chassis-number <UUID> token <OTP>
request platform software sdwan root-cert-chain install bootflash:<fichier>
request platform software sdwan config reset
controller-mode enable
```

### Dépannage de paquets

```text
monitor capture CAP1 interface <if> both
monitor capture CAP1 match ipv4 any host <ip>
monitor capture CAP1 start | stop
show monitor capture CAP1 buffer brief
debug platform condition ipv4 <ip>/32 both
debug platform packet-trace packet 128 fia-trace
debug platform condition start | stop
show platform packet-trace summary
show platform packet-trace packet 0 decode
```

## 2. Contrôleurs (Validator, Controller, Manager)

```text
show control connections
show control local-properties
show certificate installed
show orchestrator connections          (Validator)
show orchestrator valid-vedges         (Validator)
show omp peers | routes | tlocs        (Controller)
show running-config policy             (Controller)
show running-config apply-policy       (Controller)
request root-cert-chain install /home/admin/<fichier>
request nms configuration-db backup path <fichier>      (Manager)
request nms configuration-db restore path <fichier>     (Manager)
request nms all status                                   (Manager)
```

## 3. Ports et protocoles

| Usage | Port / protocole |
|---|---|
| Connexion de contrôle DTLS | UDP 12346 (+ décalages 12366, 12386, 12406, 12426) |
| Connexion de contrôle TLS | TCP 23456 (+ décalages) |
| Interface web et API du Manager | TCP 443 |
| NETCONF | TCP 830 |
| IKE | UDP 500 |
| IPsec NAT-T | UDP 4500 |
| ESP | Protocole IP 50 |
| NTP | UDP 123 |
| DNS | UDP et TCP 53 |
| Syslog | UDP 514 (ou TCP/TLS 6514) |
| NetFlow / IPFIX | UDP 2055 (usuel) |
| SNMP | UDP 161 (interrogation), 162 (traps) |
| WireGuard (exemple du cours) | UDP 51820 |

## 4. Valeurs par défaut courantes (à vérifier)

| Paramètre | Valeur |
|---|---|
| BFD hello / multiplicateur | 1 000 ms / 7 |
| AAR poll-interval / multiplicateur | 600 000 ms / 6 |
| Graceful restart OMP | 43 200 s |
| OMP hold time | 60 s |
| Rekey IPsec du plan de données | 86 400 s |
| Fenêtre anti-rejeu | 512 |
| DSCP des paquets de contrôle BFD | 48 (CS6) |
| Distance administrative d'une route OMP | 251 |

## 5. Couleurs de transport

| Famille | Couleurs |
|---|---|
| Privées | mpls, metro-ethernet, private1 à private6 |
| Publiques | default, biz-internet, public-internet, lte, 3g, red, green, blue, gold, silver, bronze, custom1, custom2, custom3 |

## 6. Codes d'échec fréquents des connexions de contrôle

| Code | Signification probable |
|---|---|
| CRTREJSER | Certificat rejeté par le pair (équipement non autorisé, certificat invalide) |
| CRTVERFL | Échec de vérification du certificat (date, racine, chaîne) |
| SERNTPRES | Numéro de série absent de la liste d'autorisation |
| DCONFAIL | Échec de connexion DTLS/TLS (pare-feu, routage, port) |
| NOVMCFG | Aucune configuration côté Manager pour l'équipement |
| NOACTVB | Aucun Validator actif |
| LISFD | Échec d'écoute de l'interface locale |
| DISTLOC | TLOC désactivé |
| BIDNTVRFD | Identité de la carte non vérifiée |

## 7. DSCP

| Classe | DSCP | ToS (décimal) | Hexa |
|---|---|---|---|
| EF (voix) | 46 | 184 | 0xB8 |
| AF41 (vidéo) | 34 | 136 | 0x88 |
| AF31 (critique) | 26 | 104 | 0x68 |
| CS6 (contrôle) | 48 | 192 | 0xC0 |
| CS3 (signalisation) | 24 | 96 | 0x60 |
| CS1 (arrière-plan) | 8 | 32 | 0x20 |

Formule : `ToS = DSCP × 4`.

## 8. Formules utiles

```text
Maillage complet : tunnels = N(N-1)/2 x T          Hub and spoke : (N - hubs) x hubs x T
Disponibilité en parallèle : 1 - (1-a)(1-b)          En série : a x b
Budget d'erreur mensuel (30 j) : (1 - SLO) x 43 200 min      99,9 % = 43,2 min
Mathis (débit TCP) : (MSS x 8 / RTT) x (1,22 / racine(p))
Sérialisation : octets x 8 / débit          Délai de file : paquets x taille x 8 / débit
Bc (shaping) = CIR x Tc          MSS = MTU - 40 (IPv4/TCP)
Voix G.711 : 80 kbit/s IP, 87 Ethernet, ~111 avec IPsec         G.729 : 24 / 31 / ~55
Stockage séries temporelles ≈ séries x (rétention / intervalle) x 2 octets
```

## 9. Linux : commandes de laboratoire

```bash
ping -M do -s 1372 <ip>                         # test MTU (taille = MTU - 28)
iperf3 -s ; iperf3 -c <ip> -t 20 -P 4           # débit TCP ; -u -b 100k pour UDP
mtr -rwzbc 100 <ip>                             # perte et délai par saut
tc qdisc add dev eth0 root netem delay 40ms 10ms loss 1%
tc qdisc del dev eth0 root
tcpdump -ni eth0 'udp port 500 or udp port 4500 or ip proto 50'
wg show ; wg-quick up wg0
swanctl --list-sas ; ip xfrm state
```

## 10. Méthode de dépannage par plan

```text
1 CONTRÔLE (connexions, certificats, horloge)  ->  2 OMP (pairs, routes, TLOC)  ->  3 DONNÉES (BFD, tunnels)
->  4 POLITIQUES (reçues, appliquées)  ->  5 APPLICATION (chemin, qualité, QoS)  ->  hors SD-WAN (LAN, Wi-Fi, serveur)
Cinq questions : Qu'est-ce qui a changé ? Qui est touché ? Depuis quand ? Reproductible ? Qu'est-ce qui marche ?
```
