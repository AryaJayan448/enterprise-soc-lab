# Day 13 - Sudo Abuse Detection (Linux)

## 1. Goal

Today I built a Microsoft Sentinel rule to detect suspicious sudo commands in Linux logs. This matters because an attacker may use sudo to gain higher privileges, access sensitive files or run commands as root.

## 2. What sudo writes to the logs

I saw three types of sudo log entries:

* `COMMAND=` — records the command that was run.
* `session opened` — records when the sudo session starts.
* `session closed` — records when the sudo session ends.

The rule uses `COMMAND=` because it contains the actual command text. One sudo command can produce three log rows, so matching session rows as well could count the same activity more than once.

## 3. Baseline: what normal looked like

I found 6 sudo commands in the lab, all run by `arya-labadmin`.

* `grep 'Failed password' /var/log/auth.log` — 3 times
* `/usr/sbin/poweroff` — 2 times
* `grep sshd /var/log/auth.log` — 1 time

Alerting on every sudo command would create too much noise because normal administration also uses sudo. The rule focuses on specific patterns that may indicate suspicious activity.

## 4. Detection logic

The rule checks two groups of suspicious patterns:

* **Sensitive files and user changes:** commands involving `/etc/shadow`, `/etc/sudoers`, `useradd`, `usermod`, `chpasswd` and `visudo`.
* **Root shell:** commands that start a shell with elevated privileges, including patterns for `bash`, `sh` and `su`.

The KQL parts have different jobs:

* `extract()` pulls the command text from the log message.
* `matches regex` checks whether the command matches one of the suspicious patterns.
* `project` selects the fields needed to review the results.

The regex uses `(usr/)?` because `usr/` is optional in the executable path. For example, the `sudo -i` command I ran was logged as `/bin/bash`. The optional path component helps the rule match paths with or without `usr/`.

## 5. Testing

| Command run               | Logged as                     | Matched pattern |
| ------------------------- | ----------------------------- | --------------- |
| `sudo ls -l /etc/shadow`  | `/usr/bin/ls -l /etc/shadow`  | `/etc/shadow`   |
| `sudo ls -l /etc/sudoers` | `/usr/bin/ls -l /etc/sudoers` | `/etc/sudoers`  |
| `sudo -i`                 | `/bin/bash`                   | Root shell      |

Before testing, the query returned 0 rows. After running the three test commands, it returned 3 rows. This showed that the tested commands matched the detection patterns.

One thing I learned is that the command recorded in the log can look different from what I typed, so the rule needs to match the logged command text.

## 6. Analytics rule settings

| Setting              | Value                             | Why                                                                                        |
| -------------------- | --------------------------------- | ------------------------------------------------------------------------------------------ |
| Frequency / lookback | 5 minutes / 15 minutes            | The rule runs every 5 minutes and checks a wider window for recent events.                 |
| Severity             | Medium                            | These commands can be suspicious, but they can also be used for legitimate administration. |
| MITRE                | T1548.003 — Sudo and Sudo Caching | Relates to abusing sudo to gain elevated privileges.                                       |
| Event grouping       | Alert per event                   | Keeps matching events available for investigation.                                         |
| Entities             | Account, Host                     | Helps identify which account ran the command and which system was involved.                |

## 7. Detection latency

| Step                     | First run (UTC)                     | Retest (UTC)               |
| ------------------------ | ----------------------------------- | -------------------------- |
| Command run              | 12:19:20 (second command: 12:19:28) | 13:11:35                   |
| Row ingested into Syslog | 12:20:39                            | 13:12:38                   |
| Alert created            | 12:27:23.206                        | 13:20:41.619               |
| Incident created         | 12:27:38.110 (Incident 9)           | 13:20:55.963 (Incident 15) |

For the first run, ingestion took about 1 minute 19 seconds. The alert was created 6 minutes 44 seconds after ingestion, and the incident followed about 15 seconds later. The total time from the first command to incident creation was approximately 8 minutes 18 seconds.

For the retest, ingestion took about 1 minute 3 seconds. The alert was created 8 minutes 4 seconds after ingestion, and the incident followed about 14 seconds later. The total time from the retest command to incident creation was approximately 9 minutes 20 seconds.

Both runs took longer than the rule's 5-minute schedule interval from ingestion to alert creation. The exact delay depends on when an event arrives relative to the scheduled run, along with query execution and processing time.

## 8. False positives and duplicate-alert fix

A legitimate administrator running `sudo -i` is not automatically malicious. The rule correctly detects the root-shell pattern, so it is a true positive for the detection logic, but a benign positive when the activity is authorised.

The original rule created 6 incidents from 2 events. The lookback was 15 minutes and the rule ran every 5 minutes, so each event could match 3 scheduled runs. That resulted in 2 events × 3 runs = 6 incidents.

I added an `ingestion_time()`-based deduplication fix. After the fix, the retest produced 1 incident from 1 event. I kept the 15-minute lookback instead of shortening it because late-arriving events could otherwise be missed.

Two tuning ideas:

* Allow-list specific administrator accounts only on approved hosts, and review matches outside that scope.
* Raise the severity when the same account or source has also had repeated failed SSH logins in the previous hour.

## 9. Limitations

The rule only detects commands covered by its regex. An attacker could use an unlisted interpreter, such as `python3 -c`, or open an editor with sudo and start a shell from inside it.

I have not tested the relative-path scenario, but based on the log format I expect the rule could miss a command such as `cd /etc && sudo ls -l shadow`. The directory may appear in `PWD=/etc`, separately from `COMMAND=/usr/bin/ls -l shadow`. Since the current rule extracts the command text and ignores `PWD=`, the `/etc/shadow` pattern may not match.

The sudo log message does not provide the source IP address. To investigate where the activity came from, I would pivot to the `sshd` events and correlate them by account and time.

I would also add `auditd` with suitable audit rules to collect more detailed process and command activity. This would help investigate activity that the current Syslog-based pattern misses.

## 10. Interview questions

**1. Why did you alert on specific sudo patterns instead of every sudo command?**

Administrators use sudo for normal tasks, so alerting on every sudo command would create too much noise. I focused on sensitive files, user-management commands and root shells because these patterns are more useful to investigate.

**2. Your rule created 6 incidents from 2 events. What happened and how did you fix it?**

The rule had a 15-minute lookback and ran every 5 minutes, so each event could match 3 runs, giving 6 incidents from 2 events. I added an `ingestion_time()`-based deduplication fix, and the retest produced 1 incident from 1 event. I kept the longer lookback because shortening it could miss late-arriving events.

**3. What are two ways an attacker could get past this rule, and what would you add?**

They could use an unlisted interpreter such as `python3 -c`. I also expect a relative-path command could evade the `/etc/shadow` pattern if the directory is recorded in `PWD=` separately from the command text, although I have not tested this yet. I would expand and test the patterns, consider the working directory when checking commands, add `auditd` for more detailed process activity, and pivot to SSH logs when investigating the source IP.
