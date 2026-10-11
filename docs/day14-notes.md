# Day 14 - Windows Logs into Microsoft Sentinel (ARM64)

## 1. Goal and the problem

Today I connected my Windows 11 ARM64 VM to Microsoft Sentinel and verified that Windows security events were being collected. The Azure Arc Connected Machine agent route I used for Linux was not supported for this Windows 11 ARM64 setup, so I used the Azure Monitor Agent client installer instead.

## 2. What I verified in the docs

I checked Microsoft's documentation to confirm which agent installation route was appropriate for Windows 11 ARM64.

| Agent route                                | Windows 11 ARM64                                      | Result   |
| ------------------------------------------ | ----------------------------------------------------- | -------- |
| Azure Arc Connected Machine agent          | ARM64 listed as unsupported for the Windows 11 client | Not used |
| Azure Monitor Agent client installer (MSI) | Supported for Windows 11 ARM64                        | Used     |

Microsoft Learn pages checked:

1. **Connected Machine agent prerequisites** — https://learn.microsoft.com/en-us/azure/azure-arc/servers/prerequisites
2. **Azure Monitor Agent supported operating systems and environments** — https://learn.microsoft.com/en-us/azure/azure-monitor/agents/azure-monitor-agent-supported-operating-systems

The documentation helped me choose the client MSI installation route rather than trying to use the Azure Arc Connected Machine agent on the ARM64 VM.

## 3. Linux vs Windows ingestion

|                 | Linux (Day 6–8)                                           | Windows (today)                                                   |
| --------------- | --------------------------------------------------------- | ----------------------------------------------------------------- |
| Identity        | Azure Arc-enabled machine using a managed identity        | Microsoft Entra-joined Windows client using device authentication |
| Agent install   | Azure Arc Connected Machine agent and Azure Monitor Agent | Azure Monitor Agent client installer (MSI)                        |
| DCR attached to | Azure Arc machine resource                                | Tenant-level monitored object for the Windows client installer    |

The main difference was how the machine authenticated and how the data collection rule was associated. Linux used the Azure Arc machine resource and its managed identity. Windows used Microsoft Entra device authentication and a monitored object for the DCR association.

## 4. Steps and checkpoints

### Checkpoint A — Microsoft Entra join

I created a separate member user named `winlab` because my Gmail-backed account was an external account and could not join the device in the required way.

I verified the device registration using `dsregcmd`. The output showed:

* `AzureAdJoined : YES`
* `DeviceAuthStatus : SUCCESS`

This confirmed that the Windows VM was Microsoft Entra-joined and that device authentication succeeded.

### Checkpoint B — Azure Monitor Agent running

I installed `AzureMonitorAgentClientSetup.msi`, approximately 91 MB in size, using `msiexec /qn`.

I then checked the service using:

```powershell
Get-Service AzureMonitorAgent
```

The service status was **Running**, confirming that the Azure Monitor Agent client was installed and running.

### Checkpoint C — DCR and data collection

For the cloud configuration, I enabled **Access management for Azure resources**, assigned the **Monitored Objects Contributor** role, and created the following resources:

* Data collection rule: `dcr-windows-events`
* DCR kind: Windows
* Region: North Europe
* Monitored object: created for the Windows client
* Association: `assoc-windows-events`
* API version: `2021-09-01-preview`

I attached the DCR to the monitored object and configured the required Windows event sources.

The DCR collected:

* **Security events:** 4624, 4625, 4688, 4719, 4720 and 4732, sent to `SecurityEvent`.
* **Sysmon events:** Event IDs 1 and 3, sent to `Event`.
* **PowerShell logging:** Event ID 4104, sent to `Event`.

I verified the collection by querying the workspace and finding Windows security, Sysmon and PowerShell events in Sentinel.

## 5. Results

| Check                    | Result                                                                               |
| ------------------------ | ------------------------------------------------------------------------------------ |
| Heartbeat                | One heartbeat per minute from `WIN-58N5TIST71S`, with Category `Azure Monitor Agent` |
| SecurityEvent 4688       | 27 events                                                                            |
| SecurityEvent 4625       | 1 event initially                                                                    |
| SecurityEvent 4624       | 1 event                                                                              |
| Sysmon Event ID 1        | 31 events                                                                            |
| Sysmon Event ID 3        | 1 event                                                                              |
| PowerShell Event ID 4104 | 3 events                                                                             |

The heartbeat confirmed that the Windows client was communicating with the monitoring service. The `SecurityEvent` and `Event` tables confirmed that the configured Windows security, process, network and PowerShell telemetry was arriving in the workspace.

## 6. Problem found: events missing

During the initial test, I generated six failed logon events, but only one appeared in Sentinel, at **09:57:21 UTC**.

This is consistent with the agent applying the DCR at about that time, although I did not verify the timing directly in the agent logs. Events generated before the DCR became active were not back-filled automatically.

After the DCR was applied, I generated six new failed logon events. All six arrived in Sentinel between **10:12:44 and 10:12:57 UTC**, bringing the total number of failed logon events in the results to seven.

The retest showed these values:

* `EventID`: 4625
* `Account`: `-\fakeuser`
* `IpAddress`: `127.0.0.1`
* `LogonType`: 3
* New events received: 6
* Total failed logon events after retesting: 7

This confirmed that collection worked for new events after the DCR was applied. The missing events were consistent with the collection configuration not yet being active when the initial events were generated.

**Lesson learned:** events generated before the agent has applied its DCR should not be assumed to be back-filled. After configuring a new collection route, I need to verify the DCR association and generate fresh test events to confirm ingestion.

## 7. Security notes

I temporarily enabled **Access management for Azure resources** by setting it to **Yes**, then returned it to **No** after completing the elevated-access step.

The **Monitored Objects Contributor** role is still assigned to my own account in the lab tenant, which has two users (my account and the `winlab` lab user). Removing this assignment requires briefly re-enabling elevated access. I plan to remove the role at the end of the project and then verify that the assignment has been removed.

**Security defaults remained enabled** throughout the lab. I was not prompted for MFA during this sign-in, so I cannot claim that MFA was completed as part of this step.

I did not include passwords, tenant IDs or authentication secrets in the notes, screenshots or repository.

## 8. Limitations

1. **Preview API:** the monitored-object association used API version `2021-09-01-preview`. Preview APIs can change, so the configuration may need updating if Microsoft changes the supported workflow.
2. **Tenant-level scope:** the Windows client installer uses tenant-level monitored-object and DCR association. This can affect client devices in the tenant using this installation route, so the scope must be understood before using it outside a lab.
3. **Lab-only validation:** I tested the setup with an Entra-joined Windows 11 ARM64 VM. The successful ingestion results demonstrate the configuration in this environment, but do not prove that every production device or tenant configuration will behave identically.
4. **No automatic historical backfill:** events generated before the agent had applied its DCR were not recovered automatically. Monitoring and testing must account for when the collection configuration becomes active.

## 9. Interview line

I connected a Windows 11 ARM64 VM to Microsoft Sentinel using the Azure Monitor Agent client installer, configured a tenant-level monitored object and DCR, verified security and Sysmon telemetry, and explained and verified a missing-event issue by confirming that six new failed logon events arrived after the DCR was applied.
