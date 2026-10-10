# Day 12

## 1. What I built

Today i built a new Microsoft Sentinel rule called `SSH Login Success After Repeated Failures`. It detects a successful SSH password login when there were 3 or more failed attempts from the same IP and account in the previous 10 minutes. The query has a success side and a failure side, joined using `SourceIP` and `TargetUser`, and keeps each failed event as a separate row. The rule runs every 5 minutes, looks back 20 minutes, has High severity, maps IP and Account entities, and uses MITRE T1110.001 and T1078.

## 2. A design mistake I corrected

My first idea was to keep only the time of the last failed login for each IP and account. This was a mistake because it could miss a successful login that happened before the last failure. I fixed the query by keeping each failure as its own row and counting the failures in the 10 minutes before each successful login.

## 3. Testing it

At first, the query matched old data and returned 3 matches: one from 7 October with 8 failures and two from 5 October with 3 failures each. Logins with only 1 or 2 failed attempts before them were skipped because they did not meet the minimum threshold of 3. After that, i generated a fresh attack from my Mac, and the new rule created Incident 7 with High severity.

## 4. What I observed in the results

The new rule had about 8 minutes of latency: the successful login happened at 18:36 UTC and the incident was created at 18:44 UTC. This was faster than the failures-only rule, which took about 13 minutes. One successful login produced 3 alerts and then 4 because the scheduled rule used overlapping lookback windows, but alert grouping merged them into one incident. The old rule did not have grouping and created duplicate incidents 6 and 8, so both rules firing on the same attack resulted in two separate incidents.

## 5. What I closed and why

I classified Incidents 5, 6, 7 and 8 as **Benign Positive** because they came from my authorised security testing. The rules detected the login patterns correctly, but the activity was generated deliberately from my Mac in the lab. I resolved the incidents after reviewing the evidence and confirming that the activity was expected.

## 6. Problems I hit

The MITRE technique picker did not show T1078 at first, so i searched by the technique name and then found it. Another issue was that Incident 7 was initially resolved without a classification. Because of this, the query showed the classification as `Undetermined` until i corrected it.

## 7. Open items / next steps

I need to filter successful logins using `ingestion_time()` to reduce duplicate alerts caused by overlapping lookback windows. I also need to look at how to link alerts from both rules into one incident when they relate to the same attack. Tomorrow i will work on the sudo-abuse detection as part of Day 13.

## 8. What I learned

I learned that keeping each failed login as a separate row helps detect a successful login that follows repeated failures. I also learned that overlapping scheduled queries can create duplicate alerts, so the query and incident grouping both need to be considered. Finally, i learned that checking the incident classification is part of completing an investigation.

