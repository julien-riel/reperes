---
marp: true
theme: default
paginate: false
size: 210mm 297mm
style: |
  @page { size: A4 portrait; }
  section {
    padding: 14mm 16mm;
    font-size: 10.5pt;
    font-family: 'Helvetica Neue', Arial, sans-serif;
    line-height: 1.4;
    color: #222;
    background: white;
  }
  h1 {
    font-size: 22pt;
    color: #1a3a5c;
    margin: 0 0 1mm 0;
    border-bottom: 2px solid #1a3a5c;
    padding-bottom: 2mm;
  }
  h2 {
    font-size: 13pt;
    color: #1a3a5c;
    margin: 5mm 0 2mm 0;
  }
  h3 {
    font-size: 11pt;
    color: #2c5f8d;
    margin: 3mm 0 1mm 0;
  }
  p { margin: 1.5mm 0; }
  ul { margin: 1mm 0 2mm 0; padding-left: 5mm; }
  li { margin: 0.5mm 0; }
  strong { color: #1a3a5c; }
  table {
    font-size: 9.5pt;
    border-collapse: collapse;
    width: 100%;
    margin: 2mm 0;
  }
  th, td {
    border: 1px solid #d0d0d0;
    padding: 1.5mm 2.5mm;
    text-align: left;
    vertical-align: top;
  }
  th {
    background-color: #1a3a5c;
    color: white;
    font-weight: 600;
  }
  hr {
    border: none;
    border-top: 1px solid #ccc;
    margin: 4mm 0 2mm 0;
  }
  .tagline {
    font-size: 11pt;
    color: #555;
    margin: 0 0 3mm 0;
    font-style: italic;
  }
  .footer {
    font-size: 10pt;
    color: #444;
    margin-top: 3mm;
  }
---

<!--
One-pager Marp en A4 portrait.

Pour rendre :
  marp docs/marketing/one-pager.md -o /tmp/one-pager.pdf --allow-local-files

Champs à compléter avant envoi : [Nom], [Courriel], [LinkedIn / site web].
Discipline conflit d'intérêts : ne pas modifier ce document pour y inclure
de référence à la municipalité-employeur.
-->

# wsmd — Audit automatisé d'actifs viaires

<p class="tagline">Plateforme d'audit visuel de la voirie par vidéo géoréférencée et IA — adaptée aux standards et données ouvertes du Québec.</p>

## Le problème

Les firmes de génie et services d'inspection municipaux maintiennent l'état du domaine public viaire par **inspection visuelle manuelle** — coûteuse en jours-personne d'inspecteur certifié, peu fréquente, et qui ne tire pas parti des **données ouvertes municipales** (permits OdP, géobase, inventaires) déjà publiées.

## La solution — deux modules sur une seule plateforme

### Module **PaveAudit** · état de chaussée selon méthodologie MTQ

- Détection automatique de **8 classes de défauts** : nids-de-poule, fissures longitudinales/transversales, faïençage, ressuage, patches, **soulèvement par gel**, **dégradation au sel**
- Score **IES 0-100** par segment routier (50-100 m), aligné catalogue MTQ
- Livrable : **GeoPackage + rapport PDF** avec carte IES, top segments dégradés, photos
- **Pour** : firmes de génie sous-traitant des audits MTQ/municipaux
- **Cible précision** : κ Cohen ≥ 0.6 vs inspecteur civil certifié, mAP ≥ 75 %

### Module **ChantierWatch** · détection chantiers + cross-check permits

- Détection automatique de **zones de chantier** : cônes, barrières Jersey, signalisation temporaire, véhicules de construction, clôtures, déblais
- **Croisement automatique** avec les données ouvertes de permits OdP (Données Québec, Mtl ouvert)
- Verdict par zone : `permitted` / `unmatched` / `permit_expired` / `data_unavailable`
- Livrable : liste de **chantiers à vérifier sur le terrain** par les inspecteurs autorisés
- **Pour** : services d'inspection et de conformité municipaux
- **Cible précision** : 85 % recall détection, 90 % précision verdict `unmatched`

## Pourquoi nous

| Différenciateur | Détail |
|---|---|
| **Méthodologie MTQ native** | Aligné aux standards et catalogues québécois, pas une importation US adaptée |
| **Défauts hivernaux QC** | Modèles entraînés sur frost heave, salt scaling, faïençage post-gel |
| **Conformité Loi 25 par design** | Traitement 100 % local, aucun envoi cloud, données chez le client |
| **Open data québécois exploité** | Géobase, Données Québec, Mtl ouvert intégrés directement |
| **Architecture extensible** | Modules signalisation, marquage, mobilier ajoutables sans refonte |

## Stack technique

iPhone 16 Pro · Python 3.12 · PyTorch MPS · Ultralytics YOLOv8 fine-tuné · Grounding DINO (pré-annotation) · OSRM (snap-to-road) · pygeoapi (OGC API Features) · GeoPackage (OGC standard).

**Captation** : un seul iPhone au pare-brise, pas de matériel professionnel requis. **Post-traitement** : Mac M-series, ~2× temps réel pour 30 min de vidéo.

## Statut et disponibilité

PoC v1 en développement actif. Validation empirique Q3-Q4 2026 sur trois zones tests (Montréal, Longueuil, Sherbrooke). **Disponible pour mandats pilotes** (gratuit ou coût matière) à partir de Q4 2026.

---

<p class="footer"><strong>Contact</strong> : [Nom] · [Courriel] · [LinkedIn / site web]</p>
