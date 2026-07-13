# CORE Firewall Rules

Subnet: 10.0.20.0/24, gateway 10.0.20.1. Devices: TrueNAS (.10), Pi-hole (.53). No internet-facing exposure.

Default posture: deny all, explicit allow-list only.

## Inbound to VLAN 20

pfSense evaluates firewall rules at the **ingress interface**, not the destination. Every row below is sourced from another internal VLAN. The real rule is configured on that source VLAN's own tab, not on VLAN 20's. **No pfSense rules are created on the VLAN 20 tab from this section** (except rule 6, the sanctioned DMZ exception, which is likewise declared and configured on VLAN 40's tab).

| #   | Source                                              | Destination            | Proto/Port           | Purpose                                                                  | Rule lives on           |
| --- | ---------------------------------------------------- | ---------------------- | -------------------- | ------------------------------------------------------------------------ | ------------------------ |
| 1   | Trusted Clients (60)                                | Pi-hole 10.0.20.53     | UDP/TCP 53           | DNS                                                                      | `VLAN_60.md` rule 3      |
| 2   | Mgmt (10)                                           | Pi-hole 10.0.20.53     | UDP/TCP 53           | DNS                                                                      | `VLAN_10.md` rule 9      |
| 3   | Lab (50)                                            | Pi-hole 10.0.20.53     | UDP/TCP 53           | DNS                                                                      | `VLAN_50.md` rule 2      |
| 4   | Apps (30)                                           | Pi-hole 10.0.20.53     | UDP/TCP 53           | DNS (preemptive, no devices in Apps yet)                                | `VLAN_30.md` rule 4      |
| 5   | Trusted Clients (60)                                | TrueNAS 10.0.20.10     | TCP 445              | SMB file share                                                           | `VLAN_60.md` rule 4      |
| 6   | **New Jellyfin only**, DMZ 10.0.40.11              | TrueNAS 10.0.20.10     | TCP 445              | SMB media access. ⚠ See DMZ exception note below.                        | `VLAN_40.md` rule 12     |
| 7   | Trusted Clients (60)                                | TrueNAS 10.0.20.10     | TCP 443              | Web UI admin                                                             | `VLAN_60.md` rule 5      |
| 8   | Trusted Clients (60)                                | Pi-hole 10.0.20.53     | TCP 80               | Web UI admin                                                             | `VLAN_60.md` rule 6      |
| 9   | Trusted Clients (60)                                | Pi-hole 10.0.20.53     | TCP 22               | SSH admin                                                                | `VLAN_60.md` rule 7      |
| 10  | Mgmt (10)                                           | TrueNAS 10.0.20.10     | TCP 443              | Web UI admin                                                             | `VLAN_10.md` rule 14     |
| 11  | Mgmt (10)                                           | Pi-hole 10.0.20.53     | TCP 80               | Web UI admin                                                             | `VLAN_10.md` rule 15     |
| 12  | Mgmt (10)                                           | Pi-hole 10.0.20.53     | TCP 22               | SSH admin                                                                | `VLAN_10.md` rule 16     |
| 13  | *(disabled)* alias `CORE_TEMP_ADMIN_HOST` (Lab, 50) | VLAN 20 (10.0.20.0/24) | TCP 445, 443, 80, 22 | Temporary real-data testing access. Procedure in `VLAN_50.md`.           | `VLAN_50.md` rule 5      |
| 14  | any                                                  | VLAN 20                | any                  | Default-deny. pfSense implicit deny covers this; explicit logged deny rules may be added to each source interface tab for visibility | (source interface tabs) |

## Outbound from VLAN 20

| #   | Source             | Destination                                                             | Proto/Port | Purpose                                                          |
| --- | ------------------ | ----------------------------------------------------------------------- | ---------- | ---------------------------------------------------------------- |
| 15  | Pi-hole 10.0.20.53 | WAN (any)                                                               | UDP/TCP 53 | Upstream DNS resolution                                          |
| 16  | TrueNAS 10.0.20.10 | WAN (any)                                                               | TCP 443    | Updates, plugins, cert renewal                                   |
| 17  | TrueNAS 10.0.20.10 | WAN (any)                                                               | TCP 80     | HTTP (ACME challenge / mirrors)                                  |
| 18  | VLAN 20            | all other internal VLANs (Mgmt, Apps, DMZ, Lab, Trusted Clients, Guest) | any        | Default-deny                                                     |

## Temporary Lab access procedure | See `VLAN 50.md`

Rule 13 above is gated by alias `CORE_TEMP_ADMIN_HOST`, matched against a Lab (VLAN 50) source host. Per the ingress-interface principle, the rule and its enable/disable toggle is actually configured on the **VLAN 50 (Lab)**, not here. The full toggle procedure is documented in `VLAN_50.md`.

## DMZ exception note (rule 6)

Rule 6 is the **only sanctioned exception** to the DMZ-never-initiates-into-internal-VLANs principle documented anywhere in this firewall project so far. It exists because New Jellyfin (DMZ, 10.0.40.11) reads media from TrueNAS. Scoped as tightly as possible:

- Single source host (10.0.40.11), not the DMZ subnet.
- Single destination host (TrueNAS 10.0.20.10), not all of Core.
- Single protocol (SMB/445), not "any."