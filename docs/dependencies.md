# Dépendances poste de travail — wsmd

**Date** : 2026-04-27
**Statut** : proposition initiale, à valider
**Cible** : poste de développement et d'exécution du pipeline (MacBook M-series)

---

## 1. Philosophie d'installation

> **Natif quand l'accélération matérielle compte. Docker quand le binaire est complexe à compiler.**

Trois principes structurent les choix :

1. **PyTorch MPS doit tourner natif.** L'accélération Apple Silicon (Metal Performance Shaders) ne traverse pas Docker sur Mac. Un pipeline CV containerisé tomberait en CPU pur, soit ~10× plus lent. Donc le pipeline Python tourne dans un `venv` natif.
2. **OSRM tourne en Docker.** C'est un service C++ avec build complexe (Boost, LuaJIT, etc.). L'image officielle `osrm/osrm-backend` est le mode d'installation standard. Aucun avantage à compiler natif.
3. **Le frontend d'annotation tourne natif.** Node + Vite + SvelteKit s'itèrent plus vite en dev natif (HMR, debug navigateur). Pas de gain à containeriser un outil mono-utilisateur localhost.

Conséquence pratique : **un seul service Docker** (OSRM), tout le reste en natif via Homebrew + uv + npm.

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
  docker \
  docker-compose
```

| Paquet | Usage | Notes |
|---|---|---|
| `git` | Versioning code, prompts, manifest modèles | — |
| `ffmpeg` | Extraction frames + métadonnées timestamp depuis MP4 H.265 | obligatoire |
| `gdal` | CLI `ogr2ogr` / `ogrinfo` pour debug GeoPackage hors Python | optionnel mais utile |
| `uv` | Gestionnaire Python tout-en-un : versions Python, environnements, dépendances | remplace `pyenv` + `venv` + `pip` |
| `node@20` | Build SvelteKit (outil annotation) | Node 22 LTS quand stable |
| `quarto` | Génération rapports PDF clients | LaTeX direct, mais plus verbeux |
| `docker` + `docker-compose` | Service OSRM | Colima si refus Docker Desktop |

**Note Docker Desktop** : sur Mac, alternative open-source = `colima` (`brew install colima`). Pour OSRM seul, les deux fonctionnent. Docker Desktop est plus simple ; Colima évite la licence pro.

---

## 4. Outils développement iOS

Installés via App Store, pas Homebrew.

| Outil | Version | Usage |
|---|---|---|
| Xcode | 16+ | Build app SwiftUI, signature free-provisioning |
| Apple ID personnel | — | Free-provisioning (7 jours, renouvelable) — suffit pour dev solo et démos |
| Apple Developer Program | 99 $/an | **Différé** : seulement quand un partenaire externe doit installer l'app |

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
```

Workflow d'ajout de dépendance (au lieu de `pip install`) :

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

**Géospatial**
```
pyproj>=3.7             # EPSG:32188 NAD83 MTM zone 8
shapely>=2.0
geopandas>=1.0
fiona>=1.10             # IO GeoPackage
rtree>=1.3              # spatial index permits cache
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

## 7. Services Docker

Un unique service : OSRM pour le snap-to-road. Déclaré dans `docker/compose.yaml`.

```yaml
# docker/compose.yaml
services:
  osrm:
    image: osrm/osrm-backend:latest
    container_name: wsmd-osrm
    ports:
      - "5500:5000"
    volumes:
      - ../data/osrm:/data
    command: osrm-routed --algorithm mld /data/quebec-latest.osrm
    restart: unless-stopped
```

**Préparation des données OSRM (one-shot, ~1 h pour Québec)** :

```bash
mkdir -p data/osrm && cd data/osrm
curl -O https://download.geofabrik.de/north-america/canada/quebec-latest.osm.pbf
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-extract -p /opt/car.lua /data/quebec-latest.osm.pbf
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-partition /data/quebec-latest.osrm
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-customize /data/quebec-latest.osrm
```

**Lancement quotidien** :
```bash
docker compose -f docker/compose.yaml up -d osrm
# Vérification : curl http://localhost:5500/nearest/v1/driving/-73.5673,45.5012
```

**Empreinte disque** : extrait Québec OSM ≈ 800 MB ; fichiers OSRM dérivés ≈ 4-6 GB.

---

## 8. Variables d'environnement et configuration locale

Fichier `.envrc` (à charger via `direnv` — `brew install direnv`) ou `.env` selon la préférence.

```bash
# .envrc
export WSMD_DATA_DIR="$HOME/wsmd-data"          # hors repo, sessions volumineuses
export WSMD_MODELS_DIR="$WSMD_DATA_DIR/models"
export OSRM_URL="http://localhost:5500"
export PYTORCH_ENABLE_MPS_FALLBACK=1            # fallback CPU sur ops non-MPS
export HF_HOME="$WSMD_DATA_DIR/.huggingface"    # cache modèles HF
```

Les `sessions/` et `data/` ne vivent **pas** dans le repo Git (volume + données potentiellement sensibles avant floutage Phase 2). Le repo contient uniquement code, prompts YAML, et manifest modèles.

---

## 9. Récapitulatif install — chemin rapide

Pour un poste neuf, dans l'ordre :

```bash
# 1. Homebrew + outils système (~10 min)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install git ffmpeg gdal uv node@20 quarto docker docker-compose direnv

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

# 6. OSRM data prep (~1 h, à lancer en arrière-plan)
bash scripts/setup_osrm.sh

# 7. Test end-to-end sur session-fixture
wsmd ingest tests/fixtures/sample_session/
```

**Temps total install poste neuf** : ~2 h dont 1 h de download/build OSRM en arrière-plan.

---

## 10. Ce qui est explicitement *différé*

Pour rester focus PoC v1, **ne pas installer maintenant** :

| Outil | Pourquoi différer | Quand l'ajouter |
|---|---|---|
| DVC + bucket B2 | Pas de besoin de versioning data tant que dataset < 5 GB | Phase 2, quand dataset Quebec dépasse 10 k frames annotées |
| PostGIS | GeoPackage suffit en mono-utilisateur ; PostGIS arrive si SaaS | Si pivot SaaS multi-tenant |
| MLflow / W&B | Manifest YAML manuel suffit pour solo | Si > 1 contributeur sur fine-tune |
| Floutage (egolossas, OpenDP) | Pas de partage externe en Phase 0-1 | Avant tout livrable client externe |
| GitHub Actions / CI | Pas de collab, tests locaux suffisent | Quand un partenaire commence à pusher |
| Apple Developer Program 99 $/an | Free-provisioning suffit pour dev solo | Premier partenaire externe à équiper |

---

## 11. Questions ouvertes

- **Docker Desktop vs Colima** : préférence ? Colima évite la licence Docker payante pour usage commercial, mais légèrement plus complexe à debugger.
- **Stockage sessions** : disque interne suffit pour Phase 0-1 (3 zones × ~30 km × ~5 GB/h ≈ 30 GB). Externe NVMe à prévoir Phase 2 quand le volume grossit.
- **Backup** : Time Machine sur les sessions ? Ou exclusion + backup ciblé vers B2 ? À trancher avant la première vraie campagne de captation.
