# Access Point Plan

Planning wireless hardware for the new apartment. No purchases made yet. I want the wireless side of the redesign to hit the same segmentation bar as the wired side, Trusted Clients, Guest, and IoT traffic each isolated on their own VLAN, while still letting some personality show in the apartment instead of everything looking like a server closet.

## Candidates

### Davolink Minions WiFi 6E Router ("Kevin")

AXE5400 tri-band (2.4/5/6GHz), 2.5GbE WAN, WPA3, up to 128 connected devices. I was leaning heavily toward this one. It's genuinely capable hardware, and it's a funny novelty piece to have sitting out in the open.

I checked whether it could carry my three SSIDs on separate tagged VLANs, and the answer is no, for two compounding reasons. The stock firmware has no native per-SSID VLAN tagging to begin with. Worse, it actively mishandles VLAN tags: per a report on the OpenWrt support-request thread, it "doesn't bother filtering out VLAN-tagged packets, it just throws them all on the wifi regardless of VLAN tag." That's not just a missing feature, it means if I ever put this router on a trunk port, tagged frames from other VLANs could leak onto whatever SSID a client is connected to.

There's no custom firmware path around this either. Both hardware variants (Bob: Realtek; Kevin: Qualcomm IPQ5018) only have an open, unworked OpenWrt support request, no actual port exists. Not something I can flash my way out of.

Conclusion: this router isn't going to be the device that does my segmentation, stock or modded.

### TP-Link Omada EAP723 ("Omada 7 BE5000")

WiFi 7 (BE5000, dual-band), 2.5GbE port, 802.3at PoE or DC power (adapter not included in box), ~$80 street price, rated 250+ concurrent clients / 1500 sq ft coverage. Ceiling-mount form factor, part of TP-Link's managed Omada SDN line.

This is built for exactly what I need: the Omada platform supports SSID-to-VLAN mapping natively (SSID VLAN and SSID VLAN Override), up to 16 SSIDs per AP, managed through a free self-hosted Omada Software Controller or the free Omada Essentials cloud controller. No paid license required to get VLAN tagging working.

## Decision

I'm running both, in different roles.

The EAP723 does the actual segmentation: one AP, three SSIDs, each tagged to its VLAN over a trunk port, Trusted Clients (60), Guest (70), IoT (80, new, see below). This is the enterprise half of the plan.

The Minions router becomes a single-VLAN novelty AP instead, not a segmentation device. It goes on a plain switch access port rather than a trunk, so its broken VLAN-tag handling never encounters a tagged frame in the first place; the switch strips VLAN awareness down to "everything on this port is VLAN 70" before traffic ever reaches the router. I get to keep it out, powered on, and doing real work, it just isn't the device carrying my trust boundaries.

I'm putting it on the Guest VLAN (70) specifically. That fits the novelty/visitor-facing angle: anyone connecting to the silly-named SSID is, by definition, a guest. Guest is already internet-only egress with no route to anything sensitive, so the Minions' weak VLAN handling has nothing to leak even in the worst case.

## New VLAN: IoT (80)

My existing `VLANs/VLAN_70.md` and `General/IP Assignment.md` modeled VLAN 70 as a combined "GUEST/IoT" segment. It was never built (no pfSense interface, no DHCP scope, `Future-Planning/Open Items.md` #11), so this is the right moment to split it into the three categories I actually want, before there's any live config to migrate.

| VLAN | Name | Subnet | Gateway | DHCP |
|---|---|---|---|---|
| 70 | GUEST | 10.0.70.0/24 | 10.0.70.1 | Dynamic `.50-.199` |
| 80 | IOT (new) | 10.0.80.0/24 | 10.0.80.1 | Dynamic `.50-.199` |

Same posture on both: default-deny, client isolation, internet-only egress, narrow Pi-hole DNS exception. Full rule sets live in `VLANs/VLAN_70.md` and `VLANs/VLAN_80.md`.

## Bill of materials

| Item | Role | Status |
|---|---|---|
| TP-Link EAP723 (Omada 7 BE5000) | Primary AP, Trusted/Guest/IoT segmentation | Not purchased |
| PoE injector | Powers the EAP723 (switch has no PoE) | Not purchased, preferred option |
| DC power adapter for EAP723 | Backup power option if I skip the injector | Not purchased, backup option |
| Davolink Minions router (Kevin) | Novelty single-SSID AP on Guest VLAN | Not purchased |

## Switch port plan

Cisco Catalyst 2960G (`WS-C2960G-24TC-L`, 10.0.10.12) has no PoE, so the EAP723 needs either the PoE injector or its own DC power. Either way it still needs a data run to a switch port.

- **EAP723 port:** trunk, `switchport trunk allowed vlan 60,70,80`, only the three VLANs it's actually serving, not the full VLAN set (unlike the Proxmox trunk on `g0/21`, which has to carry everything).
- **Minions port:** access, `switchport access vlan 70`.

Exact port numbers TBD at purchase/install time. Full config goes in `Hardware/Cisco Catalyst 2960G/Hardware Upgrade.md` alongside the existing port table.

## Omada controller

EAP723 needs an Omada controller for full functionality (SSID VLAN mapping, seamless roaming, etc.), and I haven't decided between the two free options yet. A self-hosted Omada Software Controller runs as a container/VM, fits my existing homelab-hosts-everything pattern, but is one more thing to deploy and keep patched. The Omada Essentials cloud controller is free and TP-Link-hosted, zero infra to maintain, but depends on TP-Link's cloud being up for management access (not for the AP's actual data plane, which keeps working locally either way).

Logged as an open item, deciding once I'm actually provisioning the AP.

## Open items surfaced by this plan

Logged in `Future-Planning/Open Items.md`:

- #26: EAP723 power delivery (PoE injector vs. DC adapter) not yet purchased.
- #27: fate of the existing "Trusted Access Point" on switch port `g0/5` once the EAP723 goes live.
- #28: Omada controller choice (self-hosted vs. Essentials cloud) not finalized.
- #29: VLAN 80 physical build-out (pfSense interface, DHCP scope, switch VLAN database entry) needed before either AP can actually pass IoT traffic.
