---
marp: true
theme: default
paginate: true
size: 16:9
header: 'wsmd · PaveAudit'
footer: 'Confidentiel — pour discussion · 2026'
style: |
  section {
    font-family: 'Helvetica Neue', Arial, sans-serif;
    padding: 50px 60px;
  }
  h1 {
    color: #1a3a5c;
    border-bottom: 3px solid #1a3a5c;
    padding-bottom: 8px;
  }
  h2 {
    color: #2c5f8d;
  }
  table {
    font-size: 0.9em;
    border-collapse: collapse;
  }
  th {
    background-color: #1a3a5c;
    color: white;
  }
  th, td {
    border: 1px solid #ccc;
    padding: 6px 12px;
  }
  .highlight {
    background-color: #fff3cd;
    padding: 2px 6px;
    border-radius: 3px;
  }
  .small {
    font-size: 0.8em;
    color: #666;
  }
---

<!--
Pitch deck pour firmes de génie — focus PaveAudit.

Pour générer :
  marp docs/marketing/pitch-firme.md -o /tmp/pitch-firme.pdf --allow-local-files
  marp docs/marketing/pitch-firme.md -o /tmp/pitch-firme.pptx --allow-local-files

Champs à compléter avant envoi : [Nom], [Courriel], [LinkedIn], [Nom du contact],
[Nom de la firme] (si version personnalisée).

Discipline conflit d'intérêts : aucune référence à la municipalité-employeur.
Tous les exemples et zones de captation sont Montréal/Longueuil/Sherbrooke.
-->

<!-- _class: lead -->
<!-- _paginate: false -->

# wsmd · PaveAudit

## Pré-classement automatisé d'état de chaussée selon méthodologie MTQ

Pour les firmes de génie sous-traitant des audits voirie

[Nom] · [Courriel] · 2026

---

# Le problème

Vos mandats d'audit visuel MTQ et municipaux mobilisent **des jours-personne d'inspecteur certifié** — le facteur coût et calendrier dominant.

| Mandat type | jp d'inspection visuelle |
|---|---|
| 50 km de réseau urbain | 7-10 jp |
| 200 km de réseau régional | 25-40 jp |
| 500 km (campagne complète MTQ) | 60-100 jp |

À ~600-800 $/jp facturés et un cycle d'audit qui se répète **tous les 3-5 ans**, c'est un poste structurel coûteux qu'aucun outil grand public ne réduit aujourd'hui pour le marché québécois.

<span class="small">Estimations basées sur des productivités d'inspection visuelle typiques de 5-10 km/jp.</span>

---

# Notre proposition

**wsmd PaveAudit pré-classifie 80 % des défauts à 75 %+ d'agreement avec un inspecteur certifié.**

L'inspecteur passe en mode **vérification ciblée** :

- 1-2 h par segment au lieu de 1-2 h par km
- Focus sur les segments problématiques pré-identifiés
- Validation et signature des livrables

<div class="highlight">

**Économie estimée : 60-70 % des jp d'inspection visuelle**

</div>

Sur un mandat de 200 km : 25-40 jp → **8-15 jp**.

---

# Comment ça marche

```
iPhone 16 Pro mounté pare-brise (votre véhicule, votre conducteur)
    ↓
30 min de captation = 10-20 km de réseau couvert
    ↓
Pipeline post-captation Mac (notre infrastructure, vos données restent locales)
    ↓
Livrable : GeoPackage + rapport PDF + couche OGC consommable QGIS/ArcGIS
```

**Aucun matériel professionnel requis.** Pas de RTK, pas de profileur, pas de stéréo.
**Vos données restent chez vous.** Traitement 100 % local, conformité Loi 25 par design.

---

# Méthodologie alignée MTQ

| Élément | Approche |
|---|---|
| **Catalogue de défauts** | 8 classes MTQ (nid-de-poule, fissure long./trans., faïençage, ressuage, patch) + spécificités QC (frost heave, salt scaling) |
| **Indices de sévérité** | Légère / Modérée / Sévère selon barèmes MTQ |
| **Score par segment** | IES 0-100 calculé par densité × sévérité × valeurs de déduction MTQ |
| **Géoréférencement** | Snap-to-road sur Géobase Québec, segments 50-100 m |
| **Projection** | NAD83 MTM zone 8 (EPSG:32188) — cohérence avec vos SIG |

**Limitation honnête** : l'**IRI** (International Roughness Index) nécessite un profileur inertiel. PaveAudit produit l'**indice visuel uniquement**. Le couplage IRI via accéléromètres iPhone est en roadmap Phase 2.

---

# Performance cible (PoC v1)

| KPI | Cible | Méthode de validation |
|---|---|---|
| **κ Cohen** sur classification IES vs ingénieur civil | ≥ 0.6 (substantial agreement) | 5-10 segments captés à Montréal, indexés manuellement |
| **mAP** par classe de défaut | ≥ 0.75 sur 8 classes | Test set Québec annoté |
| **Erreur IES moyenne** par segment | ±10 / 100 | Comparaison segment-à-segment |
| **Répétabilité** entre 3 passages | < 5 points IES | Captations multi-passes |
| **Temps de traitement** | ≤ 2× temps réel | Benchmark Mac M-series |

**Engagement** : la précision est mesurée et contractualisée. Si κ < 0.6 sur votre échantillon de validation, **retravail à nos frais**.

---

# Différenciateurs vs concurrence

| Compétiteur | Approche | Limite |
|---|---|---|
| **RoadBotics** (Michelin Mobility Intel.) | SaaS dashcam IA, 5-50 k$/an | Modèles US, pas adaptés à l'hiver QC, méthodo PCI ASTM |
| **Vialytics** (DE) | SaaS smartphone, abonnement | Européen, méthodologie non-MTQ |
| **Carbin** (MIT spinoff) | Smartphone, accent recherche | Pas de produit commercial mature |
| **Profileurs pros** (Trimble, Leica) | Mesure IRI/PCI complète, équipement 250 k$+ | Inaccessibles pour mandats à coût-cible bas |

**wsmd PaveAudit** : **méthodologie MTQ native**, **modèles entraînés sur défauts hivernaux QC**, **mode sous-traitance ponctuelle** (per-mandat ou per-km), **pas d'abonnement annuel imposé**.

---

# Ce qu'on livre par mandat

**GeoPackage (`assets.gpkg`)**
- Layer `pavement_segments` : segments routiers + IES + classification MTQ
- Layer `pavement_defects` : défauts individuels (point + bbox + frame source)
- Lisible **QGIS, ArcGIS, GDAL**, importable dans vos SIG existants

**Rapport PDF (10-30 pages selon mandat)**
- Sommaire exécutif (km, défauts, IES médian)
- Carte du réseau colorée par IES
- Top 20 segments les plus dégradés avec photos
- Tableau complet en annexe

**Service OGC API Features** (optionnel)
- `pygeoapi` configuré sur le GeoPackage
- Consommable directement par votre SIG via WFS

---

# Modèle commercial

**Tarification au km audité** (à valider avec premiers clients pilotes — fourchette indicative).

| Volume | Approche |
|---|---|
| **Mandat pilote** (Q4 2026) | Gratuit ou coût matière, en échange de : (a) accès aux données pour calibration, (b) feedback structuré, (c) référence si succès |
| **Mandats commerciaux** (2027+) | Per-km ou per-mandat, engagement précision contractualisé |
| **Volume récurrent** | Tarification dégressive, abonnement annuel possible |

**Toujours inclus** : retravail à nos frais si les KPIs ne sont pas atteints sur l'échantillon de validation client.

---

# Roadmap

| Période | Étape |
|---|---|
| **Q3 2026** | PoC v1 livré, validation Montréal complète, KPIs mesurés et publiés |
| **Q4 2026** | **Mandat pilote** avec firme partenaire (50-100 km, gratuit ou coût matière) |
| **Q1 2027** | Premiers mandats commerciaux, ajustements basés sur retours pilote |
| **2027** | Module **ChantierWatch** mature (détection chantiers + cross-check permits) |
| **2027-2028** | Modules supplémentaires : signalisation, marquage chaussée, mobilier urbain |
| **Phase 2** | Couplage **IRI** via accéléromètres iPhone — livrable comparable aux profileurs pros |

---

# Pourquoi maintenant

- **Modèles CV open-source matures** : YOLOv8/v11, Grounding DINO, YOLO-World atteignent en 2025-2026 la qualité requise pour la voirie
- **iPhone 16 Pro** = capteur GNSS bi-bande L1/L5, IMU haute fréquence, caméra 4K HDR — un device unique remplace une rig matérielle
- **Données ouvertes Québec** publiées en continu (Géobase, Données Québec, Mtl ouvert) — exploitables sans contrats préalables
- **Loi 25** crée un avantage structurel pour les solutions **traitant localement au Québec** (vs SaaS cloud étranger)

**Le moat est québécois et il se construit maintenant.**

---

# Prochaines étapes proposées

1. **Rencontre technique** (1 h) : approfondissement, questions/réponses, alignement sur vos protocoles d'audit internes

2. **Identification d'un mandat pilote** : 50-100 km de réseau représentatif de votre portefeuille, captation Q4 2026

3. **Validation conjointe** : votre ingénieur civil de référence valide les sorties wsmd sur 5-10 segments

4. **Décision commerciale** : si les KPIs sont atteints, contrat-cadre pour mandats 2027+

---

<!-- _class: lead -->
<!-- _paginate: false -->

# Discussion

**[Nom]**
[Courriel] · [LinkedIn]

Documents techniques disponibles sur demande :
PRD complet · spécifications de validation · exemples de livrables

<span class="small">wsmd est un projet personnel et indépendant. Ce document n'engage aucune municipalité ni employeur de l'auteur.</span>
