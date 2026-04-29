# PRD — App iOS de captation Repères

**Date** : 2026-04-29
**Statut** : draft v2 (MVP serré)
**Auteur** : Julien Riel
**Cible** : livrer une app iOS minimale, autonome, qui capture des sessions de roulage géoréférencées + inertielles, et produit un bundle local exploitable plus tard pour tout traitement post-captation.

> v2 du PRD : recentrage MVP suite revue critique. Tout ce qui ne sert pas directement le besoin "dataset personnel propre, calibré, réutilisable" est sorti du scope v1.

---

## 1. Pourquoi ce pivot

Le PoC v1 précédent combinait app iOS + pipeline Python + fine-tune YOLO + protocole de validation κ + boucle Label Studio + 30+ segments de référence + EFVP Loi 25. En side project solo, c'est ~12 mois calendaires réalistes avec un risque commercial non résolu.

L'app de captation est, à elle seule, un livrable :

- **Valeur intrinsèque** : permet de constituer un dataset personnel propre, calibré, réutilisable pour n'importe quel projet de traitement futur (ML, IRI inertiel, mapping, etc.).
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
- Logging **cap vrai** via `CLHeading.trueHeading`.
- **Sync multi-streams rigoureuse** via `CMClock` partout (voir §5.2) + protocole de sync mark au début/fin de session.
- **Mount-check au démarrage** : vérification pitch + roll, alerte bloquante si hors plage. Pas de check continu en cours de session.
- **Mode calibration intrinsèque in-app** : capture du damier dans la même config `AVCaptureSession` que la captation roulante (voir §7).
- Manifest de session complet : device, version app, lentille, ref calibration, codec, dates, **paramètres caméra exacts au lock**, événements thermal/interruption.
- Persistance locale dans bundle structuré (mp4 + jsonl + manifest) accessible via Files.app.
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

---

## 5. Exigences fonctionnelles

### 5.1 Configuration caméra

1. Lentille `builtInWideAngleCamera` pinned explicitement (jamais `builtInDualCamera` / `builtInTripleCamera`).
2. Au démarrage de session : 5-10 sec de scène typique pour AE/AF/WB auto, puis verrouillage `lockForConfiguration` + `setExposureMode(.locked)` + `setFocusMode(.locked)` + `setWhiteBalanceMode(.locked)`.
3. **Monitoring d'unlock iOS** : KVO sur `adjustingExposure`, `adjustingFocus`, `adjustingWhiteBalance` ; toute reprise auto par iOS est loggée comme événement (`camera_unlock` avec timestamp + raison) et le manifest marque `lock_drift_events: N`.
4. **Paramètres exacts loggés au lock** dans le manifest : `iso`, `exposureDuration`, `lensPosition`, `whiteBalanceGains` (R/G/B), `focusMode`, `exposureMode`. Permet la comparaison/normalisation entre sessions en post-traitement.
5. Format vidéo par défaut : 3840×2160 H.265 30 fps, conteneur `.mp4`.
6. **Toggle ProRes** (réglage avancé) : si activé, ProRes 4K 30 fps. ~6× plus de stockage, mais pas de B-frames et meilleure fidélité pour CV downstream. Désactivé par défaut.
7. Stabilisation logicielle : **désactivée** via `preferredVideoStabilizationMode = .off`. Attention : l'**OIS hardware** reste actif et n'est pas désactivable ; à compenser en post si nécessaire pour fusion IMU.
8. Audio caméra : **désactivé** sur la captation (privacy + simplicité).
9. Si throttling thermique force une descente en gamme (ex. 1080p) : la session continue, le manifest logge la transition, la session est marquée `degraded: true`.

### 5.2 Synchronisation multi-streams

**Principe** : un seul timebase de référence, `CMClock` host time (`CMClockGetHostTimeClock()`).

- Vidéo : `CMSampleBuffer.presentationTimeStamp` est nativement en `CMClock` host. Aucun mapping nécessaire.
- IMU : `CMDeviceMotion.timestamp` est en `mach_absolute_time` (même base que `CMClock` host) — conversion triviale.
- GNSS : `CLLocation.timestamp` est en `Date` (temps mur). On capture **simultanément** `Date()` et `CMClockGetTime(CMClockGetHostTimeClock())` à `T₀` (ouverture session) pour établir l'offset, et on convertit chaque fix GNSS dans le timebase vidéo.
- Heading : idem GNSS.

**Sync mark protocole** : au début ET à la fin de chaque session, l'app émet :
- Un **flash écran** blanc/noir 200 ms (visible dans la vidéo).
- Un **bip court via haut-parleur** (capté dans accel via vibration micro-structure du téléphone — détectable post-hoc dans `imu.jsonl`).
- Un **événement explicite** dans tous les jsonl avec timestamp `CMClock`.

→ Permet une **vérification automatique de drift** post-session : `Δ(flash_video_timestamp, sync_mark_imu_timestamp) < 50 ms` est le critère mesurable.

**Format des jsonl** (timestamps tous en nanosecondes depuis boot, `CMClock` host) :
- `gnss.jsonl` : `{ts_ns, lat, lon, alt, h_acc, v_acc, course, speed_mps}`.
- `imu.jsonl` : `{ts_ns, quat: [w,x,y,z], gyro: [x,y,z], accel: [x,y,z], user_accel: [x,y,z]}`.
- `heading.jsonl` : `{ts_ns, true_heading_deg, accuracy_deg}`.

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
5. **Suppression** : swipe-to-delete sur la liste. C'est tout.

Pas de bouton AirDrop, pas d'écran détail. Files.app fait le partage et l'inspection.

### 5.5 Manifest minimal

```json
{
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
  "thermal_events": [
    {"ts_ns": 124000000000000, "state": "fair"}
  ],
  "interruption_events": [],
  "counters": {
    "video_frames": 76590,
    "imu_samples": 255300,
    "gnss_fixes": 2553,
    "heading_samples": 2553
  }
}
```

---

## 6. Exigences non-fonctionnelles

### 6.1 Performance (cibles testées tôt)

- 4K H.265 30 fps soutenu pendant **30 min sans descente de gamme** à 25°C ambiant pare-brise. **60 min en intérieur tempéré**. Pas de promesse 60 min plein soleil — c'est explicitement testé Phase A.
- IMU 100 Hz sans gap > 50 ms.
- GNSS 1 Hz en zone Mtl centre, sans gap > 5 sec.
- Empreinte disque session ≈ 5-8 GB / heure (H.265). ProRes ~30-50 GB / heure.

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
- Active la même `AVCaptureSession` (lentille pinned, format identique, locks après scène) mais en mode **photo** (extraction de frames individuelles).
- L'utilisateur capture 25-40 frames du damier d'échiquier 10×7 (taille **40 mm**, imprimé rigide) sous angles variés.
- Frames exportées dans `calibration/<timestamp>/frame_NN.png` (PNG 16-bit non compressé).
- Traitement OpenCV `calibrateCamera` sur Mac (hors app), produit `K` + `D`.
- Critère d'acceptation : RMS reprojection < 0.5 px.
- Fichier `iphone_<model>_wide_calibration_v1.yaml` versionné Git, **embarqué dans les assets de l'app** et référencé dans chaque manifest.

Si l'iPhone exécutant la session n'a pas de fichier de calibration disponible : alerte au démarrage, captation possible mais manifest marqué `calibration_ref: null`.

---

## 8. Plan phasé

Side project solo. Durées indicatives = temps technique pur. **Calendrier réaliste = 2× à 3× cette durée** (soirées + weekends).

| Phase | Durée tech | Livrable | Décision en sortie |
|---|---|---|---|
| **A — Setup + capture + thermal-test** | ~1 sem | Xcode 16 + projet SwiftUI + `AVCaptureSession` 4K H.265, lentille pinned, AF/AE/WB verrouillés avec monitoring unlock, mp4 écrit localement. **Test thermal 30 min pare-brise plein soleil.** | Vidéo lisible, lentille pinned (vérifié métadonnées), AE/AF/WB stables 30 min. **Si thermal tue la session < 30 min : reconsidérer scope (sessions plus courtes, ou switch 1080p par défaut).** |
| **B — Sync rigoureuse + manifest** | ~2 sem | jsonl GNSS/IMU/heading sync via `CMClock` host, sync mark protocole (flash + bip + events), manifest complet avec params lock. Script Python de test parse + vérifie drift < 50 ms automatiquement. | Bundle de 10 min auto-validé par script Python tiers (drift mesurable, format conforme). |
| **C — Calibration in-app + UX minimale** | ~1 sem | Mode calibration in-app produisant frames non-traitées, mount-check démarrage, écran liste sessions, Files.app intégration, swipe-to-delete. Calibration iPhone 16 Pro produite avec RMS < 0.5 px. | Captation solo au volant utilisable sans manuel, calibration exploitable. |
| **D — Robustesse réelle** | ~1.5 sem | Tests 60 min répétés, gestion appel/backgrounding (arrêt propre + bundle valide), thermal events, flush périodique. Validation automatisée bundles. | App stable 60 min indoor + 30 min outdoor, bundles toujours valides après interruption. |
| **E — Polish + buffer** | ~1 sem | Bug fixes, README minimal (incluant note Loi 25), déploiement Apple Developer Program. | App utilisable de façon répétée. |

**Total technique** : ~6.5 sem. **Calendrier side project réaliste** : **13-20 semaines.**

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
- **Verrouillage caméra** : `lock_drift_events` ≤ 2 sur 30 min en conditions stables ; tout drift loggé avec paramètres avant/après.

### 9.2 Tests automatisés

- XCTest unitaire sur parsing manifest, validation timestamps, format jsonl.
- **Script Python `validate_bundle.py`** (livré avec l'app, exécuté localement) : vérifie structure bundle, parse jsonl, calcule drift sync mark, vérifie compteurs vs durée. Run en CI sur bundles d'exemple.
- Pas d'UI tests automatisés en v1.

---

## 10. Compétiteurs

Aucun concurrent direct sur la combinaison **(capture locale + caméra calibrée verrouillée + IMU 100 Hz + bundle ouvert + zero cloud)**. Étude de marché détaillée déplacée dans `docs/competitive-landscape.md` (à créer).

Positionnement implicite : *"Sensor Logger qui aurait été pensé pour la photogrammétrie / CV routière, local-first."*

---

## 11. Risques (top 5 reévalués)

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| **Sync vidéo/IMU drift > 50 ms non détecté** | Moyenne-Haute | Dataset inutilisable pour fusion | Sync mark automatisé + script Python de validation. **Test sync drift est un livrable Phase B**, pas une vérification post-hoc. |
| **Throttling thermique tue les sessions outdoor** | Haute en été | Sessions tronquées à 15-25 min | **Test Phase A explicite** (pas Phase D). Si problème : sessions plus courtes acceptées, ou 1080p par défaut, ou mount avec dissipation. |
| **Calibration intrinsèque invalide (config camera ≠ capture)** | Haute si calib externe | Matrice K inutile | **Mode calibration in-app obligatoire**, même `AVCaptureSession` config. |
| **Side project, dérive de scope** | Élevée | Délai indéfini | Discipline : tout out-of-scope §4.2 reste out. v1 = liste + 2 boutons + calibration + bundle. |
| **iOS interrompt session (appel/background)** | Élevée (chaque appel) | Sessions tronquées | Comportement honnête : arrêt propre + bundle valide + marqueur. Pas de "pause magique". |

Risques v1 du PRD précédent (Loi 25 partage, mount glisse, free-provisioning) toujours pertinents mais moins critiques que les 5 ci-dessus.

---

## 12. Décisions ouvertes

| ID | Décision | Échéance | Statut |
|---|---|---|---|
| A-01 | Achat Apple Developer Program 99 $/an | Avant Phase A | À faire |
| A-02 | Achat mount pare-brise qualité (RAM Mounts type) | Avant Phase D | À faire |
| A-03 | Choix iPhone primaire (16 Pro vs 17 Pro selon disponibilité) | Avant Phase A | À confirmer |
| A-04 | Décision codec par défaut : H.265 (économique) ou ProRes (CV-friendly) | Phase A | H.265 par défaut, toggle ProRes en advanced |
| A-05 | Imprimé damier rigide 10×7 / 40 mm | Avant Phase C | À commander |
| A-06 | Arbitrage en sortie de Phase A si thermal tue 30 min outdoor | Sortie Phase A | Décision conditionnelle |

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

---

*Fin du document. v2 du 2026-04-29 — recentrage MVP post-revue critique.*
