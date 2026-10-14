# CP03. IPsec en profondeur : IKEv2, ESP, NAT-T et dépannage

| | |
|---|---|
| **Partie** | Compléments (hors parcours principal) |
| **Durée indicative** | 1,5 semaine (12 h) |
| **Prérequis** | CCNA/CCNP (notions IPsec) ; SW02, SW08 ; Linux, Wireshark |
| **Niveau** | Avancé |
| **Lien examen** | ENCOR 350-401 (VPN), SCOR 350-701 (VPN), ENSDWI (chiffrement) |
| **Lab associé** | Lab CP03 (tunnel IKEv2/ESP entre deux Linux avec strongSwan, capture et analyse) |

> Module précédent : [CP02](../CP02-qos-wan/CP02-qos-wan.md) | Module suivant : [CP04](../CP04-flexiwan/CP04-flexiwan.md)

> **Pourquoi ce complément.** Dans Catalyst SD-WAN, les clés des tunnels de données sont distribuées par OMP et il n'y a pas d'IKE entre WAN Edge. Mais IKEv2/IPsec reste omniprésent : tunnels vers les services de sécurité (SIG/SSE), vers le cloud, vers des équipements tiers, VPN hérités. Un architecte doit maîtriser IPsec pour **construire, sécuriser et dépanner** ces liaisons.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Les briques d'IPsec](#2-les-briques-dipsec)
3. [IKEv2 pas à pas](#3-ikev2-pas-à-pas)
4. [ESP : modes, en-têtes et chiffrement](#4-esp--modes-en-têtes-et-chiffrement)
5. [Algorithmes et paramètres recommandés](#5-algorithmes-et-paramètres-recommandés)
6. [Authentification : clés pré-partagées et certificats](#6-authentification--clés-pré-partagées-et-certificats)
7. [NAT-T, DPD et durée de vie](#7-nat-t-dpd-et-durée-de-vie)
8. [Configuration IOS XE : tunnel VTI en IKEv2](#8-configuration-ios-xe--tunnel-vti-en-ikev2)
9. [Configuration Linux : strongSwan](#9-configuration-linux--strongswan)
10. [Cas des tunnels vers SIG, SSE et cloud](#10-cas-des-tunnels-vers-sig-sse-et-cloud)
11. [Capture et analyse avec Wireshark](#11-capture-et-analyse-avec-wireshark)
12. [Dépannage méthodique](#12-dépannage-méthodique)
13. [Lab guidé CP03](#13-lab-guidé-cp03)
14. [Approfondissements](#14-approfondissements)
15. [Quiz](#15-quiz)
16. [Aide-mémoire](#16-aide-mémoire)
17. [Pour aller plus loin](#17-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Décrire les échanges IKEv2 et le rôle de chaque message.
- Distinguer IKE SA et Child SA, mode tunnel et mode transport, ESP et AH.
- Choisir des algorithmes et des durées de vie conformes aux recommandations.
- Configurer un tunnel IKEv2 sur IOS XE et sous Linux (strongSwan).
- Traiter NAT-T, DPD, MTU et rekey.
- Capturer, lire et dépanner un tunnel IPsec.

---

## 2. Les briques d'IPsec

| Brique | Rôle | Référence |
|---|---|---|
| **IKEv2** | Négociation, authentification, échange de clés | RFC 7296 |
| **ESP** (protocole IP 50) | Confidentialité, intégrité, anti-rejeu des données | RFC 4303 |
| **AH** (protocole IP 51) | Intégrité seule (rare, incompatible avec NAT) | RFC 4302 |
| **SA** (Security Association) | Contexte **unidirectionnel** : algorithmes, clés, SPI, compteurs | RFC 4301 |
| **SPD / SAD** | Base des politiques / base des associations | RFC 4301 |
| **NAT-T** | Encapsulation UDP 4500 pour traverser le NAT | RFC 3947/3948 |
| **DH / ECDH** | Échange de clés Diffie-Hellman (confidentialité persistante) | |

```text
Ports et protocoles à autoriser : UDP 500 (IKE), UDP 4500 (IKE + ESP en NAT-T), protocole IP 50 (ESP direct, sans NAT)
```

### 2.1 Deux niveaux de SA

| SA | Rôle | Durée de vie typique |
|---|---|---|
| **IKE SA** (SA de contrôle) | Canal protégé entre pairs pour négocier | 8 à 24 heures |
| **Child SA** (IPsec SA) | Protège le trafic de données (une par sens) | 1 à 8 heures et/ou volume |

---

## 3. IKEv2 pas à pas

### 3.1 Les échanges

| Échange | Contenu | Résultat |
|---|---|---|
| **IKE_SA_INIT** (2 messages) | Propositions (chiffrement, intégrité, PRF, groupe DH), valeurs DH, nonces, détection de NAT | Clés de l'IKE SA |
| **IKE_AUTH** (2 messages) | Identités, authentification (PSK ou certificats), **première Child SA** | IKE SA authentifiée + Child SA |
| **CREATE_CHILD_SA** | Nouvelle Child SA ou **rekey** | SA renouvelées |
| **INFORMATIONAL** | Notifications, suppression, **DPD** | Maintenance |

```text
Initiateur                                  Répondeur
  | -- IKE_SA_INIT (SAi1, KEi, Ni) -------->  |
  | <- IKE_SA_INIT (SAr1, KEr, Nr) ---------  |   [clés dérivées : SK_e, SK_a, SK_d ...]
  | -- IKE_AUTH {IDi, AUTH, SAi2, TSi, TSr} ->|   (chiffré par l'IKE SA)
  | <- IKE_AUTH {IDr, AUTH, SAr2, TSi, TSr} --|
  |  Child SA établie : trafic ESP            |
```

### 3.2 Sélecteurs de trafic (TS)

Les **traffic selectors** définissent quels flux sont protégés (par exemple 192.168.10.0/24 ↔ 192.168.20.0/24). Avec un **tunnel VTI** (interface de tunnel routée), les sélecteurs sont `0.0.0.0/0` et le **routage** décide ; avec un VPN « policy-based », les sélecteurs doivent **correspondre exactement** des deux côtés (cause très fréquente d'échec).

### 3.3 PFS et dérivation de clés

Chaque rekey de Child SA peut refaire un échange DH (**PFS**, Perfect Forward Secrecy) : la compromission d'une clé ne révèle pas les trafics passés ni futurs.

---

## 4. ESP : modes, en-têtes et chiffrement

### 4.1 Mode tunnel et mode transport

| Mode | Principe | Usage |
|---|---|---|
| **Tunnel** | Le paquet **entier** est encapsulé et protégé, avec un nouvel en-tête IP | VPN site à site (le standard) |
| **Transport** | Seule la charge utile est protégée ; l'en-tête IP d'origine reste | Entre deux hôtes, GRE sur IPsec |

### 4.2 Structure d'un paquet ESP (mode tunnel)

```text
| IP externe | ESP : SPI (4) + N° de séquence (4) | IV | [ IP interne | données | bourrage | long. bourrage | en-tête suivant ] | ICV |
              <------------- en clair ------------>  <---------------------- chiffré ------------------------->     intégrité
```

Avec NAT-T, un en-tête **UDP (8 octets, port 4500)** s'insère entre l'IP externe et l'ESP (et un marqueur de 4 octets « non-ESP » pour IKE).

### 4.3 Anti-rejeu

Le **numéro de séquence** croît à chaque paquet ; le récepteur garde une **fenêtre** (par exemple 64 à 512 paquets) et rejette les doublons et les paquets trop anciens. Un fort désordonnancement (répartition par paquet sur plusieurs chemins) peut provoquer des **rejets anti-rejeu** légitimes : augmentez la fenêtre.

### 4.4 Surcoût (rappel)

| Algorithme | Surcoût indicatif (IPv4, tunnel, hors NAT-T) |
|---|---|
| ESP AES-GCM-256 | 20 (IP) + 8 (ESP) + 8 (IV) + 2 à 5 (bourrage, suite) + 16 (ICV) ≈ 54 à 57 |
| Avec NAT-T | + 8 (UDP) ≈ 62 à 65 |
| ESP AES-CBC + HMAC-SHA256 | + 16 (IV) + 16 à 32 (ICV tronquée 128 bits) ≈ 70 à 85 |

---

## 5. Algorithmes et paramètres recommandés

### 5.1 Tableau de recommandations (à confronter à votre politique de sécurité et aux recommandations de l'ANSSI/NIST)

| Fonction | Recommandé | Acceptable (transition) | À éviter |
|---|---|---|---|
| **Chiffrement** | AES-256-GCM, AES-128-GCM, ChaCha20-Poly1305 | AES-256-CBC + HMAC-SHA-256/384 | DES, 3DES, RC4, AES-CBC avec SHA-1 |
| **Intégrité (hors AEAD)** | HMAC-SHA-256/384/512 | HMAC-SHA-1 (transition) | MD5 |
| **PRF** | HMAC-SHA-256/384 | | MD5, SHA-1 |
| **Groupe DH** | **19, 20** (ECP 256/384), 21, 31 (Curve25519) ; **14 (MODP 2048)** au minimum | 15, 16 (MODP 3072/4096) | Groupes 1, 2, 5 |
| **Authentification** | Certificats (ECDSA ou RSA ≥ 3072) | PSK longues (≥ 32 caractères aléatoires) | PSK courtes ou partagées entre de nombreux sites |
| **PFS** | Activé (même groupe fort) | | Désactivé |

### 5.2 Durées de vie

| Paramètre | Valeur usuelle | Remarque |
|---|---|---|
| IKE SA | 8 à 24 h | Plus court = plus de CPU de rekey |
| Child SA (temps) | 1 à 8 h | |
| Child SA (volume) | 4 à 16 Go (ou « 0 » = non utilisé) | À prévoir pour AES-GCM (limites de volume de chiffrement) |
| Rekey avant expiration | ~90 % de la durée | Évite la coupure |

### 5.3 Pourquoi AES-GCM

Mode **AEAD** : chiffrement et intégrité en une opération, accélérée matériellement (instructions AES-NI), moins de surcoût que CBC + HMAC. Contrainte : ne **jamais** réutiliser un nonce avec la même clé (géré par l'implémentation) et respecter les limites de volume.

---

## 6. Authentification : clés pré-partagées et certificats

| Critère | PSK | Certificats |
|---|---|---|
| Mise en œuvre | Simple | Demande une PKI |
| Évolutivité (nombreux sites) | Mauvaise (une clé par paire ou secret partagé) | Excellente |
| Révocation | Changer la clé partout | CRL/OCSP, invalidation |
| Sécurité | Dépend de la longueur et de la gestion de la clé | Meilleure (clé privée non partagée) |
| Usage | Laboratoire, quelques tunnels, services tiers imposant PSK | **Production à échelle** |

Bonnes pratiques PSK : **32 caractères aléatoires ou plus**, une PSK **par pair**, stockage en coffre, rotation planifiée, jamais dans un dépôt.

Génération d'une PSK :

```bash
openssl rand -base64 36        # 48 caractères environ
```

---

## 7. NAT-T, DPD et durée de vie

### 7.1 NAT-T

Pendant `IKE_SA_INIT`, chaque pair envoie un **hachage de sa propre adresse/port**. Si le hachage reçu ne correspond pas à l'adresse source vue, un NAT est détecté : IKE bascule sur **UDP 4500** et ESP est encapsulé en UDP.

**Keepalive NAT** : paquets de 1 octet envoyés toutes les 20 secondes environ pour conserver la correspondance.

### 7.2 DPD (Dead Peer Detection)

Détection d'un pair silencieux par messages INFORMATIONAL périodiques.

| Paramètre | Valeur usuelle |
|---|---|
| Intervalle | 10 à 30 s |
| Tentatives | 3 à 5 |
| Action | Redémarrer (`restart`) ou nettoyer (`clear`) |

### 7.3 Rekey et micro-coupures

Un rekey bien fait est **sans coupure** (la nouvelle SA est installée avant l'ancienne). Les coupures de rekey viennent de **paramètres incohérents** entre pairs (durées, propositions, sélecteurs) ou de **pertes** pendant l'échange.

---

## 8. Configuration IOS XE : tunnel VTI en IKEv2

```text
! 1. Proposition IKEv2
crypto ikev2 proposal PROP-IKEV2
 encryption aes-gcm-256
 prf sha256
 group 19
!
crypto ikev2 policy POL-IKEV2
 proposal PROP-IKEV2
!
! 2. Clés pré-partagées
crypto ikev2 keyring KR-SITEB
 peer SITEB
  address 203.0.113.2
  pre-shared-key local <PSK-LOCALE-A-REMPLACER>
  pre-shared-key remote <PSK-DISTANTE-A-REMPLACER>
!
! 3. Profil IKEv2 (qui est autorisé, comment il s'authentifie)
crypto ikev2 profile PROF-SITEB
 match identity remote address 203.0.113.2 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KR-SITEB
 dpd 20 3 on-demand
 lifetime 28800
!
! 4. Transformations ESP et profil IPsec
crypto ipsec transform-set TS-GCM esp-gcm 256
 mode tunnel
!
crypto ipsec profile IPSEC-SITEB
 set transform-set TS-GCM
 set ikev2-profile PROF-SITEB
 set pfs group19
 set security-association lifetime seconds 3600
!
! 5. Interface de tunnel routée (VTI)
interface Tunnel10
 description IPSEC-VERS-SITEB
 ip address 10.200.0.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source GigabitEthernet1
 tunnel destination 203.0.113.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-SITEB
!
ip route 192.168.20.0 255.255.255.0 10.200.0.2
```

Vérifications :

```text
show crypto ikev2 sa
show crypto ikev2 sa detailed
show crypto ipsec sa peer 203.0.113.2
show crypto session
show interfaces Tunnel10
show crypto ikev2 stats
```

Lecture de `show crypto ipsec sa` : compteurs `#pkts encaps / encrypt / decrypt / decaps` (doivent croître **dans les deux sens**), `#send errors`, `#recv errors`, SPI entrants et sortants, `current outbound spi`.

---

## 9. Configuration Linux : strongSwan

### 9.1 Installation

```bash
sudo apt update && sudo apt install -y strongswan strongswan-swanctl charon-systemd tcpdump
sudo systemctl enable --now strongswan
sudo sysctl -w net.ipv4.ip_forward=1
```

### 9.2 Site A (`/etc/swanctl/conf.d/site-a.conf`)

```text
connections {
    a-b {
        version = 2
        local_addrs  = 203.0.113.1
        remote_addrs = 203.0.113.2
        proposals = aes256gcm16-prfsha256-ecp256
        dpd_delay = 20s
        rekey_time = 8h
        local  { auth = psk
                 id = siteA }
        remote { auth = psk
                 id = siteB }
        children {
            reseaux {
                local_ts  = 192.168.10.0/24
                remote_ts = 192.168.20.0/24
                esp_proposals = aes256gcm16-ecp256
                rekey_time = 1h
                start_action = start
                dpd_action = restart
                mode = tunnel
            }
        }
    }
}

secrets {
    ike-ab {
        id-1 = siteA
        id-2 = siteB
        secret = "<PSK-LONGUE-ET-ALEATOIRE>"
    }
}
```

Site B : symétrique (`local_addrs = 203.0.113.2`, `remote_addrs = 203.0.113.1`, identités et sélecteurs inversés).

### 9.3 Commandes

```bash
sudo swanctl --load-all
sudo swanctl --initiate --child reseaux
sudo swanctl --list-sas
sudo swanctl --list-conns
sudo journalctl -u strongswan -f
sudo ip xfrm state          # SA du noyau (SPI, algorithmes)
sudo ip xfrm policy         # politiques de chiffrement du noyau
```

---

## 10. Cas des tunnels vers SIG, SSE et cloud

| Sujet | Recommandation |
|---|---|
| **Paramètres imposés** | Le fournisseur impose ses propositions (chiffrement, DH, durées) : les adopter **exactement** |
| **Identité** | FQDN, adresse IP ou e-mail selon le service : cohérence avec la configuration du service |
| **Deux tunnels** | Un principal, un secours (PoP différents) ; basculement par routage ou politique |
| **MTU et MSS** | MTU du tunnel 1 400 environ, MSS 1 360 |
| **NAT** | Si le routeur est derrière un NAT : activer NAT-T, utiliser l'identité FQDN |
| **Surveillance** | Sondes (SLA) à travers le tunnel, alertes sur échec |
| **Volume et débit** | Un tunnel IPsec unique est souvent plafonné (par exemple 1 à 2 Gbit/s selon fournisseur) ; plusieurs tunnels en ECMP si besoin |
| **Cloud public** | VPN gérés (AWS Site-to-Site VPN, Azure VPN Gateway) : deux tunnels par connexion, BGP recommandé |

---

## 11. Capture et analyse avec Wireshark

### 11.1 Filtres utiles

```text
isakmp                               # messages IKE
isakmp.exchtype == 34                # IKE_SA_INIT
isakmp.exchtype == 35                # IKE_AUTH
isakmp.exchtype == 36                # CREATE_CHILD_SA (rekey)
isakmp.exchtype == 37                # INFORMATIONAL (dont DPD)
esp                                  # paquets ESP
udp.port == 4500                     # NAT-T
ip.proto == 50                       # ESP direct
```

### 11.2 Capture

```bash
sudo tcpdump -ni eth0 -w ipsec.pcap '(udp port 500 or udp port 4500 or ip proto 50)'
# Analyse dans Wireshark ; pour déchiffrer l'IKE, fournir les SPI et clés (réservé au laboratoire)
```

### 11.3 Ce qu'il faut lire

| Observation | Interprétation |
|---|---|
| `IKE_SA_INIT` envoyé, **aucune réponse** | Filtrage UDP 500, adresse erronée, pair arrêté |
| Réponse `NO_PROPOSAL_CHOSEN` | Propositions incompatibles (chiffrement, DH) |
| `AUTHENTICATION_FAILED` | PSK ou certificat erroné, identité inattendue |
| `TS_UNACCEPTABLE` | Sélecteurs de trafic différents |
| `INVALID_KE_PAYLOAD` | Groupe DH divergent |
| ESP sortant seul, sans retour | Retour filtré, NAT, routage retour |
| Fragments IP après ESP | MTU : le paquet chiffré dépasse la MTU de l'underlay |

---

## 12. Dépannage méthodique

### 12.1 Arbre de décision

```text
1 Le pair est-il joignable en IP ? (ping, route)                    non -> routage / pare-feu
2 UDP 500 (et 4500) traversent-ils ? (capture des deux côtés)       non -> pare-feu / NAT / ACL
3 IKE_SA_INIT réussit-il ? (propositions identiques ?)              non -> NO_PROPOSAL_CHOSEN / groupe DH
4 IKE_AUTH réussit-il ? (identités, PSK/certificats, dates)         non -> AUTHENTICATION_FAILED
5 La Child SA s'établit-elle ? (sélecteurs, PFS)                     non -> TS_UNACCEPTABLE / PFS divergent
6 Le trafic est-il chiffré dans les DEUX sens ? (compteurs)         non -> routage, retour, NAT
7 Les gros paquets passent-ils ? (MTU/MSS)                          non -> réduire MTU, clamper le MSS
8 Le tunnel est-il stable ? (rekey, DPD, logs)                       non -> durées, NAT keepalive, instabilité du lien
```

### 12.2 Causes fréquentes de pannes

| Symptôme | Cause | Remède |
|---|---|---|
| Tunnel `down` après changement de pare-feu | UDP 500/4500 bloqués ou ALG | Autoriser, désactiver l'ALG IPsec |
| Tunnel up, pas de trafic | Route de retour absente, sélecteurs, NAT d'exemption | Vérifier routes et exemptions de NAT |
| Tunnel up, pings OK, gros transferts lents | MTU/MSS | `ip tcp adjust-mss`, MTU du tunnel |
| Coupure toutes les heures | Rekey incohérent | Aligner les durées de vie, activer le rekey proactif |
| Échec d'authentification | PSK avec espace final, identité incorrecte, certificat expiré, horloge fausse | Contrôler caractère par caractère, NTP |
| Tunnel instable derrière 4G | NAT mobile expirant les correspondances | Keepalive NAT plus court, DPD |
| Débit faible | CPU de chiffrement saturé, algorithme sans accélération | Vérifier l'accélération matérielle, choisir AES-GCM |

### 12.3 Commandes de débogage (à utiliser avec prudence)

```text
debug crypto ikev2 error
debug crypto ikev2 internal
debug crypto ipsec error
! Toujours : terminal monitor, limiter par adresse (debug condition peer), et "undebug all" à la fin
```

---

## 13. Lab guidé CP03

### 13.1 Objectif

Construire un tunnel IKEv2/ESP entre deux Linux avec strongSwan, **le casser de plusieurs façons**, capturer et lire les échanges.

### 13.2 Topologie

```text
Hôte A 192.168.10.10 -- [ Site A 203.0.113.1 ] ====== IPsec ESP ====== [ Site B 203.0.113.2 ] -- Hôte B 192.168.20.10
```

### 13.3 Étapes

| # | Action | Vérification |
|---|---|---|
| 1 | Installer strongSwan sur les deux sites ; appliquer les configurations de la section 9 | `swanctl --list-conns` |
| 2 | Établir le tunnel ; ping hôte A vers hôte B | `swanctl --list-sas` ; compteurs |
| 3 | Capturer l'établissement (`tcpdump`) ; identifier les 4 messages IKE | Analyse dans Wireshark |
| 4 | Confirmer le chiffrement : capturer sur eth0 (ESP) et sur l'interface LAN (clair) | Comparaison |
| 5 | **Panne A** : modifier la PSK d'un côté | `AUTHENTICATION_FAILED` dans les journaux |
| 6 | **Panne B** : proposition différente (`aes128-sha1-modp1024` d'un côté) | `NO_PROPOSAL_CHOSEN` |
| 7 | **Panne C** : sélecteurs différents (`remote_ts` erroné) | `TS_UNACCEPTABLE` |
| 8 | **Panne D** : bloquer UDP 500 (`iptables -A INPUT -p udp --dport 500 -j DROP`) | Aucune réponse à IKE_SA_INIT |
| 9 | **Panne E** : MTU de l'underlay à 1 400 ; `ping -M do -s 1400` à travers le tunnel | Échec, puis correction MSS/MTU |
| 10 | Mesurer le rekey : durée de la Child SA à 2 minutes, observer `CREATE_CHILD_SA`, vérifier l'absence de perte avec un ping continu | Pas de coupure |
| 11 | Mesurer le débit : `iperf3` avec AES-GCM puis avec `aes256-sha256` (CBC) | Comparaison |
| 12 | Rédiger la fiche de dépannage IPsec de l'équipe | Document |

### 13.4 Script d'aide : surveillance du tunnel

```bash
#!/usr/bin/env bash
# surveille-ipsec.sh : alerte si aucune SA n'est établie ou si les compteurs n'évoluent pas
CHILD=${1:-reseaux}
while true; do
  N=$(sudo swanctl --list-sas | grep -c ESTABLISHED)
  B=$(sudo swanctl --list-sas | grep -oE '[0-9]+ bytes_i' | head -1 | cut -d' ' -f1)
  echo "$(date +%T) sa_etablies=$N octets_entrants=${B:-0}"
  [ "$N" -eq 0 ] && { echo "ALERTE : aucune SA établie, relance"; sudo swanctl --initiate --child "$CHILD"; }
  sleep 10
done
```

### 13.5 Grille

| Critère | Points |
|---|---|
| Tunnel établi et chiffrement démontré | 4 |
| Capture et lecture des messages IKE | 4 |
| Cinq pannes reproduites et diagnostiquées | 6 |
| Rekey sans coupure démontré | 3 |
| Comparaison de débit AEAD / CBC | 2 |
| Fiche de dépannage | 1 |

---

## 14. Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **SA / SAD / SPD** | Association de sécurité / base des SA / base des politiques |
| **SPI** | Indice des paramètres de sécurité qui identifie une SA |
| **ESP / AH** | Protocoles de protection des données IPsec |
| **IKEv2** | Protocole d'échange de clés Internet, version 2 |
| **Child SA** | SA des données, issue de l'IKE SA |
| **PFS** | Confidentialité persistante |
| **DH / ECDH** | Échange de clés Diffie-Hellman (courbes elliptiques) |
| **AEAD** | Chiffrement authentifié avec données associées |
| **TS** | Sélecteurs de trafic |
| **VTI** | Interface de tunnel virtuelle routée |
| **NAT-T** | Traversée de NAT par encapsulation UDP 4500 |
| **DPD** | Détection d'un pair mort |
| **PRF** | Fonction pseudo-aléatoire de dérivation de clés |
| **ICV** | Valeur de contrôle d'intégrité |
| **IV / nonce** | Valeur d'initialisation / nombre à usage unique |

### B. Cas d'usage détaillés

#### Cas 1. Tunnel vers un SIG qui tombe toutes les 8 heures

Le tunnel vers un service de sécurité cloud se rompt toutes les 8 h pendant 20 à 40 s. Cause : durée de vie IKE du service à 8 h, durée locale à 24 h ; l'équipement ne prend pas l'initiative du rekey et attend l'expiration. Remède : aligner les durées et activer le rekey proactif ; ajouter un second tunnel. **Leçon** : **adoptez les durées du fournisseur**, et testez un cycle complet de rekey avant la mise en production.

#### Cas 2. L'ALG IPsec du pare-feu de l'agence

Un pare-feu de bordure d'agence avec ALG IPsec réécrit les ports et casse NAT-T : le tunnel monte puis se coupe aléatoirement. Désactivation de l'ALG et ouverture explicite de UDP 500/4500. **Leçon** : « inspecter » IPsec est presque toujours une mauvaise idée.

#### Cas 3. Le tunnel monté, la comptabilité impossible

Le tunnel est up, mais le serveur de la comptabilité est injoignable. Les sélecteurs du pair listent `192.168.20.0/24` mais le serveur est en `192.168.21.0/24`. Avec une VTI et routage, le problème disparaît. **Leçon** : préférez les **VTI/routage** aux VPN « policy-based » pour toute architecture qui évolue.

#### Cas 4. Mise à l'échelle : 300 tunnels PSK

Un réseau avec 300 sites et des PSK par site est ingérable (rotation, fuite). Migration vers des certificats émis par une PKI interne avec profil automatique. **Leçon** : **au-delà de quelques tunnels, certificats**.

#### Cas 5. Débit plafonné à 300 Mbit/s

Un tunnel IPsec entre deux sites 1 Gbit/s plafonne à 300 Mbit/s. Cause : un seul flux, un seul cœur, algorithme sans accélération. Solutions : AES-GCM, plusieurs flux (`iperf3 -P 8`), plateformes avec accélération cryptographique, ECMP sur plusieurs tunnels. **Leçon** : le chiffrement est **par flux et par cœur** sur de nombreuses plates-formes.

#### Cas 6. Transition post-quantique

Une entreprise du secteur public doit anticiper les menaces « récolter maintenant, déchiffrer plus tard » : IKEv2 hybride (échange classique + algorithme post-quantique) selon les évolutions des normes, clés pré-partagées fortes en complément. **Leçon** : **suivez les recommandations nationales** (ANSSI) et privilégiez des équipements **mis à jour régulièrement**.

### C. Retours d'expérience

1. **Gardez les paramètres cryptographiques simples et identiques partout.** La diversité est l'ennemie.
2. **Ne laissez pas un algorithme faible « parce que le pair ne supporte que ça ».** Documentez une dérogation avec date de fin.
3. **Surveillez les compteurs dans les deux sens** : un tunnel up peut ne rien porter.
4. **Rédigez la fiche des paramètres de chaque tunnel tiers** (propositions, identités, durées, contacts).
5. **Testez le rekey** avant la mise en production, avec une durée courte.
6. **Fixez la MTU et le MSS dès le départ.**
7. **Alignez l'heure** : des certificats et des durées dépendent d'horloges justes.
8. **Ne pas chiffrer deux fois sans raison** (IPsec sur MACsec ou sur SD-WAN) : surcoût et complexité.

### D. Exercices supplémentaires

1. Calculez le surcoût ESP-AES-GCM-256 avec NAT-T pour un paquet interne de 1 000 octets : taille finale et part de surcoût.
2. Traduisez en config strongSwan la proposition « AES-128-GCM, PRF SHA-384, groupe 20 ».
3. Écrivez la procédure en 8 étapes de diagnostic d'un tunnel qui ne monte pas, avec la commande de chaque étape.
4. Expliquez pourquoi un tunnel peut être `up` pour IKE et sans trafic ESP.
5. Un fournisseur impose DH groupe 14 : est-ce acceptable ? Quelles conditions ?

<details>
<summary>Corrigés</summary>

1. Surcoût ≈ 20 + 8 + 8 + 8 + (bourrage 0 à 3) + 2 + 16 ≈ 62 à 65 octets ; paquet final ≈ 1 062 à 1 065 octets ; surcoût ≈ 6 %.
2. `proposals = aes128gcm16-prfsha384-ecp384` (groupe 20 = ECP 384).
4. IKE (UDP 500/4500) passe alors que ESP (protocole 50 ou UDP 4500) est filtré, ou que le routage/les sélecteurs n'envoient pas le trafic dans le tunnel, ou que le retour est bloqué.
5. MODP 2048 (groupe 14) est le minimum acceptable ; si le fournisseur n'offre pas mieux, documenter la dérogation et préférer, dès que possible, un groupe elliptique.

</details>

### E. Mini-projet (8 heures)

Rédigez le **guide d'exploitation IPsec** d'une entreprise : standard cryptographique (propositions, durées, groupes), procédure de création d'un tunnel (checklist), modèles IOS XE et strongSwan, procédures de rekey et de rotation de PSK, tableau des tunnels tiers, arbre de diagnostic, et validation en laboratoire de 5 pannes.

---

## 15. Quiz

1. Combien de messages comporte `IKE_SA_INIT` ?
2. Quelle est la différence entre IKE SA et Child SA ?
3. Quel est le rôle de la PFS ?
4. Quels ports UDP IKE et NAT-T utilise-t-on ?
5. Que signifie `TS_UNACCEPTABLE` ?
6. Pourquoi AES-GCM plutôt qu'AES-CBC + HMAC ?
7. Que fait DPD ?
8. Pourquoi préférer des certificats aux PSK à grande échelle ?

<details>
<summary>Réponses</summary>

1. Deux (requête et réponse). 2. L'IKE SA protège la négociation ; la Child SA protège les données. 3. Empêcher qu'une clé compromise ne révèle d'autres sessions. 4. UDP 500 et UDP 4500. 5. Les sélecteurs de trafic proposés ne sont pas acceptés par le pair. 6. Chiffrement et intégrité en une opération, plus rapide, moins de surcoût. 7. Détecte un pair silencieux et déclenche le nettoyage ou la relance. 8. Évolutivité, révocation, absence de secret partagé.

</details>

---

## 16. Aide-mémoire

```text
IKEv2 : SA_INIT (2) -> AUTH (2) -> CREATE_CHILD_SA (rekey) -> INFORMATIONAL (DPD) ; UDP 500/4500 ; ESP = proto 50
Mode tunnel (site à site) | VTI + routage (préféré) | sélecteurs identiques si policy-based
Recommandé : AES-256-GCM, PRF SHA-256/384, DH 19/20/21/31 (>= 14), PFS, certificats ou PSK >= 32 car.
Durées : IKE 8-24 h, Child 1-8 h ; rekey ~90 % ; DPD 10-30 s
Surcoût ESP-GCM : ~54-57 (65 avec NAT-T) ; MTU tunnel ~1400, MSS ~1360
Diagnostic : joignabilité -> UDP 500/4500 -> SA_INIT -> AUTH -> Child -> compteurs 2 sens -> MTU -> stabilité
Commandes : show crypto ikev2 sa | show crypto ipsec sa | swanctl --list-sas | ip xfrm state
```

## 17. Pour aller plus loin

- RFC 7296 (IKEv2), RFC 4301 (architecture IPsec), RFC 4303 (ESP), RFC 8247 (algorithmes IKEv2) : <https://www.rfc-editor.org>
- NIST SP 800-77 Rev. 1 : <https://csrc.nist.gov/publications/detail/sp/800-77/rev-1/final>
- ANSSI, « Recommandations de sécurité relatives à IPsec » : <https://cyber.gouv.fr>
- strongSwan : <https://docs.strongswan.org>

> Module précédent : [CP02](../CP02-qos-wan/CP02-qos-wan.md) | Module suivant : [CP04](../CP04-flexiwan/CP04-flexiwan.md)
