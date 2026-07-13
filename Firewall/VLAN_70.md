# GUEST/IoT Firewall Rules

Subnet: 10.0.70.0/24, gateway 10.0.70.1. Recommended future addition per `IP_Assignment_Revised.md`. No devices, switch port, pfSense interface, or DHCP scope exist yet. Dynamic pool `.50-.199` once built. No static hosts, so rules are subnet-scoped, not per-host, same convention as `VLAN_50.md`.

Default posture: deny all, explicit allow-list only. Strictest VLAN in this project. Client isolation between devices, internet-only egress, zero standing inbound of any kind (not even Mgmt diagnostics).

## Inbound to VLAN 70

| # | Source | Destination | Proto/Port | Purpose |
|---|---|---|---|---|
| 1 | any (WAN, all internal VLANs incl. Mgmt) | VLAN 70 | any | Default-deny, no exceptions. Unlike every other VLAN in this project, there is no Mgmt(10)→VLAN ICMP diagnostic rule here. Guest/IoT gets zero standing inbound access at all. |

## Outbound from VLAN 70

| # | Source | Destination | Proto/Port | Purpose |
|---|---|---|---|---|
| 2 | VLAN 70 | Pi-hole 10.0.20.53 | UDP/TCP 53 | DNS. Narrow exception to the "internet-only egress" principle, needed for ad-blocking/filtering on IoT devices. This rule is already correctly authoritative here per the ingress-interface principle (`Firewall.md` guiding principles). VLAN 70 is the traffic source, so the rule belongs on this tab. Once this VLAN is built, add a **documentation-only** row to `VLAN_20.md`'s Inbound table pointing back to this rule (same convention as `VLAN_60.md` rule 3 / `VLAN_20.md` rule 1), not a duplicate operative rule. |
| 3 | VLAN 70 | WAN (any) | any/any | Broad outbound. General internet access for guest devices and IoT firmware/app traffic, consistent with the "internet-only egress" recommendation. |
| 4 | VLAN 70 | all other internal VLANs (except rule 2) | any | Default-deny. |
