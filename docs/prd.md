# PRD — wsmd · Plateforme d'audit automatisé d'actifs viaires municipaux

**Date** : 2026-04-27
**Statut** : draft, en attente de validation produit
**Auteur** : Julien Riel, avec assistance Claude
**Cible PoC** : Deux modules livrables — PaveAudit (état de chaussée selon méthodo MTQ) et ChantierWatch (détection de chantiers + cross-check permits OdP), validés empiriquement à Montréal, Longueuil et Sherbrooke

> Ce document est la **seule source de vérité produit** pour wsmd. Il remplace toute spec ou plan antérieur. Les spécifications d'implémentation (architecture, code, séquence des tâches) en découlent.

---

## 1. Vision

Construire une **plateforme d'audit automatisé d'actifs viaires municipaux** par captation vidéo géoréférencée depuis un véhicule, avec une **architecture pluggable d'évaluateurs spécialisés**.

Le PoC v1 livre **trois modules** :
- **PaveAudit** — état de chaussée selon la méthodologie d'évaluation visuelle MTQ
- **ChantierWatch** — détection de chantiers et cross-référencement avec les bases de permits d'occupation du domaine public (OdP) en open data municipal
- **Annotation** — outil web local de révision/correction humaine des sorties d'évaluateurs, alimentant la boucle d'amélioration continue (active learning)

L'architecture supporte l'ajout futur d'évaluateurs (signalisation, mobilier urbain, marquage, etc.) sans refonte du pipeline. **Chaque évaluateur a son propre protocole de validation, ses propres KPIs, et son propre livrable.**

L'objectif est de **prouver la faisabilité technique** des deux modules sur trois zones tests représentatives, avec un matériel grand public et une chaîne logicielle 100 % open source.

> **Note** : la stratégie commerciale (audiences clients, cycles de vente, tarification, pitch, incorporation) est traitée séparément dans `docs/marketing/`. Le présent PRD se concentre sur le produit, son architecture et sa validation.

---

## 2. Contexte et motivation

### 2.1 Pivot stratégique

Une approche initiale (inventaire d'actifs géoréférencés à précision Niveau B 0.5–2 m) a été abandonnée pour la raison suivante :

- **Risque technique élevé** : la précision Niveau B en milieu urbain dense, sans matériel RTK, dépend de variables (heading magnétique, multipath GNSS, conditionnement de la triangulation N-vues) qui sont difficiles à maîtriser dans un side project.

Le pivot vers l'évaluation d'état :
- **Réduit le risque technique** : la précision géométrique requise descend à ±10 m (segment de rue), résolue par snap-to-road sur Géobase/OSM. Les problèmes de heading, calibration extrinsèque, et triangulation disparaissent.
- **Permet d'exploiter un référentiel québécois** : entraînement sur défauts hivernaux du Québec, reconnaissance de la signalisation MUTCD-Québec, intégration des données ouvertes municipales (Données Québec, Géobase, permits OdP).

### 2.2 Pourquoi maintenant

- Les **modèles open-vocabulary** (Grounding DINO, YOLO-World, SAM) atteignent en 2025-2026 une qualité suffisante pour la pré-annotation automatique sur classes arbitraires définies par texte. Cela réduit drastiquement le coût d'annotation initial.
- Les **iPhone récents** (16 Pro et postérieurs) embarquent un GNSS bi-bande, une centrale inertielle de qualité, et une caméra 4K HDR ProRes — accessibles directement via `AVFoundation` + `CoreLocation` + `CoreMotion`. Un dispositif unique remplace ce qui demandait précédemment caméra + GNSS + IMU + ordinateur de captation.
- Les **données ouvertes municipales** (Géobase, permits OdP, entraves planifiées) sont publiées en continu par Montréal et plusieurs villes secondaires. Le cross-référencement automatique devient techniquement faisable.
- Les **standards OGC API Features** sont matures et adoptés par les SIG modernes (QGIS, Esri, géoportails municipaux). Une livraison standard est immédiatement consommable.

### 2.3 Ce que **n'est pas** wsmd à ce stade

- Ce n'est **pas** un outil de positionnement précis Niveau B/C.
- Ce n'est **pas** un système temps réel — toute évaluation est post-captation.
- Ce n'est **pas** un service SaaS multi-utilisateurs avec auth/dashboard. Le PoC v1 est mono-utilisateur, localhost.
- Ce n'est **pas** un produit clé-en-main certifié pour responsabilité légale. Le livrable est un *signal d'audit*, pas une *preuve* — la décision d'enforcement reste au client.
- Ce n'est **pas** un outil de mesure d'IRI (International Roughness Index). PaveAudit produit l'**indice visuel uniquement** (équivalent IES MTQ). Le complément IRI nécessiterait l'exploitation des accéléromètres iPhone — Phase 2.

---

## 3. Cadre éthique et conflits d'intérêts

L'auteur est employé dans le secteur municipal québécois. Le projet wsmd est strictement **personnel et générique**, sans lien avec ses fonctions d'emploi. La discipline suivante est appliquée et inscrite au PRD pour traçabilité :

### 3.1 Discipline en exécution

- Aucune captation vidéo dans la municipalité-employeur de l'auteur, à aucune phase du projet.
- Aucune utilisation de données, documents, outils, équipements ou systèmes appartenant à l'employeur.
- Aucun travail sur le projet effectué pendant les heures rémunérées par l'employeur.
- Le PRD et tous les livrables ne référencent ni ne traitent aucun cas spécifique à la municipalité-employeur.
- Les zones de test sont sélectionnées dans des municipalités tierces (Montréal, Longueuil, Sherbrooke) sans relation avec l'employeur.

### 3.2 Audit interne du PRD

Si une partie du PRD venait à manquer à cette discipline (par exemple, en faisant référence à un dossier interne, en réutilisant un standard maison, ou en citant des chiffres confidentiels), l'auteur s'engage à corriger immédiatement. Aucune telle référence n'est connue à la date de rédaction.

---

## 4. Audiences et cas d'usage

### 4.1 Audiences produit

Profils d'utilisateurs visés par les fonctionnalités du PoC v1. Cadre les exigences fonctionnelles (formats de livrable, ergonomie, terminologie) — pas la stratégie d'aller-au-marché, qui est traitée dans `docs/marketing/`.

| Audience | Cas d'usage produit | Module pertinent |
|---|---|---|
| Inspecteurs certifiés réalisant des audits PCI/MTQ | Pré-classement automatisé de défauts pour révision ciblée | PaveAudit |
| Services d'inspection municipaux | Détection de chantiers + cross-check permits OdP | ChantierWatch |

**Hors-périmètre explicite** : aucun usage par la municipalité-employeur de l'auteur, ses fournisseurs, ses sous-traitants directs, ni les acteurs avec qui l'auteur est en relation professionnelle dans son emploi principal (cf. §3.1).

### 4.2 Cas d'usage produit

Scénarios qui dictent les exigences fonctionnelles (volume, débit, format de sortie, ergonomie de révision).

**Cas d'usage 1 — Pré-classement d'un audit visuel de chaussée à grande échelle**

> Un inspecteur certifié doit auditer 200 km de réseau. Le véhicule équipé d'un iPhone wsmd fait deux journées de captation, le pipeline tourne sur Mac, produit un GeoPackage + rapport PDF avec pré-classement des défauts. L'inspecteur relit en mode "vérification ciblée" plutôt qu'en inspection à l'aveugle.

**Cas d'usage 2 — Détection de chantiers vs permits sur tour municipal régulier**

> Un service d'inspection municipal fait un tour de 30 km mensuel avec wsmd. Le système croise les zones de chantier détectées avec les permits OdP en open data, produit une liste de zones sans correspondance permit. L'inspection terrain est ciblée plutôt qu'à l'aveugle.

---

## 5. Hypothèse produit

> Avec un **iPhone 16 Pro** mounté au pare-brise (caméra 4K + GNSS + IMU dans un seul device), des **modèles CV pré-entraînés fine-tunés sur dataset Québec hivernal**, une **boucle de pré-annotation par modèles open-vocabulary** (Grounding DINO / YOLO-World) accélérant le bootstrap, et un **pipeline post-captation Mac**, on peut livrer :
>
> - Un **classement IES (Indice d'État Subjectif) automatisé** atteignant **κ Cohen ≥ 0.6** (substantial agreement) avec un ingénieur civil de référence sur classes MTQ, sur réseau urbain et tronçon municipal.
> - Une **détection de chantiers** à **≥ 85 % de recall** avec cross-référencement automatique aux bases open data de permits OdP, isolant les détections sans correspondance avec **≥ 90 % de précision sur le verdict `unmatched`**.

**Précondition critique** : ces objectifs ne sont atteints qu'**après la Phase 1 d'annotation** (bootstrap dataset Québec via pré-annotation + révision humaine). En sortie de Phase 0, les modèles pré-entraînés out-of-the-box donneront une baseline ~60 % mAP — c'est attendu, c'est le point de départ.

Si l'hypothèse est invalidée, l'auteur saura **où** se situe le maillon faible (qualité dataset, fine-tuning, couplage permits, ergonomie annotation) et pourra investir de manière ciblée.

---

## 6. Périmètre du PoC v1

### 6.1 In-scope

- **App iOS de captation tout-en-un** (Swift/SwiftUI) : vidéo 4K H.265 + GNSS + IMU + heading vrai, persistance locale, transfert post-session vers Mac.
- **Pipeline post-captation Mac** (Python 3.12) : ingestion, pré-annotation, évaluation, livraison.
- **Module PaveAudit** : détection de défauts de chaussée selon catalogue MTQ (8 classes), agrégation par segment routier, calcul IES, livrable cartographique.
- **Module ChantierWatch** : détection de zones de chantier (cônes, barrières, signalisation temporaire, véhicules construction, clôtures privées, déblais), cross-référencement avec permits OdP, flag des `unmatched`.
- **Module Annotation** : outil web local FastAPI + SvelteKit pour révision humaine des sorties évaluateurs et export de gold labels pour ré-entraînement.
- **Boucle active learning** : pré-annotation Grounding DINO/YOLO-World → révision humaine → fine-tune → ré-évaluation, avec versioning des poids (manifest manuel + dossier).
- **Livraison OGC API Features** via pygeoapi sur GeoPackage + rapport PDF par session (template Quarto).
- **Validation empirique** sur 3 zones : Montréal (richesse de données + debug), Longueuil (réseau intermédiaire), Sherbrooke (portabilité données pauvres).

### 6.2 Out-of-scope v1 (architecture les supportera, mais pas implémentés)

- Évaluateur **état signalisation** (panneaux penchés, graffités, effacés)
- Évaluateur **mobilier urbain** (état des lampadaires, abribus, bancs, poubelles)
- Évaluateur **marquage chaussée** (lignes effacées, manquantes)
- **Détection d'accidents et urgences** en temps réel
- **Mesure IRI** par accéléromètres (Phase 2)
- **Dashboard web multi-utilisateurs** ou plateforme SaaS
- **API d'écriture** (le service OGC est read-only)
- **Authentification** / contrôle d'accès du service OGC
- **Floutage automatique** plaques d'immatriculation et visages (à ajouter avant tout partage externe — Phase 2)
- **Comparaison temporelle** multi-passages
- **Mode mobile mapping continu** sur véhicules municipaux

KPIs et protocoles de validation : voir §12.

---

## 7. Architecture

### 7.1 Vue d'ensemble

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

### 7.2 Topologie disque

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

### 7.3 Architecture évaluateurs

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

### 7.4 Stack technique

| Couche | Choix | Justification |
|---|---|---|
| App iOS | Swift / SwiftUI / AVFoundation / CoreLocation / CoreMotion | Single-device, contrôle natif des verrouillages caméra |
| Stockage session | Bundle structuré (mp4 + jsonl) sur iPhone, transfert post-session | Pas de réseau temps réel = simplicité, robustesse |
| Pipeline | Python 3.12 + venv | MPS PyTorch, écosystème CV mature |
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

---

## 8. Pipeline de captation iOS

### 8.1 Exigences fonctionnelles de l'app

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

### 8.2 Calibration

**Calibration intrinsèque caméra** : à faire **une fois** avant tout usage opérationnel, par modèle d'iPhone.
- Damier d'échiquier 10×7, taille 25 mm.
- 25–40 images variées en 4K, AF verrouillé sur la même scène.
- OpenCV `calibrateCamera` → `K` (matrice intrinsèque) + `D` (distorsion).
- Critère d'acceptation : erreur de reprojection RMS < 0.5 pixel.
- Sauvegarde : `config/iphone_<model>_calibration.yaml`, embarqué dans le manifest de chaque session.

**Pas de calibration extrinsèque mount** — sans objet pour l'évaluation d'état (pas besoin de positionnement précis 3D).

**Mount-check pré-session** : l'app vérifie via IMU que la caméra est dans une plage de pitch/roll acceptable (caméra horizontale ±10°, vue dégagée) et alerte sinon.

### 8.3 Bundle de session produit

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

## 9. Pipeline de traitement Mac

### 9.1 Ingestion

`wsmd ingest <session>` :
- Valide la structure du bundle (présence de tous les fichiers attendus).
- Extrait `frames.jsonl` à partir de la vidéo via ffmpeg (timestamp ISO 8601 par frame).
- Crée le GeoPackage de session vide avec les tables communes.
- Logge un résumé : durée, distance parcourue (interpolée GNSS), qualité moyenne `horizontalAccuracy`.

### 9.2 Pré-annotation open-vocabulary

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

### 9.3 Annotation humaine

`wsmd annotate <session>` ouvre une UI web locale qui lit/écrit le même GeoPackage de session que les évaluateurs. Architecture, workflow par module, raccourcis clavier et boucle active learning : voir §10.3 Annotation.

### 9.4 Évaluation par module

`wsmd evaluate <session> --modules pave_audit,chantier_watch` exécute chaque évaluateur en série :
1. Charge le bundle + GeoPackage de session
2. Charge le modèle custom fine-tuné (si dispo, sinon fallback sur pré-entraîné)
3. Inference frame-par-frame ou par track
4. Snap-to-road via OSRM (HTTP `/match` API)
5. Logique métier propre (calcul IES MTQ, cross-check permits, etc.)
6. Écrit ses layers de production dans le GeoPackage

### 9.5 Export labels et fine-tuning

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

### 9.6 Livraison

`wsmd report <session> --format pdf` génère un rapport client (template Quarto + Jinja2) :
- Page 1 : sommaire exécutif (km parcourus, défauts par classe, zones chantier détectées, % unmatched)
- Page 2-3 : carte du réseau colorée par IES
- Page 4 : top 20 segments les plus dégradés avec photo représentative
- Page 5-6 : zones de chantier avec verdict permit + photos extraites
- Annexe : tableau complet, métadonnées, version pipeline (modèle utilisé + commit Git)

`wsmd serve` lance pygeoapi sur `http://localhost:5000` exposant les layers publics (segments, zones) ; les layers internes (candidates, frames_index) restent privés.

---

## 10. Modules

Deux modules livrables (PaveAudit, ChantierWatch) et un module habilitant (Annotation) qui alimente la boucle d'amélioration continue. L'architecture pluggable (§7.3) supporte l'ajout futur d'évaluateurs sans refonte du pipeline. Les KPIs et protocoles de validation propres à chaque module sont regroupés au §12.

### 10.1 PaveAudit — état de chaussée

**Objectif** — produire un classement IES (Indice d'État Subjectif) automatisé par segment de chaussée selon la **méthodologie d'évaluation visuelle MTQ**, avec localisation des défauts individuels.

**Référence** : Guide d'utilisation des indices d'état des chaussées (MTQ) + catalogue des dégradations MTQ.

**Classes de défauts**

| Classe | MTQ | Spécificité QC | Source bootstrap |
|---|---|---|---|
| Nid-de-poule | Pothole | Très fréquent post-gel | RDD2022 + Pothole-600 |
| Fissure longitudinale | Long. crack | Fréquent QC | RDD2022 |
| Fissure transversale | Trans. crack | Hivernal récurrent | RDD2022 |
| Faïençage | Alligator crack | Sévère après cycles gel-dégel | RDD2022 + Crack500 |
| Ressuage | Bleeding | Estival | RDD2022 |
| Réparation rapace (patch) | Patching | Très fréquent QC | Bootstrap manuel |
| Soulèvement par gel | Frost heave | **Spécifique QC** — pas dans datasets ouverts | À annoter from scratch |
| Dégradation au sel | Salt scaling | **Spécifique QC** — subtil | À annoter from scratch |

**Pipeline interne**

1. Extraction de frames à 1-2 fps de la vidéo.
2. Inférence YOLO custom fine-tuné → bbox de défauts par frame.
3. **Snap-to-road** : chaque détection associée au segment routier le plus proche via OSRM `/nearest` ou `/match`.
4. **Agrégation par segment routier** (typiquement 50-100 m via Géobase) :
   - Comptage des défauts par classe et sévérité (légère / modérée / sévère selon barèmes MTQ)
   - Calcul de **densité de défauts** (nb / m²) par classe
   - Conversion en valeur de déduction MTQ par classe selon les barèmes
   - **Score IES 0-100** par segment
5. Output GeoPackage layers :
   - `pavement_defects` : chaque détection avec `frame_idx`, `bbox`, `class`, `severity`, `confidence`
   - `pavement_segments` : segments routiers avec `ies_score`, `classification`, `n_defects_by_class`
   - `pavement_metadata` : version modèle, version prompts, conditions de captation

**Livrable**

- Rapport PDF par mandat : carte du réseau colorée par IES, top 20 segments les plus dégradés (avec photos), liste des défauts en annexe, tableau récapitulatif.
- GeoPackage exportable QGIS / ArcGIS / GDAL.
- Endpoint OGC API Features `pavement_segments` consommable directement.

### 10.2 ChantierWatch — détection de chantiers vs permits

**Objectif** — détecter les zones de chantier sur la voirie, les cross-référencer avec les bases de permits OdP municipales open data, et flagger les chantiers sans permit correspondant comme **signaux d'audit** (non comme preuves légales).

**Classes de détection**

| Classe | COCO ? | Pré-annotation prompts |
|---|---|---|
| Cône de signalisation orange | ❌ | "orange traffic cone with reflective stripes" |
| Barrière Jersey béton | ❌ | "concrete jersey barrier", "K-rail concrete barrier" |
| Barrière de chantier métal/plastique | ❌ | "construction barrier fence", "orange barricade" |
| Panneau temporaire chantier | ❌ | "temporary construction warning sign", "diamond construction sign" |
| Véhicule de chantier (pelles, etc.) | partiel | "excavator", "asphalt paver", "construction truck", "hydraulic excavator", "mini-excavator" |
| Marquage temporaire orange | ❌ | "orange temporary lane marking" |
| Clôture de chantier privée | ❌ | "construction site fence", "chain-link construction perimeter fence", "wooden site hoarding" |
| Déblais | ❌ | "pile of excavated soil", "construction debris pile", "asphalt millings pile" |

**Pipeline interne**

1. Détection frame-par-frame YOLO custom fine-tuné (ou YOLO-World direct si fine-tune pas prêt).
2. **Clustering spatial** : DBSCAN sur coordonnées géographiques + temporelles → "zones de chantier" (polygones englobants).
3. Génération bbox géographique de chaque zone (POLYGON GeoPackage).
4. **Cross-check permits** :
   - Charger cache local des permits OdP de la ville de captation (téléchargé pré-session).
   - Pour chaque zone : requête R-tree → permits intersectant la géométrie.
   - Filtrer par fenêtre temporelle : `permit.date_debut ≤ session_date ≤ permit.date_fin`.
   - **Verdict** :
     - `permitted` : ≥ 1 permit actif intersectant
     - `unmatched` : aucun permit intersectant trouvé
     - `permit_expired` : permit existe mais hors fenêtre temporelle
     - `data_unavailable` : ville sans données ouvertes de permits → flag manuel
5. Output GeoPackage layers :
   - `construction_detections` : détections individuelles
   - `construction_zones` : zones agrégées avec `verdict`, `permit_ids` (si match)
   - `permits_reference` : copie du cache permits utilisé (traçabilité)
   - `unmatched_zones` : sous-ensemble flaggé pour audit (vue filtrée)

**Adaptateurs permits par ville**

```python
class PermitAdapter(Protocol):
    city: str
    
    def fetch_permits(self, bbox: BBox, date: datetime) -> list[Permit]: ...
    def normalize(self, raw: dict) -> Permit: ...
```

Implémentations v1 :
- `MontrealPermitAdapter` — Données ouvertes Montréal, dataset "Entraves planifiées" et permits OdP
- `SherbrookePermitAdapter` — à valider en Phase 2 (Phase 8 du plan), selon disponibilité des données ouvertes
- `LongueuilPermitAdapter` — à valider, peut tomber en `GenericNoneAdapter` selon disponibilité
- `GenericNoneAdapter` — fallback : toutes zones flaggées en `data_unavailable`, livrable identifie clairement la limitation

**Livrable**

- Rapport PDF par mandat : carte des zones détectées, tableau des `unmatched` avec photos extraites + permit IDs vérifiés/absents.
- GeoPackage avec couche "potentielles violations à vérifier".
- **Avertissement explicite** dans le rapport : *"Ce rapport produit un signal d'audit, pas une preuve légale. Toute action d'enforcement doit être précédée d'une vérification terrain par un inspecteur autorisé."*

### 10.3 Annotation et boucle active learning

**Pourquoi un module à part entière** — aucun évaluateur ne sera jamais à 100 % en sortie. Pour qu'un livrable soit acceptable, un humain doit pouvoir relire, corriger, et **certifier**. Chaque correction = un nouvel exemple d'entraînement pour le fine-tune incrémental.

**Architecture**

- App web locale (FastAPI backend + SvelteKit frontend) servie par `wsmd annotate` sur `localhost:5001`
- Lit/écrit le **même GeoPackage** de session que les évaluateurs — pas de duplication de données
- Auth absente en v1 (localhost), basic auth ajoutable en v2 quand un partenaire l'utilise
- Module-aware : UI adaptée à chaque évaluateur (PaveAudit vs ChantierWatch ont des champs différents à corriger)

**Workflow par module**

- **PaveAudit** : galerie des candidats par classe → frame contextuelle → confirmer/rejeter/refiner bbox/changer classe/ajuster sévérité MTQ → ajouter défauts manqués manuellement
- **ChantierWatch** : galerie des zones détectées → confirmer/rejeter/refiner bbox géographique → ajuster permit-match si la suggestion est incorrecte → flagger comme "vraiment sans permit" ou "permit existe ailleurs"

**Ergonomie cible** : 20-30 sec par item médian (post-pré-annotation). Raccourcis clavier obligatoires :
- `1` = confirmer
- `2` = rejeter
- `3` = corriger classe
- `4` = corriger sévérité
- Espace = item suivant
- ← → = navigation
- A = ajouter manuellement

**Stockage** : layer GeoPackage `verified_<module>` avec `candidate_id`, `verdict` (`confirmed` / `rejected` / `corrected` / `added_manually`), `corrected_class`, `corrected_severity`, `annotator_id`, `annotated_at`. Le layer `annotations_log` conserve l'historique audit trail.

**Boucle d'amélioration**

```
Capture session
    ↓
wsmd preannotate (Grounding DINO ou YOLO-World)
    ↓
GeoPackage : layer "candidates"
    ↓
wsmd annotate    ← humain vérifie (15-30 sec/box)
    ↓
GeoPackage : layer "verified" (gold labels)
    ↓
wsmd export-labels    → dataset YOLO/COCO
    ↓
wsmd train --module X    → custom YOLO rapide (production)
    ↓
Évaluateurs en production utilisent le custom YOLO entraîné
    ↓
[boucle continue : nouvelles sessions → nouvelles annotations → ré-entraînement périodique]
```

**Honnêteté sur les limites** — Grounding DINO et YOLO-World ne sont **pas parfaits**. Ils vont :
- Manquer des défauts subtils (salt damage typique du QC peut être imperceptible).
- Sortir des faux positifs sur textures ambiguës (ombres = fissures, etc.).
- Être inconsistants frame-à-frame.

C'est exactement pourquoi l'humain est dans la boucle. La pré-annotation accélère mais ne remplace pas la révision.

---

## 11. Données et livraison

Cette chapitre couvre les données en entrée (datasets de bootstrap et de fine-tuning, considérations licences) et en sortie (schéma GeoPackage produit, configuration pygeoapi, endpoints OGC API exposés).

### 11.1 État des lieux des datasets sources

| Domaine | Source | Volume | Licence | QC-spécifique ? | Utilité v1 |
|---|---|---|---|---|---|
| Défauts chaussée | RDD2022 (IEEE Big Data Cup) | 47 k images, 8 classes | Académique (à vérifier commercial) | ❌ (US, JP, IN, CZ, NO) | Bootstrap principal |
| Défauts chaussée | Crack500, CrackForest, GAP, Pothole-600 | qq centaines à milliers | CC variées | ❌ | Compléments |
| Défauts chaussée | Roboflow Universe community | variable | variable | quelques rares | À trier au cas par cas |
| Cônes / barrières | COCO (traffic objects) + OpenImages V7 | grand | CC-BY | ❌ | Bootstrap chantier |
| Cônes / barrières | Roboflow `construction-equipment` | variable | variable | ❌ | Compléments |
| **Permits OdP Montréal** | Données Québec / Mtl ouvert | actualisé en continu | CC-BY | ✅ | Cross-check direct |
| Permits OdP autres villes | Données Québec (variable selon municipalité) | variable | CC-BY | ✅ partiel | Selon ville |
| Référentiel routier | Géobase Québec + OpenStreetMap | provincial complet | CC-BY / ODbL | ✅ | Snap-to-road |

### 11.2 Stratégie d'entraînement phasée

```
Phase 0 (semaines 1-3) : Bootstrap pré-entraîné
  ├─ YOLOv8 pré-entraîné COCO (chantiers : cônes, barrières)
  ├─ YOLOv8 pré-entraîné RDD2022 (chaussée : 8 classes)
  ├─ Mesurer baseline sur 200-300 frames captées au QC
  └─ Cible : ~60 % mAP — c'est bas, c'est attendu

Phase 1 (semaines 4-12) : Bootstrap dataset QC
  ├─ Captation 5-10 sessions Montréal/Longueuil (10-20 km chacune)
  ├─ Pré-annotation Grounding DINO/YOLO-World
  ├─ Révision humaine de 5000-10 000 frames (1-2 weekends post-pré-annotation)
  ├─ Fine-tune YOLOv8 custom sur dataset combiné (RDD2022 + QC)
  └─ Cible : ~75 % mAP PaveAudit, 85 % recall ChantierWatch

Phase 2 (continue) : Boucle active learning
  ├─ Chaque nouvelle session captée + annotée → ajout au dataset
  ├─ Ré-entraînement périodique (tous les 3-5k frames ajoutés)
  ├─ Mesure régression vs version précédente
  └─ Versioning des poids dans models/MANIFEST.yaml
```

### 11.3 Considérations licences

- **RDD2022** : licence académique, à vérifier précisément en Phase 0 avant utilisation commerciale. Si bloquant : training from scratch sur dataset 100 % maison Phase 2.
- **COCO, OpenImages** : CC-BY, utilisables commercialement avec attribution.
- **Permits OdP Montréal, Géobase, Données Québec** : CC-BY, utilisables sans restriction commerciale avec attribution.
- **Dataset QC propre** : conserver privé en v1 ; décision sur publication HF Hub (CC-BY-NC) à revoir Phase 2.

### 11.4 Schéma GeoPackage `assets.gpkg`

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

### 11.5 Schéma `pavement_segments`

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

### 11.6 Schéma `construction_zones`

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

### 11.7 Configuration pygeoapi

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

### 11.8 Endpoints exposés

- `GET /collections/pavement_segments/items?bbox=...&ies_score__lt=50` — filtrage par BBox + score
- `GET /collections/pavement_segments/items/{segment_id}` — segment individuel
- `GET /collections/construction_zones/items?verdict=unmatched` — zones flaggées
- `GET /collections/construction_zones/items?f=html` — UI HTML simple intégrée

### 11.9 Consommation cible

- **QGIS** : ajout de couche WFS / OGC API Features → consommation native
- **Curl / scripts** : `curl http://localhost:5000/collections/pavement_segments/items?bbox=... > export.geojson`
- **Browser** : interface HTML générée automatiquement par pygeoapi

---

## 12. Validation et engagements qualité

### 12.1 KPIs PoC v1

Cibles que le PoC v1 doit atteindre pour valider l'hypothèse produit (§5). Mesurées selon les protocoles ci-dessous.

| Module | Métrique | Cible |
|---|---|---|
| PaveAudit | κ Cohen sur classification IES MTQ vs ingénieur référence | ≥ 0.6 |
| PaveAudit | mAP par classe de défaut | ≥ 0.75 |
| PaveAudit | Erreur IES moyenne par segment | ±10 / 100 |
| ChantierWatch | Recall détection zones de chantier | ≥ 85 % |
| ChantierWatch | Précision verdict `unmatched` | ≥ 90 % |
| ChantierWatch | F1 cross-check temporel permits | ≥ 0.85 |
| Annotation | Vitesse médiane révision post-pré-annotation | ≤ 30 s/item |
| Pipeline global | Temps traitement / heure de vidéo (Mac M-series) | ≤ 2× temps réel |

### 12.2 Protocole PaveAudit

1. Sélectionner **5-10 segments de référence** dans la zone test Montréal (3-5 km de boucle), variés en état de chaussée.
2. **Inspection manuelle** par 1 ingénieur civil de référence (à recruter Phase 1, via réseau personnel ou recommandation académique).
3. Captation wsmd des mêmes segments en 3 passages (matin/midi/fin de journée pour variabilité d'éclairage).
4. Exécution du pipeline → IES par segment + liste des défauts.
5. Comparaison segment-à-segment :
   - Accord sur classification IES (κ Cohen)
   - Précision/recall par classe de défaut (mAP)
   - Erreur IES moyenne (RMSE)
6. Production de `validation_report.md` :
   - Tableau de confusion classification IES
   - Histogramme erreurs IES
   - Cas problématiques identifiés (occlusions, distance, conditions)

### 12.3 Protocole ChantierWatch

1. Identifier **20-30 zones de chantier** captées à Montréal sur boucle de 10-20 km.
2. **Vérification manuelle** : pour chaque zone, croiser avec la base de permits OdP (téléchargée à la même date que la captation). Construire la vérité terrain `permitted` / `unmatched` / `permit_expired`.
3. Exécution du pipeline ChantierWatch → verdicts automatisés.
4. Comparaison zone-à-zone :
   - Recall détection (combien de chantiers réels ont été détectés ?)
   - Précision verdict `unmatched` (combien des "unmatched" sont vraiment des chantiers sans permit ?)
   - F1 cross-check temporel
5. **Validation manuelle des `unmatched`** par croisement Mtl-info / 311 (signalements citoyens) pour confirmer absence effective de permit.

### 12.4 Validation portage Sherbrooke

- Réplique du protocole sur 1-2 boucles à Sherbrooke en Phase 8.
- Mesure de la dégradation de performance (modèle entraîné sur Mtl appliqué à Sherbrooke).
- Si dégradation > 15 % en mAP : ajout de 500-1000 frames Sherbrooke au dataset, ré-entraînement.

---

## 13. Plan phasé

| Phase | Durée | Livrable | Effort estimé |
|---|---|---|---|
| **0 — Setup environnement** | 2 sem | venv Python, MPS PyTorch validé, calibration intrinsèque iPhone, repo Git initialisé, vérif licence RDD2022 | 8-12 h |
| **1 — App iOS captation** | 3 sem | App SwiftUI déployée free-provisioning, bundle session produit, mount-check, calibration | 25-35 h |
| **2 — Pipeline ingestion** | 1 sem | `wsmd ingest`, frames extraction, JSON schemas validés | 8-12 h |
| **3 — Pré-annotation + outil annotation web** | 3-4 sem | `wsmd preannotate` avec Grounding DINO, outil web FastAPI+SvelteKit fonctionnel | 30-45 h |
| **4 — Bootstrap dataset + fine-tune PaveAudit** | 4 sem | Annotation 3-5k frames Mtl, fine-tune YOLO PaveAudit, mAP baseline mesurée | 35-50 h (annotation = 20-25 h) |
| **5 — Bootstrap ChantierWatch + adapter Mtl** | 3 sem | Fine-tune YOLO ChantierWatch, MontrealPermitAdapter, cross-check fonctionnel | 25-35 h |
| **6 — Livraison OGC + rapports** | 2 sem | `wsmd serve` avec pygeoapi, template rapport PDF Quarto | 12-18 h |
| **7 — Validation Montréal** | 2 sem | Captation + validation 5-10 segments + 20 zones, rapport de précision, recrutement ingénieur civil | 15-25 h |
| **8 — Portage Sherbrooke** | 2 sem | SherbrookePermitAdapter (ou GenericNoneAdapter), captation + validation portabilité | 12-18 h |
| **Total** | **~22 semaines** |  | **~170-250 h** |

À 8 h/semaine effectifs (soirs + 1 weekend par mois) : **5-6 mois calendaires**.
À 12 h/semaine (intense) : **4-5 mois calendaires**.

**Plan B si retard en Phase 5** : démo PaveAudit seul en Phase 7, ChantierWatch livré en bonus visuel sans cross-check, finalisation cross-check Phase 8.

---

## 14. Risques et mitigations

### 14.1 Risques techniques

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| Grounding DINO + fine-tune ratent les défauts hivernaux subtils (frost heave, salt scaling) | Moyenne-haute | Cible IES non atteinte | Annotation manuelle dédiée à ces classes ; flag explicite dans rapport ; renvoi à v2 IRI capteur inertiel |
| Permits Sherbrooke peu/pas en open data | Élevée | ChantierWatch dégradé sur Sherbrooke | `GenericNoneAdapter` qui flag toutes zones `data_unavailable` ; livrable identifie explicitement la limitation |
| Side project, fenêtre 4-6 mois optimiste | Moyenne | Démo retardée | Découpage livraisons : démo PaveAudit seul à 4 mois si ChantierWatch retarde ; plan B documenté |
| Validation κ ≥ 0.6 non atteinte malgré fine-tune | Moyenne | Cible qualité non atteinte | Calibrer scoring vs MTQ sur données réelles avant de promettre ; ajuster les seuils MTQ ; repositionner le module en "outil de pré-priorisation" |
| Volumes de vidéos ingérables | Faible | Sessions limitées | H.265 4K = ~6 GB/h ; SSD externe 1-2 TB suffit pour la durée du PoC |
| iPhone 16 Pro indisponible / ancienne génération moins précise | Faible | Précision GNSS dégradée | Tester sur iPhone 15 Pro en backup ; mesurer empiriquement |

### 14.2 Risques juridiques et de licences

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| Dataset QC non utilisable hors recherche (licences source) | Faible | Modèle final restreint à un cadre académique | Vérifier licences en Phase 0 ; toute donnée incertaine → exclue ; possibilité re-train from scratch sur dataset 100 % maison Phase 2 |
| Loi 25 — vidéos avec plaques/visages | Moyenne | Risque légal sur la donnée brute | Stockage local SSD chiffré v1 ; floutage automatique avant tout partage externe (Phase 2, modèle ANPR + MTCNN) ; ne pas exporter de vidéos brutes |
| Disclaimer "signal d'audit" insuffisant en cas de litige | Faible | Risque légal | Avis juridique requis avant tout usage du livrable hors cadre interne ; limitation de responsabilité explicite dans chaque rapport |

### 14.3 Risques projet

- **Sous-estimation du temps d'annotation** : à minimiser, c'est facile à allonger. Bloquer la Phase 4 si volume annoté < 1500 frames à mi-phase et reconsidérer l'embauche d'un stagiaire Mitacs pour bootstrap dataset.
- **Dérive de scope** : ne pas implémenter SignageStateEvaluator ni MarkingEvaluator ni floutage auto avant que PaveAudit + ChantierWatch ne soient livrés et validés.
- **Météo défavorable au planning de captation** : prévoir buffer (Phase 7-8) pour repasser en cas de pluie/neige rendant les captures inexploitables.
- **Disponibilité ingénieur civil de référence** : démarcher dès Phase 1 (réseau personnel ou recommandation prof Poly/ÉTS), ne pas attendre Phase 7.

---

## 15. Évolutions Phase 2+

À considérer **uniquement si le PoC v1 atteint les KPIs** et que la motivation produit reste.

### 15.1 Enrichissement modules existants

- **Capteur inertiel iPhone pour IRI** : exploiter `CMDeviceMotion` accéléromètres pour produire un IRI proxy en parallèle de l'IES visuel — couplage rendrait le livrable comparable aux profileurs pros à coût marginal nul.
- **Floutage automatique plaques + visages** : modèle ANPR + MTCNN ou similaire dans le pipeline avant tout partage externe.
- **Comparaison temporelle multi-passages** : disparitions/apparitions/aggravations entre 2 captations de la même zone.

### 15.2 Nouveaux évaluateurs

- `SignageStateEvaluator` — état des panneaux (penchés, graffités, effacés)
- `MarkingEvaluator` — état du marquage chaussée (lignes effacées, manquantes, peinture éraflée)
- `StreetFurnitureEvaluator` — état du mobilier urbain (lampadaires, abribus, bancs, poubelles)

### 15.3 Mode opérationnel

- **Mobile mapping continu** sur véhicules municipaux (autobus STL/STM, balayeuses) Phase 3.
- **API d'écriture authentifiée** pour validation par utilisateur identifié.
- **Dashboard web** de supervision multi-sessions, multi-mandats, multi-clients.
- **Auth + permissions** pour usage multi-utilisateur.

### 15.4 Ouverture et recherche

- Publication du **dataset Quebec Pavement Winter Damage** sur HuggingFace Hub (CC-BY-NC).
- DATASHEET.md selon Gebru et al. 2018 + croissant.json pour métadonnées ML.
- Citation académique (CITATION.cff) si publication scientifique.

### 15.5 Cloud et passage à l'échelle

- Pipeline déchargé sur **serveur Linux + GPU NVIDIA** (post-captation) pour mandats volumineux.
- **Stockage objet** (B2 / GCS / bucket institutionnel) pour vidéos brutes archivées.
- **CI/CD** sur les modèles : versioning des poids (DVC + bucket), métriques de régression entre versions.

---

## 16. Glossaire

| Terme | Définition |
|---|---|
| **wsmd** | Worksight Make Dataset (nom de code interne du projet) |
| **PaveAudit** | Module d'évaluation d'état de chaussée selon méthodologie MTQ |
| **ChantierWatch** | Module de détection de chantiers et cross-check avec permits OdP |
| **IES** | Indice d'État Subjectif — score visuel 0-100 de l'état de chaussée selon MTQ |
| **IRI** | International Roughness Index — index inertiel non-mesurable depuis CV seul |
| **MTQ** | Ministère des Transports du Québec |
| **PCI** | Pavement Condition Index — standard ASTM D6433 (référence internationale, distincte de l'IES MTQ) |
| **OdP** | Occupation du Domaine Public — permis municipal pour chantier sur voie publique |
| **κ Cohen** | Coefficient de Kappa Cohen — mesure d'accord inter-annotateur, 0.6+ = "substantial agreement" |
| **mAP** | Mean Average Precision — métrique standard de détection d'objets |
| **GNSS** | Global Navigation Satellite Systems (GPS, GLONASS, Galileo, BeiDou) |
| **IMU** | Inertial Measurement Unit (accéléromètres + gyroscopes + magnétomètre) |
| **OGC API Features** | Standard Open Geospatial Consortium pour la publication de features géographiques en REST/JSON |
| **NAD83 MTM zone 8** | Système de coordonnées projeté pour le sud du Québec, EPSG:32188 |
| **Grounding DINO** | Modèle CV open-vocabulary par IDEA-Research, détection bbox via prompts texte |
| **YOLO-World** | Modèle CV open-vocabulary par Tencent, basé YOLOv8, détection via prompts texte |
| **DBSCAN** | Density-Based Spatial Clustering — algo de clustering non-paramétrique pour zones de chantier |
| **OSRM** | Open Source Routing Machine — moteur de routing/map-matching open source |
| **Géobase** | Référentiel routier provincial du Québec, données ouvertes |
| **MPS** | Metal Performance Shaders, accélérateur Apple Silicon pour PyTorch |
| **Active learning loop** | Boucle d'amélioration : prédiction → correction humaine → ré-entraînement |
| **Pré-annotation** | Génération automatique de bbox candidats par modèle open-vocab, à réviser par humain |
| **Free provisioning** | Déploiement Xcode avec Apple ID gratuit (vs Apple Developer Program payant) |
| **Loi 25** | Loi modernisant la protection des renseignements personnels (Québec, en vigueur 2024) |

---

## 17. Décisions ouvertes (à valider en Phase 0)

- [ ] Confirmer disponibilité de l'iPhone 16 Pro personnel pour le projet (matériel personnel, jamais de l'employeur)
- [ ] Identifier 1 ingénieur civil de référence pour validation IES en Phase 7
- [ ] Vérifier la licence exacte de RDD2022 (académique stricte ? CC-BY ? CC-BY-NC ?)
- [ ] Vérifier la disponibilité des permits OdP pour Longueuil et Sherbrooke en Phase 0 (sinon `GenericNoneAdapter` ou abandon zone)

**Décisions stratégie d'affaire** (hors PRD, traitées dans `docs/marketing/`) : nom commercial, incorporation, tarification, choix d'audience pour les premières démos, règles internes d'activités accessoires de l'employeur.

---

*Fin du document. 2026-04-27.*
