# 🚨 Threat Brief: Database & Remote Desktop Credential Harvesting

**Date:** March 2026

**Sensor:** Heralding (Credential Catching Honeypot)

**Targeted Services:** PostgreSQL (TCP/5432), VNC (TCP/5900)

## Overview
While analyzing the honeynet telemetry, a secondary campaign of automated credential harvesting was identified. Unlike the SSH brute-force attacks captured by Cowrie, this traffic was specifically directed at the **Heralding** sensor, which simulates exposed enterprise databases and remote desktop services.

## Analysis & Visual Evidence

### The Target: PostgreSQL (Port 5432)
Analysis of the payload data revealed a highly targeted effort to breach PostgreSQL databases. 

![Postgres Tag Cloud](../images/heralding_tagcloud.png)> *The tag cloud above highlights `postgres` as the overwhelmingly primary username targeted, accompanied by common default passwords (e.g., `123456`, `admin`, `password`).*

### The Logs: Multi-Protocol Scanning
By querying the `Heralding` index in the Discover tab, I was able to correlate the source IPs with the specific destination ports they were attacking.

![Heralding Discover Table](../images/heralding_table.png)> *This table demonstrates the automated nature of the attacks. Threat actors are systematically scanning for default database ports (5432) and graphical remote desktop ports (5900) to gain deeper access to enterprise networks and exfiltrate sensitive data.*

## Threat Intelligence Takeaways
This activity highlights the extreme risk of exposing databases directly to the public internet. The botnets captured in this brief are likely operating as Initial Access Brokers (IABs) or automated ransomware deployment scripts. 

**Recommended Mitigations:**
1. **Network Segmentation:** Databases should never have a public IP address. They should reside in private subnets accessible only by authorized application servers.
2. **Zero Trust / VPN:** Remote Desktop protocols (VNC/RDP) must be placed behind a VPN or Zero Trust Network Access (ZTNA) gateway requiring Multi-Factor Authentication (MFA).
3. **Disable Default Accounts:** The `postgres` default superuser account should be renamed or disabled, and strong password policies must be enforced.
