# SW08. Sécurité intégrée

| | |
|---|---|
| **Partie** | B. Cisco Catalyst SD-WAN |
| **Durée indicative** | 1,5 semaine (12 h) |
| **Prérequis** | SW04 à SW07 ; pare-feu à états, IPS, DNS, TLS ; AAA (TACACS+, RADIUS) |
| **Niveau** | Avancé |
| **Lien examen** | ENSDWI 300-415 : « Sécurité et QoS » ; NIST SP 800-207, SP 800-77 |
| **Lab associé** | T6 (pare-feu, filtrage d'URL, DIA sécurisé) |

> Module précédent : [SW07](../SW07-politiques-centralisees-localisees/SW07-politiques-centralisees-localisees.md) | Module suivant : [SW09](../SW09-cloud-onramp/SW09-cloud-onramp.md)

> **Avertissement.** Les fonctions de sécurité embarquées dépendent de la **licence**, de la **plate-forme** (mémoire, CPU) et de la **version**. Les configurations CLI sont indicatives : le Manager les génère à partir des politiques de sécurité. Vérifiez toujours la matrice de fonctions de votre version avant de promettre une fonction à un client.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Modèle de menace d'un WAN](#2-modèle-de-menace-dun-wan)
3. [Chiffrement du plan de données](#3-chiffrement-du-plan-de-données)
4. [Sécurité du plan de contrôle](#4-sécurité-du-plan-de-contrôle)
5. [La segmentation comme premier niveau](#5-la-segmentation-comme-premier-niveau)
6. [Fonctions de sécurité sur le WAN Edge](#6-fonctions-de-sécurité-sur-le-wan-edge)
7. [Accès sécurisé à Internet : SIG et SSE](#7-accès-sécurisé-à-internet--sig-et-sse)
8. [Zero Trust appliqué au SD-WAN](#8-zero-trust-appliqué-au-sd-wan)
9. [Sécurité du plan de gestion](#9-sécurité-du-plan-de-gestion)
10. [Où appliquer la sécurité : site, cloud ou hybride](#10-où-appliquer-la-sécurité--site-cloud-ou-hybride)
11. [Durcissement : check-list et audit automatisé](#11-durcissement--check-list-et-audit-automatisé)
12. [Vérifications et dépannage](#12-vérifications-et-dépannage)
13. [Lab guidé T6](#13-lab-guidé-t6)
14. [Approfondissements](#14-approfondissements)
15. [Quiz](#15-quiz)
16. [Aide-mémoire](#16-aide-mémoire)
17. [Pour aller plus loin](#17-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Identifier les menaces propres à un WAN et les contrôles qui y répondent.
- Expliquer comment sont chiffrés et authentifiés plans de données et de contrôle.
- Configurer pare-feu, IPS, filtrage d'URL, protection DNS et proxy TLS (principes et vérifications).
- Rediriger le trafic Internet vers un service de sécurité cloud (SIG/SSE).
- Sécuriser le plan de gestion (RBAC, AAA, accès restreint, journalisation).
- Décider où placer chaque fonction (site, cloud, hybride) selon coût, performance et risque.

---

## 2. Modèle de menace d'un WAN

| Menace | Exemple | Contrôle principal |
|---|---|---|
| **Écoute sur le transport** | Capture du trafic sur Internet | Chiffrement IPsec |
| **Usurpation d'équipement** | Un faux routeur tente de rejoindre la fabrique | Certificats + liste d'autorisation |
| **Déplacement latéral** | Un poste infecté atteint les caisses ou l'OT | Segmentation, pare-feu entre segments |
| **Sortie Internet non maîtrisée** | Malware, exfiltration, sites malveillants | Pare-feu, IPS, filtrage DNS/URL, SSE |
| **Compromission du plan de gestion** | Vol d'identifiants du Manager | MFA, RBAC, accès restreint, journalisation |
| **Déni de service** | Saturation des interfaces de transport | `allow-service` minimal, limitation, protection opérateur |
| **Chaîne d'approvisionnement** | Image logicielle altérée | Vérification d'intégrité, sources officielles |
| **Erreur humaine** | Politique qui ouvre trop | Revue, tests, déploiement progressif |

> Les retours d'incidents publiés par les équipes de réponse (CERT, éditeurs) suggèrent que beaucoup d'incidents graves en environnement WAN viennent de **l'erreur de configuration et des identifiants faibles**, bien avant les attaques sophistiquées. Le durcissement de base rapporte plus que n'importe quelle fonction avancée.

---

## 3. Chiffrement du plan de données

### 3.1 Principe dans Catalyst SD-WAN

- Chaque tunnel de données est un **tunnel IPsec ESP** entre TLOC (SW04).
- Il n'y a **pas d'IKE entre les WAN Edge** : chaque équipement génère localement ses clés symétriques (par TLOC et par encapsulation), les **annonce dans OMP** (route TLOC) au Controller, qui les redistribue aux pairs autorisés.
- L'authenticité des annonces repose sur la **connexion de contrôle** authentifiée par certificats : un pair non autorisé ne reçoit pas les clés.

```text
WAN Edge A : génère clé AES (TLOC A)  --OMP-->  Controller  --OMP-->  WAN Edge B, C, D
WAN Edge B : utilise la clé de A pour déchiffrer ce que A lui envoie ; envoie avec SA propre clé à A
```

### 3.2 Paramètres de sécurité

| Paramètre | Valeur usuelle | Rôle |
|---|---|---|
| **Chiffrement** | AES-256 en mode GCM (AEAD) | Confidentialité et intégrité |
| **Rekey** (renouvellement des clés) | 86 400 s (24 h) par défaut (à vérifier) | Limiter l'exposition d'une clé |
| **Fenêtre anti-rejeu** | 512 (valeurs possibles de 64 à 8 192) | Rejeter les paquets rejoués ; augmenter si de nombreux paquets arrivent désordonnés |
| **Intégrité** | HMAC-SHA ou AEAD | Détection de modification |
| **Mode** | Tunnel ESP, encapsulé en UDP (port de l'équipement) | Traversée de NAT |

Extrait indicatif de configuration :

```text
security
 ipsec
  rekey 86400
  replay-window 512
 !
 control
  protocol dtls
 !
```

> Les noms exacts des commandes et les algorithmes disponibles évoluent selon la version ; consultez la configuration générée par le Manager et la documentation de sécurité de votre version. Adoptez toujours les **suites de chiffrement recommandées** par le guide de votre version et par la politique de sécurité de l'entreprise (voir NIST SP 800-77).

### 3.3 Vérifications

```text
show sdwan ipsec inbound-connections
show sdwan ipsec outbound-connections
show sdwan tunnel statistics
show sdwan bfd sessions
```

Éléments à lire : algorithmes et SPI, compteurs de paquets chiffrés/déchiffrés, **rejets anti-rejeu** (un compteur qui augmente peut signaler un désordonnancement fort ou une attaque).

---

## 4. Sécurité du plan de contrôle

| Élément | Mécanisme |
|---|---|
| **Authentification mutuelle** | Certificats X.509 des deux extrémités (SW05) |
| **Autorisation** | Liste des équipements valides |
| **Chiffrement** | DTLS ou TLS sur toutes les connexions de contrôle |
| **Filtrage des services** | `allow-service` limité sur les interfaces de transport |
| **Protection du Validator** | Accès public nécessaire : n'exposer **que** les ports requis |
| **Révocation** | Invalidation d'un équipement dans la liste, révocation du certificat (SW17) |

### 4.1 Réduction de la surface d'attaque sur les interfaces de transport

```text
sdwan
 interface GigabitEthernet1
  tunnel-interface
   encapsulation ipsec
   color biz-internet
   no allow-service all
   allow-service dhcp
   allow-service dns
   allow-service icmp
   allow-service ntp
   allow-service stun
   no allow-service sshd
   no allow-service netconf
   no allow-service bgp
   no allow-service ospf
   no allow-service https
```

Règle d'or : **tout ce qui n'est pas explicitement nécessaire est interdit** sur une interface exposée à Internet. `allow-service all` est admissible en laboratoire, jamais en production.

---

## 5. La segmentation comme premier niveau

La segmentation par VPN (SW06) réduit la surface d'attaque **avant** tout pare-feu. Elle ne vaut que si :

1. l'**accès LAN** place bien chaque terminal dans le bon VPN (VLAN, 802.1X, ports verrouillés) ;
2. les **passerelles entre segments** passent par un point de contrôle (pare-feu, chaînage de services) ;
3. les **fuites de routes** sont interdites ou documentées.

Matrice d'exemple (autorisations entre segments) :

| De \ Vers | CORP | VOICE | IOT | GUEST | Internet | DC |
|---|---|---|---|---|---|---|
| **CORP** | — | Non | Non | Non | Filtré | Oui |
| **VOICE** | Non | — | Non | Non | Non | Oui (SIP) |
| **IOT** | Non | Non | — | Non | Non | Oui (collecte) |
| **GUEST** | Non | Non | Non | — | Oui (filtré) | Non |

---

## 6. Fonctions de sécurité sur le WAN Edge

### 6.1 Vue d'ensemble

| Fonction | Rôle | Remarque |
|---|---|---|
| **Pare-feu d'entreprise** (zone-based, avec reconnaissance des applications) | Contrôle à états entre zones | Toujours disponible sur le plan de données |
| **IPS / IDS** | Détection et blocage d'attaques par signatures | Via le conteneur de services de sécurité (UTD) |
| **Filtrage d'URL** | Autoriser/bloquer par catégorie et réputation | UTD |
| **Protection contre les logiciels malveillants (AMP)** | Réputation de fichiers, analyse | UTD + cloud |
| **Protection DNS** (Umbrella/Secure Access) | Bloquer les domaines malveillants | Redirection DNS vers le service |
| **Proxy TLS/SSL** | Déchiffrer pour inspecter | Coûteux, enjeux légaux et de vie privée |
| **ACL** | Filtrage simple L3/L4 | Localisée |

Le **conteneur UTD** (Unified Threat Defense) s'exécute sur le routeur ; il exige de la **mémoire** et des **ressources dédiées** (profil de ressources à choisir selon le débit). Sur un 8000V de laboratoire, prévoyez 8 Go de RAM ou plus et activez le profil approprié.

### 6.2 Pare-feu à zones (principe et CLI générée)

```text
zone security LAN-CORP
zone security LAN-GUEST
zone security INTERNET
!
class-map type inspect match-any CM-WEB-DNS
 match protocol http
 match protocol https
 match protocol dns
!
policy-map type inspect PM-GUEST-TO-INET
 class type inspect CM-WEB-DNS
  inspect
 class class-default
  drop log
!
zone-pair security ZP-GUEST-INET source LAN-GUEST destination INTERNET
 service-policy type inspect PM-GUEST-TO-INET
!
interface GigabitEthernet4
 zone-member security LAN-GUEST
interface GigabitEthernet1
 zone-member security INTERNET
```

Comportement par défaut : **trafic entre zones différentes interdit** tant qu'une zone-pair avec politique ne l'autorise pas ; trafic **dans** une même zone autorisé.

> Dans Catalyst SD-WAN, on construit ces règles dans **Configuration > Security** (politique de sécurité) en choisissant les zones (liées à des VPN), les règles (sources, destinations, applications) et les actions (**inspect**, **pass**, **drop**). Le Manager traduit en configuration d'équipement.

### 6.3 IPS/IDS

| Paramètre | Choix |
|---|---|
| **Mode** | **Détection** (IDS : journalise) ou **Prévention** (IPS : bloque) |
| **Jeu de signatures** | Équilibré (balanced), connectivité, sécurité renforcée ; mise à jour automatique |
| **Liste d'exceptions** | Pour les faux positifs |
| **Journalisation** | Vers le Manager ou un SIEM (SW17) |

Démarche d'introduction recommandée : **IDS pendant 2 à 4 semaines**, analyse des alertes, réglage des exceptions, puis passage en **IPS**.

### 6.4 Filtrage d'URL

| Élément | Exemple |
|---|---|
| Catégories bloquées | Logiciels malveillants, hameçonnage, contenu adulte, anonymiseurs |
| Réputation | Seuil de réputation minimale |
| Listes | Autorisations et interdictions explicites |
| Page de blocage | Message personnalisé |
| Limite | Pour du HTTPS, sans déchiffrement, le filtrage se base sur le SNI/le nom de domaine |

### 6.5 Protection DNS

Le routeur redirige les requêtes DNS vers un service de réputation (Umbrella, remplacé progressivement par Cisco Secure Access selon l'offre) : les domaines malveillants sont bloqués dès la résolution. Mise en œuvre : clé d'API et jeton de l'organisation dans le Manager, liste des domaines locaux exclus (par exemple le domaine Active Directory), exceptions.

### 6.6 Proxy TLS

| Point | Remarque |
|---|---|
| Déchiffrement | Le routeur agit en « homme du milieu » autorisé avec un certificat d'autorité déployé sur les terminaux |
| Impact | Très forte charge CPU ; réduire à des catégories ciblées |
| Aspects légaux | Respect de la vie privée et du droit du travail ; exclusions (banque, santé) |
| Alternative | Déléguer au SSE (SW12) |

---

## 7. Accès sécurisé à Internet : SIG et SSE

Au lieu de sécuriser localement chaque site, on **envoie le trafic Internet** à un service de sécurité dans le cloud (**SIG** : Secure Internet Gateway, ou **SSE**).

```text
Site --(tunnel IPsec ou GRE automatique)--> Point de présence du service (SIG/SSE) --> Internet
         trafic VPN 10 / VPN 20                  pare-feu, proxy web, DNS, CASB, DLP
```

### 7.1 Principes

| Élément | Détail |
|---|---|
| **Tunnels** | IPsec ou GRE du WAN Edge vers deux points de présence (actif et secours) |
| **Automatisation** | Le Manager peut créer les tunnels et les modèles vers les services pris en charge (Umbrella, Zscaler, services tiers par IPsec/GRE manuel) |
| **Redirection** | Route par défaut ou politique de données dirigeant les VPN choisis vers le tunnel |
| **Résilience** | Deux tunnels (principal, secours), basculement |
| **Cas particuliers** | Exclure du service les destinations internes et les applications qui supportent mal les proxys |

### 7.2 Tunnel vers un service tiers (IOS XE, principe)

```text
interface Tunnel100001
 ip unnumbered GigabitEthernet1
 tunnel source GigabitEthernet1
 tunnel destination 198.51.100.50
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROFILE-SIG
!
ip route vrf 20 0.0.0.0 0.0.0.0 Tunnel100001
```

Le service impose des paramètres (algorithmes, identité, clé pré-partagée) : suivez son guide d'intégration ; **ne mettez jamais la clé en clair** dans un dépôt.

### 7.3 Choix : pare-feu local, SIG ou les deux

Voir section 10 et le module SW12.

---

## 8. Zero Trust appliqué au SD-WAN

Les principes de l'architecture Zero Trust (NIST SP 800-207) se traduisent dans le SD-WAN :

| Principe | Mise en œuvre SD-WAN |
|---|---|
| **Ne jamais faire confiance, toujours vérifier** | Authentification des équipements (certificats), des administrateurs (MFA), des utilisateurs (SSE/ZTNA) |
| **Moindre privilège** | Segmentation, politiques inter-segments strictes, RBAC |
| **Micro-segmentation** | Segments par fonction et par risque (IoT, paiement) |
| **Vérification continue** | Surveillance, journaux, mesures de posture |
| **Hypothèse de compromission** | Journalisation, détection (SIEM), isolement rapide d'un site (révocation) |
| **Politique centralisée** | Politiques gérées dans un point central, appliquées partout |

Le SD-WAN fournit **les canaux et la segmentation** ; la vérification d'identité des utilisateurs et des terminaux relève du **SSE/ZTNA** (SW12) et de l'accès réseau (802.1X, NAC).

---

## 9. Sécurité du plan de gestion

### 9.1 Contrôles à mettre en place

| Contrôle | Détail |
|---|---|
| **Authentification forte** | SSO (SAML/OIDC) avec MFA ; comptes locaux de secours (« break-glass ») en coffre |
| **RBAC** | Rôles : lecture seule, opérateur, administrateur réseau, sécurité ; groupes d'équipements |
| **AAA externe** | TACACS+ (commandes) ou RADIUS pour les équipements |
| **Restriction réseau** | Accès au Manager (443) depuis un réseau d'administration, filtré |
| **Journalisation** | Audit des connexions et des changements → SIEM |
| **API** | Comptes de service au moindre privilège, jetons à durée limitée |
| **Mots de passe** | Aucun mot de passe par défaut ; politique de complexité ; rotation |
| **Sessions** | Délai d'inactivité, verrouillage |

### 9.2 AAA sur un WAN Edge (IOS XE)

```text
aaa new-model
!
tacacs server TAC1
 address ipv4 10.250.0.10
 key <CLE-A-NE-PAS-PUBLIER>
 timeout 3
!
aaa group server tacacs+ TACACS-GRP
 server name TAC1
!
aaa authentication login default group TACACS-GRP local
aaa authorization exec default group TACACS-GRP local
aaa accounting commands 15 default start-stop group TACACS-GRP
!
ip access-list standard ADMIN-SSH
 permit 10.250.0.0 0.0.0.255
!
line vty 0 4
 access-class ADMIN-SSH in
 transport input ssh
 exec-timeout 10 0
!
ip ssh version 2
```

> En SD-WAN, passer par les **modèles/groupes** (AAA, Logging, Security) plutôt que par CLI directe, et conserver un **compte local de secours** dont le mot de passe est dans un coffre.

### 9.3 Politique d'accès à l'équipement (device access policy)

Politique localisée qui limite les services de **gestion** (SSH, SNMP) à des préfixes autorisés : à appliquer sur tous les équipements, avec exception pour le réseau d'administration.

### 9.4 Journalisation

```text
logging host 10.250.0.20 transport udp port 514
logging trap informational
logging source-interface Loopback0
service timestamps log datetime msec
```

Utilisez, selon la version, l'interface de gestion adaptée comme source ; envoyez en **TCP/TLS** si le SIEM le permet.

---

## 10. Où appliquer la sécurité : site, cloud ou hybride

| Critère | Sécurité sur le site (pare-feu embarqué) | Sécurité dans le cloud (SIG/SSE) | Hybride |
|---|---|---|---|
| **Latence** | Minimale | +5 à 30 ms selon la proximité du point de présence | Variable |
| **Débit** | Limité par le routeur (fonctions activées) | Élevé (mutualisé) | Équilibré |
| **Coût** | Licences et matériel plus puissants | Abonnement par utilisateur ou par débit | Les deux |
| **Exploitation** | Politiques par site, mises à jour des signatures | Gestion centralisée, mise à jour par le fournisseur | Plus complexe |
| **Couverture utilisateurs distants** | Non | Oui | Oui |
| **Visibilité** | Par site | Globale | Globale |
| **Dépendance** | Aucune | Disponibilité du fournisseur | Partielle |
| **Souveraineté** | Maîtrisée | À vérifier (localisation des points de présence) | À vérifier |

Règles pratiques : **petits sites nombreux** → cloud ; **sites avec exigences locales** (OT, santé, contraintes réglementaires) → local ; **télétravailleurs** → cloud ; **flux internes inter-segments** → local.

---

## 11. Durcissement : check-list et audit automatisé

### 11.1 Check-list

| # | Contrôle | Fait |
|---|---|---|
| 1 | Aucun `allow-service all` sur les interfaces exposées | |
| 2 | Mots de passe par défaut changés partout | |
| 3 | SSO + MFA sur le Manager | |
| 4 | RBAC en place, comptes nominatifs | |
| 5 | Accès au Manager limité à un réseau d'administration | |
| 6 | Journaux envoyés vers le SIEM | |
| 7 | Certificats : suivi des échéances | |
| 8 | NTP authentifié ou source interne fiable | |
| 9 | SNMP v3 uniquement (si utilisé) | |
| 10 | Segmentation testée (matrice) | |
| 11 | Sauvegardes chiffrées et testées | |
| 12 | Mises à jour de sécurité suivies (PSIRT) | |

### 11.2 Audit automatisé des configurations

Script Python qui analyse des exports de configuration (un fichier par équipement) et signale les écarts :

```python
#!/usr/bin/env python3
"""Audit simple de durcissement sur des exports de 'show sdwan running-config'.

Usage : python3 audit_durcissement.py exports/*.txt
"""
import re
import sys
from pathlib import Path

REGLES = [
    ("allow-service all sur une interface", re.compile(r"^\s*allow-service all\s*$", re.M), "CRITIQUE"),
    ("Mot de passe/secret en clair (type 0)", re.compile(r"\b(password|secret) 0 ", re.M), "CRITIQUE"),
    ("SNMP v1/v2c avec communauté", re.compile(r"snmp-server community ", re.M), "MOYEN"),
    ("Telnet autorisé sur vty", re.compile(r"transport input .*telnet", re.M), "CRITIQUE"),
    ("Pas de journalisation distante", None, "MOYEN"),
    ("Banner de connexion absent", None, "FAIBLE"),
]

def auditer(chemin: Path):
    texte = chemin.read_text(encoding="utf-8", errors="replace")
    resultats = []
    for nom, motif, gravite in REGLES:
        if nom == "Pas de journalisation distante":
            if not re.search(r"^logging host ", texte, re.M):
                resultats.append((gravite, nom))
        elif nom == "Banner de connexion absent":
            if "banner" not in texte:
                resultats.append((gravite, nom))
        elif motif.search(texte):
            resultats.append((gravite, nom))
    return resultats

if __name__ == "__main__":
    total = 0
    for f in sys.argv[1:]:
        res = auditer(Path(f))
        print(f"\n== {f} : {len(res)} constat(s)")
        for gravite, nom in sorted(res):
            print(f"  [{gravite}] {nom}")
        total += len(res)
    sys.exit(1 if total else 0)
```

Intégrez ce script dans une chaîne d'intégration continue (SW15) : toute dérive est détectée automatiquement.

---

## 12. Vérifications et dépannage

```text
show sdwan zbfw zonepair-statistics          # compteurs par zone-pair (inspectés, rejetés)
show sdwan zbfw drop-statistics              # raisons des rejets
show sdwan security-info                     # résumé des paramètres de sécurité
show policy-firewall stats all               # statistiques pare-feu (IOS XE)
show utd engine standard status              # état du moteur UTD (IPS, URL, AMP)
show utd engine standard config              # configuration appliquée
show app-hosting list                        # conteneurs applicatifs (UTD)
show sdwan ipsec inbound-connections
show ip nat translations vrf 20
```

| Symptôme | Causes probables | Vérifications |
|---|---|---|
| Trafic légitime bloqué | Règle de zone trop stricte, application mal identifiée, IPS faux positif | Statistiques de rejets, journaux, mode IDS |
| UTD non démarré | Ressources insuffisantes, licence, profil de ressources | `show app-hosting list`, mémoire disponible |
| Débit chute après activation | Fonctions trop nombreuses pour la plate-forme | CPU, passage de certaines fonctions au cloud |
| Filtrage d'URL inefficace en HTTPS | Pas de déchiffrement ; SNI absent | Politique de proxy TLS ou SSE |
| Tunnels SIG instables | MTU, NAT, paramètres IPsec | `show crypto ipsec sa`, MTU, journaux |
| DNS non filtré | Clients utilisent un DNS public (DoH) | Bloquer DoH, forcer le DNS interne |

---

## 13. Lab guidé T6

### 13.1 Objectif

Activer pare-feu et filtrage d'URL sur un site, assurer un DIA sécurisé pour le segment invités, puis (si disponible) rediriger ce segment vers un service SIG.

### 13.2 Étapes

| # | Action | Vérification |
|---|---|---|
| 1 | Partir du lab T3 (VPN 10 CORP, VPN 20 GUEST avec DIA) | Ping Internet depuis GUEST |
| 2 | Créer des **zones** : `GUEST` (VPN 20), `CORP` (VPN 10), `INTERNET` (VPN 0) | Zones listées |
| 3 | Règle **GUEST → INTERNET** : inspecter HTTP/HTTPS/DNS, rejeter le reste ; **GUEST → CORP** : rejeter | `show sdwan zbfw zonepair-statistics` |
| 4 | Tests : navigation OK ; ping vers CORP échoue ; SSH vers Internet bloqué | Captures |
| 5 | Activer le **filtrage d'URL** sur GUEST (catégories malveillantes, adultes) ; tester `testsite` de catégorie | Page de blocage |
| 6 | Activer l'**IPS en mode détection**, générer une alerte de test (outil de test, laboratoire uniquement), consulter l'alerte | Alertes |
| 7 | Restreindre `allow-service` aux services nécessaires sur l'interface de transport ; refaire un test de connectivité | Tunnels toujours up |
| 8 | Optionnel : créer un tunnel vers un SIG de test et rediriger GUEST | Trafic visible dans la console du service |
| 9 | Exécuter `audit_durcissement.py` sur l'export de configuration | Rapport |
| 10 | Rédiger le rapport | |

### 13.3 Livrables et grille

| Élément | Points |
|---|---|
| Zones et règles correctes, preuve par tests | 5 |
| Filtrage d'URL et IPS démontrés | 4 |
| Réduction de la surface (`allow-service`) | 3 |
| Audit automatisé et correction des constats | 3 |
| Analyse du choix site vs cloud | 3 |
| Qualité du rapport | 2 |

---

## 14. Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **ZBFW** | Pare-feu à zones : politique appliquée entre zones de sécurité |
| **UTD** | Unified Threat Defense : conteneur de services de sécurité sur le routeur |
| **IDS / IPS** | Détection / prévention d'intrusion |
| **Signature** | Motif décrivant une attaque connue |
| **SIG** | Secure Internet Gateway : service de sécurité Internet dans le cloud |
| **SSE** | Security Service Edge : partie sécurité du SASE |
| **SWG** | Passerelle web sécurisée |
| **CASB** | Courtier de sécurité d'accès au cloud |
| **ZTNA** | Accès réseau Zero Trust par application |
| **DLP** | Prévention des fuites de données |
| **RBAC** | Contrôle d'accès par rôle |
| **AAA** | Authentification, autorisation, traçabilité |
| **MFA** | Authentification multifacteur |
| **PSIRT** | Équipe de réponse aux incidents de sécurité du produit chez Cisco |
| **Break-glass** | Compte de secours à usage exceptionnel, très surveillé |
| **Anti-replay** | Mécanisme de rejet des paquets rejoués |
| **DoH** | DNS sur HTTPS, qui contourne le DNS d'entreprise |

### B. Cas d'usage détaillés

#### Cas 1. Réseau de magasins : conformité des paiements

| Rubrique | Détail |
|---|---|
| **Exigence** | Isoler les terminaux de paiement (norme PCI DSS) et prouver l'isolation |
| **Solution** | VPN dédié `PAY` (jamais de sortie Internet générale), flux restreints vers l'hébergeur de paiement, pare-feu entre `PAY` et le reste, journalisation vers le SIEM |
| **Preuves d'audit** | Matrice de flux, configuration exportée, tests trimestriels de segmentation, journal des changements |
| **Leçon** | La preuve compte autant que la protection : **automatisez la production des preuves** (SW15, SW17) |

#### Cas 2. Filiale à l'étranger avec exigences locales

La filiale doit sortir sur Internet par un fournisseur local imposé. Solution : DIA local avec pare-feu embarqué, tunnels vers le siège pour les applications internes, filtrage d'URL selon la réglementation locale. **Leçon** : le cloud de sécurité unique n'est pas toujours possible ; la **sécurité sur le site** reste une option indispensable.

#### Cas 3. Attaque par rançongiciel : isoler un site en 5 minutes

| Étape | Action |
|---|---|
| Détection | Alertes IPS et flux anormaux sur un site |
| Confinement | Politique de données refusant tout trafic du VPN CORP du site, **ou** invalidation du WAN Edge dans la liste (tunnels coupés) |
| Preuve | Journaux et captures conservés |
| Reprise | Nettoyage, restauration, réintégration par validation |
| Leçon | Le **plan d'isolement** doit être **écrit, simple et testé** (SW17) |

#### Cas 4. Sous-dimensionnement après activation du pare-feu et de l'IPS

Un site à 500 Mbit/s voit son débit chuter à 180 Mbit/s après activation de l'IPS et du filtrage. Cause : le routeur choisi n'a pas la marge de calcul. Solution : routeur supérieur, ou IPS dans le cloud pour les flux Internet et pare-feu local pour l'inter-segments. **Leçon** : dimensionnez avec **toutes les fonctions activées** et prenez le chiffre de performance **« avec sécurité »** du constructeur, pas celui du routeur nu.

#### Cas 5. Administrateur qui quitte l'entreprise

Comptes partagés « admin » connus de six personnes : le départ d'un administrateur impose de changer tous les mots de passe et de vérifier les accès. Avec SSO, comptes nominatifs et MFA, la désactivation d'un compte suffit. **Leçon** : **pas de comptes partagés** (hors compte de secours en coffre).

#### Cas 6. L'IPS bloque une application métier

Une application utilise un protocole atypique, une signature se déclenche. Procédure : repasser la signature en détection sur ce flux, confirmer le faux positif avec l'éditeur, ajouter une exception **ciblée** (source, destination, signature), documenter, **suivre l'échéance de révision**. **Leçon** : ne désactivez jamais l'IPS globalement pour un faux positif.

### C. Retours d'expérience

1. **Commencez par le simple.** Chiffrement, `allow-service` minimal, MFA, segmentation : 80 % du gain pour 20 % de l'effort.
2. **IDS avant IPS.** Un IPS déployé sans phase d'observation bloque tôt ou tard quelque chose d'important, un vendredi à 17 h.
3. **Méfiez-vous du déchiffrement TLS systématique.** Coûteux, juridiquement sensible, source d'incidents (applications qui rejettent le certificat).
4. **La sécurité du Manager est la sécurité de tout.** Quelqu'un qui contrôle le Manager contrôle le réseau : traitez-le comme un contrôleur de domaine.
5. **Gérez les exceptions comme une dette.** Chaque exception doit avoir un propriétaire, une raison, une date de révision.
6. **Testez vos règles avec un outil indépendant.** Un scan depuis chaque segment prouve la matrice mieux que la lecture de la configuration.
7. **Le DNS est un angle mort.** DoH et résolveurs publics contournent le filtrage : forcez et surveillez le DNS.
8. **Gardez les signatures à jour et vérifiez qu'elles le sont.** Un IPS aux signatures périmées donne un faux sentiment de sécurité.

### D. Exercices supplémentaires

1. Établissez la **matrice de flux** de votre entreprise fictive (5 segments) et traduisez-la en règles de zones.
2. Rédigez la procédure d'**isolement d'un site compromis** en 6 étapes, avec la commande ou l'action Manager de chaque étape.
3. Comparez, pour 80 petits sites et 3 000 télétravailleurs, une solution pare-feu local et une solution SSE (coûts, risques, exploitation).
4. Étendez le script d'audit : ajoutez 4 règles de votre choix (par exemple `ip http server` activé, `snmp-server community public`, absence de NTP, `ip ssh version 1`).
5. Votre IPS produit 1 200 alertes par jour. Décrivez la démarche pour les réduire à 20 alertes exploitables.

<details>
<summary>Corrigés</summary>

2. (1) Décider de l'isolement (responsable sécurité) ; (2) politique de données `drop` sur le VPN concerné ou invalidation du WAN Edge ; (3) vérifier la coupure (`show sdwan bfd sessions`) ; (4) préserver les journaux et capturer le trafic ; (5) analyser et nettoyer ; (6) réintégrer après validation et durcissement.
3. Local : coût matériel et licences par site, exploitation répartie, pas de couverture télétravail ; SSE : abonnement par utilisateur, politique unique, couverture des télétravailleurs, dépendance au fournisseur. Hybride probable : SSE pour Internet et télétravail, pare-feu local pour l'inter-segments.
5. Regrouper par signature et source, identifier les faux positifs (exceptions ciblées), supprimer les signatures non pertinentes pour votre environnement, élever la gravité des alertes critiques, corréler avec le SIEM, boucler par une revue mensuelle.

</details>

### E. Mini-projet (8 heures)

Rédigez le **référentiel de sécurité d'un SD-WAN** pour une entreprise fictive : modèle de menace, matrice de flux, politique de sécurité (zones, règles, IPS, URL, DNS), modèle de gestion (RBAC, MFA, AAA), plan d'isolement d'incident, check-list de recette sécurité et script d'audit. Faites-le relire par un pair et défendez-le oralement.

---

## 15. Quiz

1. Pourquoi n'y a-t-il pas d'IKE entre les WAN Edge dans Catalyst SD-WAN ?
2. Quelle est la différence entre IDS et IPS ?
3. Que signifie la détection de rejeu (anti-replay) ?
4. Pourquoi `allow-service all` est-il dangereux sur Internet ?
5. Citez trois avantages d'un SIG par rapport à un pare-feu local.
6. Quelle commande donne les rejets du pare-feu à zones ?
7. Pourquoi faut-il un compte « break-glass » ?
8. Quelle fonction de sécurité protège contre les domaines malveillants dès la résolution ?

<details>
<summary>Réponses</summary>

1. Les clés sont générées localement et distribuées par le plan de contrôle (OMP) authentifié ; les équipements n'ont pas besoin de négocier entre eux. 2. L'IDS détecte et journalise ; l'IPS bloque. 3. Rejet de paquets déjà reçus (rejoués) grâce à un numéro de séquence et une fenêtre. 4. Il expose tous les services de l'équipement (SSH, NETCONF, etc.) à Internet. 5. Couverture des télétravailleurs, gestion centralisée, mise à jour assurée par le fournisseur, débit mutualisé. 6. `show sdwan zbfw drop-statistics`. 7. Pour retrouver un accès si le SSO ou l'AAA tombe. 8. La protection DNS (Umbrella / Secure Access).

</details>

---

## 16. Aide-mémoire

```text
Données  : IPsec ESP (AES-256-GCM), clés locales annoncées dans OMP, rekey 24 h, anti-replay 512
Contrôle : certificats + liste d'autorisation + DTLS/TLS + allow-service minimal
Site     : ZBFW (inspect/pass/drop), IPS/IDS (UTD), URL, AMP, DNS, proxy TLS (rare)
Cloud    : SIG/SSE par tunnels IPsec/GRE doubles ; DNS et web filtrés
Gestion  : SSO+MFA, RBAC, TACACS+, accès restreint, logs -> SIEM, comptes de service, break-glass
Vérifier : show sdwan zbfw zonepair-statistics | drop-statistics | utd engine standard status | ipsec inbound-connections
Démarche : IDS puis IPS ; exceptions ciblées et datées ; tester la matrice avec un scan
```

## 17. Pour aller plus loin

- NIST SP 800-207, Zero Trust Architecture : <https://csrc.nist.gov/publications/detail/sp/800-207/final>
- NIST SP 800-77, Guide to IPsec VPNs : <https://csrc.nist.gov/publications/detail/sp/800-77/rev-1/final>
- Cisco Catalyst SD-WAN Security Configuration Guide (votre version) : <https://www.cisco.com/c/en/us/solutions/enterprise-networks/sd-wan/index.html>
- Cisco PSIRT, avis de sécurité : <https://sec.cloudapps.cisco.com/security/center/publicationListing.x>

> Module précédent : [SW07](../SW07-politiques-centralisees-localisees/SW07-politiques-centralisees-localisees.md) | Module suivant : [SW09](../SW09-cloud-onramp/SW09-cloud-onramp.md)
