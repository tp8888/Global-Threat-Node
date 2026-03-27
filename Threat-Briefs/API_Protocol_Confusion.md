# 🔄 Threat Hunt: Application-Layer Protocol Confusion

* **Date:** March 2026
* **Primary Sensor:** Cowrie (SSH/Telnet)
* **Targeted Port:** 22 (SSH)
* **Payload:** `<se:Body>` (XML / SOAP Envelope)
* **Objective:** API Vulnerability Reconnaissance (UPnP)

## Executive Summary
While analyzing access logs for the Cowrie SSH honeypot, I discovered a highly unusual authentication attempt. A threat actor attempted to log into the SSH terminal using an empty XML SOAP envelope as the username. This event highlights the automated, blind nature of modern botnets and their attempts to map the internet for vulnerable application programming interfaces (APIs).

## 1. Log Analysis & Discovery
Using Kibana to filter the `cowrie.login.failed` index, I isolated an attack originating from a DigitalOcean VPS (IP: `167.71.124.28`). Instead of a standard dictionary username (e.g., `root` or `admin`), the attacker's script submitted the following payload into the terminal prompt:

`[<se:Body></se:Body></se:Envelope>]`

![Protocol Confusion Payload](../images/cowrie_protocol_confusion.png)
> *SIEM Telemetry: An XML SOAP envelope injected directly into an SSH login prompt.*

## 2. Threat Profiling: Protocol Confusion
This log entry is a textbook example of "Protocol Confusion." Automated botnets prioritize scanning speed over accuracy. Rather than initiating a TCP handshake to verify what service is running on a port, the script blindly sprayed an application-layer web payload (XML/HTTP) at a network-layer service (SSH). 

## 3. The Threat: UPnP API Reconnaissance
While this specific payload failed harmlessly against an SSH server, the empty SOAP envelope is a known reconnaissance probe. 

Threat actors throw these empty envelopes across the internet to map vulnerable **UPnP (Universal Plug and Play)** APIs, typically found on IoT devices and residential routers. 
* If a server responds with a standard connection drop, the bot moves on.
* If a server responds with a specific XML parsing error, the botnet flags the IP as a vulnerable web API and follows up with a malicious command injection payload designed to manipulate firewall rules or hijack DNS settings.

## Actionable Mitigations
* **Disable UPnP:** Universal Plug and Play should be disabled on all corporate and residential edge devices, as it allows unauthenticated applications to automatically forward ports through the firewall.
* **Rate Limiting:** Implementing aggressive rate limiting on public-facing APIs can stall automated reconnaissance scripts before they map the attack surface.
