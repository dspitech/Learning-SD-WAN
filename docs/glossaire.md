# Glossaire général du cours SD-WAN

> Classement alphabétique. Les définitions sont volontairement courtes ; chaque module contient un glossaire plus détaillé. Les noms de produits sont donnés à titre d'exemple et évoluent.

| Terme | Définition |
|---|---|
| **AAR** (Application-Aware Routing) | Choix du chemin selon l'application et la qualité mesurée (perte, délai, gigue) |
| **ADR** | Architecture Decision Record : décision d'architecture documentée |
| **ADVPN** | Tunnels dynamiques rayon à rayon à la demande (Fortinet) |
| **AEAD** | Chiffrement authentifié (AES-GCM, ChaCha20-Poly1305) |
| **AMP** | Protection contre les logiciels malveillants (réputation de fichiers) |
| **API REST** | Interface de programmation HTTP pour automatiser une plateforme |
| **ASN / AS** | Numéro de système autonome / système autonome (BGP) |
| **Backhauling** | Ramener tout le trafic vers un point central avant de l'envoyer ailleurs |
| **BFD** | Bidirectional Forwarding Detection : détection rapide de panne, mesure de qualité en SD-WAN |
| **BGP** | Protocole de routage entre systèmes autonomes |
| **Brownout** | Dégradation partielle d'un lien (pertes, lenteurs) sans coupure franche |
| **Bufferbloat** | Latence excessive causée par des files d'attente trop grandes |
| **CASB** | Courtier de sécurité d'accès au cloud (contrôle des usages SaaS) |
| **CE / PE / P** | Routeur client / de bordure opérateur / de cœur opérateur |
| **CGNAT** | NAT à grande échelle chez l'opérateur (adresse publique partagée) |
| **Chassis ID / UUID** | Identifiant matériel ou virtuel d'un équipement |
| **Classe de SLA** | Seuils de perte, délai et gigue définissant un chemin conforme |
| **Cloud OnRamp** | Ensemble de fonctions Cisco de connexion et d'optimisation vers les clouds |
| **Colocation** | Hébergement d'équipements dans un centre de données partagé |
| **Config drift** | Écart entre la configuration voulue et la configuration réelle |
| **Configuration group** | Ensemble de profils de configuration associé à des équipements (approche récente) |
| **Controller (vSmart)** | Composant du plan de contrôle : distribue routes, clés et politiques |
| **Couleur (color)** | Étiquette d'un type de transport (mpls, biz-internet, lte…) |
| **CSR** | Demande de signature de certificat |
| **Data policy** | Politique de transfert appliquée par le WAN Edge (NAT DIA, chaînage…) |
| **Day-0 / Day-1 / Day-2** | Amorçage / configuration de service / exploitation |
| **DIA** | Accès Internet dédié, ou sortie Internet directe depuis le site |
| **DiffServ / DSCP** | Modèle de classes de service / champ de marquage de 6 bits de l'en-tête IP |
| **DLP** | Prévention des fuites de données |
| **DMPO** | Optimisation dynamique multi-chemins (VeloCloud) |
| **DMVPN** | Maillage dynamique rayon à rayon (mGRE, NHRP, IPsec) |
| **DPD** | Détection d'un pair IPsec silencieux |
| **DPI** | Inspection approfondie de paquets pour identifier une application |
| **DTLS / TLS** | Chiffrement des connexions de contrôle (UDP / TCP) |
| **ECMP** | Utilisation de plusieurs chemins de coût égal |
| **ESP** | Protocole IPsec de chiffrement et d'intégrité (protocole IP 50) |
| **FEC** | Correction d'erreurs par redondance, sans retransmission |
| **FWaaS** | Pare-feu fourni en service cloud |
| **GRE** | Encapsulation IP générique sans chiffrement |
| **GTR** | Garantie de temps de rétablissement après panne |
| **Hairpinning** | Détour du trafic qui entre et ressort par le même point |
| **Hub and spoke** | Topologie en concentrateur et rayons |
| **IdP** | Fournisseur d'identité (SSO) |
| **IKEv2** | Protocole d'échange de clés d'IPsec |
| **IDS / IPS** | Détection / prévention d'intrusion |
| **IPAM** | Gestion centralisée des adresses IP |
| **IPsec** | Suite de protocoles de chiffrement au niveau IP |
| **LLQ** | File à priorité stricte bornée pour la voix |
| **Local breakout** | Sortie Internet locale depuis le site |
| **MPLS** | Commutation par étiquettes ; base des VPN L3 d'opérateur |
| **MRF** | Multi-Region Fabric : fabrique découpée en régions |
| **MSS / MTU** | Taille maximale de segment TCP / de paquet IP |
| **MTTD / MTTR / MTBF** | Délais moyens de détection / de rétablissement / entre pannes |
| **NAT / NAT-T** | Traduction d'adresses / traversée de NAT par encapsulation UDP 4500 |
| **NETCONF** | Protocole de gestion de configuration sur SSH (TCP 830) |
| **NetFlow / IPFIX** | Export de statistiques de flux |
| **NWPI** | Network-Wide Path Insight : trace d'un flux de bout en bout |
| **OMP** | Overlay Management Protocol : protocole de contrôle de l'overlay Cisco |
| **OT / ICS** | Technologies opérationnelles / systèmes de contrôle industriels |
| **Overlay / underlay** | Réseau logique de tunnels / réseau de transport sous-jacent |
| **PfR** | Performance Routing : choix de chemin selon la performance (précurseur du SD-WAN) |
| **PKI** | Infrastructure à clés publiques |
| **PMTUD** | Découverte de la MTU du chemin |
| **POC** | Preuve de concept |
| **PoP** | Point de présence d'un service cloud |
| **QoS** | Qualité de service : traitement différencié du trafic |
| **RBAC** | Contrôle d'accès par rôle |
| **RTO / RPO** | Durée maximale d'interruption / perte maximale de données acceptées |
| **SASE** | Convergence SD-WAN et sécurité cloud (SSE) |
| **SD-WAN Manager (vManage)** | Composant du plan de gestion |
| **SD-WAN Validator (vBond)** | Composant du plan d'orchestration |
| **SGT / TrustSec** | Groupes de sécurité étiquetant les terminaux |
| **SIEM** | Plateforme de collecte et de corrélation des journaux de sécurité |
| **SIG** | Secure Internet Gateway : passerelle de sécurité Internet dans le cloud |
| **Site ID** | Identifiant d'un site dans la fabrique |
| **SLA / SLO / SLI** | Engagement contractuel / objectif interne / indicateur mesuré de niveau de service |
| **SSE** | Security Service Edge : partie sécurité du SASE (SWG, CASB, ZTNA…) |
| **STUN** | Protocole de découverte de son adresse publique et du comportement du NAT |
| **SWG** | Passerelle web sécurisée |
| **System IP** | Identifiant unique d'un équipement dans l'overlay |
| **TCO** | Coût total de possession |
| **TLOC** | Point de terminaison de transport : (system-ip, couleur, encapsulation) |
| **UTD** | Unified Threat Defense : conteneur de services de sécurité sur le routeur |
| **VPN / VRF** | Segment de l'overlay / table de routage virtuelle |
| **VPN 0 / VPN 512** | Transport (global) / gestion hors bande |
| **VRRP** | Redondance de passerelle de premier saut |
| **WAN Edge** | Routeur de bordure d'un site, plan de données |
| **WireGuard** | Protocole de VPN moderne et léger |
| **Zero Trust** | Modèle où aucun accès n'est accordé par défaut ; vérification continue |
| **ZTNA** | Accès réseau Zero Trust par application |
| **ZTP** | Mise en service sans intervention locale |
| **ZBFW** | Pare-feu à zones |
