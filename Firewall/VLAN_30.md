# VLAN 30 - APPS Firewall Rules

Subnet: 10.0.30.0/24, gateway 10.0.30.1. Currently empty but reserved for future internal-only apps. This pass plans concretely around the anticipated first occupant: a Grafana + Prometheus + Loki monitoring stack, single all-in-one host, placeholder IP **10.0.30.10**.

Default posture: deny all, explicit allow-list only.

## Inbound to VLAN 30

pfSense evaluates firewall rules at the **ingress interface**, not the destination. Both specific rows below are sourced from another internal VLAN. The real rule is configured on that source VLAN's own tab, not on VLAN 30's.

| #   | Source                                              | Destination            | Proto/Port | Purpose                                                                                                                                | Rule lives on           |
| --- | --------------------------------------------------- | ---------------------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------- | ----------------------- |
| 1   | Trusted Clients (60)                                | Grafana 10.0.30.10     | TCP 3000   | Web UI dashboards (default port, verify)                                                                                               | `VLAN_60.md` rule 14    |
| 2   | *(disabled)* alias `APPS_TEMP_ADMIN_HOST` (Lab, 50) | VLAN 30 (10.0.30.0/24) | TCP 3000   | Temporary real-data testing access                                                                                                     | `VLAN_50.md` rule 6     |
| 3   | any                                                 | VLAN 30                | any        | Default-deny \| pfSense implicit deny covers this; explicit logged deny rules may be added to each source interface tab for visibility | (source interface tabs) |

## Outbound from VLAN 30

| #   | Source             | Destination                                           | Proto/Port | Purpose                                                                              |
| --- | ------------------ | ----------------------------------------------------- | ---------- | ------------------------------------------------------------------------------------ |
| 4   | Grafana 10.0.30.10 | Pi-hole 10.0.20.53                                    | UDP/TCP 53 | DNS resolution                                                                       |
| 5   | Grafana 10.0.30.10 | WAN (any)                                             | TCP 443    | HTTPS updates/plugins                                                                |
| 6   | Grafana 10.0.30.10 | WAN (any)                                             | TCP 80     | HTTP updates/mirrors                                                                 |
| 7   | Grafana 10.0.30.10 | pfSense 10.0.10.1                                     | TCP 9100   | Metrics scrape ⚠ port/exporter unconfirmed. **Mgmt isolation exception, see below** |
| 8   | Grafana 10.0.30.10 | Proxmox 10.0.10.10                                    | TCP 9100   | Metrics scrape ⚠ port/method unconfirmed. **Mgmt isolation exception, see below**   |
| 9   | Grafana 10.0.30.10 | TrueNAS 10.0.20.10                                    | TCP 9100   | Metrics scrape ⚠ port unconfirmed                                                    |
| 10  | Grafana 10.0.30.10 | Pi-hole 10.0.20.53                                    | TCP 80/443 | Stats scrape ⚠ port/method unconfirmed                                               |
| 11  | Grafana 10.0.30.10 | Minecraft 10.0.40.10                                  | TCP 9100   | Metrics scrape ⚠ port unconfirmed                                                    |
| 12  | Grafana 10.0.30.10 | New Jellyfin 10.0.40.11                               | TCP 9100   | Metrics scrape ⚠ port unconfirmed                                                    |
| 13  | Grafana 10.0.30.10 | nginx 10.0.40.12                                      | TCP 9100   | Metrics scrape ⚠ port unconfirmed                                                    |
| 14  | Grafana 10.0.30.10 | PiVPN 10.0.40.13                                      | TCP 9100   | Metrics scrape ⚠ port unconfirmed                                                    |
| 15  | Grafana 10.0.30.10 | Torrenting 10.0.40.14                                 | TCP 9100   | Metrics scrape ⚠ port unconfirmed. See risk note below                              |
| 16  | VLAN 30            | all other internal VLANs (except rules 4, 7-15 above) | any        | Default-deny                                                                         |

## Temporary Lab access procedure

Rule 2 above is gated by alias `APPS_TEMP_ADMIN_HOST`, matched against a Lab (VLAN 50) source host. Per the ingress-interface principle, the rule and its enable/disable toggle is actually configured on the **VLAN 50 (Lab)**. The full toggle procedure is documented in `VLAN_50.md`.

## Mgmt isolation exception (rules 7-8)

Per this project's mgmt-plane isolation principle, VLAN 10 is reachable only for admin purposes, not general traffic. Metrics scraping isn't admin activity, so rules 7-8 are a sanctioned, explicitly scoped exception:

- Single source host (Grafana, 10.0.30.10), not the whole Apps subnet.
- Single destination host per rule (pfSense or Proxmox), not all of Mgmt.
- Single port per rule, not "any."