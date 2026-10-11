# Day 15 - Windows Detections (Brute Force, New Local Admin)

## 1. Goal

Built two Microsoft Sentinel detections to identify Windows password guessing and the creation of new local administrator accounts.

Tested both rules against Windows Security events and investigated issues with audit policy, overlapping query windows, and duplicate incidents.

## 2. Reading a 4625 event

Event ID 4625 records a failed logon attempt.

| Column           | What it holds                                                                                        |
| ---------------- | ---------------------------------------------------------------------------------------------------- |
| `TargetUserName` | The username used in the attempted logon, such as `fakeuser`.                                        |
| `Account`        | The domain and username in the format `-\fakeuser`. The `-` indicates that no domain was supplied.   |
| `IpAddress`      | The source address recorded for the attempt. In this test, it was `127.0.0.1`, the loopback address. |
| `LogonType`      | The type of logon attempted. Value `3` indicates a network logon.                                    |

The event also includes fields such as `WorkstationName`, which identified the test machine as `WIN-58N5TIST71S`. A status code such as `0xC0000064` can indicate that the specified username does not exist.

## 3. Rule A - Windows brute force (T1110.001)

The detection queries `SecurityEvent` for failed logons with Event ID `4625`, then groups the events by source IP address and computer. It counts failures, collects the attempted usernames, and records the earliest and latest event times.

The rule alerts when there are at least five failures within the query window.

* **Frequency:** Every 5 minutes
* **Lookback:** 10 minutes
* **Threshold:** At least 5 failed logons
* **Severity:** Medium
* **MITRE ATT&CK:** Credential Access — T1110, Brute Force; T1110.001, Password Guessing
* **Entities:** IP address (`IpAddress`) and host (`Computer`)

The 10-minute lookback gives the rule enough time to detect a burst of failed logons across two consecutive 5-minute runs. However, it also means the same events can be seen by more than one run.

The test generated eight failed logons between `11:15:59` and `11:16:16.717 UTC`. The alert reported `Failures = 8`.

Suppression was initially enabled to stop the rule for five hours after an alert. I turned it off because it could prevent the rule from detecting further brute-force activity during that period.

## 4. Rule B - New local user added to Administrators (T1136.001 / T1098)

This detection identifies a newly created local user who is added to the local Administrators group by joining two Windows Security events:

* **Event ID 4720:** A user account was created. `TargetUserName` identifies the new account, and `TargetSid` contains its security identifier (SID).
* **Event ID 4732:** A member was added to a security-enabled local group. In this test, `MemberName` was `-`, so I used `MemberSid` to identify the account that was added.

The query joins the events where the new user's `TargetSid` matches the `MemberSid` in the group-membership event. It filters for the well-known Administrators group SID, `S-1-5-32-544`, rather than the display name `Administrators`, because group names can vary by Windows language.

Two Event ID 4732 records appeared after one `net localgroup` command. Windows automatically added the new account to the local Users group (`S-1-5-32-545`), and the second event recorded the explicit addition to the Administrators group. Filtering on `S-1-5-32-544` distinguishes the administrator-group event.

* **Frequency:** Every 5 minutes
* **Lookback:** 15 minutes
* **Severity:** High
* **MITRE ATT&CK:** T1136.001 — Create Account: Local Account; T1098 — Account Manipulation

The 15-minute lookback gives the rule enough time to find the account-creation and group-membership events together, even if they arrive at different times.

## 5. Problem found: audit policy

Initially, the query returned no Event ID 4720 or 4732 records from `SecurityEvent`.

I checked the local Windows Security log with `Get-WinEvent`, which also returned no matching events. This showed that Windows had not written the events locally, rather than Sentinel simply failing to collect them.

The cause was that auditing for **User Account Management** and **Security Group Management** was disabled, likely because of an earlier Group Policy Editor change.

I enabled success and failure auditing for both subcategories:

```powershell
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
```

I checked the settings with `auditpol /get` and confirmed that Success and Failure auditing were enabled. After that, Event IDs 4720 and 4732 appeared in the local Security log.

**Lesson:** The Azure Monitor Agent and Sentinel can only collect events that Windows actually generates. If the relevant audit policy is disabled, a detection rule cannot find events that were never logged.

## 6. Problem found: duplicate alerts

Rule A generated two incidents for the same burst of eight failed logons:

* Incident 16 was created at `11:26:15.486 UTC`.
* Incident 17 was created at `11:31:24.933 UTC`.

The corresponding alert times were `11:26:04.949 UTC` and `11:31:11.308 UTC`.

The likely cause is the overlap between the 10-minute lookback and the 5-minute query frequency. The same burst can be included in two consecutive rule runs, so both runs can meet the threshold and generate alerts.

Alert grouping was configured to match all entities, but it did not merge the two Rule A alerts into one incident. I have not yet established why the grouping behaved differently for Rule A.

Rule B also showed overlapping runs. With a 15-minute lookback and a 5-minute frequency, the same event pair can be detected by three runs. Across the two tests, four alerts were generated, but all four were grouped into incident 18.

One hypothesis to test on Day 16 is whether Rule A's loopback IP entity (`127.0.0.1`) affects entity matching or alert grouping. Rule B uses Account and Host entities, so the difference may be relevant, but this has not been confirmed.

Using `ingestion_time()` as a quick deduplication fix may be unsuitable because it could filter out events needed to detect a slower attack. The next step is to investigate the grouping behaviour and tune the overlapping query windows without weakening detection.

## 7. Results and latency

All times below are in UTC.

| Measurement                | Rule A — Brute force                                         | Rule B — New local admin                                                                             |
| -------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| Last event generated (UTC) | `11:16:16.717`                                               | `12:28:33.689`                                                                                       |
| Ingested (UTC)             | `11:16:44.812` (28 seconds)                                  | `12:29:02.545` (29 seconds)                                                                          |
| Alert (UTC)                | `11:26:04.949` and `11:31:11.308`                            | `12:28:30.134` (earlier pair); `12:33:43.983`, `12:38:42.899` and `12:38:46.091` (later test)        |
| Incident (UTC)             | Incident 16 at `11:26:15.486`; incident 17 at `11:31:24.933` | Incident 18 at `12:28:48.053`, created by the earlier 12:17 test pair; later alerts were added to it |
| Ingestion to first alert   | 9 minutes 20 seconds                                         | 4 minutes 41 seconds                                                                                 |
| Event → first alert        | About 9 minutes 48 seconds                                   | About 5 minutes 10 seconds                                                                           |

**Interpreting the timings:** Rule A's timing may not represent steady-state performance because its first alert occurred on the first run after the rule was saved. Rule B's later test gives a clearer example: the event was ingested in 29 seconds, followed by the first alert about 4 minutes 41 seconds after ingestion, consistent with a 5-minute scheduled rule.

Incident 18 was created at `12:28:48.053 UTC` by the earlier test pair. The later alerts were added to that existing incident, so they did not create another incident. The first alert from the earlier pair preceded incident creation by about 18 seconds.

## 8. Limitations

### Rule A — Windows brute force

1. **Slow-and-low attacks:** An attacker could keep failures below five within each 10-minute window.
2. **Password spraying:** Attempts spread across many accounts or source IP addresses may not reach the threshold for a single IP-and-host group.
3. **False positives and source limitations:** Users mistyping passwords or services using incorrect credentials can trigger the rule. A loopback source address does not establish that an external attacker was responsible.

### Rule B — New local administrator

1. **Legitimate administrative changes:** An approved account creation or administrator-group change can trigger an alert and should be checked against change records.
2. **Disabled auditing:** If User Account Management or Security Group Management auditing is disabled, the necessary events may never be generated.
3. **Other privilege-escalation methods:** The detection focuses on account creation and local group membership using Event IDs 4720 and 4732. Other ways of obtaining privileges, or missing telemetry, may not be detected.

## 9. Interview line

I built and tested two Microsoft Sentinel detections for Windows password guessing and new local administrator accounts, corrected missing Security events by enabling the required audit policies, and investigated how overlapping query windows and a candidate difference in entity mapping affected whether duplicate alerts were grouped into one incident.
