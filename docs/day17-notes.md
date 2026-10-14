# Day 17 - Re-Implementing Sentinel Detections in Splunk

## 1. Goal

Re-implemented four of the five existing Microsoft Sentinel detection rules in Splunk Search Processing Language (SPL) to compare detection logic across SIEM platforms.

The aim was to understand how KQL queries translate into SPL and verify whether the same exported security events could produce comparable detection results.

## 2. Platform choice

I chose the Splunk Cloud trial rather than installing Splunk Enterprise locally on my Mac.

The Splunk Cloud trial offered 14 days of access, up to 5 GB of data ingestion per day, and no credit card requirement. A local Splunk Enterprise installation would have required an approximately 841 MB download, and I had not confirmed a suitable ARM64 Docker image for this Mac. A native macOS ARM build exists in recent releases, but I preferred not to install an 841 MB package locally.

Using the cloud trial was the more practical way to complete the comparison within the lab timeframe.

## 3. How the data got into Splunk

I exported event data from Microsoft Sentinel as CSV files and uploaded those files into Splunk. The data was searched in the `main` index.

**This was a static CSV import, not a live data pipeline.** Splunk did not receive events directly from the lab machines or Sentinel in real time. The results demonstrate that the SPL detections worked against the exported data, not that a live integration had been configured.

## 4. Two export problems

I encountered two issues when preparing the Sentinel exports for Splunk.

### Timestamp format

Some exported timestamps used the `dd/MM/yyyy` format. This could be interpreted incorrectly as `MM/dd/yyyy`, potentially changing the date or causing parsing failures.

I corrected the export to use an ISO-style datetime format with `format_datetime`, making timestamps more consistent and easier to parse.

### Hidden byte-order mark (BOM)

The exported files also contained a hidden byte-order mark at the beginning of each file. A BOM could attach to the first column name, so I removed it as a precaution before uploading the corrected CSV files.

These fixes made the exported data more consistent for ingestion and searching in Splunk.

## 5. Data loaded

The three CSV exports contained the following numbers of events in the `main` index:

| CSV dataset (source)  | Events loaded |
| --------------------- | ------------: |
| `linux_ssh_sudo.csv`  |            76 |
| `win_4625.csv`        |            31 |
| `win_account.csv`     |            12 |
| **Total**             |       **119** |

These are the exported events loaded into Splunk for this lab, not a count of live events received from Sentinel.

## 6. Translating KQL to SPL

I mapped the main KQL operations to their SPL equivalents when rewriting the detections.

| KQL             | SPL                  | Purpose                                  |
| --------------- | -------------------- | ---------------------------------------- |
| `summarize`     | `stats`              | Aggregate and group events               |
| `make_set()`    | `values()`           | Collect distinct field values            |
| `extract()`     | `rex`                | Extract fields using regular expressions |
| `matches regex` | `regex`              | Filter events using a regular expression |
| `project`       | `table`              | Select fields to display                 |
| `bin`           | `bin _time span=...` | Group timestamps into time buckets       |

The syntax and behaviour are not identical in every case, so the SPL queries needed to be checked against the fields and timestamps available in the imported data.

## 7. Detection results

Four of the five Sentinel detections were re-implemented and tested against the exported CSV data. The queries are in `detections/spl/`.

| Detection               | SPL result                                                      |
| ----------------------- | --------------------------------------------------------------- |
| Windows brute force     | Four time buckets with failure counts of `6`, `8`, `8`, and `8` |
| SSH brute force         | Two bursts of eight failed attempts                             |
| Linux sudo abuse        | Six matching rows                                               |
| New local administrator | Four accounts detected                                          |

These results match the tests I ran in the lab: the Windows buckets correspond to the four failed-logon bursts, the sudo rule returned exactly the six sudo commands from Day 13, and the four accounts correspond to the four `labtest` runs. The sudo result is shown in `screenshots/day17-splunk-sudo.png`.

The results should be interpreted within the limits of the export: they demonstrate detection logic against the available historical data, not real-time alerting or end-to-end ingestion from the original systems.

The fifth rule, which detected successful SSH logons following failures, was not converted during this exercise.

## 8. A mistake I made

While writing the SPL time-bucketing query, I used `bin _time`, which modified `_time` to represent the beginning of each bucket. As a result, `FirstSeen` and `LastSeen` displayed the bucket start in epoch time instead of the original event timestamps.

I fixed this by saving the original `_time` value before applying the bucket operation, allowing the query to retain the original event time for the first-seen and last-seen calculations.

**Lesson:** When aggregating time-series data, preserve the original event timestamp before transforming it into a time bucket. Otherwise, the query can lose the detail needed to interpret when activity actually occurred.

## 9. Limitations

1. **Fixed time buckets can split a burst:** Events near a bucket boundary may be placed in separate buckets, potentially preventing a detection from reaching its intended threshold even when the events are close together.
2. **Static exported data:** The CSV import does not provide continuous ingestion, live detection, or end-to-end validation of a Sentinel-to-Splunk pipeline.
3. **One detection remains unconverted:** The SSH-success-after-failures rule was not re-implemented, so the exercise covered four of the five existing rules.

The results also depend on the completeness and timestamp accuracy of the exported events.

## 10. Interview line

I re-implemented four Microsoft Sentinel detections in Splunk SPL using exported CSV data, resolved timestamp and BOM import issues, compared the detection results across SIEM platforms, and corrected a time-bucketing mistake that affected first-seen and last-seen timestamps.
