# SD-WAN : Identité et confiance, la PKI

Résumé du cours. Objectif : comprendre comment un réseau SD-WAN décide à qui faire confiance.

## Idée centrale

Trois questions pour chaque équipement :

1. Qui es-tu ? (certificat)
2. Ton certificat est-il valable ? (chaîne, dates, révocation)
3. As-tu le droit d'être ici ? (liste d'autorisation, nom d'organisation)

Il faut réussir les trois.

## Vocabulaire minimal

| Terme | Sens simple |
|---|---|
| PKI | Système qui gère les certificats |
| CA | Autorité qui signe les certificats |
| Certificat | Carte d'identité numérique d'un équipement |
| CSR | Demande de certificat envoyée à la CA |
| TPM | Puce de sécurité qui protège la clé privée |
| SUDI | Identité unique d'un équipement, inscrite en usine |
| TLS / DTLS | Chiffrement sur TCP / sur UDP |
| CRL / OCSP | Moyens de savoir si un certificat est révoqué |
| Allowed list | Liste des équipements autorisés |

## Les acteurs

| Rôle | Nom Cisco | Fonction |
|---|---|---|
| Manager | vManage | Administration, certificats, supervision |
| Controller | vSmart | Distribue routes et politiques |
| Validator | vBond | Authentifie et oriente les Edges |
| Edge | cEdge, vEdge | Routeur du site |

## Choix de la CA

| Option | Avantage | Limite |
|---|---|---|
| CA du constructeur | Simple et rapide | Dépendance, moins de contrôle |
| CA d'entreprise | Politique unifiée, outils existants | Pas toujours adaptée aux routeurs distants |
| CA privée dédiée | Isolation et contrôle total | Il faut l'exploiter |

Bonne pratique : racine hors ligne, intermédiaire en ligne.

## Certificats

- Contrôleurs (Manager, Controllers, Validator) : certificat signé par la CA choisie. Le certificat racine est installé partout.
- Edge, option 1 : certificat matériel (TPM, SUDI). Clé non copiable, identité dès le premier démarrage.
- Edge, option 2 : certificat d'enrôlement signé par votre CA.

## Authentification mutuelle TLS / DTLS

Les deux côtés présentent un certificat. Chacun vérifie :

1. La chaîne jusqu'à une racine connue
2. La validité (dates, horloge correcte)
3. La révocation
4. Le nom d'organisation
5. Le droit d'accès (liste d'autorisation)

Une seule vérification en échec suffit pour refuser la connexion.

## Déroulement d'un Edge

1. L'Edge obtient une connectivité.
2. Il contacte le Validator, authentification mutuelle.
3. Le Validator contrôle certificat, numéro de série et organisation.
4. Il indique l'adresse du Manager et des Controllers.
5. L'Edge se connecte au Controller et au Manager.
6. Il reçoit configuration, routes et politiques, puis monte ses tunnels de données.

## Listes d'autorisation

- Allowed list des Edges : numéros de série autorisés.
- Liste des contrôleurs : seuls les vrais contrôleurs sont acceptés.
- Nom d'organisation : identique et exact sur tous les équipements.

## Révocation d'un Edge

1. Identifier le numéro de série.
2. Le marquer invalide ou le retirer de l'allowed list dans le Manager.
3. Propager vers les contrôleurs et le Validator.
4. Vérifier la fermeture des sessions.
5. En cas de compromission, révoquer aussi le certificat côté CA.
6. Journaliser l'action.

## Dépannage rapide

Vérifier dans cet ordre : heure, chaîne, validité, organisation, liste d'autorisation.

| Symptôme | Cause probable |
|---|---|
| Edge ne rejoint pas l'overlay | Serial absent de l'allowed list |
| Échec dès le premier échange | Horloge incorrecte |
| Refus avec certificat valide | Nom d'organisation différent |
| Erreur de chaîne | Certificat racine absent ou erroné |
| Pannes massives à une date | Certificat de contrôleur expiré |
| Edge retiré qui revient | Révocation non propagée |

## Bonnes pratiques

- NTP fiable partout.
- Supervision des dates d'expiration, renouvellement anticipé.
- Clé racine protégée, hors ligne.
- Clés stockées en puce (TPM) quand c'est possible.
- Allowed list limitée aux équipements réellement déployés.
- Procédure de révocation documentée et testée.
- Environnements laboratoire et production séparés.
- Journalisation et supervision des authentifications.

## Questions de révision

1. Pourquoi l'authentification est-elle mutuelle ?
2. Différence entre certificat valide et équipement autorisé ?
3. Intérêt d'une clé dans une puce TPM ?
4. Pourquoi garder la CA racine hors ligne ?
5. Que faire si un routeur est volé ?

Réponses : 1) les deux parties se vérifient. 2) le certificat prouve l'identité, la liste donne le droit d'entrer. 3) la clé ne peut pas être extraite. 4) pour protéger la clé la plus critique. 5) invalider dans le Manager, propager, révoquer le certificat.

## Remarque

Les noms de menus, commandes et ports varient selon le constructeur et la version. Se référer à la documentation officielle de la plateforme utilisée.

---

# Partie 2 : Cas pratique d'un grand distributeur (scénario fictif)

Avertissement : scénario pédagogique inventé. Les noms, numéros et durées ne décrivent pas l'architecture réelle d'une entreprise existante.

## Contexte

Un groupe de distribution possède un siège, des entrepôts et des milliers de magasins (hypermarchés, supermarchés, proximité, drives). Les flux des caisses et du paiement sont très sensibles. Un faux routeur ou un routeur volé qui rejoindrait le réseau donnerait accès aux caisses et au siège. La PKI sert à empêcher cela.

## Architecture de l'exemple

| Composant | Nombre | Emplacement |
|---|---|---|
| Manager | Cluster de 3 nœuds | DC Paris et DC Lyon |
| Controllers | 2 | 1 par centre de données |
| Validators | 2 | 1 par centre de données |
| Edges | 1 ou 2 par magasin | Sites distants |

Nom d'organisation : `GROUPE-DISTRIB-PROD`

## Hiérarchie de CA retenue

```
CA racine SD-WAN (hors ligne, coffre physique)
   |
   +-- CA intermédiaire "Controllers" : Manager, Controllers, Validators
   |
   +-- CA intermédiaire "Edges"       : Edges des magasins
```

Deux intermédiaires pour cloisonner : si celle des Edges est compromise, on la remplace sans toucher aux contrôleurs.

## Durées de vie proposées

| Certificat | Durée |
|---|---|
| CA racine | 20 ans |
| CA intermédiaires | 10 ans |
| Contrôleurs | 2 ans |
| Edges | 3 ans, avec renouvellement automatisé |

## Ouverture d'un nouveau magasin

1. Environ 30 jours avant : commande du routeur, le constructeur lie le numéro de série au compte, import dans l'allowed list du Manager.
2. Environ 15 jours avant : préparation du modèle de configuration (segmentation, qualité de service, liens fibre et 4G/5G), association au numéro de série.
3. Environ 7 jours avant : expédition directe au magasin, routeur vierge.
4. Jour J : le magasin branche alimentation, fibre et câble vers le switch, rien d'autre.
5. Le routeur contacte le Validator, qui contrôle la chaîne, la validité, la révocation, l'organisation et le numéro de série.
6. Le Validator indique les adresses du Manager et des Controllers.
7. Le routeur reçoit sa configuration, ses routes et ses politiques, puis monte ses tunnels.

## Extrait d'allowed list (fictif)

| Numéro de série | Magasin | Rôle | Statut |
|---|---|---|---|
| FGL2345A1B2 | 5873 (proximité) | Edge unique | Valide |
| FGL7788C3D4 | 0142 (hyper) | Edge principal | Valide |
| FGL7789C3D5 | 0142 (hyper) | Edge secondaire | Valide |
| FGL0011X9Z8 | 3310 (supermarché) | Edge unique | Invalide (volé) |
| FGL5566T6U7 | Entrepôt Nord | Edge | Valide |

## Incidents types

| Incident | Traitement |
|---|---|
| Routeur volé | Passer invalide dans le Manager, propager, révoquer le certificat, commander un remplaçant, journaliser |
| Horloge fausse après coupure | NTP fiable, la session reprend dès que l'heure est correcte |
| Certificat de contrôleur expiré | Alertes à 90, 60 et 30 jours, procédure de renouvellement testée |
| Magasin passé en franchise | Garder l'Edge sous contrôle du groupe avec segmentation, ou invalider, révoquer et remettre à zéro |
| Remplacement après panne | Invalider l'ancien numéro, ajouter le nouveau à l'allowed list, même configuration |

Piège classique : un routeur de remplacement neuf et valide, mais refusé car son numéro de série n'a pas été ajouté.

## Segmentation après authentification

Le certificat identifie l'équipement, pas le trafic. La protection des flux repose ensuite sur la segmentation :

| Segment | Règle |
|---|---|
| Paiement | Uniquement vers les serveurs de paiement, pas d'accès Internet direct |
| Gestion magasin | Accès aux applications internes |
| Vidéosurveillance | Vers l'enregistreur central uniquement |
| Wifi invité | Internet uniquement, isolé du reste |

## Indicateurs de suivi

| Indicateur | Fréquence |
|---|---|
| Certificats de contrôleurs expirant dans 90 jours | Hebdomadaire |
| Certificats d'Edges expirant dans 90 jours | Hebdomadaire |
| Edges de l'allowed list non connectés depuis X jours | Quotidienne |
| Authentifications refusées | Temps réel avec alerte |
| Écarts d'horloge | Quotidienne |
| Revue allowed list contre inventaire réel | Trimestrielle |

## Exercice

Le magasin 7721 vient d'ouvrir. Le journal du Validator indique : certificat valide, organisation correcte, numéro de série inconnu.

1. Cause probable : numéro de série absent de l'allowed list.
2. Correction : l'ajouter dans le Manager, propager, attendre la reconnexion.
3. Erreur de processus : import depuis le portail du constructeur oublié, ou routeur expédié différent de celui commandé.

## Checklist de déploiement

- Définir la hiérarchie de CA et les durées de vie.
- Protéger la CA racine hors ligne, avec double contrôle.
- Figer le nom d'organisation.
- Automatiser l'import des numéros de série.
- Standardiser l'installation en magasin.
- Déployer NTP fiable partout.
- Mettre en place les alertes d'expiration.
- Documenter et tester les procédures de vol, remplacement et fermeture.
- Revue trimestrielle de l'allowed list.

---

# Partie 3 : Approfondissements techniques

Cette partie va plus loin : déroulement détaillé d'une connexion, cycle de vie des certificats, révocation, commandes de diagnostic, codes d'erreur, FAQ et plan de formation. Les noms de commandes et de codes sont donnés à titre d'exemple pour une plateforme de type Cisco Catalyst SD-WAN. Ils varient selon le constructeur et la version, il faut toujours les vérifier dans la documentation officielle.

## 1. Rappel visuel : qui signe quoi

```
CA racine (hors ligne)
   |
   +-- CA intermédiaire Controllers --> Manager, Controllers, Validators
   |
   +-- CA intermédiaire Edges        --> Edges des magasins
```

Règle d'or : chaque équipement doit posséder la chaîne de confiance complète (racine et intermédiaires) pour valider le certificat présenté par son interlocuteur.

## 2. Déroulement détaillé d'une connexion d'Edge

| Étape | Qui | Action | Contrôles effectués |
|---|---|---|---|
| 1 | Edge | Obtient une adresse IP et une route (DHCP, statique, ou lien 4G/5G) | Aucun, simple connectivité |
| 2 | Edge | Résout le nom ou l'adresse du Validator et ouvre une session DTLS ou TLS | Le Validator présente son certificat, l'Edge vérifie la chaîne, la validité, l'organisation |
| 3 | Edge | Présente son propre certificat (authentification mutuelle) | Le Validator vérifie la chaîne, la validité, la révocation, l'organisation |
| 4 | Validator | Consulte son allowed list | Le numéro de série de l'Edge est-il autorisé ? |
| 5 | Validator | Envoie à l'Edge la liste des Controllers et du Manager | L'Edge apprend à qui parler ensuite |
| 6 | Edge et Validator | Ferment la session | Le Validator n'est qu'un portier, il ne reste pas connecté |
| 7 | Edge | Ouvre une session vers un Controller | Mêmes contrôles mutuels, plus allowed list du Controller |
| 8 | Edge | Ouvre une session vers le Manager | Mêmes contrôles, puis récupération de la configuration |
| 9 | Edge | Reçoit routes et politiques via le protocole de contrôle | Plan de contrôle opérationnel |
| 10 | Edge | Monte ses tunnels de données vers les autres sites | Le trafic utilisateur peut circuler |

Point clé : l'Edge n'a besoin de connaître au départ que l'adresse du Validator. Tout le reste lui est donné ensuite.

## 3. Cycle de vie complet d'un certificat d'Edge

1. **Naissance** : fabrication en usine, identité matérielle inscrite dans la puce.
2. **Enrôlement** : l'Edge demande un certificat d'entreprise, la CA le signe.
3. **Utilisation** : le certificat sert à chaque établissement de session.
4. **Surveillance** : supervision des dates d'expiration et de l'état.
5. **Renouvellement** : un nouveau certificat est émis avant l'échéance, idéalement de façon automatique.
6. **Fin de vie** : l'Edge est retiré, on l'invalide dans le Manager et on révoque son certificat.
7. **Recyclage** : remise à zéro de l'équipement avant revente ou destruction.

Règle pratique : renouveler quand il reste environ un tiers de la durée de vie, pas à la dernière minute.

## 4. Révocation : comment ça marche vraiment

### CRL (Certificate Revocation List)
- La CA publie périodiquement un fichier avec les numéros de série des certificats révoqués.
- Avantage : simple.
- Limite : la liste peut grossir, et elle n'est valable que jusqu'à sa prochaine mise à jour.

### OCSP (Online Certificate Status Protocol)
- L'équipement interroge un service en ligne : "ce certificat est il encore valide ?"
- Avantage : réponse précise et récente.
- Limite : dépend de la disponibilité du service OCSP.

### Invalidation par la plateforme SD-WAN
- Le Manager marque l'Edge invalide et propage l'information aux contrôleurs.
- Avantage : action rapide, indépendante de la CA.
- Limite : ne protège que le périmètre de cette plateforme.

### Stratégie recommandée
| Situation | Action |
|---|---|
| Équipement retiré proprement | Invalider dans le Manager, supprimer de l'allowed list |
| Équipement volé ou perdu | Invalider tout de suite, puis révoquer côté CA |
| Clé privée suspectée compromise | Invalider, révoquer, remplacer l'équipement, enquêter |
| CA intermédiaire compromise | Révoquer l'intermédiaire, en créer une nouvelle, ré-émettre tous les certificats concernés |

## 5. Les états d'un Edge dans le Manager (vue simplifiée)

| État | Signification | Peut rejoindre l'overlay ? |
|---|---|---|
| Attendu | Numéro de série importé, pas encore connecté | Oui, dès qu'il se présente |
| Valide | Connecté et authentifié | Oui |
| Invalide | Retiré ou compromis | Non |
| Staging | Mode intermédiaire pour préparation ou test | Selon la politique |

## 6. Commandes de diagnostic (exemples de type Cisco)

À lancer sur un Edge ou un contrôleur pour comprendre un problème d'authentification. Les commandes exactes dépendent de la version et du modèle.

| Objectif | Commande type |
|---|---|
| Voir l'état des connexions de contrôle | `show control connections` ou `show sdwan control connections` |
| Voir l'historique des échecs | `show control connections-history` ou `show sdwan control connections-history` |
| Voir le certificat installé | `show certificate installed` ou `show sdwan certificate installed` |
| Voir le numéro de série et le statut | `show certificate serial` ou `show sdwan certificate serial` |
| Voir les propriétés locales (organisation, identifiant) | `show control local-properties` ou `show sdwan control local-properties` |
| Vérifier l'heure | `show clock` |
| Installer la chaîne de CA racine | `request root-cert-chain install <chemin>` |

Méthode : commencer par l'historique des connexions, il contient souvent un code d'erreur explicite.

## 7. Codes d'erreur fréquents (à vérifier dans la documentation officielle)

| Code | Sens approximatif | Piste de résolution |
|---|---|---|
| CRTREJSER | Certificat rejeté par le pair | Vérifier la chaîne, la validité et la liste |
| CRTVERFL | Échec de vérification du certificat | Vérifier l'heure, la racine installée, l'expiration |
| BIDNTVRFD | Identité du pair non vérifiée | Numéro de série probablement absent de l'allowed list |
| SERNTPRES | Numéro de série absent | Contrôler l'import des équipements |
| ORGNMMIS | Nom d'organisation différent | Aligner le nom d'organisation partout |
| VMCRTREV | Certificat du Manager révoqué | Vérifier l'état du certificat du Manager |
| DCONFAIL | Échec de connexion DTLS | Vérifier les ports, le pare-feu, le transport |
| SSLNFAIL | Échec de négociation TLS | Vérifier certificats et compatibilité de version |

## 8. Arbre de décision pour un Edge qui ne se connecte pas

1. L'Edge a-t-il une adresse IP et une route vers Internet ? Sinon : problème de lien, pas de PKI.
2. Atteint-il l'adresse du Validator (ping, résolution de nom, ports ouverts) ? Sinon : pare-feu ou DNS.
3. L'heure de l'Edge est elle correcte ? Sinon : corriger NTP.
4. L'Edge possède-t-il la bonne chaîne racine ? Sinon : installer la chaîne.
5. Son certificat est-il valide et non expiré ? Sinon : renouveler ou ré-enrôler.
6. Le nom d'organisation est il identique à celui des contrôleurs ? Sinon : corriger.
7. Son numéro de série est il dans l'allowed list, avec le statut valide ? Sinon : l'ajouter ou le réactiver.
8. Le certificat est-il révoqué ? Si oui : décider si c'est volontaire, sinon ré-émettre.

## 9. Sécurité de la CA : points de vigilance

- La clé de la CA racine ne doit jamais être sur une machine connectée au réseau.
- Utiliser un module de sécurité matériel (HSM) pour protéger les clés des CA.
- Exiger un double contrôle humain pour toute opération sur la racine.
- Sauvegarder la clé racine dans plusieurs lieux sécurisés, avec procédure de restauration testée.
- Journaliser chaque signature de certificat.
- Tester périodiquement un scénario de compromission d'intermédiaire.

## 10. Résilience et continuité

| Scénario | Conséquence | Mesure |
|---|---|---|
| Panne du Validator | Les nouveaux Edges ne peuvent pas s'enrôler, les Edges déjà connectés continuent | Au moins 2 Validators dans des sites distincts |
| Panne d'un Controller | Les Edges se rabattent sur l'autre Controller | 2 Controllers minimum |
| Panne du Manager | Plus de configuration ni de supervision, le trafic continue | Cluster de 3 nœuds, sauvegardes régulières |
| Panne de la CA en ligne | Pas de nouveaux certificats, les existants restent valables | Intermédiaire redondante, procédure de reprise |
| Certificat de contrôleur expiré | Rupture du plan de contrôle | Alertes à 90, 60, 30 jours |

## 11. Exemple de politique de sécurité PKI (modèle)

1. La CA racine est hors ligne et gérée par deux personnes habilitées.
2. Les CA intermédiaires sont dédiées : une pour les contrôleurs, une pour les Edges.
3. Les certificats de contrôleurs durent 2 ans, ceux des Edges 3 ans.
4. Le renouvellement est déclenché 90 jours avant l'échéance.
5. Tout équipement perdu ou volé est invalidé sous 1 heure après détection.
6. L'allowed list est comparée à l'inventaire chaque trimestre.
7. Toute modification de l'allowed list est tracée (qui, quand, pourquoi).
8. Les journaux d'authentification sont conservés au moins 12 mois.
9. Un exercice de révocation est réalisé chaque année.

## 12. FAQ

**Un certificat valide suffit-il pour rejoindre le réseau ?**
Non. Il faut aussi que le numéro de série soit dans l'allowed list et que l'organisation corresponde.

**Pourquoi tant d'importance pour l'heure ?**
Un certificat a une date de début et de fin. Si l'horloge est fausse, un certificat valide peut sembler expiré ou pas encore actif.

**Que se passe-t-il si le Validator tombe en panne ?**
Les Edges déjà connectés continuent. Seuls les nouveaux enrôlements sont bloqués.

**Peut-on utiliser une seule CA pour tout ?**
C'est possible, mais risqué. Séparer contrôleurs et Edges limite l'impact d'une compromission.

**Le certificat protège-t-il le trafic des utilisateurs ?**
Il authentifie les équipements. Le chiffrement des tunnels de données et la segmentation assurent ensuite la protection du trafic.

**Que faire d'un routeur retiré ?**
L'invalider, révoquer son certificat, le remettre à zéro, puis seulement le revendre ou le détruire.

**Quelle différence entre TLS et DTLS ?**
Même sécurité, mais TLS fonctionne sur TCP et DTLS sur UDP, plus léger pour certains flux.

**Pourquoi un nom d'organisation si les certificats suffisent ?**
C'est une barrière supplémentaire : deux organisations différentes ne peuvent pas se mélanger même avec la même CA.

## 13. Pièges classiques à éviter

- Oublier d'ajouter le nouveau numéro de série après un remplacement.
- Laisser expirer un certificat de contrôleur.
- Mélanger production et laboratoire avec la même organisation et la même CA.
- Ne pas synchroniser l'heure.
- Écrire le nom d'organisation différemment sur deux équipements.
- Garder la CA racine en ligne par commodité.
- Ne jamais tester la procédure de révocation.
- Laisser dans l'allowed list des équipements retirés depuis longtemps.

## 14. Plan de formation en 5 séances

| Séance | Thème | Objectif |
|---|---|---|
| 1 | Bases de la PKI | Comprendre clé, certificat, CA, chaîne |
| 2 | Acteurs du SD-WAN | Distinguer Manager, Controller, Validator, Edge |
| 3 | Authentification mutuelle | Savoir ce qui est vérifié à chaque connexion |
| 4 | Listes et révocation | Gérer l'allowed list et traiter un incident |
| 5 | Diagnostic | Résoudre une panne d'enrôlement avec l'arbre de décision |

## 15. Glossaire étendu

| Terme | Définition simple |
|---|---|
| Allowed list | Liste des numéros de série autorisés |
| CA | Autorité qui signe les certificats |
| Chaîne de certification | Suite de certificats de l'équipement jusqu'à la racine |
| CRL | Liste des certificats révoqués |
| CSR | Demande de certificat |
| DTLS | Version de TLS pour UDP |
| Edge | Routeur du site distant |
| HSM | Boîtier matériel qui protège les clés |
| NTP | Protocole de synchronisation de l'heure |
| OCSP | Service qui indique l'état d'un certificat |
| Overlay | Réseau virtuel construit au dessus des liens physiques |
| PKI | Infrastructure de gestion des certificats |
| SCEP | Protocole d'enrôlement automatique de certificats |
| SUDI | Identité unique d'un équipement, inscrite en usine |
| TPM | Puce de sécurité qui protège les clés |
| Zero touch | Mise en service sans configuration manuelle sur place |

## 16. Questions de révision supplémentaires

1. Dans quel ordre l'Edge contacte-t-il Validator, Controller et Manager ?
2. Pourquoi le Validator ferme-t-il la session après avoir donné les adresses ?
3. Quelle est la différence entre invalider un Edge et révoquer son certificat ?
4. Que se passe-t-il pour les Edges existants si le Validator tombe ?
5. Pourquoi séparer la CA des contrôleurs et celle des Edges ?
6. Quel code d'erreur évoque un numéro de série absent de la liste ?
7. Quelle première vérification faire après une longue coupure de courant sur un site ?

Réponses courtes : 1) Validator, puis Controller et Manager. 2) Il n'est qu'un portier d'orientation. 3) L'invalidation agit dans la plateforme, la révocation agit côté CA. 4) Ils continuent, seuls les nouveaux enrôlements sont bloqués. 5) Pour limiter l'impact d'une compromission. 6) Par exemple BIDNTVRFD ou SERNTPRES, à confirmer dans la documentation. 7) L'heure de l'équipement.
