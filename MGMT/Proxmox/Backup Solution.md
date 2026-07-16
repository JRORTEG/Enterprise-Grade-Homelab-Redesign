
## Backup Solution

### Summary

Decision: dual-boot Proxmox Backup Server (PBS) bare-metal on the Gaming PC, with a dedicated 2TB drive as the PBS datastore. Boot into PBS on a routine basis to let scheduled backup jobs run, then boot back to normal OS.

This is an **interim, budget-driven solution**, not the end state. No backup mechanism existed for any VM/LXC or Proxmox host config before. This was the lab's single largest unrecoverable-data-loss risk. PBS closes that gap now at near-zero cost using hardware already on hand. It will be revised later.

### Why this approach

- No backup solution was a previously-flagged gap. See `Future-Planning/General.md` #2 and `Future-Planning/Open Items.md` #15 (Proxmox VE 8→9 upgrade is blocked on having backups in place first).
- A dedicated always-on PBS host (or NAS-backed PBS targeting TrueNAS, the original vision in `General.md` #2) isn't affordable right now.
- Reusing the existing Gaming PC plus a dedicated 2TB drive gets real, working backup coverage today instead of leaving the gap open indefinitely.

### Architecture

- Gaming PC dual-boots: normal OS (daily use) and PBS (bare-metal install), with a 2TB drive dedicated exclusively as the PBS datastore.
- Network: Gaming PC keeps its existing DHCP reservation, `10.0.60.10` on VLAN 60 (Trusted Clients). The reservation is MAC-based, so the same IP applies whether it's booted into the normal OS or PBS. No new IP needed.
- Proxmox host (`10.0.10.10`, VLAN 10/MGMT) pushes backup jobs to PBS over the network whenever PBS is booted and reachable.

### Firewall implications (gap to close)

No current ACL permits Proxmox (VLAN 10) to reach PBS on the Gaming PC (VLAN 60). Existing rules only cover the opposite direction: Trusted Clients → Proxmox on TCP 8006 (web UI) and TCP 22 (SSH), per `Firewall/VLAN_10.md` rules 3–4 and `Firewall/VLAN_60.md` rules 10–11. PBS backup traffic needs a new rule instead: Proxmox (VLAN 10) → PBS/Gaming PC (VLAN 60) on PBS's default port, TCP 8007. That's MGMT-initiated traffic into Trusted Clients, a direction the ruleset doesn't model yet.

Not implemented yet, needs the actual PBS install done first to confirm the port, then a new outbound rule on `VLAN_10.md` and matching inbound on `VLAN_60.md`. Logged as an open item (see below).

### Operating procedure

1. Boot Gaming PC into PBS.
2. Let Proxmox's scheduled backup job run to completion.
3. Verify job status in the PBS/Proxmox UI.
4. Reboot back to normal OS.

RPO gap: backups only happen when PBS happens to be booted, not continuously. That's the core tradeoff of the interim approach: skip a routine boot, skip a backup window.

### Retention policy

Not yet defined. Size retention (number of restore points kept) against the 2TB datastore once actual total VM/LXC storage footprint is known from an inventory pass.

### Restore procedure

Not yet written. Once the datastore is live, perform and document a full tested restore here. Required before this counts as a real backup solution, not just backup jobs running.

### Known limitations / risks

- Manual, human-dependent trigger. No backup occurs if the boot-into-PBS routine is skipped.
- No offsite/3-2-1 copy. Backups live in the same physical location as the source.
- Same electrical/physical-failure domain as the Gaming PC itself (same box, same room, same power circuit).
- Not a substitute for a dedicated backup host long-term.

### Revisit

Budget-driven stopgap. Revisit once funding allows for a dedicated, always-on PBS host, potentially NAS-backed (targeting TrueNAS), matching the original vision in `Future-Planning/General.md` #2, to close the manual-trigger and single-location gaps above.
