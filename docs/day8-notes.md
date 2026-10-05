# Day 8

Today i worked on setting up Microsoft Sentinel and researched Splunk first.

### Splunk and ARM64

First i researched Splunk documentation because i wanted to use it as the SIEM for my lab. I found that **Splunk Enterprise does not have an ARM64 build. Only the Splunk Universal Forwarder has an ARM64 build.**

So i decided to use Microsoft Sentinel for this part of my lab. I am not dropping Splunk completely because i plan to use a Splunk Cloud trial later.

### What I set up

I created an Azure resource group called `rg-soc-lab`.

Inside it i created a Log Analytics workspace called `law-soc-lab` in **North Europe** and added Microsoft Sentinel on top of the workspace.

The Log Analytics workspace is where the logs and data are stored. Sentinel uses this data for security monitoring, alerts and investigations.

The Azure portal is also moving Microsoft Sentinel into the Defender portal, so this is something i noticed while working with Sentinel.

My Sentinel free trial is from **5 October 2026 to 5 November 2026** and the free trial has a **10 GB/day limit**.

### Cost safeguards

Before sending logs to Azure, i wanted to make sure i would not create unexpected costs.

I set up a budget alert so i can get a warning if the spending reaches the amount i set. The budget alert only gives a warning. It does not stop the spending.

I have not sent any data yet. I will also keep an eye on the 10 GB/day free trial limit.

When i finish the lab, i plan to delete the `rg-soc-lab` resource group so the Azure resources are removed.

The main thing i learned today is how Sentinel and Log Analytics work together, and why it is important to check the cost before sending logs to a cloud SIEM.

