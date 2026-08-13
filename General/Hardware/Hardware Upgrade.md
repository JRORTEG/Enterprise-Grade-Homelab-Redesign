Deciding the switch replacement and evaluating the ASA firewall for `Future-Planning/General.md` project #4.

## Context

The Netgear GS308E's management interface is HTTP-only, so switch admin credentials and session traffic cross the MGMT VLAN in cleartext (`Future-Planning/Open Items.md` #1). I'm replacing it against explicit criteria: encrypted management (HTTPS and/or SSH), per-port VLAN tagging, PoE for future AP uplinks, and SNMP/syslog export for the future monitoring/SIEM stack.

Candidate hardware I have on hand is listed in `Switch Replacement Options.md`: four Cisco Catalyst switches and a Cisco ASA 5525-X firewall.

## Switch decision: WS-C2960G-24TC-L

| Model | Ports | L2/L3 | Era | Power/noise |
|---|---|---|---|---|
| **WS-C2960G-24TC-L (chosen)** | 24x GigE + 4 SFP | L2 | EOS 2011 | Low, quiet |
| WS-C3650-48TS | 48x GigE + 4 SFP+ | L3-capable | 2013, longer support tail | Higher power, DC-grade fan noise |
| WS-C3750G-24TS-S1U | 24x GigE | L3-capable, stackable | ~2005, EOL | Moderate-high noise |
| WS-3750G-48TS-S | 48x GigE | L3-capable, stackable | ~2005, EOL | Moderate-high noise |

All four run Cisco IOS classic, so all of them clear the real bar here: SSH, SNMPv3, and remote syslog, which is a stronger fix for the cleartext-credential problem than the GS308E's HTTP-only web UI ever was. That puts the decision on fit, not on capability.

I'm going with the WS-C2960G-24TC-L. pfSense already does all L3 routing and inter-VLAN ACL enforcement (`General.md` project #1, completed). My current MGMT VLAN device count is 3 (pfSense, Proxmox, switch), so 24 ports is already generous headroom over the GS308E's 8; the 48-port options are sized for a rack I don't have and run louder, higher-power fans than I want in my space. The 3750G pair is the same era and EOL status as the 2960G with no advantage for my use case. I'm not stacking, and I don't need their L3 features either.

The 3650-48TS stays on my radar as a later upgrade if port count ever becomes the actual bottleneck, since it has the longest support tail of the four.

**Gap:** none of the four candidates have PoE. Future AP uplinks will need PoE injectors, or a dedicated PoE switch/module bought separately. Logged as an open item below.

## Firewall decision: staying on pfSense, not adding the ASA 5525-X

I'm not putting the ASA 5525-X into the live network, as a replacement for pfSense or alongside it.

- pfSense already carries the completed least-privilege ACL project, full 802.1Q VLAN trunk integration, NAT/WAN rules, and WireGuard. Replacing that is a lot of rework for no clear gain.
- The ASA 5525-X hit end-of-sale in 2018 and is at or past end-of-support, so it's not getting further Cisco security patches. That's a liability for a device that would sit near WAN or inter-VLAN traffic.
- Running it inline alongside pfSense adds a second firewall to configure, patch, and keep in sync, with no capability pfSense doesn't already cover for my setup.

The one use I'd consider is dropping it into the Lab VLAN (50), fully isolated from production traffic, purely to build hands-on Cisco ASA CLI experience. That's an optional resume-building exercise, not part of this hardware upgrade, so I'm logging it separately rather than folding it into this project.

## Implementation plan

1. Rack the WS-C2960G-24TC-L.
2. Initial IOS config: hostname, `enable secret` (hashed, not `enable password`), generate RSA keys and enable SSH, disable Telnet and the HTTP server (`no ip http server`), enable `ip http secure-server` only if I want the web UI as a secondary option.
3. Build the VLAN database to match `IP Assignment.md`: VLANs 10, 20, 30, 40, 50, 60, and 70 once it exists.
4. Configure the uplink to pfSense as an 802.1Q trunk carrying all VLANs, matching the existing trunk design.
5. Set access ports to their single assigned VLAN per device, mirroring current GS308E port assignments.
6. Configure an SNMPv3 user and point `logging host` at the future Grafana/monitoring target (`General.md` project #11).
7. Migrate devices from the GS308E to the new switch port-by-port to avoid a full outage, validating connectivity per VLAN as I go.
8. Decommission the GS308E once the new switch is validated.

## Open items logged

- `Open Items.md` #1 marked RESOLVED — switch replacement decided (WS-C2960G-24TC-L).
- New item: PoE gap for future AP uplinks.
- New item: ASA 5525-X isolated Lab-VLAN CLI learning exercise, optional and out of scope for this project.
