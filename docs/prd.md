# PRD — App iOS de captation Repères

**Date** : 2026-04-29
**Statut** : draft v1
**Auteur** : Julien Riel
**Cible** : livrer une app iOS minimale, autonome, qui capture des sessions de roulage géoréférencées + inertielles, et produit un bundle local exploitable plus tard pour tout traitement post-captation.

> Ce PRD remplace le scope précédent (`docs/prd.md`, pipeline ML + validation IES MTQ). Le pipeline, l'annotation, le scoring κ, la validation Mtl sont **tous reportés à un projet ultérieur**. On livre d'abord l'app, point.

---

## 1. Pourquoi ce pivot

Le PoC v1 précédent combinait app iOS + pipeline Python + fine-tune YOLO + protocole de validation κ + boucle Label Studio + 30+ segments de référence + EFVP Loi 25. En side project solo, c'est ~12 mois calendaires réalistes avec un risque commercial non résolu.

L'app de captation est, à elle seule, un livrable :

- **Valeur intrinsèque** : permet de constituer un dataset personnel propre, calibré, réutilisable pour n'importe quel projet de traitement futur (ML, IRI inertiel, mapping, etc.).
- **Risque borné** : pas de modèle à entraîner, pas de validation statistique, pas d'EFVP complexe (captation personnelle), pas de question commerciale.
- **Délai court** : ~6-8 semaines réalistes pour un side project, vs ~12 mois.
- **Réutilisable** : si on revient un jour au pipeline ML, l'app est déjà faite et le bundle est déjà bien formé.

---

## 2. Vision

Une app iOS native qui transforme un iPhone récent monté au pare-brise en **outil de captation reproductible** pour audit routier ou cartographie. Bundle de session 100 % local, format ouvert, exploitable hors-ligne par un script Python ou QGIS.

**Non-objectifs** :
- Pas de traitement temps réel.
- Pas de cloud, pas de sync, pas de multi-utilisateurs.
- Pas d'analyse ni de score à bord.
- Pas de partage natif vers tiers (transfert via AirDrop/USB seulement).

---

## 3. Cas d'usage

**UC-1 — Captation roulante solo, 30-60 min**
L'auteur monte son iPhone 16 Pro au pare-brise, démarre l'app, vérifie le mount-check, lance la session, conduit, arrête. Bundle écrit localement.

**UC-2 — Passages multiples même parcours**
Trois passages d'un même boulevard à des heures différentes (matin/midi/soir) pour comparaison ultérieure.

**UC-3 — Transfert vers Mac post-session**
Bundle transféré via AirDrop ou USB. L'app n'est pas un visualiseur — la suite vit ailleurs.

---

## 4. Périmètre

### 4.1 In-scope

- App iOS native SwiftUI, iPhone 16 Pro / 17 Pro cible, iPhone 15 Pro fallback.
- Capture **vidéo 4K H.265 30 fps** via `AVCaptureSession`.
- **Lentille `builtInWideAngleCamera` pinned** (pas de virtual device), AF/exposition/WB verrouillés après scène de calibration courte.
- Logging **GNSS** via `kCLLocationAccuracyBestForNavigation` (1-10 Hz natif).
- Logging **IMU à 100 Hz** via `CMDeviceMotion` mode `xMagneticNorthZVertical`.
- Logging **cap vrai** via `CLHeading.trueHeading`.
- **Mount-check pré-session** : vérification pitch ±10°, alerte bloquante si hors plage.
- **Manifest de session** : device, version app, lentille, ref calibration intrinsèque, codec, dates.
- Persistance locale dans bundle structuré (mp4 + jsonl + manifest).
- Permissions iOS strictement minimales (caméra, localisation, motion ; pas Photos, pas contacts).
- Robustesse : tolère mise en veille écran, appel entrant, backgrounding momentané (≤ 30 s), throttling thermique avec dégradation propre (logged).

### 4.2 Out-of-scope

- Tout pipeline post-captation (extraction frames, snap-to-road, etc.).
- Tout ML embarqué.
- Annotation, révision, scoring.
- Floutage temps réel ou différé.
- Synchronisation cloud, multi-device, partage entre utilisateurs.
- UI de visualisation des sessions à bord (juste lister/supprimer suffit).
- Suivi temporel inter-passages.
- Saisie utilisateur de métadonnées véhicule (auto-fill avec valeur fixe v1).
- Conditions météo, type de chaussée, etc. — ajoutés plus tard si vraiment utile.

---

## 5. Exigences fonctionnelles

### 5.1 Configuration caméra

1. Lentille `builtInWideAngleCamera` pinned explicitement (jamais `builtInDualCamera` ou `builtInTripleCamera`).
2. Au démarrage de session : 5-10 sec de scène typique pour AE/AF auto, puis verrouillage `lockForConfiguration` + `setExposureMode(.locked)` + `setFocusMode(.locked)` + `setWhiteBalanceMode(.locked)`.
3. Format vidéo : 3840×2160 H.265 30 fps, conteneur `.mp4`.
4. Stabilisation logicielle : **désactivée** (préserve la signature inertielle de la suspension véhicule pour usage IMU futur).
5. Si throttling thermique force une descente en gamme (ex. 1080p) : la session continue, mais le manifest logge la transition et la session est marquée `degraded: true`.

### 5.2 Logs synchronisés

Tous les logs sont horodatés via `CMClock` mappé sur `CLLocation.timestamp` pour cohérence inter-flux.

- `gnss.jsonl` : un fix par ligne, format `{ts, lat, lon, alt, h_acc, v_acc, course, speed_mps}`.
- `imu.jsonl` : un sample par ligne à 100 Hz, format `{ts, quat, gyro, accel, user_accel}`.
- `heading.jsonl` : un sample par ligne, format `{ts, true_heading, accuracy}`.
- Timestamps ISO 8601 avec timezone (`-04:00` ou `-05:00` selon DST QC).

### 5.3 Mount-check

- À l'ouverture de l'écran "Nouvelle session", lecture continue du pitch via `CMDeviceMotion`.
- Affichage visuel d'un niveau à bulle (range ±15°).
- Bouton "Démarrer la session" actif uniquement si pitch ∈ [-10°, +10°] **stable pendant 3 sec**.
- Pas de moyenne de session enregistrée v1 (différé).

### 5.4 Cycle de session

1. Écran d'accueil : liste des sessions existantes (id, date, durée, taille), bouton "Nouvelle session".
2. Écran "Nouvelle session" : preview caméra + niveau à bulle + bouton "Démarrer" (gated par mount-check).
3. Pendant session : preview caméra avec overlay minimal (durée écoulée, distance estimée, qualité GNSS, alerte si throttling).
4. Bouton "Arrêter" : flush des fichiers, écriture du `manifest.json`, retour à l'accueil.
5. Détail de session : afficher manifest, taille fichiers, bouton "Partager via AirDrop", bouton "Supprimer".

### 5.5 Manifest minimal

```json
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
    "calibration_ref": "iphone_16pro_wide_calibration_v1",
    "stabilization": "off",
    "ae_locked": true,
    "af_locked": true,
    "wb_locked": true
  },
  "video": {
    "codec": "h265",
    "resolution": "3840x2160",
    "fps": 30,
    "duration_sec": 2553,
    "degraded": false
  },
  "thermal_events": [],
  "background_events": []
}
```

---

## 6. Exigences non-fonctionnelles

### 6.1 Performance

- 4K H.265 30 fps soutenu pendant 60 min minimum à température ambiante (cible été : 30°C).
- IMU 100 Hz sans gap > 50 ms.
- GNSS sans gap > 2 sec dans des conditions à ciel ouvert.
- Empreinte disque session ≈ 5-8 GB / heure.

### 6.2 Robustesse

- Aucun plantage app pendant 60 min de captation.
- Verrouillage écran toléré (la session continue), réveil ne perturbe pas la capture.
- Appel téléphonique entrant : la session pause proprement (audio cédé) puis reprend, ou s'arrête proprement avec marqueur dans manifest.
- Throttling thermique : pas de plantage, dégradation logged.
- Espace disque < 5 GB libre : alerte bloquante avant démarrage.
- Crash mid-session : le bundle partiel doit rester ré-ouvrable (flush périodique du jsonl, mp4 finalisable même sur stop forcé).

### 6.3 Sécurité et vie privée

- iOS Data Protection activé (par défaut). Vérification au premier lancement.
- Permissions strictement minimales :
  - `NSCameraUsageDescription`
  - `NSLocationWhenInUseUsageDescription` + capability "Location updates" en background
  - `NSMotionUsageDescription`
- Pas d'accès Photos library, Contacts, Microphone (audio désactivé sur la captation).
- Aucun envoi automatique. Transfert manuel uniquement.
- Avant captation hors zones personnelles : disclaimer visible "Captation à des fins privées, conforme Loi 25, ne pas partager hors poste sans floutage".

### 6.4 Observabilité

- Logs structurés (JSON lines) dans `sessions/<id>/logs/app.jsonl` : démarrage, verrouillages, événements thermal, backgrounding, GNSS quality drops.
- Compteur visible en fin de session : nb frames vidéo, nb samples IMU, nb fixes GNSS, durée totale, taille bundle.

### 6.5 Portabilité

- iOS 17+ (cible iOS 18+). iPhone 15 Pro / 16 Pro / 17 Pro.
- Pas d'iPad en v1.
- Localisation app : français QC (FR-CA), anglais en backup.

---

## 7. Calibration intrinsèque

Avant tout usage opérationnel, **une fois par modèle d'iPhone** :

- Damier d'échiquier 10×7, taille 25 mm, imprimé rigide.
- 25-40 images variées en 4K, AF verrouillé, lentille `builtInWideAngleCamera` pinned.
- Traitement OpenCV `calibrateCamera` sur Mac (hors app), produit `K` (matrice intrinsèque) + `D` (distorsion).
- Critère d'acceptation : RMS < 0.5 pixel.
- Fichier `iphone_<model>_wide_calibration_v1.yaml` versionné Git, **embarqué dans l'app** (assets) et référencé dans chaque manifest de session.

L'app n'effectue pas la calibration, elle référence le fichier embarqué. Si un futur iPhone est utilisé sans fichier de calibration disponible : alerte au démarrage, captation possible mais manifest marqué `calibration_ref: null`.

---

## 8. Plan phasé

Side project solo, durées indicatives à réviser après Phase A.

| Phase | Durée indic. | Livrable | Décision en sortie |
|---|---|---|---|
| **A — Setup + capture vidéo** | ~1 sem | Xcode 16 + projet SwiftUI + `AVCaptureSession` 4K H.265, lentille pinned, AF/AE/WB verrouillés, mp4 écrit localement | Vidéo 5 min lisible sur Mac, lentille effectivement pinned (vérifié dans métadonnées) |
| **B — Logs GNSS + IMU + heading + manifest** | ~1.5 sem | jsonl synchronisés, manifest.json complet, écriture atomique en fin de session | Bundle de 5 min parsable par script Python tiers |
| **C — Mount-check + UX session** | ~1 sem | Niveau à bulle ±10°, écran liste sessions, écran détail, bouton AirDrop, bouton Supprimer | Captation solo au volant utilisable sans manuel |
| **D — Robustesse + tests réels** | ~1.5 sem | Tests 60 min continus, scénarios verrouillage écran / appel / throttling, logs thermal, calibration intrinsèque iPhone 16 Pro produite | App stable 60 min, bundle valide, calibration RMS < 0.5 px |
| **E — Polish + buffer** | ~1 sem | Bug fixes découverts en D, doc README minimal, déploiement Apple Developer Program | App utilisable de façon répétée |

**Total indicatif** : ~6 sem élapsé optimiste, **plutôt 8-10 sem calendaires** pour un side project (variabilité personnelle, météo, dérapage).

**Apple Developer Program 99 $/an** : à acheter avant Phase A. Le free-provisioning expire tous les 7 jours et c'est insupportable pendant un cycle de dev intensif.

---

## 9. Validation

### 9.1 Critères d'acceptation v1

- **Test sessions longues** : 3 sessions de 60 min consécutives à températures variables (matin frais, midi chaud), zéro plantage, zéro drop frame > 100 ms.
- **Bundle structuré** : tous les fichiers attendus présents, manifest valide JSON-Schema, jsonl parsable ligne par ligne.
- **Synchronisation** : drift estimé entre vidéo et IMU < 50 ms sur 60 min (vérifié post-hoc en alignant un événement net comme un dos d'âne).
- **GNSS** : couverture ≥ 95 % du temps de session avec `h_acc` < 10 m en zone Mtl centre.
- **IMU** : couverture continue, gaps < 50 ms.
- **Calibration** : RMS reprojection < 0.5 px sur damier 10×7.
- **Verrouillage caméra** : vérifier dans métadonnées vidéo (ou via post-traitement) que AF/AE/WB sont effectivement stables sur 60 min.

### 9.2 Tests automatisés

- XCTest unitaire sur parsing manifest, validation timestamps, format jsonl.
- Pas d'UI tests automatisés en v1 (manuel suffit pour app solo).

---

## 10. Compétiteurs et positionnement

Aucun concurrent direct sur la combinaison **(capture locale + caméra calibrée verrouillée + IMU 100 Hz + bundle ouvert + zero cloud)**. Le marché segmente en familles distinctes :

### 10.1 Captation pour mapping ouvert

| Produit | Modèle | Différence vs Repères |
|---|---|---|
| **Mapillary** (Meta) | App iOS/Android, capture rue, upload cloud, contribue à OSM-like | Cloud-first, format propriétaire, AE/AF non verrouillés, IMU non exposé |
| **KartaView** (ex-OpenStreetCam, Grab/Telenav) | App iOS/Android, capture rue contributive | Idem Mapillary, cloud, pas de verrouillage caméra |

### 10.2 Capture commerciale audit chaussée

| Produit | Modèle | Différence vs Repères |
|---|---|---|
| **Roadroid** (Suède) | App iOS payante (abonnement), road roughness via accéléromètre, calibration véhicule semi-rigoureuse | Closed pipeline, focus IRI inertiel, pas de vidéo HQ, données restent dans Roadroid |
| **Total Pave RC** | App iPad payante, captation + scoring chaussée | SaaS B2B, fermé, prix élevé |
| **Vialytics** (Allemagne) | Smartphone monté + plateforme SaaS, vendu aux villes | Solution complète SaaS, pas extractible |
| **Carbin** (MIT spinout) | App smartphone road condition | Service freemium, données partiellement publiques mais pipeline fermé |
| **RoadBotics** (acquis par Michael Baker) | Service complet capture + analyse | B2B service, pas une app indépendante |
| **PaveCheck** (partenaire Esri) | Solution intégrée Esri | Bureau, pas mobile, pas comparable |

### 10.3 Loggers de capteurs génériques

| Produit | Modèle | Différence vs Repères |
|---|---|---|
| **Sensor Logger** (par Tszheichoi) | App multi-capteurs, GNSS + IMU + vidéo séparée | Très flexible mais aucun verrouillage caméra, format pas pensé pour CV downstream, calibration intrinsèque absente |
| **Phyphox** (RWTH Aachen) | App éducative capteurs | Vidéo non intégrée, focus pédagogie |
| **Allan Variance / GNSS Logger** | Apps spécialisées géodésiques | Pas de vidéo |

### 10.4 Dashcams / drive recorders

| Produit | Modèle | Différence vs Repères |
|---|---|---|
| **Nexar** (Israël) | App dashcam, vidéo + GNSS, cloud sécurité routière | Pas de IMU exposé, AE auto, format propriétaire |
| **Driver** / divers | App dashcam | Idem Nexar, focus accidents |

### 10.5 Positionnement Repères-app

L'app n'est **pas un produit commercial** en v1. Son positionnement implicite, si elle devait être partagée :

- **Outil de captation reproductible pour ML downstream** — caméra calibrée verrouillée, IMU haute fréquence, format ouvert documenté.
- **Local-first, privacy-first** — aucun cloud, aucun upload, contrôle total sur les données.
- **Bundle ouvert** — tout consommable par script Python ou QGIS sans SDK propriétaire.

À comparer à : *"Sensor Logger qui aurait été pensé pour la photogrammétrie / CV routière"*.

Une éventuelle ouverture publique (open-source ou App Store gratuite) est une décision Phase 2+ qui dépendra de l'usage personnel sur 6-12 mois.

---

## 11. Risques

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| Throttling thermique sévère en été tue les sessions > 30 min | Moyenne | Sessions tronquées | Test explicite Phase D à 30°C ; dégradation logged ; pas une dead-end (1080p reste utilisable plus tard) |
| Drift de synchronisation vidéo / IMU > 50 ms | Moyenne | Données IMU difficilement utilisables pour fusion | Utiliser `CMClock` + `presentationTimeStamp` AVFoundation, pas `Date()`. Tester drift Phase B. |
| `builtInWideAngleCamera` change de comportement entre iOS releases | Faible | Calibration invalide | Embarquer la calibration dans les assets, marquer manifest avec ref ; refaire calibration si iOS majeur change |
| Free-provisioning insupportable sans Apple Developer Program | Élevée | Friction quotidienne | Acheter Dev Program 99 $ avant Phase A, pas après |
| Loi 25 sur captation rue — plaques / visages dans la vidéo brute | Moyenne | Risque légal en cas de partage | Ne jamais partager mp4 brute hors poste personnel ; disclaimer visible dans app ; floutage = projet futur si besoin |
| Side project, dérive de scope (ajouter saisie véhicule, météo, etc.) | Élevée | Délai Phase D | Discipline : tout enrichissement métadonnées = v2. v1 = auto-fill |
| iPhone personnel cassé / volé pendant Phase D | Faible | Retard | Nuage iCloud sur backup setup Xcode + AppleID, pas sur sessions (qui restent locales par design) |
| Mount au pare-brise glisse / mauvais cadrage discret | Moyenne | Sessions inexploitables | Mount RAM ou similaire qualité ; mount-check obligatoire avant chaque session |

---

## 12. Décisions ouvertes

| ID | Décision | Échéance | Statut |
|---|---|---|---|
| A-01 | Achat Apple Developer Program 99 $/an | Avant Phase A | À faire |
| A-02 | Achat mount pare-brise qualité (RAM Mounts type) | Avant Phase D | À faire |
| A-03 | Choix iPhone primaire (16 Pro vs 17 Pro selon disponibilité) | Avant Phase A | À confirmer |
| A-04 | Vérification iOS Data Protection actif sur iPhone | Avant Phase A | À faire |
| A-05 | Licence app : privée Git → ouverture future ? | Phase 2+ | Reportée |
| A-06 | Disclaimer Loi 25 dans l'app : texte exact | Avant Phase D | À rédiger |

---

## 13. Glossaire

| Terme | Définition |
|---|---|
| **Bundle de session** | Dossier structuré sur l'iPhone contenant mp4 + jsonl + manifest pour une session unique |
| **Calibration intrinsèque** | Matrice K + coefficients de distorsion D propres à un modèle d'iPhone et une lentille |
| **Mount-check** | Vérification pré-session que l'iPhone est dans une orientation acceptable au pare-brise |
| **GNSS** | Global Navigation Satellite Systems (GPS, GLONASS, Galileo, BeiDou) |
| **IMU** | Inertial Measurement Unit — accéléromètres + gyroscopes + magnétomètre, exposé via CoreMotion |
| **CMClock** | Horloge système iOS pour synchronisation média + capteurs |
| **Free provisioning** | Déploiement Xcode avec Apple ID gratuit, profil 7 jours |
| **Apple Developer Program** | Abonnement payant 99 $/an, profil 1 an, App Store si désiré |
| **Loi 25** | Loi modernisant la protection des renseignements personnels, Québec |

---

*Fin du document. 2026-04-29.*
