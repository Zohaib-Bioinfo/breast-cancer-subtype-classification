# Breast Cancer Subtype Classification — Pipeline
### (Repository 1 of 2 — see [Companion Repository](#-companion-repository-breast-cancer-survival-biomarkers) below)

This repository contains the primary machine learning and bioinformatics pipeline designed to answer two central research questions: 
1. Can a compact XGBoost model trained on GSE45827 robustly classify PAM50 molecular subtypes from microarray expression data?
2. Which transcriptomic features drive that classification, and are they biologically coherent (pathway enrichment) and externally reproducible (GSE21653)?

The final output of this repository is a heavily validated, reannotated biomarker gene panel. This panel serves as the direct programmatic input to our companion repository, which tests whether these genes carry independent prognostic value in the METABRIC cohort.

---

## 📊 Datasets

- **Source:** [GSE45827](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE45827) (Gene Expression Omnibus) — Discovery cohort
- **Platform:** GPL570 (Affymetrix Human Genome U133 Plus 2.0 Array)
- **Samples used:** 130 primary tumor samples (Basal: 41, HER2: 30, Luminal A: 29, Luminal B: 30)
- **External validation:** [GSE21653](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE21653) — Independent, platform-matched (GPL570) cohort, n=266

---

## ⚙️ Execution Order & Automated Hand-offs

**Read this before running anything:** The notebook numbers describe the *reading/manuscript* order, not the *execution* order. 

This pipeline is designed for strict computational reproducibility via **fully automated programmatic hand-offs**. Artifacts (models, scalers, and CSV data tables) are saved to disk and seamlessly ingested by downstream notebooks, ensuring zero manual transcription errors between Python and R kernels. 

**Run the pipeline in this exact order:**

1. **`01_internal_validation_GSE45827.ipynb`** (Independent execution)
2. **`02_classifier_comparison.ipynb`** (Independent execution)
3. **`04_external_validation_GSE21653.ipynb`** — **PART A ONLY** (Cells 1–9)
   * *Outputs:* `frozen_selector.pkl`, `frozen_scaler.pkl`, `frozen_model.pkl`, `frozen_label_encoder.pkl`, `selected_probe_ids.pkl`
4. **`05_genefu_intrinsic_labels_GSE21653_R.ipynb`** (R kernel)
   * *Outputs:* `gse21653_genefu_intrinsic_labels.csv` (PAM50 calls)
5. **`04_external_validation_GSE21653.ipynb`** — **PART B** (Cells 10–11)
   * *Action:* Programmatically loads the genefu CSV output and re-scores the frozen model against intrinsic labels.
6. **`03_feature_stability_analysis.ipynb`**
   * *Action:* Loads the frozen XGBoost model.
   * *Outputs:* `46gene_panel_stability_STRICT_top50.csv` (SHAP-ranked top-50 probes, mygene symbols)
7. **`reannotate_probes_R.ipynb`** (R kernel)
   * *Action:* Dynamically reads the stability CSV from Step 6.
   * *Outputs:* `model_B_top50_reannotated.csv` (Authoritative mapping via hgu133plus2.db). **This file feeds Companion Repo 2.**
8. **`06_biomarker_characterization.ipynb`**
   * *Action:* Loads frozen artifacts (from Step 3) + the top-50 CSV (from Step 6).
   * *Outputs:* Final SHAP plots, KEGG/GO/Reactome enrichment profiles, and expression heatmaps.

*Note: Steps 7 and 8 are parallel consumers of Step 6. They do not depend on one another.*

---

## 📓 Notebook Reference

| # | Notebook | Purpose | Input / Reads | Output / Writes | Depends On |
|---|---|---|---|---|---|
| 01 | `01_internal_validation_GSE45827.ipynb` | Internal accuracy estimate: 80/20 holdout (96.15%) + leak-free 5-fold CV inside a scikit-learn `Pipeline`. | GSE45827 (GEO) | — | None |
| 02 | `02_classifier_comparison.ipynb` | Compares RF, SVM, and XGBoost on raw CV accuracy. Documents why XGBoost is retained for SHAP interpretability. | GSE45827 (GEO) | — | None |
| 03 | `03_feature_stability_analysis.ipynb` | Re-derives the top-50 gene panel via SHAP on the frozen model. Runs 20 resampling folds to verify stability (e.g., CDCA5, CMC2). | `frozen_model.pkl` | `46gene_panel_stability_STRICT_top50.csv` | **04** |
| 04 | `04_external_validation_GSE21653.ipynb` | Part A: Fits/freezes production model on 130 samples. validates vs IHC labels. Part B: Re-scores vs genefu labels. | GSE45827, GSE21653, genefu CSV | `frozen_*.pkl` files, metrics CSV/JSON | **05** (Part B) |
| 05 | `05_genefu_intrinsic_labels_GSE21653_R.ipynb` | (R Kernel) Computes intrinsic PAM50 calls for GSE21653 cohort as cleaner ground truth than IHC surrogates. | GSE21653 (GEO) | `gse21653_genefu_intrinsic_labels.csv` | None |
| 06 | `06_biomarker_characterization.ipynb` | SHAP mapping, KEGG/GO/Reactome enrichment, and expression heatmaps using the frozen model. | `frozen_*.pkl`, stability CSV | Enrichment CSVs, Figures | **04, 03** |
| — | `reannotate_probes_R.ipynb` | (R Kernel) Maps top-50 probes against `hgu133plus2.db` (manufacturer-curated) resolving to 43 unique authoritative symbols. | Stability CSV | `model_B_top50_reannotated.csv` | **03** |

---

## 📈 Results Highlights

**Classifier Performance:**
* **5-Fold CV Accuracy (XGBoost):** 93.1% ± 5.1%
* **ROC-AUC (Internal):** Basal (1.000), HER2 (0.999), Luminal A (0.987), Luminal B (0.981)
* **External Validation (GSE21653):** 63.7% vs. IHC-surrogate labels | 77.1% vs. genefu-intrinsic labels

**Feature Stability (Strict-Threshold, 20 Runs):**
* IL23A: 95% (19/20)
* AQP5: 90% (18/20)
* CDCA5: 50% (10/20)
* CMC2: 20% (4/20)

**Generated Visualizations:**
* `figures/PCA_subtypes.png` — PCA demonstrating pre-modeling subtype separability
* `figures/biomarker_heatmap.png` — Expression gradients of top biomarker genes across subtypes
* `figures/ROC_CV_curves.png` — Multi-class ROC bounds across cross-validation folds
* `figures/enrichment_dotplot.png` — Biological pathway enrichment mappings

---

## 🔗 Companion Repository (Breast Cancer Survival Biomarkers)

For the downstream clinical evaluation of our gene panel, please see the **[Companion Repository](https://github.com/Zohaib-Bioinfo/breast-cancer-survival-biomarkers)**.

**Reproducibility Hand-off:**
The output from this repository (`model_B_top50_reannotated.csv` generated by step 7) acts as the starting point for Repo 2. That CSV is loaded programmatically into `01_metabric_prognostic_validation.ipynb` to construct multivariate Cox proportional-hazards models and Kaplan-Meier stratification analyses over a 1,608-patient METABRIC cohort.

---

## 🛠 Setup & Environment

**Requirements:**
* **Python environment:** Requires `GEOparse`, `xgboost`, `shap`, `gseapy`, `mygene`, `scikit-learn`.
* **R environment:** Requires `genefu`, `hgu133plus2.db`, `AnnotationDbi` (can be run via Colab's R kernel or local RStudio).
* **Storage:** Set the `ARTIFACT_DIR` path at the top of notebooks to ensure frozen models and intermediary CSVs persist properly (e.g., Google Drive for Colab execution).

```bash
git clone [https://github.com/Zohaib-Bioinfo/breast-cancer-subtype-classification.git](https://github.com/Zohaib-Bioinfo/breast-cancer-subtype-classification.git)
cd breast-cancer-subtype-classification
pip install -r requirements.txt
```

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
