# PRD — wsmd · Inventaire d'actifs municipaux par vision artificielle géoréférencée

**Version** : 3.0 (consolidé)
**Date** : 2026-04-27
**Statut** : draft, en attente de validation produit
**Auteur** : Julien Riel, avec assistance Claude
**Cible PoC** : Précision Niveau B (0.5–2 m) sur les objets détectés, validée empiriquement à Longueuil/Montréal

> Ce document est la **seule source de vérité produit** pour wsmd. Il remplace toute spec ou plan antérieur. Les spécifications d'implémentation (calibration, code, séquence des tâches) en découlent.

---

## 1. Vision

Construire un **système de cartographie automatisée des actifs municipaux** qui, à partir d'un véhicule en circulation, détecte et géoréférence avec une précision sub-2 mètres l'inventaire d'état des éléments du domaine public : signalisation, mobilier urbain, marquage au sol, état de chaussée, présence de chantiers, graffitis.

À long terme, ce système doit permettre à une municipalité (ou à un partenaire civic-tech) de **maintenir un inventaire vivant et géolocalisé** de son patrimoine viaire, mis à jour à chaque passage d'un véhicule équipé — par opposition aux inventaires manuels actuels, ponctuels et coûteux.

À court terme, l'objectif est de **prouver la faisabilité technique** sur une zone test de quelques kilomètres carrés à Longueuil, avec un matériel grand public et une chaîne logicielle 100 % open source.

---

## 2. Contexte et motivation

### 2.1 Pourquoi maintenant

- Les **modèles de vision artificielle open-source** (YOLOv8/v11, Grounding-DINO, segmentation foundationale) atteignent en 2025 une qualité suffisante pour la signalisation urbaine, sans entraînement custom dans la majorité des cas.
- Les **iPhone récents** (16 Pro et postérieurs) embarquent un GNSS bi-bande L1/L5 et une centrale inertielle de qualité, accessibles directement via `CoreLocation` + `CoreMotion` — un récepteur GNSS dédié n'est plus indispensable au Niveau B.
- Les **standards OGC API Features** sont matures et adoptés par les SIG modernes (QGIS, Esri, géoportails municipaux). Une livraison standard est immédiatement consommable.
- Les **données ouvertes municipales** (Géobase, bornes-fontaines, lampadaires, arbres, abribus, mobilier urbain inventorié) fournissent des **landmarks de recalage** quasi-gratuits qui améliorent matériellement la précision absolue. La disponibilité varie d'une municipalité à l'autre — le PoC s'adapte aux jeux disponibles dans la zone test.

### 2.2 Le problème côté municipalité

Les villes québécoises (Longueuil, Montréal et autres) maintiennent un inventaire de leurs actifs viaires principalement par :
- Relevés de cols bleus à pied ou en tournée — coûteux, ponctuels, vite obsolètes.
- Photos aériennes — précision absolue OK, mais résolution insuffisante pour l'état (marquage effacé, graffiti, chantier actif).
- Signalements citoyens (ex. Montréal-info) — réactif, partiel, biais de couverture.

Aucune de ces approches ne produit un **inventaire dense, fréquent et standardisé**. C'est le créneau qu'attaque wsmd.

### 2.3 Pourquoi un PoC plutôt qu'un produit fini

La chaîne complète (capture → détection → géoréférencement précis → livraison OGC) implique plusieurs maillons techniques **dont aucun n'est trivial à seul**, et dont l'**addition d'erreurs** est l'enjeu central.

Avant d'investir dans :
- Du matériel professionnel (récepteur RTK, ZED stéréo)
- Un fine-tuning dataset Quebecois
- Une intégration municipale (contrats, SLA)

…il faut **mesurer empiriquement** la précision atteignable avec du matériel grand public sur une zone test représentative. C'est le rôle de ce PoC.

### 2.4 Ce que **n'est pas** wsmd à ce stade

- Ce n'est **pas** un produit clé-en-main à vendre à une municipalité. C'est un PoC de faisabilité.
- Ce n'est **pas** un dataset publié sur HuggingFace Hub à cette étape (l'ouverture éventuelle est Phase 2+).
- Ce n'est **pas** une plateforme multi-utilisateurs avec interface validateur. Une UI de validation existera si nécessaire pour qualifier la précision, pas comme produit final.
- Ce n'est **pas** un système temps réel en production. Le pipeline est **post-captation** sur Mac.

---

## 3. Audiences et cas d'usage

### 3.1 Audiences

| Audience | Quoi qu'elle attend | Période d'intérêt |
|---|---|---|
| Équipe interne (1–2 dev) | Prouver la faisabilité technique Niveau B | PoC (8 semaines) |
| Décisionnaires municipaux | Voir une démo cartographique convaincante avec chiffres de précision | Fin de PoC |
| Partenaires civic-tech (ex. CIRANO, Polytechnique, éditeurs SIG) | Échanges scientifiques, recalage croisé, dataset partagé | Phase 2 |
| Citoyens (long terme) | Cartes ouvertes des actifs municipaux | Production, hors PoC |

### 3.2 Cas d'usage cibles

**Cas d'usage 1 — Inventaire de signalisation**
> Un véhicule équipé fait une boucle de 5 km à Longueuil. À l'arrivée, la carte affiche tous les panneaux STOP, CÉDEZ, vitesse, etc. avec leur position à <2 m près. Cliquer sur un panneau ouvre la frame d'origine + classe + score de confiance.

**Cas d'usage 2 — État de chaussée localisé**
> Le même véhicule capture aussi des défauts de chaussée (nids-de-poule, fissures majeures). À l'arrivée, une carte de chaleur des défauts localise les zones les plus dégradées.

**Cas d'usage 3 — Comparaison temporelle**
> Le même trajet est repassé à 1 mois d'écart. Le système flagge les **disparitions** (panneau enlevé/abîmé) et **apparitions** (nouveau chantier, nouveau graffiti).

**Cas d'usage 4 — Couche WFS pour SIG municipal** *(Phase 2)*
> Le service de la voirie consomme la couche `detected_assets` directement dans QGIS via OGC API Features. Les détections sont superposées à leur référentiel SIG existant.

---

## 4. Hypothèse produit

> Avec un **iPhone 16 Pro (master GNSS+IMU)**, une **caméra Brio 4K (calibrée)** et un **MacBook Air**, en utilisant des **modèles CV open-source pré-entraînés** + **multi-frame triangulation** + **recalage sur landmarks municipaux ouverts**, on peut produire un **inventaire géoréférencé d'actifs municipaux à précision Niveau B (0.5–2 m)** sans matériel spécialisé ni récepteur RTK.

Si cette hypothèse est validée, elle débloque une voie pragmatique vers un produit déployable — sans le coût matériel et la complexité d'un système de mobile mapping professionnel.

Si elle est invalidée, on saura **où** se situe le maillon faible (heading, GNSS, triangulation, classification) et on pourra investir de manière ciblée (Niveau C avec RTK ; meilleure caméra ; modèle fine-tuné).

---

## 5. Périmètre du PoC

### 5.1 In-scope

- **Capture** synchronisée vidéo Brio 4K + GNSS/IMU iPhone, 30 min à 2 h par session
- **Détection** des classes suivantes via modèle CV pré-entraîné :
  - Signalisation : panneaux directionnels, STOP, CÉDEZ, limite de vitesse
  - Mobilier urbain : lampadaires, bancs, poubelles, abribus
  - **Repères fixes géoréférencés** disponibles dans les open data municipaux (bornes-fontaines, lampadaires, arbres inventoriés, etc.) — utilisés comme **landmarks de recalage** ; la liste exacte dépend des jeux de données ouverts dans la zone test
  - Cônes / barrières / véhicules de chantier
- **Tracking** multi-frame avec ID stable (ByteTrack ou équivalent)
- **Géoréférencement** par triangulation multi-vues + projection NAD83 MTM zone 8
- **Recalage** optionnel sur bornes-fontaines géoréférencées (Données Québec)
- **Livraison OGC API Features** via pygeoapi sur GeoPackage
- **Validation empirique** sur 15–20 panneaux de référence à position connue

### 5.2 Out-of-scope du PoC (revisités en Phase 2+)

- Détection d'**état** des actifs (panneau penché, graffité, marquage effacé, chaussée dégradée détaillée)
- Interface de **validation manuelle** par un humain
- Annotation **multi-passes** ou **dataset publication** (HF Hub)
- Détection en **temps réel** (le PoC est post-captation)
- Précision **Niveau C** (<20 cm — nécessite RTK)
- **Floutage automatique** plaques/visages
- Capture **multi-véhicule** ou **mobile mapping continu**
- **API d'écriture** (le service OGC est read-only)
- **Authentification** / contrôle d'accès du service OGC

### 5.3 Engagements de précision (PoC seul)

| Métrique | Cible | Méthode de mesure |
|---|---|---|
| Erreur médiane sur panneaux de référence | < 1.5 m | Distance euclidienne détection vs vérité |
| CEP95 (95e centile de l'erreur radiale) | < 2.5 m | Distribution sur 15+ panneaux |
| Taux de détection des panneaux visibles | > 85 % | Comptage manuel sur la même vidéo |
| Répétabilité entre 3 passages | < 1 m | Écart inter-passage de la même détection |

---

## 6. Architecture cible

```
┌──────────────────────────────────────┐    ┌──────────────────────────────────┐
│  iPhone 16 Pro (mount véhicule)     │    │  Logitech Brio 4K               │
│  ───────────────────────────────    │    │  ───────────────────────────    │
│  • App Swift native (SwiftUI)        │    │  • Mount succion pare-brise     │
│  • CoreLocation @ 10 Hz              │    │  • USB 3.0 vers Mac             │
│    (kCLLocationAccuracyBestForNav)   │    │  • 4K @ 30 fps, MJPEG/H.264     │
│  • CoreMotion @ 100 Hz               │    │  • Autofocus + auto-exposure    │
│    (xMagneticNorthZVertical)         │    │    verrouillés                  │
│  • Heading vrai (corrigé déclinaison)│    │                                 │
│  • Server WebSocket Bonjour          │    │                                 │
│  • Logging local CSV de backup       │    │                                 │
└─────────────┬────────────────────────┘    └────────────┬─────────────────────┘
              │ Wi-Fi local (hotspot iPhone               │ USB 3.0
              │  ou réseau partagé)                       │
              ↓                                            ↓
        ┌──────────────────────────────────────────────────────────┐
        │  MacBook Air M-series (24 GB / 1 TB + SSD externe 2 TB) │
        │  ──────────────────────────────────────────────────────  │
        │  Pipeline temps de captation :                           │
        │    • Capture Brio (OpenCV + AVFoundation)                │
        │    • Client WebSocket consommant flux iPhone             │
        │    • Écriture session_dir sur SSD externe                │
        │                                                          │
        │  Pipeline post-captation :                               │
        │    • Synchronisation horaire (NTP partagé)               │
        │    • Détection YOLO + ByteTrack (PyTorch MPS)            │
        │    • Interpolation pose 6-DOF par frame                  │
        │    • Triangulation multi-frame (bundle adjustment)       │
        │    • Reprojection NAD83 MTM zone 8 (EPSG:32188)          │
        │    • Recalage optionnel sur bornes-fontaines             │
        │    • Export GeoPackage (`assets.gpkg`)                   │
        │                                                          │
        │  Livraison :                                             │
        │    • pygeoapi (OGC API Features) sur GeoPackage          │
        │    • Visualisation QGIS via WFS                          │
        └──────────────────────────────────────────────────────────┘
```

### 6.1 Topologie disque

```
data/
  landmarks/                 # cache local des jeux open data municipaux (R-tree-indexé)
    longueuil/
      bornes_fontaines.gpkg
      lampadaires.gpkg
      arbres.gpkg
      ...
    montreal/
      ...
  geobase/                   # cache Géobase (tronçons de rue)
    troncons.geojson

sessions/
  2026-04-27_14-30-00_<uuid>/
    config.yaml              # paramètres caméra, mount véhicule, calibration intrinsèque
    metadata.json            # début/fin, version Git pipeline, conditions météo
    video.mp4                # flux Brio brut (≈11 GB/h en 4K)
    frames.csv               # frame_idx, ts_iso (par frame)
    gnss.jsonl               # samples GNSS au format JSON-Lines (un par ligne)
    imu.jsonl                # samples IMU 100 Hz
    detections.csv           # frame_idx, track_id, class, bbox, conf
    poses.csv                # frame_idx, lat, lon, alt, roll, pitch, yaw (interpolé)
    assets.gpkg              # output final (+ layers de debug, copie des landmarks utilisés)
    validation_report.md     # erreurs mesurées vs vérité terrain
```

---

## 7. Stack technique

| Couche | Choix | Justification |
|---|---|---|
| App mobile | Swift / SwiftUI / CoreLocation / CoreMotion | Accès direct GNSS bi-bande L1/L5 + IMU haute fréquence ; déploiement free-provisioning suffit |
| Communication temps réel | WebSocket sur Bonjour (`_munigps._tcp`) | Découverte automatique sur réseau local, latence faible, indépendant du Wi-Fi externe |
| Capture vidéo | Python + OpenCV (backend AVFoundation) | API stable, contrôle UVC pour verrouiller AF/AE, intégration directe avec le pipeline |
| Détection CV | Ultralytics YOLOv8 ou v11 + ByteTrack | Pré-entraîné COCO + extensions traffic-sign disponibles ; bonne perf MPS Apple Silicon |
| Inférence | PyTorch MPS | Pas de CUDA dispo sur Mac ; MPS donne performance correcte pour 4K @ 1–2 fps de traitement |
| Géoréférencement | OpenCV + pymap3d + scipy.optimize.least_squares | Standard mature, bundle adjustment N-vues simple à implémenter |
| Reprojection cartographique | pyproj | EPSG:32188 (NAD83 MTM zone 8) pour cohérence avec les SIG municipaux du Québec |
| Stockage | GeoPackage (SQLite) | Format OGC standard, lisible par QGIS, ArcGIS, GDAL ; auto-suffisant |
| Service web | pygeoapi (Python) | Implémentation OGC API Features de référence, configuration YAML, pas de stack web lourde |
| Visualisation | QGIS | Standard SIG ouvert, consommation OGC native, scriptable Python |
| Versioning code | Git | Standard |
| Versioning data | DVC ou Git LFS (à arbitrer) | Vidéos brutes ne vont pas sur Git |
| Environnement Python | venv + Python 3.12+ | Pas de docker côté Mac, ferré sur l'OS |

---

## 8. Pipeline de capture

### 8.1 App iOS — exigences fonctionnelles

L'app iPhone est le **maître temporel et spatial** du système. Elle doit :

1. Logger en continu **GNSS** à la fréquence native (typiquement 1 Hz, parfois 10 Hz) avec `kCLLocationAccuracyBestForNavigation`.
2. Logger **IMU à 100 Hz** via `CMDeviceMotion` en mode `xMagneticNorthZVertical` (orientation absolue avec correction nord magnétique).
3. **Corriger la déclinaison magnétique** (≈14° W à Montréal/Longueuil) pour obtenir le heading vrai.
4. **Persister localement** les flux dans des CSV (sécurité en cas de coupure Wi-Fi).
5. **Diffuser en temps réel** les samples via WebSocket pour qu'un consommateur (le Mac) puisse les enregistrer en parallèle.
6. Demander les permissions : `NSLocationAlwaysAndWhenInUseUsageDescription`, `NSMotionUsageDescription`, `NSLocalNetworkUsageDescription`, capability "Location updates" en background.
7. Déploiement : free Apple ID via Xcode 16+, profil de confiance accepté manuellement sur le device.

**Note** : l'iPhone 16 Pro est confirmé bi-bande L1/L5 mais Apple n'expose pas l'indicateur de bandes utilisées via API publique. La précision empirique se mesure par `horizontalAccuracy`.

### 8.2 App Mac — pipeline de capture

Script Python `wsmd capture` (à concevoir) qui en parallèle :

- Ouvre le flux Brio via OpenCV avec contrôles UVC (verrouillage AF/AE, FOV, exposition manuelle).
- Se connecte au WebSocket Bonjour de l'iPhone.
- Écrit la vidéo H.264/MP4 dans `session_dir/video.mp4` à 30 fps.
- Logge un timestamp ISO 8601 par frame dans `frames.csv`.
- Logge les samples GNSS et IMU en JSON-Lines.
- Affiche un HUD minimal : compteur de frames, lat/lon courante, taux GNSS, désynchro estimée.

### 8.3 Synchronisation temporelle

**Pour Niveau B**, NTP partagé sur les deux devices suffit (résiduel attendu : 10–50 ms).

- Mac : `sudo sntp -sS time.apple.com` au début de chaque session (à scripter).
- iPhone : Réglage automatique de l'heure activé (NTP géré par iOS).
- À documenter dans le `metadata.json` : delta horaire mesuré en début et fin de session.

Pour aller plus loin (futur Niveau C) : ping/pong WebSocket initial pour mesurer `clock_offset_ms` exact et compenser logiciellement.

### 8.4 Calibration

**Calibration intrinsèque caméra** : à faire **une fois** avant tout usage opérationnel.
- Damier d'échiquier 10×7, taille 25 mm.
- 25–40 images variées en 4K, **AF verrouillé**.
- OpenCV `calibrateCamera` → `K` (matrice intrinsèque) + `D` (distorsion).
- Critère d'acceptation : erreur de reprojection RMS < 0.5 pixel.
- Sauvegarde : `config/brio_calibration.yaml`.

**Calibration extrinsèque caméra ↔ antenne** : à faire **à chaque réinstallation du mount**.
- Mesure manuelle au ruban : position (X,Y,Z) de la caméra dans le repère véhicule (origine = antenne iPhone).
- Orientation approximative (typiquement pitch léger vers le bas, ~5°).
- Encodé dans `session_dir/config.yaml`.

---

## 9. Pipeline de détection

### 9.1 Modèle initial

- Démarrer avec un **YOLOv8/v11 pré-entraîné COCO** pour valider la pipeline.
- Pour la signalisation, intégrer un modèle **fine-tuné Mapillary Traffic Sign Dataset** (disponible sur Roboflow / HF) — couvre la majorité de la signalisation nord-américaine.
- **Classes de landmarks** (utilisées pour le recalage, voir §10.6) : à mapper sur les classes COCO disponibles selon les jeux open data — `fire hydrant`, `bench`, `traffic light`, `stop sign` (s'il existe un inventaire municipal), `tree` (via segmentation si disponible).
- Mobilier urbain non-landmark : `bench`, `traffic light` non-inventoriés, etc.

### 9.2 Tracking

- Tracker **ByteTrack** (intégré Ultralytics) avec `persist=True`, `conf=0.4`, `tracker='bytetrack.yaml'`.
- Conservation : tous les hits d'un même `track_id`, pour exploitation en triangulation Phase 4.
- Filtrage post-tracking : éliminer les tracks de < 3 frames consécutives (bruit).

### 9.3 Sortie

CSV `detections.csv` :

```
frame_idx,track_id,class_name,x1,y1,x2,y2,confidence
1542,17,stop_sign,1234.5,567.2,1389.1,725.8,0.92
```

---

## 10. Pipeline de géoréférencement (Niveau B)

C'est le **cœur technique** du PoC. Décomposé en sous-étapes claires :

### 10.1 Interpolation pose 6-DOF par frame

Pour chaque timestamp de frame :
- **Position** : interpolation linéaire entre les 2 fixes GNSS encadrants.
- **Orientation** : interpolation SLERP des quaternions IMU à 100 Hz.
- Output : `poses.csv` avec `(frame_idx, lat, lon, alt, roll, pitch, yaw)` par frame.

### 10.2 Stratégie de fusion GNSS+IMU

**Pour Niveau B**, l'orientation déjà fusionnée par iOS via `CMDeviceMotion` en mode `xMagneticNorthZVertical` est suffisante. À évaluer empiriquement.

Si la précision angulaire mesurée est insuffisante, basculer vers un **EKF léger** (position GNSS comme observation absolue, prédiction IMU à 100 Hz).

### 10.3 Triangulation multi-vues par track

Pour chaque `track_id` :
1. Collecter toutes les frames où il est visible (typiquement 5–30 frames pour un objet à 30+ m de distance).
2. Pour chaque frame : `(u, v)` centre de bbox + pose caméra 6-DOF.
3. **Bundle adjustment** : minimiser la somme des résidus de reprojection pour trouver le point 3D dans le repère monde.
4. `scipy.optimize.least_squares` est suffisant pour ce volume.
5. Initialisation : milieu du segment de trajectoire à 30 m devant.

```python
# Esquisse de la fonction de coût
def reproject_residuals(point_3d, observations, poses, K):
    residuals = []
    for (u, v), pose in zip(observations, poses):
        p_cam = pose.world_to_camera @ np.append(point_3d, 1)
        u_proj = K[0,0] * p_cam[0]/p_cam[2] + K[0,2]
        v_proj = K[1,1] * p_cam[1]/p_cam[2] + K[1,2]
        residuals.extend([u - u_proj, v - v_proj])
    return residuals
```

### 10.4 Fallback pour tracks courts

Pour les tracks < 3 frames (objet vu fugacement) :
- Estimation **mono-vue par hauteur connue** : `distance_m ≈ f * H_real / H_pixels`.
- Hauteurs de référence par classe (panneau STOP : 750 mm, lampadaire : ~9 m, etc.).
- Précision dégradée 10–25 % — acceptable comme fallback, à flagger dans la sortie (`precision_class = 'mono_estimate'`).

### 10.5 Reprojection cartographique

```python
# ENU local → WGS84 → NAD83 MTM zone 8
import pymap3d as pm
import pyproj

lat, lon, alt = pm.enu2geodetic(e, n, u, lat0, lon0, alt0)
transformer = pyproj.Transformer.from_crs("EPSG:4326", "EPSG:32188", always_xy=True)
x_mtm, y_mtm = transformer.transform(lon, lat)
```

EPSG:32188 = NAD83 / MTM zone 8 — projection de référence pour Montréal/Longueuil.

### 10.6 Recalage sur landmarks géoréférencés

Optionnel mais utile pour éliminer l'erreur systématique GNSS (multipath, bias) sans coût matériel additionnel.

**Principe** : tout actif fixe dont la position est déjà connue dans un jeu de données ouvert municipal peut servir de landmark. Lorsque le pipeline détecte un tel objet par YOLO et qu'il existe un candidat connu à proximité du GPS prior, on dispose d'une mesure `(observation_GPS, vérité_landmark)` exploitable comme correction.

**Sources de landmarks** (dépend de la municipalité de la zone test) :

| Type | Source typique | Densité | Notes |
|---|---|---|---|
| Bornes-fontaines | Données Québec, Ville de Montréal | Élevée | Bonne couverture, position fiable |
| Lampadaires | Hydro-Québec, Données Québec | Très élevée | Très fréquents, recalage dense |
| Arbres inventoriés | Ville de Montréal (arbres publics) | Élevée en zones urbaines | Détection CV par segmentation possible |
| Abribus / mobilier RTL/STM/STL | Catalogues transit ouverts | Modérée | Position fiable, spécifique |
| Panneaux STOP inventoriés | Catalogues municipaux SIG (variable) | Variable | Inégal ; à vérifier au cas par cas |

**Stockage local** :
- Avant chaque session, télécharger les jeux pertinents depuis les portails open data au format GeoJSON ou GeoPackage.
- Persister sur le MacBook dans `data/landmarks/<municipality>/` (par ex. `data/landmarks/longueuil/lampadaires.gpkg`).
- Charger en mémoire dans une **table commune** `landmarks` au sein du `assets.gpkg` du pipeline, avec un attribut `landmark_class` (= `fire_hydrant`, `lamppost`, `tree`, ...) et un identifiant municipal d'origine.

**Accès — R-tree** :
- Utiliser un index R-tree (intégré à GeoPackage / SQLite) pour requêter rapidement les candidats dans un rayon de ~10 m autour du GPS prior d'une frame.
- Coût d'une requête : sub-milliseconde, négligeable même sur des milliers de frames.

**Algorithme de recalage** :
1. Pour chaque détection CV de classe correspondante (ex. `fire_hydrant`), interroger le R-tree des landmarks de cette classe dans un rayon adapté autour du GPS prior.
2. Si un seul candidat retourné → match (et calcul du delta `gps - landmark`).
3. Si plusieurs candidats → désambiguïsation par projection bbox-camera ou rejet (Algo A pilot : on garde le plus proche, Algo B futur : on triangule).
4. Calculer un **delta médian** sur l'ensemble des matches de la session (lat, lon).
5. Appliquer le delta à toutes les détections (et optionnellement aux poses caméra) de la session.

**Robustesse** : la médiane absorbe les mauvais matches (outliers). Pour Niveau B, c'est suffisant. Niveau C bénéficierait d'un Kalman temporel.

### 10.7 Déduplication

- Même classe + distance < 2 m → même objet (probable double détection sur passages différents).
- Fusion : moyenne pondérée par confiance × nombre d'observations.

---

## 11. Modèle de données et livraison OGC

### 11.1 Schéma `detected_assets`

```sql
CREATE TABLE detected_assets (
  asset_id          INTEGER PRIMARY KEY,
  class             TEXT NOT NULL,            -- ex. 'stop_sign', 'lamppost'
  geom              POINT NOT NULL,            -- WGS84 (EPSG:4326)
  geom_mtm          POINT,                     -- NAD83 MTM zone 8 pour SIG locaux
  confidence        REAL NOT NULL,             -- score moyen sur le track
  precision_m       REAL,                      -- estimation incertitude (m)
  precision_class   TEXT,                      -- 'triangulated' | 'mono_estimate' | 'recalibrated'
  session_id        TEXT NOT NULL,             -- UUID de la session
  detected_at       TIMESTAMP NOT NULL,        -- moment de la détection
  n_observations    INTEGER,                   -- nombre de frames où l'objet a été vu
  representative_frame_idx  INTEGER,           -- index de la frame "vitrine"
  representative_bbox_json  TEXT,              -- bbox sur cette frame (pour image preview)
  attributes_json   TEXT                       -- extensions futures
);
```

Indices : sur `(class)`, `(session_id)`, R-tree spatial.

### 11.2 Configuration pygeoapi

```yaml
server:
  bind: { host: 0.0.0.0, port: 5000 }
  url: http://localhost:5000

resources:
  detected_assets:
    type: collection
    title: Détections d'actifs municipaux (wsmd PoC)
    description: Inventaire automatisé par vision artificielle géoréférencée
    keywords: [signalisation, mobilier, municipal, longueuil]
    crs: [http://www.opengis.net/def/crs/EPSG/0/4326]
    providers:
      - type: feature
        name: GeoPackage
        data: /path/to/assets.gpkg
        id_field: asset_id
        table: detected_assets
```

### 11.3 Endpoints exposés

- `GET /collections/detected_assets/items` — GeoJSON FeatureCollection
- `GET /collections/detected_assets/items?bbox=...&class=stop_sign` — filtrage
- `GET /collections/detected_assets/items/{asset_id}` — feature individuelle
- `GET /collections/detected_assets/items?f=html` — UI HTML simple intégrée à pygeoapi

### 11.4 Consommation cible

- **QGIS** : ajout d'une couche WFS / OGC API Features → `http://localhost:5000/collections/detected_assets/items`. Consommation native.
- **Curl / scripts** : `curl http://localhost:5000/collections/detected_assets/items?bbox=-73.6,45.5,-73.5,45.6 > export.geojson`.
- **Browser** : interface HTML générée automatiquement par pygeoapi.

---

## 12. Validation de précision

### 12.1 Protocole

1. Sélectionner une **zone test** de 2–5 km de rues à Longueuil (boucle représentative).
2. **Identifier 15–20 panneaux de référence** clairement visibles dans la zone.
3. **Mesurer leur position vérité** par l'une des méthodes suivantes :
   - **Préférée** : location d'un récepteur RTK (Cansel, ~200 $/jour) — précision cm.
   - **Alternative** : relevé manuel sur cartographie ouverte haute résolution (vérifier la précision de la source).
   - **Dégradée** : Google Earth Pro très haute résolution avec mesure manuelle (précision ~50 cm).
4. **Capturer le trajet 3 fois** dans la même journée à des moments différents (matin / midi / fin de journée → conditions d'éclairage variées).
5. **Exécuter la pipeline** sur chacune des 3 captures.
6. **Pour chaque panneau de référence**, calculer :
   - Erreur en mètres (distance euclidienne entre détection et vérité).
   - Identification du panneau dans la sortie (auto si possible, manuel sinon).
7. **Produire un rapport** `validation_report.md` :
   - Histogramme des erreurs.
   - Erreur médiane, moyenne, 95e centile.
   - Comparaison entre passages (répétabilité).
   - Cas problématiques identifiés (occlusions, distance, conditions).
   - Recommandation : Niveau B atteint ? Si non, où sont les pertes ?

### 12.2 Critères de succès

| Métrique | Cible Niveau B |
|---|---|
| Erreur médiane | < 1.5 m |
| CEP95 | < 2.5 m |
| Taux de détection | > 85 % des panneaux visibles |
| Répétabilité inter-passage | < 1 m |

### 12.3 Causes d'erreur attendues et investigation

Si Niveau B n'est pas atteint, attribuer l'erreur à l'un des contributeurs :

| Source | Diagnostic |
|---|---|
| GNSS bruité | Mesurer `horizontalAccuracy` moyenne par session |
| Heading imprécis | Comparer heading IMU à course GNSS sur ligne droite |
| Calibration intrinsèque | Vérifier RMS reprojection initial < 0.5 px |
| Calibration extrinsèque | Vérifier mesures de mount au ruban (incertitude ±5 cm) |
| Triangulation N-vues | Inspecter résidus de reprojection par track |
| Nombre d'observations par track | Histogramme — un track de 3 frames est moins fiable que 20 |
| Synchro horaire | Mesurer drift via flash de calibration |

---

## 13. Plan phasé (8 semaines)

| Phase | Semaine | Livrable | Effort (jp) |
|---|---|---|---|
| 0 — Setup environnement | 1 | venv Python opérationnel, YOLO MPS validé, calibration Brio | 1–2 |
| 1 — App iOS de logging | 2 | App SwiftUI déployée, GNSS+IMU loggés en CSV + WebSocket | 3–5 |
| 2 — Pipeline capture Mac | 3 | `wsmd capture` produit un session_dir complet | 3–4 |
| 3 — Détection + tracking | 4 | YOLO + ByteTrack sur vidéo, `detections.csv` produit | 2–3 |
| 4 — Géoréférencement | 5–6 | Triangulation N-vues, reprojection MTM, recalage landmarks | 5–8 |
| 5 — Livraison OGC | 7 | pygeoapi servant `detected_assets`, consommé par QGIS | 1–2 |
| 6 — Validation | 8 | Rapport de précision sur zone test | 2–3 |
| **Total** |  |  | **17–27 jours-personne** |

À temps partiel (soirs/weekends) : ≈ 6–8 semaines calendaires pour 1 personne, ≈ 3–4 semaines pour 2 personnes en parallèle.

---

## 14. Risques et points de vigilance

### 14.1 Risques techniques

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| GNSS iPhone insuffisant en zone urbaine dense (multipath) | Moyenne | Niveau B raté | Recalage landmarks + plan B = ZED-F9P RTK |
| Heading imprécis → triangulation diverge | Moyenne | Niveau B raté | EKF léger ; ou passage dual-antenna en Phase 2 |
| Désynchro temporelle > 100 ms en certains samples | Faible | Erreur ~30–60 cm @ 50 km/h | Logger drift NTP, compenser logiciellement |
| Modèle pré-entraîné insuffisant sur signalisation QC | Moyenne | Taux détection < 85 % | Fine-tuning Mapillary subset QC, fallback manuel |
| Volumes de données ingérables sur SSD 2 TB | Faible | Sessions limitées | Compression H.265, downsampling 1080p si Brio supporte |
| Vie privée (plaques, visages) fuite hors PoC | Moyenne | Risque légal | Stockage local sécurisé, pas de diffusion ; floutage avant tout partage |

### 14.2 Risques projet

- **Sous-estimation du temps de calibration** : à minimiser, c'est facile à reporter et à rallonger. Bloquer la Phase 0 si calibration RMS > 0.5 px.
- **Dérive de scope** : ne pas ajouter détection d'état, UI validateur, dataset publication tant que Niveau B n'est pas démontré.
- **Disponibilité iPhone 16 Pro** : matériel personnel, à confirmer.
- **Météo défavorable au planning** : prévoir buffer pour repasser en cas de pluie/neige rendant les captures inexploitables.

### 14.3 Conformité & vie privée (loi 25 Québec)

- Les vidéos brutes contiennent **plaques d'immatriculation** et **visages** = données personnelles.
- **Pour le PoC** : stockage local sur SSD chiffré, pas de partage externe, pas de upload cloud.
- **Si le projet va plus loin** : floutage automatique obligatoire avant tout partage (modèles CV existants : ANPR pour plaques, MTCNN ou similaire pour visages).
- Documenter dans `metadata.json` la nature des données et la rétention.

---

## 15. Évolutions Phase 2 envisageables

À considérer **uniquement si le PoC atteint Niveau B** et que la motivation produit reste.

### 15.1 Précision Niveau C (<20 cm)

- Ajout d'un récepteur **u-blox ZED-F9P** (ArduSimple simpleRTK2B, ~600 $).
- Abonnement **NTRIP** (SmartNet, CanNet ou Emlid Caster, ~50 $/mois).
- **Dual-antenna** pour heading précis (deux F9P ou GNSS-INS Septentrio low-end).
- Caméra **stéréo passive** (deux Arducam IMX678) pour redondance triangulation.

### 15.2 Élargissement de la détection

- **Fine-tuning** sur signalisation québécoise (MUTCD-Québec, panneaux bilingues).
- **Détection d'état** : panneau penché, graffité, effacé, marquage usé. Modèles disponibles + entraînement custom léger.
- **Segmentation chaussée** : faïençage, nids-de-poule, fissures (modèles type SAM ou Cityscapes-tuned).
- **Détection de signalisation temporaire de chantier**.

### 15.3 Mode opérationnel municipal

- **Mobile mapping continu** : équipement permanent sur véhicules municipaux (cols bleus, autobus, balayeuses).
- **Détection de changements** entre passages : disparitions, apparitions, déplacements.
- **API d'écriture** authentifiée (le service municipal valide ou rejette les détections automatiques).
- **Dashboard de supervision** des sessions, avec qualité de chaque passage.

### 15.4 Ouverture de la donnée

- Publication d'un **dataset annoté Quebecois** sur HuggingFace Hub sous CC-BY-NC.
- Format **HF Datasets canonique** + sidecars YOLO / COCO / GeoJSON / GPKG.
- DATASHEET.md selon Gebru et al. 2018, croissant.json pour les métadonnées ML.
- Citation académique (CITATION.cff) si publication scientifique.

### 15.5 Cloud et passage à l'échelle

- Pipeline déchargé sur un **serveur Linux + GPU NVIDIA** (post-captation).
- **Stockage objet** (S3, B2, ou bucket institutionnel) pour vidéos brutes archivées.
- **CI/CD** sur les modèles : versioning des poids, métriques de régression entre versions.

### 15.6 Interface validateur (si nécessaire pour qualité produit)

- UI desktop **SvelteKit** servie par le même backend pygeoapi (extension custom).
- Workflow : galerie de frames priorisées → édition multi-bbox → image-level → soumission.
- Persistance IndexedDB côté client + flush par batch.
- Cible : 25 sec / frame médiane.

---

## 16. Glossaire

| Terme | Définition |
|---|---|
| **Niveau B** | Précision géométrique 0.5–2 m sur les objets détectés (vs Niveau A ~5 m, Niveau C <20 cm) |
| **CEP95** | 95e centile de l'erreur radiale (Circular Error Probable 95 %) |
| **GNSS** | Global Navigation Satellite Systems (GPS, GLONASS, Galileo, BeiDou) |
| **L1/L5** | Bandes de fréquences GNSS — bi-bande L1+L5 mitige la ionosphère et améliore la précision |
| **IMU** | Inertial Measurement Unit (accéléromètres + gyroscopes + magnétomètre) |
| **6-DOF** | 6 degrés de liberté : position (3) + orientation (3) |
| **SLERP** | Spherical Linear Interpolation, pour interpoler des quaternions |
| **Bundle adjustment** | Optimisation conjointe de positions 3D et poses caméra par minimisation de résidus de reprojection |
| **OGC API Features** | Standard Open Geospatial Consortium pour la publication de features géographiques en REST/JSON |
| **NAD83 MTM zone 8** | Système de coordonnées projeté pour le sud du Québec, EPSG:32188 |
| **NTRIP** | Networked Transport of RTCM via Internet Protocol — distribution des corrections RTK |
| **RTK** | Real-Time Kinematic — précision centimétrique avec station de base |
| **Free provisioning** | Déploiement Xcode avec Apple ID gratuit (vs Apple Developer Program payant) |
| **MPS** | Metal Performance Shaders, accélérateur Apple Silicon pour PyTorch |
| **wsmd** | Worksight Make Dataset (nom de code interne) |
| **Landmark (au sens wsmd)** | Tout actif fixe géoréférencé connu issu d'un open data municipal (borne-fontaine, lampadaire, arbre, etc.), utilisé pour recaler les détections CV |
| **R-tree** | Index spatial hiérarchique pour requêtes "candidats dans une bbox", supporté nativement par SQLite/GeoPackage |

---

## 17. Décisions encore ouvertes (à valider)

- [ ] **Nom du produit** : conserver "wsmd" comme nom interne ? Choisir un nom externe différent pour Phase 2 ?
- [ ] **Zone test PoC** : préciser les rues de Longueuil — pertinence vs Montréal pour les landmarks (Données Québec).
- [ ] **Mesure de vérité terrain** : RTK loué ou cartographie ouverte ? Trade-off coût vs précision absolue.
- [ ] **Versioning data** : DVC (intégré Git) ou Git LFS ou bucket externe ?
- [ ] **Licence du code** : MIT, Apache 2, GPL ? Impact sur réutilisation future par tiers.
- [ ] **Modèle YOLO retenu** : v8n (rapide), v8s (équilibré), v8m (précis) ? À benchmarker en Phase 0.
- [ ] **Passage à 2 personnes en dev** : utile pour paralléliser app iOS + pipeline Python ?

---

*Fin du document.*
