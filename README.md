# Flock Energy — REFIT Data Cleaning, Appliance EDA & NILM

## Project overview

This project uses the REFIT Electrical Load Measurements dataset to investigate household electricity consumption and washing-machine usage. It covers raw-data quality assessment and cleaning, cross-household exploratory analysis, candidate cycle detection, and a household-held-out non-intrusive load monitoring (NILM) experiment.

The objective is to assess what can be learned about washing-machine demand from aggregate household power measurements, while making data-quality constraints and model errors explicit.

## Project objectives

1. **Raw data cleaning:** inspect raw House 1 measurements, resolve duplicate timestamps, assess irregular sampling and outages, and produce a one-minute series.
2. **Washing-machine EDA:** analyse Houses 2, 3, 5 and 6; compare data availability, power distributions, time-of-day patterns and heuristic candidate sessions.
3. **NILM:** estimate washing-machine power from aggregate power, compare a mean baseline with a Random Forest model, and test on a household excluded from training.
4. **Practical assessment:** discuss potential customer insights, measurement-resolution trade-offs and limitations, including transfer from UK REFIT households to other contexts.

## Dataset and household selection

Source: [REFIT Electrical Load Measurements dataset](https://pureportal.strath.ac.uk/en/datasets/refit-electrical-load-measurements-cleaned/).

REFIT contains aggregate and individual-appliance electricity measurements from UK homes. Raw House 1 data is used for the cleaning exercise. The official cleaned data is used for EDA and modelling.

| Purpose | Households |
|---|---|
| Washing-machine EDA | 2, 3, 5 and 6 |
| NILM training | 2, 3 and 5 |
| Held-out NILM test | 6 |

Washing-machine channels used: House 2 — `Appliance2`; House 3 — `Appliance6`; House 5 — `Appliance3`; House 6 — `Appliance2`.

## Repository structure

```text
Flock/
├── notebook/
│   ├── rawDataCleaning.ipynb
│   ├── washingMachineEda.ipynb
│   └── NILM_model.ipynb
├── outputs/
│   ├── Raw_Cleaning/
│   │   └── House1_1min_cleaned.csv
│   ├── EDA/
│   │   ├── coverage_summary.csv
│   │   ├── data_availability.csv
│   │   ├── data_quality_summary.csv
│   │   ├── energy_summary.csv
│   │   ├── power_summary.csv
│   │   ├── session_summary.csv
│   │   └── House*_candidate_sessions.csv
│   └── NILM/
│       └── final_nilm_results.csv
├── plots/
│   ├── Raw_Cleaning/
│   ├── EDA/
│   └── NILM/
├── README.md
├── DATA.md
├── RESULTS.md
└── Recommendation.md
```

Keep the cleaned House 1 CSV under `outputs/Raw_Cleaning/`, not `outputs/EDA/`. The raw dataset and official cleaned household files are not bundled; obtain them from the REFIT source and update the input paths in the notebooks.

## Methodology

### 1. Raw House 1 cleaning

- Parse and order timestamps; compare the formatted timestamp with the Unix timestamp.
- Remove exact duplicate rows.
- Group remaining repeated timestamps, retaining the first Unix timestamp and taking the median of each power channel.
- Resample available readings to one-minute means.
- Interpolate only short internal gaps (up to two consecutive minutes); retain longer gaps as missing.
- Preserve missing/outage flags so interpolated and unavailable periods are not silently treated as original observations.

### 2. Washing-machine EDA

- Filter records with `Issues = 1` before analysis.
- Select the household-specific washing-machine channel and aggregate power.
- Resample to one-minute means and retain missing intervals.
- Compare availability, zero/non-zero behaviour, power distributions, hourly and weekday patterns.
- Identify *candidate* sessions using a provisional 200 W threshold, a 15-minute merge gap, a 3-minute minimum active duration and a 120-minute maximum elapsed duration. These are heuristic segments, not verified appliance cycles.

### 3. NILM modelling

The target is washing-machine power; inputs are aggregate power and aggregate-derived temporal features (current value, lags, rolling means and power change). A mean predictor is the baseline and a Random Forest regressor is the machine-learning model. Houses 2, 3 and 5 are used for training; House 6 is held out as an unseen household. Features are generated within each household so lags do not cross household boundaries. Evaluation includes overall MAE, RMSE and R², plus errors when actual washing-machine power is above 200 W.

## Setup and execution

Python 3.9+ is recommended. Install the common dependencies:

```bash
python -m venv .venv
# Windows PowerShell: .venv\Scripts\Activate.ps1
pip install pandas numpy matplotlib seaborn scikit-learn jupyter
```

Then:

1. Download the raw House 1 file and official cleaned REFIT household files from the dataset source.
2. Update the input and output paths at the top of each notebook to match your machine.
3. Run `rawDataCleaning.ipynb` from top to bottom.
4. Run `washingMachineEda.ipynb` from top to bottom.
5. Run `NILM_model.ipynb` from top to bottom.
6. Confirm that CSV outputs and plots are saved under the repository's `outputs/` and `plots/` folders.

The notebooks may require additional package versions or memory depending on the full REFIT files. Do not commit the large raw source files unless repository size and dataset licensing permit it.

## Key results at a glance

- House 1 raw file: 4,377,877 records; 595,935 exact duplicate rows; 3,732,363 unique timestamps after timestamp consolidation.
- House 1 one-minute series: 513,322 minutes; 76,908 completely missing minute intervals before short-gap interpolation; 1,204 minute rows interpolated; 75,704 missing values per channel remained.
- Candidate sessions: 611 (H2), 1,027 (H3), 988 (H5) and 232 (H6), using the stated heuristic.
- Held-out House 6 Random Forest: MAE 13.98 W, RMSE 83.65 W, R² 0.329. On actual active minutes (>200 W), MAE was 1,036.63 W and RMSE was 1,209.41 W.

Overall metrics are strongly influenced by the large number of inactive minutes. Active-period errors show that the model is not yet reliable for accurate cycle-level power reconstruction.

## Assumptions and trade-offs

- Duplicate timestamp readings are consolidated with a median to reduce sensitivity to conflicting duplicate values.
- Two-minute interpolation is limited to short internal gaps; longer gaps remain missing to avoid inventing extended appliance behaviour.
- A 200 W threshold and 15-minute session merge rule are operational heuristics, not ground-truth labels.
- Excluding `Issues = 1` prioritises consistency between aggregate and appliance channels, but removes some observations and may affect representativeness.
- Household-held-out testing is a stricter transfer test than a random row split, but only one test household is used.


## Limitations and possible improvements

The dataset has missing intervals, household-specific behaviour and overlapping appliance signatures. The model's active-period errors are high. Improvements could include more training households, appliance-state classification followed by regression, sequence models, richer contextual features, household calibration, uncertainty estimates and validation across multiple unseen homes. A future evaluation should also compare one-minute results with explicitly downsampled 15- and 30-minute inputs.

See `DATA.md`, `RESULTS.md` and `Recommendation.md` for detailed data-quality findings, evaluation and practical implications.
