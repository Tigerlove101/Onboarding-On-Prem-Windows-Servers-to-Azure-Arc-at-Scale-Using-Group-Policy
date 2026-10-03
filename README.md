# Azure Arc Onboarding: Sending On-Prem Windows Logs to Microsoft Sentinel

Onboarding domain-joined Windows machines to Azure Arc with Group Policy, then collecting their Security event logs in Microsoft Sentinel through the Azure Monitor Agent.

This project covers the **telemetry pipeline**. The detection rules built on top of this data are documented in the [Detection Engineering project](../detection-engineering) (update this link to your repo).


<img width="2420" height="1540" alt="image" src="https://github.com/user-attachments/assets/e4b612df-bb6d-48c9-90d5-aeb32c783186" />


## Objective

Build a reliable, repeatable way to get endpoint logs from an on-prem Active Directory environment into Sentinel, and verify the data is complete enough for detection work.

## Environment

| Component | Role |
|---|---|
| MOCORPERATE.COM | On-prem Active Directory domain |
| Mo-svr | Domain controller, Group Policy management |
| MO1, MO2 | Domain-joined Windows endpoints, Arc-connected |
| Azure Arc | Connects on-prem machines to Azure as managed resources |
| Azure Monitor Agent (AMA) | Collects Windows event logs |
| Data Collection Rule (DCR) | Defines which events are collected and where they go |
| Log Analytics workspace | Log storage (`SecurityEvent` table) |
| Microsoft Sentinel | SIEM consuming the workspace data (accessed through the Microsoft Defender portal) |

## Data flow

1. A GPO on `Mo-svr` deploys the Azure Arc agent to domain-joined machines.
2. Machines register with Azure Arc over outbound HTTPS (443) using a service principal.
3. Arc-connected machines receive the Azure Monitor Agent and a Data Collection Rule.
4. Windows Security events flow into the Log Analytics workspace.
5. Sentinel reads from the workspace.

## Prerequisites

- Azure subscription, resource group, Log Analytics workspace with Microsoft Sentinel enabled
- Service principal with the **Azure Connected Machine Onboarding** role
- Domain-joined machines with outbound HTTPS access to Azure
- A network share readable by the target machines, for the onboarding package

## Implementation

### 1. Azure environment
Created the resource group and Log Analytics workspace, then enabled Microsoft Sentinel on the workspace.
<img width="944" height="401" alt="Screenshot 2026-10-03 002522" src="https://github.com/user-attachments/assets/8d6ccd62-d210-442f-8125-b9dfdd280523" />

<img width="944" height="421" alt="Screenshot 2026-10-03 002829" src="https://github.com/user-attachments/assets/fec5f892-25d1-491d-9f50-3430815a8d04" />

### 2. Service principal and onboarding script
Created a service principal limited to the onboarding role and generated the multi-server onboarding script from Azure Arc.
<img width="916" height="452" alt="Screenshot 2026-10-03 003245" src="https://github.com/user-attachments/assets/d753cdab-4869-45db-a404-878b2aa932e0" />
The script are generated automatically in the process. Service principal client secret is added before running the script on the AD DC. 

### 3. Group Policy deployment
- Placed the onboarding package on a network share
- Created the GPO on `Mo-svr` and linked it to the workstation OU
- Applied with `gpupdate /force` and verified with `gpresult /r`

<img width="1920" height="1012" alt="VirtualBox_win-svr_03_10_2026_00_43_55" src="https://github.com/user-attachments/assets/b963f5a6-4e13-4df3-b32c-608717f14daf" />


### 4. Verify Arc registration
Confirmed MO1 and MO2 show as **Connected** in Azure Arc > Machines.
<img width="952" height="511" alt="Screenshot 2026-09-28 194144" src="https://github.com/user-attachments/assets/8574bb24-0ee8-42c3-b7a2-fb92b911c668" />


### 5. Log collection
Created a Data Collection Rule for Windows Security events and associated it with the Arc machines.
<img width="946" height="459" alt="Screenshot 2026-10-02 220301" src="https://github.com/user-attachments/assets/0c64d459-a3f4-4d02-a373-76bdf6d6eb1b" />


### 6. Validate ingestion
```kql
SecurityEvent
| where TimeGenerated > ago(1h)
| summarize Events = count() by Computer, EventID
| order by Events desc
```
All Arc machines report into the `SecurityEvent` table.
<img width="950" height="473" alt="Screenshot 2026-10-03 004956" src="https://github.com/user-attachments/assets/a591f0de-b4dc-4f67-a553-bc1d3f3ec61b" />


Events collected: 4624, 4625, 4720, 4769. etc.

## Issues encountered and fixes

### Issue 1: GPO applied but the Arc agent did not install
- **Symptom:** The GPO reported success but machines did not appear in Azure Arc.
- **Investigation:** gpresult, share permissions, script path, service principal, network access, event logs.
- **Root cause:** Wrong configuration during arc script onboarding wrong path to the shared folder.]
- **Fix:** corrected the syntax error and it got fixed and script ran succesfully.

### Issue 2: Logs arrived in the wrong table
- **Symptom:** MO1 and MO2 were collecting through the generic Windows Event Log data source, so Security events were not in the `SecurityEvent` table that detection rules query.
- **Fix:** [describe the change, for example switching to the Windows Security Events via AMA collection and associating the DCR with each machine]
- **Result:** All Arc machines now report into `SecurityEvent`.

## Verification checklist

| Check | Result |
|---|---|
| Machines show Connected in Azure Arc | [ ] |
| AMA extension installed on each machine | [ ] |
| DCR associated with each machine | [ ] |
| `SecurityEvent` returns events from every machine | [ ] |
| Expected event IDs present (4624, 4625, and others) | [ ] |

## Security considerations

- Service principal secret is not stored in the repo or the GPO share beyond what onboarding requires, and is rotated or removed after onboarding
- Tenant and subscription IDs redacted in screenshots
- Least-privilege onboarding role
- DCR scoped to the events needed, which also controls ingestion cost

## Outcome

Domain-joined endpoints are onboarded through Group Policy and send Security events to Microsoft Sentinel. This pipeline is the data source for the (https://github.com/Tigerlove101/Detection-Engineering/tree/main)

## Skills demonstrated

Azure Arc · Azure Monitor Agent · Data Collection Rules · Group Policy · Active Directory · Log Analytics · Microsoft Sentinel · Troubleshooting log ingestion · Least-privilege access
