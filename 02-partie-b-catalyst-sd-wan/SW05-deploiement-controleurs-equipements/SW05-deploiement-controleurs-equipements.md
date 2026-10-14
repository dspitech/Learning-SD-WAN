# SW05. Déploiement des contrôleurs et intégration des équipements

| | |
|---|---|
| **Partie** | B. Cisco Catalyst SD-WAN |
| **Durée indicative** | 1,5 à 2 semaines (14 h) |
| **Prérequis** | SW04 ; PKI et certificats X.509 ; NTP ; pare-feu et NAT ; virtualisation (VMware, KVM ou CML) |
| **Niveau** | Avancé |
| **Lien examen** | ENSDWI 300-415 : « Déploiement des contrôleurs » et « Déploiement des routeurs » |
| **Lab associé** | T2 (intégration d'un site : contrôleurs, certificats, WAN Edge, connexions de contrôle) |

> Module précédent : [SW04](../SW04-architecture-composants/SW04-architecture-composants.md) | Module suivant : [SW06](../SW06-modeles-groupes-segmentation/SW06-modeles-groupes-segmentation.md)

> **Avertissement.** La syntaxe CLI et les chemins de menus de l'interface varient selon la version (20.x). Les blocs de commandes sont **indicatifs** : validez-les avec `?`, `show sdwan running-config` et la documentation de votre version. Certains équipements et licences exigent un compte Cisco (Smart Account, Plug and Play).

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Préparation : prérequis réseau et ressources](#2-préparation--prérequis-réseau-et-ressources)
3. [Plan d'adressage du laboratoire](#3-plan-dadressage-du-laboratoire)
4. [Déploiement des contrôleurs](#4-déploiement-des-contrôleurs)
5. [Configuration initiale en ligne de commande des contrôleurs](#5-configuration-initiale-en-ligne-de-commande-des-contrôleurs)
6. [PKI et certificats](#6-pki-et-certificats)
7. [Lab PKI d'entreprise avec OpenSSL](#7-lab-pki-dentreprise-avec-openssl)
8. [Ajout des contrôleurs dans le Manager et installation des certificats](#8-ajout-des-contrôleurs-dans-le-manager-et-installation-des-certificats)
9. [Liste des WAN Edge autorisés](#9-liste-des-wan-edge-autorisés)
10. [Intégration d'un WAN Edge](#10-intégration-dun-wan-edge)
11. [Synchronisation horaire](#11-synchronisation-horaire)
12. [Mises à jour logicielles](#12-mises-à-jour-logicielles)
13. [Sauvegardes et restauration](#13-sauvegardes-et-restauration)
14. [Dépannage de l'intégration](#14-dépannage-de-lintégration)
15. [Lab guidé T2 complet](#15-lab-guidé-t2-complet)
16. [Exercices](#16-exercices)
17. [Quiz](#17-quiz)
18. [Aide-mémoire](#18-aide-mémoire)
19. [Pour aller plus loin](#19-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Dimensionner et préparer le réseau pour des contrôleurs (adresses, ports, DNS, NTP).
- Installer les contrôleurs et réaliser leur configuration d'amorçage.
- Mettre en place une PKI (Cisco ou d'entreprise), signer et installer les certificats.
- Constituer la liste des équipements autorisés.
- Intégrer un WAN Edge par PnP, par jeton ou manuellement.
- Planifier les mises à jour (ordre, compatibilité, retour arrière) et les sauvegardes.

---

## 2. Préparation : prérequis réseau et ressources

### 2.1 Check-list avant installation

| Domaine | Exigence |
|---|---|
| **Adresses publiques** | Validator : adresse publique (ou NAT statique 1:1) ; Controller et Manager : joignables par les WAN Edge |
| **DNS** | Nom résolvant vers les adresses des Validators (par exemple `vbond.lab.example`) |
| **NTP** | Source de temps fiable accessible par **tous** les composants |
| **Pare-feu** | Ports ouverts (voir ci-dessous) |
| **Compte Cisco** | Smart Account et Virtual Account pour la liste d'autorisation PnP |
| **Licences** | Licences logicielles (DNA / Catalyst SD-WAN) |
| **Certificats** | Choix : Cisco PKI, PKI d'entreprise ou manuel |
| **Versions** | Compatibilité Manager / Controller / Validator / WAN Edge (matrice de compatibilité) |

### 2.2 Flux à autoriser

| De | Vers | Protocole et port | Rôle |
|---|---|---|---|
| WAN Edge | Validator | UDP 12346 (et décalages) | Orchestration |
| WAN Edge | Controller | UDP 12346 ou TCP 23456 | OMP |
| WAN Edge | Manager | UDP 12346 ou TCP 23456 | Gestion |
| WAN Edge | WAN Edge | UDP 12346 / 4500 (IPsec NAT-T) | Tunnels de données |
| Administrateur | Manager | TCP 443 | Interface et API |
| Controllers | Manager | UDP 12346 ou TCP 23456 | Politiques |
| Tous | NTP | UDP 123 | Heure |
| Tous | DNS | UDP et TCP 53 | Résolution |
| Manager | Internet (selon besoins) | TCP 443 | Mises à jour, cloud Cisco (PnP, licences) |

### 2.3 Ressources indicatives (laboratoire minimal)

> Valeurs de laboratoire. Pour la production, **appliquer les tailles de la documentation** (nombre d'équipements, statistiques).

| Composant | vCPU | RAM | Disque |
|---|---|---|---|
| Manager | 4 à 8 | 16 à 32 Go | 100 à 500 Go + disque de données |
| Controller | 2 | 4 à 8 Go | 16 à 20 Go |
| Validator | 2 | 4 Go | 10 à 20 Go |
| WAN Edge Catalyst 8000V | 2 à 4 | 4 à 8 Go | 8 Go |

---

## 3. Plan d'adressage du laboratoire

### 3.1 Topologie

```text
                         Réseau de transport simulé (underlay)
                                 Internet : 203.0.113.0/24
                                 MPLS     : 192.0.2.0/24
   +-----------+   +------------+   +-----------+
   | Validator |   | Controller |   |  Manager  |            Hub (site 10)     Agence (site 100)
   |  .10      |   |  .20       |   |   .30     |            +---------+      +----------+
   +-----+-----+   +-----+------+   +-----+-----+            | HUB-E1  |      | BR1-E1   |
         |               |                |                  +---+--+--+      +---+--+---+
         +---------------+----------------+-- Internet --------/     \---- MPLS --/     \
                                                    ge1 : Internet   ge2 : MPLS
```

### 3.2 Tableau d'adresses

| Équipement | Rôle | System IP | Site ID | VPN 0 : Internet | VPN 0 : MPLS |
|---|---|---|---|---|---|
| vbond1 | Validator | 10.255.255.3 | 1 | 203.0.113.10/24 | — |
| vsmart1 | Controller | 10.255.255.2 | 1 | 203.0.113.20/24 | — |
| vmanage1 | Manager | 10.255.255.1 | 1 | 203.0.113.30/24 | — |
| HUB-E1 | WAN Edge | 10.255.0.1 | 10 | 203.0.113.101/24 | 192.0.2.101/24 |
| BR1-E1 | WAN Edge | 10.255.1.100 | 100 | 203.0.113.110/24 | 192.0.2.110/24 |

| Paramètre | Valeur |
|---|---|
| Organisation | `LAB-SDWAN` |
| Passerelle Internet | 203.0.113.1 |
| NTP | 203.0.113.5 |
| DNS | 203.0.113.53 |
| Nom du Validator | `vbond.lab.example` |

---

## 4. Déploiement des contrôleurs

### 4.1 Ordre d'installation

1. **Validator** : premier arrêt des équipements.
2. **Manager** : interface et PKI.
3. **Controller**.
4. **Configuration d'amorçage** des trois (identifiants, VPN 0, adresse du Validator).
5. **Certificats**.
6. **WAN Edge**.

### 4.2 Types d'images

| Plate-forme | Format |
|---|---|
| VMware ESXi | OVA / OVF |
| KVM / CML / EVE-NG | qcow2 |
| AWS, Azure, GCP | AMI, VHD, image selon l'offre |
| Cisco hébergé | Provisionné par Cisco (pas d'installation) |

### 4.3 Premier démarrage

À la première ouverture de session sur un contrôleur :
- se connecter avec les identifiants par défaut de la version (consulter la documentation de l'image) ;
- **changer immédiatement le mot de passe** ;
- configurer l'adresse de gestion, l'organisation, l'adresse du Validator.

Le **Manager** exige en plus l'initialisation d'un disque de données (assistant en ligne de commande lors de la première utilisation) avant de pouvoir démarrer les services.

---

## 5. Configuration initiale en ligne de commande des contrôleurs

### 5.1 Validator (vBond)

```text
vbond1# config
vbond1(config)# system
vbond1(config-system)# host-name vbond1
vbond1(config-system)# system-ip 10.255.255.3
vbond1(config-system)# site-id 1
vbond1(config-system)# organization-name "LAB-SDWAN"
vbond1(config-system)# vbond 203.0.113.10 local
vbond1(config-system)# exit
vbond1(config)# vpn 0
vbond1(config-vpn-0)# interface ge0/0
vbond1(config-interface-ge0/0)# ip address 203.0.113.10/24
vbond1(config-interface-ge0/0)# no shutdown
vbond1(config-interface-ge0/0)# tunnel-interface
vbond1(config-tunnel-interface)# encapsulation ipsec
vbond1(config-tunnel-interface)# allow-service all
vbond1(config-tunnel-interface)# exit
vbond1(config-interface-ge0/0)# exit
vbond1(config-vpn-0)# ip route 0.0.0.0/0 203.0.113.1
vbond1(config-vpn-0)# exit
vbond1(config)# vpn 512
vbond1(config-vpn-512)# interface eth0
vbond1(config-interface-eth0)# ip dhcp-client
vbond1(config-interface-eth0)# no shutdown
vbond1(config-interface-eth0)# exit
vbond1(config-vpn-512)# exit
vbond1(config)# commit and-quit
```

Le mot-clé **`local`** à la fin de la ligne `vbond` désigne ce nœud comme **Validator**. Pour la production, remplacez `allow-service all` par la liste précise de services (`dhcp`, `dns`, `icmp`, `ntp`, `sshd`, `stun`, etc.).

### 5.2 Controller (vSmart)

```text
vsmart1# config
vsmart1(config)# system
vsmart1(config-system)# host-name vsmart1
vsmart1(config-system)# system-ip 10.255.255.2
vsmart1(config-system)# site-id 1
vsmart1(config-system)# organization-name "LAB-SDWAN"
vsmart1(config-system)# vbond vbond.lab.example
vsmart1(config-system)# exit
vsmart1(config)# vpn 0
vsmart1(config-vpn-0)# interface eth1
vsmart1(config-interface-eth1)# ip address 203.0.113.20/24
vsmart1(config-interface-eth1)# no shutdown
vsmart1(config-interface-eth1)# tunnel-interface
vsmart1(config-tunnel-interface)# encapsulation ipsec
vsmart1(config-tunnel-interface)# allow-service all
vsmart1(config-tunnel-interface)# exit
vsmart1(config-interface-eth1)# exit
vsmart1(config-vpn-0)# ip route 0.0.0.0/0 203.0.113.1
vsmart1(config-vpn-0)# exit
vsmart1(config)# commit and-quit
```

### 5.3 Manager (vManage)

La configuration d'amorçage du Manager est similaire (system, VPN 0 avec l'interface de transport et la route par défaut). **Le Manager se configure ensuite principalement par l'interface web.**

```text
vmanage1# config
vmanage1(config)# system
vmanage1(config-system)# host-name vmanage1
vmanage1(config-system)# system-ip 10.255.255.1
vmanage1(config-system)# site-id 1
vmanage1(config-system)# organization-name "LAB-SDWAN"
vmanage1(config-system)# vbond vbond.lab.example
vmanage1(config-system)# exit
vmanage1(config)# vpn 0
vmanage1(config-vpn-0)# interface eth0
vmanage1(config-interface-eth0)# ip address 203.0.113.30/24
vmanage1(config-interface-eth0)# no shutdown
vmanage1(config-interface-eth0)# tunnel-interface
vmanage1(config-tunnel-interface)# encapsulation ipsec
vmanage1(config-tunnel-interface)# exit
vmanage1(config-interface-eth0)# exit
vmanage1(config-vpn-0)# ip route 0.0.0.0/0 203.0.113.1
vmanage1(config-vpn-0)# exit
vmanage1(config)# commit and-quit
```

### 5.4 Vérifications d'amorçage

```text
show interface description
show ip route vpn 0           (ou show ip route sur certaines versions)
ping 203.0.113.1 vpn 0
ping 203.0.113.5 vpn 0        (NTP)
show clock
show control local-properties
```

> **Règles d'or** : l'**organization-name** doit être **identique et au caractère près** sur **tous** les composants ; le **Validator** doit être joignable ; l'**horloge** doit être correcte avant toute opération de certificat.

---

## 6. PKI et certificats

### 6.1 Rôle des certificats

Chaque composant s'authentifie par un certificat X.509 signé par une autorité de certification (CA) de confiance. Un contrôleur ou équipement **ne se connecte que si** :

1. son certificat est valide (dates, signature, chaîne de confiance) ;
2. l'**organization name** correspond ;
3. pour un WAN Edge : son numéro de série/châssis figure dans la **liste d'autorisation** distribuée par le Manager.

### 6.2 Options d'autorité de certification

| Option | Pour qui | Fonctionnement |
|---|---|---|
| **Cisco PKI** | Hébergement Cisco, simplicité | Les certificats des contrôleurs sont signés automatiquement via les services de Cisco |
| **Enterprise CA** (PKI d'entreprise) | Auto-hébergement, laboratoire, souveraineté | Le Manager génère les demandes de signature (CSR) ; vous les signez avec votre CA ; vous installez les certificats et la **racine** sur tous les composants |
| **Manuel** | Cas particuliers | Génération de CSR, signature et import manuels |
| **WAN Edge matériels** | Tous | Certificat de fabrique (SUDI) installé en usine par Cisco |
| **WAN Edge virtuels** | Tous | Certificat obtenu via **jeton** (OTP) et services Cisco ou CA d'entreprise |

### 6.3 Durée de vie et renouvellement

| Élément | Remarque |
|---|---|
| Certificats des contrôleurs | Durée limitée (à consulter dans l'interface) : prévoir un **suivi des échéances** |
| Certificats WAN Edge | Durée longue, mais renouvelable |
| Certificat racine | Doit être installé sur **tous** les composants ; un changement de racine impose une procédure planifiée |
| Interface web du Manager | Certificat HTTPS distinct (peut être signé par une CA d'entreprise) |

> **Sans certificat valide, la connexion de contrôle échoue.** Un certificat expiré est l'une des premières causes de panne en production (voir SW10 et SW17).

---

## 7. Lab PKI d'entreprise avec OpenSSL

Création d'une **racine de laboratoire** pour signer les contrôleurs. À n'utiliser **qu'en laboratoire**.

### 7.1 Racine

```bash
mkdir -p ~/sdwan-ca && cd ~/sdwan-ca

# Clé privée de la racine (RSA 4096) protégée par mot de passe
openssl genrsa -aes256 -out ROOTCA.key 4096

# Certificat racine auto-signé (5 ans ici : à adapter)
openssl req -x509 -new -nodes -key ROOTCA.key -sha256 -days 1825 \
  -subj "/C=FR/O=LAB-SDWAN/OU=PKI/CN=LAB-SDWAN Root CA" \
  -out ROOTCA.pem

openssl x509 -in ROOTCA.pem -noout -subject -issuer -dates
```

### 7.2 Signature d'un CSR de contrôleur

Le CSR est généré dans le Manager (**Configuration > Certificates > Controllers > ... > Generate CSR**), puis téléchargé (par exemple `vsmart1.csr`).

```bash
# Fichier d'extensions
cat > ext.cnf <<'EOF'
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
EOF

# Signature (2 ans)
openssl x509 -req -in vsmart1.csr -CA ROOTCA.pem -CAkey ROOTCA.key \
  -CAcreateserial -out vsmart1.crt -days 730 -sha256 -extfile ext.cnf

# Contrôle
openssl x509 -in vsmart1.crt -noout -subject -issuer -dates
openssl verify -CAfile ROOTCA.pem vsmart1.crt
```

> **Attention** : le sujet du certificat doit contenir l'**organization name** (champ `O` ou `OU`, selon ce que génère le Manager). Ne modifiez pas le sujet du CSR. Les paramètres d'extension exigés (par exemple `serverAuth`/`clientAuth`) sont décrits dans la documentation de la version : **vérifiez-les**.

### 7.3 Installer la racine sur les contrôleurs et les équipements

Sur un Validator ou un Controller (copie préalable du fichier `ROOTCA.pem` dans `/home/admin`) :

```text
vbond1# request root-cert-chain install /home/admin/ROOTCA.pem
vbond1# show control local-properties | include root
```

Sur un **WAN Edge IOS XE** :

```text
BR1-E1# request platform software sdwan root-cert-chain install bootflash:ROOTCA.pem
BR1-E1# show sdwan certificate root-ca-cert
```

Pour le Manager, la racine se charge dans **Administration > Settings > Controller Certificate Authorization > Enterprise Root Certificate**.

---

## 8. Ajout des contrôleurs dans le Manager et installation des certificats

### 8.1 Paramètres d'administration (interface web)

**Administration > Settings** :

| Paramètre | Valeur |
|---|---|
| Organization Name | `LAB-SDWAN` (identique partout) |
| Validator (vBond) | `vbond.lab.example` ou IP, port 12346 |
| Controller Certificate Authorization | Enterprise Root Certificate (laboratoire) ou Cisco PKI |
| Hardware WAN Edge Certificate Authorization | Cisco PKI (par défaut) |
| Web Server Certificate | Optionnel (certificat HTTPS de confiance) |
| Enforce Software Version (ZTP) | Selon politique |

### 8.2 Ajouter les contrôleurs

**Configuration > Devices > Controllers > Add Controller** :

| Champ | Validator | Controller |
|---|---|---|
| Type | vBond | vSmart |
| IP | 203.0.113.10 | 203.0.113.20 |
| Utilisateur et mot de passe | admin | admin |
| Génération du CSR | Oui | Oui |
| Protocole | DTLS | DTLS |

### 8.3 Signer et installer

1. **Configuration > Certificates > Controllers** : pour chaque contrôleur, **Generate CSR**, puis **View CSR / Download**.
2. Signez les CSR avec votre CA (section 7).
3. **Install Certificate** : collez ou téléchargez le certificat signé.
4. Le contrôleur apparaît avec la mention de certificat installé.

### 8.4 Contrôle

Sur chaque contrôleur :

```text
show certificate installed
show control connections
show control local-properties
show orchestrator connections          (Validator)
show orchestrator valid-vedges         (Validator : liste des équipements autorisés)
```

Résultat attendu au Manager : **Monitor > Overview** montre des Controllers et un Validator en « verts ».

---

## 9. Liste des WAN Edge autorisés

### 9.1 Principe

Le Validator n'accepte qu'un équipement dont le **numéro de châssis et le numéro de série (ou jeton)** figurent dans la liste. La liste est constituée dans le Manager.

### 9.2 Constitution

| Méthode | Description |
|---|---|
| **Synchronisation Smart Account / PnP Connect** | Dans **Configuration > Devices > WAN Edge List**, **Sync Smart Account** : importe les équipements du portail (<https://software.cisco.com> > Plug and Play Connect) |
| **Chargement d'un fichier** | Fichier `.viptela` (téléchargé du portail) ou CSV |
| **Ajout manuel** | Équipement virtuel : génération d'un jeton (voir ci-dessous) |

Après import, chaque équipement est à l'état **Valid**, **Invalid** ou **Staging** ; il faut **Valid** puis **Send to Controllers** pour que la liste soit distribuée.

### 9.3 Exemple de CSV (équipements virtuels)

```csv
Chassis Number,Serial Number,Hostname
C8K-11111111-2222-3333-4444-555555555555,12345678,HUB-E1
C8K-66666666-7777-8888-9999-000000000000,87654321,BR1-E1
```

> Les formats exacts des fichiers sont ceux produits par le portail : utilisez de préférence la **synchronisation** et conservez les exports du portail dans un coffre de secrets/dépôt privé.

---

## 10. Intégration d'un WAN Edge

### 10.1 Trois méthodes

| Méthode | Principe | Quand l'utiliser |
|---|---|---|
| **PnP (Plug and Play)** | Le routeur contacte le service de redirection Cisco avec son numéro de série et reçoit l'adresse du Validator | Équipements matériels neufs, déploiement sans intervention |
| **Jeton / fichier d'amorçage** | Fichier de configuration initial (par exemple « cloud-init » pour un 8000V) contenant UUID, jeton, Validator, organisation | Équipements virtuels, laboratoire |
| **Configuration manuelle** | Saisie de la configuration minimale en CLI + activation | Dépannage, laboratoire, sites sans accès DHCP |

### 10.2 Mode contrôleur (« controller mode »)

Un routeur Catalyst 8000 démarre soit en mode **autonome** (IOS XE classique), soit en mode **contrôleur** (SD-WAN). Pour passer en mode contrôleur (le routeur redémarre) :

```text
Router# controller-mode enable
```

Retour en mode autonome :

```text
Router# controller-mode disable
```

> Les routeurs livrés pour le SD-WAN démarrent souvent déjà en mode contrôleur. Réinitialiser la configuration SD-WAN : `request platform software sdwan config reset`.

### 10.3 Méthode manuelle (laboratoire)

**Étape 1 : récupérer UUID et jeton.** Dans le Manager, **Configuration > Devices > WAN Edge List**, ligne de l'équipement : **Generate Bootstrap Configuration** ou **Generate OTP** (selon version).

**Étape 2 : configuration minimale d'amorçage (CLI)** sur BR1-E1 :

```text
BR1-E1# config-transaction
BR1-E1(config)# hostname BR1-E1
BR1-E1(config)# interface GigabitEthernet1
BR1-E1(config-if)#  description TRANSPORT-INTERNET
BR1-E1(config-if)#  ip address 203.0.113.110 255.255.255.0
BR1-E1(config-if)#  no shutdown
BR1-E1(config-if)# exit
BR1-E1(config)# interface GigabitEthernet2
BR1-E1(config-if)#  description TRANSPORT-MPLS
BR1-E1(config-if)#  ip address 192.0.2.110 255.255.255.0
BR1-E1(config-if)#  no shutdown
BR1-E1(config-if)# exit
BR1-E1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
BR1-E1(config)# ip name-server 203.0.113.53
BR1-E1(config)# system
BR1-E1(config-system)#  system-ip 10.255.1.100
BR1-E1(config-system)#  site-id 100
BR1-E1(config-system)#  organization-name LAB-SDWAN
BR1-E1(config-system)#  sp-organization-name LAB-SDWAN
BR1-E1(config-system)#  vbond vbond.lab.example
BR1-E1(config-system)# exit
BR1-E1(config)# sdwan
BR1-E1(config-sdwan)#  interface GigabitEthernet1
BR1-E1(config-interface-GigabitEthernet1)#   tunnel-interface
BR1-E1(config-tunnel-interface)#    encapsulation ipsec
BR1-E1(config-tunnel-interface)#    color biz-internet
BR1-E1(config-tunnel-interface)#    allow-service all
BR1-E1(config-tunnel-interface)#   exit
BR1-E1(config-interface-GigabitEthernet1)#  exit
BR1-E1(config-sdwan)#  interface GigabitEthernet2
BR1-E1(config-interface-GigabitEthernet2)#   tunnel-interface
BR1-E1(config-tunnel-interface)#    encapsulation ipsec
BR1-E1(config-tunnel-interface)#    color mpls
BR1-E1(config-tunnel-interface)#    allow-service all
BR1-E1(config-tunnel-interface)#   exit
BR1-E1(config-interface-GigabitEthernet2)#  exit
BR1-E1(config-sdwan)# exit
BR1-E1(config)# commit
```

> `sp-organization-name` désigne le nom d'organisation du fournisseur de service (identique à `organization-name` dans les déploiements mono-organisation).

**Étape 3 : activer l'équipement avec UUID et jeton** :

```text
BR1-E1# request platform software sdwan vedge_cloud activate chassis-number C8K-66666666-7777-8888-9999-000000000000 token 1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d
```

### 10.4 Fichier d'amorçage de type cloud-init (équipement virtuel)

```yaml
#cloud-config
vinitparam:
 - uuid : C8K-66666666-7777-8888-9999-000000000000
 - otp  : 1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d
 - vbond: 203.0.113.10
 - org  : LAB-SDWAN
```

Le fichier s'attache à l'image au démarrage (volume CD-ROM ou *user data* dans le cloud, selon l'hyperviseur). Le **Manager** peut générer ce fichier : **Configuration > Devices > ... > Generate Bootstrap Configuration** (option Cloud-init).

### 10.5 PnP

1. Le routeur démarre sans configuration, obtient une adresse par DHCP et le DNS.
2. Il contacte le service de redirection (le nom est géré par Cisco : par exemple `devicehelper.cisco.com` pour PnP sur IOS XE ; certains anciens flux utilisent `ztp.viptela.com`).
3. Le portail PnP Connect, où l'équipement est associé à votre contrôleur, renvoie l'adresse du Validator.
4. Le reste suit la séquence habituelle (SW04, section 10).

Prérequis : DHCP avec DNS fonctionnel, accès sortant HTTPS, équipement présent dans le **profil de contrôleur** du portail PnP.

### 10.6 Premier contrôle sur le WAN Edge

```text
BR1-E1# show sdwan control connections
BR1-E1# show sdwan control local-properties
BR1-E1# show sdwan certificate installed
BR1-E1# show sdwan certificate serial
BR1-E1# show sdwan control connections-history
BR1-E1# show clock
BR1-E1# ping 203.0.113.10
```

Dans le Manager : l'équipement doit apparaître **Reachable** dans **Monitor > Devices** (ou **Configuration > Devices > WAN Edge List**).

---

## 11. Synchronisation horaire

L'heure est cruciale : un certificat n'est valide que **entre ses dates**. Un décalage de plusieurs minutes (parfois d'heures) provoque un rejet.

Sur les contrôleurs (VPN 0) :

```text
vsmart1(config)# system ntp server 203.0.113.5 vpn 0 version 4 prefer
vsmart1(config)# commit
vsmart1# show ntp associations
```

Sur le WAN Edge (modèle NTP ou CLI) :

```text
BR1-E1(config)# ntp server 203.0.113.5
BR1-E1# show ntp associations
BR1-E1# show clock
```

> Sur IOS XE SD-WAN, le VPN 0 correspond à la table de routage globale : `ntp server 203.0.113.5` s'écrit sans VRF (via la VRF `Mgmt-intf` si le NTP est joint par le VPN 512). Vérifiez sur votre version. En laboratoire, si NTP est indisponible, réglez l'heure à la main (`clock set` sur IOS XE) **avant** l'installation des certificats.

---

## 12. Mises à jour logicielles

### 12.1 Ordre

```text
1. Sauvegarder (configuration, base de données du Manager, snapshots).
2. SD-WAN Manager
3. SD-WAN Validator
4. SD-WAN Controller
5. WAN Edge (par vagues : pilote puis le reste)
Règle : les contrôleurs doivent être à une version supérieure ou égale à celle des WAN Edge.
```

### 12.2 Compatibilité

Consulter la **matrice de compatibilité** de la version cible (Manager, Controllers, WAN Edge, plates-formes). Certaines versions imposent des **sauts intermédiaires**.

### 12.3 Procédure côté Manager

**Maintenance > Software Upgrade** :

1. **Software Repository** : charger l'image (ou la télécharger depuis un dépôt distant).
2. Sélectionner les équipements, **Upgrade** (installation).
3. **Activate** (redémarrage sur la nouvelle version).
4. **Set Default Version** une fois la version validée.

Un minuteur de **retour arrière automatique** (confirm timer) rétablit l'ancienne version si l'équipement ne rétablit pas ses connexions de contrôle après activation.

### 12.4 Commandes de vérification

```text
show sdwan software
show sdwan version
show version | include Version
```

### 12.5 Stratégie de vagues

| Vague | Équipements | Critère de passage |
|---|---|---|
| 0 | Contrôleurs de laboratoire, puis de production en dehors des heures | Connexions up, OMP stable |
| 1 | 2 à 3 sites pilotes | 48 h sans incident |
| 2 | 10 à 20 % du parc | Pas de régression |
| 3 | Le reste, par lots | Surveillance |

---

## 13. Sauvegardes et restauration

| Quoi | Comment | Fréquence |
|---|---|---|
| **Base de configuration du Manager** | `request nms configuration-db backup path /home/admin/db_backup` | Quotidienne |
| **Snapshot de la VM** | Hyperviseur / cloud | Avant chaque mise à jour |
| **Modèles et politiques** | Export JSON via API (voir SW15) vers Git | À chaque changement |
| **Configurations des WAN Edge** | Export via le Manager, ou `show sdwan running-config` | Hebdomadaire |
| **Certificats et racine** | Copie chiffrée hors du Manager | À chaque émission |
| **Fichiers de la PKI d'entreprise** | Coffre-fort (clé de la racine hors ligne) | — |

Restauration de la base de configuration :

```text
vmanage1# request nms configuration-db restore path /home/admin/db_backup.tar.gz
```

> Testez **régulièrement la restauration** dans un environnement séparé : une sauvegarde jamais restaurée est une hypothèse, pas une garantie.

---

## 14. Dépannage de l'intégration

### 14.1 Codes d'échec courants des connexions de contrôle (extrait)

La commande `show sdwan control connections-history` affiche l'historique et le **motif** de la dernière défaillance :

| Code | Signification probable | Action |
|---|---|---|
| `CRTREJSER` | Certificat rejeté par le pair (par exemple équipement absent de la liste, certificat non valide) | Vérifier la liste d'autorisation, le certificat |
| `CRTVERFL` | Échec de vérification du certificat (racine absente, date invalide) | Installer la racine, corriger l'heure |
| `SERNTPRES` | Numéro de série absent de la liste | Importer/valider l'équipement |
| `DCONFAIL` | Échec de connexion DTLS/TLS | Pare-feu, routage, port |
| `NOVMCFG` | Pas de configuration côté Manager pour l'équipement | Associer un modèle/groupe |
| `NOACTVB` | Aucun Validator actif | DNS, route, Validator injoignable |
| `LISFD` | Échec d'écoute de l'interface locale | Interface sans `tunnel-interface` |
| `DISTLOC` | TLOC désactivé | Revoir la configuration de l'interface |
| `BIDNTVRFD` | Identité de la carte non vérifiée | Certificat de châssis |

> Liste officielle et à jour : documentation Cisco « control connection error codes » de votre version.

### 14.2 Arbre de décision

```text
Pas de connexion de contrôle ?
 |-- ping Validator OK ?
 |     non -> routage VPN 0 / DNS / pare-feu / interface down
 |     oui -> poursuivre
 |-- Heure correcte ?
 |     non -> NTP / clock set
 |-- Certificat installé et valide ? (show sdwan certificate installed)
 |     non -> installer (PnP, jeton, CA)
 |-- Racine installée ? (show sdwan certificate root-ca-cert)
 |     non -> request ... root-cert-chain install
 |-- Équipement dans la liste "Valid" et envoyée aux contrôleurs ?
 |     non -> WAN Edge List : Valid + Send to Controllers
 |-- organization-name identique partout ?
 |-- Ports UDP 12346 / TCP 23456 ouverts de bout en bout ?
```

---

## 15. Lab guidé T2 complet

### 15.1 Objectif

Déployer un environnement minimal (Validator, Controller, Manager, un hub et une agence), installer les certificats par une PKI d'entreprise et obtenir des **connexions de contrôle actives** et des **tunnels BFD** entre les deux WAN Edge.

### 15.2 Étapes

| # | Action | Vérification |
|---|---|---|
| 1 | Créer la topologie (voir section 3) dans CML, EVE-NG ou GNS3 ; relier les contrôleurs à un commutateur « Internet » avec routeur de sortie et NTP | Ping entre tous les nœuds de transport |
| 2 | Configurer l'amorçage des trois contrôleurs (section 5) | `show control local-properties` |
| 3 | Régler NTP ou l'heure | `show clock` identique (± quelques secondes) |
| 4 | Créer la racine OpenSSL (section 7.1) | `openssl x509 -in ROOTCA.pem -noout -dates` |
| 5 | Dans le Manager : paramètres (organisation, Validator, autorité Enterprise), charger la racine | Pas d'erreur |
| 6 | Installer la racine sur Validator et Controller (`request root-cert-chain install`) | `show control local-properties` mentionne la racine |
| 7 | Ajouter le Validator et le Controller dans le Manager | Statut visible |
| 8 | Générer les CSR, signer, installer | `show certificate installed` |
| 9 | Vérifier les connexions entre contrôleurs | `show control connections` sur le Controller : `vmanage` et `vbond` up |
| 10 | Déclarer HUB-E1 et BR1-E1 dans la **WAN Edge List** (Valid) puis **Send to Controllers** | `show orchestrator valid-vedges` |
| 11 | Passer les 8000V en mode contrôleur, importer la racine, configurer l'amorçage, activer avec UUID et jeton | `show sdwan control connections` |
| 12 | Dans le Manager : associer un modèle ou un groupe de configuration minimal (SW06) ou vérifier que les tunnels montent avec la configuration CLI | `show sdwan bfd sessions` |

### 15.3 Résultats attendus (extraits)

```text
BR1-E1# show sdwan control connections
PEER    PEER PEER            SITE DOMAIN PEER            PRIV  PEER            PUB                          
TYPE    PROT SYSTEM IP       ID   ID     PRIVATE IP      PORT  PUBLIC IP       PORT  LOCAL COLOR   STATE UPTIME
vsmart  dtls 10.255.255.2    1    1      203.0.113.20    12346 203.0.113.20    12346 biz-internet  up    0:00:12:30
vmanage dtls 10.255.255.1    1    0      203.0.113.30    12346 203.0.113.30    12346 biz-internet  up    0:00:12:28

BR1-E1# show sdwan bfd sessions
SYSTEM IP    SITE ID STATE SOURCE TLOC COLOR REMOTE TLOC COLOR SOURCE IP       DST PUBLIC IP   ENCAP
10.255.0.1   10      up    biz-internet      biz-internet       203.0.113.110   203.0.113.101   ipsec
10.255.0.1   10      up    mpls              mpls               192.0.2.110     192.0.2.101     ipsec
```

### 15.4 Livrables

- Schéma de la topologie et tableau d'adressage renseigné.
- Capture des sorties : `show sdwan control connections`, `show sdwan certificate installed`, `show sdwan bfd sessions`.
- Journal des problèmes rencontrés (au moins deux) avec cause et correction.
- Export de la configuration d'amorçage de chaque équipement (secrets masqués).

### 15.5 Grille

| Critère | Points |
|---|---|
| Contrôleurs joignables et authentifiés | 5 |
| PKI correcte (racine, CSR, signature) | 5 |
| WAN Edge liste + intégration | 5 |
| Tunnels BFD up | 3 |
| Documentation et preuves | 2 |

---

## 16. Exercices

1. Votre Manager est hébergé par Cisco. Quelles étapes de ce module disparaissent ou changent ?
2. Pourquoi faut-il valider l'équipement dans la liste **et** envoyer la liste aux contrôleurs ?
3. Un WAN Edge de laboratoire n'obtient pas de connexion de contrôle. L'heure du routeur est en 2019. Que se passe-t-il ?
4. Dans quel ordre mettre à jour : WAN Edge, Controller, Manager, Validator ?
5. Proposez un plan de sauvegarde pour un Manager de production.

<details>
<summary>Corrigés</summary>

1. L'installation des VM, la PKI des contrôleurs (Cisco PKI automatique), les sauvegardes de base de données de Cisco ; restent : la liste des WAN Edge, la configuration des sites, les politiques, l'intégration des routeurs, la sécurité des comptes.
2. Le Manager héberge la liste ; seul le **Validator** (et les contrôleurs) en tient compte après distribution ; sans l'envoi, ils ne la connaissent pas.
3. Le certificat est évalué comme **non encore valide** (date antérieure à sa date de début) : échec `CRTVERFL`. Corriger l'heure.
4. Manager → Validator → Controller → WAN Edge.
5. Sauvegarde quotidienne de la base, snapshot avant mise à jour, export JSON des modèles et politiques dans Git, copie hors site chiffrée, test de restauration trimestriel, procédure documentée.

</details>

---

## 17. Quiz

1. Quelle commande installe la racine sur un WAN Edge IOS XE ?
2. À quoi sert le mot-clé `local` dans `vbond <ip> local` ?
3. Quel élément relie l'équipement à son profil dans le portail PnP ?
4. Pourquoi l'organization name doit-il être identique partout ?
5. Quel est l'ordre de mise à jour des composants ?
6. Quelle commande donne l'historique d'échec des connexions de contrôle ?

<details>
<summary>Réponses</summary>

1. `request platform software sdwan root-cert-chain install bootflash:<fichier>`. 2. Il indique que ce nœud est le Validator. 3. Le numéro de série / châssis (et le profil de contrôleur). 4. Il fait partie de la vérification d'identité : une différence provoque un rejet. 5. Manager, Validator, Controller, WAN Edge. 6. `show sdwan control connections-history`.

</details>

---

## 18. Aide-mémoire

```text
Amorçage contrôleur : system {host-name, system-ip, site-id, organization-name, vbond} + vpn 0 {interface, tunnel-interface, route}
Racine              : request root-cert-chain install <fichier>   (Validator, Controller)
                      request platform software sdwan root-cert-chain install bootflash:<fichier> (WAN Edge)
Activation (8000V)  : request platform software sdwan vedge_cloud activate chassis-number <UUID> token <OTP>
Mode contrôleur     : controller-mode enable
Contrôles           : show sdwan control connections | connections-history | certificate installed | root-ca-cert
Sauvegarde Manager  : request nms configuration-db backup path <fichier>
Ordre MAJ           : Manager -> Validator -> Controller -> WAN Edge
```

## 19. Pour aller plus loin

- Guide de démarrage Cisco Catalyst SD-WAN (Getting Started) pour votre version : <https://www.cisco.com/c/en/us/solutions/enterprise-networks/sd-wan/index.html>
- Plug and Play Connect : <https://software.cisco.com>
- OpenSSL, documentation : <https://www.openssl.org/docs/>

---

## Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **CA / PKI** | Autorité de certification / infrastructure à clés publiques |
| **CSR** | Demande de signature de certificat |
| **Chaîne de confiance** | Certificat feuille → intermédiaire → racine |
| **OTP / token** | Jeton à usage unique pour authentifier un équipement virtuel |
| **PnP Connect** | Portail Cisco de redirection des équipements vers leur contrôleur |
| **Smart Account / Virtual Account** | Comptes de licences et d'équipements Cisco |
| **Controller mode** | Mode de fonctionnement SD-WAN d'un routeur IOS XE |
| **Day-0 / Day-1 / Day-2** | Amorçage / configuration de service / exploitation |
| **Bootstrap (cloud-init)** | Fichier de configuration initiale injecté au démarrage |
| **Confirm timer** | Délai de validation après activation d'une version, sinon retour arrière |
| **Admin-tech** | Archive de diagnostic à envoyer au support Cisco |
| **Configuration DB** | Base de configuration du Manager (à sauvegarder) |
| **Cluster** | Ensemble de nœuds Manager |
| **Disaster Recovery (DR)** | Reprise après sinistre : Manager de secours sur un autre site |

### B. Cas d'usage détaillés

#### Cas 1. Démarrage d'un projet : pré-requis qui bloquent

| Blocage rencontré | Cause | Prévention |
|---|---|---|
| Aucune connexion de contrôle au premier jour | Port UDP 12346 filtré par le pare-feu du siège | Liste de flux validée avant le démarrage |
| Certificats refusés | NTP non joignable depuis la VLAN de gestion | Test de NTP dès l'amorçage |
| Équipements « introuvables » dans PnP | Associés au mauvais Virtual Account | Contrôle croisé commande ↔ portail |
| Validator injoignable depuis Internet | NAT dynamique devant le Validator | NAT statique 1:1 |

**Leçon** : 80 % des échecs de déploiement sont des **échecs de pré-requis**, pas de configuration SD-WAN.

#### Cas 2. Expiration de certificat un vendredi soir

Les certificats des Controllers expirent. Sans surveillance, tous les sites perdent leur connexion de contrôle ; les tunnels survivent le temps du graceful restart. **Parade** : suivi des échéances à 90, 60 et 30 jours (SW17), procédure de renouvellement testée, fenêtre de maintenance.

#### Cas 3. Mise à jour qui tourne mal

| Étape | Action |
|---|---|
| Préparation | Snapshots, sauvegarde de la base, lecture des notes de version |
| Pilote | 3 sites de test, 48 h d'observation |
| Incident | Sur un modèle d'équipement, une fonction n'est plus acceptée après mise à jour : la poussée de configuration échoue |
| Retour arrière | Le confirm timer a restauré l'ancienne version sur 2 sites |
| Leçon | **Ne jamais mettre à jour tout le parc le même jour**, même si la version est « mineure » |

#### Cas 4. 80 routeurs expédiés en 3 jours (déploiement de masse)

Processus : import de la liste depuis le portail PnP, groupe de configuration par rôle, variables par CSV, **étiquetage** des colis avec le nom du site, fiche d'instruction d'une page pour le personnel du site, suivi en temps réel dans le Manager. Résultat : 74 sites en ligne en 2 jours ; 6 retardataires (accès Internet non livré, câble débranché, box en double NAT). **Leçon** : prévoir un **centre d'appels de déploiement** avec un script de questions.

#### Cas 5. Reprise après perte du Manager

Le Manager est détruit. Les tunnels continuent, mais plus de supervision ni de changement. Récupération : restauration de la base depuis la sauvegarde sur une nouvelle VM, réinstallation des certificats si nécessaire, resynchronisation. **Leçon** : le **temps de reprise** mesuré en exercice vaut mieux que celui du plan.

### C. Retours d'expérience

1. **Documentez les mots de passe et la racine PKI avant d'en avoir besoin.** La clé privée de la racine d'entreprise est un secret majeur : hors ligne, chiffrée, deux copies.
2. **Ne mettez pas à jour un vendredi.** Ni la veille de congés.
3. **Une VM clonée n'est pas un nouvel équipement.** Identifiants et certificats doivent être régénérés.
4. **La liste des équipements est un actif de sécurité.** Un équipement retiré du parc doit être **invalidé** dans la liste.
5. **Testez la restauration.** Beaucoup de sauvegardes de Manager ne se restaurent pas lors de la première tentative.
6. **Écrivez les procédures avec des captures d'écran** : l'équipe d'exploitation les suivra sous stress.
7. **Sur un 8000V, vérifiez licence et débit** : un routeur virtuel est bridé par sa licence et par ses ressources.

### D. Exercices supplémentaires

1. Rédigez la **check-list de pré-requis** (réseau, DNS, NTP, licences, comptes, certificats) qu'un client doit valider avant le démarrage de votre projet.
2. Calculez les dates de suivi d'un certificat qui expire le 15 mars : seuils 90, 60 et 30 jours.
3. Rédigez une procédure de **retour arrière** pour la mise à jour d'un Controller.
4. Proposez un plan de **reprise après sinistre** du Manager avec objectifs RTO = 4 h et RPO = 24 h.

<details>
<summary>Corrigés</summary>

2. 90 j avant : 15 décembre ; 60 j : 14 janvier ; 30 j : 13 février (années non bissextiles).
3. Snapshot préalable ; mise à jour hors heures ; contrôle `show sdwan control connections` et `omp peers` ; en cas d'échec : `request platform software sdwan software activate` de l'ancienne version ou restauration du snapshot ; vérification de l'OMP et des politiques.
4. Sauvegarde quotidienne copiée hors site (RPO 24 h), image de Manager prête à l'emploi (ou modèle de VM), procédure de restauration documentée et testée, accès aux certificats et à la racine, DNS prêt à rediriger (RTO 4 h).

</details>

### E. Mini-projet (6 heures)

Montez de bout en bout l'environnement du lab T2 en **scriptant** ce qui peut l'être (génération de la PKI OpenSSL, fichiers cloud-init, configurations d'amorçage). Livrez un **runbook** (guide pas à pas) permettant à un collègue de reconstruire l'environnement en moins de 3 heures.

---

> Module précédent : [SW04](../SW04-architecture-composants/SW04-architecture-composants.md) | Module suivant : [SW06](../SW06-modeles-groupes-segmentation/SW06-modeles-groupes-segmentation.md)
