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
