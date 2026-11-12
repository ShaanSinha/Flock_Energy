# Practical Recommendation for Household Energy Monitoring

## Executive recommendation

REFIT data demonstrates that aggregate household electricity measurements contain useful information about broad washing-machine usage patterns, but the current model is not accurate enough to provide dependable appliance-level power readings or cycle-level energy estimates. A utility or energy-service provider could use this approach as an **exploratory customer-insight feature**, provided that estimates are labelled as approximate and are not used for billing, enforcement or automated decisions.

## 1. Proposed customer opportunity: washing-machine usage feedback

A pilot could provide participating customers with a simple weekly insight showing likely washing-machine activity windows and broad usage trends. For example, it could identify recurring time-of-day patterns and help customers understand when high-power appliance activity tends to occur. The current model and heuristic cycle detector should be used only to generate provisional insights; customers should not be told that every detected event is a confirmed wash cycle.

A useful customer-facing intervention would be an optional notification or weekly report that encourages customers to review their appliance-use schedule and, where their tariff supports it, consider shifting flexible usage away from more expensive or peak periods. The recommendation must be conditional on the customer's actual tariff and household circumstances; the REFIT analysis itself does not establish that a particular time is cheaper or that shifting use will save money.

## 2. How to measure whether the pilot works

Run a controlled pilot with consenting households and establish a baseline period before showing feedback. Where possible, use plug-level or appliance-level reference measurements for a subset of homes to estimate detection precision, missed events and energy-estimation error.

Measure:

- **Detection quality:** precision and recall of likely washing-machine events against reference measurements.
- **Energy accuracy:** MAE/RMSE of estimated cycle energy against reference measurements.
- **Customer outcome:** change in washing-machine kWh per household per week, adjusted for occupancy or other relevant differences where data permits.
- **Peak shifting:** change in the share of washing-machine energy consumed during the customer's defined peak-price window.
- **Engagement:** report views, notification opt-in and customer-reported usefulness.

Compare pilot households with a suitable control group or use a pre/post design with a clearly defined baseline. Report uncertainty and avoid attributing all energy changes to the feedback intervention without a comparison design.

## 3. What the current analysis supports

The EDA shows that washing-machine usage is intermittent and that time-of-day and weekday patterns differ across the four analysed homes. This supports the feasibility of broad, household-specific usage summaries. The held-out House 6 Random Forest achieved MAE 13.98 W, RMSE 83.65 W and R² 0.329 overall, outperforming the mean baseline. However, on actual active minutes above 200 W, MAE was 1,036.63 W and RMSE was 1,209.41 W. This large active-period error means the model should not currently be presented as an accurate appliance meter.

## 4. Important limitations and deployment safeguards

- **Overlapping appliance signatures:** multiple appliances can operate simultaneously, so aggregate power does not uniquely identify the washing machine.
- **Missing data:** outages and missing intervals can hide operating phases and lead to underestimation of energy.
- **Household differences:** appliance ratings, age, cycle settings, installation and customer behaviour vary. A threshold or model learned on one set of homes may not transfer.
- **Heuristic cycle labels:** the 200 W threshold and 15-minute gap rule produce candidate sessions, not confirmed cycles.
- **UK-to-India generalisation:** REFIT contains UK homes and their appliance mix, housing characteristics and usage patterns. Indian households may differ in washing-machine types, supply conditions, household size, climate, usage schedules and metering infrastructure. The present model should not be deployed in India without collecting representative local data, remapping appliance channels and validating performance across regions and household types.
- **Privacy and consent:** household load profiles can reveal routines. Any pilot should use informed consent, data minimisation, secure handling and clear retention policies.

## 5. Impact of 15-minute and 30-minute measurements

At 15-minute resolution, several short washing-machine phases may be averaged into a single reading. Peak power is smoothed, on/off transitions become less precise and different appliances operating within the same interval become harder to separate. At 30-minute resolution, these effects become stronger: multiple phases or even separate appliance events can be combined, and short cycles may be represented by only one or two readings. This makes cycle detection, phase reconstruction and appliance-level energy attribution less reliable.

The current project was analysed at one-minute resolution. The impact of 15- and 30-minute sampling has not been quantified by a separate controlled downsampling experiment, so the statements above are expected measurement-resolution consequences rather than measured model-performance results. A deployment decision should compare all three resolutions using the same household-held-out validation design.

## 6. Recommended next step

Proceed only with a limited, opt-in validation pilot. First improve active-period detection and test on multiple unseen households; then compare one-, 15- and 30-minute inputs and validate against appliance-level reference meters. Expand to a new geography, including India, only after collecting representative local training and test data. Until those checks pass, position the system as an approximate customer-awareness tool, not a billing-grade or safety-critical measurement system.