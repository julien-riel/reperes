# PRD — RoadSense iOS Collector & Post-Processing Pipeline

**Version :** 0.1 (draft)
**Date :** 2026-04-29
**Auteur :** Julien (avec assistance Claude)
**Statut :** Discovery / spec initiale

---

## 1. Vision et objectif

Construire un **système de collecte et d'analyse de données géospatiales routières basé sur iPhone 16 Pro**, capable de produire automatiquement, à partir d'un trajet en voiture, des couches géoréférencées de qualité municipale :

- **Défauts de chaussée** géolocalisés (nids-de-poule, fissures, affaissements, marquages dégradés)
- **Inventaire et état de la signalisation** verticale (panneaux)
- **Vérification d'inventaire** des actifs municipaux ponctuels (regards d'égout, bornes-fontaines, mobilier urbain)
- **Trajectoire de référence** géoréférencée à haute précision (≤30 cm) utilisable pour d'autres analyses

L'iPhone agit comme un **capteur intelligent** : il enregistre fidèlement et de façon synchronisée. Toute l'intelligence (détection, fusion, optimisation) est déportée en **post-traitement serveur**.

### Positionnement

Alternative low-cost et déployable à grande échelle aux relevés professionnels (véhicules instrumentés type IRI, MMS LiDAR mobile à 6 chiffres). Cible : municipalités du Québec et services techniques (TP, voirie, signalisation).

### Critères de succès (v1)

- Précision de localisation des objets détectés : **≤30 cm (1σ)** en milieu urbain ouvert, **≤1 m** en conditions dégradées (canyon urbain, tunnels courts).
- Couverture : **2 h de roulage continu** sans échec thermique ni coupure d'enregistrement.
- Qualité dataset : ≥85 % rappel sur défauts majeurs (≥10 cm de profondeur ou ≥30 cm de diamètre), ≥95 % rappel sur signalisation standard.
- Rendement : **1 km de rue traité de bout-en-bout en moins de 5 min** de post-traitement (cible scalable).
- Sortie compatible avec les outils SIG existants (OGC API Features / WFS / GeoJSON / GeoPackage).

---

## 2. Personas et cas d'usage

**P1 — Technicien voirie municipal**
Veut une carte à jour des nids-de-poule sur son arrondissement, classés par sévérité, pour planifier les réparations. Aujourd'hui il dépend de signalements citoyens et de tournées visuelles.

**P2 — Responsable signalisation**
Veut savoir quels panneaux sont manquants, déplacés, abîmés, ou non conformes. Aujourd'hui il fait des inspections périodiques manuelles avec relevé GPS approximatif.

**P3 — Gestionnaire d'actifs / SIG municipal**
Veut maintenir à jour les bases de données d'inventaire (regards, bornes, panneaux) avec détection automatique des écarts entre le terrain et la base. Veut consommer les résultats via WFS dans son SIG existant (QGIS, ArcGIS).

**P4 — Opérateur de collecte**
Personne (employé municipal, sous-traitant, voire citoyen volontaire dans une v2) qui installe l'iPhone dans son véhicule et roule selon un parcours prescrit ou opportuniste. Doit pouvoir **démarrer/arrêter une session en 2 taps**, sans connaissances techniques.

---

## 3. Architecture haut niveau

```
┌─────────────────────┐        ┌──────────────────────┐       ┌────────────────────┐
│  RoadSense iOS      │  →    │  Ingestion / Storage  │  →   │  Post-processing    │
│  (collecte capteurs)│        │  (S3-like + metadata)│       │  pipeline (workers) │
└─────────────────────┘        └──────────────────────┘       └────────┬───────────┘
                                                                       │
                                                              ┌────────▼───────────┐
                                                              │  PostGIS / Tiles    │
                                                              │  OGC API Features   │
                                                              │  WFS / GeoJSON      │
                                                              └────────┬───────────┘
                                                                       │
                                                              ┌────────▼───────────┐
                                                              │  Visualisation      │
                                                              │  (deck.gl / QGIS /  │
                                                              │   web app review)   │
                                                              └────────────────────┘
```

Trois sous-systèmes :

1. **App iOS** : capture brute, robuste, synchronisée. *Aucune intelligence métier embarquée en v1.*
2. **Pipeline post-traitement** : ingestion → détection → géoréférencement → recalage → publication.
3. **Couche de service / consommation** : standards OGC, web review, intégration SIG.

---

## 4. App iOS — Spécification

### 4.1 Scope v1

**Inclus :**
- Capture multi-capteurs synchronisée pendant une session de conduite
- Gestion thermique et énergétique active
- Persistance fiable, reprise après interruption
- Upload différé des sessions vers le backend (Wi-Fi de préférence)
- Interface minimaliste opérateur (start, stop, état)

**Exclus de la v1 :**
- Inférence ML embarquée
- Visualisation des résultats
- Mode multi-utilisateurs / téléversement en temps réel
- Annotation manuelle pendant la conduite

### 4.2 Capteurs et formats

| Capteur | Source iOS | Fréquence | Format brut | Volume estimé / h |
|---|---|---|---|---|
| Vidéo principale | `AVCaptureSession` (caméra arrière 48MP) | 1080p @ 30fps HEVC | `.mov` | ~9 GB |
| Profondeur LiDAR | `ARKit` `sceneDepth` | 10 Hz (déclassé volontairement) | binaire compressé (`.depth.bin` + manifest) | ~300 MB |
| Confiance LiDAR | `ARKit` `confidenceMap` | 10 Hz | idem | ~75 MB |
| Pose ARKit | `ARFrame.camera.transform` + intrinsics | 60 Hz | JSONL ou Parquet | <5 MB |
| Accéléromètre brut | `CMMotionManager` | 100 Hz | Parquet (colonnes : t, ax, ay, az) | <10 MB |
| Gyroscope brut | `CMMotionManager` | 100 Hz | Parquet | <10 MB |
| Device motion fusionné | `CMDeviceMotion` | 100 Hz | Parquet (attitude, gravity, userAccel) | <30 MB |
| Baromètre | `CMAltimeter` | event-driven (~1 Hz) | Parquet | négligeable |
| GPS | `CLLocationManager` (`bestForNavigation`) | ~1 Hz, sans filtre distance | Parquet (lat, lon, alt, speed, course, hAcc, vAcc) | négligeable |
| Heading magnétique | `CLHeading` | événements | Parquet | négligeable |
| Lumière ambiante | (via ARKit `lightEstimate`) | 60 Hz | Parquet | négligeable |
| État système | `ProcessInfo`, batterie, charge | 1 Hz | JSONL | négligeable |

**Total typique** : ~10-12 GB par heure de roulage.

### 4.3 Synchronisation temporelle

**Référence maître :** horloge `CMClockGetHostTimeClock()`.

- Vidéo : `presentationTimeStamp` des `CMSampleBuffer` → directement dans la base host.
- ARKit / LiDAR : `ARFrame.timestamp` est en `CFTimeInterval` (mêmes secondes que `mach_absolute_time` post-conversion).
- IMU : `CMLogItem.timestamp` en `mach_absolute_time` ; conversion via `mach_timebase_info`.
- GPS : `CLLocation.timestamp` est en `Date` ; offset à mesurer une fois par session avec un échantillon de référence.

**Chaque échantillon enregistré inclut son timestamp dans la base host unifiée** (en plus du timestamp natif du capteur, conservé pour traçabilité).

### 4.4 Calibration (à exécuter une fois par installation véhicule)

L'app inclut un mode **calibration** qui collecte :
- Une mire de calibration intrinsèque (en option ; les intrinsèques ARKit suffisent en pratique pour v1).
- Un trajet en ligne droite à vitesse constante pour estimer la rotation téléphone↔véhicule (le vecteur vitesse GPS fournit l'axe X véhicule, la gravité fournit l'axe Z, on en déduit Y).
- Une mesure manuelle de la hauteur du téléphone (saisie utilisateur) et de l'offset latéral et longitudinal par rapport au centre du véhicule (saisies utilisateur).
- Le résultat est enregistré dans un fichier `vehicle_calibration.json` joint à chaque session.

### 4.5 Gestion thermique

**Surveillance :** `ProcessInfo.thermalState` polled à 1 Hz, et notifications `thermalStateDidChangeNotification`.

**Politique de dégradation :**

| État thermique | Action |
|---|---|
| `.nominal` / `.fair` | Mode normal : 1080p HEVC + LiDAR 10Hz + IMU 100Hz |
| `.serious` | Réduction : vidéo 720p, LiDAR 5Hz ; alerte UI |
| `.critical` | Pause vidéo + LiDAR ; conserver IMU+GPS uniquement ; alerte UI ; reprise auto si retour à `.fair` |

**Mitigations passives :**
- Écran à `brightness = 0.05`, idle timer désactivé pendant la session
- Préférer charge filaire à puissance modérée plutôt que MagSafe
- Afficher dans l'UI la température estimée et conseil ventilation si pertinent
- Documenter dans le manuel utilisateur : éviter soleil direct, support ventilé, ne pas laisser le téléphone seul dans la voiture en plein soleil avant session

### 4.6 Persistance et fiabilité

- Chaque session = un dossier daté avec UUID
- Écriture en streaming, **flush régulier** (pas de buffer entier en RAM)
- En cas de crash ou kill système, les données déjà flushées sont récupérables au prochain démarrage
- Checksum (SHA-256) calculé par fichier en fin de session
- Manifest de session JSON contenant : version app, modèle device, calibration véhicule, intervalles capteurs actifs, événements thermiques, durée, taille totale

### 4.7 Backend modes et permissions

- `Background Modes` : `location`, `audio` (pour maintenir la session active si l'écran s'éteint malgré tout), `bluetooth-central` (réservé v2)
- Permissions demandées : caméra, micro (réservé v2), localisation **always**, motion
- L'app reste **foreground actif** pendant la collecte (caméra requiert foreground). UI visuellement minimale pour économiser la charge écran.

### 4.8 UI minimaliste

Trois écrans :
1. **Accueil** : statut calibration, dernière session, bouton "Nouvelle session"
2. **Session active** : timer, distance parcourue, état thermique, indicateurs visuels par capteur (vert/jaune/rouge), bouton stop. *Pas d'aperçu vidéo en continu* (coût thermique).
3. **Sessions** : liste, taille, statut upload (en attente / en cours / terminé / erreur), bouton "uploader maintenant" (Wi-Fi only configurable)

### 4.9 Upload

- Découpage en segments de N minutes (configurable, défaut 5 min) pour upload résilient
- Multipart resumable upload vers backend (style S3 multipart ou tus.io)
- Wi-Fi par défaut, option cellulaire désactivable
- Les fichiers ne sont supprimés du téléphone qu'**après confirmation backend** + validation checksum

### 4.10 Stack technique iOS

- **Swift 6** (concurrence stricte)
- **AVFoundation** (capture vidéo, audio)
- **ARKit** (LiDAR, pose VIO)
- **CoreMotion** (IMU, baromètre)
- **CoreLocation** (GPS, heading)
- Sérialisation : **Apache Arrow / Parquet** via `swift-arrow` ou écriture binaire custom + conversion serveur. Fallback minimal viable : JSONL gzippé.
- Persistance metadata : SQLite (via GRDB)
- Upload : `URLSession` avec `backgroundSessionConfiguration`

---

## 5. Pipeline de post-traitement — Plan haut niveau

### 5.1 Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     PIPELINE POST-TRAITEMENT                              │
└──────────────────────────────────────────────────────────────────────────┘

[Étape 0]  Ingestion & validation
              │
              ▼
[Étape 1]  Préparation et synchronisation des séries temporelles
              │
              ▼
[Étape 2]  Pose initiale (GPS + IMU + ARKit VIO fusion)
              │
              ▼
[Étape 3]  Détections par image
              ├─→ Amers (panneaux stop, regards, bornes, etc.)
              ├─→ Défauts de chaussée
              └─→ Inventaire signalisation général
              │
              ▼
[Étape 4]  Géoréférencement 3D des détections (LiDAR + projection plan-sol)
              │
              ▼
[Étape 5]  Appariement des amers avec base municipale
              │
              ▼
[Étape 6]  Optimisation de pose globale (PnP + bundle adjustment géoréférencé)
              │
              ▼
[Étape 7]  Re-projection des détections avec pose recalée
              │
              ▼
[Étape 8]  Agrégation multi-passes / multi-trajets, déduplication
              │
              ▼
[Étape 9]  Classification finale, sévérité, contrôle qualité
              │
              ▼
[Étape 10] Publication (PostGIS, OGC API Features, WFS, tuiles vectorielles)
              │
              ▼
[Étape 11] Review humain (optionnel) → boucle d'amélioration ML
```

### 5.2 Détail des étapes

#### Étape 0 — Ingestion & validation

- Réception du dossier de session (multipart upload terminé)
- Vérification des checksums et de la complétude (manifest vs fichiers présents)
- Décompression, conversion des formats binaires custom vers Parquet/Arrow standard
- Création d'un enregistrement `Session` en base avec statut `received`
- File d'attente du job pour la suite du pipeline

#### Étape 1 — Préparation et synchronisation

- Chargement des séries IMU, GPS, baro, lumière en DataFrames Parquet
- Resampling sur grille temporelle commune (typiquement 100 Hz)
- Décodage vidéo : extraction des frames clés (1 frame par 0.5 s suffit pour la plupart des détections, plus dense près des amers)
- Association de chaque frame extraite à un timestamp host de référence
- Lecture des depth maps LiDAR avec leur pose ARKit associée

#### Étape 2 — Pose initiale

Estimation d'une **première trajectoire géoréférencée** en fusionnant :
- GPS comme observation absolue de position (basse fréquence, basse précision)
- IMU comme intégration de mouvement haute fréquence (relative, dérive)
- Pose ARKit comme observation visuelle-inertielle locale (excellente sur 30-60 secondes, dérive ensuite)

Méthode : **EKF (Extended Kalman Filter)** ou **factor graph** (GTSAM) avec :
- État : position monde (UTM Zone 18N pour le Québec sud), vitesse, attitude, biais IMU, transformation ARKit→monde
- Mesures : GPS, vitesse GPS (course), accélération IMU, rotation IMU, contraintes ARKit relatives
- Sortie : trajectoire continue à 100 Hz dans un repère monde projeté

À ce stade, précision typique : **1-3 m** en milieu urbain ouvert. Suffisant pour bootstrap, à raffiner.

#### Étape 3 — Détections par image

Trois familles de détecteurs, indépendants, exécutés sur les frames extraites :

**3a. Détecteur d'amers**
- Modèle objet (YOLO v8/v10, RT-DETR ou DINO fine-tuné) sur classes : `stop_sign`, `manhole_round`, `manhole_rect`, `hydrant`, `traffic_signal`, `light_pole`, etc.
- Sortie : bounding box + classe + score, par frame
- Données d'entraînement : Mapillary Vistas, augmenté avec des données spécifiques Québec si disponibles

**3b. Détecteur de défauts de chaussée**
- Modèle de segmentation (Mask2Former, SAM-finetuné, ou modèle dédié type RDD2022) sur classes : `pothole`, `crack_longitudinal`, `crack_transverse`, `crack_alligator`, `patch`, `rutting`, `faded_marking`
- Sortie : masque de segmentation + classe + score
- Optionnel : régression d'une sévérité (0-1) en bonus

**3c. Inventaire signalisation général**
- Variante du détecteur d'amers, étendue à toutes les classes de panneaux (200+ classes de signalisation québécoise)
- Sortie : bounding box + classe + texte OCR (pour panneaux à message variable)

Tous les résultats sont stockés en table `detections` indexée par session, frame, timestamp.

#### Étape 4 — Géoréférencement 3D des détections

Pour chaque détection :

**Si LiDAR disponible et objet à <5m :**
- Échantillonner la depth map dans la bbox/masque (filtrée par confiance)
- Médiane/moyenne robuste pour obtenir la profondeur
- Reprojection 3D dans le repère caméra ARKit
- Transformation vers repère monde via la pose courante

**Sinon (objet plus loin ou LiDAR indisponible) :**
- Hypothèse plan-sol pour les défauts de chaussée (plan z=0 dans repère véhicule, hauteur caméra connue par calibration)
- Triangulation multi-frames pour les objets verticaux (panneaux) si on les voit sur ≥2 frames consécutives avec parallaxe suffisante

Sortie : chaque détection a maintenant une **position 3D estimée dans le repère monde**, avec une covariance d'incertitude.

#### Étape 5 — Appariement avec base municipale

- Chargement des couches municipales pertinentes (regards d'égout, bornes-fontaines, panneaux stop) depuis le portail OGC ou un cache PostGIS
- Pour chaque amer détecté, recherche du candidat le plus proche dans la base, dans une fenêtre de tolérance (typiquement 5 m, ajusté selon la confiance de la pose initiale)
- Matching par **coût combiné** : distance euclidienne + compatibilité de classe + score de détection
- Validation par RANSAC à l'échelle de la fenêtre temporelle (rejeter les matches incohérents)

Sortie : ensemble d'**associations détection ↔ amer monde**.

#### Étape 6 — Optimisation de pose globale

C'est l'étape critique qui transforme la précision.

**Formulation** : bundle adjustment géoréférencé minimisant :
- Erreur de reprojection des amers (chaque amer monde doit se projeter au bon pixel dans chaque frame où il est observé)
- Erreur de pose lisse (régularisation IMU)
- Erreur GPS (contrainte molle, pour éviter dérive sur sections sans amer)

Solveur : **Ceres Solver** ou **GTSAM** avec variables = poses caméra par frame, observations = correspondances 2D-3D.

Sortie : trajectoire raffinée, précision typique attendue **20-50 cm**.

#### Étape 7 — Re-projection avec pose recalée

Toutes les détections de défauts (étape 3b) et d'objets non-référencés (étape 3c) sont re-géoréférencées avec la nouvelle pose. Précision finale : ~30 cm pour les défauts au sol détectés à courte portée, ~50-100 cm pour la signalisation à distance moyenne.

#### Étape 8 — Agrégation multi-passes

Si le même tronçon a été parcouru plusieurs fois (multi-passages, plusieurs collecteurs) :
- Clustering spatial (DBSCAN sur position monde) pour fusionner les détections du même objet réel
- Pondération par confiance et fraîcheur
- Détection de **changements** (un panneau présent à T1 mais absent à T2 → événement à signaler)

#### Étape 9 — Classification finale et contrôle qualité

- Affectation d'un **niveau de sévérité** aux défauts (basé sur taille apparente, profondeur LiDAR, classe)
- Filtrage des détections de faible confiance ou géométriquement aberrantes
- Génération de **vignettes** (crops d'image pour chaque détection, utiles à la révision humaine)

#### Étape 10 — Publication

- Insertion en PostGIS, schéma type :
  - `road_defects(id, session_id, geom POINT, type, severity, confidence, detected_at, image_thumb_url, ...)`
  - `signage_inventory(id, geom POINT, type, condition, last_seen, ...)`
  - `asset_anomalies(id, geom POINT, asset_type, expected_id, observation_type, ...)` (panneau manquant, déplacé, etc.)
- Exposition via **OGC API Features** (pygeoapi) ou **WFS** (GeoServer ou ton propre `ogc-proxy-poc`)
- Génération de tuiles vectorielles MVT pour visualisation web (deck.gl, MapLibre)

#### Étape 11 — Review humain (boucle d'amélioration)

- Interface web légère affichant les détections de faible confiance ou signalées comme anomalies
- L'opérateur valide, rejette ou corrige
- Les corrections retournent dans le set d'entraînement pour fine-tuning périodique des modèles de détection

### 5.3 Stack technique post-traitement

| Composant | Choix recommandé | Alternative |
|---|---|---|
| Orchestration | Prefect ou Dagster | Airflow |
| Stockage objets | MinIO (auto-hébergé) | S3 |
| BD géospatiale | PostgreSQL + PostGIS | — |
| Service OGC | pygeoapi | GeoServer |
| Vidéo / images | FFmpeg, OpenCV | — |
| ML inference | PyTorch + CUDA, ou ONNX Runtime | TensorRT pour prod |
| Détection objet | Ultralytics YOLO, ou Hugging Face Transformers (DETR) | — |
| Segmentation | Mask2Former, ou SAM2 finetuné | — |
| Bundle adjustment | Ceres Solver (C++) ou GTSAM (Python bindings) | g2o |
| Filtrage Kalman | `filterpy` ou impl. custom GTSAM | — |
| Tuiles vectorielles | Tippecanoe + MapTiler / Martin | pg_tileserv |
| Front review | React/SvelteKit + deck.gl ou MapLibre | — |
| Workers | Python conteneurisé (Docker), GPU dédié pour étape 3 | — |

### 5.4 Performance et scalabilité

Cible v1 : **1 km / 5 min** de traitement total, soit ~12 km/h de débit par worker.

Étapes coûteuses :
- Étape 3 (inférence ML) : ~70 % du temps. Parallélisable par lot de frames, GPU obligatoire.
- Étape 6 (BA) : ~20 %. Parallélisable par fenêtre glissante.
- Reste : I/O et orchestration.

Scalabilité horizontale : sharding par session, étape 3 distribuée sur N workers GPU, étape 6 fait sur un worker CPU dédié par session.

---

## 6. Données de référence et dépendances externes

| Source | Usage | Disponibilité |
|---|---|---|
| Bornes-fontaines Montréal | Amers de recalage | Données ouvertes Montréal |
| Regards d'égout Montréal | Amers de recalage | Données ouvertes Montréal (réseau d'égout) |
| Signalisation Montréal | Amers + base de comparaison | Données ouvertes Montréal |
| Données équivalentes Longueuil | Idem | Portail Longueuil |
| Adresse Québec / GeoBase | Filaire de voirie pour map-matching | Géoindex |
| Mapillary Vistas / RDD2022 | Entraînement détecteurs | Public, licences CC-BY |
| Modèles de signalisation MTQ | Référence visuelle, classes | MTQ — Tome V |

---

## 7. Risques et mitigations

| Risque | Impact | Mitigation |
|---|---|---|
| Throttling thermique en plein soleil | Perte de données | Politique de dégradation graduelle, support ventilé recommandé |
| Précision GPS dégradée canyon urbain | Recalage difficile sans amers | Densité d'amers généralement élevée en centre-ville ; fallback sur dead-reckoning IMU+ARKit VIO |
| Bases municipales obsolètes | Recalage faux | Validation par cohérence multi-amers (RANSAC), détection d'écarts comme feature explicite |
| Perturbations magnétiques voiture | Heading inutilisable | S'appuyer sur GPS course + ARKit, ignorer magnétomètre |
| Volume de données | Coût stockage et upload | Politique de rétention : archivage froid des données brutes après publication, conservation chaude des dérivés |
| Vie privée (visages, plaques) | Conformité Loi 25 | Anonymisation automatique en étape 0 (flou plaques + visages) avant tout stockage long terme ; pas d'export brut hors environnement contrôlé |
| Calibration véhicule changée sans recalibration | Dégradation silencieuse | Détection automatique d'incohérences (vecteur vitesse vs axe X véhicule estimé) → alerte recalibration |
| Drift ARKit sur longues sessions | Pose ARKit inutilisable | Reset périodique ARKit (toutes les ~5 min), rebootstrap à partir GPS+IMU |

---

## 8. Roadmap proposée

**M0 — Spike de faisabilité (2-3 semaines)**
- App iOS minimale : capture vidéo + IMU + GPS + LiDAR sur 30 min
- Vérification thermique sur trajet réel
- Format de données stabilisé

**M1 — Pipeline minimal sans amers (4-6 semaines)**
- Pipeline étapes 0-2 + 3b (défauts uniquement) + 7 (sans recalage par amers) + 10 (publication GeoJSON simple)
- Démo : carte des nids-de-poule sur un quartier test

**M2 — Recalage par amers (4-6 semaines)**
- Étapes 3a + 4 + 5 + 6
- Démo : amélioration mesurable de la précision (mesure d'écart sur amers tenus)

**M3 — Inventaire et anomalies (3-4 semaines)**
- Étape 3c + détection d'écarts
- Démo : panneaux manquants/déplacés sur un secteur

**M4 — Production minimale (4-6 semaines)**
- Étapes 8, 9, 11 (review humain)
- OGC API Features de sortie
- Premier test client municipal

---

## 9. Questions ouvertes / décisions à prendre

- Choix entre **EKF maison** vs **GTSAM factor graph** dès le départ pour l'étape 2.
- Stratégie d'**anonymisation** : sur device ? côté serveur en étape 0 ? avec quel modèle ?
- **Modèle économique** : SaaS municipalité, on-premise, hybride ?
- **Multi-collecteurs / crowdsourcing** dans une v2 : changements UX significatifs.
- **Audio** : finalement utile pour détection acoustique de défauts ? À explorer en parallèle, faible coût.
- Niveau de **certification métrologique** requis pour acceptation par les services techniques municipaux.

---

*Fin du document.*