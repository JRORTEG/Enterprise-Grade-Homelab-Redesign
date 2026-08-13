# Proxmox 8 to 9 Upgrade Checklist

Upgrading the Proxmox VE host from 8 to 9 before the support/compatibility cutoff, per `Future-Planning/Open Items.md` #15 (resolved once this upgrade completed). This was gated on having a working backup solution first (`Backup Solution.md`), since a major hypervisor version bump is exactly the kind of change that should have a rollback path.

## Pre-flight: `pve8to9` checker

Ran the built-in `pve8to9` compatibility checker before touching anything. Standalone node, no Ceph, so cluster/hyper-converged checks skipped clean. Out of 42 checks: 31 passed, 6 skipped, 3 warnings, 2 failures.

**Failures to fix:**

1. **Node IP mismatch:** `Resolved node IP '192.168.2.2' not configured or active for 'proxmox'`. Leftover stale mapping in `/etc/hosts` from before this box moved onto the current VLAN addressing. Fixed by editing `/etc/hosts` and pointing the `proxmox` hostname line at the host's actual current IP (`10.0.10.10`, VLAN 10/MGMT) instead of the old `192.168.2.2`.
2. **`systemd-boot` meta-package installed:** conflicts with Proxmox's own `proxmox-boot-tool` and breaks upgrades of other boot-related packages. Fixed with:
   ```
   apt remove systemd-boot
   ```
   This doesn't affect the ability to boot, it just removes the conflicting Debian wrapper package.

**Warnings cleared:**

- GRUB removable-bootloader warning (EFI bootloader present, GRUB not set to auto-update it):
  ```
  echo 'grub-efi-amd64 grub2/force_efi_extra_removable boolean true' | debconf-set-selections -v -u
  apt install --reinstall grub-efi-amd64
  ```
- Missing Intel microcode package:
  ```
  apt install intel-microcode
  ```
- 7 running guests flagged. Shut all of them down before the actual upgrade, since upgrading the hypervisor under running guests risks crashes or data corruption.

Re-ran `pve8to9` after these fixes: 0 failures, 0 warnings, cleared to proceed.

## Repository migration: bookworm → trixie

Before crossing over, brought Proxmox 8 fully current on its existing release branch (`apt update && apt dist-upgrade -y`), then repointed the APT sources from Debian 12 (bookworm) to Debian 13 (trixie) across `/etc/apt/sources.list` and the `pve-*.list` files under `/etc/apt/sources.list.d/`.

## `apt dist-upgrade` blocked by the `proxmox-ve` removal safety hook

First `apt dist-upgrade` attempt tripped the `pve-apt-hook` safety warning:

```
W: (pve-apt-hook) !! WARNING !!
W: (pve-apt-hook) You are attempting to remove the meta-package 'proxmox-ve'!
```

Didn't proceed. Forcing that through would have stripped the box of its web UI and virtualization stack. This hook fires when APT can't resolve dependencies against the new repos and falls back to solving it by uninstalling Proxmox entirely, which almost always means the Proxmox 9 repo itself isn't correctly in place yet. Traced it through three separate repo issues:

1. **Missing `non-free-firmware` component.** `/etc/apt/sources.list` had the trixie lines but was missing `non-free-firmware`, which Proxmox 9's kernel packages depend on. Rewrote it to include `main contrib non-free non-free-firmware` on all three lines (base, updates, security), and disabled the enterprise repo (`pve-enterprise.list`) since there's no paid subscription on this box.
2. **DNS resolution failure for Proxmox's own domains.** `apt update` could reach the Debian mirrors by IP but failed with `Temporary failure resolving 'download.proxmox.com'` and the same for `enterprise.proxmox.com`. Fixed by pointing `/etc/resolv.conf` at public resolvers (1.1.1.1, 8.8.8.8).
3. **Stale `ceph-quincy` repo returning 404.** Once DNS was fixed, `apt update` hit `Err: ... ceph-quincy trixie Release 404 Not Found`. Ceph Quincy doesn't exist for trixie/Proxmox VE 9; Proxmox 9 moved to Reef/Squid. No Ceph in use on this standalone node (confirmed by the earlier `pve8to9` skip), so the fix was disabling the `ceph.list` repo file rather than repointing it to a newer Ceph release.

With all three fixed, `apt update` ran clean: no `Err:` lines, no 404s, no enterprise-repo resolution failures.

## Upgrade and verification

Confirmed `proxmox-ve`, `pve-manager`, and `pve-cluster` were not in the packages-to-remove list on the next `apt full-upgrade` attempt (the actual signal that the repo issues above were resolved), then let it run: 521 packages upgraded, 169 newly installed, 73 removed. Rebooted into the new kernel afterward and confirmed the node reports as Proxmox VE 9 (`pveversion`).

Upgrade completed successfully with backups already in place, closing out `Open Items.md` #15.
