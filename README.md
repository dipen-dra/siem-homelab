# Home Lab SIEM: Detecting and Responding to an SSH Brute-Force Attack

## Overview

This project builds a small home lab to demonstrate detection and automated response to an SSH brute-force attack, using Wazuh as the SIEM platform. The lab follows a simple pipeline: an attack is simulated, a SIEM detects it, and an active response automatically blocks the attacker.

## Objective

To configure a SIEM (Wazuh) capable of detecting an SSH brute-force attack in near real time and automatically responding by blocking the attacking IP address, without relying on manual intervention.

## Lab environment

| Component | Detail |
|---|---|
| Host | MacBook (Apple Silicon, M4), 16GB RAM, 256GB storage |
| Hypervisor | VMware Fusion (Apple Silicon build) |
| Attacker VM | Kali Linux (ARM64), user `dipendra` |
| Victim / SIEM host | Ubuntu Server 26.04.1 LTS (ARM64), hostname `home-lab`, user `dipendra` |
| Network | Isolated Custom Network in VMware Fusion (Kali and Ubuntu can reach each other, isolated from the host network) |
| SIEM | Wazuh 4.14.7 (all-in-one: indexer, manager, dashboard on the same Ubuntu VM) |

Two design decisions were made to fit an Apple Silicon environment:
- **pfSense was not used** for network segmentation, since pfSense has no ARM64 build. `ufw`/iptables on the Ubuntu VM filled the equivalent role.
- **Kali and Ubuntu both run natively on ARM64** rather than through x86 emulation, keeping performance close to native.

### Architecture

![Home lab SIEM architecture: Kali attacks Ubuntu over SSH, Wazuh manager detects the brute force via rule 5763, and active response blocks the attacker's IP]
![alt text](<Ubuntu VM (home-lab).png>)

## Methodology

The engagement followed a simple four-stage pipeline: reconnaissance, attack simulation, detection, and automated response.

### 1. Reconnaissance

An Nmap service scan was run from Kali against the Ubuntu VM to confirm the attack surface before testing:

```
$ nmap -sV 192.168.151.132
PORT    STATE SERVICE   VERSION
22/tcp  open  ssh       OpenSSH 10.2p1 Ubuntu 2ubuntu3.6 (Ubuntu Linux; protocol 2.0)
443/tcp open  ssl/https
```

Two open ports were confirmed: SSH (22), the target of the attack, and 443, serving the Wazuh dashboard itself.

### 2. Attack simulation

A brute-force attack against SSH was simulated using Hydra with a custom wordlist of 15 common but incorrect passwords, targeting the known username `dipendra`:

```
$ hydra -l dipendra -P wordlist.txt ssh://192.168.151.132
[DATA] max 15 tasks per 1 server, overall 15 tasks, 15 login tries
1 of 1 target completed, 0 valid password found
```

15 failed login attempts were generated in approximately 6 seconds, none of which used the real password, to avoid an accidental successful compromise while still producing a realistic rapid-fire brute-force pattern.

### 3. Detection

Wazuh's default ruleset monitors the manager's own local `auth.log` (via `journald`) for SSH authentication events. No custom rule was required. Each individual failed login triggered rule 5760 ("sshd: authentication failed", level 5). Once the rate of failures matched a brute-force pattern (8+ failures in a short window), Wazuh escalated automatically to rule 5763:

```
sshd: brute force trying to get access to the system. Authentication failed.
Rule ID: 5763   Level: 10
MITRE ATT&CK: T1110 (Brute Force) — Credential Access
```

Wazuh's built-in MITRE ATT&CK mapping classified the activity as technique T1110 under the Credential Access tactic without any manual tagging.

### 4. Automated response

Wazuh's Active Response capability was configured to link rule 5763 to the built-in `firewall-drop` command, which blocks an offending IP address via `iptables` for a defined timeout. The relevant configuration block, added to `/var/ossec/etc/ossec.conf`:

```xml
<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>180</timeout>
</active-response>
```

## Results

The attack was re-run to verify the full detection-to-response pipeline end to end.

**Wazuh dashboard**, Threat Hunting module, showed the full chain of events:
- Multiple `sshd: authentication failed` alerts (rule 5760, level 5)
- Escalation to `sshd: brute force trying to get access to the system` (rule 5763, level 10)
- `Host Blocked by firewall-drop Active Response` (rule 651, level 3)

![Wazuh Threat Hunting dashboard showing the full alert sequence: repeated authentication failures (rule 5760), escalation to brute-force detection (rule 5763, level 10), and the automated block (rule 651)]![alt text](<Pasted Graphic 1.png>)

**Active response log** (`/var/ossec/logs/active-responses.log`) confirmed the block was executed against the correct source IP, with the full alert context including the MITRE mapping:

```
active-response/bin/firewall-drop: Starting
"rule":{"level":10,"description":"sshd: brute force trying to get access to the
system. Authentication failed.","id":"5763","mitre":{"id":["T1110"],
"tactic":["Credential Access"],"technique":["Brute Force"]}}
"data":{"srcip":"192.168.151.129","srcport":"37516","dstuser":"dipendra"}
active-response/bin/firewall-drop: Ended
```

**Network-level verification**: immediately after the second attack run, a ping from Kali to the Ubuntu VM was attempted:

```
$ ping 192.168.151.132
--- 192.168.151.132 ping statistics ---
75 packets transmitted, 0 received, 100% packet loss, time 75762ms
```

Complete packet loss confirmed the attacker's IP was genuinely blocked at the firewall level, not just flagged in the dashboard, for the full 180-second timeout window.

## Troubleshooting notes

A few real-world issues came up during setup, documented here since they reflect practical system administration skills alongside the security work:

- **Disk sizing**: the Ubuntu VM's default 10GB disk was too small for a full Wazuh install (indexer + manager + dashboard). The VM's virtual disk was grown to 50GB in VMware Fusion, then the change was propagated through the LVM stack live, without reinstalling the OS: `growpart` to extend the partition, `pvresize` to extend the LVM physical volume, and `lvextend -r` to extend the logical volume and filesystem together.
- **Broken partial install recovery**: an initial Wazuh install attempt failed partway through (unrelated to disk space) and left the `wazuh-manager` package in a broken state that `apt purge` could not clean up, because its own removal scripts (`prerm`/`postrm`) referenced files from an incomplete install. These broken maintainer scripts were removed directly from `/var/lib/dpkg/info/`, which allowed `dpkg` to purge the package cleanly, after which Wazuh was reinstalled successfully.

## Conclusion

This lab demonstrates a complete, working detection-and-response pipeline built from open-source tooling: Nmap for reconnaissance, Hydra for attack simulation, and Wazuh for both detection (via its default ruleset, automatically enriched with MITRE ATT&CK context) and automated response (via Active Response blocking the attacker's IP through iptables). The result is a SIEM that not only flags a brute-force attempt but autonomously contains it within seconds, verified independently at the network level through a packet-loss test.


