# PRD — App iOS de captation Repères

**Date** : 2026-04-29
**Statut** : draft v2.1 (MVP serré + LiDAR depth)
**Auteur** : Julien Riel
**Cible** : livrer une app iOS minimale, autonome, qui capture des sessions de roulage géoréférencées + inertielles, et produit un bundle local exploitable plus tard pour tout traitement post-captation.

> v2 du PRD : recentrage MVP suite revue critique. Tout ce qui ne sert pas directement le besoin "dataset personnel propre, calibré, réutilisable" est sorti du scope v1.
>
> v2.1 : ajout du LiDAR depth capture en MVP (méthode `AVCaptureDepthDataOutput` + `AVCaptureDataOutputSynchronizer`, **sans ARKit** pour préserver les locks AE/WB). Étend la portée downstream (audit géométrique de chaussée, détection d'obstacles proches) au prix de ~6-10 sem calendaires supplémentaires et ~7 GB/h de stockage additionnel. Détails §5.6, risques §11.

---

## 1. Pourquoi ce pivot

Le PoC v1 précédent combinait app iOS + pipeline Python + fine-tune YOLO + protocole de validation κ + boucle Label Studio + 30+ segments de référence + EFVP Loi 25. En side project solo, c'est ~12 mois calendaires réalistes avec un risque commercial non résolu.

L'app de captation est, à elle seule, un livrable :

- **Valeur intrinsèque** : permet de constituer un dataset personnel propre, calibré, réutilisable pour des projets de traitement futur — détection 2D, ML supervisé sur frames, IRI inertiel, mapping, **et audit géométrique courte portée de la chaussée grâce au LiDAR (§5.6)**. **Limites assumées** : la VIO/SLAM longue portée reste contrainte par l'OIS et le rolling shutter ; la portée pratique du LiDAR est ~5-8 m utile (dégradée en plein soleil). La calibration extrinsèque caméra wide↔IMU reste estimée ; en revanche RGB↔depth est fournie par AVFoundation à chaque frame.
- **Risque borné** : pas de modèle à entraîner, pas de validation statistique, pas d'EFVP complexe (captation personnelle), pas de question commerciale.
- **Réutilisable** : si on revient un jour au pipeline ML, l'app est déjà faite et le bundle est déjà bien formé.

---

## 2. Vision et besoin réel

Une app iOS native qui transforme un iPhone récent monté au pare-brise en **outil de captation reproductible**. Bundle de session 100 % local, format ouvert, exploitable hors-ligne par un script Python ou QGIS.

**Le seul besoin réel à satisfaire en v1** : que chaque bundle soit *post-traitable sans surprise* — caméra dans une config connue et stable, capteurs synchronisés rigoureusement, calibration intrinsèque applicable.

**Non-objectifs explicites** :
- Pas de traitement temps réel.
- Pas de cloud, pas de sync, pas de multi-utilisateurs.
- Pas d'analyse ni de score à bord.
- Pas d'UX de partage native — Files.app + Finder USB suffisent.
- Pas de visualisation des sessions à bord.

---

## 3. Cas d'usage

**UC-1 — Captation roulante solo, 30-60 min**
L'auteur monte son iPhone au pare-brise, démarre l'app, fait le mount-check, lance la session, conduit, arrête. Bundle écrit localement.

**UC-2 — Transfert vers Mac post-session**
Bundle accessible via Files.app (UI iOS standard) → AirDrop ou USB Finder. L'app n'est pas un visualiseur — la suite vit ailleurs.

> **Reporté v2** : passages multiples sur même parcours pour comparaison. Conflit avec verrouillage AE/WB ; on log les paramètres exacts au lock pour permettre cette comparaison plus tard, mais on ne livre pas de fonctionnalité dédiée v1.

---

## 4. Périmètre

### 4.1 In-scope MVP

- App iOS native SwiftUI, iPhone 16 Pro / 17 Pro cible, iPhone 15 Pro fallback.
- Capture **vidéo 4K H.265 30 fps** via `AVCaptureSession` (option ProRes flippable, voir §5.1).
- **Lentille `builtInWideAngleCamera` pinned** (jamais virtual device), AF/AE/WB verrouillés post-calibration scène, **avec monitoring continu des unlocks iOS et logging des paramètres exacts**.
- Logging **GNSS** via `kCLLocationAccuracyBestForNavigation` (1 Hz natif, parfois 2 Hz en mouvement — pas plus).
- Logging **IMU à 100 Hz** via `CMDeviceMotion` mode `xMagneticNorthZVertical`.
- Logging **cap vrai** via `CLHeading.trueHeading` (référence secondaire — voir §5.2 sur la fiabilité magnétomètre en cabine acier).
- **Capture LiDAR depth à 10 Hz** via `AVCaptureDepthDataOutput` (jamais ARKit), synchronisée avec le video output via `AVCaptureDataOutputSynchronizer`. Format `.bin` brut Float32 + sidecar `depth.jsonl` avec intrinsics, extrinsics RGB↔depth, et confidence summary par frame. Voir §5.6.
- **Sync multi-streams rigoureuse** via `CMClock` partout (voir §5.2) + protocole de sync mark au début/fin de session.
- **Mount-check au démarrage** : vérification pitch + roll, alerte bloquante si hors plage. Pas de check continu en cours de session.
- **Mode calibration intrinsèque in-app** : capture du damier dans la même config `AVCaptureSession` que la captation roulante (voir §7).
- Manifest de session complet : device, version app, lentille, ref calibration, codec, dates, **paramètres caméra exacts au lock**, événements thermal/interruption.
- Persistance locale dans bundle structuré (mp4 + jsonl + manifest + dossier `depth/` avec chunks `.bin`) accessible via Files.app.
- Permissions iOS strictement minimales (caméra, localisation, motion).

### 4.2 Out-of-scope MVP

- Tout pipeline post-captation (extraction frames, snap-to-road, etc.).
- Tout ML embarqué.
- Annotation, révision, scoring.
- Floutage temps réel ou différé.
- Synchronisation cloud, multi-device, partage entre utilisateurs.
- **Bouton AirDrop dans l'app** — on s'appuie sur Files.app (zéro UX custom).
- **Écran détail de session élaboré** — liste seulement, tap ouvre le bundle dans Files.app.
- **Niveau à bulle continu pendant session** — check au démarrage suffit ; pendant la session, seul un indicateur thermal est visible.
- **Overlay UI riche** (distance estimée, qualité GNSS, etc.) — durée écoulée + état thermal seulement.
- **Disclaimer Loi 25 dans l'app** — responsabilité utilisateur, noté dans README.
- Saisie utilisateur de métadonnées véhicule (auto-fill avec valeur fixe v1).
- Suivi temporel inter-passages (UC-2 historique reporté).
- Conditions météo, type de chaussée, etc.
- Multilangue : FR-CA seulement, EN si gain trivial.
- **Reconstruction 3D / mesh / nuage de points fusionné via ARKit** : on stocke les depth maps brutes par frame ; toute reconstruction 3D est un projet downstream.
- **Floutage / occlusion automatique de personnes via depth + segmentation** : pas en MVP.
- **Visualisation 3D embarquée** des depth maps : pas en MVP.

---

## 5. Exigences fonctionnelles

### 5.1 Configuration caméra

1. Lentille `builtInWideAngleCamera` pinned explicitement (jamais `builtInDualCamera` / `builtInTripleCamera`).
2. Au démarrage de session : 5-10 sec de scène typique pour AE/AF/WB auto, puis verrouillage `lockForConfiguration` + `setExposureMode(.locked)` + `setFocusMode(.locked)` + `setWhiteBalanceMode(.locked)`.
3. **Monitoring d'unlock iOS** : KVO sur `adjustingExposure`, `adjustingFocus`, `adjustingWhiteBalance` ; toute reprise auto par iOS est loggée comme événement (`camera_unlock` avec timestamp + raison) et le manifest marque `lock_drift_events: N`.
4. **Paramètres exacts loggés au lock** dans le manifest : `iso`, `exposureDuration`, `lensPosition`, `whiteBalanceGains` (R/G/B), `focusMode`, `exposureMode`. Permet la comparaison/normalisation entre sessions en post-traitement.
5. Format vidéo par défaut : 3840×2160 H.265 30 fps, conteneur `.mp4`.
6. **ProRes : reporté v1.1.** Si le dataset H.265 montre des artefacts gênants pour le CV downstream, un toggle ProRes 4K 30 fps sera ajouté. Évite deux codepaths à tester en MVP.
7. Stabilisation logicielle : **désactivée** via `preferredVideoStabilizationMode = .off`. Attention : l'**OIS hardware** reste actif et n'est pas désactivable ; à compenser en post si nécessaire pour fusion IMU.
8. Audio caméra : **désactivé** sur la captation (privacy + simplicité).
9. Si throttling thermique force une descente en gamme (ex. 1080p) : la session continue, le manifest logge la transition, la session est marquée `degraded: true`.
10. **`AVCaptureDepthDataOutput` ajouté à la même `AVCaptureSession`** que le video output, synchronisé via `AVCaptureDataOutputSynchronizer` (voir §5.6 pour le détail). **Validation explicite Phase A/B** : confirmer que les locks AE/WB du point 2 tiennent quand le depth output est branché. Si conflit irrémédiable, décision GO/NO-GO LiDAR (§11) — on choisit alors lock-AE/WB et on désactive depth.

**Limites connues du dataset** (à logger, pas à corriger en v1) :

- **OIS sensor-shift hardware** : déplace physiquement le capteur de quelques pixels selon l'accélération. Conséquence : `(cx, cy)` de la matrice K varie session-par-session voire frame-par-frame. Les API publiques iOS n'exposent pas la position de l'OIS. Le dataset assume une matrice K avec ±1-2 px de bruit sur le centre optique.
- **Rolling shutter** : readout d'environ 20-25 ms par frame. À 50 km/h, le bas de l'image est échantillonné ~30 cm plus loin que le haut. Le manifest logge `readout_duration_ns` pour permettre la compensation downstream.
- **Variation unit-to-unit** : la calibration est par modèle (`iphone_16pro_wide_v1`), pas par unité physique. Erreur attendue ~0.5-1 % sur la focale entre deux iPhones du même modèle. Si plusieurs unités sont utilisées : capture séparée par numéro de série recommandée (`iphone_<serial>_wide_v1.yaml`).
- **Calibration extrinsèque caméra↔IMU absente** : limite la fusion serrée vidéo+IMU et la VIO. Le manifest logge un estimé Apple-published à titre de référence ; calibration dynamique = projet downstream.

### 5.2 Synchronisation multi-streams

**Principe** : un seul timebase de référence, `CMClock` host time (`CMClockGetHostTimeClock()`).

- Vidéo : `CMSampleBuffer.presentationTimeStamp` est nativement en `CMClock` host. Aucun mapping nécessaire.
- IMU : `CMDeviceMotion.timestamp` est en `mach_absolute_time` (même base que `CMClock` host) — conversion triviale.
- GNSS : `CLLocation.timestamp` est en `Date` (temps mur). On capture **simultanément** `Date()` et `CMClockGetTime(CMClockGetHostTimeClock())` à `T₀` (ouverture session) pour établir l'offset, et on convertit chaque fix GNSS dans le timebase vidéo.
- Heading : idem GNSS. Note : `trueHeading` magnétométrique est dévié de 20-50° en cabine acier (chassis, alternateur). **Référence de cap primaire = GNSS course** au-dessus de 5 m/s ; `trueHeading` loggé en secondaire avec disclaimer.
- **Depth (LiDAR)** : `AVDepthData` est livré via `AVCaptureDataOutputSynchronizer` qui appaire chaque frame depth avec sa frame vidéo via `CMClock` host. **Pas de drift à gérer** ; chaque sample depth est garanti synchronisé avec une frame vidéo précise (timestamp identique au `CMSampleBuffer.presentationTimeStamp`).

**Sync mark protocole** : au début ET à la fin de chaque session, l'app émet :
- Un **flash écran** blanc/noir 200 ms (visible dans la vidéo).
- Un **double-tap utilisateur sur l'écran** demandé par l'app : un overlay "Tape 2× pour confirmer la sync mark" apparaît, et chaque tap produit un pic accel net (~0.5 g instantané, facilement détectable dans `imu.jsonl`). Plan B si l'utilisateur ne tape pas dans les 5 sec : `UIImpactFeedbackGenerator.impactOccurred(.heavy)` — pulse mécanique direct dans le chassis du téléphone, pic accel détectable. (Le bip haut-parleur a été abandonné : signal trop faible dans le chassis pour être détecté de façon fiable.)
- Un **événement explicite** dans tous les jsonl avec timestamp `CMClock`.

→ Permet une **vérification automatique de drift** post-session : `Δ(flash_video_timestamp, sync_mark_imu_timestamp) < 50 ms` est le critère mesurable.

**Format des jsonl** (timestamps tous en nanosecondes depuis boot, `CMClock` host) :
- `gnss.jsonl` : `{ts_ns, lat, lon, alt, h_acc, v_acc, course, speed_mps}`.
- `imu.jsonl` : `{ts_ns, quat: [w,x,y,z], gyro: [x,y,z], accel: [x,y,z], user_accel: [x,y,z]}`.
- `heading.jsonl` : `{ts_ns, true_heading_deg, accuracy_deg}`.
- `depth.jsonl` : `{ts_ns, frame_index, chunk_file, intrinsics: [[fx,0,cx],[0,fy,cy],[0,0,1]], extrinsic_rgb_to_depth: 4x4, confidence_summary: {low_pct, med_pct, high_pct}}`. Pixels eux-mêmes dans `depth/chunk_NNN.bin` (voir §5.6).

Le manifest contient le mapping `ts_ns ↔ ISO 8601` du `T₀` pour permettre la conversion en temps mur si nécessaire en post.

### 5.3 Mount-check (démarrage seulement)

- À l'ouverture de l'écran "Nouvelle session", lecture continue du pitch ET roll via `CMDeviceMotion`.
- Affichage visuel d'un niveau à bulle simple.
- Bouton "Démarrer la session" actif uniquement si pitch ∈ [-10°, +10°] **et** roll ∈ [-10°, +10°] **stable pendant 3 sec**.
- Une fois la session démarrée, **plus de mount-check actif**. Si le mount glisse, c'est dans le manifest IMU à analyser en post.

### 5.4 Cycle de session (UI minimale)

1. **Écran d'accueil** : liste des sessions (id, date, durée, taille). Bouton "Nouvelle session". Bouton "Calibrer" (mode calibration intrinsèque, voir §7). Tap sur une session → ouvre Files.app au dossier du bundle.
2. **Écran "Nouvelle session"** : preview caméra + niveau à bulle + bouton "Démarrer" (gated).
3. **Pendant session** : preview caméra + durée écoulée + indicateur thermal (vert/jaune/rouge basé sur `ProcessInfo.thermalState`). Rien d'autre.
4. **Bouton "Arrêter"** : sync mark de fin → flush des fichiers → écriture du `manifest.json` → retour à l'accueil.
5. **Suppression** : swipe-to-delete sur la liste **avec confirmation** (alerte iOS standard "Supprimer la session 2026-XX-XX ? Action irréversible"). Pas d'undo.

Pas de bouton AirDrop, pas d'écran détail. Files.app fait le partage et l'inspection.

### 5.5 Manifest minimal

```json
{
  "manifest_version": "1.0",
  "personal_only": true,
  "session_id": "2026-04-27_14-30-00_a1b2c3d4",
  "started_at_iso": "2026-04-27T14:30:00-04:00",
  "started_at_host_ns": 123456789012345,
  "ended_at_iso": "2026-04-27T15:12:33-04:00",
  "ended_at_host_ns": 126010189012345,
  "duration_sec": 2553,
  "device": {
    "model": "iPhone16,1",
    "ios_version": "18.4",
    "app_version": "0.1.0"
  },
  "camera": {
    "lens": "builtInWideAngleCamera",
    "lens_pinned": true,
    "calibration_ref": "iphone_16pro_wide_calibration_v1",
    "stabilization_software": "off",
    "ois_hardware_active": true,
    "audio": "disabled",
    "readout_duration_ns": 22000000,
    "camera_imu_extrinsic": {
      "source": "apple_published_estimate",
      "translation_mm": [0.0, -8.5, 4.2],
      "rotation_quat": [1.0, 0.0, 0.0, 0.0]
    },
    "lock_params_at_start": {
      "iso": 200,
      "exposure_duration_seconds": 0.0167,
      "lens_position": 0.85,
      "white_balance_gains": {"red": 1.7, "green": 1.0, "blue": 2.1},
      "focus_mode": "locked",
      "exposure_mode": "locked",
      "wb_mode": "locked"
    },
    "lock_drift_events": []
  },
  "video": {
    "format": "h265_4k_30",
    "resolution": "3840x2160",
    "fps": 30,
    "duration_sec": 2553,
    "degraded": false
  },
  "sync": {
    "start_mark_video_ts_ns": 123456800000000,
    "start_mark_imu_ts_ns": 123456800800000,
    "end_mark_drift_ms": 12.4
  },
  "depth": {
    "enabled": true,
    "format": "depth_float32",
    "resolution": "256x192",
    "rate_hz": 10,
    "extrinsic_rgb_to_depth_source": "AVDepthData.cameraCalibrationData",
    "intrinsics_logged_per_frame": true,
    "lock_ae_wb_compatible_verified": true
  },
  "thermal_events": [
    {"ts_ns": 124000000000000, "state": "fair"}
  ],
  "interruption_events": [],
  "counters": {
    "video_frames": 76590,
    "imu_samples": 255300,
    "gnss_fixes": 2553,
    "heading_samples": 2553,
    "depth_frames": 25530
  }
}
```

### 5.6 Capture LiDAR (depth)

**Objectif** : capter une depth map dense alignée pixel-par-pixel avec le flux vidéo, exploitable downstream pour audit géométrique courte portée de la chaussée (profondeur de fissures et nids-de-poule, hauteur de bordure, distance à obstacles proches), et pour amorcer la fusion vidéo+IMU+depth en post.

**Méthode (méthodes optimales retenues)** :

1. **`AVCaptureDepthDataOutput` ajouté à la même `AVCaptureSession`** que le video output. **Pas d'ARKit** : ARKit prend le contrôle complet de la session caméra et réajuste l'AE/WB en continu, ce qui est incompatible avec les locks de §5.1. Le pipeline AVFoundation préserve les locks (à valider explicitement Phase A/B, voir §11).
2. **Synchronisation via `AVCaptureDataOutputSynchronizer`** : chaque `AVDepthData` est livré apparié à un `CMSampleBuffer` vidéo, sur le même `CMClock` host. Pas de drift inter-stream à corriger (différent du cas GNSS/IMU).
3. **Format brut** : `kCVPixelFormatType_DepthFloat32`, résolution native (~256×192 sur LiDAR Apple, à confirmer Phase A pour 16/17 Pro). Pas de conversion lossy, pas de re-encoding.
4. **Cadence downsamplée à 10 Hz** via `activeDepthDataMinFrameDuration = CMTime(value: 1, timescale: 10)` — soit 1 frame depth pour 3 frames vidéo. Suffisant pour usage routier ; divise le coût stockage par 3 vs 30 Hz.
5. **Stockage en `.bin` brut + sidecar JSONL** (format simple, portable, lisible en Python via `numpy.fromfile`) :
   - **Pixels** : fichiers `depth/chunk_NNN.bin`, Float32 little-endian, frames concaténées (256×192 = 196 608 bytes/frame). Chunk = 60 sec de capture (~12 MB par chunk à 10 Hz, ~720 MB / heure).
   - **Métadonnées par frame** : ligne dans `depth.jsonl` (voir §5.2) avec `ts_ns`, `frame_index` dans le chunk, `chunk_file`, intrinsics K (3×3) et extrinsic RGB↔depth (4×4) **lus à chaque frame** depuis `AVDepthData.cameraCalibrationData` (l'OIS et la stabilisation peuvent les faire varier subtilement), et `confidence_summary` (% pixels low/medium/high d'après `AVDepthData.depthDataQuality` + `confidenceMap`).
6. **Calibration intrinsèque depth** : **non requise utilisateur**. `AVDepthData.cameraCalibrationData` fournit `intrinsicMatrix`, `extrinsicMatrix` (rotation+translation depth↔wide), et `lensDistortionLookupTable` à chaque frame. Loggé tel quel ; aucune procédure utilisateur à faire.

**Structure de bundle (récap visuel)** :

```
sessions/<id>/
├── video.mp4
├── gnss.jsonl
├── imu.jsonl
├── heading.jsonl
├── depth.jsonl
├── depth/
│   ├── chunk_000.bin
│   ├── chunk_001.bin
│   └── ...
├── manifest.json
└── logs/app.jsonl
```

**Limites documentées** (à logger, pas à corriger en v1) :

- **Portée pratique** : ~5 m garantis, ~10 m utile, dégradé au-delà. Excellent pour audit chaussée et obstacles proches ; inutile pour tracking véhicules à 30 m+.
- **Plein soleil** : SNR LiDAR (940 nm) chute en présence de rayonnement solaire intense. `confidenceMap` reflète cette dégradation. Sessions midi en plein soleil : depth partiellement exploitable, accepté comme limite documentée. Sessions crépuscule/nuit/intérieur restent excellentes.
- **Stockage additionnel** : ~7 GB/h à 10 Hz / 256×192 (jusqu'à ~11 GB/h si la résolution effective est 320×240 sur 16/17 Pro — à mesurer Phase A). Total session avec depth : 12-15 GB/h vs 5-8 GB/h sans.
- **Format vidéo doit supporter depth synchronisé** : pas tous les `AVCaptureDevice.Format` exposent depth en parallèle. Validation explicite Phase A : confirmer qu'un format 4K 30 fps H.265 ET depth synchronisé existe sur les modèles cibles. Sinon, négocier une descente de gamme vidéo.

---

## 6. Exigences non-fonctionnelles

### 6.1 Performance (cibles testées tôt)

- 4K H.265 30 fps soutenu pendant **30 min sans descente de gamme** à 25°C ambiant pare-brise. **60 min en intérieur tempéré**. Pas de promesse 60 min plein soleil — c'est explicitement testé Phase A.
- IMU 100 Hz sans gap > 50 ms.
- GNSS 1 Hz en zone Mtl centre, sans gap > 5 sec.
- Empreinte disque session avec LiDAR : **12-15 GB / heure** total = vidéo H.265 4K (5-8 GB) + depth 10 Hz Float32 (~7 GB, jusqu'à ~11 GB si résolution effective plus haute). Sans LiDAR : 5-8 GB / heure. ProRes (reporté v1.1) ajouterait 25-45 GB / heure.

### 6.2 Robustesse — comportement honnête

- Verrouillage écran toléré (la session continue), réveil ne perturbe pas la capture.
- **Appel téléphonique entrant** : `AVCaptureSession` est interrompu par iOS — pas de "pause/reprise propre" possible. Comportement v1 : on **arrête proprement** la session avec sync mark de fin et marqueur `interruption: phone_call` dans le manifest. Bundle reste valide.
- **Backgrounding** : iOS ne permet pas la capture caméra en background. Idem appel : on arrête proprement.
- Throttling thermique : pas de plantage, dégradation logged. Si `thermalState = .critical`, arrêt automatique avec `interruption: thermal_critical`.
- Espace disque < 5 GB libre : alerte bloquante avant démarrage.
- Crash mid-session : flush jsonl toutes les 5 sec, mp4 finalisable via segment writer ; bundle partiel reste utilisable jusqu'au dernier flush.

### 6.3 Sécurité et vie privée

- iOS Data Protection class **`.completeUntilFirstUserAuthentication`** par défaut (compatible avec capture pendant écran lock). Pas `.complete` en v1 (incompatible avec session continue après lock).
- Permissions strictement minimales :
  - `NSCameraUsageDescription`
  - `NSLocationWhenInUseUsageDescription` + capability "Location updates" en background
  - `NSMotionUsageDescription`
- Pas d'accès Photos, Contacts, Microphone (audio désactivé).
- Aucun envoi automatique. Transfert manuel uniquement via Files.app.
- Note Loi 25 dans le README du repo, pas dans l'app : la responsabilité de partage incombe à l'utilisateur.

### 6.4 Observabilité

- Logs structurés (JSON lines) dans `sessions/<id>/logs/app.jsonl` : démarrage, verrouillages, événements thermal, GNSS quality drops, lock_drift_events.
- Compteurs finaux dans le manifest (cf §5.5).

### 6.5 Portabilité

- iOS 17+ (cible iOS 18+). iPhone 15 Pro / 16 Pro / 17 Pro.
- Pas d'iPad en v1.
- Localisation app : FR-CA. EN si gain trivial.

---

## 7. Calibration intrinsèque (mode in-app)

**Critique du PRD v1 corrigée** : la calibration doit être faite avec **exactement** la même config `AVCaptureSession` que la captation roulante, sinon la matrice K calibrée ne s'applique pas à la vidéo réelle (Smart HDR / Deep Fusion / Photonic Engine de l'app Camera native altèrent la géométrie).

**Mode calibration in-app** :

- Bouton "Calibrer" sur l'écran d'accueil.
- Active la **même `AVCaptureSession` que la captation roulante** : lentille pinned, format vidéo identique (3840×2160 H.265 30 fps), locks après scène.
- Frames extraites depuis le **video output** (`AVCaptureVideoDataOutputSampleBufferDelegate.captureOutput(_:didOutput:from:)`), **jamais via `AVCapturePhotoOutput`** — le pipeline ISP photo (Smart HDR, Deep Fusion, Photonic Engine) diffère du pipeline vidéo et invaliderait la calibration. C'est exactement le piège que ce mode prétend éviter ; le contourner via le photo output reproduirait le problème.
- L'utilisateur capture **50-70 frames** du damier d'échiquier 10×7 (taille **40 mm**, imprimé rigide) sous angles variés. Le wide-angle iPhone exige plus de frames qu'un objectif standard pour bien contraindre les coefficients de distorsion radiale d'ordre élevé.
- Frames exportées dans `calibration/<timestamp>/frame_NN.png` (conversion CVPixelBuffer → PNG sans re-encoding lossy).
- Traitement OpenCV `calibrateCamera` sur Mac (hors app), produit `K` + `D`.
- Critère d'acceptation : RMS reprojection < 0.5 px.
- Fichier `iphone_<model>_wide_calibration_v1.yaml` versionné Git, **embarqué dans les assets de l'app** et référencé dans chaque manifest.

Si l'iPhone exécutant la session n'a pas de fichier de calibration disponible : alerte au démarrage, captation possible mais manifest marqué `calibration_ref: null`.

**Calibration depth (LiDAR)** : aucune procédure utilisateur. `AVDepthData.cameraCalibrationData` fournit à chaque frame les intrinsics depth, l'extrinsic RGB↔depth, et la lookup table de distorsion. Loggé dans `depth.jsonl` (§5.2 et §5.6). Aucun fichier YAML versionné côté depth ; l'autorité est l'API Apple, par frame.

---

## 8. Plan phasé

Side project solo. Durées indicatives = temps technique pur. **Calendrier réaliste = 2× à 3× cette durée** (soirées + weekends).

| Phase | Durée tech | Livrable | Décision en sortie |
|---|---|---|---|
| **A — Setup + capture + thermal-test** | ~1 sem | Xcode 16 + projet SwiftUI + `AVCaptureSession` 4K H.265, lentille pinned, AF/AE/WB verrouillés avec monitoring unlock, mp4 écrit localement. **Test thermal 30 min pare-brise** (mount RAM Mounts requis avant Phase A, voir A-02). | Vidéo lisible, lentille pinned (vérifié métadonnées), AE/AF/WB stables 30 min. ⚠️ **Validation thermique "plein soleil 30°C+" gated par la saison** : impossible avant juin-juillet 2026 à Montréal. Plan B : test en chambre chauffée (35°C + lampe halogène) pour pré-validation. La décision finale codec/résolution (A-06) ne peut être prise qu'à l'été. **Si test pré-validation tue la session < 30 min : déjà reconsidérer scope (sessions plus courtes, ou switch 1080p par défaut).** |
| **B — Sync rigoureuse + manifest + depth** | ~3 sem | jsonl GNSS/IMU/heading sync via `CMClock` host, sync mark protocole (flash + double-tap + events), manifest complet avec params lock. **Intégration `AVCaptureDepthDataOutput` + `AVCaptureDataOutputSynchronizer`** (§5.6) : format `.bin` brut + `depth.jsonl`, validation que les locks AE/WB tiennent avec depth branché et qu'un format vidéo 4K H.265 supporte depth synchronisé sur l'iPhone cible. Script Python parse + vérifie drift < 50 ms + lit depth bin et confirme alignement RGB↔depth automatiquement. | Bundle de 10 min auto-validé : drift mesurable, format conforme, depth lisible et alignée. **Si lock AE/WB cassé par depth output ou si aucun format 4K+depth disponible : décision GO/NO-GO LiDAR explicite ici** (§11). |
| **C — Calibration in-app + UX minimale** | ~1 sem | Mode calibration in-app produisant frames non-traitées, mount-check démarrage, écran liste sessions, Files.app intégration, swipe-to-delete. Calibration iPhone 16 Pro produite avec RMS < 0.5 px. | Captation solo au volant utilisable sans manuel, calibration exploitable. |
| **D — Robustesse réelle** | ~1.5 sem | Tests 60 min répétés, gestion appel/backgrounding (arrêt propre + bundle valide), thermal events, flush périodique. Validation automatisée bundles. | App stable 60 min indoor + 30 min outdoor, bundles toujours valides après interruption. |
| **E — Polish + buffer** | ~1 sem | Bug fixes, README minimal (incluant note Loi 25), déploiement Apple Developer Program. | App utilisable de façon répétée. |

**Total technique** : ~7.5 sem (+1 sem Phase B pour intégration depth). **Calendrier side project réaliste** : **28-40 semaines (6-9 mois)** — ajout LiDAR (+6-10 sem calendaires) et rabbit holes typiques (calibration intrinsèque, debug sync drift, gestion thermique saisonnière, intégration Files.app, validation lock AE/WB sous depth output, format depth, gestion stockage). À réviser à chaque sortie de phase.

**Apple Developer Program 99 $/an** : à acheter avant Phase A.

---

## 9. Validation

### 9.1 Critères d'acceptation v1

- **Test sessions longues** : 3 sessions de 30 min consécutives à températures variables, zéro plantage, zéro drop frame > 100 ms (cible 60 min reportée à v1.1 si thermal pose problème en Phase A).
- **Bundle structuré** : tous les fichiers attendus présents, manifest valide JSON-Schema, jsonl parsable ligne par ligne.
- **Synchronisation** : drift mesuré automatiquement via sync mark < 50 ms sur 30 min. Pas d'alignement manuel a posteriori.
- **GNSS** : couverture ≥ 95 % du temps de session avec `h_acc` < 10 m en zone Mtl centre.
- **IMU** : couverture continue, gaps < 50 ms.
- **Calibration** : RMS reprojection < 0.5 px sur damier 10×7 / 40 mm, frames extraites du mode in-app.
- **Verrouillage caméra** : `lock_drift_events` ≤ 2 sur 30 min en conditions stables, **avec depth output actif** ; tout drift loggé avec paramètres avant/après. Si l'ajout du depth output déclenche systématiquement des unlocks → décision GO/NO-GO LiDAR (§11).
- **Depth LiDAR** : couverture continue à 10 Hz, gaps < 200 ms, `confidenceMap` disponible pour chaque frame. Alignement RGB↔depth vérifié sur scène connue (cible : <2 px de désalignement à 3 m, mesuré via damier visible dans les deux flux). En plein soleil de midi : confidence dégradée acceptée et loggée, **pas un échec de session**.

### 9.2 Tests automatisés

- XCTest unitaire sur parsing manifest, validation timestamps, format jsonl.
- **Script Python `validate_bundle.py`** (livré avec l'app, exécuté localement) : vérifie structure bundle, parse jsonl, calcule drift sync mark, vérifie compteurs vs durée. Run en CI sur bundles d'exemple.
- Pas d'UI tests automatisés en v1.

---

## 10. Compétiteurs

Aucun concurrent direct sur la combinaison **(capture locale + caméra calibrée verrouillée + IMU 100 Hz + bundle ouvert + zero cloud)**. Étude de marché détaillée déplacée dans `docs/competitive-landscape.md` (à créer).

Positionnement implicite : *"Sensor Logger qui aurait été pensé pour la photogrammétrie / CV routière, local-first."*

---

## 11. Risques (top 8 reévalués v2.1)

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| **OIS + rolling shutter + variation unit-to-unit non documentés dans le dataset** | Certaine (caractéristiques physiques iPhone) | Dataset moins "propre" qu'annoncé ; limite réutilisabilité downstream (VIO, photogrammétrie 3D fine) | Documenté en §5.1 "Limites connues" ; manifest logge `readout_duration_ns` ; calibration par unité possible si plusieurs iPhones. **Honnêteté > marketing interne.** |
| **Branche depth force iOS à libérer les locks AE/WB** | Inconnue (combinaison `AVCaptureDepthDataOutput` + locks à valider) | Calibration intrinsèque inutile, dataset incohérent | **Validation explicite Phase B** comme livrable. Si conflit irrémédiable : choisir lock-AE/WB et désactiver depth (= GO/NO-GO LiDAR). Ne pas découvrir ce conflit en Phase D. |
| **Aucun format vidéo 4K H.265 ne supporte depth synchronisé sur l'iPhone cible** | Possible (matrice format×depth à vérifier) | Forcer descente vidéo 1080p ou abandonner depth | Validation Phase A : énumérer `AVCaptureDevice.formats` filtrés sur `supportedDepthDataFormats`. Décision conditionnelle. |
| **Sync mark fragile si on s'appuie sur signal acoustique** | Élevée | Critère drift < 50 ms invalidable = §9.1 sur du sable | **Double-tap utilisateur** + flash écran (§5.2). Plan B : taptic engine. Validation Phase B inclut détection auto du tap dans `imu.jsonl`. |
| **Depth dégradée en plein soleil (940 nm vs solaire)** | Élevée (limite physique LiDAR) | Sessions midi avec confidence basse, depth peu exploitable sur certains pixels | Loggé via `confidenceMap` et `confidence_summary`, accepté comme limite documentée. Sessions crépuscule/nuit/intérieur restent excellentes. **Aucun critère d'acceptation §9.1 basé sur qualité depth en plein soleil.** |
| **Calibration extrinsèque caméra wide↔IMU absente** | Certaine en v1 | Limite la fusion serrée vidéo+IMU | Logger un estimé Apple-published dans le manifest. RGB↔depth est elle fournie par AVFoundation, donc fusion vidéo+depth+IMU partiellement réalisable. |
| **Phase A thermal "plein soleil" gated par la saison** | Certaine (PRD démarre en avril à Mtl) | Décision A-06 (codec/résolution finale) impossible avant juin-juillet 2026 ; ajout du depth aggrave la charge thermique | Plan B chambre chauffée pour pré-validation. Phases B-C-D peuvent avancer en parallèle en assumant H.265/4K + depth, à valider à l'été. |
| **Calibration intrinsèque invalide si extraction via photo output** | Haute si dev en raccourci | Matrice K wide inutile (ISP photo ≠ ISP vidéo) | **Extraction obligatoire via `AVCaptureVideoDataOutput`** (§7), jamais `AVCapturePhotoOutput`. Test : comparer K calibré vs FoV mesuré sur scène connue. |
| **Side project, dérive de scope + estimation calendaire optimiste** | Élevée | "13-20 sem" → 28-40 sem (6-9 mois) calendaires avec LiDAR | Discipline §4.2 maintenue. Réviser estimation à chaque sortie de phase. ProRes coupé du MVP (A-04). LiDAR validé Phase B avec décision GO/NO-GO. |

Risques pertinents mais secondaires : iOS interrompt session (arrêt propre prévu §6.2), throttling thermique outdoor (testé Phase A), Loi 25 partage (manifest `personal_only: true`, README), mount glisse mid-session (logged via dérive pitch/roll moyenne).

---

## 12. Décisions ouvertes

| ID | Décision | Échéance | Statut |
|---|---|---|---|
| A-01 | Achat Apple Developer Program 99 $/an | Avant Phase A | À faire |
| A-02 | Achat mount pare-brise qualité (RAM Mounts type) | **Avant Phase A** (requis pour test thermal réaliste) | À faire |
| A-03 | Choix iPhone primaire (16 Pro vs 17 Pro selon disponibilité) | Avant Phase A | À confirmer |
| A-04 | Codec par défaut | Phase A | **H.265 seulement en MVP. ProRes reporté v1.1** pour éviter deux codepaths à tester. |
| A-05 | Imprimé damier rigide 10×7 / 40 mm | Avant Phase C | À commander |
| A-06 | Arbitrage codec/résolution selon thermal "plein soleil" | **Été 2026 (juin-juillet)** | Décision conditionnelle, gated par saison |

---

## 13. Glossaire

| Terme | Définition |
|---|---|
| **Bundle de session** | Dossier structuré sur l'iPhone contenant mp4 + jsonl + manifest pour une session unique |
| **Calibration intrinsèque** | Matrice K + coefficients de distorsion D propres à un modèle d'iPhone et une lentille |
| **Mount-check** | Vérification pré-session que l'iPhone est dans une orientation acceptable au pare-brise |
| **Sync mark** | Signal explicite (flash écran + bip + event jsonl) émis au début/fin de session pour mesurer le drift inter-streams |
| **CMClock host time** | Horloge système iOS partagée par AVFoundation et CoreMotion ; référence de sync utilisée v1 |
| **GNSS** | Global Navigation Satellite Systems (GPS, GLONASS, Galileo, BeiDou) |
| **IMU** | Inertial Measurement Unit — accéléromètres + gyroscopes + magnétomètre via CoreMotion |
| **OIS** | Optical Image Stabilization, hardware, non désactivable sur iPhone Pro |
| **ProRes** | Codec Apple intra-frame, ~6× plus volumineux que H.265 mais idéal pour CV downstream |
| **Loi 25** | Loi modernisant la protection des renseignements personnels, Québec |
| **LiDAR** | Light Detection and Ranging — capteur dToF (direct Time of Flight) intégré aux iPhones Pro depuis le 12 Pro, mesure de distance par laser pulsé à 940 nm |
| **dToF** | Direct Time of Flight — mesure directe du temps aller-retour d'une impulsion laser pour estimer une distance |
| **Depth map** | Image où chaque pixel encode une distance en mètres (Float32 sur iPhone) ; résolution native ~256×192 sur LiDAR Apple |
| **Confidence map** | Carte de confiance livrée par `AVDepthData` indiquant la fiabilité de chaque pixel depth (low/medium/high) |
| **`AVCaptureDataOutputSynchronizer`** | API AVFoundation pour appairer plusieurs flux (vidéo + depth notamment) sur le même `CMClock` host, garantissant zéro drift inter-stream |
| **`AVCaptureDepthDataOutput`** | Output AVFoundation exposant les `AVDepthData` du LiDAR (intrinsics, extrinsics RGB↔depth, depth map, confidence map) |

---

*Fin du document. v2.1 du 2026-04-29 — recentrage MVP post-revue critique + ajout LiDAR depth capture.*
