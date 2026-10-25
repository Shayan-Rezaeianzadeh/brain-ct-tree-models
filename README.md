# Brain CT prediction with tree models

This project compares XGBoost, Random Forest, Decision Tree and Gradient Boosting for predicting worsening on a second brain CT that leads to surgery. The outcome column is `second_ct_findings`, with `1` representing that outcome and `0` its absence.

The models use clinical variables and findings from the initial CT stored in a CSV. They do not process CT images directly. The precise timing of follow-up and the outcome adjudication rules are not documented in the supplied code.

## Code

[Tree Models.py](Tree%20Models.py) contains the analysis. It also exports feature importance and SHAP results. The script uses `# %%` sections and can be run from top to bottom or section by section in an editor.

## Setup and running

From the repository folder:

```bash
python -m venv .venv
```

Activate the environment on macOS/Linux:

```bash
source .venv/bin/activate
```

Or in Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Then install the packages and run the script:

```bash
python -m pip install -r requirements.txt
python "Tree Models.py"
```

Place the input file at `data/data1.csv`, or edit `file_path` in the script's **Paths** section. Edit `output_dir` there to change the destination. Paths are relative to the script's folder, so the script can also be launched from another working directory.

The input CSV is not included. The original environment was not recorded; `requirements.txt` lists dependencies rather than an exact reproduction environment. On macOS, XGBoost may also require the OpenMP runtime provided by `libomp`.

The default searches use five-fold cross-validation and up to 30 parameter combinations per model. These searches can take time. `n_jobs=-1` uses the available CPU cores.

## Required columns

| Group | Column names |
| --- | --- |
| Outcome | `second_ct_findings` |
| Continuous predictors | `age`, `first_gcs`, `time_to_first_CT` |
| Other predictors | `sex`, `vehicle_trauma`, `non_vehicle_trauma`, `epidural`, `subdural`, `contusion`, `skull_base_fx`, `IVH`, `SAH`, `SGH`, `linear_skull_fx`, `skull_fx`, `wound_findings`, `drugs`, `impaired_clothing_time` |

Column names are case-sensitive. Leading and trailing spaces in column names are removed. Use numeric `0`/`1` for the outcome. The script also maps `no`/`yes`, `false`/`true`, `negative`/`positive`, and `absent`/`present` to `0`/`1`. Missing or unmapped outcome values are dropped; other numeric values are rejected.

The CT finding columns used in feature engineering must be numeric, with `0` for absent and `1` for present. Text in those fields is coerced to missing. The code does not establish the category definitions for `sex`, `drugs`, `wound_findings` or `impaired_clothing_time`; use the coding from the original dataset.

The unit of `time_to_first_CT` has not been confirmed. The existing feature named `delayed_first_ct_6h` applies `time_to_first_CT > 6`; the `6h` name should not be interpreted as a confirmed six-hour threshold until the data dictionary is checked.

## Analysis

1. Read the CSV, check columns, prepare the binary outcome and convert numeric variables.
2. Add GCS groups (`≤ 8`, `9–12`, `≥ 13`), the `> 6` time indicator, the number of initial findings, any hemorrhage and any skull fracture.
3. One-hot encode the categorical variables and create a stratified 70%/30% train/test split with seed 42.
4. Fit median imputation and standard scaling inside each training fold. Apply SMOTE toward a positive-to-negative ratio of 0.75, then ADASYN toward 1.0 when possible. No oversampling is applied to the test set.
5. Tune each model using `RandomizedSearchCV`, five stratified folds and ROC AUC scoring.
6. Evaluate at 0.50 and at a Youden's J cutoff selected from the fitted model's training predictions.
7. Save metrics, ROC tables and figures. The script also saves feature importance and SHAP results.

Reported metrics are ROC AUC, accuracy, sensitivity, specificity, PPV, NPV, F1 and confusion-matrix counts. ROC tables use 2,001 cutoffs between 1.001 and −0.001. AUC is calculated from the predicted probabilities with `roc_auc_score`, independently of that plotting grid.

## Outputs and saved experiments

New runs write to:

```text
outputs/
└── tree_models/
    ├── final_tree_models_smote_then_adasyn_shap.xlsx
    ├── tree_models_combined_test_roc_dense.png
    ├── best_tree_model_shap_summary.png
    └── individual ROC and feature-importance plots
```

The script exports reports, not saved trained-model files. Re-running them in the same output folder replaces the reports. Generated outputs and input data are excluded by `.gitignore`.

The supplied `2nd Brain CT prediction.zip` contains three saved experiment folders for tree models, with figures and an Excel workbook in each. Those result files are not included in this code repository. The three folders cover: an earlier run, a separate SMOTE/ADASYN comparison, and the final SMOTE-then-ADASYN run. Only the source for the final sequential sampling workflow was supplied. The earlier results are historical outputs, not runs that the current scripts reproduce.

Some original workbooks contain `Recovered_Sheet` tabs. The supplied export code truncated long tab names and could reuse a name for different tables. The cleaned script preserves the sheet suffixes to prevent those collisions in new exports. They cannot restore tables missing from earlier files.

## Interpretation

- Youden's J is selected from in-sample training predictions, not out-of-fold predictions or a separate validation set. This choice is retained from the original analysis.
- The script chooses the model for SHAP using test AUC. That is a descriptive comparison on this test set; it is not an independent evaluation of a model selected without looking at the test results.
- One-hot encoding is performed before splitting, so the category vocabulary includes the test rows. Median imputation, scaling and resampling are fitted within training folds.
- The original finding summaries use `sum(skipna=True)`. Missing findings therefore do not contribute to the count, and an entirely missing set of finding fields can become zero or an absent indicator.
- SMOTE and ADASYN interpolate numeric features, including dummy variables. Synthetic rows can contain fractional indicators. The sampling method has been preserved.

The CSV and original model fits are not available here, so the saved study results cannot be independently reproduced from the repository alone.
