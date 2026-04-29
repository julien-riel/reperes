# Architecture technique — Repères

**Date** : 2026-04-28
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
│    • Lentille builtInWideAngleCamera PINNED        │
│    • CoreLocation (GNSS @ 1–10 Hz)                  │
│    • CoreMotion (IMU 100 Hz, x-magnetic-north)     │
│    • CLHeading (cap vrai, déclinaison corrigée)    │
│    • Mount-check pitch/roll, log dans manifest      │
│    • Persistance locale → session bundle           │
└──────────────────────┬─────────────────────────────┘
                       │ Transfert post-session
                       │ (AirDrop / USB / SMB)
                       ↓
┌────────────────────────────────────────────────────┐
│  MacBook M-series — pipeline post-captation        │
│  Aucun service externe, aucun Docker en v1         │
│  ────────────────────────────────────────────────  │
│  reperes ingest <session>      ← validation        │
│  reperes preannotate <session> --prompts <yaml>    │
│  reperes annotate <session>    ← UI web vérif      │
│  reperes evaluate --modules X  ← orchestrateur     │
│                                                    │
│  ╔════════════════════════════════════════════╗    │
│  ║ STAGE 1 — Préparation commune (séquentiel) ║    │
│  ║ • Extract frames @ 1-2 fps                 ║    │
│  ║ • Interpolate pose GNSS+IMU per frame      ║    │
│  ║ • Detect IMU events (peaks, normalized vit)║    │
│  ║ • Load Géobase R-tree in memory            ║    │
│  ║ → PreparedStreams                          ║    │
│  ╚════════════════════════════════════════════╝    │
│                       ↓                            │
│  ╔════════════════════════════════════════════╗    │
│  ║ STAGE 2 — Évaluateurs en parallèle         ║    │
│  ║   ProcessPoolExecutor                      ║    │
│  ║  ┌──────────────┐  ┌────────────────┐ ┌──┐ ║    │
│  ║  │ PaveAudit    │  │ ChantierWatch  │ │..│ ║    │
│  ║  │ (vis+inertl) │  │  (vis only)    │ │  │ ║    │
│  ║  └──────┬───────┘  └────────┬───────┘ └─┬┘ ║    │
│  ║         │                   │           │  ║    │
│  ║         └─────── snap-to-road via ──────┘  ║    │
│  ║                  R-tree (rtree+shapely)    ║    │
│  ╚═══════════════════╤════════════════════════╝    │
│                      ↓                             │
│  ╔════════════════════════════════════════════╗    │
│  ║ STAGE 3 — Consolidation (séquentiel)       ║    │
│  ║ • Écriture atomique GeoPackage             ║    │
│  ║ • validation_report.md                     ║    │
│  ║ • Logs structurés                          ║    │
│  ╚════════════════════════════════════════════╝    │
│                                                    │
│  reperes export-labels <session>  → dataset YOLO   │
│  reperes train --module X         → fine-tune      │
│  reperes report <session>         → PDF + GPKG     │
│  reperes serve                    → pygeoapi (OGC) │
└────────────────────────────────────────────────────┘
```

**Principes clés** :

- *Captation produit un bundle de session **fixe et standard**.* Un évaluateur n'a jamais à re-lire les fichiers bruts (vidéo, jsonl IMU/GNSS) — il consomme les artefacts préparés par Stage 1.
- *Stage 1 est commun* — extraction de frames, interpolation pose, détection événements inertiels sont faites **une seule fois** par session, partagées entre tous les évaluateurs.
- *Stage 2 est parallèle* — chaque évaluateur tourne dans son propre sous-processus Python, charge son modèle ML sur MPS, fait son inférence et sa logique métier indépendamment.
- *Stage 3 sérialise les écritures* — pour éviter les corruptions GeoPackage par accès concurrent SQLite, les résultats des évaluateurs sont consolidés en écriture atomique séquentielle.
- *Architecture pluggable* — les évaluateurs sont des plugins Python (entry points). Aucun couplage entre eux. Ajouter `reperes_signage_state` = nouveau package, zéro modification du core.

---

## 2. Topologie disque

```
data/
  permits/                       # cache local des permits OdP par ville
    montreal/entraves_2026.gpkg
    longueuil/permits_2026.gpkg  # si disponible
    sherbrooke/...
  road_network/                   # référentiel routier pour snap-to-road
    geobase_qc.gpkg              # Géobase Québec (chargée en R-tree au runtime)
    osm_qc_supplement.gpkg       # OSM extrait QC en complément si Géobase manque
  models/                         # poids fine-tunés versionnés
    pave_audit_v1.pt
    pave_audit_v2.pt
    chantier_watch_v1.pt
    MANIFEST.yaml                 # mapping version ↔ commit Git ↔ dataset ↔ métriques
  prompts/                        # versionnés Git
    pave_audit.yaml
    chantier_watch.yaml

sessions/
  2026-04-27_14-30-00_<uuid>/
    manifest.json                 # device, version app, calibration, lentille, mount, véhicule
    video.mp4                     # 4K H.265, AF/AE/WB verrouillés, lentille pinned
    frames.jsonl                  # frame_idx, ts_iso (extrait au post-traitement)
    gnss.jsonl                    # samples GNSS
    imu.jsonl                     # samples IMU 100 Hz (accel, gyro, quat)
    heading.jsonl                 # samples cap vrai
    prepared/                     # produit par Stage 1, consommé par Stage 2
      pose_per_frame.parquet      # pose interpolée par frame
      imu_events.parquet          # pics inertiels normalisés vitesse
    assets.gpkg                   # output multi-layer (un layer par évaluateur)
    report.pdf                    # rapport client final
    validation_report.md          # erreurs mesurées vs vérité (si validation lancée)
    logs/                         # logs structurés JSON par invocation CLI
      ingest_2026-04-27T15-12-00.jsonl
      evaluate_2026-04-27T15-30-00.jsonl
```

---

## 3. Architecture évaluateurs

### 3.1 Couches du codebase

```
reperes_core/             ← bibliothèque partagée
  streams.py              ← FrameStream, PoseStream, IMUEventStream, PreparedStreams
  imu_processing.py       ← détection pics, normalisation vitesse, plage 30-70 km/h
  pose_interpolation.py   ← interpolation GNSS+IMU par frame
  road_matcher.py         ← snap-to-road R-tree + shapely sur Géobase
  geobase.py              ← chargement référentiel, construction R-tree au démarrage
  logging.py              ← logger contextuel JSON (session_id)
  orchestrator.py         ← Stage 1 / Stage 2 (ProcessPoolExecutor) / Stage 3
  cli.py                  ← Click CLI : reperes ingest/preannotate/...

reperes_pave_audit/       ← plugin module
  evaluator.py            ← classe PaveAuditEvaluator (vision + inertiel)
  pyproject.toml          ← entry point: reperes.evaluators = pave_audit:PaveAuditEvaluator

reperes_chantier_watch/   ← plugin module
  evaluator.py            ← classe ChantierWatchEvaluator (vision only)
  permit_adapter.py       ← Mtl, Longueuil, Sherbrooke, GenericNone
  pyproject.toml

reperes_annotate/         ← outil web FastAPI + SvelteKit
  backend/                ← FastAPI
  frontend/               ← SvelteKit
```

### 3.2 Interface évaluateur

```python
from typing import Protocol
from dataclasses import dataclass
from pathlib import Path

@dataclass
class PreparedStreams:
    """Artefacts produits par Stage 1, partagés par tous les évaluateurs en Stage 2.
    Lecture seule. Sérialisable via pickle pour passage inter-process."""
    session: Session
    frames_index_path: Path        # parquet : frame_idx, ts_iso, frame_image_path
    pose_per_frame_path: Path      # parquet : frame_idx, lat, lon, heading, speed_mps, pitch, roll
    imu_events_path: Path          # parquet : ts, lat, lon, accel_z_normalized, severity_class

class Evaluator(Protocol):
    name: str                                  # ex. "pave_audit"
    output_layers: list[str]                   # tables GeoPackage produites
    
    def configure(self, cfg: dict) -> None: ...
    def evaluate(self, streams: PreparedStreams) -> EvaluatorResult: ...
    def validate(self, result, ground_truth: Path) -> Metrics: ...
```

**Découverte automatique** via le entry point group `reperes.evaluators` → ajouter un nouveau module = un nouveau package Python qui s'enregistre. Aucune modification du core.

### 3.3 Pipeline d'exécution à 3 étages

**Stage 1 — Préparation commune (séquentiel, fait une seule fois par session)**

Fait par `reperes_core.orchestrator.prepare_streams(session)` :

1. Validation du bundle de session (présence de tous les fichiers, manifest cohérent, lentille pinned).
2. Extraction des frames vidéo via ffmpeg à 1-2 fps, écriture des images sur disque, index parquet.
3. Interpolation pose par frame : pour chaque timestamp de frame, interpolation linéaire des samples GNSS (lat, lon, speed) et IMU (heading, pitch, roll) → écriture parquet `pose_per_frame.parquet`.
4. Détection des événements inertiels :
   - Lecture `imu.jsonl` (samples 100 Hz)
   - Filtre passe-haut sur l'accélération verticale pour retirer la composante gravité résiduelle
   - Détection des pics au-dessus d'un seuil dynamique (ex. 3× écart-type de la session) avec espacement temporel minimal
   - Pour chaque pic : recherche du sample GNSS correspondant, calcul de la vitesse instantanée
   - Filtre vitesse 30-70 km/h (pics hors plage rejetés ou flaggés)
   - Normalisation amplitude par vitesse (formule type `amplitude / sqrt(speed)`)
   - Écriture parquet `imu_events.parquet`
5. Chargement Géobase QC en R-tree (rtree + shapely), conservé en mémoire pour le reste du run.
6. Retour d'un objet `PreparedStreams` (paths vers parquets, référence Géobase passée par variable globale au worker pool).

**Stage 2 — Évaluateurs en parallèle (ProcessPoolExecutor)**

Fait par `reperes_core.orchestrator.run_evaluators(streams, evaluators, pool_size)` :

```python
from concurrent.futures import ProcessPoolExecutor, as_completed

def run_evaluators(streams, evaluators, pool_size=None):
    pool_size = pool_size or len(evaluators)
    results = {}
    with ProcessPoolExecutor(max_workers=pool_size) as pool:
        futures = {pool.submit(_run_one, ev, streams): ev.name for ev in evaluators}
        for fut in as_completed(futures):
            name = futures[fut]
            results[name] = fut.result()  # propage exception si crash
    return results

def _run_one(evaluator, streams):
    """Exécuté dans un sous-processus. Charge le modèle ML, fait l'inférence, 
    écrit les layers dans son propre fichier intermédiaire (pas directement dans 
    assets.gpkg pour éviter la concurrence SQLite)."""
    return evaluator.evaluate(streams)
```

Chaque évaluateur :
1. Reçoit `PreparedStreams` (sérialisé pickle au passage inter-process — léger car ce sont des paths)
2. Charge ses parquets en lecture seule, son modèle YOLO sur MPS
3. Fait son inférence frame-par-frame (PaveAudit consomme en plus `imu_events_path` pour sa branche inertielle)
4. Snap-to-road via le service partagé `RoadMatcher` (R-tree chargé une fois par sous-processus au démarrage)
5. Logique métier propre (déduplication spatio-temporelle, agrégation par segment, calcul IES, cross-check permits)
6. Écrit ses résultats dans un GeoPackage intermédiaire `prepared/results_<module>.gpkg`

**Stage 3 — Consolidation (séquentiel)**

Fait par `reperes_core.orchestrator.consolidate(session, results)` :

1. Ouverture de `assets.gpkg` en écriture
2. Pour chaque module : copie atomique de ses layers depuis `results_<module>.gpkg` vers `assets.gpkg` (transaction par module)
3. Génération de `validation_report.md` (résumé numérique : nb détections par module, durée Stage 1/2/3, erreurs)
4. Logs structurés finalisés dans `logs/evaluate_<ts>.jsonl`

### 3.4 CLI

```bash
reperes ingest 2026-04-27_14-30-00_*/
reperes preannotate <session> --prompts prompts/pave_audit.yaml
reperes annotate <session>                                # ouvre UI web localhost
reperes evaluate <session> --modules pave_audit,chantier_watch
reperes evaluate <session> --modules pave_audit --no-parallel  # debug
reperes export-labels <session>                            # exporte gold labels YOLO/COCO
reperes train --module pave_audit --dataset <dir>          # fine-tune custom
reperes report <session> --format pdf
reperes serve                                              # pygeoapi http://localhost:5000
```

Le flag `--no-parallel` exécute les évaluateurs en série dans le process principal — utile pour debug avec breakpoints, stack traces lisibles, ou quand la RAM est tight.

### 3.5 Gestion mémoire MPS en mode parallèle

Sur Mac Apple Silicon, la mémoire est unifiée CPU/GPU. Chaque sous-processus PyTorch qui charge un modèle YOLO sur MPS occupe ~500 Mo-1 Go selon le modèle. Pool size par défaut = nombre de modules à exécuter ; en pratique :

- 2 modules (PoC v1 PaveAudit + ChantierWatch) sur Mac M3 Pro 36 GB : aucun problème.
- 4-5 modules (Phase 2+) : surveiller via `mactop` ou `asitop` ; plafonner via `--pool-size` si swap silencieux observé.

Le mode `--no-parallel` reste disponible comme filet de sécurité.

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
| Pré-annotation open-vocab | Grounding DINO (HF `IDEA-Research/grounding-dino-tiny`) ET/OU YOLO-World (Ultralytics) | Bootstrap zéro-shot via prompts texte |
| Modèles base chaussée | RDD2022 + fine-tune Québec hivernal | Dataset ouvert + adaptation locale |
| Modèles base chantier | YOLOv8 COCO + fine-tune cônes/barrières/déblais QC | COCO partiellement, fine-tune léger |
| Format prompts | YAML versionnés Git par module | Source de vérité texte, pas dans le code |
| Orchestration plugins | Python entry points + Click CLI | Pattern standard, zéro framework |
| Parallélisation Stage 2 | `concurrent.futures.ProcessPoolExecutor` (stdlib) | Vrai parallélisme CPU+MPS, pickle pour passage inter-process |
| Géoréférencement | pyproj (EPSG:32188 NAD83 MTM zone 8), shapely | Cohérence avec SIG QC |
| Snap-to-road | **R-tree (rtree) + shapely** sur Géobase QC chargée en mémoire | Pas de service externe, pas de Docker, ~10 µs par requête |
| Format intermédiaires Stage 1 | Parquet (pyarrow) | Compact, typé, lecture rapide cross-process |
| Cache permits | GeoPackage local + R-tree | Sub-ms par requête |
| Stockage résultats | GeoPackage (SQLite) | OGC standard, lisible QGIS/ArcGIS/GDAL |
| Service web | pygeoapi | OGC API Features de référence |
| Outil annotation | FastAPI backend + SvelteKit frontend | Web local léger, raccourcis clavier ergonomiques |
| Rapport PDF | Quarto + Jinja2 templates | Templating moderne, livrable pro |
| Versioning code | Git + GitHub privé (v1) | Standard |
| Versioning poids ML | Manifest YAML manuel + dossier `models/` | Simple, suffit pour solo + side project |
| Versioning data | DVC vers bucket B2 (Phase 2) | Plus tard si besoin |
| Tests | pytest + golden frames annotées | Régression CV par évaluateur |

**Aucun service Docker en v1.** Le snap-to-road, qui aurait pu motiver OSRM/Valhalla, est implémenté en code Python via R-tree spatial sur la Géobase QC chargée en mémoire au démarrage du pipeline. Ce choix simplifie le déploiement (zéro service à orchestrer), accélère légèrement le pipeline (pas d'aller-retour HTTP), et reste cohérent avec le positionnement « 100% local, hors-ligne » du produit.

Pré-requis poste développeur et instructions d'installation : `docs/dependencies.md`.

---

## 5. Pipeline de captation iOS

### 5.1 Exigences fonctionnelles de l'app

L'app iPhone est un **device autonome** qui capture tout sur place et transfère post-session. Elle doit :

1. Capturer la **vidéo en 4K H.265** via `AVCaptureSession` avec :
   - `lockForConfiguration` + `setExposureMode(.locked)` (verrouillage exposition)
   - `setFocusMode(.locked)` (verrouillage AF)
   - `setWhiteBalanceMode(.locked)` (verrouillage WB)
   - **Lentille `builtInWideAngleCamera` pinned explicitement** (pas de virtual device qui peut basculer entre Wide et UltraWide selon les conditions de luminosité). Cela préserve la calibration intrinsèque.
   Les verrouillages se font après une scène de calibration courte (5-10 sec en début de session, scène typique du parcours).
2. Logger **GNSS** via `kCLLocationAccuracyBestForNavigation` à la fréquence native (typiquement 1 Hz, parfois 10 Hz).
3. Logger **IMU à 100 Hz** via `CMDeviceMotion` en mode `xMagneticNorthZVertical` — accéléromètres + gyroscope + magnétomètre. **Critique pour la branche inertielle de PaveAudit** : pas de drop de samples, pas de gap > 50 ms.
4. Logger le **cap vrai** via `CLHeading.trueHeading` (correction déclinaison magnétique automatique par iOS).
5. **Mount-check pré-session** : l'app vérifie via IMU que la caméra est dans une plage de pitch/roll acceptable (caméra horizontale ±10°, vue dégagée) et alerte sinon. Le **pitch et roll moyens de la session** sont enregistrés dans le manifest pour traçabilité.
6. **Saisie des métadonnées de session avant captation** :
   - Marque / modèle / année du véhicule (texte libre, suggestions auto)
   - Conditions météo brèves (clear / cloudy / rain / wet road / snow / mixed)
   - Notes libres optionnelles
   Ces métadonnées sont écrites dans `manifest.json` et utilisées pour calibration véhicule future + traçabilité.
7. Persister localement les flux dans le bundle de session (mp4 + jsonl), zéro réseau pendant la captation.
8. Demander les permissions : `NSLocationWhenInUseUsageDescription`, `NSMotionUsageDescription`, `NSCameraUsageDescription`, capability "Location updates" en background.
9. Transfert post-session : AirDrop, USB, ou montage SMB sur le Mac.
10. Déploiement : free-provisioning Apple ID + Xcode 16+ pour usage personnel et démos. Apple Developer Program (99 $/an) à partir de la première campagne de captation soutenue ou du premier partenaire externe à équiper (free-provisioning expire tous les 7 jours, irritant en usage continu).

### 5.2 Calibration

**Calibration intrinsèque caméra** : à faire **une fois** avant tout usage opérationnel, par modèle d'iPhone **et pour la lentille `builtInWideAngleCamera` exclusivement**.
- Damier d'échiquier 10×7, taille 25 mm.
- 25–40 images variées en 4K, AF verrouillé sur la même scène, lentille `builtInWideAngleCamera` pinned.
- OpenCV `calibrateCamera` → `K` (matrice intrinsèque) + `D` (distorsion).
- Critère d'acceptation : erreur de reprojection RMS < 0.5 pixel.
- Sauvegarde : `config/iphone_<model>_wide_calibration.yaml`, embarqué dans le manifest de chaque session.

**Pas de calibration extrinsèque mount** — sans objet pour l'évaluation d'état (pas besoin de positionnement précis 3D). En revanche le pitch et roll moyens de mount sont **enregistrés dans le manifest** pour traçabilité et pour permettre, en post-traitement, soit un filtrage soit une repondération si le mount sort des conditions nominales.

**Mount-check pré-session** : l'app vérifie via IMU que la caméra est dans une plage de pitch/roll acceptable (caméra horizontale ±10°, vue dégagée) et alerte sinon. La session ne démarre pas tant que le mount-check n'est pas validé.

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
  "mount": {
    "pitch_mean_deg": 2.3,
    "pitch_stdev_deg": 0.8,
    "roll_mean_deg": -0.5,
    "roll_stdev_deg": 0.3,
    "mount_check_passed": true
  },
  "vehicle": {
    "make": "Toyota",
    "model": "RAV4",
    "year": 2022,
    "notes": "véhicule personnel auteur"
  },
  "conditions": {
    "weather": "clear",
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
{"ts":"2026-04-27T14:30:00.123-04:00","lat":45.5012,"lon":-73.5673,"alt":42.1,"h_acc":3.2,"v_acc":5.1,"course":182.5,"speed_mps":13.4}
```

```jsonl
// imu.jsonl (un sample 100 Hz par ligne)
{"ts":"2026-04-27T14:30:00.012-04:00","quat":[0.123,0.456,0.789,0.012],"gyro":[0.01,0.02,0.03],"accel":[0.1,0.2,9.8],"user_accel":[0.1,0.2,0.0]}
```

Note : `accel` inclut la gravité ; `user_accel` est l'accélération utilisateur (gravité retirée par CoreMotion). La détection d'événements inertiels en Stage 1 utilise `user_accel` sur l'axe Z pour les bosses verticales.

```jsonl
// heading.jsonl (un sample par ligne)
{"ts":"2026-04-27T14:30:00.250-04:00","true_heading":182.3,"accuracy":5.0}
```

---

## 6. Pipeline de traitement Mac

### 6.1 Ingestion

`reperes ingest <session>` :
- Valide la structure du bundle (présence de tous les fichiers attendus, manifest cohérent, lentille pinned, mount-check passé).
- Logge un résumé : durée, distance parcourue (interpolée GNSS), qualité moyenne `horizontalAccuracy`, plage de vitesses, % de samples dans la plage exploitable inertiel (30-70 km/h).
- Crée le GeoPackage de session vide avec les tables communes.

L'ingestion est **légère** : elle valide et indexe, mais elle ne fait pas Stage 1 (extraction frames + interpolation pose + détection événements IMU). Stage 1 est déclenché par `reperes evaluate`.

### 6.2 Pré-annotation open-vocabulary

`reperes preannotate <session> --prompts prompts/<module>.yaml` :
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

`reperes annotate <session>` ouvre une UI web locale (FastAPI + SvelteKit, port 5001 par défaut) qui lit/écrit le même GeoPackage de session que les évaluateurs. Spec produit (workflow, raccourcis clavier, ergonomie cible) : voir PRD §7.3.

### 6.4 Évaluation — orchestration Stage 1/2/3

`reperes evaluate <session> --modules pave_audit,chantier_watch` exécute le pipeline en trois étages :

**Stage 1 — Préparation commune (séquentiel, ~10-15 min pour 1h de vidéo)**
1. Validation du bundle et du manifest (lentille pinned, mount-check passé).
2. Extraction des frames vidéo via ffmpeg à 1-2 fps → `prepared/frames_index.parquet` + images sur disque.
3. Interpolation pose GNSS+IMU par frame → `prepared/pose_per_frame.parquet`.
4. Détection des événements inertiels (filtrage passe-haut, peak detection, normalisation vitesse 30-70 km/h) → `prepared/imu_events.parquet`.
5. Chargement Géobase QC + construction R-tree en mémoire.

**Stage 2 — Évaluateurs en parallèle (`ProcessPoolExecutor`, durée = max des modules ~30-60 min)**

Pour chaque module, dans un sous-processus :
1. Réception de `PreparedStreams` (paths vers parquets)
2. Chargement du modèle custom fine-tuné (fallback sur pré-entraîné si absent)
3. Inférence frame-par-frame
4. Pour PaveAudit : exploitation supplémentaire des `imu_events` pour la branche inertielle
5. Snap-to-road via R-tree partagé (rtree + shapely sur Géobase)
6. Déduplication spatio-temporelle interne (rayon < 3 m, fenêtre < 5 sec)
7. Logique métier propre (calcul IES MTQ + score inertiel pour PaveAudit, cross-check permits pour ChantierWatch)
8. Écriture des layers dans `prepared/results_<module>.gpkg` (fichier dédié par module pour éviter la concurrence SQLite)

Flag `--no-parallel` : exécution série dans le process principal pour debug.

Flag `--pool-size N` : plafond le nombre de sous-processus simultanés (utile si > 3 modules ou si RAM tight).

**Stage 3 — Consolidation (séquentiel, < 1 min)**

1. Pour chaque module : copie atomique de ses layers depuis `prepared/results_<module>.gpkg` vers `assets.gpkg` (transaction par module).
2. Génération de `validation_report.md` (résumé numérique).
3. Logs structurés finalisés dans `logs/evaluate_<ts>.jsonl` avec durées par stage et par module, version code (commit Git court), version modèles utilisés.

### 6.5 Export labels et fine-tuning

`reperes export-labels <session> --format yolo --output datasets/qc_pave_v1/` :
- Lit le layer `verified_pave_audit` (gold labels uniquement, `verdict ∈ {confirmed, corrected, added_manually}`)
- Exporte au format YOLO ou COCO
- Maintient un index global `datasets/qc_pave_v1/index.yaml` : sessions incluses, frames count, classe distribution

`reperes train --module pave_audit --dataset datasets/qc_pave_v1/` :
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

`reperes report <session> --format pdf` génère un rapport (template Quarto + Jinja2) :
- Page 1 : sommaire exécutif (km parcourus, défauts par classe, zones chantier détectées, % unmatched, % de la session dans la plage de vitesse exploitable inertiel)
- Page 2-3 : carte du réseau colorée par **`ies_visual`**
- Page 4 : carte du réseau colorée par **`inertial_score`** (légende explicite : score relatif intra-session)
- Page 5 : top 20 segments les plus dégradés visuellement avec photo représentative
- Page 6 : top 20 segments les plus dégradés inertiellement, avec annotation des intersections (segments mauvais sur les deux scores)
- Page 7-8 : zones de chantier avec verdict permit + photos extraites
- Annexe : tableau complet, métadonnées, version pipeline (modèle utilisé + commit Git), véhicule de captation

`reperes serve` lance pygeoapi sur `http://localhost:5000` exposant les layers publics (segments, zones) ; les layers internes (candidates, frames_index, prepared) restent privés.

---

## 7. Modèle de données et livraison OGC

### 7.1 Schéma GeoPackage `assets.gpkg`

```
Tables communes:
  sessions              # toutes sessions captées (id, dates, device, conditions, vehicle, mount)
  frames_index          # frames par session (frame_idx, ts_iso, has_detection)

Tables PaveAudit (visuel):
  pavement_defects      # détections visuelles individuelles (POINT)
  pavement_segments     # segments routiers avec ies_visual ET inertial_score (LINESTRING)
  pavement_metadata     # version modèle, prompts, conditions, plage vitesse exploitable

Tables PaveAudit (inertiel):
  pavement_imu_events   # événements inertiels individuels (POINT, amplitude normalisée)

Tables ChantierWatch:
  construction_detections   # détections individuelles dédupliquées (POINT)
  construction_zones        # zones agrégées avec verdict (POLYGON)
  permits_reference         # cache permits utilisé (POLYGON, traçabilité)
  unmatched_zones           # vue filtrée des unmatched

Tables Annotation:
  candidates_<module>       # sortie pré-annotation par module
  verified_<module>         # gold labels post-révision humaine
  annotations_log           # historique audit trail
```

### 7.2 Schéma `pavement_segments` (avec deux scores)

```sql
CREATE TABLE pavement_segments (
  segment_id           INTEGER PRIMARY KEY,
  geom                 LINESTRING NOT NULL,        -- WGS84 (EPSG:4326)
  geom_mtm             LINESTRING,                  -- NAD83 MTM zone 8 pour SIG QC
  street_name          TEXT,
  start_chainage_m     REAL,
  length_m             REAL NOT NULL,
  
  -- Branche visuelle (méthodologie MTQ)
  ies_visual           REAL,                        -- 0-100, méthodologie MTQ
  ies_classification   TEXT,                        -- 'Bon' | 'Acceptable' | 'Mauvais' | 'Très mauvais'
  n_defects_total      INTEGER,
  n_defects_by_class   JSON,                        -- {"pothole": 3, "long_crack": 7, ...}
  
  -- Branche inertielle (proxy IRI)
  inertial_score       REAL,                        -- 0-100, RELATIF intra-session
  inertial_severity    TEXT,                        -- 'low' | 'medium' | 'high' | 'unavailable'
  n_imu_events         INTEGER,                     -- nb événements détectés sur le segment
  imu_events_by_severity JSON,                      -- {"faible": 2, "modérée": 5, "sévère": 1}
  pct_in_speed_range   REAL,                        -- % de la durée segment dans 30-70 km/h
  
  session_id           TEXT NOT NULL,
  evaluated_at         TIMESTAMP NOT NULL,
  model_version        TEXT NOT NULL,               -- ex. "pave_audit_v2"
  vehicle_signature    TEXT                         -- ex. "Toyota RAV4 2022" pour traçabilité
);
```

### 7.3 Schéma `pavement_imu_events`

```sql
CREATE TABLE pavement_imu_events (
  event_id             INTEGER PRIMARY KEY,
  geom                 POINT NOT NULL,              -- WGS84, position GNSS interpolée
  ts                   TIMESTAMP NOT NULL,          -- timestamp précis du pic
  accel_z_raw          REAL NOT NULL,               -- amplitude brute (m/s²)
  accel_z_normalized   REAL NOT NULL,               -- amplitude après normalisation vitesse
  speed_mps            REAL NOT NULL,               -- vitesse instantanée GNSS
  severity_class       TEXT NOT NULL,               -- 'faible' | 'modérée' | 'sévère'
  segment_id           INTEGER,                     -- référence vers pavement_segments
  session_id           TEXT NOT NULL
);
```

### 7.4 Schéma `construction_zones`

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

### 7.5 Configuration pygeoapi

```yaml
# pygeoapi-config.yaml
server:
  bind: { host: 0.0.0.0, port: 5000 }
  url: http://localhost:5000

resources:
  pavement_segments:
    type: collection
    title: Segments de chaussée évalués (PaveAudit)
    description: État de chaussée — score visuel MTQ et score inertiel proxy IRI
    keywords: [chaussée, IES, MTQ, audit, inertiel]
    crs: [http://www.opengis.net/def/crs/EPSG/0/4326]
    providers:
      - type: feature
        name: GeoPackage
        data: /path/to/assets.gpkg
        id_field: segment_id
        table: pavement_segments
  
  pavement_imu_events:
    type: collection
    title: Événements inertiels (PaveAudit)
    description: Pics d'accélération verticale géoréférencés, normalisés vitesse
    providers:
      - type: feature
        name: GeoPackage
        data: /path/to/assets.gpkg
        id_field: event_id
        table: pavement_imu_events
  
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

### 7.6 Endpoints exposés

- `GET /collections/pavement_segments/items?bbox=...&ies_visual__lt=50` — filtrage par BBox + score visuel
- `GET /collections/pavement_segments/items?inertial_score__gt=70` — segments à signal inertiel élevé
- `GET /collections/pavement_segments/items/{segment_id}` — segment individuel (les deux scores)
- `GET /collections/pavement_imu_events/items?bbox=...&severity_class=sévère` — événements inertiels filtrés
- `GET /collections/construction_zones/items?verdict=unmatched` — zones flaggées
- `GET /collections/construction_zones/items?f=html` — UI HTML simple intégrée

### 7.7 Consommation cible

- **QGIS** : ajout de couche WFS / OGC API Features → consommation native, avec layers `pavement_segments`, `pavement_imu_events`, `construction_zones`
- **Curl / scripts** : `curl http://localhost:5000/collections/pavement_segments/items?bbox=... > export.geojson`
- **Browser** : interface HTML générée automatiquement par pygeoapi
