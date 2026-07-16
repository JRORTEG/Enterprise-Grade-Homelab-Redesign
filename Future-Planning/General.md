# Future Homelab Projects (Ordered by Severity / Overall Benefit)

### 1. Firewall Rules for All VLANs | Completed
**Description:** Replace the current allow-all inter-VLAN rules on pfSense with explicit least-privilege ACLs matching the segmentation model in `IP_Assignment_Revised.md` (default-deny DMZ→internal, scoped Core DNS access, mgmt-plane isolation, etc.).
**Benefit:** Actually enforces the VLAN segmentation that's already designed on paper. This closes the single biggest live security gap in the lab and demonstrates real least-privilege firewall engineering.

### 2. Backup Solution for Proxmox
**Description:** Stand up a real backup solution for the Proxmox cluster, e.g. Proxmox Backup Server (dedicated or as a VM/LXC) targeting TrueNAS storage, with scheduled VM/LXC snapshots, retention policy, and a tested restore procedure. Currently no backup mechanism exists for any VM/LXC or Proxmox host config.
**Benefit:** Closes the lab's single largest unrecoverable-data-loss risk (disk failure, ransomware, accidental deletion, botched upgrade); a required safety net before other risky in-progress work, including the pending Proxmox VE 8→9 upgrade (`Open Items.md` #15) and the Firewall/VPN redesigns, and a core enterprise ops practice for the resume.
**Status:** Interim solution decided 2026-07-14 — dual-boot PBS on the Gaming PC with a dedicated local 2TB drive (not TrueNAS-targeted as originally described above). Budget-driven stopgap; see `MGMT/Proxmox/Backup Solution.md` and `Open Items.md` #18–19 for details and the planned revisit.

### 3. Enterprise VPN / Remote Access Redesign
**Description:** Replace PiVPN (currently DMZ-hosted) with a purpose-built remote-access architecture that doesn't grant VPN clients DMZ-equivalent or unrestricted internal reach. Give connected clients their own dedicated subnet/zone, not the DMZ subnet, routed through pfSense under the same default-deny, least-privilege model used for every other VLAN, with explicitly scoped post-connect access instead of blanket internal reachability. Longer-term, integrate with the IAM project (Authentik/Keycloak) for SSO/MFA on VPN auth.
**Benefit:** Closes a live security gap where an internet-facing VPN endpoint could otherwise become a blanket bypass of the entire VLAN segmentation model; unblocks a clean finish to the DMZ (VLAN 40) firewall pass; demonstrates real enterprise remote-access design (dedicated VPN zone, least-privilege post-connect ACLs, IAM-backed auth). Strong resume signal alongside the other security projects.

### 4. Hardware Upgrade - Switching and Access Points
**Description:** Replace aging switch/AP hardware, starting with the Netgear GS308E. Its web management interface is HTTP-only (no HTTPS option), so switch admin credentials and session traffic cross the MGMT VLAN in cleartext. Select replacement switches/APs against explicit criteria: HTTPS-only (or HTTPS-capable) management, per-port VLAN tagging, PoE (for AP uplinks), and SNMP/syslog export for the future monitoring/SIEM stack.
**Benefit:** Closes a live cleartext-credential exposure on core network infrastructure management; HTTPS-capable gear is a prerequisite for trusting the MGMT VLAN as a true control-plane boundary. New APs with proper multi-SSID/VLAN tagging also unblock the still-unbuilt GUEST/IoT VLAN 70, and syslog/SNMP export feeds directly into the Security Homelab Projects (#10) and Grafana (#11) work.

### 5. Internal DNS
**Description:** Stand up split-horizon internal DNS (pfSense DNS Resolver + Pi-hole conditional forwarding, or a dedicated internal zone) so lab services resolve by hostname (e.g., `jellyfin.lab.local`) instead of hardcoded/ IPs.
**Benefit:** Foundational for everything downstream. Enables internal cert automation, cleaner reverse-proxy configs, and service discovery; reduces the hardcoded-IP sprawl visible across the IP assignment docs.

### 6. Dynamic DNS (DDNS)
**Description:** Configure a Dynamic DNS client to auto-update a DNS record whenever the WAN public IP changes, e.g. pfSense's built-in Dynamic DNS service (Services > Dynamic DNS) pointed at Cloudflare (API-token auth, matches existing Cloudflare usage) or another provider (DuckDNS, No-IP). Replaces any hardcoded public IP in VPN client configs, port-forward-dependent bookmarks, or reverse-proxy/remote-access references to DMZ services (Jellyfin, Minecraft).
**Benefit:** Eliminates manual IP updates every time the ISP rotates the public IP; keeps remote access (VPN, reverse-proxied services) working continuously without intervention. Fixes an active recurring breakage, not just a convenience.

### 7. IAM Solution
**Description:** Deploy a centralized identity provider (Authentik or Keycloak) for SSO/MFA across self-hosted services (Jellyfin, Grafana, future apps), replacing separate local logins per service.
**Benefit:** Centralizes authN/authZ, enables MFA lab-wide, mirrors enterprise IAM practices (strong resume value), and reduces credential sprawl/attack surface.

### 8. Infrastructure as Code (Terraform/Ansible)
**Description:** Specifics not yet decided, likely Terraform for Proxmox VM/LXC provisioning and Ansible for post-provision configuration management.
**Benefit:** Moves the lab from manually-clicked to reproducible and version-controlled; makes rebuilds/disaster-recovery trivial and is one of the highest-signal enterprise practices for a resume.

### 9. Implement IPv6 Networking
**Description:** Roll out dual-stack IPv6 alongside existing IPv4 addressing: enable IPv6 on WAN/pfSense, define per-VLAN IPv6 prefixes/subnets mirroring the existing VLAN segmentation model in `IP_Assignment_Revised.md`, and extend firewall/ACL rules to cover IPv6 traffic (not just IPv4) so default-deny/DMZ-isolation posture holds across both stacks.
**Benefit:** Modernizes the lab's core addressing model to match real enterprise dual-stack practice; closes a gap where IPv6-capable devices/ISPs could bypass IPv4-only firewall rules unnoticed; strong resume signal for enterprise network engineering.

### 10. Security Homelab Projects
See: [[Security_Homelab_Projects]]
**Description:** A bucket of hands-on security-engineering projects (Wazuh SIEM, EDR via LimaCharlie/Wazuh FIM, vulnerability scanning via OpenVAS, MITRE ATT&CK via TryHackMe SOC Level 1), already prioritized internally in the linked doc.
**Benefit:** Direct, hands-on resume-building experience with SIEM/EDR/vuln-management tooling; most effective once firewall rules and internal DNS exist (real log sources, real hostnames to monitor).

### 11. Grafana
**Description:** Deploy Grafana (with Prometheus/Loki) for centralized dashboards and visualization of lab metrics and logs across Proxmox, pfSense, TrueNAS, and containers.
**Benefit:** Adds observability into resource usage and health, and demonstrates monitoring/dashboarding skill. Pairs well with log data once Wazuh/SIEM is in place.

### 12. Proxmox MCP Server (AI-Assisted Infrastructure Interaction)
**Description:** Stand up an MCP (Model Context Protocol) server exposing the Proxmox API, so an AI agent system can query and manage VMs/LXCs, resource usage, and cluster state directly through conversation instead of the Proxmox web UI or manual CLI/API calls.
**Benefit:** Lets AI-assisted workflows (troubleshooting, provisioning, status checks) interact with the hypervisor directly. Speeds up day-to-day lab operations and pairs naturally with the IaC project (#8) once that's in place, letting an agent inspect actual cluster state rather than working blind.
