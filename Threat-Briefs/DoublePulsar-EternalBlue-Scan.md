# 🍯 Threat Brief: DoublePulsar & EternalBlue Mass Scanning

**Date:** April 2026  
**Analyst:** Thomas  
**Targeted Service:** SMB (Port 445)  
**Honeypot Sensor:** T-Pot (Suricata)  

## 📌 Executive Summary
During routine analysis of the Azure-based Global Attack Map honeypot, the Suricata Network Intrusion Detection System (NIDS) registered a massive volume of alerts indicating automated exploitation attempts targeting Server Message Block (SMB). Over a 30-day period, the sensor logged over 33,000 discrete events attempting to install the DoublePulsar kernel backdoor, a tool originally associated with the Equation Group and heavily utilized in the WannaCry ransomware attacks.

This brief documents the network telemetry of these mass-scanning events and outlines the theoretical attack path.

---

## 📡 Phase 1: Network Telemetry & Exploit Targeting

The attackers utilized automated scanning scripts to identify public-facing infrastructure with Port 445 open to the internet. 

![Suricata DoublePulsar Logs](../images/doublepulsar_suricata_logs.png)
*(Note: Upload your Suricata `DestPort: 445` screenshot here)*

### Analysis of the Network Traffic:
* **Targeted Port:** `DestPort: 445` (TCP). This confirms the attackers are hunting for vulnerable Windows SMB services.
* **Exploit Signature:** `ET EXPLOIT [PTSecurity] DoublePulsar Backdoor installation communication`. This signature triggers when Suricata detects the specific packet structure used to verify if the DoublePulsar backdoor is already installed, or the traffic attempting to inject it.
* **Alert Category:** `Attempted Administrator Privilege Gain`. DoublePulsar operates as a Ring 0 (kernel-mode) payload. Successful execution grants the attacker complete, undetected control over the operating system, allowing for arbitrary code execution and secondary payload delivery (such as ransomware).

---

## 🕵️‍♂️ Phase 2: Threat Intelligence & Attribution

To understand the origin of these mass scanning events, the highest-volume offending IP addresses were extracted from the Suricata logs and analyzed using external Open-Source Intelligence (OSINT) platforms.

### Indicators of Compromise (IoCs)

| Indicator Type | Value | Description |
| :--- | :--- | :--- |
| **Attacker IP** | `83.239.79.234` | Primary source of high-volume Port 445 scanning. |
| **Target Port** | `445` | Server Message Block (SMB). |
| **Exploit Target** | `DoublePulsar` | Kernel-level backdoor (Equation Group/Shadow Brokers). |

### OSINT Analysis: 83.239.79.234

A query against AbuseIPDB yielded the following intelligence regarding the primary scanning node:

![AbuseIPDB Threat Intelligence for 83.239.79.234](../images/abuseipdb_83_239_79_234.png)
*(Note: Upload the AbuseIPDB screenshot here)*

* **Abuse Confidence Score:** 81% (Reported 43 times within the last 60 days)
* **ISP / Organization:** PJSC Rostelecom Macroregional Branch South (AS25490)
* **Location:** Krasnodar, Russian Federation 🇷🇺
* **Usage Type:** Fixed Line ISP

### Analyst Notes & Behavioral Correlation
The OSINT data reveals that this IP is classified as a "Fixed Line ISP" rather than a commercial data center. This strongly suggests that the attacking node is likely a compromised residential router or an infected endpoint on a consumer network that has been drafted into a larger botnet infrastructure.

Furthermore, community reports from AbuseIPDB corroborate the honeypot's Suricata telemetry. Other security researchers have flagged this exact IP within the last 24 to 72 hours for aggressive, automated TCP SYN scans targeting **Port 445 (SMB)** and **Port 3389 (RDP)**. This combination of ports is the textbook signature of an automated EternalBlue (MS17-010) or BlueKeep (CVE-2019-0708) scanning script hunting for vulnerable, unpatched Windows legacy systems.

---

## 🛡️ Phase 3: Defensive Mitigations

The EternalBlue exploit and subsequent DoublePulsar backdoor installation represent one of the most devastating attack chains in modern history. However, the attack relies entirely on legacy configurations and poor perimeter security. To defend a production environment against this threat, the following controls must be implemented:

1. **Disable SMBv1 Globally:** SMBv1 is a deprecated and inherently insecure protocol. It should be disabled across all Windows environments. This can be accomplished rapidly via Group Policy (GPO) or PowerShell (`Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol`).
2. **Apply Security Update MS17-010:** Microsoft released a critical patch for this specific vulnerability in March 2017. Ensuring all Windows endpoints and servers are fully patched neutralizes the EternalBlue exploit, even if SMB is exposed.
3. **Perimeter Firewall Restrictions:** Port 445 (SMB) and Port 3389 (RDP) should **never** be exposed directly to the public internet. Access to these services should be restricted to internal networks, and remote access should require a secure VPN with Multi-Factor Authentication (MFA). 
4. **Network Segmentation:** If legacy systems (such as older medical devices or industrial control systems) absolutely must use SMBv1, they should be strictly isolated on a dedicated VLAN. Heavy internal Access Control Lists (ACLs) should be applied to prevent these vulnerable machines from communicating with the broader enterprise network.
5. **Deploy Endpoint Detection & Response (EDR):** Because DoublePulsar operates entirely in memory (Ring 0) and doesn't write a traditional executable file to the disk, legacy signature-based antivirus often misses it. EDR solutions are required to monitor for anomalous kernel-level behavior and unauthorized memory injection.
