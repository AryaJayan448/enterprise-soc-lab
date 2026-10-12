# Day 16 - Sentinel Detection Tuning, Hunting and Workbook

## 1. Goal

Tuned the Windows brute-force detection to reduce duplicate alerts, investigated the results, and carried out two threat-hunting queries.

Created a Microsoft Sentinel workbook to summarise incidents, failed logons, and log volume by source.

## 2. Investigating the grouping problem

On Day 15, I suspected that Rule A's loopback IP address (`127.0.0.1`) was preventing alerts from being grouped into one incident.

I checked incidents 16 and 17 and confirmed that both contained the same Host (`WIN-58N5TIST71S`) and IP address (`127.0.0.1`). This refuted my original hypothesis that the loopback IP was the reason for the duplicate incidents.

The cause of the different grouping behaviour remains unexplained. The only visible configuration difference I identified between Rule A and Rule B was the **event grouping setting**: Rule A was configured to create a single alert, while Rule B was configured to create an alert for each event.

I have not tested whether this setting explains the difference. Testing it on Rule A remains an open item.

## 3. Tuning Rule A

I updated the Windows brute-force detection to reduce repeated alerts from overlapping query windows.

The deployed query is:

```kusto
SecurityEvent
| where EventID == 4625
| summarize Failures = count(), Users = make_set(TargetUserName), FirstSeen = min(TimeGenerated), LastSeen = max(TimeGenerated), LastIngested = max(ingestion_time()) by IpAddress, Computer
| where Failures >= 5
| where LastIngested > ago(5m)
```

The important change is the `LastIngested` condition. The failure count still uses events across the full 10-minute lookback window, but the rule only triggers when the latest matching event was ingested within the last five minutes.

I chose this approach instead of filtering all events using `ingestion_time()` before counting them. Filtering the input events too early could prevent a slow burst of failed logons from reaching the threshold of five within the 10-minute detection window.

The aim was to reduce repeated alerts while preserving the original brute-force detection logic.

## 4. Test results

I tested the tuned rule twice. Each test generated one alert and one incident, compared with two alerts and two incidents for a single burst before tuning.

| Measurement               | Test 1                                     | Test 2                      |
| ------------------------- | ------------------------------------------ | --------------------------- |
| Events generated (UTC)    | 8 events between `07:39:16` and `07:39:33` | Last event at `08:01:59.5`  |
| Ingestion completed (UTC) | By `07:40:01.8`                            | `08:02:24.8`                |
| Alert (UTC)               | `07:49:57.2`                               | `08:09:24.9`                |
| Incident created (UTC)    | Incident 19 at `07:50:03.7`                | Incident 20 at `08:09:32.6` |
| Alerts                    | 1                                          | 1                           |
| Incidents                 | 1                                          | 1                           |

Both tests produced the expected result of one alert and one incident. This is an improvement over the duplicate-alert behaviour observed on Day 15.

However, two successful tests do not establish that duplicates are eliminated in every situation. Further testing is needed, particularly with different event timings and the event grouping configuration.

## 5. A mistake I made

During the second test, I initially concluded that the detection had produced a false negative after checking for an alert too early. The alert arrived later at `08:09:24.9 UTC`, followed by incident 20 at `08:09:32.6 UTC`.

This was a reminder not to conclude that a detection has failed based on a single early check.

**Lesson:** Allow enough time for scheduled rule execution, data ingestion, alert generation, and incident creation before classifying a test as a false negative.

## 6. Latency

The two tests showed ingestion delays of approximately 25–28 seconds. However, the time from ingestion to alert was approximately 7–10 minutes, longer than the 4 minutes 41 seconds observed in the later Rule B test on Day 15.

| Measurement                 | Test 1              | Test 2              |
| --------------------------- | ------------------- | ------------------- |
| Approximate ingestion delay | 25–28 seconds       | 25 seconds          |
| Ingestion to alert          | About 10 minutes    | About 7 minutes     |
| Result                      | 1 alert, 1 incident | 1 alert, 1 incident |

The two observations suggest that the tuned rule can take several minutes to produce an alert even when ingestion is relatively quick. The cause of the longer alert latency has not been established, and two data points are not enough to determine a reliable average or prove the reason for the difference.

Further testing would be needed to distinguish the effects of scheduled rule execution, query timing, and other processing delays.

## 7. Threat hunting

I ran two hunting queries to examine process activity collected from the Windows lab machine.

### Query 1 — Rare processes

The first query searched for uncommon process activity.

The results were consistent with benign boot-time services. Microsoft Edge's updater was also observed running from a temporary directory (`Temp`).

The temporary path was worth investigating, but the observed activity was assessed as benign in this lab. A temporary execution path alone is not sufficient evidence of malicious activity.

### Query 2 — User-writable paths and potentially abusable tools

The second query looked for process activity involving user-writable paths and tools that can be abused by attackers.

The results included:

* OneDrive activity.
* `cmd.exe` launched from Explorer.
* `rundll32.exe` launched from `svchost.exe`.

These results were assessed as benign based on the observed lab activity. The process names and parent-child relationships are useful hunting leads, but they need context before being classified as malicious.

### Finding — Command-line auditing was disabled

The main finding was that the `CommandLine` field was empty in the process-creation telemetry. Without command-line arguments, it was harder to understand what a process had actually executed.

I enabled command-line auditing by setting `ProcessCreationIncludeCmdLine_Enabled = 1`.

I then verified that commands such as `whoami /groups` and `ipconfig /all` appeared with their arguments in the collected process events.

This improved the available context for future investigations. Command-line logging can contain sensitive information, so access to the resulting logs should be appropriately controlled.

## 8. Workbook

I created a Microsoft Sentinel workbook with three panels to provide a quick overview of the lab's detection and telemetry data.

| Panel                  |                                                Result | Purpose                                                 |
| ---------------------- | ----------------------------------------------------: | ------------------------------------------------------- |
| Incidents by detection |                        `7 / 7 / 4 / 1 / 1` — 20 total | Shows the distribution of incidents across detections.  |
| Failed logons per hour |                                                  `31` | Summarises failed logon activity over time.             |
| Log volume by source   | Syslog: `3.33K`; SecurityEvent: `1.08K`; Event: `637` | Compares the volume of data collected from each source. |

The workbook provides a high-level view of incident distribution, authentication failures, and data ingestion. It can help identify areas that may need further investigation, although counts alone do not establish whether activity is malicious.

## 9. Open items

* Test the event grouping setting on Rule A to determine whether it explains the different incident-grouping behaviour observed on Day 15.
* Remove the `Monitored Objects Contributor` role at the end of the lab and verify that the assignment has been removed.
* Review and classify incidents 1–4, which remain unclassified.
* Continue testing the tuned Rule A with different event timings to check whether duplicate alerts recur and to gather more reliable latency measurements.

## 10. Interview line

I tuned a Microsoft Sentinel brute-force detection to reduce duplicate alerts, investigated Windows process activity through threat hunting, enabled command-line auditing to improve investigation context, and built a workbook to summarise incidents, failed logons, and log volume.
