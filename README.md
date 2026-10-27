# MIMIC-IV hosp-only hypoglycemia prediction

The current workflow uses **only MIMIC-IV v3.1 `hosp` tables**. Insulin exposure comes from `emar`; all baseline glucose, glucose outcomes and other laboratory predictors come from `labevents`. No ICU tables or ICU-derived features are used. Patients are not excluded merely because they received intensive care—the restriction concerns the data sources.

| Notebook | Purpose |
|---|---|
| [01_Preprocess_MIMIC_IV.ipynb](01_Preprocess_MIMIC_IV.ipynb) | Raw hosp tables → T2DM phenotype, non-overlapping insulin windows, labels, pre-exposure features and patient splits. |
| [02_Train_Hypoglycemia_Model.ipynb](02_Train_Hypoglycemia_Model.ipynb) | Hosp-only cohort → grouped validation, model fitting, calibration, evaluation, feature-group ablations and cohort sensitivities. |

Both notebooks are self-contained and commented; no project `.py` files are required. Shared copies have cleared outputs and contain no patient records or credentials. Earlier project scripts and outputs describe historical experiments; use these notebooks for the current hosp-only workflow.

## Google Colab setup

1. Upload the two notebooks to your team's Google Drive folder and open them in Colab.
2. Use a **CPU runtime**. Run the installation cell first and restart the runtime if requested.
3. Mount Drive and edit the configuration cell near the top. Each teammate uses the paths visible in their own mounted Drive.
4. Run notebook 1 from top to bottom, then notebook 2. If the current **hosp-only handoff folder** already exists, start with notebook 2.

Use authorized storage for MIMIC-IV data. Sharing the notebooks does not provide dataset access. See the [official Colab files and Drive guide](https://colab.research.google.com/notebooks/io.ipynb) for mounting instructions. Local Jupyter is also supported after changing the paths; Drive mounting is skipped outside Colab.

## Required raw data

Set `HOSP_DIR` directly to the folder containing these seven files:

```text
hosp/
├── patients.csv.gz
├── admissions.csv.gz
├── diagnoses_icd.csv.gz
├── d_icd_diagnoses.csv.gz
├── prescriptions.csv.gz
├── labevents.csv.gz
└── emar.csv.gz
```

Files remain compressed. The local source folder for this project is:

```text
C:\DUKE ACADEMICS\BME790_26Fall\dataset\physionet.org\files\mimiciv\3.1\hosp
```

| Setting | Default Colab path | Notebook |
|---|---|---|
| `HOSP_DIR` | `/content/drive/MyDrive/mimiciv/3.1/hosp` | 1 |
| `CACHE_DIR` | `/content/drive/MyDrive/BME790/cache_hosp_v1` | 1 |
| `READY_DIR` | `/content/drive/MyDrive/BME790/preprocessed_hosp` | 1 |
| `DATA_DIR` | Same folder as `READY_DIR` | 2 |
| `RESULTS_DIR` | `/content/drive/MyDrive/BME790/model_results_hosp` | 2 |

A shortcut to the team folder can replace these defaults. Avoid concurrent writes to the same cache or results directory.

**Rebuild preprocessing for this version.** The earlier mixed-source `primary_cohort.parquet` is incompatible, even if ICU columns are manually dropped: its exposure selection, baseline glucose and labels may depend on other sources. Notebook 2 requires the new `protocol.json`, checks the hosp-only contract and rejects mixed-source columns/exposures.

## Cohort definitions

- **Population:** adults with a retrospective, supported T2DM phenotype using ICD-9/10, non-insulin prescriptions and HbA1c. Exclude patients with any Type 1 code. This adapted phenotype is not expert-validated. Full-record supporting evidence may occur after prediction but never enters the predictors.
- **Exposure:** an eMAR record marked `Administered` with insulin in the medication name. Prescription orders do not establish exposure. Explicitly named hyperkalemia index administrations are excluded.
- **Prediction time:** max(actual administration, posting), allowing at most a 60-minute documentation delay.
- **Unit:** greedily selected, non-overlapping 24-hour windows per patient. Select before checking baseline, outcomes or follow-up. A failed window is not replaced by a more convenient dose inside it.
- **Baseline:** a laboratory glucose collected and available within six hours before actual administration; exclude any glucose below70 collected in the specified six-hour exclusion window. Require a full24h hospital follow-up.
- **Outcome:** any documented lab glucose below70 mg/dL in the following24h. Explicit mg/dL units or converted mmol/L are accepted; glucose units are not inferred from another table.
- **Monitoring:** for both positive and negative examples, require a reading in each six-hour bin and maximum gap≤8h. Collapse15-minute rechecks for monitoring while retaining any low. Insufficient monitoring is excluded, never treated as negative.
- **Predictors:** age, recorded gender, race/admission context, prior24h lab glucose summaries, other labs, diagnoses from earlier completed admissions and prior24h actual eMAR medication records. Collection and posting must both precede actual administration.

There are no ICU-derived vitals, weight, diet/NPO or feeding-volume predictors. Available hosp nutrition-related information remains through laboratory values, prior diagnosis groups and medication-class records.

Six exported cohorts cover primary monitoring, relaxed monitoring, strict monitoring, first-window-only admissions, broad-coded T2DM and prior-documented T2DM. They share the original exposure-window selection. Redundant lab-only and explicit-unit sensitivity copies have been removed because laboratory glucose is already the only source.

## Splits, checkpoints and reproducibility

All admissions/windows from a patient stay together. New assignments use an outcome-independent hash with approximately **60/15/10/15%** train/validation/calibration/test allocation.

`EXISTING_SPLITS` optionally accepts an earlier patient manifest (`.csv.gz` or `.parquet`). The original `outputs/primary/split_assignments.csv.gz` preserves historical patient allocations, but cannot reproduce historical performance after the data-source change. Without a supplied manifest, assignments are fresh and deterministic. Exported `patient_splits.parquet` lets teammates reuse allocations.

The `new_patient` flag means absent from the supplied earlier manifest. Notebook 2 reports that subset separately only when preprocessing metadata confirms a manifest was supplied. It does not establish external validation.

Large raw scans are streamed in chunks. Completed source passes and per-partition feature/label checkpoints persist in `CACHE_DIR`. Rerun with identical settings to resume. Choose a **new empty cache directory** after changing raw data or clinical definitions. Cache signatures distinguish this hosp-only protocol from earlier versions.

## Model and evaluation

The model uses **38 candidate hosp-derived clinical inputs**, followed by training-only coverage/rare-feature filtering, percentile clipping, imputation, scaling and categorical encoding. The transformed input count depends on training-data coverage and categories.

The starting histogram-gradient-boosting settings retain the previous regularization:100 trees, at most4 leaves, depth≤2, minimum leaf size300, L2=50 and learning rate0.05. **These settings must be reassessed for the new, potentially smaller hosp-only cohort.** Three-fold patient-grouped CV diagnoses the fixed architecture; the notebook does not repeat the earlier broad hyperparameter search. It warns when training folds are small relative to the leaf-size setting. Reassess model capacity using training-only CV, without tuning on test results.

Each patient receives equal total training weight across their windows. Preprocessing is fitted separately inside every CV fold. The final model fits training patients, probability calibration uses calibration patients, and the primary threshold targets at least80% validation sensitivity. Test sensitivity can differ.

Evaluation reports average precision (AP), AUROC, precision/PPV, sensitivity, specificity, alert fraction and Brier score. AP is a ranking summary, not precision at one threshold; random AP equals event prevalence. Confidence intervals resample whole patients.

Optional settings in notebook 2:

- `RUN_GROUP_ABLATION`: exploratory training-only comparisons of recent lab glucose, other labs, prior diagnoses/admissions, eMAR medications and demographics/context. Positive AP loss means removal hurts prediction; these are predictive associations.
- `RUN_SENSITIVITIES`: evaluate the frozen primary model and threshold on the six cohort definitions without retuning.
- `BOOTSTRAPS`: number of patient-bootstrap repetitions; default250.

## Outputs

Notebook 1 writes six `*_cohort.parquet` files, `patient_splits.parquet`, `cohort_counts.csv` and the required `protocol.json` to `READY_DIR`. Notebook 2 needs this handoff folder, not the raw data or cache.

Notebook 2 saves CV/performance tables, bootstrap intervals, operating points, a performance figure, optional ablation/sensitivity tables, `model.pkl` and `model_metadata.json` to `RESULTS_DIR`. Cloudpickle includes notebook-defined preprocessing/calibration classes in the model file; retain matching package versions and metadata when reusing it.

## Interpretation

Hospital laboratory monitoring can be less frequent than other glucose monitoring, so usable counts and event ascertainment change. Earlier mixed-source counts and model metrics are not results for this version. Missing monitoring can conceal transient lows, and retrospective phenotype/monitoring selection limits generalizability. The task predicts documented hypoglycemia after insulin; it does not establish that insulin caused the event.

## Local hosp-only verification

Both notebooks were executed locally. Preprocessing passed synthetic raw-table tests with no ICU folder present, including baseline availability, monitoring exclusions, Type 1 exclusion, labels and patient isolation. A real hosp-only cohort was then rebuilt using only the previously verified hosp lab/eMAR and hosp diagnosis/patient/prescription caches; earlier selected windows, derived features and labels were not reused. The model notebook was executed on that new handoff. These are local checks, not remote Colab execution.

The current v3.1 data produced:

| Cohort | Patients | Windows | Positive windows |
|---|---:|---:|---:|
| Primary monitoring | 590 | 1,227 | 34 |
| Relaxed monitoring | 3,422 | 8,100 | 240 |
| Strict monitoring | 64 | 68 | 7 |
| First window per admission | 205 | 208 | 7 |
| Broad-coded T2DM | 744 | 1,577 | 38 |
| Prior-documented T2DM | 372 | 792 | 27 |

The primary cohort has only **3 positive test windows**, so its model performance cannot support reliable conclusions. Relaxed monitoring is retained as a predefined sensitivity; it changes ascertainment and may miss more lows. It has not silently replaced the primary definition, and it should not be chosen by comparing test scores.

The locally generated hosp-only handoff and exploratory model outputs are saved in `outputs/hosp_only/preprocessed/` and `outputs/hosp_only/modeling/`. To start with notebook 2 in Colab, upload the complete hosp-only handoff folder, including `protocol.json`, and point `DATA_DIR` to it.
