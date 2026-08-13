## Backup Solution

### Summary

Decision: dual-boot Proxmox Backup Server (PBS) bare-metal on the Main PC, with a dedicated 2TB drive as the PBS datastore. Boot into PBS on a routine basis to let scheduled backup jobs run, then boot back to normal OS.

This is an **interim, budget-driven solution**, not the end state. No backup mechanism existed for any VM/LXC or Proxmox host config before. This was the lab's single largest unrecoverable-data-loss risk. PBS closes that gap now at near-zero cost using hardware already on hand. It will be revised later.

### Why this approach

- No backup solution was a previously-flagged gap. See `Future-Planning/General.md` #2 and `Future-Planning/Open Items.md` #15 (Proxmox VE 8→9 upgrade was blocked on having backups in place first; completed 2026-08-13 once PBS was live, per `Proxmox 8 to 9 Upgrade Checklist.md`).
- A dedicated always-on PBS host (or NAS-backed PBS targeting TrueNAS, the original vision in `General.md` #2) isn't affordable right now.
- Reusing the existing Main PC plus a dedicated 2TB drive gets real, working backup coverage today instead of leaving the gap open indefinitely.

### Architecture

- Main PC dual-boots: normal OS (daily use) and PBS (bare-metal install), with a 2TB drive dedicated exclusively as the PBS datastore. Install/config walkthrough in `Proxmox Backup Server Datastore Setup.md`.
- Network: PBS did **not** end up sharing the Main PC's existing DHCP reservation (`10.0.60.10`) as originally planned. The MAC-based reservation didn't carry the IP over cleanly between the two boots. PBS is instead statically assigned `10.0.60.15` on VLAN 60 (Trusted Clients).
- That's the wrong VLAN long-term: PBS is infrastructure control-plane and belongs on MGMT (VLAN 10), not Trusted Clients. It's parked on VLAN 60 for now because the Main PC only has one NIC. Plan: add a second NIC to the Main PC so each OS gets its own interface/VLAN, PBS's NIC moves to VLAN 10 and the daily-use OS keeps VLAN 60. See `Future-Planning/Open Items.md` #24.
- Proxmox host (`10.0.10.10`, VLAN 10/MGMT) pushes backup jobs to PBS (`10.0.60.15`) over the network whenever PBS is booted and reachable.

### Firewall implications

Proxmox (VLAN 10) → PBS (VLAN 60, `10.0.60.15`) on TCP 8007 is implemented: outbound rule 18 on `VLAN_10.md`, matching inbound rule 18 on `VLAN_60.md`. Existing rules already covered the opposite direction (Trusted Clients → Proxmox on TCP 8006/22, `VLAN_10.md` rules 3–4 / `VLAN_60.md` rules 10–11); this closed the MGMT-initiated direction into Trusted Clients that was previously missing.

Will need re-pointing once PBS moves to VLAN 10 (see NIC plan above): a same-VLAN destination for Proxmox → PBS wouldn't need this cross-VLAN rule at all.

### Operating procedure

1. Boot Main PC into PBS.
2. Let Proxmox's scheduled backup job run to completion.
3. Verify job status in the PBS/Proxmox UI.
4. Reboot back to normal OS.

RPO gap: backups only happen when PBS happens to be booted, not continuously. That's the core tradeoff of the interim approach: skip a routine boot, skip a backup window.

### Retention policy

Sized around the weekly boot cadence rather than a daily one, since PBS only sees one snapshot per week: keep-last 4, keep-weekly 4, keep-monthly 6, keep-yearly 1. That covers roughly a month of weekly rollback points, six months of monthly checkpoints, and one yearly archive, without letting the 2TB datastore fill up on chunks nothing points to anymore. Full reasoning and the prune/GC setup steps are in `Proxmox Backup Server Datastore Setup.md`.

### Restore procedure

Not yet written. Once the datastore is live, perform and document a full tested restore here. Required before this counts as a real backup solution, not just backup jobs running.

### Known limitations / risks

- Manual, human-dependent trigger. No backup occurs if the boot-into-PBS routine is skipped.
- No offsite/3-2-1 copy. Backups live in the same physical location as the source.
- Same electrical/physical-failure domain as the Main PC itself (same box, same room, same power circuit).
- Not a substitute for a dedicated backup host long-term.

### Revisit

Budget-driven stopgap. Revisit once funding allows for a dedicated, always-on PBS host, potentially NAS-backed (targeting TrueNAS), matching the original vision in `Future-Planning/General.md` #2, to close the manual-trigger and single-location gaps above.
