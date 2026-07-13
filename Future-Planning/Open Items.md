# Open Items

Running list of loose ends surfaced during other work in this repo, not full projects (those go in `General.md`), just things flagged for future review or decision. Resolved items stay in this list tagged `RESOLVED (date)` rather than being deleted, to keep a history. Numbering is never reused or reshuffled for this reason.

## 1. Switch admin UI is HTTP-only (no HTTPS/SSH)

Netgear GS308E doesn't support HTTPS or SSH for management. Plaintext credentials on the wire, even though access is scoped to Trusted Clients → Mgmt only.

**Source:** `Firewall/VLAN_10.md`, open risk section.
**Resolve by:** deciding on switch replacement (managed switch with HTTPS support), or in the meantime scoping the admin rule to a single admin host instead of all of VLAN 60.

## 2. Grafana host IP is a placeholder

`Firewall/VLAN_30.md` assumes the future Grafana/Prometheus/Loki stack lives at 10.0.30.10 (first static app-server slot per addressing convention). Nothing deployed yet.

**Source:** `Firewall/VLAN_30.md`.
**Resolve by:** confirm/adjust the IP once the stack is actually deployed.

## 3. Monitoring scrape ports/exporters unconfirmed for every target

`Firewall/VLAN_30.md` rules 7-15 use a placeholder port (TCP 9100, generic node_exporter) for scraping pfSense, Proxmox, TrueNAS, Pi-hole, and 5 DMZ hosts. Each target likely needs a different exporter, port, or scrape method.

**Source:** `Firewall/VLAN_30.md`, open items section.
**Resolve by:** verify per-target exporter/port before implementing rules 7-15.

## 4. Proxmox monitoring: push vs. pull undecided

Proxmox supports native push-based external metrics (statsd/InfluxDB) as an alternative to pull-based Prometheus scraping (`VLAN_30.md` rule 8).

**Source:** `Firewall/VLAN_30.md`, open items section.
**Resolve by:** decide push vs. pull before finalizing rule 8.

## 5. Loki log shipping not yet ruled

Loki (bundled on the future Grafana host, Apps VLAN) receives logs via a push model. Source hosts initiate outbound into Apps, opposite direction from the scrape rules. Requires new outbound-allow rules on `VLAN_10.md` and `VLAN_20.md` (already finalized) and eventual `VLAN_40.md` (not yet written), on Loki's default port 3100.

**Source:** `Firewall/VLAN_30.md`, "out of scope this pass" section.
**Resolve by:** once Grafana/Loki is actually deployed, amend `VLAN_10.md`, `VLAN_20.md`, and `VLAN_40.md` outbound sections to allow log shipping into Apps.

## 6. PiVPN outbound WAN not included

`Firewall/VLAN_40.md` doesn't grant PiVPN (10.0.40.13) outbound WAN access. The current box may still need it for package/firmware updates while the replacement project is pending.

**Source:** `Firewall/VLAN_40.md`, open items section.
**Resolve by:** reconsider once the Enterprise VPN / Remote Access Redesign project (`General.md` #2) clarifies the new architecture; add an outbound rule for the current PiVPN box in the meantime if updates are needed.

## 7. Old Jellyfin port assumed 8096 (HTTP), unverified

`Firewall/VLAN_40.md` rules 5 and 15 (Trusted Clients direct access, nginx forward) assume Jellyfin's default HTTP port 8096.

**Source:** `Firewall/VLAN_40.md`, open items section.
**Resolve by:** verify against actual Jellyfin config. Could be 8920 (HTTPS) instead or in addition.

## 8. Torrenting any/any is the largest open attack surface in this ruleset

`Firewall/VLAN_40.md` rules 4 and 13 allow unrestricted inbound/outbound any/any for the Torrenting host (10.0.40.14), required for unpredictable P2P tracker/peer ports.

**Source:** `Firewall/VLAN_40.md`, risk note.
**Resolve by:** revisit once the torrent client's actual configured port range is known. Most clients support pinning to a specific range instead of full any/any.

## 9. nginx ACME renewal method unconfirmed

`Firewall/VLAN_40.md` scopes nginx's WAN-inbound to port 443 only (rule 2). If cert renewal uses the HTTP-01 challenge, port 80 inbound would also be needed; DNS-01 or TLS-ALPN-01 would not.

**Source:** `Firewall/VLAN_40.md`, open items section.
**Resolve by:** confirm which ACME challenge method is (or will be) used before implementing.

## 10. Switch Jellyfin and Minecraft to NAT hairpinning instead of direct-access rules

For the strongest security posture, user wants local (Trusted Clients) access to DMZ-hosted Jellyfin and Minecraft to go through NAT reflection/hairpinning on pfSense, where local clients hit the same public port-forward as remote clients, rather than a dedicated direct-access ACL rule. This is why no Trusted Clients→Minecraft rule exists. It also flags the current `VLAN_40.md` rule 5 (Trusted Clients→Old Jellyfin direct, TCP 8096) as a candidate for replacement.

**Source:** `Firewall/VLAN_60.md`, open items section.
**Resolve by:** configure NAT reflection/hairpinning on pfSense for the Jellyfin and Minecraft port-forwards, then remove `VLAN_40.md` rule 5 (and the corresponding intra-DMZ nginx forward if it changes) in favor of the hairpinned path.

## 11. VLAN 70 not yet physically built

`Firewall/VLAN_70.md` is a rules design only. No switch port, pfSense interface, or DHCP scope exists yet for Guest/IoT.

**Source:** `Firewall/VLAN_70.md`, open items section.
**Resolve by:** build the VLAN (interface, DHCP scope) when the first guest/IoT device is acquired, then implement the documented rules.

## 12. VLAN 70 Pi-hole DNS exception breaks strict internet-only egress

`Firewall/VLAN_70.md` rule 2 allows VLAN 70 → Pi-hole (10.0.20.53) UDP/TCP 53, the only cross-VLAN exception to Guest/IoT's otherwise internet-only egress. Needed for ad-blocking/filtering on IoT devices.

**Source:** `Firewall/VLAN_70.md`, open items section.
**Resolve by:** add a matching inbound cross-ref rule to `VLAN_20.md` once VLAN 70 is actually built.

## 13. VLAN 70 client isolation enforcement mechanism unconfirmed

`Firewall/VLAN_70.md` rule 5 default-denies device-to-device traffic at the firewall layer, but a pfSense ACL alone won't stop same-subnet L2 traffic between devices on the same Wi-Fi AP.

**Source:** `Firewall/VLAN_70.md`, open items section.
**Resolve by:** confirm whether AP-level Wi-Fi client isolation is also needed, once actual AP/switch hardware for this VLAN is chosen.

## 14. Physical/hardware setup not documented

No existing doc covers the homelab's physical layer: rack layout, physical devices/models, cabling, power, physical port-to-VLAN mapping, etc. Everything documented so far is logical/network config only.

**Source:** User request, 2026-07-12.
**Resolve by:** write a physical/hardware documentation doc (e.g. new file under `General/`) covering rack layout, hardware inventory, cabling, and power, not tracked as a `General.md` project since it's documentation, not an engineering task.

## 15. Upgrade Proxmox VE 8 to VE 9

Proxmox cluster currently on VE 8; needs upgrade to VE 9 before support/compatibility cutoff.

**Source:** User request, 2026-07-12.
**Deadline:** 2026-08-08.
**Resolve by:** review Proxmox 8→9 upgrade guide (repo/package changes, Ceph/ZFS compat if applicable), snapshot/backup config first, then perform the upgrade before the deadline.

## 16. Fabric forwarding-secret mechanism unconfirmed for modded Minecraft server

Modded backend switched from Paper to Fabric. Paper has native Velocity modern-forwarding support via `paper-global.yml`; Fabric has no built-in equivalent; forwarding requires a compatible mod (e.g. FabricProxy-Lite) installed and configured with the same secret. Exact mod/config used isn't documented yet.

**Source:** `DMZ/Minecraft/Minecraft Server Home Lab Project Summary.md`, Server Software / Secret Handshake Integration.
**Resolve by:** confirm which mod (or other mechanism) is providing Velocity forwarding on the Fabric backend, then document it in `Minecraft Server Home Lab Project Summary.md` and `Guide.md`.

## 17. PiVPN internal-access scope note referenced but never written

`Firewall/VLAN 40.md` rule 3 says WireGuard's "internal access scope deferred (see PiVPN note)"; `Firewall/WAN.md` rule W3 points back to "PiVPN scope note in VLAN_40.md". The two files reference each other for a note that doesn't exist in either.

**Source:** `Firewall/VLAN 40.md`, rule 3 / `Firewall/WAN.md`, rule W3.
**Resolve by:** decide and document what internal VLANs (if any) a connected WireGuard client should reach, then write the actual scope note in `VLAN_40.md` (or wherever it belongs) and fix the cross-reference.
