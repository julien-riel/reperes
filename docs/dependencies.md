# Dépendances poste de travail — Repères

**Date** : 2026-04-28
**Statut** : proposition initiale, à valider
**Cible** : poste de développement et d'exécution du pipeline (MacBook M-series)

---

## 1. Philosophie d'installation

> **Tout natif en v1. Aucun service externe, aucun Docker.**

Trois principes structurent les choix :

1. **PyTorch MPS doit tourner natif.** L'accélération Apple Silicon (Metal Performance Shaders) ne traverse pas Docker sur Mac. Un pipeline CV containerisé tomberait en CPU pur, soit ~10× plus lent. Donc le pipeline Python tourne dans un `venv` natif.
2. **Snap-to-road en code Python via R-tree.** Plutôt que d'installer un service de routing/map-matching externe (OSRM, Valhalla), Repères implémente le snap-to-road en code, en chargeant la Géobase Québec dans une structure R-tree en mémoire. Justification : (a) le besoin est un nearest-segment, pas du routing complet ; (b) la précision GNSS bi-bande de l'iPhone 16 Pro rend le bruit GPS modeste ; (c) zéro dépendance externe simplifie le déploiement.
3. **Le frontend d'annotation tourne natif.** Node + Vite + SvelteKit s'itèrent plus vite en dev natif (HMR, debug navigateur). Pas de gain à containeriser un outil mono-utilisateur localhost.

Conséquence pratique : **aucun service Docker en v1**, tout en natif via Homebrew + uv + npm. Si Docker devient nécessaire en Phase 2 (PostGIS pour SaaS multi-utilisateur, Mitsuba pour synthèse de défauts, etc.), il sera ré-introduit à ce moment-là.

---

## 2. Pré-requis matériel

| Élément | Minimum | Recommandé | Justification |
|---|---|---|---|
| Mac | M1 Pro 16 GB | M3 Pro/Max 36 GB+ | Inférence YOLO/Grounding DINO sur MPS, batches confortables |
| Disque libre | 200 GB | 500 GB+ | Sessions vidéo 4K H.265 ≈ 5-8 GB / heure ; modèles + datasets ≈ 50 GB |
| iPhone | 16 Pro / 16 Pro Max | idem | Caméra 4K HDR ProRes, GNSS bi-bande, IMU haute fréquence |
| macOS | 14 (Sonoma) | 15+ (Sequoia) | Compatibilité Xcode 16, PyTorch MPS récent |

---

## 3. Mac host — outils système

Installés une seule fois via Homebrew. Le pipeline Python s'attend à ce qu'ils soient sur le `PATH`.

```bash
# Homebrew lui-même
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Outils CLI
brew install \
  git \
  ffmpeg \
  gdal \
  uv \
  node@20 \
  quarto \
  direnv
```

| Paquet | Usage | Notes |
|---|---|---|
| `git` | Versioning code, prompts, manifest modèles | — |
| `ffmpeg` | Extraction frames + métadonnées timestamp depuis MP4 H.265 | obligatoire pour Stage 1 |
| `gdal` | CLI `ogr2ogr` / `ogrinfo` pour debug GeoPackage hors Python | optionnel mais utile |
| `uv` | Gestionnaire Python tout-en-un : versions Python, environnements, dépendances | remplace `pyenv` + `venv` + `pip` |
| `node@20` | Build SvelteKit (outil annotation) | Node 22 LTS quand stable |
| `quarto` | Génération rapports PDF clients | LaTeX direct, mais plus verbeux |
| `direnv` | Chargement `.envrc` automatique au cd dans le repo | pratique pour variables d'environnement |

**Pas de Docker requis en v1.** Si Phase 2 introduit PostGIS ou autre, on l'ajoutera à ce moment-là (Colima est l'alternative open-source à Docker Desktop sur Mac, à privilégier pour usage commercial sans licence).

---

## 4. Outils développement iOS

Installés via App Store, pas Homebrew.

| Outil | Version | Usage |
|---|---|---|
| Xcode | 16+ | Build app SwiftUI, signature free-provisioning |
| Apple ID personnel | — | Free-provisioning (7 jours, renouvelable) — suffit pour dev solo et premières démos courtes |
| Apple Developer Program | 99 $/an | **À prévoir** : dès la première campagne de captation soutenue (Phase 7) ou dès qu'un partenaire externe doit installer l'app. Free-provisioning expire tous les 7 jours, ce qui devient un irritant majeur en captation régulière (rebuild + redéploiement avant chaque session). |

L'app n'a pas de dépendances Swift externes au démarrage (AVFoundation, CoreLocation, CoreMotion sont natifs). Si Swift Package Manager devient nécessaire (ex. logging), géré dans Xcode directement.

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
```Workflow d'ajout de dépendance (au lieu de `pip install`) :

```bash
uv add ultralytics             # runtime
uv add --dev pytest-cov        # dev only
uv lock --upgrade-package torch  # bump ciblé
```

Le fichier `uv.lock` est commité au repo : il garantit la reproductibilité du résolveur sur tous les postes.

### 5.2 Dépendances par couche

À déclarer dans `pyproject.toml` (sous `[project.dependencies]` et `[project.optional-dependencies.dev]`). Liste indicative — versions à figer via `uv lock` au moment du setup.

**Core CLI & orchestration**
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
ultralytics>=8.3        # YOLOv8/v11 + YOLO-World
transformers>=4.45      # Grounding DINO via HF
huggingface_hub>=0.25
opencv-python>=4.10
pillow>=10.4
```

**Géospatial & traitement signal**
```
pyproj>=3.7             # EPSG:32188 NAD83 MTM zone 8
shapely>=2.0            # géométrie + snap-to-road
geopandas>=1.0
fiona>=1.10             # IO GeoPackage
rtree>=1.3              # spatial index Géobase + permits cache
scipy>=1.14             # peak detection inertiel, filtrage signal
numpy>=2.0
pandas>=2.2
pyarrow>=18.0           # parquet pour artefacts Stage 1 (PreparedStreams)
```

**Service web & livraison**
```
pygeoapi>=0.18          # OGC API Features
fastapi>=0.115          # backend annotation
uvicorn>=0.32
jinja2>=3.1             # templating rapports + pygeoapi
```

**Tests & qualité**
```
pytest>=8.3
pytest-cov>=5.0
ruff>=0.7               # linter + formatter
mypy>=1.13              # typage statique
```

**Phase 2 (différé, ne pas installer en Phase 0)**
```
dvc[s3]                 # versioning data vers bucket B2
mitsuba                 # synthèse de défauts si dataset trop pauvre
```

### 5.3 Validation post-install

```bash
# Vérifier MPS disponible
python -c "import torch; print('MPS:', torch.backends.mps.is_available())"
# → MPS: True

# Vérifier Ultralytics + MPS
python -c "from ultralytics import YOLO; m=YOLO('yolov8n.pt'); print(m.predict('https://ultralytics.com/images/bus.jpg', device='mps')[0].boxes)"
```

---

## 6. Frontend annotation (Node, natif)

Localisé dans `apps/annotate-ui/`. Stack SvelteKit + TypeScript.

```bash
cd apps/annotate-ui
npm install
npm run dev    # http://localhost:5001
```

Dépendances clés (déclarées dans `package.json`) :
- `@sveltejs/kit` — framework
- `vite` — bundler
- `tailwindcss` — styling rapide
- `openseadragon` ou `leaflet` — viewer images/cartes (à choisir Phase 3)

Le backend FastAPI sert l'API REST et lit/écrit le GeoPackage de session ; le frontend consomme cette API. Aucune base de données séparée.

---

## 7. Services externes

**Aucun en v1.** Le snap-to-road est implémenté en code Python (rtree + shapely sur Géobase QC chargée en mémoire au démarrage du pipeline). Le service web OGC (pygeoapi) tourne en process Python natif. Le backend annotation (FastAPI) idem.

Données de référence à placer dans `data/road_network/` (téléchargement manuel one-shot) :

```bash
mkdir -p data/road_network && cd data/road_network

# Option A — Géobase officielle Québec (préférée pour cohérence avec SIG QC)
# Téléchargement depuis https://www.donneesquebec.ca/recherche/dataset/adresses-quebec
# (chemin exact à confirmer Phase 0)

# Option B — Extrait OSM Québec (fallback ou complément)
curl -O https://download.geofabrik.de/north-america/canada/quebec-latest.osm.pbf
# Conversion en GeoPackage via ogr2ogr si besoin
ogr2ogr -f GPKG osm_qc_supplement.gpkg quebec-latest.osm.pbf lines -where "highway IS NOT NULL"
```

**Empreinte disque** : Géobase QC ≈ 200-500 MB ; OSM extrait Québec ≈ 800 MB. Total < 1.5 GB.

**Si Phase 2 introduit Docker** (PostGIS pour SaaS multi-utilisateur, Mitsuba pour synthèse de défauts, etc.), on ajoutera Docker à ce moment-là. Sur Mac, l'alternative open-source à Docker Desktop est `colima` (`brew install colima`), à privilégier pour usage commercial sans licence.

---

## 8. Variables d'environnement et configuration locale

Fichier `.envrc` (à charger via `direnv` — `brew install direnv`) ou `.env` selon la préférence.

```bash
# .envrc
export REPERES_DATA_DIR="$HOME/reperes-data"        # hors repo, sessions volumineuses
export REPERES_MODELS_DIR="$REPERES_DATA_DIR/models"
export REPERES_ROAD_NETWORK="$REPERES_DATA_DIR/road_network/geobase_qc.gpkg"
export PYTORCH_ENABLE_MPS_FALLBACK=1                # fallback CPU sur ops non-MPS
export HF_HOME="$REPERES_DATA_DIR/.huggingface"     # cache modèles HF
```

Les `sessions/` et `data/` ne vivent **pas** dans le repo Git (volume + données potentiellement sensibles avant floutage Phase 2). Le repo contient uniquement code, prompts YAML, et manifest modèles.

---

## 9. Récapitulatif install — chemin rapide

Pour un poste neuf, dans l'ordre :

```bash
# 1. Homebrew + outils système (~10 min)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install git ffmpeg gdal uv node@20 quarto direnv

# 2. Xcode (~30 min, depuis App Store, manuel)

# 3. Repo + Python venv (~5 min)
git clone <repo> ~/src/reperes && cd ~/src/reperes
uv python install 3.12
uv venv --python 3.12 && source .venv/bin/activate
uv sync --extra dev

# 4. Validation MPS (~30 sec)
python -c "import torch; assert torch.backends.mps.is_available()"

# 5. Frontend annotation (~3 min)
cd apps/annotate-ui && npm install && cd -

# 6. Référentiel routier (~5 min, téléchargement one-shot)
bash scripts/setup_road_network.sh

# 7. Test end-to-end sur session-fixture
reperes ingest tests/fixtures/sample_session/
```

**Temps total install poste neuf** : ~1 h hors temps Xcode (essentiellement download/install des paquets).

---

## 10. Ce qui est explicitement *différé*

Pour rester focus PoC v1, **ne pas installer maintenant** :

| Outil | Pourquoi différer | Quand l'ajouter |
|---|---|---|
| Docker / Colima | Aucun service externe en v1 (snap-to-road en code) | Si Phase 2 introduit PostGIS, Mitsuba, ou besoin de packaging service-orienté |
| DVC + bucket B2 | Pas de besoin de versioning data tant que dataset < 5 GB | Phase 2, quand dataset Quebec dépasse 10 k frames annotées |
| PostGIS | GeoPackage suffit en mono-utilisateur ; PostGIS arrive si SaaS | Si pivot SaaS multi-tenant |
| MLflow / W&B | Manifest YAML manuel suffit pour solo | Si > 1 contributeur sur fine-tune |
| Floutage (egolossas, OpenDP) | Pas de partage externe en Phase 0-1 | Avant tout livrable client externe |
| GitHub Actions / CI | Pas de collab, tests locaux suffisent | Quand un partenaire commence à pusher |
| Apple Developer Program 99 $/an | Free-provisioning suffit pour dev solo | Première campagne de captation soutenue ou premier partenaire externe à équiper |

---

## 11. Questions ouvertes

- **Géobase officielle vs OSM Québec** : la Géobase QC officielle est préférable pour la cohérence avec les SIG municipaux (mêmes IDs de tronçons que les bases internes des villes), mais demande inscription / demande d'accès. OSM est immédiatement téléchargeable mais ses IDs ne matchent pas ceux des bases municipales. Décision à prendre Phase 2 (ingestion). Approche probable : OSM en fallback automatique si Géobase manquante pour une zone donnée.
- **Stockage sessions** : disque interne suffit pour Phase 0-1 (3 zones × ~30 km × ~5 GB/h ≈ 30 GB). Externe NVMe à prévoir Phase 2 quand le volume grossit.
- **Backup** : Time Machine sur les sessions ? Ou exclusion + backup ciblé vers B2 ? À trancher avant la première vraie campagne de captation.
