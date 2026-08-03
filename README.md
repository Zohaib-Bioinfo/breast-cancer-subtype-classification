# Breast Cancer Subtype Classification — Pipeline README
### (Repo 1 of 2 — see [Companion Repository](#companion-repository-breast-cancer-survival-biomarkers) below)
[breast-cancer-subtype-classification](https://github.com/Zohaib-Bioinfo/breast-cancer-subtype-classification)

This repo answers two questions: (1) can a small XGBoost model trained on
GSE45827 classify PAM50 molecular subtype from expression data, and
(2) which genes drive that classification, and are they biologically
coherent (pathway enrichment) and externally reproducible (GSE21653)?

Its output — a reannotated biomarker gene panel — is also the direct
input to a **second, separate repo** (`breast-cancer-survival-biomarkers`)
that tests whether those same genes carry independent prognostic value in
METABRIC. See the bottom of this file for that hand-off.

**Read this before running anything:** the notebook numbers describe a
*reading* order, not the *execution* order. Several notebooks depend on
artifacts produced by higher-numbered notebooks. See
[Execution Order](#execution-order) below — do not just run 01→02→...→06
top to bottom, it will fail partway through.

> ⚠️ **Reproducibility risk — three manual copy-paste hand-offs.**
> There is no automated file-passing at three points in this pipeline:
> `05` → `04` (genefu labels), `03` → `reannotate_probes_R` (probe list),
> and `reannotate_probes_R` → **Companion Repo 2** (final gene panel).
> Each is a human retyping/copy-pasting a list between notebooks. A typo
> at any of these three points silently changes the reported gene panel
> downstream with no error raised. See
> [Known rough edges](#known-rough-edges) for specifics, and the mirrored
> note in Companion Repo 2's README, which is the last of the three
> hand-offs.

---

## Dataset

- **Source:** [GSE45827](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE45827) (Gene Expression Omnibus) — discovery cohort
- **Platform:** GPL570 (Affymetrix Human Genome U133 Plus 2.0 Array)
- **Samples used:** 130 primary tumor samples (Basal: 41, HER2: 30, Luminal A: 29, Luminal B: 30)
- **External validation:** [GSE21653](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE21653) — independent, platform-matched (GPL570) cohort, n=266

## Execution order

Run in **this** order, not filename order:

```
1. 01_internal_validation_GSE45827.ipynb        (independent)
2. 02_classifier_comparison.ipynb               (independent)
3. 04_external_validation_GSE21653.ipynb  — PART A ONLY (Cells 1-9)
      creates: frozen_selector.pkl, frozen_scaler.pkl, frozen_model.pkl,
               frozen_label_encoder.pkl, selected_probe_ids.pkl
4. 05_genefu_intrinsic_labels_GSE21653_R.ipynb  (R kernel)
      creates: genefu-derived PAM50 calls for the GSE21653 cohort
5. 04_external_validation_GSE21653.ipynb  — PART B (Cells 10-11)
      paste in the Cell-4 output as the hardcoded lookup table,
      re-score against genefu labels instead of IHC-surrogate labels
6. 03_feature_stability_analysis.ipynb
      loads frozen_model.pkl from step 3
      creates: reference_46gene_panel.csv (SHAP-ranked, top-50 probe list,
               gene symbols via mygene "reporter" scope)
7. reannotate_probes_R.ipynb  (R kernel)
      paste in the top-50 probe list from step 6
      creates: model_B_top50_reannotated.csv (authoritative gene symbols
               via hgu133plus2.db, replacing the mygene lookup for those
               50 probes)
      ── this file's gene list is what feeds Companion Repo 2 ──
8. 06_biomarker_characterization.ipynb
      loads frozen_*.pkl from step 3 + reference_46gene_panel.csv from
      step 6
      creates: SHAP plot, KEGG/GO/Reactome enrichment, heatmap
```

Note: step 7 (`reannotate_probes_R.ipynb`) and step 8 (`06`) are
**parallel consumers** of step 6's output, not sequential — 06 does not
depend on 7. See [Known rough edges](#known-rough-edges) for a
consistency note between them.

---

## Dependency graph

```
01_internal_validation_GSE45827.ipynb         02_classifier_comparison.ipynb
   (standalone — reports CV/holdout              (standalone — justifies
    accuracy only, no saved artifacts)             XGBoost choice over RF/SVM)

04_external_validation_GSE21653.ipynb  (Part A)
   │
   ├──creates──> frozen_selector.pkl
   ├──creates──> frozen_scaler.pkl
   ├──creates──> frozen_model.pkl
   ├──creates──> frozen_label_encoder.pkl
   └──creates──> selected_probe_ids.pkl
   │
   ├─────────────────────────────────────────────┐
   │                                              │
   ▼                                              ▼
05_genefu_intrinsic_labels_GSE21653_R.ipynb   03_feature_stability_analysis.ipynb
   │  (R kernel; computes genefu PAM50             │  loads frozen_model.pkl
   │   calls for GSE21653 samples)                 │  (SHAP-based re-derivation,
   │                                                │   20x resampling stability
   ▼                                                │   check)
04_external_validation_GSE21653.ipynb (Part B)      │
   │  paste genefu output in, re-score              ▼
   │  IHC-surrogate accuracy (63.7%)          reference_46gene_panel.csv
   └─ → genefu-relabeled accuracy (77.1%)     (top-50 probes, mygene symbols)
                                                     │
                                       ┌─────────────┴─────────────┐
                                       ▼                           ▼
                       06_biomarker_characterization.ipynb   reannotate_probes_R.ipynb
                          loads frozen_model.pkl + panel.csv    (R kernel; hgu133plus2.db
                          → SHAP (labeled) / KEGG / GO /         reannotation, 43 unique
                            Reactome / heatmap                   symbols)
                                                                  │
                                                                  ▼
                                                    model_B_top50_reannotated.csv
                                                                  │
                                            ════════ REPO BOUNDARY ════════
                                                                  │
                                                                  ▼
                                          COMPANION REPO 2: breast-cancer-survival-biomarkers
                                          01_metabric_prognostic_validation.ipynb
                                             Cell 5: biomarker_genes = [...43 symbols...]
                                             → METABRIC Cox/KM prognostic validation
```

**Why 03 and 06 are numbered lower/higher than what they depend on:**
03 and 06 are downstream of the frozen model created in 04. The numbering
reflects manuscript section order (feature stability is discussed before
external validation in the write-up), not code dependency order. This is
intentional but easy to trip over — hence this README.

---

## Notebook reference

| # | Notebook | Purpose | Reads | Writes | Depends on |
|---|---|---|---|---|---|
| 01 | `01_internal_validation_GSE45827.ipynb` | Honest internal accuracy estimate: 80/20 holdout (96.15%) + leak-free 5-fold CV inside a `Pipeline` (**93.08% ± 5.10%**) | GSE45827 (GEO) | — (prints metrics only) | none |
| 02 | `02_classifier_comparison.ipynb` | Compares RF / SVM / XGBoost on raw CV accuracy; documents why XGBoost was retained despite RF/SVM scoring higher on raw accuracy (SHAP interpretability) | GSE45827 (GEO) | — (prints comparison table) | none |
| 03 | `03_feature_stability_analysis.ipynb` | Re-derives the top-50/43-gene panel via SHAP **on the frozen model**, then runs 20 resampling folds to check gene selection frequency (e.g. CDCA5, CMC2). Gene symbols via `mygene` | `frozen_model.pkl` (from 04) | `reference_46gene_panel.csv` | **04** |
| 04 | `04_external_validation_GSE21653.ipynb` | Part A: fits + freezes the production model on all 130 GSE45827 samples, validates on GSE21653 against IHC-surrogate labels (63.7%). Part B: re-scores against genefu-derived labels (77.1%) | GSE45827, GSE21653 (GEO), genefu output (from 05) | `frozen_*.pkl` × 5, prediction/metrics CSV+JSON | **05** (Part B only) |
| 05 | `05_genefu_intrinsic_labels_GSE21653_R.ipynb` | R notebook; computes genefu-package intrinsic PAM50 calls for the GSE21653 cohort as a cleaner ground truth than IHC-surrogate labels | GSE21653 (GEO), R `genefu` package | genefu label table (pasted manually into 04 Part B) | none |
| 06 | `06_biomarker_characterization.ipynb` | SHAP (properly labeled, on the frozen model) + KEGG/GO/Reactome enrichment + expression heatmap, all sourced only from the frozen model + panel — no refitting | `frozen_*.pkl` (from 04), `reference_46gene_panel.csv` (from 03) | SHAP/enrichment/heatmap figures, `enrichment_results_all.csv` | **04, 03** |
| — | `reannotate_probes_R.ipynb` | R notebook; re-annotates the top-50 panel probes against `hgu133plus2.db` (manufacturer-curated) instead of `mygene`'s lossier "reporter" scope. 46/50 probes resolve to 43 unique symbols; 4 probes remain unmapped even with the curated DB (`215593_at`, `229150_at`, `242580_at`, `233445_at`). **Its output is the actual gene panel used in Companion Repo 2**, not an internal dependency of `06` | Top-50 probe list (pasted manually from 03) | `model_B_top50_reannotated.csv` | **03** |

---

## Companion Repository: `breast-cancer-survival-biomarkers`

`reannotate_probes_R.ipynb`'s 43-symbol output is pasted directly into
**`01_metabric_prognostic_validation.ipynb`** (Cell 5, `biomarker_genes`
list) in the separate `breast-cancer-survival-biomarkers` repo, which:

1. Loads METABRIC clinical data (`data_clinical_patient.txt`,
   `data_clinical_sample.txt`), filters to the 4 matching subtypes →
   **1,608-patient cohort**
2. Pulls z-scored mRNA expression for the 43 panel genes via the
   cBioPortal API — **42 of 43 resolve** to Entrez IDs in cBioPortal
3. Fits two Cox proportional-hazards models: clinical-only (age, lymph
   nodes, NPI, subtype) vs. clinical + biomarker genes — **41 genes**
   enter the final model (some further drop from the Cox design matrix)
4. Reports C-index improvement: **0.6567 → 0.6752** with the gene panel
   added
5. Identifies CDCA5, IL23A, and AQP5 as independently significant
   (p < 0.05) in the multivariate model; CDCA5 is used for the
   Kaplan-Meier stratification figure (log-rank p = 1.69×10⁻⁹)

**The 43 → 42 → 41 gene count reduction across these two repos is
expected, not a bug** — it reflects (a) one symbol not present in
cBioPortal's METABRIC gene set, and (b) further attrition when building
the Cox design matrix. Worth stating explicitly in the manuscript Methods
so a reviewer doesn't read the changing gene count as an inconsistency.

---

## Known rough edges

1. **05 → 04 genefu hand-off is a manual copy-paste**, not a file read. `05` prints a lookup table; that table gets pasted into a hardcoded cell in `04` Part B. If revisited, `05` should write a CSV and `04` should read it.

2. **03 → reannotate_probes_R probe-list hand-off is also a manual copy-paste.** Same fix applies.

3. **reannotate_probes_R → Companion Repo 2 hand-off is a third manual copy-paste** (the 43-symbol list retyped into Cell 5 of `01_metabric_prognostic_validation.ipynb`). Three manual hand-offs in the full pipeline is the main reproducibility risk in this project — a single typo anywhere in this chain silently changes the reported gene panel downstream with no error thrown.

4. **Optional, not required:** `06`'s Cell 6 still falls back to `mygene` (`"reporter"` scope) for the ~250 probes outside the top-50, since `reannotate_probes_R.ipynb` was written for the Companion Repo 2 panel, not for `06`. If you want full consistency within Repo 1 itself, `06` could optionally merge `model_B_top50_reannotated.csv` in for its top-50 labels too — but this is a nice-to-have, not something the actual manuscript pipeline depends on.

---

## Environment

- Google Colab (all `0X_*.ipynb` notebooks except `05` and `reannotate_probes_R.ipynb`, which need an R kernel — either Colab's R runtime or local RStudio)
- Frozen artifacts and intermediate CSVs are persisted to Google Drive between sessions; set `ARTIFACT_DIR` at the top of each notebook to the correct Drive path before running
- Key packages: `GEOparse`, `xgboost`, `shap`, `gseapy`, `mygene`, `scikit-learn`, R `genefu`, R `hgu133plus2.db`/`AnnotationDbi`
- Companion repo additionally needs: `requests` (cBioPortal API), `lifelines`, `statsmodels`

## Manuscript mapping

Model architecture (`SelectKBest k=300 → StandardScaler → XGBoost,
n_estimators=100, max_depth=6, learning_rate=0.3`) corresponds to Methods
2.2 / Supplementary Table S1. Internal CV accuracy reported in the
manuscript: **93.08% ± 5.10%** (from notebook 01).

## Results

| Metric | Value |
|---|---|
| 5-Fold CV Accuracy (XGBoost) | 93.1% ± 5.1% |
| ROC-AUC (Basal) | 1.000 |
| ROC-AUC (HER2) | 0.999 |
| ROC-AUC (Luminal A) | 0.987 |
| ROC-AUC (Luminal B) | 0.981 |
| External validation accuracy (vs. IHC-surrogate labels, GSE21653) | 63.7% |
| External validation accuracy (vs. genefu-intrinsic labels, GSE21653) | 77.1% |
| CDCA5 strict-threshold feature stability (20 runs) | 50% (10/20) |
| CMC2 strict-threshold feature stability (20 runs) | 20% (4/20) |
| AQP5 strict-threshold feature stability (20 runs) | 90% (18/20) |
| IL23A strict-threshold feature stability (20 runs) | 95% (19/20) |

### Visualizations

- `figures/PCA_subtypes.png` — PCA showing subtype separability
- `figures/biomarker_heatmap.png` — Expression heatmap of top biomarker genes
- `figures/ROC_CV_curves.png` — Cross-validated multi-class ROC curves
- `figures/enrichment_dotplot.png` — Pathway enrichment results
- `figures/confusion_matrix.png` — Classification confusion matrix

## Setup

```
git clone https://github.com/Zohaib-Bioinfo/breast-cancer-subtype-classification.git
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
