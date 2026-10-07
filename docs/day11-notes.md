# Day 11

Today i closed the open item from Day 10 and investigated the SSH brute-force detection in Microsoft Sentinel.

### Closing the open item

On Day 10 i did not see any `Accepted password` events in Sentinel, so i wanted to check if there was a collection problem.

I compared the Ubuntu `auth.log` with the data in Sentinel. The same `Accepted password` lines were present in Sentinel, so there was no collection gap. The successful SSH login was being collected correctly.

### The attack from the Mac

The failed SSH logins came from `192.168.64.1`, which is my Mac and the admin workstation i used for the lab.

I generated the failed login attempts as part of the security testing. The attempts were against the `arya-labadmin` account.

The rule detected the repeated failures because there were enough attempts from the same source within the 10-minute detection window.

### Investigation

I started with the alert evidence in Sentinel. I checked the source IP, username, number of failures, and the first and last times of the activity.

The alert showed 8 counted failures from `192.168.64.1` for `arya-labadmin`.

Looking at the raw Syslog rows in Sentinel, i saw that rsyslog had collapsed two repeated events into one `message repeated 2 times` line. I knew i had typed 9 passwords, so the real count was 9 failed attempts even though the rule counted 8 rows.

I then used a pivot query that listed **all `auth` and `authpriv` events from 20:49 to 21:10 UTC**. I used this to check what happened after the successful login instead of only looking at the alert itself.

After that i checked the post-login activity. The `Accepted password` event was present, the SSH session opened, and then the session closed after about 11 seconds. I did not find any sudo activity, other sessions, or activity against other accounts.

### Detection latency

The last failed login happened at **20:49:08 UTC**.

The Sentinel incident was created at **21:02:19 UTC**.

This means the incident appeared about **13 minutes after the last failed login**.

This is useful because detection latency is something i should measure when testing a real detection. A detection can work correctly but still take too long to create an incident.

### What i would improve

I would improve the detection by creating another rule for **failed logins followed by a successful login**. This could make the alert more important because a successful login after several failures can be more suspicious.

I would also improve command visibility. The SSH logs showed that the session was opened and closed, but they did not show exactly what commands were run. Using `auditd` could give more information about activity after login.

I would also look at alert grouping and detection latency to reduce duplicate incidents and make the alerts appear faster.

### What i learned

I learned that the incident's **First activity and Last activity times are based on the rule's lookback window**, so i should use the actual log events to build the timeline instead of using those times as the exact event times.

I also learned that Defender shows **local time**, while Sentinel Logs shows **UTC**. I initially thought some older Day 10 evidence was from the new attack, but checking and converting the times showed that it was from the earlier activity.

The main thing i learned today is that investigating an alert is more than checking why the rule fired. I need to look at the actual events before and after the alert, compare the SIEM data with the original logs, and make sure the timestamps are from the same timezone.

