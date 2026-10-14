# CP02. QoS pour le WAN : classification, files d'attente et lissage

| | |
|---|---|
| **Partie** | Compléments (hors parcours principal) |
| **Durée indicative** | 1,5 semaine (12 h) |
| **Prérequis** | CCNA (notions de QoS) ; SW03, SW07 ; Linux (tc) |
| **Niveau** | Avancé |
| **Lien examen** | ENCOR 350-401 (QoS), ENSDWI 300-415 (QoS), CCNP Enterprise |
| **Lab associé** | Lab CP02 (QoS sous Linux avec tc : priorité de la voix sous congestion) |

> Module précédent : [CP01](../CP01-routage-avance/CP01-routage-avance.md) | Module suivant : [CP03](../CP03-ipsec-en-profondeur/CP03-ipsec-en-profondeur.md)

> **Pourquoi ce complément.** La qualité de service est l'un des sujets les plus mal compris en WAN. On promet « de la QoS » sans la dimensionner, sans lisser le débit, ou en oubliant que l'opérateur Internet ne respecte pas le marquage. Ce module donne la méthode et les calculs pour que la QoS apporte vraiment quelque chose.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Pourquoi la QoS : congestion, délai, gigue, perte](#2-pourquoi-la-qos--congestion-délai-gigue-perte)
3. [Le modèle DiffServ](#3-le-modèle-diffserv)
4. [Plan de classes recommandé](#4-plan-de-classes-recommandé)
5. [Dimensionner la bande passante des classes](#5-dimensionner-la-bande-passante-des-classes)
6. [Files d'attente et ordonnancement](#6-files-dattente-et-ordonnancement)
7. [Policing et shaping](#7-policing-et-shaping)
8. [Gestion de la congestion : WRED et tail-drop](#8-gestion-de-la-congestion--wred-et-tail-drop)
9. [QoS hiérarchique sur IOS XE (MQC)](#9-qos-hiérarchique-sur-ios-xe-mqc)
10. [QoS dans Catalyst SD-WAN](#10-qos-dans-catalyst-sd-wan)
11. [QoS de bout en bout et sur Internet](#11-qos-de-bout-en-bout-et-sur-internet)
12. [Vérification et dépannage](#12-vérification-et-dépannage)
13. [Outils de calcul](#13-outils-de-calcul)
14. [Lab guidé CP02](#14-lab-guidé-cp02)
15. [Approfondissements](#15-approfondissements)
16. [Quiz](#16-quiz)
17. [Aide-mémoire](#17-aide-mémoire)
18. [Pour aller plus loin](#18-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Expliquer quand la QoS est utile et quand elle ne l'est pas.
- Concevoir un plan de classes (marquage DSCP) cohérent.
- Dimensionner les files par calcul : voix, vidéo, applications, masse.
- Configurer files, lissage et gestion de congestion sur IOS XE (MQC) et sous Linux (tc).
- Appliquer la QoS dans Catalyst SD-WAN et comprendre ses limites sur Internet.
- Vérifier et dépanner une QoS.

---

## 2. Pourquoi la QoS : congestion, délai, gigue, perte

### 2.1 Quand la QoS sert-elle ?

La QoS **n'ajoute pas de bande passante**. Elle décide **qui attend et qui est sacrifié lorsqu'un lien est congestionné**.

| Situation | QoS utile ? |
|---|---|
| Lien rarement chargé (< 50 % en pointe) | Peu |
| Lien parfois saturé (pointes à 100 %) | **Oui** |
| Lien toujours saturé | **Non : il faut plus de débit** (la QoS répartit la pénurie) |
| Lien rapide, trafic rare | Non (mais conserver le marquage) |

### 2.2 Les composantes du délai

```text
Délai total = sérialisation + propagation + file d'attente + traitement
Sérialisation = taille du paquet (bits) / débit du lien
```

Exemples de sérialisation :

| Paquet | 1 Mbit/s | 10 Mbit/s | 100 Mbit/s |
|---|---|---|---|
| 200 octets (voix) | 1,6 ms | 0,16 ms | 0,016 ms |
| 1 500 octets (donnée) | **12 ms** | 1,2 ms | 0,12 ms |

Sur un lien de 1 Mbit/s, un paquet de données de 1 500 octets en cours d'émission fait attendre la voix **12 ms** : d'où la fragmentation (LFI) sur les très petits débits. À partir de quelques dizaines de Mbit/s, la sérialisation est négligeable : le **temps d'attente en file** devient le facteur essentiel.

### 2.3 Effet d'une file qui se remplit

```text
Une file de 100 paquets de 1 500 octets sur un lien de 10 Mbit/s = 100 × 1 500 × 8 / 10 000 000 = 120 ms de délai ajouté
```

C'est le phénomène de **bufferbloat** : des files trop grandes transforment la congestion en latence. Une bonne QoS borne les files des classes sensibles.

---

## 3. Le modèle DiffServ

### 3.1 Principe (RFC 2474, 2475)

Les paquets sont **marqués** dans l'en-tête IP (champ DSCP, 6 bits) puis chaque routeur applique un **comportement par saut** (PHB) selon la classe.

```text
Octet ToS (8 bits) : | DSCP (6 bits) | ECN (2 bits) |
DSCP 46 (EF) = 101110  -> octet ToS = 101110 00 = 0xB8 = 184
```

### 3.2 Valeurs de DSCP courantes

| Classe | DSCP | Binaire | ToS (décimal) | Usage |
|---|---|---|---|---|
| **EF** | 46 | 101110 | 184 | Voix (flux média) |
| **AF41** | 34 | 100010 | 136 | Vidéo interactive |
| **AF31** | 26 | 011010 | 104 | Applications métier critiques |
| **AF21** | 18 | 010010 | 72 | Applications transactionnelles |
| **CS6** | 48 | 110000 | 192 | Contrôle réseau (routage, BFD) |
| **CS5** | 40 | 101000 | 160 | Signalisation de diffusion vidéo |
| **CS3** | 24 | 011000 | 96 | Signalisation voix (SIP) |
| **CS1** | 8 | 001000 | 32 | Arrière-plan, sauvegardes (scavenger) |
| **Défaut (BE)** | 0 | 000000 | 0 | Reste |

Formule de conversion : `ToS = DSCP × 4` ; `DSCP = ToS ÷ 4`.

### 3.3 Les familles AF

Les classes **AFxy** ont une classe **x** (1 à 4, plus haut = priorité) et une **précédence d'abandon y** (1 = faible, 3 = forte). Exemple : AF31, AF32, AF33 appartiennent à la même file, mais AF33 est abandonné plus tôt sous congestion.

---

## 4. Plan de classes recommandé

### 4.1 Un plan à 6 classes pour un WAN d'entreprise

| Classe | DSCP | Exemples | Traitement | Part indicative |
|---|---|---|---|---|
| **Voix** | EF (46) | RTP des appels | Priorité stricte, bornée | 10 à 20 % |
| **Contrôle / signalisation** | CS6 / CS3 | BFD, routage, SIP | Garantie minimale | 5 % |
| **Vidéo interactive** | AF41 (34) | Visioconférence | Bande passante garantie, WRED | 15 à 25 % |
| **Applications critiques** | AF31 (26) | ERP, VDI | Bande passante garantie, WRED | 20 à 30 % |
| **Par défaut** | 0 | Web, e-mail | Reste, WRED | 25 à 40 % |
| **Arrière-plan** | CS1 (8) | Sauvegardes, mises à jour | Minimum, premier sacrifié | 1 à 5 % |

### 4.2 Règles de conception

1. **Peu de classes** (4 à 8) : un plan à 12 classes n'est ni compris ni respecté.
2. **Marquer le plus près possible de la source**, **faire confiance** seulement aux appareils maîtrisés, **re-marquer** le reste.
3. Même plan sur **tous les sites** ; documenté et publié.
4. **Ne pas tout marquer EF** : la file prioritaire non bornée affame les autres.
5. **Contrôle réseau prioritaire** : BFD et routage ne doivent jamais être perdus par congestion (BFD en CS6 par défaut dans Catalyst SD-WAN, DSCP 48).
6. **Marquer les applications chiffrées** : la reconnaissance par signature est limitée ; marquez à la source (politique de groupe, client, passerelle).

---

## 5. Dimensionner la bande passante des classes

### 5.1 Débit d'un appel voix

| Codec | Charge utile | Paquet IP (RTP/UDP/IP) | Débit IP | Avec Ethernet (+18 octets) | Avec Ethernet et IPsec (+~60 octets) |
|---|---|---|---|---|---|
| **G.711** (20 ms) | 160 octets | 200 octets | 80 kbit/s | 87,2 kbit/s | ≈ 111 kbit/s |
| **G.729** (20 ms) | 20 octets | 60 octets | 24 kbit/s | 31,2 kbit/s | ≈ 55 kbit/s |
| **Opus** (variable) | 20 à 100+ octets | variable | 20 à 60 kbit/s | + en-têtes | + en-têtes |

Calcul G.711 : `(160 + 12 RTP + 8 UDP + 20 IP) = 200 octets × 50 paquets/s × 8 = 80 000 bit/s`.

Avec chiffrement IPsec (environ 60 octets de surcoût par paquet à 50 paquets/s) et en-tête Ethernet : `(200 + 60 + 18) × 50 × 8 = 111 200 bit/s`. **Le chiffrement augmente nettement le débit des petits paquets** : le surcoût relatif est énorme pour G.729.

### 5.2 Dimensionnement de la file voix

```text
Bande passante voix = appels simultanés × débit par appel (avec en-têtes et chiffrement) × (1 + marge)
```

Exemple : 30 appels G.711 chiffrés (≈ 111 kbit/s) avec 20 % de marge : `30 × 111,2 × 1,2 = 4 003 kbit/s ≈ 4,0 Mbit/s`.

Sur un lien de 50 Mbit/s : 8 % : confortable. Sur un lien de 10 Mbit/s : 40 % : **excessif pour une file prioritaire** (au-delà de 33 %, risque de famine des autres classes). Limitez la file prioritaire à 33 % du lien (recommandation classique) et utilisez le **contrôle d'admission d'appels** (CAC) pour limiter les appels simultanés.

### 5.3 Vidéo

| Qualité | Débit typique par flux |
|---|---|
| Visioconférence SD | 0,5 à 1 Mbit/s |
| HD (720p) | 1,5 à 2,5 Mbit/s |
| Full HD (1080p) | 3 à 4 Mbit/s |
| Salle immersive / 4K | 6 à 15 Mbit/s |

### 5.4 Répartition : exemple de lien de 100 Mbit/s

| Classe | Part | Mbit/s | Remarque |
|---|---|---|---|
| Voix (prioritaire) | 10 % | 10 | Borne du policer implicite |
| Contrôle | 5 % | 5 | |
| Vidéo | 20 % | 20 | |
| Critique | 25 % | 25 | |
| Par défaut | 38 % | 38 | |
| Arrière-plan | 2 % | 2 | |
| **Total** | **100 %** | **100** | |

> Les parts **garanties** valent sous congestion ; hors congestion, une classe peut consommer **le reste inutilisé** (bande passante restante).

---

## 6. Files d'attente et ordonnancement

### 6.1 Principaux mécanismes

| Mécanisme | Principe | Usage |
|---|---|---|
| **FIFO** | Premier arrivé, premier servi | Défaut sans QoS |
| **PQ** (priorité stricte) | La file haute est toujours servie d'abord | Risque de famine |
| **WFQ/CBWFQ** | Partage pondéré entre classes selon la bande passante garantie | Classes de données |
| **LLQ** (Low Latency Queuing) | CBWFQ + une file prioritaire **bornée** par un policer implicite | **Voix** |
| **WRR / DRR** | Tourniquet pondéré | Commutateurs |
| **FQ-CoDel / CAKE** | Équité par flux + contrôle actif du délai (Linux) | Contre le bufferbloat |

### 6.2 LLQ en pratique

```text
File prioritaire (voix)  -> servie en premier, mais plafonnée à X % (policer implicite) pour ne pas affamer
Files CBWFQ (vidéo, critique, défaut) -> partagent le reste selon leurs poids
```

### 6.3 Pourquoi borner la file prioritaire

Sans plafond, une source mal marquée EF (par exemple une sauvegarde) occuperait tout le lien. Le plafond limite les dégâts : le **trafic excédentaire de la classe EF est abandonné** en cas de congestion.

---

## 7. Policing et shaping

| | **Policing** (écrêtage) | **Shaping** (lissage) |
|---|---|---|
| Action sur l'excédent | **Abandonne** ou re-marque | **Met en file** (retarde) |
| Délai ajouté | Non | Oui (borné par la file) |
| Direction typique | Entrée et sortie | **Sortie** |
| Usage | Limiter un flux, protéger une file | **Adapter le débit sortant au contrat** |
| Effet sur TCP | Pertes brutales | Plus doux |

### 7.1 Pourquoi lisser à la vitesse du contrat

Un routeur relié à une fibre 1 Gbit/s mais abonné à 100 Mbit/s **émet à 1 Gbit/s** sur le support : c'est l'équipement de l'opérateur qui limite à 100 Mbit/s en **abandonnant** les paquets excédentaires **sans respecter votre QoS**. En **lissant à 100 Mbit/s** (un peu moins, 95 à 98 %), la file se forme **chez vous**, où la QoS décide qui passe.

```text
Sans shaping : routeur (1G) --> [opérateur : abandon aveugle à 100M] : la voix est perdue comme le reste
Avec shaping : routeur (shape 95M) : la QoS gère la file : la voix passe, les sauvegardes attendent
```

### 7.2 Paramètres de lissage

```text
Tc (intervalle de temps) = Bc / CIR
Bc (burst committed, bits) = CIR × Tc
Exemple : CIR = 100 Mbit/s, Tc = 10 ms -> Bc = 100 000 000 × 0,01 = 1 000 000 bits = 125 000 octets
```

Un Tc plus grand permet des rafales plus grosses (donc plus de gigue) ; un Tc plus petit lisse mieux mais augmente la charge de traitement. Pour la voix, la gigue induite par le lissage est environ **Tc** : gardez Tc petit (≤ 10 ms) sur les liens à voix.

### 7.3 Règle de réglage

| Contrat | Lissage recommandé |
|---|---|
| Débit garanti symétrique | 95 à 98 % du contrat |
| Débit « jusqu'à » (accès grand public) | Mesurer le débit réel et lisser à 85 à 90 % du débit stable |
| Lien 4G/5G (débit variable) | Pas de lissage fixe fiable ; QoS plus réactive (CAKE) ou classes très prudentes |

---

## 8. Gestion de la congestion : WRED et tail-drop

| Mécanisme | Principe | Remarque |
|---|---|---|
| **Tail-drop** | On abandonne quand la file est pleine | Provoque la **synchronisation globale** de TCP |
| **RED** | Abandon aléatoire croissant avant la saturation | Évite la synchronisation |
| **WRED** | RED par DSCP/précédence : abandon plus précoce pour les marquages de faible priorité | Pour les classes de données |
| **ECN** | Marquage au lieu d'abandon | Nécessite le support des extrémités |
| **CoDel / FQ-CoDel** | Surveille le délai de séjour en file, abandonne pour le maintenir bas | Contre le bufferbloat |

Règle : **pas de WRED sur la file voix** (UDP : l'abandon ne ralentit rien) ; WRED sur les classes TCP.

---

## 9. QoS hiérarchique sur IOS XE (MQC)

### 9.1 Les trois blocs du MQC

```text
class-map  : QUI (classification)
policy-map : QUOI (action : priorité, bande passante, shaping, marquage)
service-policy : OÙ (application sur une interface, entrée ou sortie)
```

### 9.2 Politique complète de sortie avec lissage

```text
! 1. Classification (par DSCP)
class-map match-any CM-VOIX
 match dscp ef
class-map match-any CM-CONTROLE
 match dscp cs6 cs3
class-map match-any CM-VIDEO
 match dscp af41 af42 af43
class-map match-any CM-CRITIQUE
 match dscp af31 af32 af33
class-map match-any CM-ARRIERE
 match dscp cs1
!
! 2. Politique enfant : files et bande passante
policy-map PM-ENFANT-WAN
 class CM-VOIX
  priority percent 15
 class CM-CONTROLE
  bandwidth remaining percent 5
 class CM-VIDEO
  bandwidth remaining percent 25
  random-detect dscp-based
 class CM-CRITIQUE
  bandwidth remaining percent 30
  random-detect dscp-based
 class CM-ARRIERE
  bandwidth remaining percent 5
 class class-default
  bandwidth remaining percent 35
  random-detect dscp-based
!
! 3. Politique parent : lissage au débit du contrat (100 Mbit/s -> 95 Mbit/s)
policy-map PM-PARENT-95M
 class class-default
  shape average 95000000
  service-policy PM-ENFANT-WAN
!
! 4. Application en sortie sur l'interface WAN
interface GigabitEthernet1
 service-policy output PM-PARENT-95M
```

Remarques :

| Élément | Explication |
|---|---|
| `priority percent 15` | File LLQ ; plafonnée à 15 % du débit de **référence** (celui du parent shapé : 95 Mbit/s) |
| `bandwidth remaining percent` | Part du **reste** après la file prioritaire ; plus robuste que `bandwidth percent` quand le débit varie |
| `random-detect dscp-based` | WRED selon DSCP |
| Parent/enfant | Le **parent** lisse, l'**enfant** ordonnance : indispensable pour que la QoS agisse à la vitesse du contrat |

### 9.3 Marquage à l'entrée (re-marquage)

```text
class-map match-any CM-MARQUE-VOIX
 match protocol rtp audio
policy-map PM-ENTREE-LAN
 class CM-MARQUE-VOIX
  set dscp ef
 class class-default
  set dscp default
interface GigabitEthernet3
 service-policy input PM-ENTREE-LAN
```

### 9.4 Vérifications

```text
show policy-map interface GigabitEthernet1 output
show policy-map interface GigabitEthernet1 output class CM-VOIX
show running-config | section policy-map
```

Lisez surtout : **offered rate** (débit présenté), **drop rate** (abandonné), **queue depth** (profondeur de file), **total drops** par classe.

---

## 10. QoS dans Catalyst SD-WAN

### 10.1 Deux niveaux

| Niveau | Mécanisme |
|---|---|
| **Politique localisée de QoS** (SW07) | Classes, files (jusqu'à 8), ordonnanceurs, `qos-map` appliqué sur l'interface de transport par le modèle |
| **MQC natif** via CLI add-on | Politiques hiérarchiques complètes avec lissage (section 9) |
| **Marquage via politique de données** | Définir le DSCP selon l'application ou l'adresse (SW07) |

### 10.2 Points propres au SD-WAN

| Point | Détail |
|---|---|
| **Encapsulation IPsec** | Le DSCP du paquet interne est généralement **copié** dans l'en-tête externe ; vérifier selon la version |
| **Lissage par interface de transport** | À configurer sur **chaque** interface, au débit contractuel de ce lien (débit aval/amont dans le modèle) |
| **BFD** | Marqué prioritaire par défaut (DSCP 48 pour les paquets de contrôle) |
| **Trafic de contrôle OMP** | Prioritaire |
| **Comportement au niveau tunnel** | Lissage possible par tunnel dans certaines versions ; consulter la documentation |
| **Sécurité et QoS** | La classification applicative (DPI) intervient avant la sortie ; **vérifier l'ordre des fonctions** |

### 10.3 Dimensionnement d'une interface de transport (exemple)

Site avec fibre 300/100 Mbit/s et 4G de secours :

| Interface | Débit amont contrat | Lissage sortie | Remarque |
|---|---|---|---|
| Fibre | 100 Mbit/s | 95 Mbit/s | Contrat symétrique : mesurer |
| 4G (dernier recours) | variable (≈ 20 Mbit/s) | 15 Mbit/s prudent ou CAKE | Seules les classes critiques autorisées |

---

## 11. QoS de bout en bout et sur Internet

### 11.1 Sur MPLS

L'opérateur propose en général 3 à 5 **classes de service** correspondant à des DSCP. Alignez votre plan de classes sur sa **table de correspondance** (contrat) : une voix en EF vers la classe « temps réel », vidéo/critique vers « données privilégiées ».

### 11.2 Sur Internet

| Fait | Conséquence |
|---|---|
| L'opérateur Internet **ne s'engage pas** sur le DSCP | La QoS agit **uniquement chez vous** (files de sortie) |
| Le DSCP peut être **effacé** (remis à 0) par un opérateur | N'en dépendez pas pour la voix entre sites distants sans mesures |
| La congestion se produit souvent **chez l'opérateur** ou sur l'interconnexion | Le SD-WAN contourne par le **choix de chemin** (AAR), pas par la QoS |
| Le lissage est vital | Pour que la file se forme chez vous |

### 11.3 Sens descendant (entrant)

Vous ne contrôlez pas ce que l'opérateur vous envoie : en cas de saturation **entrante**, la file se forme chez lui. Options : **lissage entrant** (ingress shaping avec CAKE/FQ-CoDel sur Linux), négociation de classes de service, ou **AAR** pour détourner la voix.

### 11.4 Conformité de bout en bout : check-list

| # | Contrôle |
|---|---|
| 1 | Marquage des téléphones et applications cohérent |
| 2 | Confiance au port d'accès configurée (seulement pour les appareils fiables) |
| 3 | Plan de classes identique aux deux extrémités |
| 4 | Lissage sur chaque interface WAN au débit du contrat |
| 5 | Correspondance DSCP / classes de service opérateur (MPLS) |
| 6 | Capture pour vérifier que le DSCP survit au transport |
| 7 | Test sous charge (voir lab) |

---

## 12. Vérification et dépannage

| Symptôme | Causes probables | Action |
|---|---|---|
| La voix hache quand un transfert démarre | Pas de lissage, file voix absente, DSCP perdu | `show policy-map interface`, capture DSCP |
| La classe voix abandonne | Plafond LLQ trop bas ou trafic EF excessif | Re-dimensionner, vérifier les sources EF |
| Aucune différence après QoS | Lien non saturé, politique non appliquée, mauvaise direction | Compteurs `offered/drop` |
| Les sauvegardes ralentissent tout | Pas de classe arrière-plan, marquage CS1 absent | Marquer, borner |
| Débit réel inférieur au contrat | Lissage trop bas, ou Tc | Mesurer, réajuster |
| DSCP réinitialisé | Équipement ou opérateur qui remet à 0 | Capture de chaque côté |
| Performance en chute avec QoS | CPU saturé (classification applicative lourde) | Classifier par DSCP/ACL plutôt que par DPI |

Capture pour vérifier le DSCP (Linux) :

```bash
sudo tcpdump -ni eth0 -v 'ip and udp' | grep -E "tos 0xb8"     # paquets EF
sudo tcpdump -ni eth0 -v 'ip[1] & 0xfc == 0xb8'                  # filtre BPF sur DSCP EF
```

---

## 13. Outils de calcul

```python
#!/usr/bin/env python3
"""qos_calc.py : calculs de dimensionnement pour la QoS WAN."""

OVERHEAD = {"eth": 18, "ipsec": 60}

CODECS = {                       # (charge utile en octets, paquets par seconde)
    "G.711": (160, 50),
    "G.729": (20, 50),
}

def debit_appel_kbps(codec: str, ethernet=True, ipsec=False) -> float:
    charge, pps = CODECS[codec]
    taille = charge + 12 + 8 + 20                    # RTP + UDP + IP
    if ipsec:
        taille += OVERHEAD["ipsec"]
    if ethernet:
        taille += OVERHEAD["eth"]
    return taille * pps * 8 / 1000

def file_voix_mbps(appels: int, codec="G.711", ipsec=True, marge=0.2) -> float:
    return appels * debit_appel_kbps(codec, True, ipsec) * (1 + marge) / 1000

def serialisation_ms(octets: int, debit_mbps: float) -> float:
    return octets * 8 / (debit_mbps * 1e6) * 1000

def delai_file_ms(paquets: int, taille_octets: int, debit_mbps: float) -> float:
    return paquets * taille_octets * 8 / (debit_mbps * 1e6) * 1000

def burst_bits(cir_mbps: float, tc_ms: float) -> float:
    return cir_mbps * 1e6 * tc_ms / 1000

if __name__ == "__main__":
    print("G.711 non chiffré :", round(debit_appel_kbps("G.711", True, False), 1), "kbit/s")
    print("G.711 IPsec       :", round(debit_appel_kbps("G.711", True, True), 1), "kbit/s")
    print("G.729 IPsec       :", round(debit_appel_kbps("G.729", True, True), 1), "kbit/s")
    print("30 appels G.711 IPsec + 20 % :", round(file_voix_mbps(30), 2), "Mbit/s")
    print("Sérialisation 1500 o à 1 Mbit/s :", round(serialisation_ms(1500, 1), 1), "ms")
    print("File de 100 paquets de 1500 o à 10 Mbit/s :", round(delai_file_ms(100, 1500, 10), 1), "ms")
    b = burst_bits(100, 10)
    print(f"Bc pour 100 Mbit/s et Tc 10 ms : {b:.0f} bits = {b/8:.0f} octets")
```

Résultats attendus : G.711 non chiffré 87,2 ; G.711 IPsec 111,2 ; G.729 IPsec 55,2 kbit/s ; 30 appels ≈ 4,0 Mbit/s ; sérialisation 12,0 ms ; file 120,0 ms ; Bc 1 000 000 bits = 125 000 octets.

---

## 14. Lab guidé CP02

### 14.1 Objectif

Reproduire sous Linux un lien WAN de 20 Mbit/s congestionné, puis démontrer que **sans QoS la voix se dégrade** et **avec QoS (priorité + lissage) elle est protégée**.

### 14.2 Topologie

```text
[ client voix + client données ] --- [ routeur Linux (QoS) eth1 ] === lien 20 Mbit/s === [ serveur (iperf3) ]
```

Réalisation possible avec deux espaces de noms réseau (`ip netns`) ou trois VM.

### 14.3 Étape 1 : limiter le lien et mesurer SANS QoS

```bash
# Sur le routeur, interface de sortie eth1 : lien simulé à 20 Mbit/s avec une grosse file (bufferbloat)
sudo tc qdisc replace dev eth1 root tbf rate 20mbit burst 32kbit latency 400ms
tc -s qdisc show dev eth1

# Sur le serveur
iperf3 -s &
# Client : saturer avec 4 flux TCP (données) pendant que la « voix » (UDP 100 kbit/s) est mesurée
iperf3 -c 10.0.0.2 -t 40 -P 4 &
iperf3 -c 10.0.0.2 -u -b 100k -l 160 -t 30 --tos 184 -p 5202 --get-server-output    # nécessite un 2e serveur sur 5202
```

Notez : gigue et perte du flux voix pendant la saturation (typiquement : délai de plusieurs centaines de ms et pertes).

### 14.4 Étape 2 : activer la QoS (HTB + FQ-CoDel + filtres DSCP)

```bash
#!/usr/bin/env bash
# qos-wan.sh : QoS à 4 classes sur eth1, lissage à 19 Mbit/s (95 % du contrat de 20)
DEV=eth1
RATE=19mbit
sudo tc qdisc del dev $DEV root 2>/dev/null

sudo tc qdisc add dev $DEV root handle 1: htb default 40
sudo tc class add dev $DEV parent 1:  classid 1:1  htb rate $RATE ceil $RATE
# 1:10 voix (prioritaire, 15 %), 1:20 vidéo, 1:30 critique, 1:40 défaut/arrière-plan
sudo tc class add dev $DEV parent 1:1 classid 1:10 htb rate 3mbit   ceil $RATE prio 0
sudo tc class add dev $DEV parent 1:1 classid 1:20 htb rate 5mbit   ceil $RATE prio 1
sudo tc class add dev $DEV parent 1:1 classid 1:30 htb rate 6mbit   ceil $RATE prio 2
sudo tc class add dev $DEV parent 1:1 classid 1:40 htb rate 5mbit   ceil $RATE prio 3

for c in 10 20 30 40; do
  sudo tc qdisc add dev $DEV parent 1:$c handle ${c}: fq_codel
done

# Filtres par DSCP : EF (0xb8), AF41 (0x88), AF31 (0x68) ; masque 0xfc sur l'octet ToS
sudo tc filter add dev $DEV parent 1: protocol ip prio 1 u32 match ip dsfield 0xb8 0xfc flowid 1:10
sudo tc filter add dev $DEV parent 1: protocol ip prio 2 u32 match ip dsfield 0x88 0xfc flowid 1:20
sudo tc filter add dev $DEV parent 1: protocol ip prio 3 u32 match ip dsfield 0x68 0xfc flowid 1:30

tc -s class show dev $DEV
```

Relancez les mêmes tests (saturation + voix EF) et comparez.

### 14.5 Étape 3 : analyser

| Mesure | Sans QoS | Avec QoS |
|---|---|---|
| Perte voix (%) | | |
| Gigue voix (ms) | | |
| Délai voix (ms, `ping -Q 0xb8`) | | |
| Débit TCP de saturation (Mbit/s) | | |

Questions : pourquoi le lissage à 19 Mbit/s plutôt que 20 ? Que se passe-t-il si l'on met un marquage EF sur le transfert de données ? Que se passe-t-il si la classe EF dépasse son débit garanti ?

### 14.6 Étape 4 : vérifier le DSCP bout en bout

```bash
sudo tcpdump -ni eth1 -c 20 'ip[1] & 0xfc == 0xb8'
ping -Q 0xb8 -c 20 10.0.0.2
```

### 14.7 Livrables et grille

| Élément | Points |
|---|---|
| Lien congestionné reproduit, mesures sans QoS | 4 |
| QoS configurée (classes, filtres, lissage) | 5 |
| Mesures avec QoS et comparaison | 4 |
| Analyse (lissage, risques d'un mauvais marquage) | 4 |
| Équivalent MQC rédigé pour IOS XE | 3 |

---

## 15. Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **QoS** | Qualité de service : traitement différencié du trafic |
| **DiffServ / DSCP** | Modèle de classes de service / marquage de 6 bits |
| **PHB** | Comportement par saut associé à une classe |
| **EF / AF / CS** | Expedited Forwarding / Assured Forwarding / Class Selector |
| **LLQ / CBWFQ** | File prioritaire bornée / files pondérées par classe |
| **Policing / shaping** | Écrêtage / lissage |
| **CIR / Bc / Tc** | Débit garanti / rafale autorisée / intervalle |
| **WRED / ECN** | Abandon aléatoire pondéré / notification explicite de congestion |
| **Bufferbloat** | Latence excessive due à des files trop grandes |
| **CoDel / CAKE** | Gestions actives de files contre le bufferbloat |
| **MQC** | Modular QoS CLI de Cisco |
| **Scavenger** | Classe sacrifiable (arrière-plan) |
| **CAC** | Contrôle d'admission d'appels |
| **MOS** | Note de qualité vocale de 1 à 5 |
| **Offered rate / drop rate** | Débit présenté / débit abandonné |

### B. Cas d'usage détaillés

#### Cas 1. « On a de la QoS mais la voix hache toujours »

Lien 100 Mbit/s en fibre grand public, routeur sur interface 1 Gbit/s, QoS appliquée **sans lissage**. Les files du routeur ne se forment jamais (le port est à 1 Gbit/s) : la congestion se produit chez l'opérateur. Après ajout d'un **parent shaping à 90 Mbit/s**, la voix redevient propre. **Leçon** : **la QoS sans lissage ne sert à rien sur un lien de débit inférieur à l'interface**.

#### Cas 2. L'ERP écrasé par un téléchargement de mise à jour

Chaque matin, un parc de postes télécharge des mises à jour et l'ERP ralentit. Solution : classe d'arrière-plan (CS1) pour les mises à jour (par GPO ou par proxy) et classe critique AF31 pour l'ERP. **Leçon** : **marquez à la source** ce qui doit être sacrifié.

#### Cas 3. Le DSCP effacé par l'opérateur

Entre deux sites reliés par Internet, la voix arrive en DSCP 0. Un opérateur intermédiaire remet à zéro. La QoS n'agit alors que **en sortie locale**. Solution : AAR avec classe VOICE et chemin préféré MPLS pour la voix, QoS locale seulement. **Leçon** : sur Internet, **choix de chemin** plutôt que dépendance au marquage de bout en bout.

#### Cas 4. File prioritaire trop large

Une file prioritaire à 40 % sur un lien de 10 Mbit/s : un équipement de visioconférence mal marqué en EF occupe 4 Mbit/s, et les applications critiques sont étranglées. Correction : EF limité à 15 %, vidéo en AF41 avec WRED. **Leçon** : **un EF mal contrôlé est un risque** ; auditez les sources de DSCP 46.

#### Cas 5. Satellite : QoS et TCP

Sur un lien satellite à 600 ms, la QoS d'ordonnancement fonctionne, mais le débit TCP reste limité par le produit débit × délai. Solution : optimisation TCP (SW03), fenêtre élargie. **Leçon** : QoS et optimisation sont **complémentaires**.

#### Cas 6. 4G de secours qui s'effondre

Sur 4G, le débit varie de 5 à 40 Mbit/s : un lissage fixe à 20 Mbit/s est faux. Solution : classes très restreintes en secours (voix et applications critiques), CAKE avec ajustement automatique, mesures régulières. **Leçon** : sur débit variable, **moins de classes** et **réactivité**.

### C. Retours d'expérience

1. **Mesurez avant de configurer.** Sans congestion mesurée, la QoS est un placebo.
2. **Lissez toujours les liens WAN** dont le débit contractuel est inférieur à l'interface.
3. **Auditez qui marque EF.** Une seule source mal configurée peut annuler la QoS.
4. **Un plan de classes simple est respecté**, un plan complexe est contourné.
5. **Testez sous charge** (iperf3 + voix réelle) avant la recette.
6. **Ne comptez pas sur la QoS pour compenser un lien trop petit.**
7. **Vérifiez le marquage de bout en bout** avec une capture, jamais sur la foi de la configuration.
8. **Surveillez les abandons par classe** : des abandons dans la classe voix sont un signal d'alarme.

### D. Exercices supplémentaires

1. Dimensionnez la file voix d'un site de 80 appels simultanés en G.711 chiffré avec 20 % de marge sur un lien de 100 Mbit/s. Est-ce raisonnable ?
2. Calculez Bc pour CIR = 50 Mbit/s et Tc = 8 ms.
3. Combien de temps une file de 200 paquets de 1 200 octets met-elle à se vider sur 20 Mbit/s ?
4. Écrivez en MQC la politique d'entrée qui re-marque en CS1 tout le trafic vers `10.9.0.0/16` (serveur de sauvegarde) et en EF le trafic RTP.
5. Quel est l'octet ToS d'un paquet marqué AF31 ? Quel filtre `tc u32` l'identifie ?

<details>
<summary>Corrigés</summary>

1. `80 × 111,2 kbit/s × 1,2 = 10 675 kbit/s ≈ 10,7 Mbit/s` = 10,7 % du lien : raisonnable (< 33 %).
2. `50 000 000 × 0,008 = 400 000 bits = 50 000 octets`.
3. `200 × 1 200 × 8 / 20 000 000 = 0,096 s = 96 ms`.
4. `ip access-list extended ACL-BACKUP` / `permit ip any 10.9.0.0 0.0.255.255` ; `class-map match-all CM-BACKUP` / `match access-group name ACL-BACKUP` ; `policy-map PM-IN` / `class CM-BACKUP` / `set dscp cs1` ; `class CM-RTP` (`match protocol rtp audio`) / `set dscp ef`.
5. AF31 = DSCP 26, ToS = 104 = 0x68 ; filtre : `u32 match ip dsfield 0x68 0xfc`.

</details>

### E. Mini-projet (8 heures)

Concevez le **plan de QoS complet** d'une entreprise de 40 sites : plan de classes (DSCP), dimensionnement par type de site (calculs), politiques IOS XE (parent/enfant) pour 3 débits (20, 100, 500 Mbit/s), plan de marquage (téléphones, applications, postes), règles pour les liens 4G, plan de test sous charge, tableau de contrôle d'abandons. Validez le principe sous Linux comme dans le lab.

---

## 16. Quiz

1. La QoS crée-t-elle de la bande passante ?
2. Pourquoi lisser à 95 % du contrat ?
3. Quelle est la valeur de DSCP de la voix et son octet ToS ?
4. Pourquoi borner la file prioritaire ?
5. Pourquoi pas de WRED sur la voix ?
6. Que devient le DSCP sur Internet ?
7. Quelle est la différence entre policing et shaping ?
8. Quel est le débit approximatif d'un appel G.711 chiffré en IPsec ?

<details>
<summary>Réponses</summary>

1. Non, elle arbitre l'usage d'un débit limité. 2. Pour que la file se forme chez soi et non chez l'opérateur. 3. EF, DSCP 46, ToS 184 (0xB8). 4. Pour éviter qu'un trafic mal marqué n'affame les autres classes. 5. La voix est en UDP : l'abandon ne ralentit pas la source. 6. Il n'est pas garanti ; il peut être effacé ; la QoS agit surtout localement. 7. Le policing abandonne l'excédent ; le shaping le retarde en file. 8. Environ 100 à 111 kbit/s.

</details>

---

## 17. Aide-mémoire

```text
DSCP : EF 46 (ToS 184, 0xB8) voix | AF41 34 vidéo | AF31 26 critique | CS6 48 contrôle | CS3 24 signalisation | CS1 8 arrière-plan | 0 défaut
ToS = DSCP x 4 ; G.711 = 80 kbit/s IP (87 Eth, ~111 IPsec) ; 1500 o à 1 Mbit/s = 12 ms
LLQ borné (<= 33 %, typique 10-20 %) + CBWFQ + WRED (sauf voix) ; parent shape 95 % du contrat + enfant ordonnancement
Bc = CIR x Tc ; délai de file = paquets x taille x 8 / débit ; bufferbloat = files trop grandes
Internet : DSCP non garanti -> QoS locale + AAR ; MPLS : aligner sur les classes de l'opérateur
Vérifier : show policy-map interface ... output ; tcpdump 'ip[1] & 0xfc == 0xb8'
```

## 18. Pour aller plus loin

- RFC 2474, 2475, 4594 (DiffServ et recommandations de configuration) : <https://www.rfc-editor.org>
- Cisco, Enterprise QoS Solution Reference Network Design.
- Linux, `man tc`, `tc-htb`, `tc-fq_codel`, `tc-cake` ; Bufferbloat.net : <https://www.bufferbloat.net>

> Module précédent : [CP01](../CP01-routage-avance/CP01-routage-avance.md) | Module suivant : [CP03](../CP03-ipsec-en-profondeur/CP03-ipsec-en-profondeur.md)
