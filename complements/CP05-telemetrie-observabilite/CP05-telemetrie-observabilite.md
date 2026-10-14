# CP05. Télémétrie et observabilité du WAN

| | |
|---|---|
| **Partie** | Compléments (hors parcours principal) |
| **Durée indicative** | 1,5 semaine (12 h) |
| **Prérequis** | SW10, SW15, SW18 ; Linux, Docker, notions de séries temporelles |
| **Niveau** | Avancé |
| **Lien examen** | DevNet, ENAUTO ; culture SRE |
| **Lab associé** | Lab CP05 (pile Prometheus, Grafana, Blackbox et SNMP pour un WAN de laboratoire) |

> Module précédent : [CP04](../CP04-flexiwan/CP04-flexiwan.md) | Module suivant : [CP06](../CP06-preparation-certifications/CP06-preparation-certifications.md)

> **Avertissement.** Versions d'images Docker, noms de métriques et OID varient. Les exemples sont des **modèles de départ** ; épinglez des versions éprouvées et validez sur laboratoire. Les identifiants (mots de passe, communautés, clés) sont des **marqueurs à remplacer** et ne doivent jamais être publiés.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Superviser, mesurer, observer](#2-superviser-mesurer-observer)
3. [Les sources de télémétrie d'un WAN](#3-les-sources-de-télémétrie-dun-wan)
4. [SNMP v3](#4-snmp-v3)
5. [Syslog structuré](#5-syslog-structuré)
6. [NetFlow et IPFIX](#6-netflow-et-ipfix)
7. [Télémétrie en continu (streaming)](#7-télémétrie-en-continu-streaming)
8. [Sondes actives : mesurer ce que vit l'utilisateur](#8-sondes-actives--mesurer-ce-que-vit-lutilisateur)
9. [La pile Prometheus, Grafana et Alertmanager](#9-la-pile-prometheus-grafana-et-alertmanager)
10. [Tableaux de bord utiles](#10-tableaux-de-bord-utiles)
11. [Alertes : peu mais utiles](#11-alertes--peu-mais-utiles)
12. [Corrélation et analyse de cause](#12-corrélation-et-analyse-de-cause)
13. [Capacité, rétention et coûts](#13-capacité-rétention-et-coûts)
14. [Lab guidé CP05](#14-lab-guidé-cp05)
15. [Approfondissements](#15-approfondissements)
16. [Quiz](#16-quiz)
17. [Aide-mémoire](#17-aide-mémoire)
18. [Pour aller plus loin](#18-pour-aller-plus-loin)

---

## 1. Objectifs pédagogiques

- Distinguer supervision, métrologie et observabilité.
- Choisir les sources de télémétrie adaptées (SNMP, syslog, flux, API, sondes).
- Déployer une pile Prometheus/Grafana/Alertmanager pour un WAN.
- Construire des tableaux de bord et des alertes exploitables.
- Corréler métriques, journaux et flux pour établir une cause.
- Dimensionner rétention et capacité de la pile.

---

## 2. Superviser, mesurer, observer

| Notion | Question | Exemple |
|---|---|---|
| **Supervision** (monitoring) | « Est-ce que ça marche ? » (états connus) | Le tunnel est up, la CPU est à 40 % |
| **Métrologie** | « Quelle quantité, quelle qualité ? » | Perte moyenne de 0,3 %, P95 d'utilisation 62 % |
| **Observabilité** | « Pourquoi ça ne marche pas ? » (états inconnus) | Corréler une hausse de gigue avec une alarme opérateur et un flux anormal |

Les **trois piliers** : **métriques** (valeurs numériques dans le temps), **journaux** (événements), **traces/flux** (chemins des requêtes ou des paquets).

Principe : **mesurer à plusieurs endroits** (équipement, transport, application) pour **localiser** un problème par comparaison.

---

## 3. Les sources de télémétrie d'un WAN

| Source | Données | Fréquence | Forces | Limites |
|---|---|---|---|---|
| **API du Manager** | États de fabrique, BFD, AAR, alarmes | 30 à 60 s | Vue SD-WAN native | Charge sur le Manager |
| **SNMP** (v3) | Interfaces, CPU, mémoire, températures | 30 à 60 s | Universel | Ancien, peu de métriques SD-WAN |
| **Syslog** | Événements | Temps réel | Détail des événements | Non structuré |
| **NetFlow/IPFIX** | Flux (qui parle à qui, combien) | Export 1 à 5 min | Analyse de trafic et de sécurité | Volume, échantillonnage |
| **Télémétrie en continu** (gNMI, etc.) | Compteurs fréquents | 1 à 10 s | Précis, efficace | Dépend de la plate-forme |
| **Sondes actives** (ICMP, HTTP, DNS) | Expérience de bout en bout | 10 à 60 s | Voit ce que vit l'utilisateur | Mesure un chemin à la fois |
| **BFD / mesures du SD-WAN** | Perte, délai, gigue par tunnel | 1 s (BFD) / agrégées | Natif et fin | Accès via API/CLI |

---

## 4. SNMP v3

### 4.1 Pourquoi v3

SNMP v1/v2c transmet la **communauté en clair** et n'authentifie pas ; v3 apporte **authentification et chiffrement**.

| Niveau | Authentification | Chiffrement |
|---|---|---|
| noAuthNoPriv | Non | Non |
| authNoPriv | Oui | Non |
| **authPriv** | Oui | **Oui** (à utiliser) |

### 4.2 Configuration IOS XE

```text
snmp-server view VUE-LECTURE iso included
snmp-server group GRP-SUPERVISION v3 priv read VUE-LECTURE
snmp-server user supervision GRP-SUPERVISION v3 auth sha <MOT-DE-PASSE-AUTH-LONG> priv aes 128 <MOT-DE-PASSE-PRIV-LONG>
snmp-server host 10.250.0.30 version 3 priv supervision
snmp-server enable traps
snmp-server location "Site LYO-01"
snmp-server contact "noc@exemple.fr"
```

Restreindre l'accès SNMP au serveur de supervision (ACL) et utiliser le réseau de gestion.

### 4.3 Test depuis un serveur

```bash
snmpwalk -v3 -l authPriv -u supervision -a SHA -A '<AUTH>' -x AES -X '<PRIV>' 10.255.1.100 1.3.6.1.2.1.2.2.1.8   # ifOperStatus
snmpget  -v3 -l authPriv -u supervision -a SHA -A '<AUTH>' -x AES -X '<PRIV>' 10.255.1.100 1.3.6.1.2.1.1.3.0     # sysUpTime
```

### 4.4 OID utiles

| Information | OID / table |
|---|---|
| Nom, uptime | `1.3.6.1.2.1.1.5.0`, `1.3.6.1.2.1.1.3.0` |
| État des interfaces | `ifOperStatus` (`1.3.6.1.2.1.2.2.1.8`) |
| Octets entrants/sortants (64 bits) | `ifHCInOctets`, `ifHCOutOctets` (`1.3.6.1.2.1.31.1.1.1.6` et `.10`) |
| Erreurs et rejets | `ifInErrors`, `ifOutDiscards` |
| CPU et mémoire Cisco | MIB `CISCO-PROCESS-MIB`, `CISCO-ENHANCED-MEMPOOL-MIB` |

---

## 5. Syslog structuré

### 5.1 Bonnes pratiques

| Règle | Détail |
|---|---|
| **Format** | RFC 5424 si possible ; horodatage ISO 8601 en UTC |
| **Transport** | TCP/TLS pour ne pas perdre de messages |
| **Niveau** | `informational` en production, `debugging` ponctuel |
| **Source** | Interface de gestion stable (loopback) |
| **Collecte** | Collecteur (rsyslog, Vector, Fluent Bit, Logstash, Loki) |

### 5.2 Exemple rsyslog : réception et stockage par équipement

```text
# /etc/rsyslog.d/10-reseau.conf
module(load="imudp")
input(type="imudp" port="514")
template(name="PerHost" type="string" string="/var/log/reseau/%HOSTNAME%/%$YEAR%-%$MONTH%-%$DAY%.log")
if $fromhost-ip startswith '10.255.' then {
    action(type="omfile" dynaFile="PerHost")
    stop
}
```

### 5.3 Transformer en métriques

On peut **compter** les événements (échecs de connexion, flaps de tunnel) et les exposer comme métriques : outils `grok_exporter`, `mtail`, ou Loki avec règles d'enregistrement. L'intérêt : **alerter sur un taux** d'événements plutôt que sur un message isolé.

---

## 6. NetFlow et IPFIX

### 6.1 Principe

Le routeur résume chaque **flux** (adresses, ports, protocole, octets, paquets) et l'exporte vers un **collecteur**. Utilisations : **qui consomme la bande passante**, détection de comportements anormaux, preuve d'usage, planification de capacité.

### 6.2 Configuration IOS XE (Flexible NetFlow)

```text
flow record FR-WAN
 match ipv4 source address
 match ipv4 destination address
 match ipv4 protocol
 match transport source-port
 match transport destination-port
 match ipv4 dscp
 collect interface input
 collect counter bytes long
 collect counter packets long
 collect timestamp absolute first
 collect timestamp absolute last
!
flow exporter FE-COLLECTEUR
 destination 10.250.0.30
 source Loopback0
 transport udp 2055
 export-protocol ipfix
 template data timeout 60
!
flow monitor FM-WAN
 record FR-WAN
 exporter FE-COLLECTEUR
 cache timeout active 60
!
interface GigabitEthernet1
 ip flow monitor FM-WAN input
 ip flow monitor FM-WAN output
```

> En Catalyst SD-WAN, l'export de flux s'active par une **politique de données (cflowd)** et les paramètres du modèle ; la configuration CLI ci-dessus illustre le principe sur IOS XE autonome.

### 6.3 Collecteurs

| Outil | Remarque |
|---|---|
| **nfdump / nfsen** | Léger, ligne de commande |
| **pmacct** | Agrégation et export vers bases |
| **ElastiFlow, Kentik, Plixer** | Analyse riche |
| **Akvorado** | Collecte et visualisation de flux (open source) |

Exemple nfdump :

```bash
nfcapd -w -D -p 2055 -l /var/cache/nfdump
nfdump -R /var/cache/nfdump -s srcip/bytes -n 10        # top 10 des sources
nfdump -R /var/cache/nfdump -s dstport/bytes -n 10      # top 10 des ports de destination
```

---

## 7. Télémétrie en continu (streaming)

| Technologie | Principe |
|---|---|
| **Model-driven telemetry (gNMI, NETCONF dial-out)** | L'équipement **pousse** des données structurées (modèles YANG) à intervalle court |
| **Avantages** | Faible latence, données précises, moins de charge qu'une interrogation SNMP |
| **Collecteurs** | Telegraf (plugin gNMI), gnmic, Cisco Telegraf/Pipeline |
| **Prérequis** | Support par la plate-forme et la version ; certificats |

Exemple de collecte avec **gnmic** (outil open source) :

```bash
gnmic -a 10.255.1.100:9339 -u supervision -p '<MOT-DE-PASSE>' --skip-verify \
      subscribe --path "/interfaces/interface/state/counters" --stream-mode sample --sample-interval 10s
```

> Disponibilité, chemins YANG et port dépendent de la plate-forme : consultez la documentation de la version.

---

## 8. Sondes actives : mesurer ce que vit l'utilisateur

Les métriques d'équipement disent **l'état** ; les sondes disent **l'expérience**. Utilisez le **Blackbox Exporter** de Prometheus pour mesurer depuis plusieurs sites.

| Sonde | Mesure |
|---|---|
| **ICMP** | Disponibilité, délai, perte |
| **TCP** | Ouverture de port (service joignable) |
| **HTTP(S)** | Code de réponse, durée par phase (DNS, connexion, TLS, premier octet) |
| **DNS** | Réponse et durée de résolution |

Où placer les sondes : un petit hôte (mini-PC ou conteneur) dans **chaque type de site** ; sondes vers : **le DC, les applications SaaS, Internet, la passerelle de sécurité**.

---

## 9. La pile Prometheus, Grafana et Alertmanager

### 9.1 Architecture

```text
Équipements (SNMP) --> snmp_exporter  --+
Manager (API)       --> sdwan_exporter -+--> Prometheus --> Grafana (tableaux de bord)
Sondes              --> blackbox       -+          |
                                                    +--> Alertmanager --> e-mail, chat, astreinte
Syslog --> Loki/Promtail (journaux) ----------------------------> Grafana
```

### 9.2 `docker-compose.yml` de laboratoire

```yaml
services:
  prometheus:
    image: prom/prometheus:latest        # en production : épingler une version
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alerts.yml:/etc/prometheus/alerts.yml:ro
      - prom-data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=90d
    ports: ["9090:9090"]
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    volumes: ["./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro"]
    ports: ["9093:9093"]
    restart: unless-stopped

  blackbox:
    image: prom/blackbox-exporter:latest
    volumes: ["./blackbox.yml:/etc/blackbox_exporter/config.yml:ro"]
    ports: ["9115:9115"]
    restart: unless-stopped

  snmp-exporter:
    image: prom/snmp-exporter:latest
    volumes: ["./snmp.yml:/etc/snmp_exporter/snmp.yml:ro"]
    ports: ["9116:9116"]
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/grafana_admin
    secrets: [grafana_admin]
    volumes: [grafana-data:/var/lib/grafana]
    ports: ["3000:3000"]
    restart: unless-stopped

volumes: {prom-data: {}, grafana-data: {}}
secrets:
  grafana_admin:
    file: ./secrets/grafana_admin.txt
```

### 9.3 `prometheus.yml`

```yaml
global:
  scrape_interval: 30s
  evaluation_interval: 30s

rule_files: ["/etc/prometheus/alerts.yml"]
alerting:
  alertmanagers:
    - static_configs: [{targets: ["alertmanager:9093"]}]

scrape_configs:
  - job_name: prometheus
    static_configs: [{targets: ["localhost:9090"]}]

  - job_name: sdwan
    scrape_interval: 60s
    static_configs: [{targets: ["host.docker.internal:9877"]}]   # exporter du module SW18

  - job_name: snmp
    metrics_path: /snmp
    params: {module: [if_mib], auth: [auth_v3]}
    static_configs:
      - targets: ["10.255.1.100", "10.255.1.101"]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116

  - job_name: blackbox-icmp
    metrics_path: /probe
    params: {module: [icmp]}
    static_configs:
      - targets: ["10.1.1.10", "10.1.5.20", "1.1.1.1"]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115

  - job_name: blackbox-http
    metrics_path: /probe
    params: {module: [http_2xx]}
    static_configs:
      - targets: ["https://outlook.office.com", "https://erp.exemple.fr"]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115
```

### 9.4 `blackbox.yml`

```yaml
modules:
  icmp:
    prober: icmp
    timeout: 5s
  http_2xx:
    prober: http
    timeout: 10s
    http:
      valid_status_codes: []          # 2xx par défaut
      method: GET
      preferred_ip_protocol: ip4
```

### 9.5 Remarque sur la configuration SNMP

Le fichier `snmp.yml` se **génère** avec l'outil `generator` du projet `snmp_exporter` à partir des MIB (modules `if_mib`, etc.) et d'une section `auth` pour SNMP v3 (utilisateur, niveau, mots de passe). Consultez la documentation du projet : le fichier généré est volumineux et ne doit pas être écrit à la main.

---

## 10. Tableaux de bord utiles

### 10.1 Cinq tableaux

| Tableau | Question | Contenu |
|---|---|---|
| **1. Santé globale** | Le WAN est-il en bonne santé ? | Sites up/down, tunnels up, disponibilité 30 j, alertes actives |
| **2. Vue par site** | Que se passe-t-il à LYO-01 ? | Tunnels, perte/délai/gigue par transport, utilisation, CPU, bascules |
| **3. Transports et opérateurs** | Quel opérateur dégrade ? | Disponibilité et qualité par transport/opérateur, classement |
| **4. Applications et expérience** | Comment vont les applications ? | Sondes HTTP (DNS, connexion, TLS, total), Cloud OnRamp |
| **5. Capacité** | Où faut-il investir ? | P95 par lien, saturation, prévision |

### 10.2 Requêtes PromQL utiles

```text
# Disponibilité d'un hôte par ping (sondes) sur 30 jours
100 * avg_over_time(probe_success{job="blackbox-icmp", instance="10.1.1.10"}[30d])

# Délai de réponse HTTP total (ms) et par phase
1000 * probe_http_duration_seconds{phase="connect"}

# Débit entrant d'une interface (bit/s) à partir des compteurs 64 bits SNMP
8 * rate(ifHCInOctets{instance="10.255.1.100", ifName="GigabitEthernet1"}[5m])

# Utilisation en % d'un lien de 100 Mbit/s
100 * 8 * rate(ifHCOutOctets{ifName="GigabitEthernet1"}[5m]) / (100 * 1000 * 1000)

# P95 d'utilisation sur 30 jours (sous-requête)
quantile_over_time(0.95, (8 * rate(ifHCOutOctets{ifName="GigabitEthernet1"}[5m]) / 1e8 * 100)[30d:5m])

# Nombre d'erreurs d'interface par minute
rate(ifInErrors[5m]) * 60
```

### 10.3 Principes de lisibilité

1. **Une question par tableau**, titre explicite.
2. **Objectifs visibles** (lignes de SLO).
3. **Unités claires** (ms, %, bit/s).
4. **Variables Grafana** (site, transport) pour naviguer.
5. **Pas plus de 8 à 12 panneaux** par tableau.
6. **Liens** (de la vue globale vers la vue par site).

---

## 11. Alertes : peu mais utiles

### 11.1 Principes

| Principe | Application |
|---|---|
| **Alerter sur les symptômes**, pas sur toutes les causes | « Site injoignable » plutôt que « CPU 82 % » |
| **Durée minimale** (`for:`) | Éviter les alertes sur micro-événements |
| **Gravité** | critique (astreinte), majeure (ticket), mineure (revue) |
| **Chaque alerte appelle une action** | Sinon, la supprimer |
| **Documentation** (annotation `runbook`) | Lien vers la procédure |
| **Inhibition** | Une alerte de site coupé inhibe les alertes de ses tunnels |

### 11.2 `alerts.yml`

```yaml
groups:
  - name: wan
    rules:
      - alert: SiteInjoignable
        expr: probe_success{job="blackbox-icmp"} == 0
        for: 3m
        labels: {severity: critical}
        annotations:
          summary: "{{ $labels.instance }} injoignable depuis 3 minutes"
          runbook: "https://wiki.exemple.fr/runbooks/site-injoignable"

      - alert: LienSature
        expr: 100 * 8 * rate(ifHCOutOctets{ifName="GigabitEthernet1"}[5m]) / 1e8 > 90
        for: 15m
        labels: {severity: warning}
        annotations: {summary: "Lien {{ $labels.instance }} saturé (> 90 % depuis 15 min)"}

      - alert: ErreursInterface
        expr: rate(ifInErrors[10m]) * 60 > 10
        for: 10m
        labels: {severity: warning}
        annotations: {summary: "Erreurs sur {{ $labels.instance }} {{ $labels.ifName }}"}

      - alert: ApplicationLente
        expr: probe_http_duration_seconds{job="blackbox-http"} > 3
        for: 10m
        labels: {severity: warning}
        annotations: {summary: "Temps de réponse élevé : {{ $labels.instance }}"}

      - alert: CertificatBientotExpire
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
        for: 1h
        labels: {severity: warning}
        annotations: {summary: "Certificat de {{ $labels.instance }} expire dans moins de 30 jours"}
```

### 11.3 `alertmanager.yml` (extrait)

```yaml
route:
  receiver: noc-mail
  group_by: [alertname, instance]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers: [severity="critical"]
      receiver: astreinte
receivers:
  - name: noc-mail
    email_configs:
      - to: noc@exemple.fr
        from: supervision@exemple.fr
        smarthost: smtp.exemple.fr:587
  - name: astreinte
    webhook_configs:
      - url: https://astreinte.exemple.fr/hooks/wan
inhibit_rules:
  - source_matchers: [alertname="SiteInjoignable"]
    target_matchers: [alertname=~"ErreursInterface|LienSature"]
    equal: [instance]
```

---

## 12. Corrélation et analyse de cause

### 12.1 Démarche

```text
1 Symptôme (alerte ou plainte)  ->  2 Fenêtre de temps  ->  3 Métriques : où est l'anomalie ? (site, transport, application)
4 Journaux : événements sur la même fenêtre ?  ->  5 Flux : trafic inhabituel ?  ->  6 Changement : modification récente ?
7 Hypothèse  ->  8 Test  ->  9 Cause racine et action
```

### 12.2 Exemple de corrélation

| Heure | Signal | Source |
|---|---|---|
| 17:02 | Gigue + 25 ms sur SITE-03 transport internet | Métriques de tunnel |
| 17:03 | Utilisation du lien à 96 % | SNMP |
| 17:03 | Flux : 80 % du trafic vers `10.9.0.50:873` (sauvegarde) | NetFlow |
| 17:05 | Plaintes « Teams hache » | Ticket |
| 16:55 | Planification de sauvegarde modifiée la veille | Changement |

**Conclusion** : la sauvegarde déplacée à 17 h sature le lien. Action : retour à 22 h et classe d'arrière-plan.

### 12.3 Avantage de l'approche multi-sources

Chaque source seule est ambiguë ; croisées, elles **prouvent** la cause et donnent un argument incontestable (opérateur, métiers, direction).

---

## 13. Capacité, rétention et coûts

### 13.1 Estimation du stockage Prometheus

```text
Octets ≈ séries × (durée / intervalle) × 1,5 à 2 octets par échantillon
Exemple : 20 000 séries, intervalle 30 s, rétention 90 jours
Échantillons = 20 000 × (90 × 86 400 / 30) = 20 000 × 259 200 = 5,184 × 10^9
Stockage ≈ 5,184e9 × 2 octets ≈ 10,4 Go
```

### 13.2 Cardinalité

Le nombre de séries croît avec le **nombre de combinaisons d'étiquettes** (site × tunnel × interface × classe). Une étiquette à valeurs nombreuses (adresse IP source, identifiant de session) **fait exploser** la base : réservez les flux détaillés à un système adapté (collecteur de flux), pas à Prometheus.

### 13.3 Rétention par niveau

| Donnée | Rétention | Résolution |
|---|---|---|
| Métriques brutes | 30 à 90 jours | 30 à 60 s |
| Métriques agrégées | 1 à 3 ans | 5 min à 1 h (règles d'enregistrement, stockage à long terme : Thanos, Mimir, VictoriaMetrics) |
| Journaux | 90 jours à 12 mois (obligations légales) | Événements |
| Flux | 7 à 30 jours détaillés, 12 mois agrégés | Flux ou agrégats |

---

## 14. Lab guidé CP05

### 14.1 Objectif

Déployer la pile de supervision, brancher SNMP v3, sondes actives et exporter SD-WAN (ou simulation), construire deux tableaux de bord et trois alertes, puis résoudre un incident par corrélation.

### 14.2 Étapes

| # | Action | Vérification |
|---|---|---|
| 1 | Créer les fichiers de la section 9 ; `docker compose up -d` | Conteneurs `Up` |
| 2 | Vérifier Prometheus : `http://localhost:9090/targets` | Cibles `UP` |
| 3 | Configurer SNMP v3 sur un routeur de lab ; tester `snmpwalk` | Réponses |
| 4 | Générer `snmp.yml` (générateur officiel) ; ajouter la cible | Métriques `ifHCInOctets` |
| 5 | Ajouter des sondes ICMP et HTTP | `probe_success` |
| 6 | (Option) Lancer l'exporter du module SW18 ou un **générateur de métriques simulées** | Métriques `sdwan_*` |
| 7 | Dans Grafana : source de données Prometheus ; créer « Santé globale » et « Vue par site » | Tableaux |
| 8 | Ajouter les alertes (section 11) ; tester en coupant un lien | Alerte reçue |
| 9 | **Incident simulé** : saturer un lien avec `iperf3 -P 8` ; observer utilisation, gigue, perte, flux | Corrélation |
| 10 | Rédiger l'analyse de cause (modèle 12.2) | Document |
| 11 | Estimer le stockage pour 300 sites (section 13) | Calcul |

### 14.3 Générateur de métriques simulées (si pas d'API disponible)

```python
#!/usr/bin/env python3
"""metriques_simulees.py : expose des métriques de tunnels fictives (port 9877) pour s'entraîner sans Manager."""
import math
import random
import time
from prometheus_client import Gauge, start_http_server

SITES = [f"SITE-{i:02d}" for i in range(1, 9)]
UP = Gauge("sdwan_tunnel_up", "Tunnel up", ["site", "color"])
LOSS = Gauge("sdwan_tunnel_loss_percent", "Perte %", ["site", "color"])
LAT = Gauge("sdwan_tunnel_latency_ms", "Délai ms", ["site", "color"])
JIT = Gauge("sdwan_tunnel_jitter_ms", "Gigue ms", ["site", "color"])

start_http_server(9877)
t0 = time.time()
while True:
    t = time.time() - t0
    for s in SITES:
        for color, base in (("mpls", (0.05, 28, 3)), ("biz-internet", (0.3, 25, 8))):
            perte = max(0, random.gauss(base[0], 0.05))
            lat = max(1, random.gauss(base[1], 2))
            jit = max(0, random.gauss(base[2], 0.5))
            # incident simulé : SITE-03 / biz-internet se dégrade périodiquement
            if s == "SITE-03" and color == "biz-internet" and int(t // 300) % 2 == 1:
                perte += 3 + 0.5 * math.sin(t); lat += 60; jit += 20
            UP.labels(s, color).set(1)
            LOSS.labels(s, color).set(perte); LAT.labels(s, color).set(lat); JIT.labels(s, color).set(jit)
    time.sleep(15)
```

### 14.4 Grille

| Critère | Points |
|---|---|
| Pile déployée et cibles UP | 4 |
| SNMP v3 et sondes opérationnels | 4 |
| Deux tableaux de bord pertinents | 4 |
| Alertes testées (déclenchées et résolues) | 3 |
| Corrélation et rapport de cause | 4 |
| Estimation de capacité | 1 |

---

## 15. Approfondissements

### A. Glossaire du module

| Terme | Définition |
|---|---|
| **Série temporelle** | Suite de valeurs horodatées |
| **Exporter** | Programme qui expose des métriques à Prometheus |
| **Scrape** | Interrogation périodique d'une cible par Prometheus |
| **PromQL** | Langage de requête de Prometheus |
| **Cardinalité** | Nombre de séries distinctes |
| **Alertmanager** | Routage, regroupement et inhibition d'alertes |
| **Blackbox** | Sonde de l'extérieur (ICMP, HTTP, DNS, TCP) |
| **NetFlow / IPFIX** | Export de flux |
| **gNMI / YANG** | Protocole de gestion et modèles de données |
| **Loki** | Base de journaux orientée étiquettes |
| **Règle d'enregistrement** | Précalcul d'une requête coûteuse |
| **SLO / SLI** | Objectif / indicateur de niveau de service |
| **MTTD** | Temps moyen de détection |
| **Golden signals** | Latence, trafic, erreurs, saturation |
| **Runbook** | Procédure associée à une alerte |

### B. Cas d'usage détaillés

#### Cas 1. La preuve par les sondes

Une agence se plaint de lenteurs ; les équipements « ne voient rien ». Les sondes HTTP montrent un temps de connexion TLS de 2,1 s vers un SaaS **depuis cette agence seulement**, alors que ping et DNS sont normaux. Cause : un proxy transparent de l'opérateur local. Preuves transmises, correction par l'opérateur. **Leçon** : mesurez **par phase** (DNS, connexion, TLS, premier octet).

#### Cas 2. L'alerte qui sonne tous les soirs

Une alerte « CPU > 80 % » sonne chaque soir à 22 h (sauvegardes, analyse de journaux). Elle est ignorée, puis une vraie panne passe inaperçue. Correctif : alerter sur la **saturation durable** (30 min) et sur des **symptômes**, supprimer les alertes sans action. **Leçon** : la **fatigue d'alerte** est un risque opérationnel.

#### Cas 3. Détection d'un comportement anormal par les flux

Les flux montrent un site qui envoie 40 Go vers une adresse inconnue la nuit. Investigation : un poste compromis exfiltre des données. L'alerte est arrivée du **flux**, pas du pare-feu. **Leçon** : la **télémétrie de flux** est aussi un outil de sécurité.

#### Cas 4. Cardinalité hors de contrôle

Un tableau de bord ajoute l'adresse IP source comme étiquette : 2 millions de séries, Prometheus saturé. Remède : retirer l'étiquette, envoyer ces données vers le collecteur de flux. **Leçon** : **maîtrisez la cardinalité**.

#### Cas 5. Capacité prévisible

Un graphique de P95 sur 12 mois montre une progression régulière de 2,5 points par mois : le lien atteindra 70 % dans 5 mois. La commande est passée à temps. **Leçon** : la rétention longue des **agrégats** est ce qui rend la prévision possible.

#### Cas 6. Panne de la supervision elle-même

Le serveur de supervision tombe un week-end : personne ne le remarque, des incidents passent inaperçus. Remède : **supervision de la supervision** (battement de cœur vers un service externe, alerte « absence de données »). **Leçon** : « Qui surveille le surveillant ? »

### C. Retours d'expérience

1. **Mesurez ce que l'utilisateur vit**, pas seulement l'état des équipements.
2. **Peu de tableaux, bien pensés**, valent mieux que cent graphiques.
3. **Chaque alerte a un propriétaire et une procédure.**
4. **La supervision doit survivre à la panne qu'elle surveille** : hébergez-la hors du périmètre qu'elle observe, ou prévoyez un témoin externe.
5. **Documentez les seuils** (pourquoi 90 % ? pourquoi 15 minutes ?).
6. **Gardez les unités et les conventions de nommage** cohérentes.
7. **Automatisez le déploiement de la supervision** (Compose/Ansible) : elle se reconstruit en 10 minutes.
8. **Sécurisez-la** : accès, mots de passe, SNMP v3, TLS : la supervision connaît tout de votre réseau.

### D. Exercices supplémentaires

1. Écrivez la requête PromQL de disponibilité d'un site sur 7 jours à partir de `probe_success`.
2. Estimez le stockage pour 300 sites, 40 séries par site pour SNMP et 60 séries par site pour les tunnels, intervalle 60 s, rétention 180 jours (2 octets par échantillon).
3. Rédigez 5 alertes pour un WAN avec seuils et durées justifiés.
4. Expliquez la différence entre `rate()` et `increase()` en PromQL et quand utiliser chacun.
5. Proposez l'architecture de supervision d'une entreprise de 500 sites dispersés (collecteurs régionaux, fédération).

<details>
<summary>Corrigés</summary>

1. `100 * avg_over_time(probe_success{instance="X"}[7d])`.
2. Séries : 300 × (40 + 60) = 30 000 ; échantillons : 30 000 × (180 × 86 400 / 60) = 30 000 × 259 200 = 7,776e9 ; stockage ≈ 15,6 Go.
4. `rate()` : taux par seconde moyen sur la fenêtre (graphiques de débit) ; `increase()` : augmentation totale sur la fenêtre (nombre d'erreurs sur 1 h).
5. Collecteurs Prometheus régionaux (un par zone), fédération ou écriture distante vers un stockage central (Thanos/Mimir), Grafana central, Alertmanager en haute disponibilité, supervision externe de la chaîne.

</details>

### E. Mini-projet (10 heures)

Concevez et déployez la **plateforme d'observabilité complète** d'un WAN fictif de 20 sites : pile Prometheus/Grafana/Alertmanager, SNMP v3, sondes, export SD-WAN (réel ou simulé), flux (nfdump), journaux (rsyslog + Loki ou fichiers), 5 tableaux de bord, 8 alertes avec runbooks, supervision de la supervision, estimation de capacité et rapport d'un incident corrélé.

---

## 16. Quiz

1. Quelle est la différence entre supervision et observabilité ?
2. Pourquoi SNMP v3 en mode authPriv ?
3. Que mesure NetFlow que SNMP ne mesure pas ?
4. Pourquoi des sondes actives en plus des métriques d'équipement ?
5. Qu'est-ce que la cardinalité ?
6. Pourquoi une alerte doit-elle avoir une durée minimale ?
7. À quoi sert l'inhibition dans Alertmanager ?
8. Quelle est la formule de stockage approximatif d'une base de séries temporelles ?

<details>
<summary>Réponses</summary>

1. La supervision vérifie des états connus ; l'observabilité aide à comprendre des états inconnus en croisant plusieurs sources. 2. Il authentifie et chiffre. 3. Qui parle à qui, avec quel volume (flux). 4. Elles mesurent l'expérience de bout en bout. 5. Le nombre de séries distinctes (combinaisons d'étiquettes). 6. Pour éviter de déclencher sur de brefs événements. 7. À supprimer des alertes dérivées quand une alerte parente est active. 8. Séries × échantillons par série × octets par échantillon.

</details>

---

## 17. Aide-mémoire

```text
Sources : API Manager | SNMP v3 authPriv | syslog (TCP/TLS) | NetFlow/IPFIX | streaming gNMI | sondes (blackbox) | BFD/AAR
Pile : exporters -> Prometheus -> Grafana (+ Alertmanager) ; journaux -> Loki ; flux -> nfdump/Akvorado
PromQL : rate() pour débit ; avg_over_time pour disponibilité ; quantile_over_time(0.95, ...) pour P95
Alertes : symptôme, durée (for), gravité, runbook, inhibition ; peu mais actionnables ; supervision de la supervision
Stockage ≈ séries x (rétention/intervalle) x 2 octets ; maîtriser la cardinalité ; agrégats longs
Corrélation : métriques + journaux + flux + changements sur la même fenêtre
```

## 18. Pour aller plus loin

- Prometheus : <https://prometheus.io/docs/>
- Grafana : <https://grafana.com/docs/>
- snmp_exporter, blackbox_exporter : <https://github.com/prometheus>
- gnmic : <https://gnmic.openconfig.net>
- Akvorado : <https://github.com/akvorado/akvorado>
- Livre : « Prometheus: Up & Running » ; « Observability Engineering ».

> Module précédent : [CP04](../CP04-flexiwan/CP04-flexiwan.md) | Module suivant : [CP06](../CP06-preparation-certifications/CP06-preparation-certifications.md)
