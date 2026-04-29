# PRD — Repères · Plateforme d'audit automatisé d'actifs viaires municipaux

**Date** : 2026-04-29
**Statut** : draft, en attente de validation produit
**Auteur** : Julien Riel, avec assistance Claude
**Cible PoC** : Deux modules livrables — PaveAudit (état de chaussée selon méthodo MTQ, vision + inertiel) et ChantierWatch (détection de chantiers + cross-check permits OdP), validés empiriquement à Montréal, Longueuil et Sherbrooke

> Ce document est la **seule source de vérité produit** pour Repères. Il remplace toute spec ou plan antérieur. Les spécifications d'implémentation (architecture, code, séquence des tâches) en découlent.

---

## 1. Vision

Construire une **plateforme d'audit automatisé d'actifs viaires municipaux** par captation vidéo géoréférencée et **inertielle** depuis un véhicule, avec une **architecture pluggable d'évaluateurs spécialisés**.

Le PoC v1 livre **deux modules** :
- **PaveAudit** — état de chaussée selon la méthodologie d'évaluation visuelle MTQ. Un **démonstrateur inertiel expérimental** (proxy IRI à partir des accéléromètres iPhone) est construit en parallèle mais **hors KPI v1** : non-calibré véhicule par véhicule, livré comme PoC technique séparé pour calibration sérieuse en Phase 2.
- **ChantierWatch** — détection de chantiers et cross-référencement avec les bases de permits d'occupation du domaine public (OdP) en open data municipal

L'**annotation humaine** des sorties s'appuie en v1 sur **Label Studio local** (outil open-source mature), pas sur une UI custom — décision de scope pour livrer plus vite. Une UI custom n'est codée en v1.5+ que si Label Studio se révèle bloquant.

L'architecture est modulaire (chaque évaluateur a son propre protocole de validation, ses propres KPIs, et son propre livrable) mais **monolithique en v1** : orchestration séquentielle, imports directs. Un système de plugins (entry points Python) sera ajouté en v2 si > 3 modules réels deviennent nécessaires.

L'objectif est de **prouver la faisabilité technique** des deux modules sur **deux zones tests** représentatives (Montréal + 1 zone de portage proche), avec un matériel grand public et une chaîne logicielle 100 % open source, **100 % locale, sans aucune dépendance à un service externe**.

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
- Ce n'est **pas** une mesure d'IRI (International Roughness Index) certifiée ASTM E1926. PaveAudit produit l'**indice visuel MTQ** ET un **score inertiel proxy** corrélé au confort de roulement, mais ce dernier n'est pas calibré véhicule par véhicule. Il est livré comme **signal complémentaire** au score visuel, pas comme substitut à un profileur professionnel.

---

## 3. Audiences et cas d'usage

### 3.1 Audiences produit

Profils d'utilisateurs visés par les fonctionnalités du PoC v1. Cadre les exigences fonctionnelles (formats de livrable, ergonomie, terminologie) — pas la stratégie d'aller-au-marché, qui est traitée dans `docs/marketing/`.

| Audience | Cas d'usage produit | Module pertinent |
|---|---|---|
| Inspecteurs certifiés réalisant des audits PCI/MTQ | Pré-classement automatisé de défauts pour révision ciblée | PaveAudit |
| Services d'inspection municipaux | Détection de chantiers + cross-check permits OdP | ChantierWatch |

**Hors-périmètre explicite** : aucun usage par la municipalité-employeur de l'auteur, ses fournisseurs, ses sous-traitants directs, ni les acteurs avec qui l'auteur est en relation professionnelle dans son emploi principal.

### 3.2 Cas d'usage produit

Scénarios qui dictent les exigences fonctionnelles (volume, débit, format de sortie, ergonomie de révision).

**Cas d'usage 1 — Pré-classement d'un audit visuel de chaussée à grande échelle**

> Un inspecteur certifié doit auditer 200 km de réseau. Le véhicule équipé d'un iPhone Repères fait deux journées de captation, le pipeline tourne sur Mac, produit un GeoPackage + rapport PDF avec pré-classement des défauts. L'inspecteur relit en mode "vérification ciblée" plutôt qu'en inspection à l'aveugle.

**Cas d'usage 2 — Détection de chantiers vs permits sur tour municipal régulier**

> Un service d'inspection municipal fait un tour de 30 km mensuel avec Repères. Le système croise les zones de chantier détectées avec les permits OdP en open data, produit une liste de zones sans correspondance permit. L'inspection terrain est ciblée plutôt qu'à l'aveugle.

### 3.3 Critères de succès et kill criteria

Distinct des KPIs techniques par module (§9.1) — ce sont les seuils qui décident de la suite du projet.

**Succès complet** — tous les KPIs §9.1 atteints sur **les 2 zones tests** (Mtl + 1 zone de portage proche), avec rapport de validation signé par l'ingénieur civil de référence. Décision : poursuivre vers Phase 2+ (§12).

**Succès partiel acceptable** — PaveAudit visuel atteint κ ≥ 0.4 (au lieu de 0.5) **OU** ChantierWatch atteint recall ≥ 60 % (au lieu de 75 %) **OU** portage zone secondaire dégradé > 15 % mAP. Décision : repositionner PaveAudit en outil de pré-priorisation, ChantierWatch en démonstrateur, calibration et précision en Phase 2.

**Kill criteria** — conditions qui justifient suspension ou abandon plutôt que continuation :

| Condition | Mesuré à | Action |
|---|---|---|
| **GATE 1** : Grounding DINO + RDD2022 baseline mAP < 0.4 sur classes simples (cône, fissure visible) | T0 + 8 sem (fin Phase 1) | **Stop ou pivot**. Le bootstrap ML ne marchera pas sur Quebec hivernal sans investissement dataset majeur — repenser scope avant d'avoir bâti le pipeline complet. |
| **GATE 2** : κ Cohen IA-humain < 0.4 après premier fine-tune | T0 + 16-20 sem (fin Phase 4) | **Repositionner** en outil pré-priorisation pure, abandonner prétention IES MTQ. |
| Licence RDD2022 incompatible et coût Stade B' (dataset 100 % maison) > 50 h supplémentaires | Fin Phase 0 | Suspendre. Décider entre rescoping ou abandon. |
| Indisponibilité de l'ingénieur civil de référence après 3 mois de démarchage | Fin Phase 3 | Suspendre validation. Pivoter vers auto-annotation rigoureuse + revue par pair non-certifié. |
| Volume annoté < 2000 frames à mi-Phase 4, sans plan d'accélération crédible | Mi-Phase 4 | Bloquer. Recouper avec stagiaire Mitacs ou réduire scope à 1 module. |
| Dérive cumulée d'effort > 50 % vs estimé (i.e. > 600 h consommées sans avoir atteint GATE 2) | Suivi continu | Stop and reassess. Le side-project a déraillé ; décider arrêt ou refonte. |

**Suivi** : mise à jour de cette section à chaque revue de phase (§10). Aucun kill criterion n'a été déclenché à la date de rédaction.

---

## 4. Hypothèse produit

> Avec un **iPhone récent** (16 Pro ou 17 Pro) monté au pare-brise (caméra 4K + GNSS + IMU dans un seul device), des **modèles CV pré-entraînés fine-tunés sur dataset Québec hivernal**, une **boucle de pré-annotation par modèles open-vocabulary** (Grounding DINO / YOLO-World) accélérant le bootstrap, et un **pipeline post-captation Mac**, on peut livrer :
>
> - Un **classement IES (Indice d'État Subjectif) automatisé** atteignant **κ Cohen ≥ 0.5** (moderate-to-substantial agreement) avec un ingénieur civil de référence sur classes MTQ, sur réseau urbain et tronçon municipal. Cible calibrée sur la borne humain-humain typique de la littérature (κ humain-humain sur IES MTQ tourne entre 0.5 et 0.7).
> - Une **détection de chantiers** à **≥ 75 % de recall** avec cross-référencement automatique aux bases open data de permits OdP, isolant les détections sans correspondance avec **≥ 80 % de précision sur le verdict `unmatched`**.
>
> Un **démonstrateur inertiel** est construit en parallèle (proxy IRI à partir de l'IMU iPhone) mais **n'est pas inclus dans les KPIs v1** : sa calibration véhicule par véhicule et sa validation contre une vérité IRI mesurée sont des objectifs Phase 2.

**Précondition critique** : ces objectifs ne sont atteints qu'**après la Phase 1 d'annotation** (bootstrap dataset Québec via pré-annotation + révision humaine). En sortie de Phase 0, les modèles pré-entraînés out-of-the-box donneront une baseline ~60 % mAP — c'est attendu, c'est le point de départ.

Si l'hypothèse est invalidée, l'auteur saura **où** se situe le maillon faible (qualité dataset, fine-tuning, couplage permits, ergonomie annotation) et pourra investir de manière ciblée.

---

## 5. Périmètre du PoC v1

### 5.1 In-scope

- **App iOS de captation tout-en-un** (Swift/SwiftUI) : vidéo 4K H.265 + GNSS + IMU 100 Hz (accéléro + gyro + magnéto) + heading vrai + métadonnées de mount (lentille AVFoundation pinned, pitch/roll moyen) + métadonnées véhicule (marque/modèle/année saisis avant session), persistance locale, transfert post-session vers Mac.
- **Pipeline post-captation Mac** (Python 3.12) : ingestion, pré-annotation, évaluation, livraison. **Orchestration séquentielle monolithique en v1** : préparation commune (frames, pose interpolée par frame), puis itération sur les évaluateurs, puis consolidation GeoPackage. Aucun service externe, aucun Docker, snap-to-road en code Python via R-tree spatial sur référentiel routier en mémoire. Parallélisation `ProcessPoolExecutor` différée v1.5 (gain négligeable à 2 modules, complexité élevée).
- **Module PaveAudit** : détection visuelle de défauts de chaussée selon catalogue MTQ (8 classes), agrégation par segment routier, calcul **IES visuel**, livrable cartographique. Branche inertielle **construite séparément en démonstrateur expérimental hors KPI** (cf. §7.1).
- **Module ChantierWatch** : détection visuelle de zones de chantier (cônes, barrières, signalisation temporaire, véhicules construction, clôtures privées, déblais), cross-référencement avec permits OdP, flag des `unmatched`. Vision-only en v1.
- **Annotation humaine** : intégration **Label Studio local** (open-source mature) avec import/export GeoPackage. Pas d'UI custom en v1 (différée si Label Studio bloquant).
- **Boucle active learning** : pré-annotation Grounding DINO/YOLO-World → révision Label Studio → export gold labels → fine-tune → ré-évaluation, avec versioning des poids (manifest manuel + dossier).
- **Livraison** : GeoPackage exportable directement (lisible QGIS/ArcGIS/GDAL) + rapport PDF par session (template Quarto). Service OGC API Features via pygeoapi **différé v1.5**.
- **Validation empirique** sur **2 zones** : Montréal (richesse de données + debug) + 1 zone de portage proche (Longueuil ou Laval, à décider en Phase 0). Sherbrooke différée Phase 2 (250 km aller-retour, friction trop élevée pour PoC).

### 5.2 Out-of-scope v1 (architecture les supportera, mais pas implémentés)

- Évaluateur **état signalisation** (panneaux penchés, graffités, effacés)
- Évaluateur **mobilier urbain** (état des lampadaires, abribus, bancs, poubelles)
- Évaluateur **marquage chaussée** (lignes effacées, manquantes)
- **Détection d'accidents et urgences** en temps réel
- **IRI calibré ASTM E1926** (le score inertiel v1 est un proxy non-calibré véhicule par véhicule)
- **Calibration véhicule** pour score inertiel absolu inter-sessions (v1 = score relatif intra-session uniquement)
- **Fusion vision + inertiel** dans un score unique IES (v1 expose les deux scores en parallèle, fusion en Phase 2 après accumulation de données)
- **Suivi temporel inter-frames** (ByteTrack/BoT-SORT) — v1 utilise déduplication spatio-temporelle post-hoc par module
- **Dashboard web multi-utilisateurs** ou plateforme SaaS
- **API d'écriture** (le service OGC est read-only)
- **Authentification** / contrôle d'accès du service OGC
- **Floutage automatique** plaques d'immatriculation et visages (à ajouter avant tout partage externe — Phase 2)
- **Comparaison temporelle** multi-passages
- **Mode mobile mapping continu** sur véhicules municipaux
- **Pipeline parallèle Stage 2** (`ProcessPoolExecutor`) — différé v1.5 si gains observables avec > 2 modules
- **Plugin system par entry points Python** — différé v2 si > 3 modules réels deviennent nécessaires
- **Service OGC API Features** via pygeoapi — différé v1.5 (export GeoPackage suffit en mono-utilisateur)
- **UI annotation custom** (FastAPI + SvelteKit) — différée si Label Studio bloquant pour v1
- **Inertiel en livrable client** — démonstrateur seulement en v1, intégration v1.5 après calibration véhicule

KPIs fonctionnels et protocoles de validation : voir §9.

### 5.3 Exigences non-fonctionnelles (NFR)

Contraintes transverses qui s'appliquent à l'ensemble du système, par-delà chaque module.

**Performance**
- Pipeline post-captation Mac : ≤ 2× temps réel sur Mac M-series (i.e. 1 h de vidéo traitée en ≤ 2 h, hors fine-tune). Cible facilitée par le pipeline parallèle (Stage 2 exécute les évaluateurs concurremment via `ProcessPoolExecutor`).
- Captation iOS : pas de drop frame vidéo, gaps IMU < 50 ms, GNSS samples logged au taux natif. Lentille `builtInWideAngleCamera` pinned (fallback automatique multi-lentille désactivé pour préserver la calibration intrinsèque).
- UI annotation : interaction click-to-next < 200 ms, navigation entre items fluide à 60 fps.

**Conditions d'exploitation du signal inertiel**
- Plage de vitesse exploitable pour l'inertiel : **30-70 km/h**. En dessous, signal trop faible (suspension absorbe). Au-dessus, signal saturé et corrélation au défaut routier brouillée par le bruit aérodynamique. Les frames hors plage sont conservées pour la vision mais exclues du calcul inertiel par segment.
- Score inertiel **relatif intra-session** uniquement en v1 (pas de calibration véhicule, pas de comparabilité absolue inter-sessions). Marque/modèle/année du véhicule logués dans le manifest pour traçabilité et calibration future Phase 2.
- Mount-check pré-session : pitch caméra ±10° de l'horizontale, sinon alerte. Pitch/roll moyen de la session enregistré dans le manifest pour filtrage/repondération en post-traitement.

**Fiabilité**
- Captation iOS : robuste à mise en veille de l'écran, à appel téléphonique entrant, à passage en arrière-plan momentané (backgrounding tolerance ≥ 30 sec).
- Pipeline Mac : idempotent par évaluateur — relancer `reperes evaluate` deux fois produit le même résultat. Reprise possible après crash sans perte de progression.
- GeoPackage : transactions atomiques sur écriture multi-layer ; pas de corruption partielle.

**Sécurité et données**
- Stockage local sur disque chiffré FileVault (macOS) et iOS data-protection (iPhone) — vérifié avant toute captation.
- Aucun secret (token HF Hub, clés API) en clair dans le repo Git ; usage de `.envrc` ignoré + 1Password ou keychain.
- Aucun envoi automatique vers cloud avant Phase 2 (floutage requis avant tout partage externe).
- Permissions iOS demandées strictement minimales (pas de contacts, pas de Photos library).

**Observabilité**
- Logs structurés (JSON lines) par invocation CLI dans `sessions/<id>/logs/`.
- Métriques par run d'évaluateur : durée, frames traitées, détections produites, version modèle, version code (commit Git court).
- `validation_report.md` automatique en fin de `reperes evaluate` avec résumé numérique.
- Erreurs avec localisation (fichier, frame_idx ou ligne CSV concernée), pas de stack trace cryptique en sortie utilisateur.

**Rétention et cycle de vie des données**
- Sessions vidéo brutes : conservation locale ≤ 90 jours sauf sessions "golden fixture" identifiées explicitement.
- Annotations vérifiées : conservées indéfiniment (font partie du dataset).
- Politique de suppression : commande `reperes session purge <id>` supprime mp4 + jsonl mais conserve annotations + métriques (audit trail).
- Pas de partage de session brute hors poste sans floutage préalable (Phase 2).

**Portabilité**
- Cible exclusive PoC v1 : macOS 14+ sur Apple Silicon (M1+).
- Aucun lock-in cloud ; tout fonctionne hors-ligne (aucun service externe à configurer).
- iOS : iPhone 16 Pro v1, fallback iPhone 15 Pro testé en backup.

**Conformité**
- Loi 25 (QC) : EFVP/PIA légère à produire avant toute captation hors zones de l'auteur — *à faire en Phase 0*.
- Captation : pas de zone privée intentionnelle (cours intérieures, propriété privée) sans mandat documenté.

---

## 6. Architecture (vue de haut)

Le système se compose de trois éléments :

1. **App iOS de captation** sur iPhone monté au pare-brise — produit un *bundle de session* (vidéo 4K + GNSS + IMU 100 Hz + heading + manifest enrichi : lentille pinned, pitch/roll mount, véhicule) qui est transféré post-session vers le Mac.
2. **Pipeline post-captation Mac** — une CLI Python (`reperes`) orchestre ingestion, pré-annotation, annotation humaine, évaluation, training, livraison. Tourne en local sur Apple Silicon (MPS PyTorch). **Aucun service externe, aucun Docker** : snap-to-road en code Python via R-tree spatial sur référentiel routier en mémoire (Géobase QC ou OSM extrait).
3. **Pipeline d'évaluation à trois étages** :
   - **Stage 1 (commun, séquentiel, fait une fois)** : extraction des frames vidéo, interpolation pose GNSS+IMU par frame, détection des événements inertiels normalisés par vitesse → produit un objet `PreparedStreams` consommé par tous les modules.
   - **Stage 2 (modules en parallèle)** : chaque évaluateur tourne dans son propre sous-processus via `ProcessPoolExecutor`, consomme `PreparedStreams`, fait son inférence ML + sa logique métier, écrit ses layers dans le GeoPackage de session.
   - **Stage 3 (consolidation, séquentiel)** : finalisation du GeoPackage, génération du `validation_report.md`, logs structurés.
4. **Architecture pluggable d'évaluateurs** — chaque module (PaveAudit, ChantierWatch, Annotation) est un plugin Python découvert via entry points. Aucun couplage entre évaluateurs ; ajouter un nouveau module ne demande aucune modification du core. Les services partagés (snap-to-road, accès Géobase, logger contextuel) vivent dans `reperes_core`.

Toute la spec technique détaillée (schémas de données, signatures d'API plugin, exigences caméra/IMU iOS, étapes du pipeline Mac, configuration pygeoapi, stack technique complet) vit dans `docs/architecture.md`. Pré-requis poste développeur : `docs/dependencies.md`.

---

## 7. Modules

Deux modules livrables (PaveAudit visuel, ChantierWatch). L'annotation humaine en v1 s'appuie sur **Label Studio local** (outil open-source tiers, pas un module Repères — cf. §7.3). L'architecture est modulaire mais **monolithique en v1** (cf. `docs/architecture.md`) ; un système de plugins par entry points Python sera ajouté en v2 si > 3 modules réels deviennent nécessaires. Les KPIs et protocoles de validation propres à chaque module sont regroupés au §9.

### 7.1 PaveAudit — état de chaussée

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

PaveAudit consomme les artefacts préparés (frames + pose interpolée). Le pipeline v1 se concentre sur la **branche visuelle (méthodologie MTQ)**. Une branche inertielle expérimentale est construite en parallèle hors du flux de scoring KPI (cf. encadré ci-bas).

*Branche visuelle (méthodologie MTQ)*

1. Extraction de frames à 1-2 fps de la vidéo (déjà faite en Stage 1, consommée ici).
2. Inférence YOLO custom fine-tuné → bbox de défauts par frame.
3. **Déduplication spatio-temporelle** : clustering des détections de même classe dans un rayon spatial < 3 m et fenêtre temporelle < 5 sec → 1 défaut représentatif par cluster (centroide GNSS, max confidence). Remplace un vrai tracker temporel (différé en Phase 2).
4. **Snap-to-road** : chaque détection associée au segment routier le plus proche via R-tree spatial sur Géobase QC chargée en mémoire (test additionnel sur cohérence cap route ↔ cap véhicule pour départager voies parallèles).
5. **Agrégation par segment routier** (typiquement 50-100 m via Géobase) :
   - Comptage des défauts par classe et sévérité (légère / modérée / sévère selon barèmes MTQ)
   - Calcul de **densité de défauts** (nb / m²) par classe
   - Conversion en valeur de déduction MTQ par classe selon les barèmes
   - **Score IES visuel 0-100** par segment (`ies_visual`)

*Démonstrateur inertiel — hors KPI v1*

> Construit séparément en Phase 4 (≤ 1 semaine d'effort). **Non livré au client en v1**, non inclus dans les KPIs §9.1. Objectif : valider qualitativement que le signal IMU iPhone capte bien quelque chose sur la chaussée, sans prétention de score absolu ni de calibration véhicule.

1. Détection des pics d'accélération verticale (filtrage passe-haut + peak detection) à partir de `imu.jsonl`.
2. Filtrage plage de vitesse exploitable (30-70 km/h GNSS), normalisation amplitude par vitesse.
3. Snap-to-road via R-tree, comptage événements par segment.
4. Visualisation côte-à-côte avec `ies_visual` sur 5-10 segments choisis aux GATE 2 pour inspection qualitative (notebook ou export QGIS).

Aucun `inertial_score` n'est exposé dans le GeoPackage livrable v1. Si le démonstrateur révèle un signal qualitatif intéressant, la calibration véhicule et l'intégration en livrable sont l'objet de Phase 2 (§12.1).

*Sortie consolidée*

5. Output GeoPackage layers :
   - `pavement_defects` : chaque détection visuelle avec `frame_idx`, `bbox`, `class`, `severity`, `confidence`
   - `pavement_segments` : segments routiers avec `ies_visual`, `classification_visual`, `n_defects_by_class`
   - `pavement_metadata` : version modèle, version prompts, conditions de captation, véhicule

(Le démonstrateur inertiel produit des artefacts séparés en `prepared/imu_demo/`, hors livrable principal.)

**Livrable**

- Rapport PDF par mandat : carte du réseau colorée par `ies_visual`, top 20 segments les plus dégradés visuellement, liste des défauts en annexe, tableau récapitulatif.
- GeoPackage exportable QGIS / ArcGIS / GDAL avec `ies_visual` sur chaque segment.
- Démonstrateur inertiel : visualisation séparée (notebook ou capture d'écran QGIS) sur 5-10 segments d'inspection qualitative, **non livré au client**.

### 7.2 ChantierWatch — détection de chantiers vs permits

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

ChantierWatch consomme `PreparedStreams` (issu de Stage 1 commun). Vision-only en v1 (l'IMU est disponible mais non exploité par ce module).

1. Détection frame-par-frame YOLO custom fine-tuné (ou YOLO-World direct si fine-tune pas prêt).
2. **Déduplication spatio-temporelle** par classe (rayon < 3 m, fenêtre < 5 sec) — un cône vu sur 8 frames consécutives = 1 cône, pas 8.
3. **Clustering spatial** : DBSCAN sur coordonnées géographiques + temporelles → "zones de chantier" (polygones englobants).
4. Génération bbox géographique de chaque zone (POLYGON GeoPackage).
5. Snap-to-road de la zone via R-tree (segment englobant principal).
6. **Cross-check permits** :
   - Charger cache local des permits OdP de la ville de captation (téléchargé pré-session).
   - Pour chaque zone : requête R-tree → permits intersectant la géométrie.
   - Filtrer par fenêtre temporelle : `permit.date_debut ≤ session_date ≤ permit.date_fin`.
   - **Verdict** :
     - `permitted` : ≥ 1 permit actif intersectant
     - `unmatched` : aucun permit intersectant trouvé
     - `permit_expired` : permit existe mais hors fenêtre temporelle
     - `data_unavailable` : ville sans données ouvertes de permits → flag manuel
7. Output GeoPackage layers :
   - `construction_detections` : détections individuelles dédupliquées
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

### 7.3 Annotation humaine via Label Studio

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

**Ergonomie cible** : 30-60 sec par item médian (Label Studio est moins optimisé qu'une UI dédiée mais reste utilisable). Si vitesse < 1500 frames/semaine soutenue après 2 semaines d'usage : revoir vers UI custom en v1.5 (différer pygeoapi en compensation).

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

**Décision Phase 0 (D-10)** : spike d'1 jour Label Studio avec une session-fixture. Si Label Studio est utilisable → poursuivre v1 sans UI custom. Si bloquant (ergonomie inacceptable, format JSONL incompatible avec workflow GeoPackage) → coder UI minimale (différer pygeoapi en compensation).

**Honnêteté sur les limites** — Grounding DINO et YOLO-World ne sont **pas parfaits**. Ils vont :
- Manquer des défauts subtils (salt damage typique du QC peut être imperceptible).
- Sortir des faux positifs sur textures ambiguës (ombres = fissures, etc.).
- Être inconsistants frame-à-frame.

C'est exactement pourquoi l'humain est dans la boucle. La pré-annotation accélère mais ne remplace pas la révision.

---

## 8. Données et stratégie d'entraînement

Datasets de bootstrap et de fine-tuning, stratégie phasée d'entraînement, considérations licences. Le schéma technique de livraison (GeoPackage, pygeoapi, endpoints OGC) est dans `docs/architecture.md`.

### 8.1 État des lieux des datasets sources

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

### 8.2 Stratégie d'entraînement par stade

Trois **stades** d'entraînement qui se chevauchent avec les **phases** projet (§10). Terminologie distincte pour éviter la confusion : *stade* = état du modèle ; *phase* = bloc de planning.

| Stade modèle | Phases projet correspondantes | Activité | Cible quantitative |
|---|---|---|---|
| **Stade A — Baseline pré-entraînée** | §10 Phase 0 + début Phase 3 | YOLOv8 COCO (chantiers) et YOLOv8 RDD2022 (chaussée) appliqués out-of-the-box. Mesure baseline sur 200-300 frames QC captées. | ~60 % mAP, attendu et bas |
| **Stade B — Bootstrap dataset QC** | §10 Phases 3 → 5 | Captation 5-10 sessions Mtl/Longueuil. Pré-annotation Grounding DINO/YOLO-World. Révision humaine 5 000-10 000 frames. Fine-tune YOLOv8 custom sur dataset combiné (RDD2022 + QC). | ~75 % mAP PaveAudit, 85 % recall ChantierWatch |
| **Stade C — Boucle active learning** | §10 Phase 7+ et au-delà | Chaque nouvelle session captée + annotée → ajout au dataset. Ré-entraînement périodique (tous les 3-5 k frames). Mesure régression vs version précédente. Versioning des poids dans `models/MANIFEST.yaml`. | Amélioration monotone vs Stade B |

**Précondition critique** : passage de Stade A à B nécessite que la licence RDD2022 soit confirmée utilisable (cf. §8.3 et §14). Si bloquant en Phase 0, basculer en stratégie "Stade B' — dataset 100 % maison" qui rallonge significativement la Phase 4.

### 8.3 Considérations licences

- **RDD2022** : licence académique, à vérifier précisément en §10 Phase 0 avant utilisation commerciale. Si bloquant : training from scratch sur dataset 100 % maison (cf. Stade B' ci-dessus).
- **COCO, OpenImages** : CC-BY, utilisables commercialement avec attribution.
- **Permits OdP Montréal, Géobase, Données Québec** : CC-BY, utilisables sans restriction commerciale avec attribution.
- **Dataset QC propre** : conserver privé en v1 ; décision sur publication HF Hub (CC-BY-NC) à revoir au passage en Stade C.

---

## 9. Validation et engagements qualité

### 9.1 KPIs PoC v1

Cibles que le PoC v1 doit atteindre pour valider l'hypothèse produit (§4). Mesurées selon les protocoles ci-dessous.

| Module | Métrique | Cible |
|---|---|---|
| PaveAudit (visuel) | κ Cohen sur classification IES MTQ vs ingénieur référence | ≥ 0.5 |
| PaveAudit (visuel) | mAP par classe de défaut | ≥ 0.65 |
| PaveAudit (visuel) | Erreur IES moyenne par segment | ±15 / 100 |
| ChantierWatch | Recall détection zones de chantier | ≥ 75 % |
| ChantierWatch | Précision verdict `unmatched` | ≥ 80 % |
| ChantierWatch | F1 cross-check temporel permits | ≥ 0.7 |
| Annotation (Label Studio) | Vitesse médiane révision post-pré-annotation | ≤ 60 s/item |
| Pipeline global | Temps traitement / heure de vidéo (Mac M-series) | ≤ 3× temps réel |

**Note sur les cibles** : revues à la baisse vs version antérieure du PRD pour calibrage sur la borne humain-humain de la littérature (κ humain-humain sur IES MTQ tourne typiquement entre 0.5 et 0.7 — un IA-vs-humain à 0.5 est déjà un succès opérationnel). Une cible inatteignable transforme un succès partiel en échec perçu. L'inertiel n'apparaît plus dans les KPIs : il est traité comme démonstrateur expérimental hors livrable v1 (cf. §7.1). Le pipeline ≤ 3× temps réel reflète l'orchestration séquentielle v1 (la cible 2× du PRD antérieur supposait `ProcessPoolExecutor` qui est différé).

### 9.2 Protocole PaveAudit

1. Sélectionner **5-10 segments de référence** dans la zone test Montréal (3-5 km de boucle), variés en état de chaussée.
2. **Inspection manuelle** par 1 ingénieur civil de référence (à recruter Phase 1, via réseau personnel ou recommandation académique).
3. Captation Repères des mêmes segments en 3 passages (matin/midi/fin de journée pour variabilité d'éclairage), véhicule unique pour limiter la variabilité de suspension.
4. Exécution du pipeline → `ies_visual` par segment + liste des défauts visuels.
5. Comparaison segment-à-segment :
   - Accord sur classification IES (κ Cohen)
   - Précision/recall par classe de défaut (mAP)
   - Erreur IES moyenne (RMSE)
6. Production de `validation_report.md` :
   - Tableau de confusion classification IES
   - Histogramme erreurs IES
   - Cas problématiques identifiés (occlusions, distance, conditions)
7. Inspection qualitative démonstrateur inertiel sur 5-10 segments d'intérêt : noter si le signal IMU semble corrélé qualitativement à la perception terrain. Pas d'agrégation chiffrée en v1 (Phase 2 si motivation).

### 9.3 Protocole ChantierWatch

1. Identifier **20-30 zones de chantier** captées à Montréal sur boucle de 10-20 km.
2. **Vérification manuelle** : pour chaque zone, croiser avec la base de permits OdP (téléchargée à la même date que la captation). Construire la vérité terrain `permitted` / `unmatched` / `permit_expired`.
3. Exécution du pipeline ChantierWatch → verdicts automatisés.
4. Comparaison zone-à-zone :
   - Recall détection (combien de chantiers réels ont été détectés ?)
   - Précision verdict `unmatched` (combien des "unmatched" sont vraiment des chantiers sans permit ?)
   - F1 cross-check temporel
5. **Validation manuelle des `unmatched`** par croisement Mtl-info / 311 (signalements citoyens) pour confirmer absence effective de permit.

### 9.4 Validation portage Sherbrooke

- Réplique du protocole sur 1-2 boucles à Sherbrooke en Phase 8.
- Mesure de la dégradation de performance (modèle entraîné sur Mtl appliqué à Sherbrooke).
- Si dégradation > 15 % en mAP : ajout de 500-1000 frames Sherbrooke au dataset, ré-entraînement.

---

## 10. Plan phasé

| Phase | Durée | Livrable | Effort estimé |
|---|---|---|---|
| **0 — Setup environnement** | 2 sem | venv Python, MPS PyTorch validé, calibration intrinsèque iPhone (lentille pinned), repo Git initialisé, vérif licence RDD2022 | 8-12 h |
| **1 — App iOS captation** | 3 sem | App SwiftUI déployée free-provisioning, bundle session produit (vidéo + GNSS + IMU 100 Hz + heading + manifest enrichi : lentille, pitch/roll mount, véhicule), mount-check | 30-45 h |
| **2 — Pipeline ingestion + Stage 1 commun** | 2 sem | `reperes ingest`, frames extraction, pose interpolation, détection événements inertiels normalisés vitesse, JSON schemas validés, R-tree Géobase chargé, `PreparedStreams` | 15-22 h |
| **3 — Pré-annotation + outil annotation web + orchestration Stage 2/3** | 4 sem | `reperes preannotate` avec Grounding DINO, outil web FastAPI+SvelteKit fonctionnel, orchestrateur `ProcessPoolExecutor` Stage 2, écriture consolidée Stage 3 | 35-50 h |
| **4 — Bootstrap dataset + fine-tune PaveAudit visuel** | 4 sem | Annotation 3-5k frames Mtl, fine-tune YOLO PaveAudit, mAP baseline mesurée | 35-50 h (annotation = 20-25 h) |
| **5 — Branche inertielle PaveAudit + Bootstrap ChantierWatch + adapter Mtl** | 4 sem | Détection événements inertiels intégrée à PaveAudit, `inertial_score` calculé par segment, fine-tune YOLO ChantierWatch, MontrealPermitAdapter, cross-check fonctionnel | 30-42 h |
| **6 — Livraison OGC + rapports** | 2 sem | `reperes serve` avec pygeoapi, template rapport PDF Quarto avec deux scores PaveAudit côte-à-côte | 14-20 h |
| **7 — Validation Montréal** | 2 sem | Captation + validation 5-10 segments + 20 zones, mesure Spearman inertiel/visuel, rapport de précision, recrutement ingénieur civil | 18-28 h |
| **8 — Portage Sherbrooke** | 2 sem | SherbrookePermitAdapter (ou GenericNoneAdapter), captation + validation portabilité | 12-18 h |
| **Total** | **~25 semaines** |  | **~200-290 h** |

À 8 h/semaine effectifs (soirs + 1 weekend par mois) : **6-7 mois calendaires**.
À 12 h/semaine (intense) : **5-6 mois calendaires**.

**Plan B si retard en Phase 5** : démo PaveAudit visuel seul en Phase 7 (sans branche inertielle), branche inertielle livrée en Phase 8 ou bonus. ChantierWatch livré en bonus visuel sans cross-check, finalisation cross-check Phase 8.

---

## 11. Risques et mitigations

### 11.1 Risques techniques

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| Grounding DINO + fine-tune ratent les défauts hivernaux subtils (frost heave, salt scaling) | Moyenne-haute | Cible IES non atteinte | Annotation manuelle dédiée à ces classes ; flag explicite dans rapport ; complément possible par signal inertiel pour les défauts de profil |
| Signal inertiel trop bruité (suspension véhicule, vibrations parasites du mount) — Spearman < 0.5 | Moyenne | KPI inertiel non atteint, signal rejeté en Phase 2 | Logger marque/modèle véhicule en manifest ; tester sur 2-3 véhicules différents en Phase 5 ; si bruit dominant, repositionner comme "événements inertiels seuls" sans score agrégé |
| Permits Sherbrooke peu/pas en open data | Élevée | ChantierWatch dégradé sur Sherbrooke | `GenericNoneAdapter` qui flag toutes zones `data_unavailable` ; livrable identifie explicitement la limitation |
| Pipeline parallèle : RAM MPS saturée avec 2+ modèles YOLO chargés simultanément | Faible | Crash Stage 2 ou swap silencieux | Pool size configurable (défaut = nb modules en v1, plafonnable si > 3 modules) ; mode `--no-parallel` pour debug |
| Side project, fenêtre 5-7 mois optimiste | Moyenne | Démo retardée | Découpage livraisons : démo PaveAudit visuel seul à 4-5 mois si branche inertielle ou ChantierWatch retardent ; plan B documenté |
| Validation κ ≥ 0.6 non atteinte malgré fine-tune | Moyenne | Cible qualité non atteinte | Calibrer scoring vs MTQ sur données réelles avant de promettre ; ajuster les seuils MTQ ; repositionner le module en "outil de pré-priorisation" |
| Volumes de vidéos ingérables | Faible | Sessions limitées | H.265 4K = ~6 GB/h ; SSD externe 1-2 TB suffit pour la durée du PoC |
| iPhone 16 Pro indisponible / ancienne génération moins précise | Faible | Précision GNSS dégradée, IMU moins fiable | Tester sur iPhone 15 Pro en backup ; mesurer empiriquement |
| Lentille AVFoundation bascule automatiquement (Wide → UltraWide en basse lumière) invalidant la calibration intrinsèque | Faible-moyenne | Détections ratées ou bbox déformées en certaines conditions | Lentille `builtInWideAngleCamera` pinned explicitement, fallback désactivé ; logged dans manifest ; rejet de session en post-traitement si lentille a changé |

### 11.2 Risques juridiques et de licences

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| Dataset QC non utilisable hors recherche (licences source) | Faible | Modèle final restreint à un cadre académique | Vérifier licences en Phase 0 ; toute donnée incertaine → exclue ; possibilité re-train from scratch sur dataset 100 % maison Phase 2 |
| Loi 25 — vidéos avec plaques/visages | Moyenne | Risque légal sur la donnée brute | Stockage local SSD chiffré v1 ; floutage automatique avant tout partage externe (Phase 2, modèle ANPR + MTCNN) ; ne pas exporter de vidéos brutes |
| Disclaimer "signal d'audit" insuffisant en cas de litige | Faible | Risque légal | Avis juridique requis avant tout usage du livrable hors cadre interne ; limitation de responsabilité explicite dans chaque rapport |

### 11.3 Risques projet

- **Sous-estimation du temps d'annotation** : à minimiser, c'est facile à allonger. Bloquer la Phase 4 si volume annoté < 1500 frames à mi-phase et reconsidérer l'embauche d'un stagiaire Mitacs pour bootstrap dataset.
- **Dérive de scope** : ne pas implémenter SignageStateEvaluator ni MarkingEvaluator ni floutage auto avant que PaveAudit + ChantierWatch ne soient livrés et validés.
- **Météo défavorable au planning de captation** : prévoir buffer (Phase 7-8) pour repasser en cas de pluie/neige rendant les captures inexploitables.
- **Disponibilité ingénieur civil de référence** : démarcher dès Phase 1 (réseau personnel ou recommandation prof Poly/ÉTS), ne pas attendre Phase 7.

---

## 12. Évolutions Phase 2+

À considérer **uniquement si le PoC v1 atteint les KPIs** et que la motivation produit reste.

### 12.1 Enrichissement modules existants

- **Calibration véhicule pour score inertiel absolu** : sessions de calibration sur segments connus (avec mesure IRI de référence par profileur si possible), dérivation d'un facteur de transfert par véhicule. Permet la comparabilité inter-sessions et l'alignement sur l'IRI ASTM E1926.
- **Fusion vision + inertiel dans un score unique IES augmenté** : après accumulation de données en Phase 2, calibration empirique de poids `w_visual`, `w_inertiel` ou modèle ML léger (régression) qui apprend à combiner les deux signaux. Livré comme `ies_combined` en complément des deux scores parallèles.
- **Suivi temporel inter-frames** (ByteTrack ou BoT-SORT intégrés à Ultralytics) : passage du clustering spatio-temporel post-hoc à un vrai tracker pour stabilisation des `track_id` cross-modules et amélioration de précision sur objets mobiles (ChantierWatch).
- **Floutage automatique plaques + visages** : modèle ANPR + MTCNN ou similaire dans le pipeline avant tout partage externe.
- **Comparaison temporelle multi-passages** : disparitions/apparitions/aggravations entre 2 captations de la même zone.

### 12.2 Nouveaux évaluateurs

- `SignageStateEvaluator` — état des panneaux (penchés, graffités, effacés)
- `MarkingEvaluator` — état du marquage chaussée (lignes effacées, manquantes, peinture éraflée)
- `StreetFurnitureEvaluator` — état du mobilier urbain (lampadaires, abribus, bancs, poubelles)

### 12.3 Mode opérationnel

- **Mobile mapping continu** sur véhicules municipaux (autobus STL/STM, balayeuses) Phase 3.
- **API d'écriture authentifiée** pour validation par utilisateur identifié.
- **Dashboard web** de supervision multi-sessions, multi-mandats, multi-clients.
- **Auth + permissions** pour usage multi-utilisateur.

### 12.4 Ouverture et recherche

- Publication du **dataset Quebec Pavement Winter Damage** sur HuggingFace Hub (CC-BY-NC).
- DATASHEET.md selon Gebru et al. 2018 + croissant.json pour métadonnées ML.
- Citation académique (CITATION.cff) si publication scientifique.

### 12.5 Cloud et passage à l'échelle

- Pipeline déchargé sur **serveur Linux + GPU NVIDIA** (post-captation) pour mandats volumineux.
- **Stockage objet** (B2 / GCS / bucket institutionnel) pour vidéos brutes archivées.
- **CI/CD** sur les modèles : versioning des poids (DVC + bucket), métriques de régression entre versions.

---

## 13. Glossaire

| Terme | Définition |
|---|---|
| **Repères** | Nom du produit (présent document, repo `reperes`, CLI `reperes`) |
| **PaveAudit** | Module d'évaluation d'état de chaussée selon méthodologie MTQ (vision) + signal inertiel proxy IRI |
| **ChantierWatch** | Module de détection de chantiers et cross-check avec permits OdP |
| **IES** | Indice d'État Subjectif — score visuel 0-100 de l'état de chaussée selon MTQ |
| **`ies_visual`** | Score IES MTQ calculé par la branche visuelle de PaveAudit |
| **`inertial_score`** | Score 0-100 par segment dérivé du RMS d'accélération verticale normalisée par vitesse, relatif intra-session |
| **IRI** | International Roughness Index — index inertiel ASTM E1926 mesurable par profileur professionnel ; le `inertial_score` v1 est un proxy non-calibré |
| **MTQ** | Ministère des Transports du Québec |
| **PCI** | Pavement Condition Index — standard ASTM D6433 (référence internationale, distincte de l'IES MTQ) |
| **OdP** | Occupation du Domaine Public — permis municipal pour chantier sur voie publique |
| **κ Cohen** | Coefficient de Kappa Cohen — mesure d'accord inter-annotateur, 0.6+ = "substantial agreement" |
| **Spearman** | Coefficient de corrélation de rang Spearman — mesure de monotonie entre deux séries, ne nécessite pas linéarité |
| **mAP** | Mean Average Precision — métrique standard de détection d'objets |
| **GNSS** | Global Navigation Satellite Systems (GPS, GLONASS, Galileo, BeiDou) |
| **IMU** | Inertial Measurement Unit (accéléromètres + gyroscopes + magnétomètre) — sur iPhone, exposé via CoreMotion |
| **`PreparedStreams`** | Objet produit par Stage 1 du pipeline, consommé par tous les évaluateurs en Stage 2 (frames, pose, événements inertiels) |
| **Stage 1 / 2 / 3** | Étages du pipeline Mac : préparation commune / évaluateurs parallèles / consolidation |
| **OGC API Features** | Standard Open Geospatial Consortium pour la publication de features géographiques en REST/JSON |
| **NAD83 MTM zone 8** | Système de coordonnées projeté pour le sud du Québec, EPSG:32188 |
| **Grounding DINO** | Modèle CV open-vocabulary par IDEA-Research, détection bbox via prompts texte |
| **YOLO-World** | Modèle CV open-vocabulary par Tencent, basé YOLOv8, détection via prompts texte |
| **DBSCAN** | Density-Based Spatial Clustering — algo de clustering non-paramétrique pour zones de chantier |
| **R-tree** | Structure d'index spatial pour requêtes nearest-neighbor et intersection rapides en mémoire |
| **Géobase** | Référentiel routier provincial du Québec, données ouvertes |
| **MPS** | Metal Performance Shaders, accélérateur Apple Silicon pour PyTorch |
| **`ProcessPoolExecutor`** | Mécanisme Python `concurrent.futures` pour parallélisation par sous-processus (vrai parallélisme CPU+MPS) |
| **Active learning loop** | Boucle d'amélioration : prédiction → correction humaine → ré-entraînement |
| **Pré-annotation** | Génération automatique de bbox candidats par modèle open-vocab, à réviser par humain |
| **Free provisioning** | Déploiement Xcode avec Apple ID gratuit (vs Apple Developer Program payant) |
| **Loi 25** | Loi modernisant la protection des renseignements personnels (Québec, en vigueur 2024) |

---

## 14. Décisions ouvertes

Format : `[ID] Décision — Owner — Échéance — Statut — Bloquant si non résolu en`. L'auteur étant unique contributeur en v1, `Owner = Julien` partout ; le champ est conservé pour formalité et pour usage futur si un partenaire rejoint le projet. Les échéances sont relatives au démarrage projet (T0 = première journée Phase 0).

| ID | Décision | Owner | Échéance | Statut | Bloquant si non résolu en |
|---|---|---|---|---|---|
| D-01 | Confirmer disponibilité iPhone 16 Pro personnel pour le projet (jamais de l'employeur) | Julien | T0 + 3 j | À faire | Phase 1 |
| D-02 | Vérifier licence exacte RDD2022 (académique stricte ? CC-BY ? CC-BY-NC ?) — lecture terms of use + email auteurs si ambigu | Julien | T0 + 14 j | À faire | Phase 4 (Stade B) |
| D-03 | Vérifier disponibilité permits OdP open data pour Longueuil et Sherbrooke (sinon `GenericNoneAdapter` ou abandon zone) | Julien | T0 + 14 j | À faire | Phase 5 / Phase 8 |
| D-04 | Identifier 1 ingénieur civil de référence pour validation IES — réseau perso ou contact prof Poly/ÉTS | Julien | T0 + 90 j | À faire | Phase 7 (validation finale) |
| D-05 | Décision Stade B vs Stade B' selon résultat D-02 | Julien | Fin Phase 0 (T0 + 14 j) | Bloquée par D-02 | Phase 4 |
| D-06 | EFVP/PIA Loi 25 légère à produire avant captation hors zones auteur | Julien | T0 + 30 j | À faire | Phase 1 captation Mtl |
| D-07 | Vérifier chiffrement FileVault actif sur Mac + data-protection iOS sur iPhone | Julien | T0 + 1 j | À faire | Phase 1 |

Statuts possibles : `À faire` · `En cours` · `Résolu (lien commit ou doc)` · `Bloquée par <ID>` · `Reportée Phase 2+`.

**Décisions stratégie d'affaire** (hors PRD, traitées dans `docs/marketing/`) : nom commercial, incorporation, tarification, choix d'audience pour les premières démos, règles internes d'activités accessoires de l'employeur.

---

*Fin du document. 2026-04-27.*
