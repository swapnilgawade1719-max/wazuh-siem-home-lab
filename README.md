# Wazuh SIEM Home Lab — Build, Deploy & Detect

A self-directed SOC analyst project: building a working **Wazuh SIEM** from scratch on Linux, bringing a host under monitoring, and validating real-time threat detection by generating a live SSH brute-force attack and confirming the SIEM caught it.

> **Note:** This is a hands-on home lab built on self-provisioned infrastructure — not a production environment.

---

## What I built

A complete single-node Wazuh 4.14 deployment (indexer + server + dashboard) on Ubuntu Server 22.04, running in a VirtualBox VM, reachable from the host browser, and actively analyzing security events in real time.

## Skills & tools demonstrated

- **SIEM deployment from scratch** — Wazuh indexer, server (manager), and dashboard, installed and configured on Linux
- **Linux server administration** — Ubuntu Server install, LVM disk configuration, networking (bridged adapter, static-ish DHCP), systemd services
- **Real-world troubleshooting** — resolved hardware-virtualization (VBS/Hyper-V) conflicts, an IPv6 routing failure (forced IPv4 for `wget`/`apt`), and slow-mirror download stalls
- **Threat detection & SOC analysis** — generated a live SSH brute-force attack and confirmed automated detection in the SIEM
- **Detection logic** — interpreted Wazuh rule levels and correlation (composite) rules
- **MITRE ATT&CK** — mapped the detection to adversary technique T1110

## The detection (proof it works)

After deployment, I launched an SSH brute-force against the monitored host and the SIEM detected it automatically:

| Wazuh Rule | Description | Level |
|------------|-------------|-------|
| 5710 | sshd: Attempt to login using a non-existent user | 5 |
| 5760 | sshd: authentication failed | 5 |
| 2502 | syslog: user missed the password more than one time | 10 |
| **5712** | **sshd: brute force trying to get access to the system** | **10** |

The key insight: individual failed logins are **low severity (level 5)**, but Wazuh's **correlation rule 5712** counts repeated failures from one source within a time window and escalates to a **high-severity (level 10)** brute-force alert — automating the exact investigation done manually in my earlier log-analysis labs.

## MITRE ATT&CK

`T1110` — Brute Force

---

## Architecture

```
            ┌──────────────────────────────────────────┐
            │            Wazuh Server (Ubuntu)          │
            │   ┌───────────┐ ┌─────────┐ ┌──────────┐  │
  Attacker  │   │  Indexer  │ │ Manager │ │Dashboard │  │
  (SSH) ───►│   │ (storage/ │ │(detect/ │ │(analyst  │  │◄── Analyst
  brute     │   │  search)  │ │  rules) │ │   UI)    │  │    (browser)
  force     │   └───────────┘ └─────────┘ └──────────┘  │
            │        Monitors its own auth logs          │
            └──────────────────────────────────────────┘
```

## Repository contents

| File | Description |
|------|-------------|
| [`LAB5_REPORT.md`](LAB5_REPORT.md) | Full build report — deployment, troubleshooting, detection, lessons |
| `images/` | Dashboard, detection events, and MITRE mapping screenshots |

**➡️ Read the full build & detection write-up in [LAB5_REPORT.md](LAB5_REPORT.md).**
