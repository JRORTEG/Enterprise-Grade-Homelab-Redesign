# Proxmox Backup Server Datastore Setup

First-time PBS setup on the Main PC, connecting it to Proxmox VE as a backup target. See `Backup Solution.md` for the overall architecture and decision behind running PBS this way; this doc is the walkthrough of getting the datastore actually stood up and taking backups.

## Datastore on the existing filesystem, not a blank disk

I installed PBS directly onto the dedicated 2TB drive, so by the time I got to **Storage / Disks → Directory** to create a datastore, that drive already had partitions and an active filesystem on it (boot, root, swap). The `Create: Directory` button there is built to format a completely blank, unpartitioned disk, so it correctly showed "No unused disks" since I don't have a second drive to give it. Single-drive setup, so the datastore has to live on the same filesystem as the OS.

Fix: create a folder on the existing filesystem and register that as the datastore instead of formatting anything new.

1. In the PBS shell (**Administration → Shell**), create the backup directory and hand it to the `backup` system user:
   ```
   mkdir /backups
   chown backup:backup /backups
   ```
2. In the web UI, **Add Datastore**, name it (e.g. `main-backups`), and set **Backing Path** to `/backups`.

Once added, the datastore shows up under the Datastore section in the left nav, ready to receive data.

## Connecting Proxmox VE to PBS

My PBS target is a separate box from the PVE cluster it's backing up, so PVE needs to be told where PBS is and given a job to run against it.

1. Grab the PBS API fingerprint: PBS dashboard → **Show API Fingerprint**, copy the hex string.
2. In PVE, **Datacenter → Storage → Add → Proxmox Backup Server**, and fill in:
   - **ID:** a friendly name (e.g. `PBS-Backup-Server`)
   - **Server:** PBS's IP
   - **User name:** `root@pam` (or a dedicated API user)
   - **Datastore:** the datastore name from the step above
   - **Fingerprint:** pasted from step 1
3. In PVE, **Datacenter → Backup → Add**, select the PBS storage target, choose **Include selected VMs/CTs** and pick the critical workloads, set the schedule, and use **Snapshot** mode so backups don't require stopping running guests.

Operational constraint that falls straight out of the dual-boot setup: if PBS isn't booted when the scheduled job runs, the job fails on a connection error. I'm running this on a weekly boot cadence, so the schedule has to line up with when I'm actually going to have the Main PC booted into PBS, or I trigger the job manually (**Datacenter → Backup → Run now**) once I'm booted in.

## Retention policy: keep-last 4, keep-weekly 4, keep-monthly 6, keep-yearly 1

PBS prunes on a time-bucket system, not a simple countdown, so retention has to be sized against how often I actually generate snapshots. Since I'm only booting into PBS about once a week, I only get one snapshot per week. Keep-daily or keep-hourly settings wouldn't do anything for me here, they'd just evaluate to that same single weekly snapshot.

I set:

| Parameter | Value | Coverage |
|---|---|---|
| keep-last | 4 | last 4 runs regardless of schedule drift |
| keep-weekly | 4 | ~1 month of weekly points |
| keep-monthly | 6 | 6 months of monthly checkpoints |
| keep-yearly | 1 | 1 annual archive |

That balances having enough rollback history against not letting the 2TB drive fill up.

Pruning alone doesn't free disk space, since PBS deduplicates into shared chunks. It takes two steps to actually reclaim it:
1. **Prune job** removes snapshot metadata (manifests/indices) per the retention rules above.
2. **Garbage collection** scans for chunks nothing references anymore and deletes them from disk.

PBS enforces a 24-hour-and-5-minute safety window on chunks, so anything touched within the last day survives a GC pass even if the snapshot that used it was just pruned. Given my weekly cadence, that means space from a snapshot pruned right after a backup run won't actually free up until the *next* boot into PBS a week later; a GC run in the same session as the prune won't reclaim everything.

Configured under **Datastore → Prune & GC**: set the keep-* values under Prune Jobs, and run Garbage Collection manually from the same tab whenever PBS is up after a backup run.
