# 🛡️ Wazuh SIEM Home Lab

> A fully operational Security Information and Event Management (SIEM) lab built on open-source tooling; demonstrating threat detection, incident response, and custom rule development across four real-world attack scenarios.

![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=flat-square)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh%204.7-blue?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-Vmware-183A61?style=flat-square)
![Attacker](https://img.shields.io/badge/Attacker-Kali%20Linux-557C94?style=flat-square)

---

## 🌐 Live Lab Summary

| Component | Details |
|---|---|
| SIEM Platform | Wazuh 4.7 (Manager + Indexer + Dashboard) |
| Target Machine | Ubuntu Server 22.04 (Wazuh Agent) |
| Attacker Machine | Kali Linux |
| Virtualisation | Vmware — Host-Only Network (192.168.102.x) |
| Attack Scenarios | 4 completed |
| Custom Rules Written | 1 |
| Total Alerts Generated | 100+ |

---

## 📐 Lab Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      HOST MACHINE                           │
│                   (Vmware Host)                         │
│                                                             │
│  ┌──────────────────┐        ┌──────────────────────────┐  │
│  │  VM 1            │        │  VM 2                    │  │
│  │  Wazuh Server    │◄───────│  Ubuntu Agent            │  │
│  │  Ubuntu 22.04    │  logs  │  Ubuntu 22.04            │  │
│  │  192.168.102.128   │        │  192.168.102.129           │  │
│  └────────┬─────────┘        └──────────────────────────┘  │
│           │ alerts                                          │
│           │                  ┌──────────────────────────┐  │
│           └──────────────────│  VM 3                    │  │
│                    detects   │  Kali Linux (Attacker)   │  │
│                              │  192.168.102.130           │  │
│                              └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

Data Flow:
Kali attacks Ubuntu Agent → Agent ships logs to Wazuh Server
→ Wazuh correlates and fires alerts → Dashboard visualises
```

---

## 🛠️ Technologies Used

| Tool | Role |
|---|---|
| Wazuh 4.7 | SIEM — log collection, correlation, alerting |
| Wazuh Indexer | Log storage and search (OpenSearch-based) |
| Wazuh Dashboard | Alert visualisation and investigation UI |
| Ubuntu Server 22.04 | Wazuh server host + monitored agent endpoint |
| Kali Linux | Attacker machine — offensive tools |
| Vmware | Virtualisation platform |
| Hydra | SSH brute force tool |
| Nmap | Network reconnaissance and port scanning |
| Nikto | Web application vulnerability scanner |

---

## ✅ What Was Completed

- [x] Wazuh server deployed on Ubuntu Server 22.04 via single-line installer
- [x] Ubuntu Agent enrolled and reporting to Wazuh Manager
- [x] Kali Linux attacker VM configured on isolated Host-Only network
- [x] File Integrity Monitoring (FIM) configured on sensitive directories
- [x] SSH brute force attack — detected and documented
- [x] Nmap port scan — detected and documented
- [x] Privilege escalation attempt — detected and documented
- [x] File Integrity Monitoring triggered — detected and documented
- [x] Web application scan (Nikto) — detected and documented
- [x] 1 custom detection rule written and validated
- [x] Alerts mapped to MITRE ATT&CK framework
- [x] Incident reports written for all 4 scenarios

---

## ⚔️ Attack Scenarios & Detections

---

### Scenario 1 — SSH Brute Force

**Objective:** Simulate a credential stuffing attack against the Ubuntu Agent SSH service.

**Tool:** Hydra

**Command run from Kali:**
```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt \
  192.168.56.20 ssh -t 4 -V
```

**Wazuh Rules Fired:**

| Rule ID | Description | Level |
|---|---|---|
| 5551 | SSH brute force (multiple auth failures) | 10 |
| 5763 | SSHD authentication failed | 5 |
| 5710 | Attempt to login using a non-existent user | 5 |

**Detection:** Wazuh fired Rule 5551 after detecting more than 8 failed authentication attempts within 60 seconds from the same source IP — correctly identifying the brute force pattern rather than treating each failure as an isolated event.

**SOC Response Actions:**
- Identify source IP (192.168.102.130 — Kali)
- Review auth logs: `sudo cat /var/log/auth.log | grep Failed`
- Check for any successful logins following the failed attempts
- Implement IP block via UFW: `sudo ufw deny from 192.168.102.130`
- Review SSH configuration — disable root login, enforce key-based auth

**MITRE ATT&CK:** T1110.001 — Brute Force: Password Guessing

---

### Scenario 2 — Nmap Port Scan (Network Reconnaissance)

**Objective:** Simulate attacker reconnaissance to map open ports and services.

**Tool:** Nmap

**Commands run from Kali:**
```bash
# Stealth SYN scan
sudo nmap -sS -O 192.168.102.129

# Aggressive scan
sudo nmap -A 192.168.102.129
```

**Wazuh Rules Fired:**

| Rule ID | Description | Level |
|---|---|---|
| 40111 | Host scan detected | 8 |
| 40112 | Port scan detected | 8 |

**Detection:** Wazuh's network monitoring detected the volume and pattern of connection attempts across multiple ports — distinguishing scan behaviour from normal traffic. The `-A` aggressive scan generated significantly more alerts than the stealth scan, demonstrating that scan aggressiveness directly correlates to detection likelihood.

**SOC Response Actions:**
- Identify scanning source IP and time of scan
- Correlate with other alerts — did scanning precede a brute force or exploitation attempt?
- Review which ports were found open and assess unnecessary exposure
- Ensure firewall rules are appropriately restrictive

**MITRE ATT&CK:** T1595 — Active Scanning | T1046 — Network Service Discovery

---

### Scenario 3 — Privilege Escalation Attempt

**Objective:** Simulate a low-privilege user attempting to escalate to root.

**Steps executed on Ubuntu Agent:**
```bash
# Create low-privilege test user
sudo adduser testuser

# Switch to low-privilege user
su testuser

# Attempt privilege escalation — multiple sudo failures
sudo cat /etc/shadow
sudo su root
sudo bash
```

**Wazuh Rules Fired:**

| Rule ID | Description | Level |
|---|---|---|
| 5503 | Sudo command used | 4 |
| 5502 | Sudo failed — incorrect password | 7 |
| 2502 | User authentication failure | 5 |
| 100002 | Custom: Multiple sudo failures — possible escalation | 10 |

**Detection:** Built-in rules caught individual sudo failures. The custom Rule 100002 (see Custom Rules section) fired after 3 sudo failures within 60 seconds — correctly escalating the alert severity to Level 10, which would trigger a high-priority SOC notification in a production environment.

**SOC Response Actions:**
- Identify the user account attempting escalation
- Review whether the account has legitimate admin requirements
- Check for successful escalation attempts in the same timeframe
- Review sudoers configuration: `sudo visudo`
- Consider locking the account pending investigation: `sudo passwd -l testuser`

**MITRE ATT&CK:** T1548.003 — Abuse Elevation Control Mechanism: Sudo

---


### Scenario 4 — Web Application Scan (Nikto)

**Objective:** Simulate an attacker scanning a web server for vulnerabilities.

**Setup:** Apache2 installed on Ubuntu Agent:
```bash
sudo apt install apache2 php -y
sudo systemctl start apache2
```

**Tool:** Nikto (run from Kali)

**Command:**
```bash
nikto -h http://192.168.102.129
```

**Wazuh Rules Fired:**

| Rule ID | Description | Level |
|---|---|---|
| 31101 | Web server 400 error code | 5 |
| 31103 | Web server 404 error code | 5 |
| 31151 | Multiple web server errors (scan detected) | 10 |
| 100001 | Custom: Kali Linux detected in user agent | 12 |

**Detection:** Nikto sends hundreds of probing requests in rapid succession, each testing for known vulnerabilities. Wazuh's Apache log monitoring detected the volume of 400/404 errors as scan behaviour. The custom Rule 100001 fired immediately when Nikto's user agent string — which contains "Kali" — appeared in the Apache access logs, elevating the alert to Level 12.

**Key Insight:** This scenario demonstrated that attacker tooling often betrays itself through user agent strings and request patterns before any successful exploitation occurs — passive signature-based detection at the web layer is a first line of defence.

**SOC Response Actions:**
- Review the full list of URLs Nikto probed — did any return 200 (success) responses?
- Check for follow-on exploitation attempts against identified vulnerabilities
- Review web application firewall (WAF) rules
- Ensure unnecessary Apache modules and default pages are disabled
- Consider rate limiting or IP blocking at the web server level

**MITRE ATT&CK:** T1595.002 — Vulnerability Scanning

---

## 📋 Custom Detection Rules

All custom rules are stored in `/var/ossec/etc/rules/local_rules.xml`.

---

### Rule 100001 — Kali Linux User Agent Detection

```xml
<group name="custom_rules,web_attacks">

  <rule id="100001" level="12">
    <if_sid>31101</if_sid>
    <match>Kali</match>
    <description>
      Custom: Kali Linux detected in web request user agent —
      possible attacker reconnaissance tool in use
    </description>
    <mitre>
      <id>T1595</id>
    </mitre>
  </rule>

</group>
```

**Logic:** Builds on Wazuh's base web server rule (31101). If any web request contains the string "Kali" in the user agent — a signature of Kali Linux tools including Nikto — fire a Level 12 alert. Level 12 in Wazuh triggers a high-priority notification.

---

## 🗺️ MITRE ATT&CK Coverage

| Tactic | Technique | ID | Scenario |
|---|---|---|---|
| Reconnaissance | Active Scanning | T1595 | Nmap scan, Nikto scan |
| Reconnaissance | Vulnerability Scanning | T1595.002 | Nikto web scan |
| Discovery | Network Service Discovery | T1046 | Nmap port scan |
| Credential Access | Brute Force: Password Guessing | T1110.001 | SSH brute force |
| Privilege Escalation | Abuse Elevation: Sudo | T1548.003 | Sudo escalation |
| Defence Evasion | Masquerading | T1036 | FIM scenario |
| Impact | Data Manipulation | T1565 | FIM scenario |
| Persistence | Create Account: Local Account | T1136.001 | Custom Rule 100003 |

---

## 🚧 Challenges & Lessons Learned

### Challenge 1 — Wazuh Indexer Memory Requirements
**Problem:** The Wazuh Indexer service crashed silently on first boot due to insufficient RAM allocation on the VM.
**Solution:** Increased the Wazuh Server VM RAM from 2 GB to 4 GB in VirtualBox settings. The Wazuh Indexer requires a minimum of 4 GB to run stably.
**Lesson:** Wazuh's all-in-one installer is convenient but resource-intensive. In production, the Manager, Indexer, and Dashboard are typically deployed on separate nodes.

### Challenge 2 — Agent Connectivity
**Problem:** The Ubuntu Agent showed as Disconnected in the Dashboard despite the wazuh-agent service running on the VM.
**Solution:** The Host-Only network adapter was not correctly configured in VirtualBox — both VMs needed to be on the same vboxnet interface. After correcting the adapter assignment and confirming with `ping`, the agent connected within 30 seconds.
**Lesson:** Network connectivity debugging is a core SOC skill. The same principle applies in real environments — agent connectivity issues are almost always a network or firewall problem, not a Wazuh problem.

### Challenge 3 — Custom Rule Syntax Validation
**Problem:** First attempt at writing custom rules produced XML syntax errors that prevented Wazuh Manager from restarting.
**Solution:** Used Wazuh's built-in log test tool to validate rules before restarting the service:
```bash
sudo /var/ossec/bin/wazuh-logtest
```
Paste a sample log line and the tool shows which rules match — invaluable for debugging.
**Lesson:** Always validate rule syntax before applying. In production, a Wazuh Manager restart with broken rules takes the SIEM offline.

### Challenge 4 — Alert Noise vs. Signal
**Problem:** The Nmap scan generated hundreds of low-level alerts that made it difficult to identify the meaningful signal.
**Solution:** Used Wazuh Dashboard filters to group alerts by Rule ID and source IP — immediately surfacing the scan pattern. Applied alert level filters to focus on Level 8+ events.
**Lesson:** Alert fatigue is a real SOC problem. Tuning alert thresholds and writing correlation rules that surface patterns (not individual events) is more valuable than raw detection volume.

---

## 💡 Future works

- **Deploy a Windows Agent** — most enterprise environments are Windows-heavy. Adding a Windows endpoint would extend coverage to Windows Event Log monitoring, registry changes, and PowerShell execution — significantly more realistic SOC experience
- **Integrate VirusTotal API** — Wazuh supports automatic file hash lookups against VirusTotal. Enabling this would add threat intelligence enrichment to FIM alerts
- **Build automated response playbooks** — Wazuh supports active response scripts that automatically block IPs, kill processes, or quarantine files when specific rules fire. This is the next logical step after detection
- **Set up email/Slack alerting** — configuring Wazuh to send high-severity alerts to a communication channel simulates real SOC notification workflows

---

## 📚 References

- [Wazuh Documentation](https://documentation.wazuh.com/)
- [Wazuh Rule Syntax Reference](https://documentation.wazuh.com/current/user-manual/ruleset/ruleset-xml-syntax/rules.html)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Hydra Documentation](https://github.com/vanhauser-thc/thc-hydra)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [Nikto Web Scanner](https://cirt.net/Nikto2)

---

> _Part of a broader portfolio of cloud, cybersecurity, and AI operations projects._
> _Built to learn. Documented to teach._
