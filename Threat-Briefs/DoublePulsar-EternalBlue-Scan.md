# 🍯 Threat Brief: DoublePulsar & EternalBlue Mass Scanning

**Date:** April 2026  
**Analyst:** Thomas Price

**Targeted Service:** SMB (Port 445)  
**Honeypot Sensor:** T-Pot (Suricata)  

## 📌 Executive Summary
During routine analysis of the Azure-based Global Attack Map honeypot, the Suricata Network Intrusion Detection System (NIDS) registered a massive volume of alerts indicating automated exploitation attempts targeting Server Message Block (SMB). Over a 30-day period, the sensor logged over 33,000 discrete events attempting to install the DoublePulsar kernel backdoor. 

DoublePulsar and its delivery mechanism, EternalBlue, are notoriously tied to the U.S. National Security Agency's (NSA) elite hacking unit, the Equation Group. Since the tools were leaked to the public in 2017, they have been responsible for billions of dollars in global damages. This brief documents the network telemetry of these mass-scanning events, explores the historical context of the exploit, and outlines why this decade-old threat continues to aggressively blanket the public internet today.

---

## 📡 Phase 1: Network Telemetry & Exploit Targeting

The attackers utilized automated scanning scripts to identify public-facing infrastructure with Port 445 open to the internet. 

![Suricata DoublePulsar Logs](../images/doublepulsar_suricata_logs.png)
Suricata `DestPort: 445` 

### Analysis of the Network Traffic:
* **Targeted Port:** `DestPort: 445` (TCP). This confirms the attackers are indiscriminately hunting for vulnerable Windows SMB services.
* **Exploit Signature:** `ET EXPLOIT [PTSecurity] DoublePulsar Backdoor installation communication`. This signature triggers when Suricata detects the specific packet structure used to verify if the DoublePulsar backdoor is already installed, or the traffic attempting to actively inject it.
* **Alert Category:** `Attempted Administrator Privilege Gain`. DoublePulsar operates as a Ring 0 (kernel-mode) payload. Successful execution grants the attacker complete, undetected control over the operating system, allowing for arbitrary code execution and secondary payload delivery.

---

## 🕵️‍♂️ Phase 2: Threat Intelligence & Historical Context

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

AbuseIPDB This IP was reported 43 times. Confidence of Abuse is 81%:

* **Abuse Confidence Score:** 81% (Reported 43 times within the last 60 days)
* **ISP / Organization:** PJSC Rostelecom Macroregional Branch South (AS25490)
* **Location:** Krasnodar, Russian Federation 🇷🇺
* **Usage Type:** Fixed Line ISP

### The NSA Connection: Equation Group & Shadow Brokers
DoublePulsar was originally developed as a highly classified cyber-espionage tool by the NSA's Tailored Access Operations (TAO), often tracked by researchers as the "Equation Group." For years, the NSA used it alongside a zero-day exploit named **EternalBlue (MS17-010)** to quietly infiltrate target networks. 

However, in April 2017, a mysterious hacker collective known as **The Shadow Brokers** leaked the NSA's toolkit to the public internet. This handed military-grade cyberweapons to civilian cybercriminals. Just one month later, North Korean-linked threat actors weaponized EternalBlue and DoublePulsar to create the **WannaCry** ransomware worm, infecting over 300,000 computers globally and crippling organizations like the UK's National Health Service (NHS). Shortly after, Russian state-sponsored actors used the same exploits to launch the devastating **NotPetya** wiper attack.

### Analyst Notes: Why is this still happening in 2026?
Microsoft patched the EternalBlue vulnerability in March 2017 (MS17-010). So why is the Azure honeypot still logging 33,000+ hits for DoublePulsar?
1. **The Zombie Botnet Effect:** Many of the IPs scanning the honeypot (like the Russian residential ISP flagged above) are compromised consumer endpoints. These "zombie" machines are infected with old, automated worm scripts that simply never received a "kill" command. They will blindly scan the internet for Port 445 in perpetuity.
2. **Technical Debt:** Threat actors know that critical infrastructure, manufacturing plants, and healthcare facilities often run legacy systems (like Windows XP or Windows 7) that cannot be easily taken offline for patching. By continuously scanning, attackers hope to catch an administrator making a firewall mistake that accidentally exposes one of these vulnerable legacy systems to the public web.

---

## 🛡️ Phase 3: Defensive Mitigations

The EternalBlue exploit and subsequent DoublePulsar backdoor installation represent one of the most devastating attack chains in modern history. However, the attack relies entirely on legacy configurations and poor perimeter security. To defend a production environment against this threat, the following controls must be implemented:

1. **Disable SMBv1 Globally:** SMBv1 is a deprecated and inherently insecure protocol. It should be disabled across all Windows environments. This can be accomplished rapidly via Group Policy (GPO) or PowerShell (`Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol`).
2. **Apply Security Update MS17-010:** Ensuring all Windows endpoints and servers are fully patched neutralizes the EternalBlue exploit, even if SMB is exposed.
3. **Perimeter Firewall Restrictions:** Port 445 (SMB) and Port 3389 (RDP) should **never** be exposed directly to the public internet. Access to these services should be restricted to internal networks, and remote access should require a secure VPN with Multi-Factor Authentication (MFA). 
4. **Network Segmentation:** If legacy systems (such as older medical devices or industrial control systems) absolutely must use SMBv1, they should be strictly isolated on a dedicated VLAN. Heavy internal Access Control Lists (ACLs) should be applied to prevent these vulnerable machines from communicating with the broader enterprise network.
5. **Deploy Endpoint Detection & Response (EDR):** Because DoublePulsar operates entirely in memory (Ring 0) and doesn't write a traditional executable file to the disk, legacy signature-based antivirus often misses it. EDR solutions are required to monitor for anomalous kernel-level behavior and unauthorized memory injection.
