# Brute-force-attack-incident-report 
---

# Goal: 
Stimulate a real cyberattack (brute-force attack) in a home lab and investigate it like a real SOC analyst. 

I did:

- **Generate fake malicious activity**
- **Collect logs**
- **Detect suspicious behaviour**
- **Investigate alerts**
- **Write an incident report**

# A. LAB OVERVIEW

# I built a mini SOC environment:

**SIEM**: Wazuh SIEM & XDR

**Attacker**: Kail Linux

**Target**: Windows (Wazuh agent)

**Set up Wazuh (SIEM)**

- **Go to https://wazuh.com/**
- **Download Wazuh**
- **Install Wazuh manager on your virtual machine.**
- **Once installed, open Wazuh web: http://your-vm-ip**


**Set up Windows (Agent)**

- **Install windows as agent into the Wazuh agent**
- **Set the Audit logon event (check the box for failure)**
- **Disable NLA (Network Level Authentication)—Settings > System > Remote Desktop (Click the down arrow next to Remote Desktop), and uncheck the box “Require computers to use Network Level Authentication to connect.**

# Note: 
These settings are necessary so Wazuh (SIEM) can pick up the logs.

---

# B. Generate Attack Activity

Hydra executed a password brute-force attack against Windows RDP service on host **(Windows)**:

- **Command execution**: hydra -l Administrator -P rockyou.txt rdp://<target_IP>
- **Attack Source**: Kali Linux
- **Attack Target**: Windows machine
- **Protocol**: RDP
- **(MITRE ATT&CK T1110 — Brute Force)**

---

# Stimulating brute-force attack with Hydra

[![](<screenshots/Screenshot 2026-06-01 202704.png>)](<screenshots/Screenshot 2026-06-01 202704.png>)

---

# Incident Report 

[![](<screenshots/Screenshot 2026-06-01 162216.png>)](<screenshots/Screenshot 2026-06-01 162216.png>)

# Wazuh detecting a brute-force attack

**On June 1, 2026,** at approximately 15:02 UTC, the Wazuh SIEM platform detected a **high-severity alert triggering Rule 60204 (Multiple Windows Logon Failures, Level 10)** targeting an asset named MAZZI-NEW-HOST **(IP: 192.168.0.xxx)**. A downstream threat actor operating from a machine hostnamed **Kali** initiated a rapid, automated authentication assault targeting the local user account, **Mitch**.


The high frequency of authentication attempts within a tight four-second window strongly indicates a **brute force / credential guessing attack (Mitre Att&ck -1110)**. The attack was successfully blocked by system authentication guards, resulting exclusively in explicit **AUDIT_FAILURE events**. There is no evidence of successful lateral movement or data exfiltration.

# Key Questions Answered

- **What happened?** A rapid, automated network logon attack targeted a Windows endpoint, generating multiple consecutive authentication failures within seconds.

- **What was the attacker trying to do?** The attacker was attempting to crack the credentials of a local user account to gain unauthorized entry, establish a foothold, and perform lateral movement **(MITRE ATT&CK T1110 — Brute Force)**.

- **What account was used?** The target user account was **Mitch**.

- **What systems were affected?** The target endpoint **MAZZI-NEW-HOST (Windows Computer Name: Mitch)** running the Wazuh agent was affected by the traffic.

- **What tools and techniques were used?** The attacker utilized **a network NTLM authentication assault (Logon Type: 3, Protocol: NtLmSsp)**. The origin hostname indicates the use of a penetration testing distribution (Kali Linux) running automated credential spraying or dictionary tools.

- **What IP did the attacker use?** **192.168.0.xxx** (operating over the local loopback/internal subnet environment during the laboratory simulation).

- **What data, if any, was accessed?** None. Every tracked event resulted in sub-status code 0xC0000064 (user name does not exist or bad password). The attack failed.

[![](<screenshots/Screenshot 2026-06-01 205600.png>)](<screenshots/Screenshot 2026-06-01 205600.png>)

# Timeline of Events

---
# Indicators of Compromise (IoCs)

- **Source IP Address: 192.168.0.xxx**

- **Source Workstation Name**: kali

**Targeted Account**: Mitch

**Specific Error Code Signals**: * Windows Status: 0xC000006D (Logon failure value)

**Windows Sub-Status**: 0xC0000064 (Indicates wrong password/invalid identifier matching)

**Authentication Mechanism**: NtLmSsp / NTLM via network interface (Logon Type 3)

---

# Impact Assessment

- **Confidentiality**: No Impact. Authentication barriers successfully resisted access requests.

- **Integrity**: No Impact. No configuration modifications, binary changes, or unauthorized system writes occurred.

- **Availability**: Low Impact. The rapid log generation increases disk utilization slightly under high-volume log pipelines, but no denial-of-service conditions occurred on target server services.

# Containment & Actions Taken

- **Log Isolation and Triaging**: Incident response parsed the raw JSON strings delivered by decoder windows_eventchannel on index wazuh-alerts-4.x-2026.06.01 to confirm status fields.

- **Mitigation Verification**: I confirmed that no corresponding Windows Event ID 4624 (Successful Logon) occurred from source 192.168.0.107 during or immediately following the timeline window.

- **Host-Level Isolation (Simulated)**: In a live SOC production workflow, the active response engine would execute scripts to black-hole communication pathways with the attacker endpoint (Kali) via firewall filters.

# Recommendations

# Short-Term

- **Account Lockout Policies**: Implement or enforce strict account lockout thresholds in Windows Group Policy (GPO) to lock accounts temporarily after 5 consecutive bad attempts.

- **Network Isolation**: Block NTLM traffic from traversing external boundary perimeters; encourage switching to more robust authentication layers.

# Long-Term

- **Active Response Integration**: Configure the Wazuh active-response module on MAZZI-NEW-HOST to run a script (e.g., netsh firewall drop rules) automatically when Rule 60204 flags an attack.

- **Transition to Kerberos**: Disable insecure or legacy NTLM implementations across systems to stop password sniffing or rapid brute-forcing methods.

- **Multi-Factor Authentication (MFA)**: Ensure any network-facing endpoints require an extra layer of authentication beyond simple text passphrases.
