# Lab 5 — Wazuh SIEM: Build, Deploy & Detect

| Field | Detail |
|---|---|
| **Project** | Deploy a SIEM and validate real-time threat detection |
| **Platform** | Wazuh 4.14 (indexer + server + dashboard), all-in-one |
| **Host OS** | Ubuntu Server 22.04 LTS (VirtualBox VM, bridged network) |
| **Outcome** | Working SIEM; live SSH brute-force detected automatically |
| **Status** | Complete |

---

## 1. Objective

Stand up a functioning SIEM from nothing, bring a host under monitoring, and prove the platform detects a real attack in real time — moving from *manual* log analysis (earlier labs) to *automated* detection at scale.

## 2. Architecture

A single-node, all-in-one Wazuh deployment running three components on one Ubuntu server:

- **Wazuh indexer** — stores and searches event data (OpenSearch-based).
- **Wazuh server (manager)** — the detection engine; ingests logs and runs the ruleset.
- **Wazuh dashboard** — the web console used for analysis.

The manager also monitors its own host's logs locally, which is what made end-to-end detection testing possible on a single node.

## 3. Deployment steps

1. Built an Ubuntu Server 22.04 VM (4 vCPU / RAM / 50 GB disk, full-disk LVM), bridged networking so the dashboard is reachable from the host.
2. Downloaded the Wazuh installation assistant and ran the all-in-one install:
   ```bash
   sudo bash ./wazuh-install.sh -a
   ```
3. Retrieved the generated admin credentials and logged into the dashboard at `https://<server-ip>`.
4. Verified the server was already analyzing its own host logs (baseline alerts present on first login).

## 4. Troubleshooting (real problems solved)

This build surfaced several real-world issues — documenting them because diagnosing them *is* the skill:

- **Hardware virtualization conflict:** the VM stalled on boot with kernel RCU-stall errors. Root cause: Windows **VBS/Hyper-V** was holding VT-x, forcing VirtualBox into slow emulation. Resolved by disabling VBS/Hyper-V and confirming VT-x availability.
- **IPv6 routing failure:** `curl`/`wget` resolved package hosts to IPv6 addresses the network couldn't route, so downloads silently failed. Diagnosed by comparing `ping 8.8.8.8` (IPv4, OK) vs `ping <host>` (IPv6, 100% loss). Resolved by forcing IPv4 (`wget -4`) and setting `Acquire::ForceIPv4 "true"` for apt.
- **Slow-mirror stalls:** the indexer package download crawled/stalled on a slow link. Diagnosed by monitoring `du -sh /var/cache/apt/archives/` across intervals to confirm progress vs freeze.
- **Agent/manager conflict:** the Wazuh agent refuses to install on the same host as the manager (they conflict by design). Learned that the manager self-monitors its own host, so a separate agent wasn't required for this single-node detection test.

## 5. Detection validation

Launched an SSH brute-force against the monitored host (repeated failed logins with invalid users and wrong passwords) and observed automatic detection in **Threat Hunting**:

| Rule | Description | Level |
|------|-------------|-------|
| 5710 | sshd: Attempt to login using a non-existent user | 5 |
| 5760 | sshd: authentication failed | 5 |
| 5503 | PAM: user login failed | 5 |
| 2502 | syslog: user missed the password more than one time | 10 |
| **5712** | **sshd: brute force trying to get access to the system** | **10** |

**Analysis:** Wazuh scores every rule 0–15 (0–6 low, 7–11 medium, 12–14 high, 15+ critical). Individual failures (5710/5760) are level 5 — noise on their own. The **composite/correlation rule 5712** aggregates repeated failures from a single source within a time window and escalates to **level 10**, flagging a genuine brute-force attack. This automates the attempt-vs-attack judgment an analyst makes by hand.

## 6. MITRE ATT&CK

| Tactic | Technique | ID |
|---|---|---|
| Credential Access | Brute Force | T1110 |

## 7. Skills evidenced

SIEM deployment and administration · Linux server setup · systemd services · network and virtualization troubleshooting · log analysis · detection/correlation rule interpretation · MITRE ATT&CK mapping · real-time threat detection.

## 8. What I learned

- A SIEM turns a flood of low-severity events into a single high-severity, actionable alert via correlation rules — that aggregation is the core value.
- Most of the real work in standing up security tooling is **troubleshooting the environment** (virtualization, networking, package sources), not running the install command.
- Rule levels and composite rules are how a SOC prioritizes — the same attempt-vs-success instinct from manual log analysis, encoded into the platform.

---

*Prepared as a hands-on SOC analyst lab exercise (Lab 5 — Wazuh SIEM Build). Deployment verified against Wazuh 4.14.*
