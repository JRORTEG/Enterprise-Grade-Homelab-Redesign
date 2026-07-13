# Enterprise-Grade Homelab

A documentation vault for a real, running home network, redesigned from a flat "everything on one subnet" setup into an enterprise-style segmented architecture. Built and maintained as a portfolio artifact for system administration, network administration, and networking roles. Every doc here reflects an actual configuration change made on real hardware, not a theoretical exercise.

## About this project

This repo is the design record for a home network rebuilt around VLAN segmentation, least-privilege firewall rules, and standard sysadmin services (DNS, file storage, virtualization, containerized application hosting). Containing documentation for IP assignment tables, per-VLAN firewall rule tables, deployment guides, and a running list of open items and technical debt. The goal is to show the reasoning behind network design decisions, not just the end state.

## Network architecture

Seven VLANs, each a `/24`, with a consistent addressing convention: `.1` gateway, `.2 – .9` reserved infra, `.10 – .49` static services, `.50 – .99` static/DHCP reservations, `.100 – .239` DHCP pool, `.240 – .254` reserved/expansion.

| VLAN | Name            | Subnet       | Purpose                                                                          | DHCP                                      |
| ---- | --------------- | ------------ | -------------------------------------------------------------------------------- | ----------------------------------------- |
| 10   | MGMT            | 10.0.10.0/24 | Infra control-plane only (pfSense, Proxmox, switch)                              | Disabled, static only                     |
| 20   | CORE            | 10.0.20.0/24 | Internal infra services (TrueNAS, Pi-hole)                                       | Static-only recommended                   |
| 30   | APPS            | 10.0.30.0/24 | Reserved, future internal-only apps                                              | Static `.10 - .49`, dynamic `.100 - .199` |
| 40   | DMZ             | 10.0.40.0/24 | Internet-facing services only (Minecraft, Jellyfin x2, nginx, PiVPN, torrenting) | Disabled, static only                     |
| 50   | LAB             | 10.0.50.0/24 | Disposable OS-testing VMs                                                        | Dynamic, short lease                      |
| 60   | TRUSTED CLIENTS | 10.0.60.0/24 | End-user devices                                                                 | Mostly dynamic                            |
| 70   | GUEST/IoT       | 10.0.70.0/24 | Recommended, not yet built                                                       | Dynamic (future)                          |

Full detail: [`General/IP Assignment.md`](General/IP%20Assignment.md)

**Core security principle:** default-deny between every VLAN. DMZ isolation is the highest-priority rule: DMZ hosts (internet-facing services) must never be able to initiate a connection into Mgmt, Core, Apps, Lab, or Trusted Clients.

## Firewall / least-privilege ACL project

The bulk of the security engineering work: replacing "allow all inter-VLAN traffic" with explicit, rule-by-rule least-privilege ACLs on pfSense. Roughly 80 explicit rules across all 9 VLAN/WAN interfaces, including:

- **Ingress-based rule modeling.** Every rule lives on the interface where the traffic actually originates (pfSense's stateful filtering model), with cross-reference tables on the destination side for readability.
- **Scoped exceptions** to the default-deny rule, each locked to a single source host, single destination host, and single port. Example: Jellyfin → TrueNAS over SMB/445 for media library access, Grafana → Mgmt-plane hosts over TCP/9100 for metrics scraping. No blanket subnet-to-subnet exceptions.
- **NAT port-forwarding** for internet-facing DMZ services (Minecraft, WireGuard VPN, reverse-proxied web traffic), including auto-generated WAN filter rules, anti-spoofing (RFC1918/bogon filtering), and identification of the biggest open attack surface (an intentionally broad any/any rule for torrenting P2P traffic, flagged for future scoping).
- **Temporary access procedures** using disabled aliases for one-off admin/debug access into locked-down VLANs, so nothing stays open between sessions.

Start here: [`Firewall/Project Overview.md`](Firewall/Project%20Overview.md), the index and per-VLAN progress tracker linking to `VLAN_10.md` through `VLAN_70.md` and `WAN.md`.

## Deployed services: DMZ

A working example of the DMZ tier in production: dual Minecraft servers (Vanilla + Modded) running as separate Docker containers behind a Velocity reverse proxy, all inside a single Proxmox LXC. Covers container orchestration with per-service `docker-compose.yml` files on a shared external Docker network, modern player-info-forwarding with a shared secret handshake between proxy and backends, and real troubleshooting history (network-driver deprecation, forwarding config crash loops, a chunk-index world-generation bug).

- Deployment guide: [`DMZ/Minecraft/Guide.md`](DMZ/Minecraft/Guide.md)
- Architecture decisions and implementation history: [`DMZ/Minecraft/Minecraft Server Home Lab Project Summary.md`](DMZ/Minecraft/Minecraft%20Server%20Home%20Lab%20Project%20Summary.md)

## Docker networking reference

A from-scratch reference doc on Docker network drivers (bridge, host, overlay, macvlan/ipvlan) and when to use each, written to be reused across future container deployments rather than re-derived per project: [`General/Docker/Docker Network.md`](General/Docker/Docker%20Network.md)

## Repo map

| Path               | Contents                                                                         |
| ------------------ | -------------------------------------------------------------------------------- |
| `General/`         | IP assignment plan (current + historical), Docker networking reference, Diagrams |
| `Firewall/`        | Per-VLAN pfSense ACL documentation, WAN rules, project index                     |
| `DMZ/`             | Internet-facing service deployments (Minecraft dual-server behind Velocity)      |
| `Future-Planning/` | Project backlog, security-engineering roadmap, open items log                    |
| `Apps/`            | Reserved for future internal-only app documentation (VLAN 30)                    |


## Roadmap

Ordered backlog of what's next, from [`Future-Planning/General.md`](Future-Planning/General.md):

- **Backup solution.** Proxmox Backup Server targeting TrueNAS, with tested restore procedures (closes the current largest data-loss risk)
- **VPN redesign.** Move PiVPN off the DMZ into a dedicated, scoped VPN zone under default-deny
- **Hardware upgrade.** Replace HTTP-only managed switch with HTTPS-capable, per-port VLAN tagging
- **Internal DNS.** Split-horizon DNS via pfSense + Pi-hole, hostname-based service resolution
- **Dynamic DNS**, **IAM (SSO/MFA)**, **Infrastructure as Code (Terraform/Ansible)**, **dual-stack IPv6**, **centralized monitoring (Grafana/Prometheus/Loki)**

There's also a dedicated security-engineering track (SIEM via Wazuh, EDR via LimaCharlie, vulnerability scanning via OpenVAS, and MITRE ATT&CK-mapped exercises), detailed in [`Future-Planning/Security/Security_Homelab_Projects.md`](Future-Planning/Security/Security_Homelab_Projects.md).

## Contact

- LinkedIn: https://www.linkedin.com/in/jorge-ortega-00b16a75/
- Email: Jorgeortega0723@hotmail.com
