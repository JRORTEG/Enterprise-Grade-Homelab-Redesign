Assigned IP addresses to align with enterprise network segmentation practices.

## Addressing convention

Applied uniformly to every `/24` VLAN:

| Range         | Purpose                                         |
| ------------- | ----------------------------------------------- |
| `.1`          | Gateway (pfSense interface)                     |
| `.2 – .9`     | Reserved, additional network infra / future HA |
| `.10 – .49`   | Static server/service assignments               |
| `.50 – .99`   | Static reserved / DHCP reservations             |
| `.100 – .239` | DHCP dynamic pool                               |
| `.240 – .254` | Reserved / expansion buffer                     |

---

## VLAN 10: MGMT (10.0.10.0/24, gateway 10.0.10.1)

Infrastructure control-plane only. No end-user or application devices.

| Device         | IP         |
| -------------- | ---------- |
| pfSense Router | 10.0.10.1  |
| Proxmox Host   | 10.0.10.10 |
| Managed Switch | 10.0.10.11 |

**DHCP:** Disabled. Management-plane devices should always be static. A DHCP failure or lease conflict on this VLAN should never be able to lock you out of your own infrastructure.

---

## VLAN 20: CORE (10.0.20.0/24, gateway 10.0.20.1)

Internal infrastructure services (DNS, storage). No internet-facing exposure.

| Device  | IP         |
| ------- | ---------- |
| TrueNAS | 10.0.20.10 |
| Pi-hole | 10.0.20.53 |

*Note: Pi-hole's `.53` deliberately breaks the addressing convention as a DNS-port mnemonic, kept intentionally.*

**DHCP:** Static-only. If a dynamic pool is ever needed for temporary core-service testing, reserve `.100 – .150`.

---

## VLAN 30: APPS (10.0.30.0/24, gateway 10.0.30.1)

Reserved for future **internal-only** applications (no internet exposure). Currently no devices qualify. Both Jellyfin instances and the Minecraft server are internet-facing and now live in DMZ instead.

**DHCP:** Static range `.10 – .49` reserved for future app servers; dynamic pool `.100 – .199` for staging new app containers/VMs before promotion to a static IP.

---

## VLAN 40: DMZ (10.0.40.0/24, gateway 10.0.40.1)

Internet-facing services only. Fully static, no DHCP. Firewall rules should default-deny any connection initiated from DMZ toward Mgmt, Core, Apps, Lab, or Trusted Clients.

| Device                                                  | IP         | Notes                                                                                 |
| ------------------------------------------------------- | ---------- | ------------------------------------------------------------------------------------- |
| Minecraft (Velocity proxy + Paper backends, single LXC) | 10.0.40.10 | Port-forwarded 25565                                                                  |
| New Jellyfin                                            | 10.0.40.13 | Reverse-proxied for remote streaming                                                  |
| nginx (reverse proxy LXC)                               | 10.0.40.12 | Fronts DMZ services                                                                   |
| PiVPN                                                   | 10.0.40.11 | Internet-facing VPN endpoint                                                          |
| Torrenting                                              | 10.0.40.14 | High-risk P2P traffic, isolated                                                       |
| Old Jellyfin                                            | 10.0.40.15 | **Legacy, pending decommission** once migration to New Jellyfin is confirmed complete |
| MusiQ                                                   | 10.0.40.16 |                                                                                       |

**DHCP:** Disabled. Every DMZ host must be statically assigned so firewall rules can be written against known, fixed IPs.

---

## VLAN 50: LAB (10.0.50.0/24, gateway 10.0.50.1)

Sandbox for OS testing and disposable VMs. Isolated from production VLANs.

| Device                     | IP   |
| -------------------------- | ---- |
| OS Testing (ephemeral VMs) | DHCP |

**DHCP:** Dynamic pool `.50 – .240`, short lease time (1–4 hours) to accommodate frequent VM churn.

---

## VLAN 60: TRUSTED CLIENTS (10.0.60.0/24, gateway 10.0.60.1)

New VLAN. End-user devices, separated from both management infrastructure and internet-facing services.

| Device          | IP                            |
| --------------- | ----------------------------- |
| Main PC         | 10.0.60.10 (DHCP reservation) |
| PBS (Proxmox Backup Server) | 10.0.60.15 (static) |
| Personal Laptop | DHCP                          |
| Work Laptop     | DHCP                          |
| Cellphone       | DHCP                          |

**DHCP:** Dynamic pool `.50 – .199`. Main PC gets a DHCP reservation at `.10` for consistent access (game hosting, remote desktop, file shares); other client devices remain fully dynamic.

*Note: PBS dual-boots from the Main PC and doesn't belong here long-term — it's infrastructure control-plane and should sit on MGMT (VLAN 10). Parked on Trusted Clients temporarily since the Main PC has only one NIC. See `Future-Planning/Open Items.md` #24.*

---

## VLAN 70: GUEST / IoT *(future addition, no devices yet)*

Not yet implemented, but planned to complete the segmentation model before adding any smart-home or guest-network devices.

- Client isolation enabled (devices cannot see each other).
- Internet-only egress, no route to any other VLAN.
- Dynamic pool `.50 – .199` once created, gateway `10.0.70.1`.

---

## Security & segmentation notes

- **Default-deny between VLANs.** Only explicitly required inter-VLAN traffic should be allowed (e.g., Trusted Clients → Core for DNS, Mgmt → all VLANs for administration). Everything else denied by default on the pfSense firewall.
- **DMZ isolation is the highest-priority rule.** DMZ hosts (Minecraft, Jellyfin x2, PiVPN, Torrenting) must never be able to initiate connections into Mgmt, Core, Apps, Lab, or Trusted Clients. This contains a compromise of any single internet-facing service.
- **No DHCP on Mgmt or DMZ.** Both are fully static so firewall rules, monitoring, and troubleshooting are always working against known, fixed addresses.
- **Trunk (802.1Q) from the managed switch to pfSense**, as already reflected in the network diagram, carries all VLANs; access ports on the switch should be locked to a single VLAN per port unless a device explicitly needs tagged trunking.
- **Old Jellyfin decommission:** once the New Jellyfin instance is validated as a full replacement, retire the old instance and remove its entry (10.0.40.15) from this document.
