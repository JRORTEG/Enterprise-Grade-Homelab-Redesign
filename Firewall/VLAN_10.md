# MGMT Firewall Rules

Subnet: 10.0.10.0/24, gateway 10.0.10.1. Devices: pfSense (.1), Proxmox host (.10), Netgear GS308E switch (.11).

Default posture: deny all, explicit allow-list only.

## Inbound to VLAN 10

pfSense evaluates firewall rules at the **ingress interface**, not the destination. Traffic arriving *at* VLAN 10 from Trusted Clients (60) or Lab (50) is evaluated on that source interface's tab, not here. **No pfSense rules are created on the VLAN 10 tab from this section.**

The table below is a documentation reference showing what traffic is permitted into VLAN 10 and where each rule actually lives:

| #   | Source                                              | Destination            | Proto/Port            | Purpose                                                                                                                              | Rule lives on           |
| --- | --------------------------------------------------- | ---------------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ----------------------- |
| 1   | Trusted Clients (60)                                | pfSense 10.0.10.1      | TCP 443               | Web UI                                                                                                                               | `VLAN_60.md` rule 8     |
| 2   | Trusted Clients (60)                                | pfSense 10.0.10.1      | TCP 22                | SSH                                                                                                                                  | `VLAN_60.md` rule 9     |
| 3   | Trusted Clients (60)                                | Proxmox 10.0.10.10     | TCP 8006              | Web UI                                                                                                                               | `VLAN_60.md` rule 10    |
| 4   | Trusted Clients (60)                                | Proxmox 10.0.10.10     | TCP 22                | SSH                                                                                                                                  | `VLAN_60.md` rule 11    |
| 5   | Trusted Clients (60)                                | Switch 10.0.10.11      | TCP 80                | Web UI (HTTP only, see risk note)                                                                                                    | `VLAN_60.md` rule 12    |
| 6   | Trusted Clients (60)                                | VLAN 10 (10.0.10.0/24) | ICMP echo-request     | Troubleshooting ping                                                                                                                 | `VLAN_60.md` rule 13    |
| 7   | *(disabled)* alias `MGMT_TEMP_ADMIN_HOST` (Lab, 50) | VLAN 10 (10.0.10.0/24) | TCP 443, 8006, 22, 80 | Temporary real-data testing access. Procedure in `VLAN_50.md`.                                                                       | `VLAN_50.md` rule 4     |
| 8   | any                                                 | VLAN 10                | any                   | Default-deny: pfSense implicit deny covers this; explicit logged deny rules may be added to each source interface tab for visibility | (source interface tabs) |

## Outbound from VLAN 10

| #   | Source  | Destination                                      | Proto/Port        | Purpose                                                                        |
| --- | ------- | ------------------------------------------------ | ----------------- | ------------------------------------------------------------------------------ |
| 9   | VLAN 10 | Core Pi-hole 10.0.20.53                          | TCP/UDP 53        | DNS resolution - destination-side doc at `VLAN_20.md` rule 2                   |
| 10  | VLAN 10 | WAN (any)                                        | TCP 443           | HTTPS updates (Proxmox/pfSense repos)                                          |
| 11  | VLAN 10 | WAN (any)                                        | TCP 80            | HTTP updates (mirrors using plain HTTP)                                        |
| 12  | VLAN 10 | WAN (any)                                        | UDP 123           | NTP time sync                                                                  |
| 13  | VLAN 10 | Trusted Clients (10.0.60.0/24)                   | ICMP echo-request | Troubleshooting/diagnostics ping - destination-side doc at `VLAN_60.md` rule 1 |
| 14  | VLAN 10 | TrueNAS 10.0.20.10                               | TCP 443           | Web UI admin - destination-side doc at `VLAN_20.md` rule 10                    |
| 15  | VLAN 10 | Pi-hole 10.0.20.53                               | TCP 80            | Web UI admin - destination-side doc at `VLAN_20.md` rule 11                    |
| 16  | VLAN 10 | Pi-hole 10.0.20.53                               | TCP 22            | SSH admin - destination-side doc at `VLAN_20.md` rule 12                       |
| 17  | VLAN 10 | all other internal VLANs (Apps, DMZ, Lab, Guest) | any               | Default-deny                                                                   |

Note on rule 17: Proxmox's own VM traffic into other VLANs rides the hypervisor's virtual bridges, not this L3 ACL, and is unaffected by this deny. Core (20) and Trusted Clients (60) are both excluded from this catch-all. They're covered by the explicit allow rules above (9, 13–16).

## Temporary Lab access procedure | see `VLAN_50.md`

Rule 7 above is gated by alias `MGMT_TEMP_ADMIN_HOST`, matched against a Lab (VLAN 50) source host. Per the ingress-interface principle, that means the rule and its enable/disable toggle is actually configured and edited on the **VLAN 50 (Lab)** pfSense tab, not here, since Lab-sourced traffic ingresses at the Lab interface. The full toggle procedure is documented in `VLAN_50.md`.