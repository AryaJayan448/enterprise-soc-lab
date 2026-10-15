# IR-002 — Windows - New Local User Added to Administrators

## 1. Summary

Microsoft Sentinel generated incident 18 after the local account `labtest` was created and added to the local Administrators group on `WIN-58N5TIST71S` by `arya-labadmin`.

The activity was part of a controlled security test to validate the detection for new local administrator accounts. The detection correctly identified the account creation and privilege assignment.

## 2. Detection

* **Incident ID:** 18
* **Detection rule:** `Windows - New Local User Added to Administrators`
* **Severity:** High
* **MITRE ATT&CK:** T1136.001 — Create Account: Local Account; T1098 — Account Manipulation
* **Events:** Windows Security Event ID 4720 (user account created) and Event ID 4732 (member added to a security-enabled local group)
* **Correlation:** The events were joined using the new account's SID.
* **Group identification:** The Administrators group was identified using SID `S-1-5-32-544`.
* **Schedule:** Every 5 minutes, with a 15-minute lookback.
* **Entities:** Two Account entities and one Host entity.

## 3. Timeline (UTC)

| Time (UTC) | Event                                                                                                         |
| ---------- | ------------------------------------------------------------------------------------------------------------- |
| 12:17:35   | The `labtest` account was created and added to the Users and Administrators groups. Its SID ended in `-1005`. |
| 12:28:30   | The first alert was generated for the account-creation and group-membership events from the 12:17:35 test.    |
| 12:28:33   | A second test pair of account-related events was generated.                                                   |
| 12:28:48   | Incident 18 was created.                                                                                      |
| 12:33:43   | An alert from the second test pair was added to incident 18.                                                  |
| 12:38:42   | Another alert from the second test pair was added to incident 18.                                             |
| 12:38:46   | A further alert from the second test pair was added to incident 18.                                           |
| 12:39:13   | Incident 18 was last updated.                                                                                 |

Four alerts across the two test runs were grouped into one incident. The first alert related to the earlier test pair, while the three later alerts related to the second test pair.

## 4. Evidence

The investigation used the following evidence:

* **New account:** `labtest`
* **Creator:** `arya-labadmin`
* **Added by:** `arya-labadmin`
* **Member SID:** Ends in `-1005`
* **Target host:** `WIN-58N5TIST71S`
* **Target group SID:** `S-1-5-32-544` (local Administrators)
* **Screenshot:** `screenshots/day15-windows-incident-18.png`

The account name, SID, creator, host, and event timing were consistent with the controlled test.

## 5. Analysis

The evidence indicates that `arya-labadmin`, the lab machine's administrator account, created `labtest` and added it to the local Administrators group.

The account name and event timestamps matched the planned test activity. The SPL investigation of failed logons showed bursts at 10:12 and 11:16 on 11 October, and at 07:39 and 08:01 on 12 October. No failed-logon bursts were identified around the 12:17 UTC account-creation test.

The available evidence did not indicate a remote origin. The detection correctly correlated the account-creation and group-membership events.

The high severity reflects the potential risk of creating a privileged account; it does not by itself mean the activity was malicious.

## 6. Verdict and Incident Status

**Investigation verdict: Benign positive.**

The detection worked as intended and identified a new local administrator account. The activity was authorised and performed as part of security testing, so the alert was a valid detection of benign activity rather than a false positive.

**Microsoft Defender incident status:** Incident 18 was resolved with classification *Informational, expected activity* and determination *Security testing*, with the comment "Own lab test".

## 7. Impact and Account Cleanup

No impact to production systems was identified. The activity occurred on the lab VM `WIN-58N5TIST71S` as part of a controlled test.

**Test account cleanup:** The temporary `labtest` account was confirmed absent from the VM: both `net user labtest /delete` and `net user labtest` reported that the user name could not be found (it had already been removed during earlier test cycles).

## 8. Recommendations

1. Compare account creation and administrator-group changes against approved change records before closing similar incidents.
2. Consider alerting on these changes outside approved administrative change windows while retaining visibility into legitimate privileged-account changes.
3. In a real environment, extend the detection to relevant domain-group membership events, including Event IDs 4728 and 4756, where applicable.
4. Keep User Account Management and Security Group Management auditing enabled so the required events are generated and available for collection.
5. Test the event-grouping configuration to determine whether it can reduce duplicate alerts without losing useful evidence.
6. Remove temporary test accounts after testing and record the verified cleanup result.

## 9. Lessons Learned

* The required audit policy was initially disabled, so Windows did not generate the 4720 and 4732 events needed by the detection. Sentinel cannot detect events that were never logged.
* Event ID 4732 provided the group-member SID even though `MemberName` was `-`, making the SID join necessary to correlate the events reliably.
* The first alert arrived on a rule run after the detection was saved, so the initial response time should not be treated as a steady-state latency measurement.
* Four alerts from the two test runs were grouped into one incident, demonstrating the grouping behaviour observed for this rule.
* A valid detection, a benign verdict, incident resolution, and account cleanup are separate steps, and each needs its own check.
