# Architecture technique — wsmd

**Date** : 2026-04-27
**Statut** : draft, dérive du PRD `docs/prd.md`
**Public** : développeurs, intervenants techniques, futurs contributeurs

> Ce document décrit le **COMMENT**. Le **QUOI** et le **POURQUOI** vivent dans `docs/prd.md`. Toute divergence entre les deux doit être résolue en faveur du PRD ; ce document suit les décisions produit, il ne les fixe pas.

---

## 1. Vue d'ensemble

```
┌────────────────────────────────────────────────────┐
│  iPhone 16 Pro (mount pare-brise) — DEVICE UNIQUE │
│  ────────────────────────────────────────────────  │
│  App SwiftUI native                                 │
│    • AVCaptureSession 4K H.265 (AF/AE/WB locked)   │
│    • CoreLocation (GNSS @ 1–10 Hz)                  │
│    • CoreMotion (IMU 100 Hz, x-magnetic-north)     │
│    • CLHeading (cap vrai, déclinaison corrigée)    │
│    • Persistance locale → session bundle           │
└──────────────────────┬─────────────────────────────┘
                       │ Transfert post-session
                       │ (AirDrop / USB / SMB)
                       ↓
┌────────────────────────────────────────────────────┐
│  MacBook M-series — pipeline post-captation        │
│  ────────────────────────────────────────────────  │
│  wsmd ingest <session>      ← validation + index   │
│  wsmd preannotate <session> --prompts <yaml>       │
│  wsmd annotate <session>    ← UI web vérification  │
│  wsmd evaluate --modules X  ← lance évaluateurs    │
│    ┌──────────────┐  ┌────────────────┐  ┌─────┐  │
│    │  PaveAudit   │  │  ChantierWatch │  │ ... │  │
│    └──────┬───────┘  └────────┬───────┘  └──┬──┘  │
│           ↓                   ↓             ↓     │
│        ┌──────────────────────────────────────┐   │
│        │  GeoPackage assets.gpkg (multi-layer)│   │
│        └──────────────────────────────────────┘   │
│  wsmd export-labels <session>  → dataset YOLO/COCO │
│  wsmd train --module X         → fine-tune custom  │
│  wsmd report <session>      ← PDF + GPKG final    │
│  wsmd serve                  ← pygeoapi (OGC)      │
└────────────────────────────────────────────────────┘
```

**Principe clé** : la captation produit un bundle de session **fixe et standard**. Les évaluateurs sont des **plugins indépendants** (Python entry points) qui consomment ce bundle et écrivent leurs résultats dans des layers GeoPackage distincts. Aucun couplage entre évaluateurs.

---

## 2. Topologie disque

```
data/
  permits/                       # cache local des permits OdP par ville
    montreal/entraves_2026.gpkg
    longueuil/permits_2026.gpkg  # si disponible
    sherbrooke/...
  geobase/                        # cache Géobase Québec (référentiel routier)
    troncons_qc.gpkg
  models/                         # poids fine-tunés versionnés (option B)
    pave_audit_v1.pt
    pave_audit_v2.pt
    chantier_watch_v1.pt
    MANIFEST.yaml                 # mapping version ↔ commit Git ↔ dataset ↔ métriques
  prompts/                        # versionnés Git
    pave_audit.yaml
    chantier_watch.yaml

sessions/
  2026-04-27_14-30-00_<uuid>/
    manifest.json                 # device, version app, calibration, conditions
    video.mp4                     # 4K H.265, AF/AE/WB verrouillés
    frames.jsonl                  # frame_idx, ts_iso (extrait au post-traitement)
    gnss.jsonl                    # samples GNSS
    imu.jsonl                     # samples IMU 100 Hz
    heading.jsonl                 # samples cap vrai
    assets.gpkg                   # output multi-layer (un layer par évaluateur)
    report.pdf                    # rapport client final
    validation_report.md          # erreurs mesurées vs vérité (si validation lancée)
```

---

## 3. Architecture évaluateurs

**Interface évaluateur (registry pattern via Python entry points)** :

```python
class Evaluator(Protocol):
    name: str                         # ex. "pave_audit"
    output_layers: list[str]          # tables GeoPackage produites
    
    def configure(self, cfg: dict) -> None: ...
    def evaluate(self, session: Session) -> EvaluatorResult: ...
    def validate(self, result, ground_truth: Path) -> Metrics: ...
```

**Découverte automatique** via le entry point group `wsmd.evaluators` → ajouter un nouveau module = un nouveau package Python qui s'enregistre. Aucune modification du core.

**Pipeline d'exécution** par évaluateur, en série (parallélisable plus tard) :
1. Charge le bundle de session
2. Extrait frames + interpole pose/heading par frame
3. Inférence CV (modèle propre à l'évaluateur)
4. Géoréférence les détections (snap-to-road via OSRM Docker)
5. Logique métier propre (cross-check permits, calcul IES, etc.)
6. Écrit ses layers dans le GeoPackage de session

**CLI** :
```bash
wsmd ingest 2026-04-27_14-30-00_*/
wsmd preannotate <session> --prompts prompts/pave_audit.yaml
wsmd annotate <session>                                # ouvre UI web localhost
wsmd evaluate <session> --modules pave_audit,chantier_watch
wsmd export-labels <session>                            # exporte gold labels YOLO/COCO
wsmd train --module pave_audit --dataset <dir>         # fine-tune custom
wsmd report <session> --format pdf
wsmd serve                                              # pygeoapi http://localhost:5000
```

---

## 4. Stack technique

| Couche | Choix | Justification |
|---|---|---|
| App iOS | Swift / SwiftUI / AVFoundation / CoreLocation / CoreMotion | Single-device, contrôle natif des verrouillages caméra |
| Stockage session | Bundle structuré (mp4 + jsonl) sur iPhone, transfert post-session | Pas de réseau temps réel = simplicité, robustesse |
| Pipeline | Python 3.12 + uv | MPS PyTorch, écosystème CV mature, deps reproductibles |
| Frame extraction | ffmpeg + opencv-python | Standard, contrôle codecs |
| Inférence CV | PyTorch MPS + Ultralytics YOLOv8/v11 | Apple Silicon natif |
| Pré-annotation open-vocab | Grounding DINO (HF `IDEA-Research/grounding-dino-tiny`) ET/OU YOLO-World (Ultralytics) | Bootstrap zéro-shot via prompts texte |
| Modèles base chaussée | RDD2022 + fine-tune Québec hivernal | Dataset ouvert + adaptation locale |
| Modèles base chantier | YOLOv8 COCO + fine-tune cônes/barrières/déblais QC | COCO partiellement, fine-tune léger |
| Format prompts | YAML versionnés Git par module | Source de vérité texte, pas dans le code |
| Orchestration plugins | Python entry points + Click CLI | Pattern standard, zéro framework |
| Géoréférencement | pyproj (EPSG:32188 NAD83 MTM zone 8), shapely | Cohérence avec SIG QC |
| Snap-to-road | **OSRM en Docker local** | Powerful, open-source, supporte routing + map matching |
| Cache permits | GeoPackage local + R-tree | Sub-ms par requête |
| Stockage résultats | GeoPackage (SQLite) | OGC standard, lisible QGIS/ArcGIS/GDAL |
| Service web | pygeoapi | OGC API Features de référence |
| Outil annotation | FastAPI backend + SvelteKit frontend | Web local léger, raccourcis clavier ergonomiques |
| Rapport PDF | Quarto + Jinja2 templates | Templating moderne, livrable pro |
| Versioning code | Git + GitHub privé (v1) | Standard |
| Versioning poids ML | Manifest YAML manuel + dossier `models/` (option B) | Simple, suffit pour solo + side project |
| Versioning data | DVC vers bucket B2 (Phase 2) | Plus tard si besoin |
| Tests | pytest + golden frames annotées | Régression CV par évaluateur |

Pré-requis poste développeur et instructions d'installation : `docs/dependencies.md`.

---

## 5. Pipeline de captation iOS

### 5.1 Exigences fonctionnelles de l'app

L'app iPhone est un **device autonome** qui capture tout sur place et transfère post-session. Elle doit :

1. Capturer la **vidéo en 4K H.265** via `AVCaptureSession` avec :
   - `lockForConfiguration` + `setExposureMode(.locked)` (verrouillage exposition)
   - `setFocusMode(.locked)` (verrouillage AF)
   - `setWhiteBalanceMode(.locked)` (verrouillage WB)
   Les verrouillages se font après une scène de calibration courte (5-10 sec en début de session, scène typique du parcours).
2. Logger **GNSS** via `kCLLocationAccuracyBestForNavigation` à la fréquence native (typiquement 1 Hz, parfois 10 Hz).
3. Logger **IMU à 100 Hz** via `CMDeviceMotion` en mode `xMagneticNorthZVertical`.
4. Logger le **cap vrai** via `CLHeading.trueHeading` (correction déclinaison magnétique automatique par iOS).
5. Persister localement les flux dans le bundle de session (mp4 + jsonl), zéro réseau pendant la captation.
6. Demander les permissions : `NSLocationWhenInUseUsageDescription`, `NSMotionUsageDescription`, `NSCameraUsageDescription`, capability "Location updates" en background.
7. Transfert post-session : AirDrop, USB, ou montage SMB sur le Mac.
8. Déploiement : free-provisioning Apple ID + Xcode 16+ pour usage personnel et démos. Apple Developer Program (99 $/an) seulement quand un partenaire externe doit l'installer.

### 5.2 Calibration

**Calibration intrinsèque caméra** : à faire **une fois** avant tout usage opérationnel, par modèle d'iPhone.
- Damier d'échiquier 10×7, taille 25 mm.
- 25–40 images variées en 4K, AF verrouillé sur la même scène.
- OpenCV `calibrateCamera` → `K` (matrice intrinsèque) + `D` (distorsion).
- Critère d'acceptation : erreur de reprojection RMS < 0.5 pixel.
- Sauvegarde : `config/iphone_<model>_calibration.yaml`, embarqué dans le manifest de chaque session.

**Pas de calibration extrinsèque mount** — sans objet pour l'évaluation d'état (pas besoin de positionnement précis 3D).

**Mount-check pré-session** : l'app vérifie via IMU que la caméra est dans une plage de pitch/roll acceptable (caméra horizontale ±10°, vue dégagée) et alerte sinon.

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
  "calibration_ref": "iphone_16pro_calibration_v1",
  "conditions": {
    "weather": "clear",      // saisi par utilisateur, optionnel
    "road_wet": false,
    "lighting": "daylight"
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
{"ts":"2026-04-27T14:30:00.123-04:00","lat":45.5012,"lon":-73.5673,"alt":42.1,"h_acc":3.2,"v_acc":5.1,"course":182.5}
```

```jsonl
// imu.jsonl (un sample 100 Hz par ligne)
{"ts":"2026-04-27T14:30:00.012-04:00","quat":[0.123,0.456,0.789,0.012],"gyro":[0.01,0.02,0.03],"accel":[0.1,0.2,9.8]}
```

```jsonl
// heading.jsonl (un sample par ligne)
{"ts":"2026-04-27T14:30:00.250-04:00","true_heading":182.3,"accuracy":5.0}
```

---

## 6. Pipeline de traitement Mac

### 6.1 Ingestion

`wsmd ingest <session>` :
- Valide la structure du bundle (présence de tous les fichiers attendus).
- Extrait `frames.jsonl` à partir de la vidéo via ffmpeg (timestamp ISO 8601 par frame).
- Crée le GeoPackage de session vide avec les tables communes.
- Logge un résumé : durée, distance parcourue (interpolée GNSS), qualité moyenne `horizontalAccuracy`.

### 6.2 Pré-annotation open-vocabulary

`wsmd preannotate <session> --prompts prompts/<module>.yaml` :
- Charge un modèle Grounding DINO (`IDEA-Research/grounding-dino-tiny` via HuggingFace) ou YOLO-World (Ultralytics).
- Sample des frames à 1-2 fps (les défauts ne changent pas vite).
- Pour chaque classe définie dans le YAML, applique les prompts texte → bbox candidats.
- Écrit le layer `candidates_<module>` dans le GeoPackage de session.

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

```yaml
# prompts/chantier_watch.yaml
classes:
  traffic_cone:
    prompts: ["orange traffic cone with reflective stripes"]
  jersey_barrier:
    prompts: ["concrete jersey barrier", "K-rail concrete barrier"]
  construction_fence:
    prompts: ["construction site fence", "chain-link construction perimeter fence", "wooden site hoarding"]
  temporary_sign:
    prompts: ["temporary construction warning sign", "orange diamond construction sign"]
  construction_vehicle:
    prompts: ["excavator on construction site", "asphalt paver", "construction truck", "hydraulic excavator"]
  excavated_soil:
    prompts: ["pile of excavated soil", "construction debris pile", "asphalt millings pile"]
```

### 6.3 Annotation humaine

`wsmd annotate <session>` ouvre une UI web locale (FastAPI + SvelteKit, port 5001 par défaut) qui lit/écrit le même GeoPackage de session que les évaluateurs. Spec produit (workflow, raccourcis clavier, ergonomie cible) : voir PRD §7.3.

### 6.4 Évaluation par module

`wsmd evaluate <session> --modules pave_audit,chantier_watch` exécute chaque évaluateur en série :
1. Charge le bundle + GeoPackage de session
2. Charge le modèle custom fine-tuné (si dispo, sinon fallback sur pré-entraîné)
3. Inference frame-par-frame ou par track
4. Snap-to-road via OSRM (HTTP `/match` API)
5. Logique métier propre (calcul IES MTQ, cross-check permits, etc.)
6. Écrit ses layers de production dans le GeoPackage

### 6.5 Export labels et fine-tuning

`wsmd export-labels <session> --format yolo --output datasets/qc_pave_v1/` :
- Lit le layer `verified_pave_audit` (gold labels uniquement, `verdict ∈ {confirmed, corrected, added_manually}`)
- Exporte au format YOLO ou COCO
- Maintient un index global `datasets/qc_pave_v1/index.yaml` : sessions incluses, frames count, classe distribution

`wsmd train --module pave_audit --dataset datasets/qc_pave_v1/` :
- Lance Ultralytics YOLO training (MPS PyTorch)
- Mesure mAP sur split de validation
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
    notes: "Ajout 2500 frames Longueuil + frost_heave annotation dédiée"
```

### 6.6 Livraison

`wsmd report <session> --format pdf` génère un rapport (template Quarto + Jinja2) :
- Page 1 : sommaire exécutif (km parcourus, défauts par classe, zones chantier détectées, % unmatched)
- Page 2-3 : carte du réseau colorée par IES
- Page 4 : top 20 segments les plus dégradés avec photo représentative
- Page 5-6 : zones de chantier avec verdict permit + photos extraites
- Annexe : tableau complet, métadonnées, version pipeline (modèle utilisé + commit Git)

`wsmd serve` lance pygeoapi sur `http://localhost:5000` exposant les layers publics (segments, zones) ; les layers internes (candidates, frames_index) restent privés.

---

## 7. Modèle de données et livraison OGC

### 7.1 Schéma GeoPackage `assets.gpkg`

```
Tables communes:
  sessions              # toutes sessions captées (id, dates, device, conditions)
  frames_index          # frames par session (frame_idx, ts_iso, has_detection)

Tables PaveAudit:
  pavement_defects      # détections individuelles (POINT, attributs ci-dessous)
  pavement_segments     # segments routiers avec IES (LINESTRING)
  pavement_metadata     # version modèle, prompts, conditions

Tables ChantierWatch:
  construction_detections   # détections individuelles (POINT)
  construction_zones        # zones agrégées avec verdict (POLYGON)
  permits_reference         # cache permits utilisé (POLYGON, traçabilité)
  unmatched_zones           # vue filtrée des unmatched

Tables Annotation:
  candidates_<module>       # sortie pré-annotation par module
  verified_<module>         # gold labels post-révision humaine
  annotations_log           # historique audit trail
```

### 7.2 Schéma `pavement_segments`

```sql
CREATE TABLE pavement_segments (
  segment_id        INTEGER PRIMARY KEY,
  geom              LINESTRING NOT NULL,           -- WGS84 (EPSG:4326)
  geom_mtm          LINESTRING,                     -- NAD83 MTM zone 8 pour SIG QC
  street_name       TEXT,
  start_chainage_m  REAL,
  length_m          REAL NOT NULL,
  ies_score         REAL,                           -- 0-100, méthodologie MTQ
  ies_classification TEXT,                          -- ex. 'Bon', 'Acceptable', 'Mauvais', 'Très mauvais'
  n_defects_total   INTEGER,
  n_defects_by_class JSON,                          -- {"pothole": 3, "long_crack": 7, ...}
  session_id        TEXT NOT NULL,
  evaluated_at      TIMESTAMP NOT NULL,
  model_version     TEXT NOT NULL                   -- ex. "pave_audit_v2"
);
```

### 7.3 Schéma `construction_zones`

```sql
CREATE TABLE construction_zones (
  zone_id           INTEGER PRIMARY KEY,
  geom              POLYGON NOT NULL,               -- WGS84
  centroid          POINT,
  detections_count  INTEGER NOT NULL,
  classes_present   JSON,                           -- ["traffic_cone", "jersey_barrier", ...]
  verdict           TEXT NOT NULL,                  -- 'permitted' | 'unmatched' | 'permit_expired' | 'data_unavailable'
  matched_permit_ids JSON,                          -- ['PERMIT-2026-12345', ...] si verdict=permitted
  session_id        TEXT NOT NULL,
  detected_at       TIMESTAMP NOT NULL,
  model_version     TEXT NOT NULL
);
```

### 7.4 Configuration pygeoapi

```yaml
# pygeoapi-config.yaml
server:
  bind: { host: 0.0.0.0, port: 5000 }
  url: http://localhost:5000

resources:
  pavement_segments:
    type: collection
    title: Segments de chaussée évalués (PaveAudit)
    description: État de chaussée selon méthodo MTQ
    keywords: [chaussée, IES, MTQ, audit]
    crs: [http://www.opengis.net/def/crs/EPSG/0/4326]
    providers:
      - type: feature
        name: GeoPackage
        data: /path/to/assets.gpkg
        id_field: segment_id
        table: pavement_segments
  
  construction_zones:
    type: collection
    title: Zones de chantier détectées (ChantierWatch)
    description: Détection automatisée + cross-check permits OdP
    providers:
      - type: feature
        name: GeoPackage
        data: /path/to/assets.gpkg
        id_field: zone_id
        table: construction_zones
```

### 7.5 Endpoints exposés

- `GET /collections/pavement_segments/items?bbox=...&ies_score__lt=50` — filtrage par BBox + score
- `GET /collections/pavement_segments/items/{segment_id}` — segment individuel
- `GET /collections/construction_zones/items?verdict=unmatched` — zones flaggées
- `GET /collections/construction_zones/items?f=html` — UI HTML simple intégrée

### 7.6 Consommation cible

- **QGIS** : ajout de couche WFS / OGC API Features → consommation native
- **Curl / scripts** : `curl http://localhost:5000/collections/pavement_segments/items?bbox=... > export.geojson`
- **Browser** : interface HTML générée automatiquement par pygeoapi
