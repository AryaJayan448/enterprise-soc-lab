# Day 5

Today i started installing Sysmon in my Windows 11 ARM64 VM.

Sysmon is used to monitor Windows activities and processes. It helps to see more information for security monitoring.

First i tried `Sysmon64.exe` but it did not work and the driver was blocked. I thought maybe there was a problem with the file. I changed Vulnerable Driver Blocklist and Tamper Protection and tried to reach Secure Boot settings, but these were not the problem.

Then i found that the problem was the file architecture. My Windows VM is ARM64, so i used `Sysmon64a.exe` and it worked.

After installing Sysmon i checked Event ID 1. It showed `Image`, `ParentImage` and `User`. Image shows the process, ParentImage shows which process started it, and User shows the user.

I learned that ParentImage is important. If PowerShell starts `whoami.exe`, it is normal because i ran it. But if `winword.exe` starts `whoami.exe`, it can be suspicious because a Word document or macro may be running it.

The main thing i learned today is to check the architecture first and check which process started another process.

After finding the problem i turned Vulnerable Driver Blocklist and Tamper Protection back on.

