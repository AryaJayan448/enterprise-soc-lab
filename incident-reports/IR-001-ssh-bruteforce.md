# IR-001 — SSH Brute-Force Detection

## 1. Summary

Microsoft Sentinel detected repeated failed SSH login attempts against the `arya-labadmin` account from `192.168.64.1` between 20:48 and 20:49 UTC. The rule triggered because the number of failed attempts reached the configured threshold within the detection window. A successful SSH login happened shortly after the failed attempts. `192.168.64.1` is my Mac, which i used as the admin workstation for this lab, so the activity was expected security testing.

## 2. Detection

The detection was created as a Microsoft Sentinel scheduled analytics rule.

* **Rule:** SSH Brute Force - Repeated Failed Logins From One Source
* **MITRE ATT&CK:** T1110.001 — Password Guessing
* **Tactic:** Credential Access
* **Threshold:** 5 or more failed SSH logins
* **Time window:** 10 minutes
* **Source:** Linux SSH authentication logs collected through Azure Monitor Agent
* **Entity:** Source IP address

The rule looks for repeated failed SSH login attempts from the same source. If there are 5 or more failures within 10 minutes, the rule creates an alert.

## 3. Timeline (UTC)

| Time                | Event                                                                                                      |
| ------------------- | ---------------------------------------------------------------------------------------------------------- |
| 20:48:14 - 20:48:26 | 3 failed passwords for arya-labadmin from 192.168.64.1 (connection port 51167)                             |
| 20:48:39 - 20:48:49 | 3 failed passwords (1 log line + "message repeated 2 times"), port 51169                                   |
| 20:48:56 - 20:49:08 | 3 failed passwords, port 51170                                                                             |
| 20:49:19            | Accepted password for arya-labadmin from 192.168.64.1 (port 51171); SSH session opened (systemd session 6) |
| 20:49:30            | User disconnected; session closed (about 11 seconds)                                                       |
| 21:02:19            | Incident 5 created in Microsoft Sentinel                                                                   |

## 4. Evidence

| Item                   | Value                                                                                               |
| ---------------------- | --------------------------------------------------------------------------------------------------- |
| Source of logs         | Linux auth log, collected by Azure Monitor Agent, table Syslog                                      |
| Alert evidence         | Failures = 8, FirstSeen 20:48:14Z, LastSeen 20:49:08Z, SourceIP 192.168.64.1, Users = arya-labadmin |
| Actual failed attempts | 9 (the rule counted 8 rows because rsyslog collapsed 2 repeats into one line)                       |
| Activity after login   | No sudo, no other sessions, no other accounts; session closed by the user                           |
| Classification         | Informational, expected activity (Security testing), status Resolved                                |

## 5. Analysis

Repeated failed SSH logins from one source are suspicious because this can be a sign of password guessing or brute-force activity. In a real environment, several attempts against the same account within a short period would need to be investigated.

In this case, the source IP was `192.168.64.1`, which is my Mac and the admin workstation for this lab. The failed logins were deliberately generated as part of my security testing, so the activity was expected.

The rule itself only checks for the failed login condition. It triggered because there were 8 counted failures from the same IP within the 10-minute window. I found the successful login later during the investigation. The successful login happened at 20:49:19, shortly after the failed attempts. In a real environment, failures followed by a successful login would be more concerning because it could mean that someone eventually guessed the password.

There is a difference between the 8 failures reported by the rule and the 9 actual failed attempts. The reason is that rsyslog used `message repeated 2 times` for repeated messages. The rule counted the log rows, so two repeated events were represented by one row. This shows why it is important to understand how the logging system formats repeated events before relying only on row counts.

The incident was created at 21:02:19 UTC, while the last failed login was at 20:49:08 UTC. This means the incident appeared about **13 minutes after the last failed login**. This is useful as a detection latency measurement and is something i would want to monitor when tuning a real detection.

The logs also do not show exactly what commands were run during the successful SSH session. I can see that the session opened and closed after about 11 seconds, and there was no sudo activity or other session recorded. But without command auditing, i cannot say exactly what happened inside that session.

## 6. Verdict

**Classification: Informational, expected activity (Security testing)**
**Status: Resolved**

The rule condition was met because there were **8 counted failed login events from one IP within 10 minutes**.

During the investigation, i found that the source IP was my own Mac and that the failed logins were deliberately generated for the lab. I also found a successful login shortly after the failures, but this was discovered during the investigation and was **not part of the rule's detection condition**.

There was no evidence of malicious activity after the login. There were no sudo commands, no other sessions and no other accounts involved. The session also closed shortly afterwards.

## 7. Impact

No impact was identified.

The activity affected only the Linux lab VM and the `arya-labadmin` test account. There was no evidence of privilege escalation, activity against other accounts, or additional sessions.

## 8. Recommendations

1. **Use SSH keys instead of passwords** where possible. This reduces the risk of password guessing against SSH accounts.

2. **Use rate limiting or account protection** for SSH. For example, `fail2ban` can block or slow down repeated failed login attempts.

3. **Create a detection for failures followed by a successful login.** This could give higher priority to cases where several failed attempts are followed by a successful authentication.

4. **Use `auditd` for better command visibility.** SSH logs show authentication and session information, but they do not always show the commands run after login.

5. **Improve alert grouping.** Related alerts from the same source should be grouped properly so analysts do not have to investigate multiple incidents for the same activity.

## 9. Lessons Learned

* I learned that the rule condition and the investigation findings are two different things. The rule detected the failed logins, and i found the successful login while investigating.

* I learned that log rows do not always equal the actual number of events because rsyslog can combine repeated messages.

* I learned that detection latency is also useful to measure. In this case, the incident appeared about 13 minutes after the last failed login.

