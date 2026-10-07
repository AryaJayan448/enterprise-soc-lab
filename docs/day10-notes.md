# Day 10

Today i worked on creating a brute-force detection in Microsoft Sentinel for repeated failed SSH logins.

### What I built

First i created a KQL query to find failed SSH login attempts.

I used `extract()` to get the source IP and username from the SSH log message. Then i counted the failed logins from each source using `summarize`.

I also added `ProcessName == "sshd"` so the query only counts real SSH login events and does not count other commands that i ran while checking the logs.

The detection looks for **5 or more failed logins from the same source within 10 minutes**.

I then created a scheduled analytics rule in Sentinel. The rule runs on a schedule and looks back over the configured time window. I mapped the source IP as an entity so the IP can be used during investigation.

I also mapped the detection to **MITRE ATT&CK T1110.001 – Password Guessing**, under the **Credential Access** tactic.

### What went wrong

At first, i was only generating 3 failed logins in each test round. The rule threshold was 5, so it correctly did not create an alert. I had to generate enough failed logins to reach the threshold.

My Mac also went to sleep during testing, which paused the VM. This stopped the test activity until i started working again.

I first used `bin()` to group the failed logins into time buckets. I found that a burst of failed logins could be split across two bucket boundaries, which could make the count look lower. I removed `bin()` and used the analytics rule's lookback window instead.

Another problem was that the query was counting some of my own `sudo grep` commands because they appeared in the logs. I fixed this by adding `ProcessName == "sshd"` so the query only counts SSH events.

The IP extraction pattern originally only handled IPv4 addresses. I changed the pattern so it can also accept IPv6 addresses.

### Result

After fixing the query and generating enough failed logins, the detection fired and created incidents in Sentinel.

The portal grouped related alerts for the same source IP into incidents, which is why i saw **3 incidents**. Each had a **priority score of 2**. They were hidden at first because of the default filter in the incidents view.

The rule was working as expected and was detecting the repeated failed SSH login pattern.

### Classification

I classified the detection as a **true positive, benign**.

The rule fired correctly because the failed logins matched the detection. The evidence showed that the source IP was my own Linux lab VM, and i had deliberately generated the failed logins as part of my authorised testing.

So this was not a real attack.

### Open items

I still need to check the successful-login side of the investigation. I did not see any `Accepted password` lines in Sentinel, so on Day 11 i will compare this with the Ubuntu `auth.log` to confirm whether any successful login followed the failed attempts.

Another open item is that the same burst can create repeated alerts. I will look at tuning the detection on Day 15 to reduce repeated alerts and unnecessary noise.

