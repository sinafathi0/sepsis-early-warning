# Early Sepsis Warning in ICU Patients

Predicting sepsis hours before clinical onset from hourly ICU measurements, using the PhysioNet/Computing in Cardiology Challenge 2019 data (two hospital systems, 40,336 patients, 1.55 M patient-hours).

## Results (held-out test set, evaluated once)

An alert is raised when the predicted risk stays above 0.1 for 2 consecutive hours. Both values were chosen on the validation set only.

| Metric | Test |
|---|---|
| AUROC | 0.828 |
| AUPRC | 0.104 (about 1.8% of hours are positive) |
| Septic patients alerted before onset | 326 / 440 (74%) |
| Healthy patients with at least one false alarm | 1,686 / 5,610 (30%) |
| False-alarm hours among healthy patients | 9.7% |

**Generalisation to an unseen hospital** (test on hospital B):

| Trained on | AUROC | AUPRC | Septic alerted before onset | Healthy with false alarm |
|---|---|---|---|---|
| Hospital A only | 0.749 | 0.057 | 86 / 166 | 719 / 2,797 |
| Hospitals A+B | 0.840 | 0.104 | 113 / 166 | 598 / 2,797 |

A model trained on one hospital loses a lot on the other, because the hospitals record data differently (see the EDA).

## Approach

1. **Load** the per-patient `.psv` files into one table (`patient_id`, `hospital` added).
2. **EDA**: sepsis is rare (7.3% of patients, 1.8% of hours), most labs are >90% missing, and the two hospitals differ in prevalence and recording practice. `ICULOS` is tied to the outcome by how the dataset was assembled, so it is not used as an input.
3. **Preprocessing**: patient-level 70/15/15 split (stratified), implausible values set to NaN, `*_measured` flags, forward-fill within patient, training-median fill, and features that only use the past (6 h rolling mean and difference, hours since last measurement, measurement counts).
4. **Model**: LightGBM with early stopping on average precision, patient-level evaluation and an alert rule that needs `k` consecutive high-risk hours.

## Repository layout

```
notebooks/         01_load_data, 02_eda, 03_preprocessing, 04_modeling
data/raw/          put training_setA/ and training_setB/ here (not included)
data/processed/    pickles written by the notebooks (not committed)
requirements.txt
```

## How to run

1. Download the training data from the [PhysioNet challenge page](https://physionet.org/content/challenge-2019/1.0.0/) and copy the folders `training_setA` and `training_setB` into `data/raw/` (sub-folders are fine).
2. `pip install -r requirements.txt` (Python 3.10+).
3. Run the notebooks in `notebooks/` in order. Loading and feature building take several minutes, and the processed pickles need a few GB of disk and RAM.

## Limitations

- "Alerted before onset" counts any earlier alert, even days earlier; no lead-time window is applied.
- Threshold and `k` are a trade-off, not tuned for a clinical target; no confidence intervals or calibration.
- Row-level precision is low: 74% patient coverage costs many false alarms.
- A single random split of one public dataset, not a prospective evaluation.

## Data and reference

The data is not redistributed here. Reyna et al., *Early Prediction of Sepsis From Clinical Data: The PhysioNet/Computing in Cardiology Challenge 2019*, Critical Care Medicine, 2020.
