# TRUSTED CLIENTS Firewall Rules

Subnet: 10.0.60.0/24, gateway 10.0.60.1. Devices: Gaming PC (10.0.60.10, DHCP reservation), Personal Laptop, Work Laptop, Cellphone (dynamic DHCP `.50-.199`) per `IP_Assignment_Revised.md`.

Default posture: deny all, explicit allow-list only. Per the ingress-interface principle, Trusted Clients is the traffic source for nearly all of its own rules, so this doc is the **authoritative source** for the Outbound table below. `VLAN_10.md`, `VLAN_20.md`, `VLAN_30.md`, and `VLAN_40.md` each carry destination-side documentation rows only, pointing back here.

## Inbound to VLAN 60

| # | Source | Destination | Proto/Port | Purpose | Rule lives on |
|---|---|---|---|---|---|
| 1 | Mgmt (10) | VLAN 60 | ICMP echo-request | Troubleshooting/diagnostics. Consistent with "Mgmt → all VLANs for administration" principle, scoped to ping only since client devices run no standing admin services | `VLAN_10.md` rule 13 |
| 2 | any (WAN, all other internal VLANs) | VLAN 60 | any | Default-deny. No WAN-inbound rule at all. Any hosting needs (game servers, etc.) go in DMZ instead, not on client devices. Explicit logged deny rules may be added to each source interface tab for visibility. | (source interface tabs) |

## Outbound from VLAN 60

| #   | Source  | Destination                             | Proto/Port        | Purpose                                                                                                                                   |
| --- | ------- | --------------------------------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| 3   | VLAN 60 | Pi-hole 10.0.20.53                      | UDP/TCP 53        | DNS. Destination-side doc at `VLAN_20.md` rule 1                                                                                          |
| 4   | VLAN 60 | TrueNAS 10.0.20.10                      | TCP 445           | SMB file share. Destination-side doc at `VLAN_20.md` rule 5                                                                               |
| 5   | VLAN 60 | TrueNAS 10.0.20.10                      | TCP 443           | Web UI admin. Destination-side doc at `VLAN_20.md` rule 7                                                                                 |
| 6   | VLAN 60 | Pi-hole 10.0.20.53                      | TCP 80            | Web UI admin. Destination-side doc at `VLAN_20.md` rule 8                                                                                 |
| 7   | VLAN 60 | Pi-hole 10.0.20.53                      | TCP 22            | SSH admin. Destination-side doc at `VLAN_20.md` rule 9                                                                                    |
| 8   | VLAN 60 | pfSense 10.0.10.1                       | TCP 443           | Web UI. Destination-side doc at `VLAN_10.md` rule 1                                                                                       |
| 9   | VLAN 60 | pfSense 10.0.10.1                       | TCP 22            | SSH. Destination-side doc at `VLAN_10.md` rule 2                                                                                          |
| 10  | VLAN 60 | Proxmox 10.0.10.10                      | TCP 8006          | Web UI. Destination-side doc at `VLAN_10.md` rule 3                                                                                       |
| 11  | VLAN 60 | Proxmox 10.0.10.10                      | TCP 22            | SSH. Destination-side doc at `VLAN_10.md` rule 4                                                                                          |
| 12  | VLAN 60 | Switch 10.0.10.11                       | TCP 80            | Web UI (HTTP only). Destination-side doc at `VLAN_10.md` rule 5                                                                           |
| 13  | VLAN 60 | VLAN 10 (10.0.10.0/24)                  | ICMP echo-request | Troubleshooting ping. Destination-side doc at `VLAN_10.md` rule 6                                                                         |
| 14  | VLAN 60 | Grafana 10.0.30.10                      | TCP 3000          | Web UI dashboards. Destination-side doc at `VLAN_30.md` rule 1                                                                            |
| 15  | VLAN 60 | Old Jellyfin 10.0.40.15                 | TCP 8096          | Direct home streaming. Destination-side doc at `VLAN_40.md` rule 5 ⚠ candidate for replacement with NAT hairpinning, see open items below |
| 16  | VLAN 60 | WAN (any)                               | any/any           | Broad outbound. General browsing/apps/gaming, no port restriction for end-user devices                                                    |
| 17  | VLAN 60 | all other internal VLANs (except above) | any               | Default-deny                                                                                                                              |
|     |         |                                         |                   |                                                                                                                                           |
