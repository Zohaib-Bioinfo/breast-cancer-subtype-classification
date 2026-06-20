# Breast Cancer Molecular Subtype Classification

Machine learning pipeline for classifying breast cancer molecular subtypes (Basal, HER2, Luminal A, Luminal B) from gene expression microarray data using the GSE45827 dataset.

## Overview

This project applies XGBoost classification to Affymetrix GPL570 microarray expression profiles to distinguish the four major intrinsic molecular subtypes of breast cancer. The pipeline includes feature selection, model training, cross-validation, biomarker identification, SHAP-based explainability, and pathway enrichment analysis.

## Dataset

- **Source:** [GSE45827](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE45827) (Gene Expression Omnibus)
- **Platform:** GPL570 (Affymetrix Human Genome U133 Plus 2.0 Array)
- **Samples used:** 130 primary tumor samples
  - Basal: 41
  - HER2: 30
  - Luminal A: 29
  - Luminal B: 30

## Methods

1. **Data acquisition** — Downloaded via `GEOparse`, expression matrix extracted with subtype labels from sample metadata.
2. **Feature selection** — Top 300 probes selected using mutual information (`SelectKBest`, `mutual_info_classif`).
3. **Classification** — XGBoost multi-class classifier (`XGBClassifier`).
4. **Evaluation** — Stratified 5-fold cross-validation (leak-free, using `sklearn.Pipeline`) as the primary performance metric, since the dataset size (n=130) makes a single train/test split unreliable.
5. **Explainability** — SHAP (TreeExplainer) for feature-level model interpretation.
6. **Biomarker mapping** — Probe IDs converted to HGNC gene symbols via `mygene`.
7. **Pathway enrichment** — Over-representation analysis against KEGG and GO Biological Process using `gseapy` (Enrichr).

## Results

| Metric | Value |
|---|---|
| 5-Fold CV Accuracy | 94.6% ± 3.1% |
| ROC-AUC (Basal) | 1.000 |
| ROC-AUC (HER2) | 0.999 |
| ROC-AUC (Luminal A) | 0.987 |
| ROC-AUC (Luminal B) | 0.981 |

Top enriched biological pathways among selected biomarker genes include cell cycle regulation, mitotic spindle checkpoint signaling, and PI3K signaling — consistent with known proliferative and signaling differences across breast cancer subtypes.

### Visualizations

- `figures/PCA_subtypes.png` — PCA showing subtype separability
- `figures/biomarker_heatmap.png` — Expression heatmap of top biomarker genes
- `figures/ROC_CV_curves.png` — Cross-validated multi-class ROC curves
- `figures/enrichment_dotplot.png` — Pathway enrichment results
- `figures/confusion_matrix.png` — Classification confusion matrix

## Repository Structure

```
breast-cancer-subtype-classification/
├── README.md
├── requirements.txt
├── notebooks/
│   └── BreastCancerMolecularSubtypeClassification.ipynb
├── figures/
│   └── (generated plots)
├── results/
│   └── top_biomarkers.csv
└── .gitignore
```

## Setup

```bash
git clone https://github.com/<your-username>/breast-cancer-subtype-classification.git
cd breast-cancer-subtype-classification
pip install -r requirements.txt
```

Run the notebook in `notebooks/` (Jupyter or Google Colab).

## Limitations

- Sample size (n=130) is modest for a 4-class problem; cross-validation accuracy is reported as the primary metric rather than a single held-out test score.
- Microarray-based expression (not RNA-seq); platform-specific normalization effects may apply.
- Class imbalance (Basal: 41 vs. Luminal A: 29) is mitigated via stratified sampling but not explicitly reweighted.

## Author

Muhammad Zohaib — BS Bioinformatics, University of Agriculture Faisalabad (UAF)

## License

MIT License (see `LICENSE`)
