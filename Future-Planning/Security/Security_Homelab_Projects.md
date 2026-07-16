# Security Home-Lab Projects
*Gap-filling projects for the Junior Security Engineer skillset, all runnable on a standard Proxmox home lab.*

---

## 1. SIEM — Deploy Wazuh on Proxmox

**What it covers**: SIEM, log ingestion, detection rules, alert triage, log analysis

**Steps**
1. Spin up a new Proxmox VM (Ubuntu 22.04, 4 GB RAM, 50 GB disk).
2. Install Wazuh All-in-One: `curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh && bash wazuh-install.sh -a`
3. Install the Wazuh agent on your other lab VMs (Linux and Windows) and enroll them in the manager.
4. Forward pfSense syslog to Wazuh (pfSense → Status → System Logs → Settings → Remote logging).
5. Open the Wazuh dashboard → explore Security Events, write a custom detection rule, and run a simulated brute-force to trigger an alert.
6. Document: "Onboarded 3 log sources; authored detection rule for SSH brute-force; analyzed alert timelines."

**Outcome**
> Deployed Wazuh SIEM in a virtualized home-lab environment; onboarded pfSense firewall, Linux, and Windows log sources; authored custom detection rules and conducted log analysis to surface anomalous activity.

**Time estimate**: 4–6 hours  
**Cost**: Free

---

## 2. EDR — LimaCharlie or Wazuh FIM

**What it covers**: EDR agent deployment, endpoint detection, alert triage, policy tuning

### Option A — LimaCharlie (Recommended, real EDR)
1. Create a free account at limacharlie.io (free tier: 2 sensors).
2. Download and deploy the LC agent on one of your lab VMs (Windows or Linux).
3. Navigate to the Detection & Response (D&R) rules — review built-in rules and clone/edit one.
4. Simulate a detection: run `mimikatz.exe` (test/simulation tool) or a simple PowerShell `Invoke-Expression` — observe the alert.
5. Perform a remediation action (isolate endpoint, kill process) from the LC console.
6. Document your tuning: which rules generated noise, what you suppressed and why.

### Option B — Wazuh FIM (if already running Wazuh from Project 1)
1. Enable File Integrity Monitoring in the Wazuh agent config (`ossec.conf`).
2. Add a monitored directory (e.g., `/etc/` on Linux, `C:\Windows\System32\` on Windows).
3. Create a file in the monitored path and observe the FIM alert in the dashboard.
4. Configure an active response to automatically block an IP after N failed SSH attempts.

**Outcome**
> Deployed EDR agents across lab endpoints using LimaCharlie; triaged endpoint detections, performed simulated incident response actions, and tuned detection rules to reduce false positives while maintaining threat visibility.

**Time estimate**: 3–5 hours  
**Cost**: Free

---

## 3. MITRE ATT&CK — TryHackMe SOC Level 1 Path

**What it covers**: MITRE ATT&CK TTPs, SOC alert triage, phishing analysis, log analysis, incident response

**Steps**
1. Create a free account at tryhackme.com.
2. Complete the **SOC Level 1** learning path (free rooms included). Key rooms:
   - *Cyber Defense Frameworks* — covers MITRE ATT&CK directly
   - *Phishing Analysis Fundamentals* — email security (maps to Mimecast skills)
   - *Splunk: Basics* — intro SIEM querying
   - *Investigating with Splunk* — hands-on log analysis and threat hunting
3. After each room, open **MITRE ATT&CK Navigator** (attack.mitre.org/versions/v14/software/) and highlight the TTPs you encountered.
4. Export a screenshot of your navigator layer — this is a portfolio artifact.

**Bonus**: If you want to go deeper, the **Blue Team Labs Online** free tier has hands-on SOC alert scenarios.

**Outcome**
> Applied the MITRE ATT&CK framework to map adversary TTPs across SOC simulation exercises on TryHackMe (SOC Level 1 path); practiced log analysis, phishing investigation, and alert triage using Splunk.

**Time estimate**: 8–12 hours (spread across weekends)  
**Cost**: Free

---

## 4. Vulnerability Scanning — OpenVAS (Greenbone Community) on Proxmox

**What it covers**: Vulnerability management, CVE prioritization, remediation tracking, scan policy configuration

**Steps**
1. Deploy the Greenbone Community Edition VM on Proxmox — grab the pre-built `.ova` or use the official install script on Debian 12:
   ```
   curl -f -L https://greenbone.github.io/docs/latest/_static/setup-and-start-greenbone-community-edition.sh | sh
   ```
2. Log into the web UI (port 9392 by default), create a scan target pointing at your lab network.
3. Run a Full and Fast scan against your lab VMs.
4. Review findings → filter by CVSS score → identify the top 5 Critical/High CVEs on your lab hosts.
5. Pick one finding, research the CVE on nvd.nist.gov, apply the patch or mitigation, and re-scan to confirm remediation.
6. Document in a simple table: CVE ID, CVSS score, affected host, remediation action, re-scan result.

**Outcome**
> Conducted network vulnerability assessments using OpenVAS/Greenbone in a virtualized lab; reviewed and prioritized CVE findings by CVSS severity; tracked remediation actions and validated patches through re-scanning.

**Time estimate**: 4–6 hours  
**Cost**: Free

---

## Quick-Start Order (Recommended Sequence)

| # | Project | Time | Why First |
|---|---|---|---|
| 1 | TryHackMe SOC Level 1 | Weekend 1 | Cheapest start; also covers Splunk SIEM basics |
| 2 | Wazuh SIEM on Proxmox | Weekend 2 | Builds on SOC knowledge; you already have Proxmox |
| 3 | LimaCharlie EDR | Weekend 3 | Fastest EDR experience; no new VM needed |
| 4 | OpenVAS Vulnerability Scanning | Weekend 4 | Rounds out the skillset |

Each completed project produces a concrete, verifiable outcome suitable for a resume or portfolio entry.

---

*All tools above are open-source or have free tiers. No paid subscriptions required.*
