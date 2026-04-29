# Dépendances poste de travail — Repères (v1 fail-fast)

**Date** : 2026-04-29
**Statut** : proposition initiale, à valider en Phase 0
**Cible** : poste de développement et d'exécution du pipeline (MacBook M-series)
**Scope** : v1 PaveAudit visuel, monolithique séquentiel, 100 % local

---

## 1. Philosophie d'installation

> **Tout natif en v1. Aucun service externe, aucun Docker, aucun Node.js, aucun pygeoapi.**

Trois principes structurent les choix :

1. **PyTorch MPS doit tourner natif.** L'accélération Apple Silicon (Metal Performance Shaders) ne traverse pas Docker sur Mac. Un pipeline CV containerisé tomberait en CPU pur, soit ~10× plus lent. Donc le pipeline Python tourne dans un `venv` natif.
2. **Snap-to-road en code Python via R-tree.** Plutôt que d'installer un service de routing/map-matching externe (OSRM, Valhalla), Repères implémente le snap-to-road en code, en chargeant OSM Québec dans une structure R-tree en mémoire.
3. **Annotation via Label Studio installé Python.** Pas d'UI custom en v1 — donc pas de Node, pas de SvelteKit, pas de Vite.

Conséquence pratique : **aucun service Docker en v1**, tout en natif via Homebrew + uv. Le frontend SvelteKit, pygeoapi, scipy (peak detection IMU), seront ajoutés en Phase 2 quand ChantierWatch / branche inertielle / OGC API deviendront pertinents.

---

## 2. Pré-requis matériel

| Élément | Minimum | Recommandé | Justification |
|---|---|---|---|
| Mac | M1 Pro 16 GB | M3 Pro/Max 36 GB+ | Inférence YOLO/Grounding DINO sur MPS, batches confortables |
| Disque libre | 200 GB | 500 GB+ | Sessions vidéo 4K H.265 ≈ 5-8 GB / heure ; modèles + datasets ≈ 50 GB |
| iPhone | 16 Pro / 16 Pro Max | idem | Caméra 4K HDR, GNSS bi-bande, IMU haute fréquence |
| macOS | 14 (Sonoma) | 15+ (Sequoia) | Compatibilité Xcode 16, PyTorch MPS récent |

---

## 3. Mac host — outils système

Installés une seule fois via Homebrew. Le pipeline Python s'attend à ce qu'ils soient sur le `PATH`.

```bash
# Homebrew lui-même
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Outils CLI v1
brew install \
  git \
  ffmpeg \
  gdal \
  uv \
  direnv
```

| Paquet | Usage | Notes |
|---|---|---|
| `git` | Versioning code, prompts, manifest modèles, scoring auteur | — |
| `ffmpeg` | Extraction frames + métadonnées timestamp depuis MP4 H.265 | obligatoire pour pipeline |
| `gdal` | CLI `ogr2ogr` / `ogrinfo` pour conversion OSM → GeoPackage et debug | obligatoire pour setup OSM |
| `uv` | Gestionnaire Python tout-en-un : versions Python, environnements, dépendances | remplace `pyenv` + `venv` + `pip` |
| `direnv` | Chargement `.envrc` automatique au cd dans le repo | pratique pour variables d'environnement |

**Pas installés en v1** (Phase 2 si pertinent) : `node@20` (UI annotation custom), `quarto` (rapport PDF), `colima` ou Docker (PostGIS / synthèse Mitsuba).

---

## 4. Outils développement iOS

Installés via App Store, pas Homebrew.

| Outil | Version | Usage |
|---|---|---|
| Xcode | 16+ | Build app SwiftUI, signature free-provisioning |
| Apple ID personnel | — | **Free-provisioning** : déploiement initial sur l'iPhone personnel de l'auteur. Approche v1. Le profil expire tous les 7 jours mais peut être renouvelé via redéploiement Xcode. |

L'app n'a pas de dépendances Swift externes au démarrage (AVFoundation, CoreLocation, CoreMotion sont natifs).

**Apple Developer Program (99 $/an)** : à acheter avant la grosse campagne d'annotation Phase 4 si redéploiement répété devient un irritant (cf. PRD D-06). Reportable au-delà de Phase 4 si free-provisioning suffit.

---

## 5. Pipeline Python (natif, venv)

### 5.1 Création de l'environnement

`uv` gère Python lui-même (pas besoin de `pyenv`), crée le venv, et résout les dépendances en parallèle.

```bash
cd ~/src/reperes
uv python install 3.12        # installe l'interpréteur si absent
uv venv --python 3.12          # crée .venv/
source .venv/bin/activate
uv sync --extra dev            # installe deps depuis pyproject.toml + uv.lock
```

Workflow d'ajout de dépendance (au lieu de `pip install`) :

```bash
uv add ultralytics             # runtime
uv add --dev pytest-cov        # dev only
uv lock --upgrade-package torch  # bump ciblé
```

Le fichier `uv.lock` est commité au repo : il garantit la reproductibilité du résolveur sur tous les postes.

### 5.2 Dépendances par couche v1

À déclarer dans `pyproject.toml` (sous `[project.dependencies]` et `[project.optional-dependencies.dev]`). Liste indicative — versions à figer via `uv lock` au moment du setup.

**Core CLI**
```
click>=8.1
pydantic>=2.6
pyyaml>=6.0
rich>=13.7              # logs lisibles
```

**ML & vision**
```
torch>=2.5              # MPS support
torchvision>=0.20
ultralytics>=8.3        # YOLOv8/v11 + Label Studio export interop
transformers>=4.45      # Grounding DINO via HF
huggingface_hub>=0.25
opencv-python>=4.10
pillow>=10.4
```

**Géospatial**
```
pyproj>=3.7             # EPSG:32188 NAD83 MTM zone 8 + EPSG:4326
shapely>=2.0            # géométrie + snap-to-road
geopandas>=1.0
fiona>=1.10             # IO GeoPackage
rtree>=1.3              # spatial index OSM
numpy>=2.0
pandas>=2.2
pyarrow>=18.0           # parquet pour pose interpolée
```

**Annotation**
```
label-studio>=1.13      # outil annotation v1
```

**Tests & qualité**
```
pytest>=8.3
pytest-cov>=5.0
ruff>=0.7               # linter + formatter
mypy>=1.13              # typage statique
```

**Différé Phase 2 (ne pas installer en v1)**
```
scipy                   # peak detection IMU pour branche inertielle
pygeoapi                # service OGC API Features
fastapi                 # backend annotation custom (uniquement si Label Studio bloquant)
uvicorn                 # idem
jinja2                  # templating rapports Quarto
dvc[s3]                 # versioning data vers bucket B2
```

### 5.3 Validation post-install

```bash
# Vérifier MPS disponible
python -c "import torch; print('MPS:', torch.backends.mps.is_available())"
# → MPS: True

# Vérifier Ultralytics + MPS
python -c "from ultralytics import YOLO; m=YOLO('yolov8n.pt'); print(m.predict('https://ultralytics.com/images/bus.jpg', device='mps')[0].boxes)"

# Vérifier Label Studio démarrable
label-studio start --no-browser &
curl -sS http://localhost:8080/health
kill %1
```

---

## 6. Annotation humaine

L'annotation utilise **Label Studio installé en Python** (cf. `pyproject.toml`). Démarrage local :

```bash
label-studio start
# UI sur http://localhost:8080
```

Pas de frontend SvelteKit / FastAPI custom en v1 (différé Phase 2).

Workflow :
1. `reperes preannotate <session>` produit `predictions.jsonl` au format Label Studio.
2. Import dans Label Studio (création de projet avec config bbox + classes MTQ).
3. Révision humaine, raccourcis clavier natifs.
4. Export YOLO depuis Label Studio.
5. `reperes import-labels <session> --from labels.zip` ré-importe vers GeoPackage.

---

## 7. Référentiel routier

**v1 = OpenStreetMap Quebec via Geofabrik.** La Géobase QC officielle est reportée Phase 2 (inscription requise, pertinente seulement si livraison directe aux SIG municipaux avec IDs cohérents).

```bash
mkdir -p data/road_network && cd data/road_network

# Téléchargement OSM Québec
curl -O https://download.geofabrik.de/north-america/canada/quebec-latest.osm.pbf

# Conversion en GeoPackage pour rues uniquement
ogr2ogr -f GPKG osm_quebec.gpkg quebec-latest.osm.pbf lines \
  -where "highway IN ('primary', 'secondary', 'tertiary', 'residential', 'unclassified', 'service', 'trunk', 'motorway')" \
  -nln roads

# Cleanup
rm quebec-latest.osm.pbf
```

**Empreinte disque** : OSM extrait Québec ≈ 800 MB compressé, ~2-3 GB en GeoPackage filtré. Reste sous 5 GB total.

---

## 8. Variables d'environnement et configuration locale

Fichier `.envrc` (à charger via `direnv`) :

```bash
# .envrc
export REPERES_DATA_DIR="$HOME/reperes-data"        # hors repo, sessions volumineuses
export REPERES_MODELS_DIR="$REPERES_DATA_DIR/models"
export REPERES_ROAD_NETWORK="$REPERES_DATA_DIR/road_network/osm_quebec.gpkg"
export PYTORCH_ENABLE_MPS_FALLBACK=1                # fallback CPU sur ops non-MPS
export HF_HOME="$REPERES_DATA_DIR/.huggingface"     # cache modèles HF
```

Les `sessions/` et `data/` ne vivent **pas** dans le repo Git (volume + données potentiellement sensibles avant floutage Phase 2). Le repo contient uniquement code, prompts YAML, manifest modèles, et rubrique de scoring.

---

## 9. Récapitulatif install — chemin rapide

Pour un poste neuf, dans l'ordre :

```bash
# 1. Homebrew + outils système v1
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install git ffmpeg gdal uv direnv

# 2. Xcode depuis App Store (manuel)

# 3. Repo + Python venv
git clone <repo> ~/src/reperes && cd ~/src/reperes
uv python install 3.12
uv venv --python 3.12 && source .venv/bin/activate
uv sync --extra dev

# 4. Validation MPS
python -c "import torch; assert torch.backends.mps.is_available()"

# 5. Référentiel routier (téléchargement one-shot)
bash scripts/setup_road_network.sh

# 6. Validation Label Studio
label-studio start --no-browser &
sleep 3 && curl -sS http://localhost:8080/health && kill %1

# 7. Test end-to-end sur session-fixture
reperes ingest tests/fixtures/sample_session/
```

---

## 10. Ce qui est explicitement *différé*

Pour rester focus PoC v1 fail-fast, **ne pas installer maintenant** :

| Outil | Pourquoi différer | Quand l'ajouter |
|---|---|---|
| Docker / Colima | Aucun service externe en v1 (snap-to-road en code) | Si Phase 2 introduit PostGIS, Mitsuba, ou besoin de packaging service-orienté |
| Node.js + SvelteKit | Pas d'UI annotation custom en v1 (Label Studio suffit) | Phase 2 si Label Studio se révèle bloquant |
| pygeoapi | Pas de service OGC en v1 (export GeoPackage suffit) | Phase 2 si livraison aux SIG municipaux le justifie |
| Quarto | Pas de rapport PDF en v1 (`validation_report.md` Markdown suffit) | Phase 2 pour livrable client formel |
| scipy | Pas de branche inertielle en v1 (IMU loggé, non exploité) | Phase 2 (branche inertielle PaveAudit) |
| Géobase QC officielle | OSM suffit pour snap-to-road v1 | Phase 2 si cohérence IDs SIG municipaux requise |
| DVC + bucket B2 | Pas de besoin de versioning data tant que dataset < 5 GB | Phase 2, quand dataset Quebec dépasse 10 k frames annotées |
| PostGIS | GeoPackage suffit en mono-utilisateur | Si pivot SaaS multi-tenant |
| MLflow / W&B | Manifest YAML manuel suffit pour solo | Si > 1 contributeur sur fine-tune |
| Floutage (egolossas, OpenDP) | Pas de partage externe en v1 | Avant tout livrable client externe |
| GitHub Actions / CI | Pas de collab, tests locaux suffisent | Quand un partenaire commence à pusher |
| Apple Developer Program 99 $/an | Free-provisioning suffit pour Phase 0-3 | Avant Phase 4 si redéploiement fréquent (cf. PRD D-06) |

---

## 11. Questions ouvertes

- **Backup sessions** : Time Machine sur les sessions ? Ou exclusion + backup ciblé vers B2 ? À trancher avant la première vraie campagne de captation Phase 4.
- **Stockage sessions** : disque interne suffit pour Phase 0-4 (3-5 sessions × ~30 km × ~5 GB/h ≈ 30-50 GB). Externe NVMe à prévoir si Phase 2+ croît.
- **Label Studio config** : projet pré-configuré (bbox + classes MTQ + sévérités) à committer comme template `validation/label_studio_pave_audit.xml` pour reproductibilité.
