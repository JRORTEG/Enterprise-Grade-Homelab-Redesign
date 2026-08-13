# VLAN 50 - LAB Firewall Rules

Subnet: 10.0.50.0/24, gateway 10.0.50.1. Disposable OS-testing VMs, dynamic DHCP pool `.50-.240`, short (1-4hr) leases per `IP_Assignment_Revised.md`. No static hosts. Rules are subnet-scoped, not per-host.

Default posture: deny all, explicit allow-list only. Isolated from every production VLAN.

This doc is the **authoritative source** for every rule where Lab (VLAN 50) is the traffic source. Per the ingress-interface principle (`Firewall.md` guiding principles, discovered/modeled in `VLAN_40.md`), a rule can only be configured on the tab where the packet ingresses, and Lab-initiated traffic always ingresses at the VLAN 50 interface. That includes the three temp-access rules (4-6) below, which were previously mis-documented as living on the destination VLANs' tabs.

## Inbound to VLAN 50

| # | Source | Destination | Proto/Port | Purpose |
|---|---|---|---|---|
| 1 | any (WAN, all internal VLANs) | VLAN 50 | any | Default-deny. No inbound access. VM interaction happens via Proxmox console (Mgmt plane, `VLAN_10.md`), not a network-layer rule here. |

## Outbound from VLAN 50

| #   | Source                                               | Destination                                                   | Proto/Port            | Purpose                                                                                   |
| --- | ---------------------------------------------------- | ------------------------------------------------------------- | --------------------- | ----------------------------------------------------------------------------------------- |
| 2   | VLAN 50                                              | Pi-hole 10.0.20.53                                            | UDP/TCP 53            | DNS resolution. Destination-side doc at `VLAN_20.md` rule 3                              |
| 3   | VLAN 50                                              | WAN (any)                                                     | any/any               | Broad outbound. OS installs/updates/arbitrary software testing ⚠ see risk note below     |
| 4   | *(disabled)* Lab VM via alias `MGMT_TEMP_ADMIN_HOST` | VLAN 10 (10.0.10.0/24)                                        | TCP 443, 8006, 22, 80 | Temp real-data testing access against Mgmt. Destination-side doc at `VLAN_10.md` rule 7  |
| 5   | *(disabled)* Lab VM via alias `CORE_TEMP_ADMIN_HOST` | VLAN 20 (10.0.20.0/24)                                        | TCP 445, 443, 80, 22  | Temp real-data testing access against Core. Destination-side doc at `VLAN_20.md` rule 13 |
| 6   | *(disabled)* Lab VM via alias `APPS_TEMP_ADMIN_HOST` | VLAN 30 (10.0.30.0/24)                                        | TCP 3000              | Temp real-data testing access against Apps. Destination-side doc at `VLAN_30.md` rule 2  |
| 7   | VLAN 50                                              | all other internal VLANs (DMZ, Trusted Clients, except above) | any                   | Default-deny                                                                              |

## Temporary access procedures (rules 4-6)

Configured and toggled **here**, on the VLAN 50 (Lab) tab. Per the ingress-interface principle, this is the only tab where Lab-sourced traffic is ever evaluated, so the alias-gated rule must live here regardless of which VLAN it grants access into. `VLAN_10.md`, `VLAN_20.md`, and `VLAN_30.md` each carry a destination-side documentation row only, pointing back to this section.

Same pattern for all three (Mgmt/rule 4, Core/rule 5, Apps/rule 6):

1. Edit the relevant alias (`MGMT_TEMP_ADMIN_HOST`, `CORE_TEMP_ADMIN_HOST`, or `APPS_TEMP_ADMIN_HOST`) in pfSense, set it to the specific Lab VM's IP.
2. Enable the corresponding rule.
3. Do the test.
4. Disable the rule again. Optionally clear the alias IP.

Rules stay disabled by default. Never left enabled between sessions.
