# VLAN 80 - IOT Firewall Rules

Subnet: 10.0.80.0/24, gateway 10.0.80.1. New addition per `General/IP Assignment.md`. No devices, switch port, pfSense interface, or DHCP scope exist yet. Dynamic pool `.50-.199` once built. No static hosts, so rules are subnet-scoped, not per-host, same convention as `VLAN_50.md` and `VLAN_70.md`.

Split off from a combined GUEST/IoT VLAN 70 so smart-home/IoT devices get their own dedicated VLAN separate from guest wireless clients — see `VLAN_70.md` and `Hardware/Access Point/Plan.md`. Wireless AP for this VLAN: TP-Link EAP723, tagged SSID over trunk (shared AP with Trusted Clients and Guest SSIDs).

Default posture: deny all, explicit allow-list only. Same strictness as Guest. Client isolation between devices, internet-only egress, zero standing inbound of any kind (not even Mgmt diagnostics).

## Inbound to VLAN 80

| # | Source | Destination | Proto/Port | Purpose |
|---|---|---|---|---|
| 1 | any (WAN, all internal VLANs incl. Mgmt) | VLAN 80 | any | Default-deny, no exceptions. No Mgmt(10)→VLAN ICMP diagnostic rule here, same posture as Guest — IoT gets zero standing inbound access at all. |

## Outbound from VLAN 80

| # | Source | Destination | Proto/Port | Purpose |
|---|---|---|---|---|
| 2 | VLAN 80 | Pi-hole 10.0.20.53 | UDP/TCP 53 | DNS. Narrow exception to the "internet-only egress" principle, needed for ad-blocking/filtering on IoT devices. Rule is authoritative here per the ingress-interface principle (`Firewall.md` guiding principles) — VLAN 80 is the traffic source. Once this VLAN is built, add a **documentation-only** row to `VLAN_20.md`'s Inbound table pointing back to this rule, not a duplicate operative rule. |
| 3 | VLAN 80 | WAN (any) | any/any | Broad outbound. General internet access for IoT firmware/app/cloud-service traffic, consistent with internet-only egress. |
| 4 | VLAN 80 | all other internal VLANs (except rule 2) | any | Default-deny. |

## Open items

- VLAN not yet physically built — no pfSense interface, DHCP scope, or switch VLAN database entry. Logged as `Future-Planning/Open Items.md` #29.
- Client isolation enforcement mechanism unconfirmed at the AP layer, same caveat as `VLAN_70.md` — a pfSense ACL alone won't stop same-subnet L2 traffic between IoT devices on the same Wi-Fi AP. Confirm AP-level client isolation is enabled on the EAP723's IoT SSID once provisioned.
