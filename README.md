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



