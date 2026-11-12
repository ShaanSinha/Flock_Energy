# Data Description and Preprocessing

## 1. Dataset source and contents

This project uses the [REFIT Electrical Load Measurements dataset](https://pureportal.strath.ac.uk/en/datasets/refit-electrical-load-measurements-cleaned/), collected from residential properties in the United Kingdom. It contains aggregate household power, individual appliance channels and timestamps. The nominal raw sampling interval is approximately eight seconds, but actual intervals are irregular.

Two dataset forms were used:

- **Raw House 1:** used for the required independent data-quality inspection and cleaning exercise. It has 12 columns: `Time`, `Unix`, `Aggregate` and `Appliance1` through `Appliance9`.
- **Official cleaned REFIT data:** used for washing-machine EDA and NILM. It includes the appliance and aggregate channels plus an `Issues` quality indicator.

## 2. Household selection and appliance mapping

| Household | Washing-machine channel | Use |
|---|---|---|
| House 2 | `Appliance2` | EDA and model training |
| House 3 | `Appliance6` | EDA and model training |
| House 5 | `Appliance3` | EDA and model training |
| House 6 | `Appliance2` | EDA and held-out model test |

The household-specific channel mapping is taken from the REFIT appliance mapping used in the analysis. NILM training uses Houses 2, 3 and 5; House 6 is held out entirely for evaluation.

## 3. Raw House 1: initial quality assessment

The raw House 1 file contained 4,377,877 rows and 12 columns. There were no initially missing cell values in the source rows, but timestamp coverage was irregular.

| Inspection | Result |
|---|---:|
| Exact duplicate rows | 595,935 |
| Unique timestamps after timestamp consolidation | 3,732,363 |
| Duplicate timestamp records before consolidation | 645,514 |
| Minimum sampling interval | 1 second |
| Median sampling interval | 5 seconds |
| Mean sampling interval | 8.25 seconds |
| Maximum sampling interval | 3,594,413 seconds (about 41.6 days) |

The formatted `Time` and `Unix` timestamps agreed to within one second. The observed maximum interval and other large gaps indicate substantial periods without measurements. The raw coverage extended from 2013-10-09 14:06:17 to 2014-10-01 01:27:07.

No negative values were found in the aggregate or appliance power channels. However, several channel maxima were extreme relative to their upper percentiles and should be treated as suspicious values requiring source-level verification rather than automatically interpreted as genuine appliance loads. Observed maxima included Aggregate 24,000 W; Appliance1 69,629 W; Appliance2 98,301 W; Appliance3 32,751 W; Appliance4 98,301 W; Appliance5 32,767 W; Appliance6 14,336 W; Appliance7 98,299 W; Appliance8 65,549 W; and Appliance9 65,535 W. The workflow did not silently clip these values.

## 4. Duplicate handling and timestamp ordering

The cleaning sequence was:

1. Remove exact duplicate rows.
2. Group records sharing the same `Time` timestamp.
3. Retain the first `Unix` value in each timestamp group.
4. Aggregate each power channel using the median within that timestamp group.
5. Sort the consolidated data chronologically.

After consolidation, the dataset contained 3,732,363 unique timestamps, with no duplicate timestamps and no missing cell values in the consolidated source table. Median aggregation was selected to reduce the influence of conflicting repeated readings. It does not establish which conflicting reading is physically correct.

## 5. One-minute resampling and missing-data treatment

The consolidated data was resampled to one-minute intervals using the mean of available readings within each minute.

| Measure | Result |
|---|---:|
| Total minute rows | 513,322 |
| Completely missing minute rows before interpolation | 76,908 |
| Missing share before interpolation | About 14.98% |
| Minute rows filled by short-gap interpolation | 1,204 |
| Remaining missing values per power channel | 75,704 |
| Remaining missing share per channel | About 14.75% |

Time-based interpolation was limited to at most two consecutive missing minutes and only applied to internal gaps. Longer gaps were left missing. The cleaning notebook also creates an original-missing indicator and a long-outage indicator based on runs longer than two minutes. The original-missing flag identifies minutes absent from the resampled source even when a short gap is subsequently interpolated; the long-outage flag distinguishes longer missing runs.

The cleaned one-minute series should therefore not be interpreted as fully observed throughout its time span. Missing values remain meaningful quality information.

## 6. Official cleaned REFIT data: preprocessing

The official cleaned REFIT release describes corrections including daylight-saving-time handling, appliance-channel alignment, forward filling of missing values, zeroing IAM spikes above 4,000 W, and an `Issues` indicator for records where the sum of individual appliance readings exceeds aggregate consumption.

For this analysis, records with `Issues = 1` were excluded. Aggregate and household-specific washing-machine channels were resampled to one-minute means. Missing minute values were retained rather than filled again in the EDA workflow.

The source files covered the following periods and issue rates:

| Household | Source start | Source end | Rows flagged `Issues=1` | Share |
|---|---|---|---:|---:|
| House 2 | 2013-09-17 22:08:11 | 2015-05-28 08:05:43 | 28,444 | 0.4961% |
| House 3 | 2013-09-25 19:21:09 | 2015-06-02 10:55:42 | 408,627 | 5.8420% |
| House 5 | 2013-09-26 09:56:09 | 2015-07-06 17:48:55 | 425,766 | 5.7298% |
| House 6 | 2013-11-28 12:15:35 | 2015-06-28 22:14:45 | 34,451 | 0.5519% |

After issue filtering and one-minute resampling:

| Household | Total minute rows | Valid minutes | Missing minutes | Availability |
|---|---:|---:|---:|---:|
| House 2 | 889,078 | 669,445 | 219,633 | 75.30% |
| House 3 | 885,095 | 748,997 | 136,098 | 84.62% |
| House 5 | 933,593 | 809,933 | 123,660 | 86.75% |
| House 6 | 831,480 | 703,854 | 127,626 | 84.65% |

Availability is the fraction of one-minute intervals with valid measurements after the selected filtering and resampling procedure. It is not a claim that every appliance channel is independently complete at every timestamp.

## 7. Washing-machine power characteristics

| Measure | House 2 | House 3 | House 5 | House 6 |
|---|---:|---:|---:|---:|
| Mean power (W) | 17.88 | 25.03 | 22.52 | 6.94 |
| Median power (W) | 0 | 0 | 0 | 0 |
| Zero-power share | 94.40% | 93.26% | 52.43% | 97.45% |
| Non-zero share | 5.60% | 6.74% | 47.57% | 2.55% |
| 99th percentile (W) | 206.27 | 1,472.34 | 464.89 | 99.80 |
| Maximum (W) | 2,250.56 | 2,111.89 | 2,345.00 | 2,185.00 |

The markedly different non-zero shares indicate household/channel heterogeneity. In particular, House 5 has many more non-zero readings, including low-power readings. These differences make a single threshold imperfect across all homes.

## 8. Candidate session and energy estimation assumptions

Candidate activity is defined as washing-machine power above 200 W. Active segments separated by gaps of up to 15 minutes are merged. Candidate sessions must contain at least three active minutes and have an elapsed duration no greater than 120 minutes. This produces heuristic candidate sessions, not verified ground-truth cycles.

Session energy is estimated by summing available one-minute power readings and converting watt-minutes to kWh. Missing readings during a session can lead to underestimation. The threshold and gap-merging choices can also merge distinct events or split a true cycle.

## 9. NILM feature preparation and remaining concerns

The NILM input uses aggregate power and aggregate-derived temporal features: current aggregate reading, 1-, 5- and 10-minute lags, 5- and 10-minute rolling averages, and aggregate power change. Rows lacking the required history or valid target/input values are excluded. Feature construction is performed within household boundaries to prevent lag values from carrying over between homes. The split is by household, not random rows: Houses 2/3/5 train and House 6 tests.

Remaining concerns include missing measurements, possible residual sensor anomalies, the official `Issues` filtering trade-off, household-specific appliance signatures, and the fact that REFIT represents UK homes. These limitations constrain generalisation to other populations, housing stock, appliances and electricity-monitoring systems.
