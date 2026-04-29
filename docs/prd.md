# PRD — Repères · Plateforme d'audit automatisé d'actifs viaires municipaux

**Date** : 2026-04-29
**Statut** : draft v1 fail-fast, en attente de validation produit
**Auteur** : Julien Riel, avec assistance Claude
**Cible PoC v1** : **Un seul module livrable** — PaveAudit visuel (état de chaussée selon méthodologie MTQ, vision uniquement), validé empiriquement à **Montréal seulement**. Tout le reste (ChantierWatch, branche inertielle, zone secondaire) est différé Phase 2.

> Ce document est la **seule source de vérité produit** pour Repères. Il remplace toute spec ou plan antérieur. Les spécifications d'implémentation (architecture, code, séquence des tâches) en découlent.

---

## 1. Vision

Construire à terme une **plateforme d'audit automatisé d'actifs viaires municipaux** par captation vidéo géoréférencée et inertielle depuis un véhicule. La vision long terme reste une architecture multi-modules (vision + inertiel + permits OdP + signalisation + mobilier urbain). Mais le PoC v1 est volontairement **réduit au minimum permettant de fail fast** sur l'hypothèse centrale.

**Le PoC v1 livre un seul module : PaveAudit visuel** — détection de défauts de chaussée selon la méthodologie d'évaluation visuelle MTQ, fine-tuné sur dataset Québec hivernal, validé empiriquement sur 30+ segments de référence à Montréal, livré comme GeoPackage exploitable QGIS.

**Reportés Phase 2** (cf. §12) :
- **ChantierWatch** (détection chantiers + cross-check permits OdP)
- **Branche inertielle PaveAudit** (proxy IRI via IMU normalisé vitesse)
- **Zone secondaire** (Longueuil ou Laval)
- **Rapport PDF Quarto** client (livraison v1 = GeoPackage seul)
- **Service OGC API Features** via pygeoapi
- **UI annotation custom** (v1 = Label Studio local)
- **Architecture pluggable plugin entry points** (v1 = monolithique, imports directs)
- **Pipeline parallèle ProcessPoolExecutor** (v1 = séquentiel)

**Principe fail-fast** : la Phase 0 est un spike de 2 semaines qui valide le risque #1 (la baseline ML donne-t-elle un signal exploitable sur les défauts hivernaux du Québec ?) **avant** tout développement iOS ou pipeline. Si la baseline rate, on stop ou on pivote sans avoir construit l'infrastructure.

L'**annotation humaine** s'appuie sur **Label Studio local** (open-source mature). Aucune UI custom en v1.

L'architecture est **monolithique en v1** : orchestration séquentielle, imports directs Python, écriture directe dans un GeoPackage de session. Aucun service externe, aucun Docker, 100 % local hors-ligne.

> **Note** : la stratégie commerciale (audiences clients, cycles de vente, tarification, pitch, incorporation) est traitée séparément dans `docs/marketing/`. Le présent PRD se concentre sur le produit, son architecture et sa validation.

---

## 2. Contexte et motivation

### 2.1 Pivot stratégique

- **Réduit le risque technique** : la précision géométrique requise descend à ±10 m (segment de rue), résolue par snap-to-road sur Géobase/OSM. Les problèmes de heading, calibration extrinsèque, et triangulation disparaissent.
- **Permet d'exploiter un référentiel québécois** : entraînement sur défauts hivernaux du Québec, reconnaissance de la signalisation MUTCD-Québec, intégration des données ouvertes municipales (Données Québec, Géobase, permits OdP).

### 2.2 Pourquoi maintenant

- Les **modèles open-vocabulary** (Grounding DINO, YOLO-World, SAM) atteignent en 2025-2026 une qualité suffisante pour la pré-annotation automatique sur classes arbitraires définies par texte. Cela réduit drastiquement le coût d'annotation initial.
- Les **iPhone récents** (16 Pro et postérieurs) embarquent un GNSS bi-bande, une centrale inertielle de qualité (accéléromètres + gyroscope + magnétomètre à 100 Hz), et une caméra 4K HDR ProRes — accessibles directement via `AVFoundation` + `CoreLocation` + `CoreMotion`. Un dispositif unique remplace ce qui demandait précédemment caméra + GNSS + IMU + ordinateur de captation. **L'exploitation conjointe vidéo + inertiel sur smartphone est inédite sur le marché québécois** — c'est le différenciateur structurel de Repères vs RoadBotics, Vialytics ou Carbin.
- Les **données ouvertes municipales** (Géobase, permits OdP, entraves planifiées) sont publiées en continu par Montréal et plusieurs villes secondaires. Le cross-référencement automatique devient techniquement faisable.
- Les **standards OGC API Features** sont matures et adoptés par les SIG modernes (QGIS, Esri, géoportails municipaux). Une livraison standard est immédiatement consommable.

### 2.3 Ce que **n'est pas** Repères à ce stade

- Ce n'est **pas** un outil de positionnement précis Niveau B/C.
- Ce n'est **pas** un système temps réel — toute évaluation est post-captation.
- Ce n'est **pas** un service SaaS multi-utilisateurs avec auth/dashboard. Le PoC v1 est mono-utilisateur, localhost.
- Ce n'est **pas** un produit clé-en-main certifié pour responsabilité légale. Le livrable est un *signal d'audit*, pas une *preuve* — la décision d'enforcement reste au client.
- Ce n'est **pas** une mesure d'IRI (International Roughness Index) certifiée ASTM E1926. PaveAudit v1 produit l'**indice visuel MTQ** uniquement. Un **score inertiel proxy** (IMU + vitesse) est prévu Phase 2 comme signal complémentaire, mais il restera non calibré véhicule par véhicule.

---

## 3. Audiences et cas d'usage

### 3.1 Audiences produit

Profils d'utilisateurs visés par les fonctionnalités du PoC v1. Cadre les exigences fonctionnelles (formats de livrable, ergonomie, terminologie) — pas la stratégie d'aller-au-marché, qui est traitée dans `docs/marketing/`.

| Audience | Cas d'usage produit | Module pertinent |
|---|---|---|
| Inspecteurs certifiés réalisant des audits PCI/MTQ | Pré-classement automatisé de défauts pour révision ciblée | PaveAudit (visuel) |

ChantierWatch (audience services d'inspection municipaux) est reporté Phase 2 (§12).

**Hors-périmètre explicite** : aucun usage par la municipalité-employeur de l'auteur, ses fournisseurs, ses sous-traitants directs, ni les acteurs avec qui l'auteur est en relation professionnelle dans son emploi principal.

### 3.2 Cas d'usage produit

Scénario qui dicte les exigences fonctionnelles (volume, débit, format de sortie, ergonomie de révision).

**Cas d'usage 1 — Pré-classement d'un audit visuel de chaussée à grande échelle**

> Un inspecteur certifié doit auditer 200 km de réseau. Le véhicule équipé d'un iPhone Repères fait deux journées de captation, le pipeline tourne sur Mac, produit un GeoPackage avec pré-classement des défauts par segment routier. L'inspecteur relit en mode "vérification ciblée" plutôt qu'en inspection à l'aveugle, dans QGIS.

### 3.3 Critères de succès et kill criteria

Distinct des KPIs techniques (§9.1) — ce sont les seuils qui décident de la suite du projet.

**Succès complet** — tous les KPIs §9.1 atteints sur **Montréal**. Décision : poursuivre vers Phase 2+ (§12) — ouverture ChantierWatch, branche inertielle, zone secondaire.

**Succès partiel acceptable** — PaveAudit visuel atteint κ ≥ 0.4 (au lieu de 0.5). Décision : repositionner PaveAudit en outil de pré-priorisation pure, calibration et précision en Phase 2.

**Kill criteria** — conditions qui justifient suspension ou abandon plutôt que continuation :

| Condition | Mesuré à | Action |
|---|---|---|
| **GATE 0 — Faisabilité ML** : Grounding DINO + RDD2022 baseline mAP < 0.4 sur 100-200 frames Quebec annotées vite (classes simples : pothole, fissure longitudinale visible, alligator) | **Fin Phase 0 (spike, ~2 sem)** | **Stop ou pivot avant tout dev iOS / pipeline.** Le bootstrap ML ne marchera pas sur Quebec hivernal sans investissement dataset majeur — repenser scope. C'est le test fail-fast central. |
| **GATE 1 — Licence dataset** : Licence RDD2022 incompatible usage commercial et bascule Stade B' (dataset 100 % maison) hors scope soutenable | Fin Phase 0 | Suspendre. Décider entre rescoping ou abandon. |
| **GATE 2 — κ pipeline** : κ Cohen IA-vs-auteur < 0.4 après premier fine-tune | Fin Phase 4 | **Repositionner** en outil pré-priorisation pure, abandonner prétention IES MTQ. |
| **GATE Validité** : κ_self auteur-vs-auteur (test-retest 10 segments, ≥ 14 j d'écart) < 0.6 | Avant Phase 4 validation finale | **Bloquer la validation**. Le scoring auteur n'est pas assez stable pour servir de vérité de référence — réviser la rubrique écrite, re-scorer, re-tester avant de mesurer le κ pipeline. |
| Volume annoté insuffisant à mi-Phase 3, sans plan d'accélération crédible | Mi-Phase 3 | Bloquer. Recouper avec stagiaire Mitacs ou réduire scope. |
| Effort consommé bien au-delà du scope initial sans avoir atteint GATE 2 | Suivi continu | Stop and reassess. Le side-project a déraillé ; décider arrêt ou refonte. |

**Suivi** : mise à jour de cette section à chaque revue de phase (§10). Aucun kill criterion n'a été déclenché à la date de rédaction.

---

## 4. Hypothèse produit

> Avec un **iPhone récent** (16 Pro ou 17 Pro) monté au pare-brise (caméra 4K + GNSS dans un seul device), des **modèles CV pré-entraînés fine-tunés sur dataset Québec hivernal**, une **boucle de pré-annotation par modèles open-vocabulary** (Grounding DINO / YOLO-World) accélérant le bootstrap, et un **pipeline post-captation Mac**, on peut livrer un **classement IES (Indice d'État Subjectif) automatisé** atteignant **κ Cohen ≥ 0.5** (moderate-to-substantial agreement) vs scoring manuel par l'auteur selon catalogue MTQ, sur réseau urbain de Montréal. Cible calibrée sur la borne humain-humain typique de la littérature (κ humain-humain sur IES MTQ tourne entre 0.5 et 0.7).
>
> La branche inertielle (proxy IRI), la détection de chantiers + cross-check permits, le portage zone secondaire, et la calibration véhicule restent des objectifs Phase 2.

**Précondition critique** : cet objectif n'est atteint qu'**après la Phase 3 d'annotation** (bootstrap dataset Québec via pré-annotation + révision humaine). En sortie de Phase 0 (spike), les modèles pré-entraînés out-of-the-box donneront une baseline ~50-60 % mAP sur classes simples — c'est attendu, c'est le point de référence pour décider si la suite est viable.

**Précondition fail-fast** : la Phase 0 (spike de 2 semaines, captation manuelle + baseline) doit franchir GATE 0 (mAP ≥ 0.4 sur classes simples) **avant** d'engager le développement iOS et pipeline. Si GATE 0 échoue, on stop ou on pivote sans avoir construit l'infrastructure.

Si l'hypothèse est invalidée en aval, l'auteur saura **où** se situe le maillon faible (qualité dataset, fine-tuning, ergonomie annotation, méthodologie de scoring) et pourra investir de manière ciblée.

---

## 5. Périmètre du PoC v1

### 5.1 In-scope

- **Phase 0 — Spike de faisabilité ML** (~2 semaines) : captation manuelle ~1 h Montréal avec iPhone existant et appli native iOS Camera, extraction frames Python, baseline Grounding DINO + YOLO RDD2022 sur 100-200 frames Quebec annotées vite. Mesure mAP sur classes simples (pothole, fissure longitudinale, alligator). **Décision GO/NO-GO sur GATE 0 avant tout dev iOS / pipeline.**
- **App iOS de captation minimale** (Swift/SwiftUI) : vidéo 4K H.265 + GNSS + IMU 100 Hz + heading vrai + manifest minimal (device, calibration ref, lentille `builtInWideAngleCamera` pinned). **Saisie véhicule reportée Phase 2** (auto-fill avec valeur fixe en v1). **Mount-check sophistiqué reporté Phase 2** (alerte simple ±10° pitch). IMU loggé pour usage Phase 2 mais **non exploité v1**. Persistance locale, transfert post-session vers Mac.
- **Pipeline post-captation Mac** (Python 3.12) : ingestion → extraction frames → interpolation pose GNSS → snap-to-road → inférence YOLO custom → agrégation par segment → IES visuel → écriture directe GeoPackage. **Orchestration séquentielle monolithique** : un seul processus, imports directs, aucun service externe, aucun Docker, aucun ProcessPoolExecutor.
- **Module PaveAudit visuel uniquement** : détection visuelle de défauts de chaussée selon catalogue MTQ (8 classes), agrégation par segment routier, calcul **IES visuel**, écriture dans GeoPackage de session.
- **Annotation humaine** : intégration **Label Studio local** (open-source mature) avec import/export GeoPackage. Pas d'UI custom en v1.
- **Boucle active learning** : pré-annotation Grounding DINO → révision Label Studio → export gold labels → fine-tune YOLO → ré-évaluation, avec versioning des poids (manifest YAML manuel + dossier `models/`).
- **Snap-to-road** : code Python via R-tree spatial sur **OSM Québec** (téléchargeable directement geofabrik). Géobase QC officielle reportée Phase 2 (inscription requise, IDs Mtl SIG-compatibles seulement utiles si livraison aux villes).
- **Livraison v1** : GeoPackage exportable, lisible QGIS/ArcGIS/GDAL. **Pas de rapport PDF Quarto en v1.** Pas de service OGC pygeoapi en v1.
- **Validation empirique** sur **Montréal uniquement** : 30+ segments de référence (3-5 km de boucle), captation 3 passages, scoring auteur à l'aveugle selon rubrique MTQ committée Git, mesure κ_self + κ_pipeline.

### 5.2 Out-of-scope v1 (différés Phase 2+, cf. §12)

**Modules différés**
- **ChantierWatch** entier : détection chantiers + cross-check permits OdP + adapter Mtl/Longueuil
- **Branche inertielle PaveAudit** : peak detection IMU, normalisation vitesse, `inertial_score` par segment
- Évaluateur **état signalisation**, **mobilier urbain**, **marquage chaussée**
- **Détection d'accidents et urgences** temps réel

**Infrastructure différée**
- **Pipeline parallèle ProcessPoolExecutor** — séquentiel monolithique en v1
- **Plugin system par entry points Python** — imports directs en v1
- **Service OGC API Features** via pygeoapi — export GeoPackage suffit en mono-utilisateur
- **UI annotation custom** (FastAPI + SvelteKit) — Label Studio en v1
- **Rapport PDF Quarto** — GeoPackage seul en v1
- **Géobase QC officielle** — OSM Québec en v1

**Métrologie différée**
- **IRI calibré ASTM E1926** (le score inertiel Phase 2 sera un proxy non-calibré)
- **Calibration véhicule** pour score inertiel absolu inter-sessions
- **Fusion vision + inertiel** dans un score unique IES
- **Validation κ par inspecteur certifié tiers** (auteur seul en v1)

**Production différée**
- **Suivi temporel inter-frames** (ByteTrack/BoT-SORT)
- **Dashboard web multi-utilisateurs** ou plateforme SaaS
- **API d'écriture**, authentification, contrôle d'accès
- **Floutage automatique** plaques d'immatriculation et visages (à ajouter avant tout partage externe)
- **Comparaison temporelle** multi-passages
- **Mode mobile mapping continu** sur véhicules municipaux
- **Zone secondaire** (Longueuil / Laval / Sherbrooke)

KPIs fonctionnels et protocoles de validation : voir §9.

### 5.3 Exigences non-fonctionnelles (NFR)

Contraintes transverses qui s'appliquent à l'ensemble du système v1.

**Performance**
- Pipeline post-captation Mac : ≤ 3× temps réel sur Mac M-series, hors fine-tune (cf. §9.1).
- Captation iOS : pas de drop frame vidéo, gaps IMU < 50 ms (logged même si non exploité v1), GNSS samples logged au taux natif. Lentille `builtInWideAngleCamera` pinned (fallback automatique multi-lentille désactivé pour préserver la calibration intrinsèque).

**Fiabilité**
- Captation iOS : robuste à mise en veille de l'écran, à appel téléphonique entrant, à passage en arrière-plan momentané (backgrounding tolerance ≥ 30 sec). Throttling thermique : la session continue à 1080p si l'iPhone descend en gamme automatique (à logger dans manifest).
- Pipeline Mac : idempotent — relancer `reperes evaluate` deux fois produit le même résultat. Reprise possible après crash sans perte de progression.
- GeoPackage : transactions atomiques sur écriture multi-layer ; pas de corruption partielle.

**Sécurité et données**
- Stockage local sur disque chiffré FileVault (macOS) et iOS data-protection (iPhone) — vérifié avant toute captation.
- Aucun secret (token HF Hub, clés API) en clair dans le repo Git ; usage de `.envrc` ignoré + 1Password ou keychain.
- Aucun envoi automatique vers cloud en v1 (floutage requis avant tout partage externe — Phase 2).
- Permissions iOS demandées strictement minimales (pas de contacts, pas de Photos library).

**Observabilité**
- Logs structurés (JSON lines) par invocation CLI dans `sessions/<id>/logs/`.
- Métriques par run : durée, frames traitées, détections produites, version modèle, version code (commit Git court).
- `validation_report.md` automatique en fin de `reperes evaluate` avec résumé numérique.
- Erreurs avec localisation (fichier, frame_idx), pas de stack trace cryptique en sortie utilisateur.

**Rétention et cycle de vie des données**
- Sessions vidéo brutes : conservation locale ≤ 90 jours sauf sessions "golden fixture" identifiées explicitement.
- Annotations vérifiées : conservées indéfiniment (font partie du dataset).
- Politique de suppression : commande `reperes session purge <id>` supprime mp4 + jsonl mais conserve annotations + métriques (audit trail).
- Pas de partage de session brute hors poste sans floutage préalable (Phase 2).

**Portabilité**
- Cible exclusive PoC v1 : macOS 14+ sur Apple Silicon (M1+).
- Aucun lock-in cloud ; tout fonctionne hors-ligne.
- iOS : iPhone 16 Pro v1, fallback iPhone 15 Pro testé en backup.

**Conformité**
- Loi 25 (QC) : EFVP/PIA légère à produire avant toute captation hors zones de l'auteur — *à faire en Phase 0*.
- Captation : pas de zone privée intentionnelle (cours intérieures, propriété privée) sans mandat documenté.

---

## 6. Architecture (vue de haut)

Le système se compose de deux éléments en v1 :

1. **App iOS de captation** sur iPhone monté au pare-brise — produit un *bundle de session* (vidéo 4K + GNSS + IMU 100 Hz + heading + manifest minimal : lentille pinned, ref calibration intrinsèque) qui est transféré post-session vers le Mac.
2. **Pipeline post-captation Mac** — une CLI Python (`reperes`) orchestre ingestion, pré-annotation, annotation humaine (Label Studio), évaluation, training, export. Tourne en local sur Apple Silicon (MPS PyTorch). **Aucun service externe, aucun Docker, aucun ProcessPoolExecutor** : snap-to-road en code Python via R-tree spatial sur OSM Québec chargé en mémoire.

Le pipeline d'évaluation v1 est **séquentiel monolithique** dans un seul processus Python :

```
ingest → extract frames → interpolate pose → snap-to-road → infer YOLO →
  → dedupe spatio-temporal → aggregate by segment → compute IES → write GeoPackage
```

Pas de découpage en stages parallèles. Pas de `PreparedStreams`. Pas d'entry points plugins. Le code de PaveAudit vit dans le module Python principal (`reperes/`), pas dans un package séparé. Cette simplicité est un choix v1 explicite — le découpage en stages, la parallélisation, et le système de plugins reviendront en v2 si plus de 2 modules deviennent nécessaires.

Toute la spec technique détaillée (schémas de données, exigences caméra iOS, étapes du pipeline Mac, schémas GeoPackage, stack technique) vit dans `docs/architecture.md`. Pré-requis poste développeur : `docs/dependencies.md`.

---

## 7. Modules

Un seul module livrable v1 : **PaveAudit visuel**. L'annotation humaine en v1 s'appuie sur **Label Studio local** (outil open-source tiers, pas un module Repères — cf. §7.2). ChantierWatch et la branche inertielle de PaveAudit sont reportés Phase 2 (cf. §12). Les KPIs et protocoles de validation sont regroupés au §9.

### 7.1 PaveAudit visuel — état de chaussée

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

PaveAudit consomme les frames extraites + la pose interpolée par frame. Pipeline séquentiel :

1. Extraction de frames à 1-2 fps de la vidéo (ffmpeg).
2. Interpolation pose GNSS+heading par timestamp de frame (lat, lon, speed, heading).
3. Inférence YOLO custom fine-tuné → bbox de défauts par frame.
4. **Déduplication spatio-temporelle** : clustering des détections de même classe dans un rayon spatial < 3 m et fenêtre temporelle < 5 sec → 1 défaut représentatif par cluster (centroide GNSS, max confidence). Remplace un vrai tracker temporel (différé en Phase 2).
5. **Snap-to-road** : chaque détection associée au segment routier le plus proche via R-tree spatial sur OSM Québec chargé en mémoire (test additionnel sur cohérence cap route ↔ cap véhicule pour départager voies parallèles).
6. **Agrégation par segment routier** (typiquement 50-100 m via OSM `way`) :
   - Comptage des défauts par classe et sévérité (légère / modérée / sévère selon barèmes MTQ)
   - Calcul de **densité de défauts** (nb / m²) par classe
   - Conversion en valeur de déduction MTQ par classe selon les barèmes
   - **Score IES visuel 0-100** par segment (`ies_visual`)
7. Écriture des layers GeoPackage :
   - `pavement_defects` : chaque détection visuelle avec `frame_idx`, `bbox`, `class`, `severity`, `confidence`
   - `pavement_segments` : segments routiers avec `ies_visual`, `classification_visual`, `n_defects_by_class`
   - `pavement_metadata` : version modèle, version prompts, conditions de captation

**Livrable v1**

- **GeoPackage** exportable QGIS / ArcGIS / GDAL avec `ies_visual` par segment et défauts individuels géoréférencés.
- **`validation_report.md`** — résumé numérique automatique en fin de `reperes evaluate`.
- **Pas de rapport PDF en v1** (template Quarto reporté Phase 2).
- **Avertissement explicite** dans le `validation_report.md` :
  - **Score visuel uniquement** : aucune mesure inertielle n'est calculée en v1. Le score reflète l'évaluation visuelle MTQ exclusivement.
  - **Validation par auteur seul** : κ mesure l'accord IA-vs-auteur, pas IA-vs-inspecteur certifié. Mise à niveau Phase 2 (validation par inspecteur tiers).

### 7.2 Annotation humaine via Label Studio

**Pourquoi pas une UI custom en v1** — coder un outil web complet (FastAPI + SvelteKit + ergonomie raccourcis clavier + audit trail) est un projet de 4-6 semaines en soi. Label Studio est un outil open-source mature qui couvre 80 % du besoin (révision bbox, ajustement classe/sévérité, raccourcis clavier natifs, audit trail) en zéro lignes de code custom. La décision v1 : utiliser Label Studio local et ne coder une UI custom que si Label Studio se révèle bloquant à l'usage.

**Workflow**

1. Pré-annotation Grounding DINO produit un fichier JSONL au format Label Studio par session.
2. Lancement Label Studio local (`label-studio start`), import du JSONL.
3. Révision humaine : confirmer / rejeter / ajuster bbox / corriger classe / corriger sévérité MTQ. Raccourcis clavier natifs Label Studio.
4. Export gold labels au format YOLO ou COCO (export natif Label Studio).
5. Re-import des gold labels dans le GeoPackage de session via `reperes import-labels` pour traçabilité et ré-entraînement.

**Stockage**

- Annotations vivent dans Label Studio (SQLite local par défaut, peut être pointé vers `sessions/<id>/annotations.sqlite`).
- Gold labels exportés sont importés dans GeoPackage layer `verified_<module>` avec `candidate_id`, `verdict`, `corrected_class`, `corrected_severity`, `annotated_at` pour traçabilité.
- L'audit trail (qui a annoté quoi quand) est natif Label Studio.

**Ergonomie cible** : 30-60 sec par item médian (Label Studio est moins optimisé qu'une UI dédiée mais reste utilisable). Si vitesse < 1500 frames/semaine soutenue après une période de rodage : réduire le nombre de classes annotées (Plan B §10) plutôt que coder une UI custom (différée Phase 2).

**Boucle d'amélioration**

```
Capture session
    ↓
reperes preannotate (Grounding DINO ou YOLO-World)
    ↓
JSONL Label Studio
    ↓
Label Studio local    ← humain vérifie (30-60 sec/item)
    ↓
Export YOLO/COCO (natif Label Studio)
    ↓
reperes import-labels    → layer GeoPackage `verified_<module>`
    ↓
reperes train --module X    → custom YOLO fine-tuné
    ↓
Évaluateurs en production utilisent le custom YOLO entraîné
    ↓
[boucle continue : nouvelles sessions → nouvelles annotations → ré-entraînement périodique]
```

**Décision Phase 0 (D-10)** : spike d'1 jour Label Studio avec une session-fixture (en parallèle du spike ML faisabilité). Si Label Studio est utilisable → poursuivre v1 sans UI custom. Si bloquant (ergonomie inacceptable, format JSONL incompatible) → différer le module en re-scopant ; ne **pas** coder une UI custom en v1 (Phase 2 si vraiment nécessaire).

**Honnêteté sur les limites** — Grounding DINO et YOLO-World ne sont **pas parfaits**. Ils vont :
- Manquer des défauts subtils (salt damage typique du QC peut être imperceptible).
- Sortir des faux positifs sur textures ambiguës (ombres = fissures, etc.).
- Être inconsistants frame-à-frame.

C'est exactement pourquoi l'humain est dans la boucle. La pré-annotation accélère mais ne remplace pas la révision.

---

## 8. Données et stratégie d'entraînement

Datasets de bootstrap et de fine-tuning pour PaveAudit visuel, stratégie phasée d'entraînement, considérations licences. Le schéma technique de livraison (GeoPackage) est dans `docs/architecture.md`.

### 8.1 État des lieux des datasets sources

| Domaine | Source | Volume | Licence | QC-spécifique ? | Utilité v1 |
|---|---|---|---|---|---|
| Défauts chaussée | RDD2022 (IEEE Big Data Cup) | 47 k images, 8 classes | Académique (à vérifier commercial) | ❌ (US, JP, IN, CZ, NO) | Bootstrap principal |
| Défauts chaussée | Crack500, CrackForest, GAP, Pothole-600 | qq centaines à milliers | CC variées | ❌ | Compléments |
| Défauts chaussée | Roboflow Universe community | variable | variable | quelques rares | À trier au cas par cas |
| Référentiel routier | OpenStreetMap (extrait Geofabrik QC) | provincial complet | ODbL | ✅ | Snap-to-road v1 |

Permits OdP, Géobase QC officielle, datasets ChantierWatch : reportés Phase 2.

### 8.2 Stratégie d'entraînement par stade

Trois **stades** d'entraînement qui se chevauchent avec les **phases** projet (§10). Terminologie distincte pour éviter la confusion : *stade* = état du modèle ; *phase* = bloc de planning.

| Stade modèle | Phases projet correspondantes | Activité | Cible quantitative |
|---|---|---|---|
| **Stade A — Baseline pré-entraînée** | Phase 0 (spike) | YOLOv8 RDD2022 + Grounding DINO appliqués out-of-the-box. Mesure baseline sur 100-200 frames QC annotées vite (sans rubrique formelle). | ≥ 40 % mAP sur classes simples (GATE 0) |
| **Stade B — Bootstrap dataset QC** | Phases 2-3 | Captation 3-5 sessions Mtl. Pré-annotation Grounding DINO. Révision Label Studio 2 000-3 000 frames. Fine-tune YOLOv8 custom sur dataset combiné (RDD2022 + QC). | ~70 % mAP PaveAudit |
| **Stade C — Boucle active learning** | Phase 4+ et au-delà | Chaque nouvelle session captée + annotée → ajout au dataset. Ré-entraînement si > 1000 nouvelles frames. Mesure régression vs version précédente. Versioning des poids dans `models/MANIFEST.yaml`. | Amélioration monotone vs Stade B |

**Précondition critique** : passage de Stade A à B nécessite (a) GATE 0 franchi (mAP ≥ 0.4 baseline) et (b) licence RDD2022 confirmée utilisable. Si l'un des deux échoue, basculer en stratégie "Stade B' — dataset 100 % maison" qui rallonge significativement la Phase 3, ou stop fail-fast.

### 8.3 Considérations licences

- **RDD2022** : licence académique, à vérifier précisément en Phase 0 avant utilisation commerciale. Si bloquant : training from scratch sur dataset 100 % maison (Stade B'), à évaluer comme rescoping ou abandon.
- **OpenImages, COCO** : CC-BY, utilisables commercialement avec attribution (réservés Phase 2 ChantierWatch).
- **OSM Québec** : ODbL, utilisable avec attribution et conditions ODbL respectées.
- **Dataset QC propre** : conserver privé en v1 ; décision sur publication HF Hub (CC-BY-NC) à revoir au passage en Stade C.

---

## 9. Validation et engagements qualité

### 9.1 KPIs PoC v1

Cibles que le PoC v1 doit atteindre pour valider l'hypothèse produit (§4). Mesurées selon les protocoles ci-dessous.

| Module | Métrique | Cible |
|---|---|---|
| **GATE 0 — Faisabilité ML** (Phase 0 spike) | mAP baseline RDD2022 + Grounding DINO sur 100-200 frames QC, classes simples | ≥ 0.4 |
| **Prérequis méthodologique** | κ_self test-retest auteur-vs-auteur (10 segments, ≥ 14 j d'écart) | ≥ 0.6 |
| PaveAudit (visuel) | κ_pipeline Cohen IA-vs-auteur sur classification IES MTQ | ≥ 0.5 |
| PaveAudit (visuel) | Ratio κ_pipeline / κ_self (cohérence interne) | ≤ 1.0 |
| PaveAudit (visuel) | mAP par classe de défaut (post fine-tune) | ≥ 0.65 |
| PaveAudit (visuel) | Erreur IES moyenne par segment | ±15 / 100 |
| Annotation (Label Studio) | Vitesse médiane révision post-pré-annotation | ≤ 60 s/item |
| Pipeline global | Temps traitement / heure de vidéo (Mac M-series, séquentiel) | ≤ 3× temps réel |

KPIs inertiels (ρ Spearman, reproductibilité), KPIs ChantierWatch (recall, précision unmatched, F1) : différés Phase 2.

**Note sur les cibles**

- **GATE 0 (mAP ≥ 0.4)** est le test fail-fast central. Mesuré à la fin du spike Phase 0, sur 100-200 frames Quebec annotées vite (annotation rapide, pas selon rubrique MTQ formelle). C'est un seuil bas exprès : si même la baseline rate à ce niveau, le fine-tune ne sauvera pas. Sa mesure prend 1-2 jours de travail.
- **Validation par 30+ segments** (au lieu de 5-10 dans une version antérieure du PRD) : avec n=10 les intervalles de confiance sur le κ sont énormes (~±0.3 typique). 30+ segments donnent une mesure interprétable. Si contrainte de temps en Phase 3-4, prioriser le volume sur la complexité du protocole.
- **κ_self ≥ 0.6 est un prérequis** : si l'auteur n'est pas d'accord avec lui-même à 0.6 entre deux scorings espacés, le κ_pipeline n'a aucune valeur interprétable (cf. §9.2 et kill criterion §3.3). Le ratio κ_pipeline/κ_self ≤ 1.0 est une vérification de cohérence : un pipeline ne peut structurellement pas être plus en accord avec l'auteur que l'auteur ne l'est avec lui-même ; un ratio > 1 trahit une fuite (le scoring auteur a été influencé par les sorties pipeline).
- **κ_pipeline calibré sur la borne humain-humain** de la littérature (κ humain-humain sur IES MTQ tourne typiquement entre 0.5 et 0.7).
- **Pipeline ≤ 3× temps réel** reflète l'orchestration séquentielle v1.

### 9.2 Protocole PaveAudit

**Conditions de validité du κ Cohen** (mitigation du biais d'auto-validation)

La vérité de référence v1 étant produite par l'auteur seul (pas par un ingénieur certifié), le κ mesure l'accord IA-vs-auteur. Pour qu'il ait une **valeur interprétable** et ne soit pas une simple validation circulaire, les contrôles méthodologiques suivants sont des **prérequis** :

- **Rubrique écrite préalable** : grille concrète de conversion défaut → déduction MTQ, rédigée et **committée Git en Phase 0** (avant tout entraînement). Ex. *"≥ 3 fissures longitudinales > 5 m → −20 IES, +1 nid-de-poule visible → −15 IES, +1 faïençage modéré sur > 1 m² → −25 IES"*. Permet la reproductibilité du scoring auteur dans le temps. Non modifiable rétroactivement (toute révision = nouveau scoring complet).
- **Photos de référence MTQ** : 5-10 photos par classe de sévérité du catalogue, consultées pendant chaque scoring. Archivées dans `validation/reference_photos/`.
- **Scoring à l'aveugle pré-pipeline** : l'auteur score manuellement chaque segment **avant** d'exécuter le pipeline d'évaluation. Scoring committé Git (horodaté par commit), aucune modification après visualisation des sorties pipeline.
- **Ordre randomisé** : segments scorés dans un ordre aléatoire (`reperes validate plan-shuffle <session>`), pas dans l'ordre du parcours GPS, pour éviter que la mémoire visuelle d'un segment biaise le suivant.
- **Délai capture-scoring ≥ 7 jours** : minimum entre la captation et le scoring manuel, pour réduire la mémoire visuelle de la session.
- **Test-retest auteur (κ_self)** : 10 segments re-scorés ≥ 14 jours après le premier scoring, sans regarder le précédent. Calcul du **κ auteur-vs-auteur**. **Si κ_self < 0.6, le scoring n'est pas assez stable pour servir de vérité de référence** — réviser la rubrique avant de continuer la validation (cf. kill criterion §3.3).

Le rapport de validation final reporte **trois mesures** : (a) **κ_self** (test-retest auteur, plancher de stabilité), (b) **κ_pipeline** (IA-vs-auteur, KPI principal), (c) **ratio κ_pipeline / κ_self** — un pipeline ne peut structurellement pas être plus en accord avec l'auteur que l'auteur ne l'est avec lui-même ; un ratio > 1 trahit une fuite (le scoring auteur a été influencé par les sorties pipeline).

**Étapes du protocole**

1. Sélectionner **30+ segments de référence** dans la zone test Montréal (5-10 km de boucle), variés en état de chaussée. Le minimum est 30 pour que les intervalles de confiance sur le κ soient interprétables.
2. Captation Repères des mêmes segments en **3 passages** (matin/midi/fin de journée pour variabilité d'éclairage), véhicule unique pour limiter la variabilité de suspension. (Les 3 passages servent en v1 à mesurer la stabilité de la détection visuelle ; en Phase 2 ils serviront aussi à la reproductibilité inertielle.)
3. **Délai capture-scoring ≥ 7 jours.**
4. **Scoring manuel à l'aveugle par l'auteur** selon catalogue MTQ et rubrique préalable : pour chaque segment (ordre randomisé), identification des défauts visibles, sévérité par défaut, conversion en valeur de déduction, IES par segment. Scoring committé Git horodaté.
5. Exécution du pipeline → `ies_visual` par segment + liste des défauts visuels.
6. Comparaison segment-à-segment :
   - **κ_pipeline** : accord IA-vs-auteur sur classification IES
   - Précision/recall par classe de défaut (mAP)
   - Erreur IES moyenne (RMSE)
   - Stabilité visuelle inter-passages (variance `ies_visual` par segment sur 3 passages)
7. **Test-retest κ_self** : 10 segments choisis aléatoirement parmi ceux scorés en étape 4, re-scorés par l'auteur ≥ 14 jours plus tard, sans accès au premier scoring. Calcul de κ_self.
8. Production de `validation_report.md` :
   - Tableau de confusion classification IES
   - Histogramme erreurs IES
   - **Trois κ** (κ_self, κ_pipeline, ratio)
   - Stabilité inter-passages
   - Cas problématiques identifiés (occlusions, distance, conditions)

**Honnêteté méthodologique** : le κ mesure l'accord IA-vs-auteur, pas IA-vs-inspecteur certifié. Cette limite est explicitée dans le `validation_report.md` ; un upgrade vers validation par inspecteur tiers est une étape Phase 2 (cf. §12.1).

---

## 10. Plan phasé

Plan v1 fail-fast en **5 phases**. Durées indicatives pour un side project solo (à réviser après Phase 0 réelle). Les étapes différées Phase 2+ sont en §12.

| Phase | Durée indicative | Livrable | Décision en sortie |
|---|---|---|---|
| **0 — Spike de faisabilité** | ~2 sem | Setup minimal (venv Python, MPS validé), vérif licence RDD2022, calibration intrinsèque iPhone pinned, **rubrique de scoring IES MTQ écrite et committée** (§9.2), captation manuelle ~1 h Mtl avec iPhone existant + appli iOS Camera native, extraction frames Python, baseline Grounding DINO + YOLO RDD2022 sur 100-200 frames Quebec annotées vite. **Mesure mAP baseline.** | **GO/NO-GO GATE 0** : mAP ≥ 0.4 ? Si non → stop ou pivot |
| **1 — App iOS minimum** | ~3 sem | App SwiftUI déployée sur iPhone personnel via free-provisioning : capture 4K H.265 + GNSS + IMU 100 Hz + heading + manifest minimal (device, calibration ref, lentille pinned). Mount-check simple ±10° pitch (alerte non bloquante). Pas de saisie véhicule (auto-fill). Transfert AirDrop/USB. | App fonctionnelle 30 min sans drop |
| **2 — Pipeline Python minimum** | ~4 sem | `reperes ingest`, extraction frames ffmpeg, interpolation pose GNSS+heading par frame (parquet), R-tree OSM QC chargé, snap-to-road, schémas GeoPackage. Pas de détection IMU events (différé Phase 2). | Bundle session → `pavement_segments` vide mais routable |
| **3 — PaveAudit visuel + Label Studio + fine-tune** | ~6 sem | `reperes preannotate` avec Grounding DINO. Captation 3-5 sessions Mtl. Annotation Label Studio sur 2 000-3 000 frames. `reperes import-labels` vers GeoPackage. `reperes train` fine-tune YOLO. Pipeline complet : inférence → dedupe → snap-to-road → agrégation IES par segment → écriture GPKG. | mAP fine-tune ≥ 0.65 sur classes simples |
| **4 — Validation Montréal** | ~2 sem | Captation 3 passages sur 30+ segments de référence (5-10 km). Scoring auteur à l'aveugle selon rubrique §9.2 (ordre randomisé, délai ≥ 7 j). Pipeline → `ies_visual` par segment. Mesure κ_pipeline + mAP par classe + RMSE IES + stabilité inter-passages. **Test-retest κ_self ≥ 14 j plus tard** sur 10 segments. `validation_report.md`. | **GO/NO-GO GATE 2** : κ_pipeline ≥ 0.5 et κ_self ≥ 0.6 ? |

**Total v1 indicatif** : ~17 semaines élapsé optimiste pour un side project, plus probablement 6-8 mois calendaires (variabilité personnelle, météo, dérapage). **Si κ ≥ 0.5 atteint** : ouvrir Phase 2 (ChantierWatch, branche inertielle, zone secondaire, rapport PDF, pygeoapi). **Si κ < 0.4** : stop ou pivot. **Si 0.4 ≤ κ < 0.5** : repositionner en outil pré-priorisation et décider.

**Plan B en cours de v1** : si dérapage Phase 3, livrer Phase 4 avec un nombre réduit de classes (4-5 plutôt que 8) plutôt qu'allonger Phase 3.

---

## 11. Risques et mitigations

### 11.1 Risques techniques

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| **GATE 0 raté** : baseline RDD2022 + Grounding DINO < 0.4 mAP sur classes simples Quebec | Moyenne | Stop fail-fast | C'est précisément le test du spike Phase 0. Coût d'avoir testé : 2 semaines. Pivot possible : dataset 100 % maison (Stade B'), à évaluer comme rescoping. |
| Grounding DINO + fine-tune ratent les défauts hivernaux subtils (frost heave, salt scaling) | Moyenne-haute | Cible mAP par classe non atteinte | Annotation manuelle dédiée à ces classes ; flag explicite dans `validation_report` ; reporter ces classes en Phase 2 si bloquant |
| Validation κ ≥ 0.5 non atteinte malgré fine-tune | Moyenne | Cible qualité non atteinte | Rubrique committée Git avant entraînement ; protocole §9.2 ; repositionner en outil pré-priorisation si κ ≥ 0.4, stop si < 0.4 |
| Side project, scope optimiste | Élevée | Démo retardée | Plan §10 résumé en 5 phases ; plan B explicite (réduire classes plutôt qu'allonger Phase 3) |
| Captation iOS plante : drop frames, gaps IMU, throttling thermique, backgrounding | Moyenne | Sessions inutilisables | Phase 1 dédiée à robustesse captation ; logs détaillés ; rejet automatique en ingest si gaps critiques ; iPhone 15 Pro en backup |
| Volumes de vidéos ingérables | Faible | Sessions limitées | H.265 4K ≈ 6 GB/h ; SSD externe 1-2 TB pour campagne Phase 3-4 |
| Lentille AVFoundation bascule automatiquement (Wide → UltraWide en basse lumière) invalidant la calibration intrinsèque | Faible-moyenne | Détections ratées ou bbox déformées en certaines conditions | Lentille `builtInWideAngleCamera` pinned explicitement, fallback désactivé ; logged dans manifest ; rejet de session en post-traitement si lentille a changé |
| Free-provisioning expire tous les 7 jours en pleine campagne Phase 3-4 | Élevée | Friction redéploiement répétée | Acheter Apple Developer Program 99 $ avant Phase 3 (campagne de captation pour annotation) si redéploiement fréquent |

### 11.2 Risques juridiques et de licences

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| Dataset QC non utilisable hors recherche (licences source) | Faible | Modèle final restreint à un cadre académique | Vérifier licences en Phase 0 ; toute donnée incertaine → exclue ; possibilité re-train from scratch sur dataset 100 % maison Phase 2 |
| Loi 25 — vidéos avec plaques/visages | Moyenne | Risque légal sur la donnée brute | Stockage local SSD chiffré v1 ; floutage automatique avant tout partage externe (Phase 2, modèle ANPR + MTCNN) ; ne pas exporter de vidéos brutes |
| Disclaimer "signal d'audit" insuffisant en cas de litige | Faible | Risque légal | Avis juridique requis avant tout usage du livrable hors cadre interne ; limitation de responsabilité explicite dans chaque rapport |

### 11.3 Risques projet

- **Sous-estimation du volume d'annotation** : à minimiser, c'est facile à allonger. Bloquer la Phase 3 si volume annoté insuffisant à mi-phase et reconsidérer l'embauche d'un stagiaire Mitacs pour bootstrap dataset.
- **Dérive de scope** : ne pas implémenter ChantierWatch, branche inertielle, SignageStateEvaluator, MarkingEvaluator, floutage auto, rapport PDF, ni service OGC avant que PaveAudit visuel v1 ne soit livré et validé. Phase 2 commence après le κ ≥ 0.5, pas avant.
- **Météo défavorable au planning de captation** : prévoir buffer (Phases 3-4) pour repasser en cas de pluie/neige rendant les captures inexploitables.
- **Validation auto-administrée** : la vérité de référence v1 est produite par l'auteur seul. Risque de **biais auto-confirmant** : l'auteur, ayant entraîné le modèle sur ses propres annotations, peut inconsciemment scorer la vérité terrain de manière à confirmer le pipeline. **Mitigations explicites au protocole §9.2** :
  - Rubrique de scoring **écrite et committée Git en Phase 0** (avant tout entraînement du modèle)
  - **Scoring à l'aveugle** pré-pipeline, ordre randomisé, délai capture-scoring ≥ 7 jours
  - **Test-retest κ_self** comme prérequis de validité (kill criterion §3.3 si < 0.6)
  - Ratio κ_pipeline / κ_self ≤ 1.0 vérifié dans chaque rapport (détecte les fuites de scoring auteur ↔ sortie pipeline)
  - Limite explicite documentée dans `validation_report.md`

---

## 12. Évolutions Phase 2+

À considérer **uniquement si le PoC v1 atteint κ_pipeline ≥ 0.5** et que la motivation produit reste. Phase 2 n'a pas de plan détaillé en v1 — c'est volontaire (reportée tant que v1 n'a pas validé son hypothèse centrale).

### 12.1 Modules différés depuis v1 (priorité Phase 2 si v1 réussit)

**Branche inertielle PaveAudit (priorité 1)** — exploitation de l'IMU loggé en v1 (sans nouvelle captation) :
- Filtrage passe-haut accélération verticale, peak detection, normalisation par vitesse, plage exploitable 30-70 km/h
- `inertial_score` par segment, relatif intra-session
- KPIs Phase 2 : ρ Spearman ≥ 0.5 vs `ies_visual` (validation convergente), reproductibilité ≤ 15/100 sur 3 passages, écart entre strates (vitesse/type rue) ≤ 0.3 (contrôle biais commun)
- Schémas GeoPackage : `pavement_imu_events` (POINT), ajout colonnes `inertial_score`, `inertial_severity`, `n_imu_events`, `pct_in_speed_range` à `pavement_segments`

**ChantierWatch (priorité 2)** — détection de chantiers + cross-check permits OdP :
- Classes : cônes, barrières Jersey, panneaux temporaires, véhicules construction, clôtures, déblais
- DBSCAN spatial pour zones, R-tree permits pour cross-check, fenêtre temporelle
- Adapters : `MontrealPermitAdapter`, `LongueuilPermitAdapter`, `GenericNoneAdapter`
- KPIs Phase 2 : recall ≥ 75 %, précision verdict `unmatched` ≥ 80 %, F1 cross-check ≥ 0.7
- Schémas : `construction_detections`, `construction_zones`, `permits_reference`, `unmatched_zones`

**Zone secondaire (priorité 3)** — Longueuil ou Laval (Sherbrooke écartée pour friction trop élevée) :
- Mesure de la dégradation du modèle entraîné sur Mtl appliqué ailleurs
- Si dégradation > 15 % en mAP : ajout de 500-1000 frames de la zone secondaire au dataset, ré-entraînement

### 12.2 Métrologie et validation

**Calibration véhicule pour score inertiel absolu** : transformation du score relatif intra-session en score absolu inter-sessions via un facteur de transfert par véhicule, étalonné contre une vérité IRI mesurée. Pistes :
- **Roadroid** (app iOS commerciale, abonnement annuel) — calibration véhicule semi-rigoureuse
- **Total Pave RC** ou solutions équivalentes
- **Partenariat académique** (Polytechnique, ÉTS, Université Laval) ou **MTQ** : session conjointe avec profileur professionnel
- **Régression empirique** sur segments avec données IRI MTQ ouvertes (faisabilité incertaine)

Sans cet ancrage, le score inertiel reste un signal d'auto-cohérence sans validité métrologique externe.

**Validation κ par inspecteur certifié tiers** : recrutement d'un ingénieur certifié MTQ pour scorer indépendamment 5-10 segments. Mesure de κ pipeline-vs-inspecteur et κ auteur-vs-inspecteur. Si κ auteur-vs-inspecteur < 0.5, la validation v1 doit être réinterprétée.

**Fusion vision + inertiel dans un score unique IES augmenté** : après accumulation de données, calibration empirique de poids `w_visual`, `w_inertiel` ou modèle ML léger (régression). Livré comme `ies_combined`.

### 12.3 Infrastructure

- **Rapport PDF Quarto** par mandat (template + Jinja2)
- **Service OGC API Features** via pygeoapi
- **Pipeline parallèle ProcessPoolExecutor** si > 2 modules simultanés
- **Architecture pluggable par entry points Python** si > 3 modules réels
- **UI annotation custom** uniquement si Label Studio se révèle bloquant
- **Géobase QC officielle** (vs OSM v1) pour cohérence IDs SIG municipaux
- **Suivi temporel inter-frames** (ByteTrack ou BoT-SORT) — vrai tracker au lieu du clustering post-hoc
- **Floutage automatique plaques + visages** (ANPR + MTCNN ou similaire) avant tout partage externe
- **Comparaison temporelle multi-passages**

### 12.4 Nouveaux évaluateurs

- `SignageStateEvaluator` — état des panneaux (penchés, graffités, effacés)
- `MarkingEvaluator` — état du marquage chaussée (lignes effacées, manquantes)
- `StreetFurnitureEvaluator` — mobilier urbain (lampadaires, abribus, bancs, poubelles)

### 12.5 Mode opérationnel

- **Mobile mapping continu** sur véhicules municipaux
- **API d'écriture authentifiée** pour validation par utilisateur identifié
- **Dashboard web** multi-sessions / multi-mandats / multi-clients
- **Auth + permissions**

### 12.6 Ouverture et recherche

- Publication du **dataset Quebec Pavement Winter Damage** sur HuggingFace Hub (CC-BY-NC)
- DATASHEET.md selon Gebru et al. 2018 + croissant.json
- Citation académique (CITATION.cff) si publication scientifique

### 12.7 Cloud et passage à l'échelle

- Pipeline déchargé sur **serveur Linux + GPU NVIDIA** pour mandats volumineux
- **Stockage objet** (B2 / GCS) pour vidéos brutes archivées
- **CI/CD** sur les modèles : versioning des poids (DVC + bucket), métriques de régression

---

## 13. Glossaire

| Terme | Définition |
|---|---|
| **Repères** | Nom du produit (présent document, repo `reperes`, CLI `reperes`) |
| **PaveAudit** | Module d'évaluation d'état de chaussée selon méthodologie MTQ. **v1 = visuel uniquement**. Branche inertielle : Phase 2. |
| **ChantierWatch** | Module de détection de chantiers et cross-check avec permits OdP. **Reporté Phase 2.** |
| **IES** | Indice d'État Subjectif — score visuel 0-100 de l'état de chaussée selon MTQ |
| **`ies_visual`** | Score IES MTQ calculé par la branche visuelle de PaveAudit |
| **`inertial_score`** | Score 0-100 par segment dérivé du RMS d'accélération verticale normalisée par vitesse, relatif intra-session — **Phase 2** |
| **IRI** | International Roughness Index — index inertiel ASTM E1926 mesurable par profileur professionnel |
| **MTQ** | Ministère des Transports du Québec |
| **PCI** | Pavement Condition Index — standard ASTM D6433 (référence internationale, distincte de l'IES MTQ) |
| **OdP** | Occupation du Domaine Public — permis municipal pour chantier sur voie publique |
| **κ Cohen** | Coefficient de Kappa Cohen — mesure d'accord inter-annotateur, 0.6+ = "substantial agreement" |
| **mAP** | Mean Average Precision — métrique standard de détection d'objets |
| **GNSS** | Global Navigation Satellite Systems (GPS, GLONASS, Galileo, BeiDou) |
| **IMU** | Inertial Measurement Unit (accéléromètres + gyroscopes + magnétomètre) — sur iPhone, exposé via CoreMotion. Loggé en v1 sans être exploité (matière première Phase 2). |
| **OGC API Features** | Standard Open Geospatial Consortium pour la publication de features géographiques en REST/JSON — **Phase 2** |
| **NAD83 MTM zone 8** | Système de coordonnées projeté pour le sud du Québec, EPSG:32188 |
| **Grounding DINO** | Modèle CV open-vocabulary par IDEA-Research, détection bbox via prompts texte |
| **YOLO-World** | Modèle CV open-vocabulary par Tencent, basé YOLOv8, détection via prompts texte |
| **DBSCAN** | Density-Based Spatial Clustering — algo de clustering non-paramétrique (utilisé Phase 2 par ChantierWatch) |
| **R-tree** | Structure d'index spatial pour requêtes nearest-neighbor et intersection rapides en mémoire |
| **OSM** | OpenStreetMap — référentiel routier ouvert, utilisé en v1 pour snap-to-road |
| **Géobase** | Référentiel routier provincial du Québec, données ouvertes — **Phase 2** (cohérence IDs SIG municipaux) |
| **MPS** | Metal Performance Shaders, accélérateur Apple Silicon pour PyTorch |
| **Active learning loop** | Boucle d'amélioration : prédiction → correction humaine → ré-entraînement |
| **Pré-annotation** | Génération automatique de bbox candidats par modèle open-vocab, à réviser par humain |
| **Free provisioning** | Déploiement Xcode avec Apple ID gratuit (vs Apple Developer Program payant) |
| **Loi 25** | Loi modernisant la protection des renseignements personnels (Québec, en vigueur 2024) |

---

## 14. Décisions ouvertes

Format : `[ID] Décision — Owner — Échéance — Statut — Bloquant si non résolu en`. L'auteur étant unique contributeur en v1, `Owner = Julien` partout ; le champ est conservé pour formalité et pour usage futur si un partenaire rejoint le projet. Les échéances sont relatives aux phases du plan §10.

| ID | Décision | Owner | Échéance | Statut | Bloquant si non résolu en |
|---|---|---|---|---|---|
| D-01 | Confirmer disponibilité iPhone personnel pour le projet (jamais de l'employeur) | Julien | Avant Phase 1 | À faire | Phase 1 |
| D-02 | Vérifier licence exacte RDD2022 (académique stricte ? CC-BY ? CC-BY-NC ?) — lecture terms of use + email auteurs si ambigu | Julien | Phase 0 | À faire | Phase 3 (Stade B) |
| D-03 | EFVP/PIA Loi 25 légère à produire avant captation hors zones auteur | Julien | Avant Phase 1 captation Mtl | À faire | Phase 1 captation Mtl |
| D-04 | Vérifier chiffrement FileVault actif sur Mac + data-protection iOS sur iPhone | Julien | Phase 0 | À faire | Phase 1 |
| D-05 | Décision Stade B vs Stade B' selon résultat D-02 | Julien | Fin Phase 0 | Bloquée par D-02 | Phase 3 |
| D-06 | Achat Apple Developer Program 99 $/an si free-provisioning devient bloquant pendant campagne | Julien | Avant Phase 3 | À évaluer | Phase 3 (captation pour annotation) |

Décisions différées Phase 2 (n'apparaissent ici que si v1 atteint κ ≥ 0.5) : choix permits OdP par ville, Géobase QC officielle vs OSM, calibration véhicule, partenariat académique, etc.

Statuts possibles : `À faire` · `En cours` · `Résolu (lien commit ou doc)` · `Bloquée par <ID>` · `Reportée Phase 2+`.

**Décisions stratégie d'affaire** (hors PRD, traitées dans `docs/marketing/`) : nom commercial, incorporation, tarification, choix d'audience pour les premières démos, règles internes d'activités accessoires de l'employeur.

---

*Fin du document. 2026-04-29.*
