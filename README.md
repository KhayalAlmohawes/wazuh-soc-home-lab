# Wazuh SOC Home Lab: Detecting and Investigating Repeated Failed Logons

A small SOC lab built in VirtualBox. I deployed Wazuh, enrolled a Windows 11 endpoint, simulated repeated failed logons on it, and investigated the resulting alerts the way a Tier 1 SOC analyst would. The work ends with a written incident report.

## Lab architecture

<img width="1800" height="1440" alt="image" src="https://github.com/user-attachments/assets/2d1fb544-1a79-4db1-ae40-e38df190caf4" />

| VM | Role | OS | IP |
|---|---|---|---|
| Wazuh-Server | Wazuh manager, indexer, dashboard (Wazuh 4.14.9, all-in-one) | Ubuntu Server 24.04 | 192.168.56.10 |
| Windows-Employee | Monitored endpoint with Wazuh agent (ID 001) | Windows 11 Home | 192.168.56.20 |

- **Adapter 1 (NAT):** internet access and port forwarding to reach the dashboard from the host (`https://127.0.0.1:8443`).
- **Adapter 2 (Host-only, 192.168.56.0/24):** traffic between the two VMs.

<img width="2712" height="1716" alt="image" src="https://github.com/user-attachments/assets/0776e1f0-8b14-4923-9771-82426f755ba3" />

## What I did

1. Installed the Wazuh server and enrolled the Windows agent.
2. Confirmed the agent was active and sending events.
3. In my own lab, entered a wrong password repeatedly on the Windows sign-in screen (6 attempts).
4. Found the alerts in Wazuh Threat Hunting and investigated one event field by field.
5. Extracted indicators, mapped the activity to MITRE ATT&CK, and wrote an incident report.

## Detection and investigation

**Agent connected**

<img width="965" height="541" alt="image" src="https://github.com/user-attachments/assets/f5c53d01-41d8-4fbd-94f1-5477a70f1486" />

**Dashboard: 6 authentication failures**

<img width="698" height="388" alt="image" src="https://github.com/user-attachments/assets/10606863-a97d-4c2e-a7f2-7fc567fbed22" />

**Filter `rule.groups: authentication_failed` returns 6 events**

<img width="1366" height="646" alt="image" src="https://github.com/user-attachments/assets/a32e81ae-beb5-4b5c-99c6-4a687f2f3ac1" />

### Key findings

| Item | Value |
|---|---|
| Rule | 60122, "Logon Failure - Unknown user or bad password" (level 5) |
| Windows event | 4625 (failed logon) |
| Target account | Employye (local account) |
| Source | 127.0.0.1, logon type 2 (interactive, at the console) |
| Failures | 6, between 00:24:01 and 00:25:05 (Oct 31, 2026, Asia/Riyadh) |
| Failure reason | status 0xC000006D, subStatus 0xC000006A (wrong password for an existing account) |

**How I read it:** the failures came from the local machine (logon type 2), not from the network (type 3), so there is no sign of a remote attack. Six attempts in about one minute is a small burst, not a sustained brute-force campaign.

### MITRE ATT&CK

- Wazuh tags rule 60122 as **T1531 Account Access Removal** (Impact).
- The simulated pattern is closer to **T1110 Brute Force** (Credential Access), but six attempts is too few to call it a campaign. The report records both.

### Alert detail
<img width="488" height="694" alt="image" src="https://github.com/user-attachments/assets/bce0d020-e869-42d4-aa76-808e13211979" />

<img width="540" height="700" alt="image" src="https://github.com/user-attachments/assets/be1280f7-4eb7-442b-8814-96ec20fa54a8" />

<img width="604" height="364" alt="image" src="https://github.com/user-attachments/assets/48e74f08-be7f-4463-9172-809cd7cc9189" />

## Recommendations (from the report)

- Set an account lockout threshold.
- Alert on repeated failures from the same account in a short window.
- Prioritize failures with logon type 3 or an external source IP.
- Enable MFA where possible.

## Full incident report

[Incident Report: Repeated Failed Logons on Windows-Employee](Incident-Report.pdf)
## Skills practiced

Wazuh deployment and agent enrollment, Windows Security event analysis (4625), alert triage, log field analysis, IOC extraction, MITRE ATT&CK mapping, incident reporting.

## Limitations and next steps

- Single endpoint and a small simulated event; this shows the workflow, not production-scale detection.
- Next: Sysmon on the endpoint, a custom detection rule, a simulated new-user (persistence) scenario, and File Integrity Monitoring.

> All simulations were done inside my own VirtualBox lab, on machines I own.
