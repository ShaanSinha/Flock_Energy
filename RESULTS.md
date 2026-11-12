# Results and Evaluation

## 1. Executive summary

The project completed a raw House 1 cleaning workflow, washing-machine exploratory analysis across Houses 2, 3, 5 and 6, and a household-held-out NILM experiment. The Random Forest model improved aggregate test metrics over a mean baseline on House 6. However, its error on actual active washing-machine minutes remained high, so the result should be treated as an exploratory estimate rather than an appliance-level measurement product.

## 2. Raw House 1 cleaning results

The raw House 1 file contained 4,377,877 records, 595,935 exact duplicate rows and 645,514 duplicate timestamp records. Timestamp consolidation produced 3,732,363 unique timestamps. The mean sampling interval was 8.25 seconds, the median was 5 seconds and the maximum observed gap was 3,594,413 seconds (about 41.6 days).

After resampling, the one-minute series contained 513,322 rows. There were 76,908 completely missing minute intervals before interpolation. Time-based interpolation filled 1,204 rows in short internal gaps (up to two minutes); 75,704 missing values per channel remained. Long gaps were not interpolated. The before/after cleaning plot compares aggregate power and two appliance channels over the first 24 hours.

**Interpretation:** timestamp deduplication produces a consistent time index, but it cannot restore unobserved consumption. Retaining long gaps and flags avoids presenting extended outages as measured or reconstructed energy use.

## 3. Washing-machine EDA

### 3.1 Households and availability

EDA uses Houses 2, 3, 5 and 6, with household-specific washing-machine channels documented in `DATA.md`. After excluding `Issues = 1` records and resampling to one minute, availability was 75.30% (H2), 84.62% (H3), 86.75% (H5) and 84.65% (H6). Unequal availability should be considered when comparing usage frequency and energy.

### 3.2 Power distributions and inactivity

Median washing-machine power was zero in all four households. Zero-power shares were 94.40% (H2), 93.26% (H3), 52.43% (H5) and 97.45% (H6). House 5 had a much larger non-zero share (47.57%) than the other homes. High-power observations generally formed a cluster around 1,800–2,100 W, but low-power readings and differences in channel behaviour mean that a simple threshold cannot perfectly identify all operating phases.

The `power_distribution.png` and `power_distribution_log.png` plots show the strongly right-skewed, intermittent distributions. The high zero share makes it important to report active-period performance separately from overall model metrics.

### 3.3 Threshold and candidate sessions

A provisional 200 W threshold was selected after comparing the proportion of readings above several candidate thresholds. Candidate sessions were formed by merging active segments separated by at most 15 minutes, retaining sessions with at least three active minutes and no more than 120 minutes elapsed duration.

| Measure | House 2 | House 3 | House 5 | House 6 |
|---|---:|---:|---:|---:|
| Candidate sessions | 611 | 1,027 | 988 | 232 |
| Median elapsed duration (min) | 17 | 12 | 12 | 18 |
| Mean elapsed duration (min) | 21.18 | 13.47 | 19.65 | 19.26 |
| Median active minutes | 8 | 10 | 8 | 9 |
| Median peak power (W) | 2,034.89 | 1,849.57 | 2,028.05 | 1,932.48 |

These counts and durations describe rule-based candidate events only. They are not confirmed washing cycles. The 15-minute gap rule may merge separate uses, while the 200 W threshold may omit lower-power phases.

### 3.4 Estimated energy per candidate session

| Measure | House 2 | House 3 | House 5 | House 6 |
|---|---:|---:|---:|---:|
| Mean estimated energy (kWh) | 0.285 | 0.262 | 0.210 | 0.292 |
| Median estimated energy (kWh) | 0.220 | 0.278 | 0.168 | 0.095 |

Energy is calculated from available one-minute readings inside each candidate session. Missing measurements can cause underestimation; these estimates should not be interpreted as metered cycle-level ground truth.

### 3.5 Time-of-day and weekday patterns

The hourly comparison indicates different household routines: H2 has a morning peak around 8–9 AM and secondary early-afternoon activity; H3 activity is concentrated around 8–11 AM; H5 has broader daytime activity from morning into evening, with a peak around 10 AM; and H6 is concentrated around 10 AM–12 PM. The weekday comparison shows H2 highest on Monday, H3 relatively high on Monday and Tuesday, H5 highest on Saturday, and H6 with higher weekend activity.

These are descriptive patterns from the available observations, not causal explanations or forecasts. Availability differences and household-specific behaviour may affect the comparisons.

### 3.6 Required visual interpretation

- `representative_24hour_usage.png` overlays aggregate household power and washing-machine power for a representative day. Appliance activity occurs within the wider household load, illustrating why aggregate-only disaggregation is ambiguous.
- `detected_washing_machine_cycle.png` shows one heuristic candidate session, its surrounding power trace and the 200 W activation threshold. The highlighted interval is an algorithmic segment, not independently verified ground truth.
- `hourly_usage.png` compares all four selected households on a common hourly view; differences suggest household-specific usage schedules.
- `weekday_usage.png` compares usage across days of the week; the observed patterns vary across homes.
- `session_duration.png` and `session_energy.png` compare candidate-session characteristics. Their values inherit the limitations of the threshold-based detector and missing-data coverage.

## 4. NILM modelling and validation

### 4.1 Validation design

The target is washing-machine power and the inputs are aggregate power plus lag, rolling-average and change features derived from aggregate power. Houses 2, 3 and 5 are used for training; House 6 is kept as the unseen test household. This household-level split tests transfer to a home not seen during training and avoids the overly optimistic result that can arise when adjacent time rows from the same home are randomly split across train and test. Lagged features are generated within each household so history does not cross household boundaries.

### 4.2 Overall metrics

| Metric | Mean baseline | Random Forest with temporal features |
|---|---:|---:|
| MAE (W) | 28.0718 | 13.9797 |
| RMSE (W) | 102.8463 | 83.6482 |
| R² | -0.0218 | 0.3293 |

The Random Forest reduces MAE by approximately 50.2% relative to the mean baseline and also improves RMSE and R². The positive R² indicates that the model explains some test-set variation, but the score remains moderate.

### 4.3 Active-period error

For test observations where actual washing-machine power exceeded 200 W:

| Measure | Random Forest |
|---|---:|
| Active test observations | 2,794 |
| Active MAE (W) | 1,036.633 |
| Active RMSE (W) | 1,209.405 |

This is the critical qualification to the overall scores. Most minutes are inactive, so a model can obtain a relatively low overall MAE while still failing to reconstruct the high-power portions of a washing cycle. The active-period errors are large relative to typical operating power and make precise cycle-level energy estimation unreliable.

### 4.4 Visual evaluation and failure cases

- `final_actual_vs_predicted.png` compares the actual and predicted washing-machine power on the held-out household.
- `final_prediction_timeseries.png` shows actual and predicted traces over a sample period.

The main failure mode is inaccurate reconstruction of active high-power intervals, including underestimation of peaks and imperfect separation of washing-machine load from other simultaneous household loads. The model can also produce non-zero estimates during inactive periods. Overlapping appliance signatures, household-specific patterns and missing observations all contribute to the difficulty.

### 4.5 Model conclusion

The temporal-feature Random Forest is better than the mean baseline on the held-out household according to overall MAE, RMSE and R². It is not yet sufficiently accurate for reliable appliance-level billing, precise cycle detection or automated control. Its current use is best limited to exploratory aggregate monitoring and broad trend estimation, with active-period uncertainty clearly disclosed.

## 5. Limitations and next improvements

1. Only one household (House 6) is used for the final held-out test; results may not generalise to other homes.
2. The candidate-cycle detector uses heuristic thresholds and has no verified cycle labels.
3. Missing data and `Issues` filtering can change the observed usage distribution and underestimate energy.
4. Aggregate load contains overlapping signatures from multiple appliances.
5. REFIT represents UK homes and appliances; direct transfer to Indian homes requires local data, calibration and validation.
6. At 15- or 30-minute resolution, short phases are averaged together, peaks are smoothed, and cycle boundaries become harder to identify. This is expected to reduce disaggregation detail; it should be measured with a separate downsampling experiment rather than assumed from the current one-minute model.

Potential next steps include testing on multiple unseen households, collecting/using more representative labelled data, adding state classification before regression, evaluating sequence models, calibrating per household, quantifying uncertainty and conducting explicit 1-, 15- and 30-minute resolution comparisons.
