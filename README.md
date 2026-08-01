## Dataset Summary (CERT r4.2)

**Source:** CERT Insider Threat Test Dataset, release r4.2  
(Carnegie Mellon University / KiltHub — see link above)

**Time span:** ~18 months of simulated organizational activity  
**Log sources:** logon.csv, device.csv, file.csv, email.csv, http.csv, LDAP/ (org directory), answers/insiders.csv (ground truth)

**Ground truth:** r4.2 contains 3 injected insider threat scenarios, each with 
multiple simulated malicious users engaging in realistic bad behavior blended 
into otherwise normal organizational activity.

### Scenario Descriptions (r4.2)

**Scenario 1 — Data exfiltration before departure**  
A user with no prior history of removable-drive use or after-hours work 
suddenly begins logging in after hours, using a USB drive, and uploading 
data to an external file-sharing site (wikileaks.org). The user leaves the 
organization shortly after.

**Scenario 2 — Job-seeking data theft**  
A user starts browsing job websites and soliciting employment from a 
competitor. Before leaving the company, they use a thumb drive at a 
significantly elevated rate compared to their normal baseline to steal data.

**Scenario 3 — Disgruntled admin sabotage**  
A system administrator becomes disgruntled, downloads a keylogger, and 
transfers it via USB to their supervisor's machine. The next day, they use 
the captured keystrokes to log in as the supervisor and send a mass 
alarming email, causing organizational panic. The admin leaves immediately 
afterward.

*(Note: scenario descriptions 4 and 5 from CERT's documentation belong to 
other releases, not r4.2, and are not part of this project's scope.)*

### Why This Matters for Modeling

All three r4.2 scenarios share a common signature that guides our feature 
engineering:
- A **behavioral shift** from an established baseline (not random one-off 
  anomalies, but a sustained change in pattern)
- Involvement of **removable media (USB/device activity)** in every scenario
- Activity **escalating shortly before departure** or a triggering event
- Cross-source signals — no single log type tells the full story; the 
  signal emerges from combining logon, device, file, and email behavior

This confirms the project's core approach: model *normal per-user behavior*, 
then flag *sustained deviations* — rather than looking for single suspicious 
events in isolation.

### Ground Truth Stats (r4.2)

| Metric | Value |
|---|---|
| Total scenarios | 3 |
| Total malicious users | 70 |
| Malicious activity date range | 2010-06-10 7:54:10 to 2011-04-29 20:04:27 |

## Key Findings (Manual Exploration)

### Confirmed Detection: User AAM0658 (Scenario 1)
Manual inspection of `logon.csv` identified 9 after-hours sessions 
(00:00–05:54 AM) between Oct 21 – Nov 2, 2010, following 9 months of 
completely stable 9am–9pm logon behavior with zero prior after-hours 
activity. 

The detected window's start (Oct 23 01:34:19) and end (Oct 29 05:23:28) 
timestamps exactly match the CERT answer key's documented scenario window 
for this user. Additionally, 2 sessions (Oct 21, Nov 2) fall just outside 
the officially labeled window — suggesting the behavioral change may have 
begun slightly earlier and persisted slightly after the labeled incident 
period. This is a useful reminder that ground-truth windows can be 
narrower than the full behavioral shift, relevant for threshold tuning 
later in the project.

This finding validates the core project approach: behavioral deviation 
from an established personal baseline is detectable using simple 
aggregation, confirming that an anomaly-detection model has a legitimate, 
learnable signal in this dataset.

### Confirmed Multi-Source Detection: User AAM0658 (Scenario 1)

Cross-referencing `logon.csv` and `device.csv` reveals a complete, 
correlated behavioral signature:

| Date | Logon (after-hours) | USB Activity |
|---|---|---|
| Oct 21 | 00:18–01:20 AM | 3x Connect/Disconnect cycles (00:22–01:18) |
| Oct 23 | 01:34 AM | Connect/Disconnect (06:18–06:26) |
| Oct 27 | 04:31–05:54 AM | Connect/Disconnect (04:37–04:59) |
| Oct 29 | 00:06–05:23 AM | Connect/Disconnect (01:30–05:00) |
| Nov 2 | 02:31–03:02 AM | Connect (02:40) |

Zero USB events exist anywhere in this user's history before Oct 20, 2010. 
The tight time correlation between after-hours logons and USB connect/
disconnect cycles across 5 separate nights strongly matches Scenario 1's 
described behavior: an employee with no prior removable-media usage who 
begins working after hours and using a USB drive shortly before departure.

This confirms the injected threat signal is detectable through combined 
logon + device behavioral analysis, validating the project's anomaly 
detection approach.

## Cross-Source Aggregation Complete
Built a unified user-day master table combining all 5 log sources 
(logon, device, file, email, http), resulting in 330,452 user-day 
records. Each row represents one user's complete activity summary for 
one day, forming the foundation for feature engineering and 
model training.

Train/Test Split — Design Note

The CERT r4.2 dataset's earliest injected scenario begins June 10, 2010, just ~5 months into the ~16.5-month dataset. This constrains the training period to roughly the first 32% of the timeline (Jan 2 – June 1, 2010) rather than a more typical 70-75% split, since no scenario data can be included in training without leakage. This is a structural characteristic of the r4.2 dataset (70 malicious users active across a wide, overlapping timeframe) rather than a modeling choice, and is noted as a limitation: the model has a smaller "clean" baseline period to learn normal behavior from than would be ideal.

Split date: 2010-06-01
Train period: 2010-01-02 to 2010-06-01 (~32% of data)
Test period: 2010-06-01 to 2011-05-17 (~68% of data, contains all scenarios)


## EDA Highlights

Logon Hour Distribution
Malicious users show a visibly heavier tail into late-night/early-morning 
hours compared to normal users, consistent with the after-hours behavior 
described in the insider threat scenarios.

USB Activity Over Time
Malicious users show sharp, isolated spikes in USB activity clustered in 
narrow windows, while normal users show flat, near-zero device usage 
throughout — a clear behavioral signature.

Class Imbalance
Only 70 out of [total_users] users are malicious — confirming why accuracy 
is not a viable evaluation metric for this project.

Feature Correlation
Feature correlations are generally low, suggesting each log source 
contributes independent signal rather than redundant information — 
supporting the use of all 5 sources in feature engineering.

### Feature Engineering Note: Rolling Baseline Absorption Effect
Testing the device_count_deviation feature on user AAM0658 (Scenario 1) 
shows a large deviation spike (+6.0) on the first anomalous day (Oct 21), 
but subsequent deviations shrink and even turn negative as the 7-day 
rolling average absorbs the new (malicious) behavior into its baseline. 
This confirms rolling-window features are most sensitive to the ONSET of 
anomalous behavior rather than its persistence, a known limitation that 
motivates using a longer, stable-period autoencoder model (trained only 
on the pre-June 2010 "clean" period) alongside rolling features, rather 
than relying on rolling deviation alone.

## Feature Engineering Summary
Final feature table (`final_features.csv`) built at user-day granularity, 
combining:
- Raw daily counts: logon, device, file, email, http
- Behavioral features: after_hours_logon_count, is_first_usb_use, 
  external_email_count
- Personal baseline deviations: 7-day rolling average and deviation for 
  each raw count feature
- Ground truth labels (for evaluation only, not model input): 
  is_malicious_user, is_scenario_day

Total: 330452 user-day records, 1364 flagged as within an actual scenario window.

## Baseline Model: Isolation Forest

Trained on the pre-June 2010 "clean" period (32% of data), evaluated on 
the test period containing all 70 malicious users' scenario windows.

**Threshold-based results (contamination=0.02):**
- Scenario days flagged as anomalous: 6.0% (82/1,364)
- Normal days flagged as anomalous: 1.8% (4,004/223,917)
- Scenario days flagged at ~3.3x the rate of normal days

**Ranking-based results (more realistic for SOC alert-budget context):**
- Precision@100: 8% — a ~13x improvement over the 0.6% random base rate
- Precision@500: 5%

**Conclusion:** the baseline model demonstrates a real, learnable signal 
well above chance, but recall remains low and most alerts are false 
positives at any fixed threshold. This establishes the floor the 
autoencoder is expected to improve upon.

## Autoencoder Model
Architecture: 13 → 8 → 4 (bottleneck) → 8 → 13, trained on the 
pre-June 2010 clean period only.

- Final training loss (MSE): 0.261
- Final validation loss (MSE): 0.272
- Training converged smoothly over 50 epochs with no overfitting 
  (train/val loss remain close throughout)
- Training time: ~10 seconds

### Evaluation Nuance: Model Surfaces Unlabeled Malicious Activity
Investigating the highest-error case (user RKD0604, July 9 2010) reveals 
a textbook insider-threat signature: first-ever USB use (device_count=18, 
full deviation from a zero baseline), a corresponding file activity spike 
(file_count=27), after-hours logon activity, and 5 external emails — all 
on the same day. This closely mirrors the confirmed Scenario 1 pattern 
(user AAM0658), strongly suggesting genuine malicious behavior that falls 
outside the CERT answer key's officially labeled scenario window for 
this user. 

This illustrates a realistic evaluation challenge: precision@k penalizes 
the model for this "miss" even though it correctly surfaced meaningful 
threat-like behavior — underscoring that ground-truth-based metrics 
should be interpreted alongside qualitative case review, not in isolation.

## Model Evaluation Summary

| Metric | Isolation Forest | Autoencoder | Ensemble (avg rank) |
|---|---|---|---|
| PR-AUC (primary) | 0.0215 | 0.0250 | **0.0280** |
| ROC-AUC | 0.8561 | 0.8472 | — |
| Precision@100 | 8% | 5% | **12%** |


**Key findings:**
- The ensemble (averaging both models' rank scores) outperforms either 
  individual model on both PR-AUC and precision@100, confirming that 
  Isolation Forest and the autoencoder catch partially different anomaly 
  patterns.
- ROC-AUC (~0.85 for both models) looks strong, but PR-AUC (~0.02-0.03) 
  reveals the real difficulty of this task: with only 1,364 scenario 
  days out of 225,281 total test user-days (0.6%), ROC-AUC is not 
  sensitive enough to extreme imbalance, which is exactly why PR-AUC and 
  precision@k are used as the primary metrics per the evaluation plan.
- At a realistic analyst alert budget (top 100 daily alerts), the 
  ensemble model surfaces genuine threat activity ~20x more often than 
  random chance (12% vs 0.6% base rate).

## Deployment Demo — Sample Predictions

### 1. True Positive: User MCF0600, 2010-09-20 (confirmed scenario day)
**Input:** logon_count=7, device_count=20, file_count=27, first_usb_use=True, after_hours_logon_count=3

**API Response:**
```json
{
  "isolation_forest_score": -0.16686463826795328,
  "isolation_forest_flagged": true,
  "autoencoder_reconstruction_error": 35.82804377392311,
  "ensemble_indicator": 35.994908412191066,
  "risk_level": "HIGH"
}
```
Both models correctly flag this confirmed scenario day as high risk, driven by the first-time USB use and after-hours activity.

### 2. True Negative: User CAB0614, 2010-10-22 (normal day)
**Input:** logon_count=3, device_count=2, file_count=7, first_usb_use=False, after_hours_logon_count=0

**API Response:**
```json
{
  "isolation_forest_score": 0.1292862581739569,
  "isolation_forest_flagged": false,
  "autoencoder_reconstruction_error": 0.3521314941768232,
  "ensemble_indicator": 0.22284523600286632,
  "risk_level": "LOW"
}
```
Both models correctly clear this normal day as low risk.

### 3. Special Case: User RKD0604, 2010-07-09 (unlabeled anomaly, discussed in Day 10)
**Input:** logon_count=3, device_count=18, file_count=27, first_usb_use=True, after_hours_logon_count=1, external_email_count=5

**API Response:**
```json
{
  "isolation_forest_score": -0.11468990854929728,
  "isolation_forest_flagged": true,
  "autoencoder_reconstruction_error": 37.360431666230475,
  "ensemble_indicator": 37.47512157477977,
  "risk_level": "HIGH"
}
```
This user is confirmed malicious but this specific day falls outside their officially labeled scenario window. The model still correctly surfaces it as the single highest-risk case in the entire test set — a first-time USB spike with correlated file activity, matching the same signature as the confirmed Scenario 1 pattern.

### 4. Known Failure Case: User KPC0073, 2010-07-08 (missed insider — scenario day scored as low-risk)
**Input:** logon_count=2, device_count=0, file_count=0, first_usb_use=False, after_hours_logon_count=0

**API Response:**
```json
{
  "isolation_forest_score": 0.23229149666985138,
  "isolation_forest_flagged": false,
  "autoencoder_reconstruction_error": 0.03366122291454252,
  "ensemble_indicator": -0.19863027375530887,
  "risk_level": "LOW"
}
```
Both models score this confirmed scenario day as low risk, missing it entirely.

**This demonstrates a real limitation:** not all injected scenario behavior produces a strong anomaly signal in the engineered features, particularly if the malicious activity that day doesn't deviate sharply from the user's personal baseline. In this case, zero device/file activity and no after-hours logons meant the model had no elevated signal to act on — suggesting the malicious behavior that day likely manifested through a channel (e.g., email content, subtler access patterns) not captured by the current count-based features.


## Deployment Demo — Sample Predictions

The FastAPI scoring service (`deployment/app.py`) loads the trained 
Isolation Forest, StandardScaler, and Autoencoder, and returns a combined 
risk assessment for any user-day feature record.

| Case | User / Day | Iso. Forest Flag | Autoencoder Error | Risk Level | Ground Truth |
|---|---|---|---|---|---|
| True Positive | MCF0600, 2010-09-20 | Yes | 35.83 | HIGH | Confirmed scenario day |
| True Negative | CAB0614, 2010-10-22 | No | 0.35 | LOW | Normal day |
| Unlabeled Anomaly | RKD0604, 2010-07-09 | Yes | 37.36 | HIGH | Malicious user, outside labeled window (see Day 10 finding) |
| **Known Failure Case** | KPC0073, 2010-07-08 | No | 0.03 | **LOW** | **Confirmed scenario day — MISSED** |

### Known Limitation
User KPC0073's scenario-day activity on 2010-07-08 shows no elevated 
device, file, or after-hours signals (all near-zero deviations from 
baseline). This demonstrates that the current feature set does not 
capture every form of malicious behavior — likely because this 
particular day's insider activity manifested through channels (e.g., 
email content, subtler access patterns) not captured by the current 
count-based features. This is an honest, documented gap for future 
feature engineering work.
