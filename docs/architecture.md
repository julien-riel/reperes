# Architecture technique — Repères (v1 fail-fast)

**Date** : 2026-04-29
**Statut** : draft, dérive du PRD `docs/prd.md`
**Public** : développeurs, intervenants techniques, futurs contributeurs
**Scope** : v1 PaveAudit visuel, Montréal seulement, monolithique séquentiel, 100 % local

> Ce document décrit le **COMMENT** du PoC v1. Le **QUOI** et le **POURQUOI** vivent dans `docs/prd.md`. Toute divergence entre les deux doit être résolue en faveur du PRD ; ce document suit les décisions produit, il ne les fixe pas.
>
> Les éléments différés Phase 2 (ChantierWatch, branche inertielle, parallélisation, plugin entry points, pygeoapi, rapport PDF, Géobase officielle, UI annotation custom) sont mentionnés en bas de document à titre de roadmap, pas de spec implémentable.

---

## 1. Vue d'ensemble

```
┌────────────────────────────────────────────────────┐
│  iPhone 16 Pro (mount pare-brise) — DEVICE UNIQUE │
│  ────────────────────────────────────────────────  │
│  App SwiftUI native (minimum)                       │
│    • AVCaptureSession 4K H.265 (AF/AE/WB locked)   │
│    • Lentille builtInWideAngleCamera PINNED        │
│    • CoreLocation (GNSS @ 1–10 Hz)                  │
│    • CoreMotion (IMU 100 Hz, x-magnetic-north)     │
│    • CLHeading (cap vrai, déclinaison corrigée)    │
│    • Mount-check simple ±10° pitch (alerte)         │
│    • Persistance locale → session bundle           │
└──────────────────────┬─────────────────────────────┘
                       │ Transfert post-session
                       │ (AirDrop / USB / SMB)
                       ↓
┌────────────────────────────────────────────────────┐
│  MacBook M-series — pipeline post-captation        │
│  Aucun service externe, aucun Docker, aucun pool   │
│  ────────────────────────────────────────────────  │
│                                                    │
│  reperes ingest <session>      ← validation        │
│  reperes preannotate <session> --prompts <yaml>    │
│  reperes import-labels <session> --from <file>     │
│  reperes train --module pave_audit --dataset <dir> │
│  reperes evaluate <session>    ← pipeline complet  │
│                                                    │
│  ╔════════════════════════════════════════════╗    │
│  ║  Pipeline séquentiel monolithique          ║    │
│  ║  (un seul processus Python)                ║    │
│  ║                                            ║    │
│  ║  1. Extract frames @ 1-2 fps               ║    │
│  ║  2. Interpolate pose GNSS+heading per frame║    │
│  ║  3. Load OSM R-tree in memory              ║    │
│  ║  4. YOLO inference (PaveAudit)             ║    │
│  ║  5. Spatio-temporal dedup                  ║    │
│  ║  6. Snap-to-road via R-tree                ║    │
│  ║  7. Aggregate by segment, compute IES      ║    │
│  ║  8. Write GeoPackage (assets.gpkg)         ║    │
│  ║  9. Write validation_report.md             ║    │
│  ╚════════════════════════════════════════════╝    │
│                                                    │
│  Livraison v1 : assets.gpkg lisible QGIS/ArcGIS    │
└────────────────────────────────────────────────────┘
```

**Principes clés v1** :

- *Aucun service externe, aucun Docker, aucun ProcessPoolExecutor.* Un seul processus Python, imports directs, code dans un seul package `reperes/`.
- *Captation produit un bundle de session standard.* L'IMU 100 Hz est loggé mais **non exploité v1** (matière première Phase 2).
- *Snap-to-road via R-tree spatial sur OSM Québec* chargé en mémoire au démarrage (Géobase officielle reportée Phase 2).
- *Écriture directe dans `assets.gpkg`*, pas de fichiers intermédiaires par module (un seul module en v1).
- *Pas de plugin system entry points en v1.* Le code de PaveAudit vit dans `reperes/pave_audit.py`. Un plugin system par entry points reviendra en v2 si > 3 modules deviennent nécessaires.

---

## 2. Topologie disque

```
data/
  road_network/                      # référentiel routier pour snap-to-road
    osm_quebec.gpkg                  # OSM Quebec (extrait Geofabrik, conversion ogr2ogr)
  models/                            # poids fine-tunés versionnés
    pave_audit_v1.pt
    pave_audit_v2.pt
    MANIFEST.yaml                    # mapping version ↔ commit Git ↔ dataset ↔ métriques
  prompts/                           # versionnés Git
    pave_audit.yaml
  datasets/
    qc_pave_v1/                      # gold labels exportés Label Studio (YOLO/COCO)
      images/
      labels/
      index.yaml

sessions/
  2026-04-27_14-30-00_<uuid>/
    manifest.json                    # device, version app, calibration, lentille
    video.mp4                        # 4K H.265, AF/AE/WB verrouillés, lentille pinned
    gnss.jsonl                       # samples GNSS
    imu.jsonl                        # samples IMU 100 Hz (loggé mais non exploité v1)
    heading.jsonl                    # samples cap vrai
    frames/                          # extraites en Phase 2 du pipeline (ffmpeg)
      000000.jpg
      000001.jpg
      ...
    pose_per_frame.parquet           # produit par interpolation pose
    assets.gpkg                      # output GeoPackage (PaveAudit visuel uniquement v1)
    validation_report.md             # résumé numérique de la session
    logs/                            # logs structurés JSON par invocation CLI
      ingest_2026-04-27T15-12-00.jsonl
      evaluate_2026-04-27T15-30-00.jsonl

validation/
  reference_photos/                  # photos MTQ par classe / sévérité
  rubrique_ies_mtq.md                # rubrique de scoring committée Git (cf. PRD §9.2)
  scoring_2026-XX-XX.csv             # scoring auteur à l'aveugle, horodaté Git
```

---

## 3. Codebase

### 3.1 Layout

```
reperes/                       ← package Python unique v1
  __init__.py
  cli.py                       ← Click CLI : reperes ingest/preannotate/...
  ingest.py                    ← validation bundle session
  frames.py                    ← extraction ffmpeg, index parquet
  pose.py                      ← interpolation pose GNSS+heading par frame
  road.py                      ← chargement OSM, R-tree, snap-to-road
  pave_audit.py                ← module PaveAudit visuel (inférence + IES)
  preannotate.py               ← Grounding DINO → JSONL Label Studio
  import_labels.py             ← import gold labels Label Studio → GeoPackage
  train.py                     ← Ultralytics YOLO fine-tune
  validation.py                ← κ_pipeline, κ_self, mAP, RMSE
  storage.py                   ← schémas et écriture GeoPackage
  logging.py                   ← logger contextuel JSON (session_id)

tests/
  fixtures/
    sample_session/            ← mini-bundle session committable
  test_ingest.py
  test_pave_audit.py
  ...

apps/                          ← non utilisé en v1 (UI annotation custom Phase 2)

scripts/
  setup_road_network.sh        ← téléchargement + conversion OSM
```

Pas de packaging multi-package, pas d'entry points, pas de découpage en `reperes_core` / `reperes_pave_audit` / etc. Tout dans un seul `pyproject.toml`. Le découpage modulaire reviendra en v2 si nécessaire.

### 3.2 Pipeline d'évaluation

`reperes evaluate <session>` exécute le pipeline complet en séquentiel dans un seul processus :

```python
def evaluate(session_path: Path) -> EvaluationResult:
    session = load_session(session_path)            # ingest + manifest validation
    frames_index = extract_frames(session)          # ffmpeg → frames/*.jpg + index
    pose = interpolate_pose(session, frames_index)  # GNSS+heading → parquet
    road_index = load_osm_rtree("data/road_network/osm_quebec.gpkg")
    
    detections = run_yolo_inference(frames_index, model_path="models/pave_audit_v1.pt")
    detections = dedupe_spatio_temporal(detections, pose, radius_m=3, window_s=5)
    detections = snap_to_road(detections, pose, road_index)
    segments = aggregate_by_segment(detections, road_index)
    segments = compute_ies_visual(segments)
    
    write_geopackage(session.assets_gpkg, segments, detections)
    write_validation_report(session, segments, detections)
    return EvaluationResult(...)
```

Aucune signature `Evaluator` Protocol, aucune découverte d'entry points. Le code est appelé en imports directs.

### 3.3 CLI

```bash
reperes ingest <session>                                # validation bundle
reperes preannotate <session> --prompts prompts/pave_audit.yaml
                                                        # Grounding DINO → JSONL Label Studio
reperes import-labels <session> --from labels.jsonl     # import retour Label Studio → GPKG
reperes train --module pave_audit --dataset datasets/qc_pave_v1/
                                                        # fine-tune YOLO
reperes evaluate <session>                              # pipeline complet
reperes session purge <session>                         # retire mp4+jsonl, garde annotations
```

Pas de `reperes serve` en v1 (pygeoapi reporté Phase 2). Pas de `reperes report` en v1 (Quarto reporté Phase 2). Pas de flags `--modules`, `--no-parallel`, `--pool-size`.

### 3.4 Gestion mémoire

Sur Mac Apple Silicon, la mémoire est unifiée CPU/GPU. Un seul modèle YOLO chargé sur MPS occupe ~500 Mo-1 Go. Aucun problème en v1 (un seul modèle, un seul processus). En cas de saturation observée pendant Phase 4 (rare avec un seul modèle), réduire `imgsz` Ultralytics ou `batch` lors du fine-tune.

---

## 4. Stack technique

| Couche | Choix | Justification |
|---|---|---|
| App iOS | Swift / SwiftUI / AVFoundation / CoreLocation / CoreMotion | Single-device, contrôle natif des verrouillages caméra |
| Lentille caméra iOS | `builtInWideAngleCamera` pinned, fallback désactivé | Préserve la calibration intrinsèque, évite bascule auto vers UltraWide en basse lumière |
| Stockage session | Bundle structuré (mp4 + jsonl) sur iPhone, transfert post-session | Pas de réseau temps réel = simplicité, robustesse |
| Pipeline | Python 3.12 + uv | MPS PyTorch, écosystème CV mature, deps reproductibles |
| Frame extraction | ffmpeg + opencv-python | Standard, contrôle codecs |
| Inférence CV | PyTorch MPS + Ultralytics YOLOv8/v11 | Apple Silicon natif |
| Pré-annotation open-vocab | Grounding DINO (HF `IDEA-Research/grounding-dino-tiny`) | Bootstrap zéro-shot via prompts texte |
| Annotation humaine | **Label Studio local** (open-source) | Outil mature, pas d'UI custom à coder en v1 |
| Modèles base chaussée | RDD2022 + fine-tune Québec hivernal | Dataset ouvert + adaptation locale |
| Format prompts | YAML versionnés Git | Source de vérité texte, pas dans le code |
| Orchestration | Click CLI + imports directs Python | Pattern standard, zéro framework, pas de plugins entry points en v1 |
| Géoréférencement | pyproj (EPSG:32188 NAD83 MTM zone 8 + EPSG:4326), shapely | Cohérence avec SIG QC |
| Snap-to-road | R-tree (rtree) + shapely sur **OSM Québec** chargé en mémoire | Pas de service externe, pas de Docker, ~10 µs par requête |
| Format intermédiaires | Parquet (pyarrow) pour pose interpolée | Compact, typé, lecture rapide |
| Stockage résultats | GeoPackage (SQLite) | OGC standard, lisible QGIS/ArcGIS/GDAL |
| Versioning code | Git + GitHub privé (v1) | Standard |
| Versioning poids ML | Manifest YAML manuel + dossier `models/` | Simple, suffit pour solo + side project |
| Tests | pytest + golden frames annotées | Régression CV |

**Différé Phase 2** : pygeoapi (service OGC API Features), Quarto + Jinja2 (rapport PDF), DVC (versioning data), ProcessPoolExecutor (parallélisation), entry points Python (plugins), FastAPI + SvelteKit (UI annotation custom), scipy (peak detection IMU pour branche inertielle), Géobase QC officielle.

Pré-requis poste développeur et instructions d'installation : `docs/dependencies.md`.

---

## 5. Pipeline de captation iOS

### 5.1 Exigences fonctionnelles de l'app v1

L'app iPhone est un **device autonome** qui capture tout sur place et transfère post-session. Périmètre v1 minimum :

1. Capturer la **vidéo en 4K H.265** via `AVCaptureSession` avec :
   - `lockForConfiguration` + `setExposureMode(.locked)` (verrouillage exposition)
   - `setFocusMode(.locked)` (verrouillage AF)
   - `setWhiteBalanceMode(.locked)` (verrouillage WB)
   - **Lentille `builtInWideAngleCamera` pinned explicitement** (pas de virtual device qui peut basculer entre Wide et UltraWide selon les conditions de luminosité)
   Les verrouillages se font après une scène de calibration courte (5-10 sec en début de session, scène typique du parcours).
2. Logger **GNSS** via `kCLLocationAccuracyBestForNavigation` à la fréquence native (typiquement 1 Hz, parfois 10 Hz).
3. Logger **IMU à 100 Hz** via `CMDeviceMotion` en mode `xMagneticNorthZVertical` — accéléromètres + gyroscope + magnétomètre. **Non exploité v1**, mais loggé pour usage Phase 2 (branche inertielle PaveAudit).
4. Logger le **cap vrai** via `CLHeading.trueHeading` (correction déclinaison magnétique automatique par iOS).
5. **Mount-check simple pré-session** : vérification que le pitch est dans ±10° de l'horizontale, alerte non bloquante si hors plage. Pas de moyenne de session enregistrée en v1 (différé Phase 2).
6. **Métadonnées de session** v1 : auto-fill avec valeurs fixes (pas de saisie utilisateur). Phase 2 ajoutera la saisie marque/modèle/véhicule + conditions météo.
7. Persister localement les flux dans le bundle de session (mp4 + jsonl), zéro réseau pendant la captation.
8. Demander les permissions : `NSLocationWhenInUseUsageDescription`, `NSMotionUsageDescription`, `NSCameraUsageDescription`, capability "Location updates" en background.
9. Transfert post-session : AirDrop, USB, ou montage SMB sur le Mac.
10. Déploiement v1 : **free-provisioning** Apple ID + Xcode 16+, déploiement initial sur l'iPhone personnel de l'auteur. Apple Developer Program à acheter avant la grosse campagne d'annotation Phase 4 si redéploiement fréquent devient un irritant (cf. PRD D-06).

### 5.2 Calibration

**Calibration intrinsèque caméra** : à faire **une fois** avant tout usage opérationnel, par modèle d'iPhone et pour la lentille `builtInWideAngleCamera` exclusivement.
- Damier d'échiquier 10×7, taille 25 mm.
- 25–40 images variées en 4K, AF verrouillé sur la même scène, lentille `builtInWideAngleCamera` pinned.
- OpenCV `calibrateCamera` → `K` (matrice intrinsèque) + `D` (distorsion).
- Critère d'acceptation : erreur de reprojection RMS < 0.5 pixel.
- Sauvegarde : `config/iphone_<model>_wide_calibration.yaml`, embarqué dans le manifest de chaque session.

**Pas de calibration extrinsèque mount** — sans objet pour l'évaluation visuelle d'état (pas besoin de positionnement précis 3D).

### 5.3 Bundle de session produit

```json
// manifest.json
{
  "session_id": "2026-04-27_14-30-00_a1b2c3d4",
  "started_at": "2026-04-27T14:30:00-04:00",
  "ended_at": "2026-04-27T15:12:33-04:00",
  "device": {
    "model": "iPhone16,1",
    "ios_version": "18.4",
    "app_version": "0.1.0"
  },
  "camera": {
    "lens": "builtInWideAngleCamera",
    "lens_pinned": true,
    "calibration_ref": "iphone_16pro_wide_calibration_v1"
  },
  "video": {
    "codec": "h265",
    "resolution": "3840x2160",
    "fps": 30,
    "duration_sec": 2553
  }
}
```

```jsonl
// gnss.jsonl (un fix par ligne)
{"ts":"2026-04-27T14:30:00.123-04:00","lat":45.5012,"lon":-73.5673,"alt":42.1,"h_acc":3.2,"v_acc":5.1,"course":182.5,"speed_mps":13.4}
```

```jsonl
// imu.jsonl (un sample 100 Hz par ligne — loggé mais non exploité v1)
{"ts":"2026-04-27T14:30:00.012-04:00","quat":[0.123,0.456,0.789,0.012],"gyro":[0.01,0.02,0.03],"accel":[0.1,0.2,9.8],"user_accel":[0.1,0.2,0.0]}
```

```jsonl
// heading.jsonl (un sample par ligne)
{"ts":"2026-04-27T14:30:00.250-04:00","true_heading":182.3,"accuracy":5.0}
```

---

## 6. Pipeline de traitement Mac

### 6.1 Ingestion

`reperes ingest <session>` :
- Valide la structure du bundle (présence de tous les fichiers attendus, manifest cohérent, lentille pinned).
- Logge un résumé : durée, distance parcourue (interpolée GNSS), qualité moyenne `horizontalAccuracy`, plage de vitesses.
- Crée le GeoPackage de session vide avec les tables communes.

L'ingestion est **légère** : elle valide et indexe.

### 6.2 Pré-annotation open-vocabulary

`reperes preannotate <session> --prompts prompts/pave_audit.yaml` :
- Charge un modèle Grounding DINO (`IDEA-Research/grounding-dino-tiny` via HuggingFace).
- Sample des frames à 1-2 fps (les défauts ne changent pas vite).
- Pour chaque classe définie dans le YAML, applique les prompts texte → bbox candidats.
- Écrit un fichier `predictions.jsonl` au format Label Studio importable directement.

Exemple de fichier prompts :

```yaml
# prompts/pave_audit.yaml
classes:
  pothole:
    prompts:
      - "pothole on asphalt road"
      - "circular hole in pavement filled with water"
    confidence_threshold: 0.35
  longitudinal_crack:
    prompts:
      - "long crack along asphalt direction"
      - "linear crack parallel to road centerline"
    confidence_threshold: 0.30
  alligator_cracking:
    prompts:
      - "alligator cracking pattern on asphalt"
      - "interconnected cracks forming polygons on pavement"
    confidence_threshold: 0.30
  patched_repair:
    prompts:
      - "rectangular asphalt patch repair"
      - "darker rectangular patch on road surface"
  frost_heave:
    prompts:
      - "frost heave deformation on asphalt"
      - "vertical bump in road surface from frost"
  salt_scaling:
    prompts:
      - "salt damage scaling on concrete pavement"
      - "surface degradation from de-icing salt"
```

### 6.3 Annotation humaine via Label Studio

L'annotation humaine s'appuie sur **Label Studio local** (cf. PRD §7.2). Workflow :
1. `reperes preannotate` produit `predictions.jsonl` au format Label Studio.
2. Lancement Label Studio local (`label-studio start`), import du JSONL, projet avec config bbox + classes MTQ + sévérités.
3. Révision humaine : confirmer / rejeter / ajuster bbox / corriger classe / corriger sévérité.
4. Export gold labels au format YOLO (export natif Label Studio).
5. `reperes import-labels <session> --from labels.zip` ré-importe les gold labels dans le GeoPackage de session (layer `verified_pave_audit`) pour traçabilité.

Pas d'API REST / WebSocket / SvelteKit en v1.

### 6.4 Évaluation — pipeline séquentiel

`reperes evaluate <session>` exécute le pipeline complet dans un seul processus Python :

1. **Validation** du bundle et du manifest (lentille pinned).
2. **Extraction des frames** vidéo via ffmpeg à 1-2 fps → `sessions/<id>/frames/*.jpg` + `frames_index.parquet`.
3. **Interpolation pose** GNSS+heading par frame → `pose_per_frame.parquet`.
4. **Chargement OSM Québec** + construction R-tree en mémoire.
5. **Inférence YOLO custom** fine-tuné (fallback sur pré-entraîné si `models/pave_audit_v1.pt` absent — utile pour Phase 0 spike).
6. **Déduplication spatio-temporelle** : clustering des détections de même classe dans rayon < 3 m + fenêtre < 5 sec → centroide GNSS + max confidence.
7. **Snap-to-road** via R-tree (test additionnel cap route ↔ cap véhicule pour départager voies parallèles).
8. **Agrégation par segment** : comptage par classe et sévérité, densité de défauts par m², conversion en valeur de déduction MTQ par classe selon barèmes, **score IES visuel 0-100**.
9. **Écriture GeoPackage** `assets.gpkg` (transactions atomiques par layer).
10. **Production** `validation_report.md` (résumé numérique : nb détections par classe, IES par segment, durée, version modèle, version code).

### 6.5 Export labels et fine-tuning

`reperes train --module pave_audit --dataset datasets/qc_pave_v1/` :
- Lance Ultralytics YOLO training (MPS PyTorch).
- Mesure mAP sur split de validation.
- Sauvegarde `models/pave_audit_v<N>.pt` et met à jour `models/MANIFEST.yaml` :

```yaml
# models/MANIFEST.yaml
pave_audit:
  - version: v1
    weights: pave_audit_v1.pt
    git_commit: 7a3f2b1
    dataset: qc_pave_v1
    trained_at: 2026-06-15T12:00:00-04:00
    metrics:
      mAP_50: 0.62
      mAP_50_95: 0.41
    notes: "Bootstrap RDD2022 + 1500 frames Mtl annotées"
  - version: v2
    weights: pave_audit_v2.pt
    git_commit: c9d8a7e
    dataset: qc_pave_v2
    trained_at: 2026-08-02T18:30:00-04:00
    metrics:
      mAP_50: 0.78
      mAP_50_95: 0.54
    notes: "Ajout 1500 frames Mtl + frost_heave annotation dédiée"
```

### 6.6 Livraison

**v1 = GeoPackage seul.** Pas de PDF Quarto, pas de service OGC. Le client (auteur) ouvre `assets.gpkg` dans QGIS pour visualiser :
- Layer `pavement_segments` colorée par `ies_visual`
- Layer `pavement_defects` pour le détail des détections individuelles

`validation_report.md` complète avec :
- Sommaire numérique (km parcourus, défauts par classe, IES par segment, % de session traitée)
- Trois κ (κ_self, κ_pipeline, ratio) si Phase 4 validation lancée
- Cas problématiques identifiés

---

## 7. Modèle de données

### 7.1 Schéma GeoPackage `assets.gpkg` v1

```
Tables communes:
  sessions              # toutes sessions captées (id, dates, device)
  frames_index          # frames par session (frame_idx, ts_iso, has_detection)

Tables PaveAudit (visuel uniquement v1):
  pavement_defects      # détections visuelles individuelles (POINT)
  pavement_segments     # segments routiers avec ies_visual (LINESTRING)
  pavement_metadata     # version modèle, prompts, conditions

Tables Annotation:
  candidates_pave_audit     # sortie pré-annotation Grounding DINO
  verified_pave_audit       # gold labels post-révision Label Studio
  annotations_log           # historique audit trail (importé depuis Label Studio)
```

Pas de `pavement_imu_events`, `construction_*`, `permits_reference` en v1.

### 7.2 Schéma `pavement_segments` v1

```sql
CREATE TABLE pavement_segments (
  segment_id           INTEGER PRIMARY KEY,
  geom                 LINESTRING NOT NULL,        -- WGS84 (EPSG:4326)
  geom_mtm             LINESTRING,                  -- NAD83 MTM zone 8 pour SIG QC
  street_name          TEXT,
  start_chainage_m     REAL,
  length_m             REAL NOT NULL,
  
  -- Branche visuelle (méthodologie MTQ)
  ies_visual           REAL,                        -- 0-100
  ies_classification   TEXT,                        -- 'Bon' | 'Acceptable' | 'Mauvais' | 'Très mauvais'
  n_defects_total      INTEGER,
  n_defects_by_class   JSON,                        -- {"pothole": 3, "long_crack": 7, ...}
  
  session_id           TEXT NOT NULL,
  evaluated_at         TIMESTAMP NOT NULL,
  model_version        TEXT NOT NULL                -- ex. "pave_audit_v2"
);
```

Colonnes `inertial_score`, `inertial_severity`, `n_imu_events`, `pct_in_speed_range`, `vehicle_signature` ajoutées Phase 2 (migration ALTER TABLE).

### 7.3 Schéma `pavement_defects` v1

```sql
CREATE TABLE pavement_defects (
  defect_id            INTEGER PRIMARY KEY,
  geom                 POINT NOT NULL,              -- WGS84, position GNSS interpolée
  frame_idx            INTEGER NOT NULL,
  bbox                 JSON,                         -- {x, y, w, h} en pixels
  class                TEXT NOT NULL,                -- 'pothole', 'long_crack', etc.
  severity             TEXT,                         -- 'légère' | 'modérée' | 'sévère'
  confidence           REAL NOT NULL,
  segment_id           INTEGER,                      -- FK vers pavement_segments
  session_id           TEXT NOT NULL,
  ts                   TIMESTAMP NOT NULL,
  model_version        TEXT NOT NULL
);
```

---

## 8. Roadmap Phase 2 (résumé technique)

Pas de spec implémentable, juste les changements architecturaux à anticiper :

**Branche inertielle PaveAudit** (priorité 1) :
- Ajout `reperes/imu.py` : filtrage passe-haut, peak detection, normalisation vitesse, plage 30-70 km/h
- Production `imu_events.parquet` à partir de `imu.jsonl` (déjà loggé en v1)
- Calcul `inertial_score` par segment, ajout colonnes à `pavement_segments`
- Schéma `pavement_imu_events` (POINT)

**ChantierWatch** (priorité 2) :
- Ajout `reperes/chantier_watch.py` (ou refactor en deux modules `reperes_pave_audit/`, `reperes_chantier_watch/`)
- Adaptateurs permits par ville (`MontrealPermitAdapter`, etc.)
- Schémas `construction_detections`, `construction_zones`, `permits_reference`
- DBSCAN spatial pour zones, R-tree permits pour cross-check temporel

**Quand le découpage en plugins entry points devient utile** :
- Si Phase 2 introduit > 2 modules (PaveAudit visuel + inertiel + ChantierWatch + ?), refactorer en `reperes_core/`, `reperes_pave_audit/`, `reperes_chantier_watch/` avec entry points Python.

**Quand la parallélisation devient utile** :
- Si > 2 modules tournent simultanément et que le temps total dépasse 5× temps réel, ajouter `ProcessPoolExecutor` avec écriture intermédiaire par module dans `prepared/results_<module>.gpkg` puis consolidation atomique.

**Service OGC (pygeoapi) Phase 2** :
- Configuration dans `pygeoapi-config.yaml`, exposition `pavement_segments`, `pavement_imu_events`, `construction_zones`.

**Rapport PDF Quarto Phase 2** :
- Template Quarto + Jinja2 dans `reports/templates/`, généré par `reperes report <session> --format pdf`.

**UI annotation custom** : à coder uniquement si Label Studio s'avère bloquant à l'usage en v1 — en pratique cela veut dire Phase 2+, et seulement si une démo client justifie l'effort.
