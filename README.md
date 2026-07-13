# Breast Cancer Molecular Subtype Classification

Machine learning pipeline for classifying breast cancer molecular subtypes (Basal, HER2, Luminal A, Luminal B) from gene expression microarray data — plus systematic robustness testing of the resulting biomarker panel before downstream prognostic validation.

This repository covers **biomarker discovery and pipeline robustness testing**. Prognostic validation of the resulting gene panel in the METABRIC cohort is covered in the companion repository: [breast-cancer-survival-biomarkers](https://github.com/Zohaib-Bioinfo/breast-cancer-survival-biomarkers).

## Overview

This project applies XGBoost classification to Affymetrix GPL570 microarray expression profiles to distinguish the four major intrinsic molecular subtypes of breast cancer, then subjects the resulting 46-gene SHAP panel to four independent robustness analyses before it is trusted for survival testing: classifier benchmarking, feature selection stability, external cohort validation, and extended pathway enrichment.

## Dataset

- **Source:** [GSE45827](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE45827) (Gene Expression Omnibus) — discovery cohort
- **Platform:** GPL570 (Affymetrix Human Genome U133 Plus 2.0 Array)
- **Samples used:** 130 primary tumor samples (Basal: 41, HER2: 30, Luminal A: 29, Luminal B: 30)
- **External validation:** [GSE21653](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE21653) — independent, platform-matched (GPL570) cohort, n=266

## Methods

1. **Data acquisition** — Downloaded via `GEOparse`, expression matrix extracted with subtype labels from sample metadata.
2. **Feature selection** — Top 300 probes selected using mutual information (`SelectKBest`, `mutual_info_classif`).
3. **Classification** — XGBoost multi-class classifier (`XGBClassifier`), 94.6% ± 3.1% 5-fold CV accuracy.
4. **Explainability** — SHAP (TreeExplainer) for feature-level model interpretation; top 46 genes retained.
5. **Classifier comparison** — Benchmarked against Random Forest, SVM, and Logistic Regression under identical preprocessing. Reported transparently: RF and SVM scored numerically higher on raw CV accuracy (96.2% vs. XGBoost's 92.3%); XGBoost retained for compatibility with SHAP's attribution framework, not raw accuracy.
6. **Feature selection stability** — 20 repeated stratified resampling runs, evaluated at both a broad threshold (top-300 retention) and a strict threshold (top-50 SHAP-rank retention, matching the original panel-defining criterion).
7. **External classification validation** — Frozen production model applied without refitting to GSE21653, evaluated against both native IHC-surrogate labels and genefu-derived PAM50 intrinsic labels.
8. **Extended pathway enrichment** — GO, KEGG (original panel) plus Reactome (41 significant pathways, FDR<0.05), specifically annotating CDCA5 to sister chromatid separation.

## Results

| Metric | Value |
|---|---|
| 5-Fold CV Accuracy (XGBoost) | 94.6% ± 3.1% |
| ROC-AUC (Basal) | 1.000 |
| ROC-AUC (HER2) | 0.999 |
| ROC-AUC (Luminal A) | 0.987 |
| ROC-AUC (Luminal B) | 0.981 |
| External validation accuracy (vs. IHC-surrogate labels, GSE21653) | 63.7% |
| External validation accuracy (vs. genefu-intrinsic labels, GSE21653) | 77.1% |
| CDCA5 strict-threshold feature stability (20 runs) | 50% (10/20) |
| CMC2 strict-threshold feature stability (20 runs) | 20% (4/20) |

**Key finding:** CDCA5 and CMC2 — the two genes independently prognostic in the companion repository's METABRIC analysis — are **not equally well-supported** by this pipeline's robustness testing. CDCA5 shows convergent evidence across every axis (top SHAP rank, moderate stability, Reactome annotation); CMC2's supporting evidence is comparatively thinner (SHAP rank 47/50, low stability, no Reactome hit).

Top enriched biological pathways among selected biomarker genes include cell cycle regulation, mitotic spindle checkpoint signaling, sister chromatid separation (Reactome), and PI3K signaling — consistent with known proliferative and signaling differences across breast cancer subtypes.

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
├── .gitignore
├── notebooks/
│   ├── BreastCancerMolecularSubtypeClassification.ipynb   # Original discovery pipeline
│   ├── 02_classifier_comparison.ipynb                     # XGBoost vs RF, SVM, Logistic Regression
│   ├── 03_feature_stability_analysis.ipynb                # 20-run repeated resampling stability
│   ├── 04_external_validation_GSE21653.ipynb              # Frozen-model external validation
│   └── 05_genefu_intrinsic_labels_GSE21653.R              # PAM50 intrinsic re-labeling (R/genefu)
│                      
├── models/                                                 # Frozen production model artifacts
│   ├── frozen_selector.pkl
│   ├── frozen_scaler.pkl
│   ├── frozen_model.pkl
│   ├── frozen_label_encoder.pkl
│   └── selected_probe_ids.pkl
├── figures/
│   └── (generated plots)
├── results/
│   ├── top_biomarkers.csv
│   ├── classifier_comparison/
│   ├── feature_stability/
│   ├── external_validation/
│   └── pathway_enrichment_results.csv
└── manuscript/
    └── (full manuscript + supporting information)
```

## Setup

```
git clone https://github.com/Zohaib-Bioinfo/breast-cancer-subtype-classification.git
cd breast-cancer-subtype-classification
pip install -r requirements.txt
```

Run `notebooks/BreastCancerMolecularSubtypeClassification.ipynb` first (Jupyter, Colab, or Kaggle) to reproduce the frozen production model in `models/`. Notebooks 02-04 and 06 depend on these frozen artifacts. Notebook 05 requires an R environment (Kaggle or Colab R kernel) with Bioconductor packages `genefu`, `GEOquery`, and `hgu133plus2.db` — see in-notebook comments for exact installation steps, including a known genefu/Bioconductor-3.20 build issue worked around via GitHub source install.

## Limitations

- Sample size (n=130) is modest for a 4-class problem; cross-validation accuracy is reported as the primary metric rather than a single held-out test score.
- Microarray-based expression (not RNA-seq); platform-specific normalization effects may apply.
- Class imbalance (Basal: 41 vs. Luminal A: 29) is mitigated via stratified sampling but not explicitly reweighted.
- External validation performed in a single independent cohort (GSE21653); replication in additional platform-matched cohorts would further strengthen generalizability claims.
- Classifier comparison and stability results should be read alongside the confidence intervals reported in the manuscript — differences between algorithms are not always robustly distinguishable given the necessarily limited number of cross-validation folds on n=130.

## Citation

If you use this pipeline or its outputs, please cite the associated manuscript (see `manuscript/`).

## Author

Muhammad Zohaib — BS Bioinformatics, Department of Computer Science, University of Agriculture Faisalabad (UAF)

## License

MIT License (see `LICENSE`)
