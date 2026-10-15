# MITRE ATT&CK Coverage

## 1. What the five rules cover

| Rule | Platform | Tactic | Technique | Data source |
| --- | --- | --- | --- | --- |
| SSH brute force | Linux | Credential Access | T1110, T1110.001 | auth.log (sshd) |
| SSH success after failures | Linux | Credential Access / Initial Access | T1110.001, T1078 | auth.log (sshd) |
| Sudo abuse | Linux | Privilege Escalation | T1548.003 | auth.log (sudo) |
| Windows brute force | Windows | Credential Access | T1110, T1110.001 | Security 4625 |
| New local admin | Windows | Persistence | T1136.001, T1098 | Security 4720 + 4732 |

The five existing rules mainly cover credential attacks, privilege escalation, and persistence.

## 2. Gaps

Several other attacker activities are not directly detected by the current rules:

* Execution: Running suspicious scripts or commands, including PowerShell activity.
* Defense Evasion: Clearing Windows event logs or changing audit settings to hide activity.
* Lateral Movement: Using compromised credentials to access other machines over the network.
* Command and Control: Processes communicating with suspicious external destinations.
* Exfiltration: Sending sensitive data outside the environment.

These activities may generate logs, but the current five rules do not specifically detect them. This means an attacker could move beyond the initial account compromise without triggering one of the existing alerts.

## 3. Gaps with data but no rule

Some gaps are not missing data. They are missing detections. Based on the data collection rule (`config/dcr-windows-events.json`), these telemetry sources are already collected:

| Telemetry collected | Event IDs | Technique it could support | Rule today |
| --- | --- | --- | --- |
| Windows Security | 4719 (audit policy changed) | Defense Evasion: impair logging (T1562.002) | None |
| Windows Security | 4688 (process creation, command line enabled) | Execution (T1059) | None |
| PowerShell | 4104 (script block logging) | Execution: PowerShell (T1059.001) | None |
| Sysmon | 1 (process create), 3 (network connection) | Execution, Command and Control | None |

I already collect Sysmon Event ID 1 for process creation and PowerShell Event ID 4104 for script-block logging. I also enabled Windows process command-line auditing.

One useful gap to address is suspicious PowerShell execution. The available telemetry can provide the process name, command-line arguments, and PowerShell script content, but I do not yet have a dedicated rule to identify suspicious script execution.

For example, a detection could look for encoded PowerShell commands or suspicious combinations of command-line arguments. These patterns would need careful tuning because legitimate administration scripts can use similar techniques.

Gaps that need a new data source first: log clearing (Windows event 1102) is not in the data collection rule, so a detection would need that event added to collection before it could be built.

## 4. Which gap to close first

I would prioritise a detection for suspicious PowerShell execution because the required telemetry is already available, making it a practical next step without needing to deploy another data source.

I would start by hunting for suspicious command-line arguments and script-block patterns, then create a rule with a clear threshold or combination of indicators to reduce false positives. I would test it using controlled lab commands and confirm that the resulting alerts contain enough process and user context for investigation.

This would extend the lab beyond detecting failed logons and account changes to identifying potentially malicious activity after an attacker gains access.
