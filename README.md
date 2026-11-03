# Risk Models

The `risk_models` package predicts the probability that a loan application will
default. It prepares application data, trains several machine-learning
pipelines, compares them, selects a champion model, chooses an operating
threshold based on business cost, and uses that threshold to create a
prioritized review queue.

This package is responsible for **risk prediction**, not final loan-policy
eligibility. The policy-oriented checks (age, maximum loan amount, collateral,
and unsecured credit score) are implemented in
[`../core/entities.py`](../core/entities.py). A model score is a risk signal;
it is not a replacement for those domain rules.

## End-to-end flow

```text
data/loan_applications_train.csv
data/loan_applications_predict.csv
             |
             v
data_access.cleaning_pipeline.ApplicationCleaner
             |
             +--> processed_data/loan_applications_cleaned.csv
             +--> processed_data/loan_applications_predict_cleaned.csv
             +--> output/rejected_records.csv
             |
             v
feature_engineering.py
             |
             +--> monthly_installment_inr
             +--> installment_to_income
             +--> credit_band
             |
             +--> processed_data/loan_applications_enriched.csv
             +--> processed_data/loan_applications_predict_enriched.csv
             |
             +----------------------------+
             |                            |
             v                            v
feature_pipeline.py                 champion_model.py
prepare numeric/categorical data    tune candidate models
             |                            |
             v                            +--> output/champion_search.json
baseline/tree/SVM pipelines              +--> artifacts/champion.pkl
                                          +--> output/threshold_choice.json
                                                   |
                                                   v
                                      work_queue.py
                                                   |
                                                   +--> output/work_queue.csv
                                                   +--> review summary
```

The normal training/promotion sequence is:

1. Clean raw application CSVs.
2. Add engineered features.
3. Split training data into a stratified training set and holdout set.
4. Compare candidate models with cross-validation.
5. Tune the selected champion candidates and save the best fitted pipeline.
6. Sweep default-probability thresholds using business costs.
7. Score waiting applications and classify each as `refer` or `approve`.

The prediction dataset must contain the same feature columns expected by the
fitted pipeline. The enriched prediction file is used by the work queue because
the queue displays the engineered values as well as the model score.

## Package files

### [`__init__.py`](./__init__.py)

This marks `risk_models` as a Python package. It does not currently define
runtime logic or public exports.

### [`feature_pipeline.py`](./feature_pipeline.py)

This module defines the shared preprocessing contract used by every model.
Keeping preprocessing inside a scikit-learn `Pipeline` prevents training and
prediction from applying different transformations.

#### `build_feature_pipeline(extra_numeric=(), extra_categorical=())`

Builds an **unfitted** `ColumnTransformer` with two branches:

```text
numeric columns
    -> median imputation
    -> StandardScaler

categorical columns
    -> most-frequent imputation
    -> OneHotEncoder(handle_unknown="ignore")
```

The base columns come from
[`config/constants.py`](../../config/constants.py):

- Numeric: age, income, credit score, debt-to-income ratio, existing loans,
  late payments, loan amount, and tenure.
- Categorical: employment type and loan type.

`extra_numeric` and `extra_categorical` allow enriched features to be added
without duplicating the preprocessing implementation. `sparse_threshold=0`
ensures that the transformed result is dense.

#### `select_modelling_columns(records, extra_numeric=(), extra_categorical=())`

Selects only the configured model columns that are present in the DataFrame.
It returns numeric columns followed by categorical columns. This protects the
model from accidentally receiving identifiers, names, dates, or other
non-model fields.

#### `split_for_modelling(cleaned_file_path, extra_numeric=(), extra_categorical=())`

Reads a cleaned CSV, selects the features, extracts the target column
`defaulted`, and uses `train_test_split` to create:

```text
X_train, X_holdout, y_train, y_holdout
```

The split is:

- stratified, so the default/non-default ratio is preserved;
- 20% holdout, from `HOLDOUT_SHARE`;
- reproducible, using `RANDOM_SEED`.

The holdout set is reserved for an evaluation that is separate from model
fitting.

### [`baseline_model.py`](./baseline_model.py)

This module provides the logistic-regression baseline and common model
persistence/prediction helpers.

#### `_patch_sklearn_object(obj)`

Recursively repairs compatibility attributes on objects loaded from joblib.
This is intended for scikit-learn version differences, especially missing
`SimpleImputer` attributes. It can walk:

- dictionaries of models;
- pipeline steps;
- fitted or unfitted `ColumnTransformer` branches;
- imputers with `statistics_`.

It is an internal compatibility helper, not a model-training step.

#### `build_baseline_model()`

Builds an unfitted pipeline:

```text
feature preprocessing
        |
        v
LogisticRegression(
    class_weight="balanced",
    max_iter=1000,
    random_state=RANDOM_SEED
)
```

The balanced class weight gives additional importance to the less common
default class.

#### `train_baseline_model(cleaned_file_path)`

Splits the cleaned training data, fits the baseline, predicts default
probabilities on the holdout set, and returns:

```text
(fitted_model, holdout ROC-AUC)
```

ROC-AUC measures how well the model ranks defaulters above non-defaulters
across possible thresholds.

#### `save_model(model, model_path)` and `load_model(model_path)`

`save_model` creates the destination directory, applies the compatibility
patch, and serializes the fitted pipeline with `joblib`.

`load_model` deserializes a pipeline and applies the same compatibility patch.
The usual saved model paths are:

```text
artifacts/baseline_model.pkl
artifacts/champion.pkl
```

#### `predict_default_probability(model, records)`

Returns one default probability per input row using `predict_proba()[:, 1]`.
It also inspects a fitted pipeline's preprocessing columns so a model trained
with extra engineered columns can select those same columns at prediction
time.

#### `write_predictions(model, prediction_file_path, output_path)`

Reads a prediction CSV, scores it, and writes a small output CSV containing:

```text
application_id, default_probability
```

The example writes `output/predictions.csv`.

### [`tree_models.py`](./tree_models.py)

This module defines tree-based candidate models and tree-specific diagnostics.

#### `build_decision_tree()`

Builds a preprocessing pipeline followed by a balanced
`DecisionTreeClassifier` configured with:

- `max_depth=6`
- `min_samples_leaf=20`
- `random_state=RANDOM_SEED`

The depth and minimum leaf size constrain overfitting.

#### `build_random_forest(extra_numeric=(), extra_categorical=())`

Builds a preprocessing pipeline followed by a balanced
`RandomForestClassifier` configured with:

- 100 trees;
- maximum depth 8;
- minimum leaf size 20;
- the shared random seed.

The optional extra columns are passed through to
`build_feature_pipeline`, which is how engineered features can be included.

#### `train_and_score(model, cleaned_file_path)`

Fits any supplied model on the training split, predicts probabilities on the
holdout split, and returns:

```text
(fitted_model, holdout ROC-AUC)
```

#### `feature_importances(fitted_model)`

Reads transformed feature names from the fitted preprocessor and pairs them
with the classifier's tree feature importances. It returns a DataFrame sorted
from most to least important.

#### `compare_class_weighting(cleaned_file_path, output_path)`

Trains two random forests:

1. `class_weight="balanced"`
2. `class_weight=None`

It compares holdout ROC-AUC and default recall (`defaulters_caught`) and writes
the result to `output/class_weight_comparison.json`.

### [`cross_validation.py`](./cross_validation.py)

This module compares several model families using repeated stratified folds.

#### `build_linear_svm()`

Builds a preprocessing pipeline followed by `LinearSVC` with:

- `C=0.1`;
- balanced class weighting;
- the shared random seed;
- a maximum of 3,000 iterations.

`LinearSVC` does not provide `predict_proba`, so its decision function is used
as the ranking score during evaluation.

#### `every_model()`

Returns fresh, unfitted candidates:

```text
logistic_baseline
decision_tree
random_forest
linear_svm
```

Fresh instances are important because each fold must fit an independent model.

#### `score_across_folds(model, features, target)`

Uses `StratifiedKFold` with five shuffled folds. For each fold it:

1. clones the supplied model;
2. fits the clone on the fold's training rows;
3. obtains probabilities or decision scores;
4. calculates ROC-AUC on the validation rows.

It returns:

```python
{
    "fold_scores": [...],
    "mean_roc_auc": ...,
    "spread": ...
}
```

`spread` is the maximum fold score minus the minimum fold score and indicates
how stable the result is across folds.

#### `training_against_holdout(model, cleaned_file_path)`

Fits a model on the training portion and reports both training ROC-AUC and
holdout ROC-AUC. A large difference can indicate overfitting.

#### `compare_across_folds(cleaned_file_path, output_path)`

Builds all four candidates, evaluates them using the training portion, and
writes the fold results to `output/fold_scores.json`.

The module's command-line example prints each model's fold range, mean,
spread, training score, and holdout score, then identifies the best mean
cross-validation model. This comparison is diagnostic; champion promotion is
implemented separately in `champion_model.py`.

### [`feature_engineering.py`](./feature_engineering.py)

This module creates derived application features and measures whether they
improve a random forest.

#### `add_engineered_features(records)`

Copies the input DataFrame and adds:

1. `monthly_installment_inr`
   - looks up the annual rate for each loan type;
   - converts the loan amount and tenure to numeric values;
   - calls
     [`core.loan_calculations.calculate_monthly_installment`](../core/loan_calculations.py);
   - rounds the result to two decimal places.
2. `installment_to_income`
   - divides the calculated instalment by monthly income;
   - rounds to four decimal places.
3. `credit_band`
   - places the credit score into the configured `poor`, `fair`, `good`, or
     `excellent` interval using `pandas.cut`.

The original DataFrame is not modified.

#### `write_enriched_file(cleaned_file_path, output_path)`

Reads a cleaned CSV, adds the three derived features, and writes an enriched
CSV. The normal outputs are:

```text
processed_data/loan_applications_enriched.csv
processed_data/loan_applications_predict_enriched.csv
```

#### `default_rate_by_band(enriched_file_path)`

Groups training rows by credit band and calculates the mean of `defaulted`.
Because `defaulted` is treated as 0/1, the mean is the default rate. Missing
bands are reported as `0.0`.

#### `measure_the_lift(enriched_file_path, output_path)`

Evaluates a random forest twice:

1. with the base modeling columns;
2. with the three engineered columns.

It writes both mean ROC-AUC values and their difference (`lift`) to
`output/feature_lift.json`.

### [`champion_model.py`](./champion_model.py)

This module performs the promotion decision and chooses a business operating
threshold.

#### `candidate_grids()`

Defines the hyperparameter searches:

- random forest: `max_depth` 6 or 8, `min_samples_leaf` 10 or 20;
- logistic baseline: `C` 0.01, 0.1, or 1.0.

Each candidate is a complete preprocessing-plus-classifier pipeline.

#### `search_for_the_champion(cleaned_file_path, output_path)`

Uses `GridSearchCV` with five stratified folds and ROC-AUC scoring. For each
candidate it:

1. searches the configured grid;
2. refits the best setting;
3. records its best score and settings.

The candidate with the greatest mean ROC-AUC becomes the champion. The
function writes the comparison to `output/champion_search.json` and returns:

```text
(champion_name, candidates_info)
```

`candidates_info` contains the fitted best estimator, settings, and score for
each candidate.

#### `cost_of_a_threshold(default_scores, actual_outcomes, threshold)`

Treats a probability threshold as an operational decision boundary:

```text
score < threshold  -> approve
score >= threshold -> refer
```

It counts:

- missed defaults: actual default but score below the threshold;
- refused good applicants: actual non-default but score at or above the
  threshold.

It prices those errors with:

- `COST_OF_A_MISSED_DEFAULT_INR`;
- `COST_OF_A_REFUSED_GOOD_APPLICANT_INR`.

It returns the counts and total cost.

#### `choose_threshold_on_cost(champion, cleaned_file_path, output_path)`

Scores the holdout set with the champion and evaluates thresholds from 0.05
through 0.95 in increments of 0.05. It selects the cheapest threshold and
also records the cost of the habitual 0.50 threshold.

The result is written to `output/threshold_choice.json`. The threshold is
chosen on business cost, not simply assumed to be 0.50.

The command-line example:

1. searches for the champion;
2. saves it as `artifacts/champion.pkl`;
3. chooses and saves the threshold.

### [`work_queue.py`](./work_queue.py)

This module applies the promoted model to new applications and creates an
officer-facing queue.

#### `score_waiting_applications(champion, waiting_file_path)`

Reads the enriched prediction file and calls
`baseline_model.predict_default_probability`. It returns a DataFrame with:

```text
application_id
default_probability
loan_amount_inr
monthly_installment_inr
installment_to_income
credit_band
```

#### `build_work_queue(champion, waiting_file_path, threshold_path, output_path)`

Reads the chosen threshold and assigns:

```text
default_probability >= threshold -> refer
default_probability < threshold  -> approve
```

It then sorts applications from highest to lowest default probability and
writes `output/work_queue.csv`.

`refer` means the application should receive additional human/risk review.
`approve` means it is below this model-based referral threshold; this does not
override separate policy validation.

#### `queue_summary(queue_path)`

Reads a generated queue and calculates:

- total waiting applications;
- referred count;
- approved count;
- referred loan exposure;
- referred exposure as a share of total exposure.

The command-line example loads `artifacts/champion.pkl`, scores
`processed_data/loan_applications_predict_enriched.csv`, writes the queue, and
prints its summary and riskiest first row.

## Inputs, outputs, and ownership

| Stage | Reads | Writes | Main owner |
|---|---|---|---|
| Raw cleaning | `data/loan_applications_train.csv`, `data/loan_applications_predict.csv` | `processed_data/*_cleaned.csv`, `output/rejected_records.csv` | `data_access/cleaning_pipeline.py` |
| Feature enrichment | `processed_data/*_cleaned.csv` | `processed_data/*_enriched.csv`, `output/feature_lift.json` | `feature_engineering.py` |
| Baseline training | cleaned training CSV | `artifacts/baseline_model.pkl`, `output/predictions.csv` | `baseline_model.py` |
| Tree training | cleaned training CSV | `artifacts/decision_tree.pkl`, `artifacts/random_forest.pkl`, diagnostics | `tree_models.py` |
| Fold comparison | cleaned training CSV | `output/fold_scores.json` | `cross_validation.py` |
| Champion search | cleaned training CSV | `output/champion_search.json`, `artifacts/champion.pkl`, `output/threshold_choice.json` | `champion_model.py` |
| Waiting decisions | enriched prediction CSV, threshold JSON, champion pickle | `output/work_queue.csv` | `work_queue.py` |

Model pickle files are binary artifacts and should be loaded only from trusted
locations because joblib/pickle deserialization can execute code.

## Connections to other project folders

### `data_access`

[`data_access/cleaning_pipeline.py`](../data_access/cleaning_pipeline.py)
creates the cleaned CSVs consumed by this package. The cleaning stage removes
invalid rows and normalizes dates, text, and numeric types before model
training.

[`data_access/database_loader.py`](../data_access/database_loader.py) can load
the cleaned application CSV into SQLite. The database is useful for operational
queries and reports; the risk-model training code reads CSV files directly.

[`data_access/file_storage.py`](../data_access/file_storage.py) loads CSV rows
as dictionaries and converts them into domain objects through
[`core.entities.create_loan_application`](../core/entities.py). That is the
operational loan-rule path, separate from the statistical risk-model path.

### `core`

[`core/loan_calculations.py`](../core/loan_calculations.py) supplies the
monthly-installment formula used by `feature_engineering.py`. This gives the
model feature and the application/business layer the same EMI calculation.

[`core/entities.py`](../core/entities.py) applies deterministic eligibility
rules to a loan object. The risk model estimates default probability; it does
not replace `LoanApplication.is_eligible()`.

[`core.decorators.py`](../core/decorators.py) protects the shared installment
calculation from negative positional numeric arguments.

### `config`

[`config/constants.py`](../../config/constants.py) is the configuration
contract for this package. It supplies:

- model input column lists;
- target column `defaulted`;
- engineered column names;
- random seed;
- holdout and fold counts;
- error costs;
- credit-score band boundaries;
- model artifact paths.

### `artifacts`

The trained pipelines are saved as:

```text
artifacts/baseline_model.pkl
artifacts/decision_tree.pkl
artifacts/random_forest.pkl
artifacts/champion.pkl
```

### `output`

Model diagnostics and operational outputs are written here:

```text
output/fold_scores.json
output/champion_search.json
output/threshold_choice.json
output/feature_lift.json
output/class_weight_comparison.json
output/predictions.csv
output/work_queue.csv
```

## Running the stages

Run commands from the repository root:

```powershell
python -m loanserve.data_access.cleaning_pipeline
python -m loanserve.risk_models.feature_engineering
python -m loanserve.risk_models.baseline_model
python -m loanserve.risk_models.tree_models
python -m loanserve.risk_models.cross_validation
python -m loanserve.risk_models.champion_model
python -m loanserve.risk_models.work_queue
```

The commands have dependencies. In particular:

1. cleaning must produce the cleaned CSVs;
2. feature engineering must produce the enriched prediction CSV before the
   work queue is built;
3. champion selection must produce both `champion.pkl` and
   `threshold_choice.json` before the work queue is built.

For a complete normal run, execute:

```powershell
python -m loanserve.data_access.cleaning_pipeline
python -m loanserve.risk_models.feature_engineering
python -m loanserve.risk_models.champion_model
python -m loanserve.risk_models.work_queue
```

The baseline, tree, and cross-validation modules are useful diagnostics and
comparisons; they are not prerequisites for creating the champion when
`champion_model.py` performs its own candidate search.

## Important distinction: policy decision vs model decision

There are two different decisions in the project:

1. **Deterministic product eligibility**
   - handled by `core.entities`;
   - checks product rules such as age, amount, DTI, collateral/LTV, and
     unsecured credit score.
2. **Statistical default-risk referral**
   - handled by this package;
   - estimates default probability and compares it with a cost-selected
     threshold.

A production workflow should use both results. A model-generated `approve`
decision is not evidence that the application satisfies every deterministic
loan policy, and a `refer` decision is not itself a rejection.
