# Open Items

Running list of loose ends surfaced during other work in this repo, not full projects (those go in `General.md`), just things flagged for future review or decision. Resolved items stay in this list tagged `RESOLVED (date)` rather than being deleted, to keep a history. Numbering is never reused or reshuffled for this reason.

## 1. Switch admin UI is HTTP-only (no HTTPS/SSH) — RESOLVED (2026-07-16)

Netgear GS308E doesn't support HTTPS or SSH for management. Plaintext credentials on the wire, even though access is scoped to Trusted Clients → Mgmt only.

**Source:** `Firewall/VLAN_10.md`, open risk section.
**Resolve by:** deciding on switch replacement (managed switch with HTTPS support), or in the meantime scoping the admin rule to a single admin host instead of all of VLAN 60.
**Resolved:** replacement switch chosen — Cisco Catalyst WS-C2960G-24TC-L (SSH + SNMPv3 + syslog). See `General/Hardware/Hardware Upgrade.md` for full decision and implementation plan. Physical swap still pending.

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

## 15. Upgrade Proxmox VE 8 to VE 9 — RESOLVED (2026-08-13)

Proxmox cluster currently on VE 8; needs upgrade to VE 9 before support/compatibility cutoff.

**Source:** User request, 2026-07-12.
**Deadline:** 2026-08-08.
**Resolve by:** review Proxmox 8→9 upgrade guide (repo/package changes, Ceph/ZFS compat if applicable), snapshot/backup config first, then perform the upgrade before the deadline.
**Resolved:** upgraded to VE 9 successfully, backups (PBS) were in place first per plan. Walkthrough (repo/package fixes, systemd-boot removal, apt source migration) in `MGMT/Proxmox/Proxmox 8 to 9 Upgrade Checklist.md`.

## 16. Fabric forwarding-secret mechanism unconfirmed for modded Minecraft server

Modded backend switched from Paper to Fabric. Paper has native Velocity modern-forwarding support via `paper-global.yml`; Fabric has no built-in equivalent; forwarding requires a compatible mod (e.g. FabricProxy-Lite) installed and configured with the same secret. Exact mod/config used isn't documented yet.

**Source:** `DMZ/Minecraft/Minecraft Server Home Lab Project Summary.md`, Server Software / Secret Handshake Integration.
**Resolve by:** confirm which mod (or other mechanism) is providing Velocity forwarding on the Fabric backend, then document it in `Minecraft Server Home Lab Project Summary.md` and `Guide.md`.

## 17. PiVPN internal-access scope note referenced but never written

`Firewall/VLAN 40.md` rule 3 says WireGuard's "internal access scope deferred (see PiVPN note)"; `Firewall/WAN.md` rule W3 points back to "PiVPN scope note in VLAN_40.md". The two files reference each other for a note that doesn't exist in either.

**Source:** `Firewall/VLAN 40.md`, rule 3 / `Firewall/WAN.md`, rule W3.
**Resolve by:** decide and document what internal VLANs (if any) a connected WireGuard client should reach, then write the actual scope note in `VLAN_40.md` (or wherever it belongs) and fix the cross-reference.

## 18. Firewall rule needed for Proxmox → PBS (Main PC) backup traffic — RESOLVED (2026-08-13)

Dual-boot PBS on the Main PC (`MGMT/Proxmox/Backup Solution.md`) needs Proxmox (10.0.10.10, VLAN 10) to reach PBS on VLAN 60 to push backup jobs. No existing rule permits this direction or port — `VLAN_10.md`/`VLAN_60.md` only cover Trusted Clients → Proxmox (TCP 8006/22), not Proxmox → Trusted Clients.

**Source:** `MGMT/Proxmox/Backup Solution.md`, Firewall implications section.
**Resolve by:** after PBS is installed and its port confirmed (default TCP 8007), add an outbound rule on `VLAN_10.md` and matching inbound rule on `VLAN_60.md` for Proxmox → PBS/Main PC.
**Resolved:** rule implemented — `VLAN_10.md` rule 18 (outbound, TCP 8007) and matching `VLAN_60.md` rule 18 (inbound). PBS's actual destination IP turned out to be `10.0.60.15`, not `.10` as assumed here (see #24).

## 19. Dual-boot PBS backup solution is a budget-driven interim fix

Decision to dual-boot PBS on the Main PC with a dedicated 2TB drive, manually booted on a routine basis, was made due to limited budget rather than being the ideal end state (dedicated always-on backup host).

**Source:** User decision, 2026-07-14.
**Resolve by:** once budget allows, replace with a dedicated/always-on PBS solution (possibly NAS-backed, targeting TrueNAS per the original `General.md` #2 vision) to remove the manual-trigger and single-location risks documented in `MGMT/Proxmox/Backup Solution.md`.

## 20. None of the switch replacement candidates have PoE

The four Cisco Catalyst switches evaluated for the switch replacement (`General.md` #4) are all non-PoE (`TC`/`TS` suffixes). Future AP uplinks need PoE and this purchase doesn't provide it.

**Source:** `General/Hardware/Hardware Upgrade.md`.
**Resolve by:** budget for PoE injectors per AP, or a dedicated PoE switch/module, once AP hardware is actually selected.

## 21. ASA 5525-X isolated Lab-VLAN CLI learning exercise

Decided not to add the Cisco ASA 5525-X to the live network (EOL/unsupported, redundant with pfSense). The only remaining use considered was standing it up fully isolated on the Lab VLAN (50), disconnected from production traffic, purely for hands-on Cisco ASA CLI practice.

**Source:** `General/Hardware/Hardware Upgrade.md`.
**Resolve by:** optional, low priority — pick up only if/when I want dedicated ASA CLI resume practice; not required for any other project.

## 22. Excalidraw network diagram not migrated to Notion

The network topology diagram (`General/Diagrams/Excalidraw/Drawing 2026-07-12 00.28.52.excalidraw.md`) wasn't brought over during the Obsidian-to-Notion documentation migration. Notion has no native Excalidraw renderer, and the underlying file is raw Excalidraw JSON, not a format worth pasting as inert text.

**Source:** Obsidian-to-Notion migration, 2026-07-22.
**Resolve by:** decide on an export/embed approach (e.g. export the diagram to PNG/SVG and embed as an image, or rebuild it as a Notion-native diagram) then migrate it into the General/Diagrams page in Notion.

## 23. PBS bare-metal install deferred due to severe weather/power risk — RESOLVED (2026-08-13)

Decided to skip the Proxmox Backup Server OS install step on the Main PC's dedicated 2TB drive for now — severe local weather raised the risk of a power surge/outage corrupting the drive mid-install. Proceeded with the Enterprise VPN / Remote Access Redesign (`General.md` #3) instead.

**Source:** User decision, 2026-08-03.
**Resolve by:** install PBS on the Main PC once the weather/power risk has passed, per `MGMT/Proxmox/Backup Solution.md`.
**Resolved:** installed, datastore created and connected to the Proxmox VE cluster for scheduled backups. Walkthrough in `MGMT/Proxmox/Proxmox Backup Server Datastore Setup.md`.

## 24. PBS temporarily on Trusted Clients (10.0.60.15) instead of MGMT

PBS install landed on VLAN 60 (Trusted Clients), static `10.0.60.15`, not VLAN 10 (MGMT) where it belongs as infrastructure control-plane. Happened because the Main PC only has one NIC, shared between the daily-use OS and the PBS dual-boot — no way to put PBS on its own VLAN yet. Plan: add a second NIC to the Main PC so each OS gets its own interface, and move PBS's interface to VLAN 10.

**Source:** User decision, 2026-08-13.
**Resolve by:** add a second NIC to the Main PC, assign PBS's interface a static MGMT IP (next open static slot per addressing convention), then update `General/IP Assignment.md`, `VLANs/VLAN_10.md`, `VLANs/VLAN_60.md`, and `MGMT/Proxmox/Backup Solution.md` to match — including retiring the cross-VLAN Proxmox→PBS firewall rule (#18) once PBS and Proxmox share VLAN 10.

## 25. Switch running EOL 2010-era IOS image — needs firmware/security review

Cisco WS-C2960G-24TC-L runs `c2960-lanbasek9-mz.122-50.SE5` (compiled 2010, IOS 12.2(50)SE5). Getting SSH working required forcing my SSH client to accept legacy, deprecated crypto (diffie-hellman-group1-sha1 KEX, aes128-cbc cipher, hmac-sha1 MAC) since the switch doesn't support anything newer — a live weak-crypto exposure on MGMT-plane administration, and the switch itself has been end-of-support since 2011 (`General/Hardware/Hardware Upgrade.md`) so it isn't getting further Cisco security patches at the IOS level either.

**Source:** `Hardware/Switch Upgrade.md`, SSH troubleshooting steps, 2026-08-16.
**Resolve by:** check Cisco's archive for a later 12.2 SE-train image for this model that supports modern SSH algorithms and install it if one exists; if not, treat this as a known lab-scale limitation of the 2010-era hardware and factor into the next hardware refresh cycle instead of a firmware fix.

## 26. EAP723 power delivery not yet purchased

Cisco 2960G switch has no PoE, so the EAP723 access point needs either a PoE injector or its own DC power adapter (not included in box) to power on. Leaning toward the injector.

**Source:** `Hardware/Access Point/Plan.md`, 2026-08-18.
**Resolve by:** buy a PoE injector (or the DC adapter as backup) before installing the EAP723.

## 27. Fate of existing "Trusted Access Point" on g0/5 undecided

Once the EAP723 goes live with its own Trusted-Clients SSID, the current AP on switch port `g0/5` (VLAN 60, access) may become redundant — or I might keep it as a dedicated Trusted-only AP for coverage reasons. Not resolved.

**Source:** `Hardware/Access Point/Plan.md`, `Hardware/Cisco Catalyst 2960G/Hardware Upgrade.md`, 2026-08-18.
**Resolve by:** decide once the EAP723 is installed and I can compare coverage/performance against the existing AP.

## 28. Omada controller choice not finalized

EAP723 needs an Omada controller for full functionality (SSID VLAN mapping, roaming, etc.). Choosing between a free self-hosted Omada Software Controller (container/VM, fits the existing homelab-hosts-everything pattern but is one more thing to patch) and the free Omada Essentials cloud controller (zero infra, but management access depends on TP-Link's cloud being up).

**Source:** `Hardware/Access Point/Plan.md`, 2026-08-18.
**Resolve by:** decide once actually provisioning the EAP723.

## 29. VLAN 80 (IoT) not yet physically built

`VLANs/VLAN_80.md` is a rules design only, split off from the former combined GUEST/IoT VLAN 70. No pfSense interface, DHCP scope, or switch VLAN database entry exists yet — same status as VLAN 70 was before item #11.

**Source:** `VLANs/VLAN_80.md`, `Hardware/Access Point/Plan.md`, 2026-08-18.
**Resolve by:** build the VLAN (pfSense interface, DHCP scope, switch VLAN database entry) once the EAP723 is being provisioned, then implement the documented rules.
