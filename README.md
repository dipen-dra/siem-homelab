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

<img width="2720" height="1800" alt="Ubuntu VM (home-lab)" src="https://github.com/user-attachments/assets/09531e44-0173-47a1-bdd0-e3c9e40aa489" />


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

<img width="1470" height="692" alt="Pasted Graphic 1" src="https://github.com/user-attachments/assets/0e5e47a8-5418-45ba-a0da-9614bdf94525" />


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

This lab demonstrates a complete, working detection-and-response pipeline built from open-source tooling: Nmap for reconnaissance, Hydra and manual payloads for attack simulation, and Wazuh for detection, using both its default ruleset (enriched automatically with MITRE ATT&CK context) and hand-written custom decoders and rules for a self-built vulnerable web application. Both SSH brute-force and web-application attacks (SQL injection, stored XSS) are detected and automatically responded to by blocking the attacker's IP, verified independently at the network level through packet-loss testing in both cases. Getting the web-attack response working end to end required diagnosing and fixing two separate real bugs along the way, an active-response binding collision in Wazuh's own configuration, and a stale-state bug in a custom watcher script built as an alternative to Wazuh's native dispatch mechanism, both resolved through direct evidence-based investigation rather than trial and error.

## Phase 2: Custom Detection Rules and a Second Attack Surface

Phase 1 relied entirely on Wazuh's built-in ruleset. Phase 2 extends the lab with a self-built vulnerable web application and hand-written Wazuh decoders and rules, going beyond configuring a SIEM to actually building detection logic for it.

<img width="1520" height="1120" alt="image" src="https://github.com/user-attachments/assets/530650bf-7879-481d-a6fb-ef644232016d" />

### VulnApp: a self-built vulnerable web application

A small PHP application, VulnApp, was built on the Ubuntu VM (Apache + PHP + MySQL) with two intentional vulnerabilities:

- **`login.php`**: a login form vulnerable to SQL injection through raw string concatenation in the query (`SELECT * FROM users WHERE username='$username' AND password='$password'`). Confirmed exploitable using a classic comment-based bypass (`admin' -- `), logging in as `admin` without knowing the password.
- **`comments.php`**: a comment box vulnerable to stored XSS, echoing user input with no output escaping. Confirmed exploitable with `<script>alert('XSS')</script>`, which executed in the browser.

Both pages log every attempt (timestamp, source IP, username/comment, and for logins, the full raw SQL query) to `/var/log/vulnapp/access.log`, giving Wazuh a realistic application-level log source to analyze.

### Custom decoders and rules

Two custom decoders were written to parse VulnApp's log format (`vulnapp-login` and `vulnapp-comment`), and five custom rules were built on top of them:

| Rule ID | Level | Description |
|---|---|---|
| 100100 | 3 | Failed login attempt |
| 100101 | 5 | Successful login |
| 100102 | 12 | Possible SQL injection detected in login query (MITRE T1190) |
| 100103 | 3 | Comment posted |
| 100104 | 12 | Possible XSS payload detected in comment (MITRE T1059.007) |

Both attacks were verified against the live rules using `wazuh-logtest` and confirmed again in the Wazuh dashboard, firing at the correct severity level with the correct MITRE ATT&CK mapping.

### Debugging the detection engine

Building these rules surfaced several real configuration issues, each resolved through direct investigation rather than trial and error:

- Wazuh's default regex engine (OS_Regex) does not support PCRE character classes like `\S`; decoders needed an explicit `type="pcre2"` attribute to use full regex syntax.
- Field names `status` and `data` are reserved internally by Wazuh and cannot be reused as custom decoder field names; both were renamed (`login_status`, `comment_text`).
- Literal `<` and `>` characters in a rule's regex pattern must be XML-escaped (`&lt;`, `&gt;`), since the rules file is itself XML.
- A complex PCRE2 alternation pattern for the XSS rule silently failed to match despite the decoder extracting the field correctly. This was isolated using `wazuh-logtest`: removing the field condition confirmed the rule and decoder both worked in principle, and progressively simplifying the pattern identified a working baseline (a direct substring match), which was reinstated as the final rule condition.

### Active response for web attacks: diagnosed and resolved

An attempt was made to extend the same `firewall-drop` active response used for SSH brute force (Phase 1) to also trigger on rules 100102 and 100104. This surfaced two distinct bugs, each diagnosed and fixed in turn.

**Bug 1: active-response binding collision.** Defining a second `<active-response>` block with the same `command` and `timeout` as the existing SSH block caused Wazuh to silently drop one of them internally, since it generates an internal binding identifier from `command+timeout` (e.g. `firewall-drop180`). Two blocks sharing both values collide. This was confirmed by inspecting Wazuh's generated `/var/ossec/etc/shared/ar.conf` file, which showed only one entry where two were expected. The fix was to give the second block a distinct timeout (`181` instead of `180`), producing two separate bindings (`firewall-drop180`, `firewall-drop181`).

**Bug 2: unreliable native dispatch for custom-decoded rules.** Even with distinct bindings confirmed correct, Wazuh's native active-response dispatch (via `wazuh-execd`) continued to fire inconsistently for the custom rules, despite firing reliably every time for the built-in SSH rule. Manually invoking the `firewall-drop` script directly, comparing alert JSON structures, and enabling debug logging on both `analysisd` and `execd` did not produce a conclusive root cause; the alert data and configuration were verifiably correct at every layer inspected.

Rather than continuing to chase this specific internal dispatch behaviour indefinitely, an alternative architecture was implemented: a small watcher script (`vulnapp-ar-watch.sh`, run as a `systemd` service) that tails `/var/ossec/logs/alerts/alerts.json` directly for rules 100102/100104 and applies an `iptables DROP` rule itself, with automatic expiry after the same 180-second window used elsewhere in the lab. This bypasses `wazuh-execd` entirely, trading native Wazuh integration for a simpler, independently verifiable mechanism.

This workaround initially appeared to fail on retesting after being left running for an extended period. Diagnosis (comparing the watcher's saved byte-offset state against the alert log's actual current file size) revealed a bug in the watcher script itself: it tracked its read position as a raw byte offset but never accounted for Wazuh's alert log being periodically rotated and truncated, so after a rotation the script's saved position was larger than the file's new size and it silently stopped processing new alerts. The fix added a check to reset the tracked position whenever the file shrinks. After this fix, the full chain was confirmed working end to end: a live SQL injection request from Kali against the real `login.php` produced a Wazuh alert, the watcher picked it up within seconds, an `iptables DROP` rule for the attacker's IP appeared, and a follow-up ping from Kali showed 100% packet loss, the identical signature confirmed earlier for the native SSH active response.


