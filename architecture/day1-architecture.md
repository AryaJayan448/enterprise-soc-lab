+--------------------+
| Windows 11 VM      |
| Ubuntu VM          |
+--------------------+
          |
          v
+--------------------+
| Log Collection     |
| Sysmon             |
| Windows Logs       |
| Linux Logs         |
+--------------------+
          |
          v
+--------------------+
| Log Forwarding     |
| AMA / Forwarder    |
+--------------------+
          |
          v
+-------------------------------+
| SIEM                          |
| Microsoft Sentinel            |
| Splunk Enterprise             |
+-------------------------------+
          |
          v
+-------------------------------+
| Detection                     |
| KQL Rules                     |
| SPL Queries                   |
+-------------------------------+
          |
          v
+-------------------------------+
| SOC Analyst                   |
| Alerts                        |
| Investigation                 |
| Incident Response             |
+-------------------------------+