# VLAN 40 - DMZ Firewall Rules

Subnet: 10.0.40.0/24, gateway 10.0.40.1. Devices: Minecraft LXC (.10, Velocity+Paper), New Jellyfin (.11), nginx reverse proxy (.12), PiVPN (.13), Old Jellyfin (.15, pending decommission). Fully static, no DHCP.

Default posture: deny all, explicit allow-list only. **Highest-priority isolation VLAN**, DMZ hosts must never initiate connections into Mgmt, Core, Apps, Lab, or Trusted Clients.

## Inbound to VLAN 40

pfSense evaluates firewall rules at the **ingress interface**, not the destination. Traffic arriving *at* VLAN 40 from another interface  is evaluated on that source interface, not here.

The table below is a documentation reference showing what traffic is permitted into VLAN 40 and where each rule actually lives:

| #   | Source               | Destination                                   | Proto/Port | Purpose                                                                                                                               | Rule lives on            |
| --- | -------------------- | --------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| 1   | WAN (any)            | Minecraft 10.0.40.10                          | TCP 25565  | Velocity proxy, player connections                                                                                                    | `WAN.md` rules W1/F1     |
| 2   | WAN (any)            | nginx 10.0.40.12                              | TCP 443    | Reverse proxy HTTPS entry                                                                                                             | `WAN.md` rules W2/F2     |
| 3   | WAN (any)            | PiVPN 10.0.40.13                              | UDP 51820  | WireGuard VPN endpoint. Internal access scope deferred (see PiVPN note)                                                              | `WAN.md` rules W3/F3     |
| 4   | WAN (any)            | Torrenting 10.0.40.14                         | any/any    | Incoming P2P peer connections ⚠ broad, see risk note                                                                                  | `WAN.md` rules W4/F4     |
| 5   | Trusted Clients (60) | Old Jellyfin 10.0.40.15                       | TCP 8096   | Direct home streaming, bypasses nginx ⚠ port assumed default HTTP, verify                                                             | `VLAN_60.md` rule 15     |
| 6   | Apps (30)            | Minecraft/New Jellyfin/nginx/PiVPN/Torrenting | TCP 9100   | Grafana scrape                                                                                                                        | `VLAN_30.md` rules 11–15 |
| 7   | any                  | VLAN 40                                       | any        | Default-deny. pfSense implicit deny covers this; explicit logged deny rules may be added to each source interface tab for visibility | (source interface tabs)  |

## Outbound from VLAN 40

| #   | Source                  | Destination              | Proto/Port | Purpose                                                                                                                                              |
| --- | ----------------------- | ------------------------ | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| 8   | Minecraft 10.0.40.10    | WAN (any)                | TCP 443    | Mojang session-server auth (Velocity handles auth; backends run offline-mode per `DMZ/Minecraft/Guide.md`)                                           |
| 9   | nginx 10.0.40.12        | WAN (any)                | TCP 443/80 | ACME cert renewal / updates                                                                                                                          |
| 10  | Old Jellyfin 10.0.40.15 | WAN (any)                | TCP 443/80 | Metadata providers, updates                                                                                                                          |
| 11  | New Jellyfin 10.0.40.11 | WAN (any)                | TCP 443/80 | Metadata providers, updates. Pre-staged, not live yet                                                                                               |
| 12  | New Jellyfin 10.0.40.11 | TrueNAS 10.0.20.10       | TCP 445    | SMB media access. Declared here because VLAN 40 is where the connection originates. `VLAN_20.md` rule 6 documents the destination-side perspective. |
| 13  | VLAN 40                 | all other internal VLANs | any        | Default-deny. Explicit logged deny; enforces DMZ isolation. Rule 12 is the only sanctioned exception above.                                         |

