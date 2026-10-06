# Day 9

Today i worked on connecting my Ubuntu VM to Microsoft Sentinel and sending Linux SSH logs to Azure.

### Architecture check

Before building anything, i read the Azure documentation to check the ARM64 support for Azure Arc and the Azure Monitor Agent.

Per the documentation, the **Azure Arc agent does not support Windows 11 ARM64**. The **Azure Monitor Agent documentation lists Windows 11 ARM64 as supported through the client installer**, but it has extra requirements.

I have not tested the Windows ARM64 setup yet, so this is still an open item.

Linux ARM64 is supported, so i used my Ubuntu VM for this part of the lab.

### The pipeline

The setup works like this:

`Ubuntu VM → Azure Arc → Azure Monitor Agent → Data Collection Rule → law-soc-lab → Syslog table → KQL`

Azure Arc connects the Ubuntu VM to Azure. The Azure Monitor Agent collects the logs based on the Data Collection Rule. The logs are then sent to my `law-soc-lab` workspace.

The Linux logs appear in the `Syslog` table, where i can use KQL to search and analyse them.

### What went wrong

First, i went to the Sentinel Data connectors page, but the connector list did not load. Because of this, i created the data collection rule directly in Azure Monitor.

After that, my first query showed some facilities that i had not selected.

The portal checkboxes looked unticked, which was misleading. I checked the JSON code and found that the rule actually had two data sources. One of them was collecting everything.

I fixed this in the JSON code editor by removing the extra data source block.

I still need to re-check that `daemon` and `cron` stopped arriving after the change.

### Cost controls

To keep the amount of data low, i configured the rule to collect only `auth` and `authpriv` facilities from `Info` level and above.

I also have the Azure budget alert and the Sentinel trial limit as extra cost controls.

### Testing SSH logs

I tested failed SSH logins and searched for them in the `Syslog` table using KQL.

I used `has_any` to find the failed SSH login messages.

This helped me find the failed SSH login events in Sentinel and practise searching the logs with KQL.

### What i learned

The main lesson today was to verify what is actually configured and what is actually being ingested.

The portal UI is not always enough. I need to check the JSON configuration and query the actual data to confirm what is happening.

This is similar to what i learned on Day 6 with `auditpol`. The setting shown in the UI is not proof that the logging is actually working. I need to check the real configuration and the logs.

