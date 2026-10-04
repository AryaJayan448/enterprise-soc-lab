# Day 6

Today i worked on Windows audit policy and Event ID 4625.

I enabled Process Creation auditing. It creates Event 4688 when a new process starts. I also enabled command line logging to see the command used. I enabled PowerShell Script Block Logging to see PowerShell commands.

Sysmon and Event 4688 both show process creation, but Sysmon gives more information for security monitoring.

First i checked `auditpol` and Logon auditing was Success and Failure. After i changed the policy in gpedit, failed logons were not showing. I checked the logs and there were no 4624 or 4625 events after 20:01. I found some Event 4719 events and checked `auditpol` again. It showed No Auditing. I turned Audit Logon back to Success and Failure and it started working again.

From Event 4625 i learned about Logon Type, Sub Status and Logon Process. Logon Type 2 means local login and Type 3 means network login. I saw `0xC000006A` in my test, which means the password was wrong. I did not see `0xC0000064` myself, but i learned that it means the username or account does not exist.

Many `0xC0000064` events can mean someone is checking different usernames. Many `0xC000006A` events for one account can mean brute force. If it happens for many accounts it can be password spraying. If the failed logons are followed by 4624, the password may have worked.

The main thing i learned today is to check `auditpol` after changing audit policy because the settings can change or reset.

