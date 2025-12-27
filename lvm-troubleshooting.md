# 🧯 LVM Troubleshooting Labs (Advanced)

These labs focus on diagnosing and repairing real-world LVM problems you’ll encounter in production-like environments: full LVs, missing PVs, corrupted metadata, snapshots, migrations, and boot issues caused by LVM/fstab. Practice in disposable VMs only — do NOT run these on production servers.

This README is organized as scenario → symptoms → step-by-step commands → verification → explanation → troubleshooting notes → safe exercises. Use snapshots/checkpoints before destructive steps.

---

## Lab Index

1. LV is full — extend filesystem
2. VG has no free space — extend VG
3. Disk removed / PV missing — recovery options
4. Corrupted LVM metadata — restore from backups
5. Convert non-LVM disk to LVM safely (data migration)
6. Reduce LVM size (safe method)
7. Snapshot create, restore, and snapshot full troubleshooting
8. Boot failure due to wrong fstab / LVM
9. LVM on top of software RAID
10. Add new disk and migrate LV (pvmove) without downtime

---

## Prerequisites & Safety

- Always work on lab VMs with snapshots.
- Backup data before any metadata or volume operations.
- Install: lvm2 tools (`yum install lvm2`), mdadm for RAID, and audit logs enabled if possible.
- Commands shown assume systemd and RHEL-family tools; adapt for Debian (pvcreate/lvcreate are the same).
- Use `--help` or `man` pages for flags if unsure.

---

# 1 — LV is full (extend filesystem)

Symptoms
- Application logs: `No space left on device`
- `df -h` shows 100% for mountpoint
- Writes fail even though VG may have free space

Steps

1. Check filesystem usage:
   ```bash
   df -h
   ```

2. Identify LV backing the mountpoint:
   ```bash
   lsblk -f
   # or
   findmnt -no SOURCE,TARGET /path/to/mount
   ```

   Example output showing LV:
   ```
   /dev/mapper/vgdata-lvapps  /apps
   ```

3. Check VG free space:
   ```bash
   vgs
   # or detailed:
   vgdisplay vgdata
   ```

Case A — VG has free space (fast path):

4A. Extend LV and filesystem in one go (XFS/ext4 depending):
   - For XFS (online grow):
     ```bash
     sudo lvextend -L +2G /dev/vgdata/lvapps
     sudo xfs_growfs /apps
     ```
   - Combined option (automatic grow if lvextend supports `-r`):
     ```bash
     sudo lvextend -r -L +2G /dev/vgdata/lvapps
     ```
   - For ext4:
     ```bash
     sudo lvextend -L +2G /dev/vgdata/lvapps
     sudo resize2fs /dev/vgdata/lvapps
     ```

Verification:
```bash
df -h /apps
lvs -o+devices /dev/vgdata/lvapps
```

Case B — VG has no free space → go to Lab 2.

Explanation
- `lvextend` increases logical volume size. XFS can be grown online with `xfs_growfs`. Most recent LVM allows `-r` to run filesystem grow automatically.

Troubleshooting
- If `xfs_growfs` says not an XFS filesystem, check `blkid` or `file -sL /dev/vgdata/lvapps`.
- If `lvextend` fails: ensure VG has free extents (`vgdisplay` shows `Free PE / Size`).

Exercise
- Create a 1G LV, fill it with a file (`fallocate -l 900M file`), then extend it by 1G and verify writes succeed.

---

# 2 — VG has no free space (extend VG)

Symptoms
- `vgs` shows `Free PE` = 0
- `lvextend` fails with "insufficient free space"

Goal
- Add physical storage and extend VG, then extend LV & filesystem.

Steps

1. Add virtual disk to VM (e.g., /dev/sdb). Detect it:
   ```bash
   lsblk
   sudo parted -l
   ```

2. Create PV on new disk (or partition):
   ```bash
   sudo pvcreate /dev/sdb
   ```

3. Extend VG:
   ```bash
   sudo vgextend vgdata /dev/sdb
   ```

4. Extend LV and filesystem:
   ```bash
   sudo lvextend -L +5G -r /dev/vgdata/lvapps
   # or explicit:
   sudo lvextend -L +5G /dev/vgdata/lvapps
   sudo xfs_growfs /apps
   ```

Verification:
```bash
pvs
vgs
lvs
df -h /apps
```

Explanation
- Adding a PV increases the pool of extents; `vgextend` attaches the PV to the VG.

Troubleshooting
- If PV creation errors due to existing signatures, wipe them:
  ```bash
  sudo wipefs -a /dev/sdb
  sudo pvcreate /dev/sdb
  ```
- If `vgextend` refuses, check `pvdisplay /dev/sdb` and `vgdisplay vgdata`.

Exercise
- Add two small disks, create PVs, extend VG across both, then strip an LV across both and confirm distribution.

---

# 3 — Disk removed / PV missing (recovery)

Symptoms
- `vgdisplay` or `vgs -o +devices` shows PV missing or inactive devices
- LVs partially missing, IO errors on mount

Investigation

1. Show detailed VG/PV state:
   ```bash
   vgs -o +devices
   pvs -o+pv_used
   lvdisplay
   ```

2. Try to reactivate VG:
   ```bash
   sudo vgchange -ay vgdata
   ```

Scenarios

A — Disk temporarily unplugged (hotplug):

- If disk re-attached, trigger scan:
  ```bash
  echo "- - -" | sudo tee /sys/class/scsi_host/host*/scan
  sudo partprobe
  sudo vgchange -ay
  ```

B — Disk permanently gone and LV not mirrored (data loss risk):

- Remove missing PVs from VG to allow activation of remaining LVs (data may be lost if LV was striped):
  ```bash
  sudo vgreduce --removemissing vgdata
  ```
  - This attempts to remove missing PVs and repair VG metadata. Use only after careful assessment.

C — Recreate missing PV metadata if possible:
- If disk replaced with identical device and LVM metadata is intact, use `pvcreate --uuid` combined with `vgcfgrestore`—this is advanced and risky; prefer vgcfgrestore (next lab).

Precautions
- If LV spanned the missing PV and there was no redundancy (RAID or mirror), expect partial or total data loss.
- If mirrored (LV mirrored or RAID), use `pvmove` and replace failed disk.

Verification:
```bash
dmesg | tail -n 50
sudo vgdisplay vgdata
sudo lvdisplay
```

Exercise
- Simulate unplug: create two PVs and a striped LV, remove one PV (simulate by zeroing device), and attempt recovery to observe data loss behaviour (do this only in lab).

---

# 4 — Corrupted LVM metadata recovery

Symptoms
- Errors like `metadata read failed` or `unable to read metadata header: incorrect metadata checksum`
- VG won't activate; LVs inaccessible

Where LVM keeps backups
- `/etc/lvm/backup/`
- `/etc/lvm/archive/`

Steps

1. Inspect backups and archives:
   ```bash
   ls -l /etc/lvm/backup
   ls -l /etc/lvm/archive
   sudo less /etc/lvm/archive/vgdata_*.vgs
   ```

2. Try activating VG read-only:
   ```bash
   sudo vgchange -ay --partial vgdata
   ```

3. Restore metadata from an archive (choose appropriate timestamp):
   ```bash
   sudo vgcfgrestore -f /etc/lvm/archive/vgdata_00000xx.vg vgdata
   ```

4. Reactivate:
   ```bash
   sudo vgchange -ay vgdata
   ```

Explanation
- LVM stores periodic metadata snapshots; `vgcfgrestore` writes a selected metadata file back to the VG's metadata area on PVs.

Troubleshooting
- If `vgcfgrestore` says PV not present, ensure devices are re-scanned and accessible.
- If metadata backups themselves are corrupted, you may need to craft a manual restore by editing archive files—dangerous and advanced.

Safety tips
- Make a copy of current `/etc/lvm/backup` and `/etc/lvm/archive` to a safe location before writing changes.
- Work read-only initially; copy metadata and inspect carefully.

Exercise
- Intentionally corrupt a copy of an LVM archive, then practice restoring using an older archive copy to reactivate VG (lab only).

---

# 5 — Convert non-LVM disk to LVM (safe migration)

Scenario
- You have data on `/dev/sdb1` (non-LVM) and want to convert to LVM without losing data.

Safe pattern (backup → wipe → create LVM → restore):

Steps

1. Backup data to alternative storage (NFS, another disk):
   ```bash
   sudo tar -cvzf /tmp/data-backup.tgz /data
   # OR rsync: sudo rsync -aHAX /data /backup_location/
   ```

2. Unmount and wipe or remove partition:
   ```bash
   sudo umount /data
   sudo wipefs -a /dev/sdb1   # wipes filesystem signatures
   sudo parted /dev/sdb mkpart primary 1MiB 100%
   sudo partprobe
   ```

3. Create LVM on device:
   ```bash
   sudo pvcreate /dev/sdb1
   sudo vgcreate vgdata /dev/sdb1
   sudo lvcreate -n lvdata -l 100%FREE vgdata
   sudo mkfs.xfs /dev/vgdata/lvdata    # choose filesystem
   sudo mkdir -p /data
   sudo mount /dev/vgdata/lvdata /data
   ```

4. Restore data:
   ```bash
   sudo tar -xvzf /tmp/data-backup.tgz -C /data
   sudo chown -R original_user:original_group /data
   ```

Verification:
```bash
df -h /data
lvs
```

Notes
- Always verify backup integrity (`tar -tzf` or `rsync --dry-run`) before destructive steps.
- For large filesystems, consider `pvmove` (if adding new PV) to migrate data live.

Exercise
- Practice by converting a small test partition (100MB) into LVM and restoring data.

---

# 6 — Reduce LVM size (SAFE method)

Important: Shrinking XFS is NOT supported. Shrinking ext4 online is risky. Best practice is backup → recreate → restore.

Safe procedure

1. Backup filesystem contents:
   ```bash
   sudo rsync -aHAX /mnt/target /tmp/target_backup
   ```

2. Unmount:
   ```bash
   sudo umount /mnt/target
   ```

3. Remove LV and recreate at smaller size (or create new LV and migrate):
   ```bash
   sudo lvremove /dev/vgdata/lvdata
   sudo lvcreate -L 10G -n lvdata vgdata
   sudo mkfs.ext4 /dev/vgdata/lvdata   # ext filesystem if you plan future shrink
   sudo mount /dev/vgdata/lvdata /mnt/target
   sudo rsync -aHAX /tmp/target_backup/ /mnt/target/
   ```

Alternative (safer): create a new smaller LV, copy data, switch mountpoints.

Caution
- Always confirm backup contents before removing old LV.

Exercise
- Create an ext4 LV, fill it, backup, recreate smaller LV, restore, and confirm permissions/ownership preserved.

---

# 7 — LVM Snapshots: create, restore, and troubleshoot full snapshots

Create snapshot (CoW snapshot):
```bash
sudo lvcreate -L 1G -s -n snap_apps /dev/vgdata/lvapps
```

Mount snapshot:
```bash
sudo mkdir -p /mnt/snap_apps
sudo mount /dev/vgdata/snap_apps /mnt/snap_apps
```

Common problem — Snapshot full

- Snapshots are Copy-on-Write; if original LV changes more than snapshot size, snapshot becomes full and is marked `COW` 100%.

Check snapshots:
```bash
sudo lvs -a -o+devices
# or
sudo lvs -o lv_name,lv_size,data_percent,metadata_percent
```

If snapshot full:

1. If not needed, remove:
   ```bash
   sudo umount /mnt/snap_apps
   sudo lvremove /dev/vgdata/snap_apps
   ```

2. If you need snapshot content:
   - Try to increase snapshot LV space (if free extents exist in VG):
     ```bash
     sudo lvextend -L +500M /dev/vgdata/snap_apps
     ```
   - Then copy needed files off snapshot immediately.

Best practices
- Keep snapshot size conservative but large enough for expected write activity.
- Use thin snapshots (thin provisioning) for better efficiency.

Exercise
- Create heavy write activity on original LV, watch snapshot growth and behavior. Practice removing and restoring snapshots.

---

# 8 — Boot failure due to wrong fstab / LVM path error

Symptoms
- Boot drops to emergency or initramfs shell
- Errors like:
  ```
  cannot mount /dev/mapper/vgdata-lvapps
  dependency failed for mount
  ```

Steps to recover (rescue mode or live ISO):

1. Boot into rescue/emergency or use live media.
2. Mount root filesystem or use chroot to edit `/etc/fstab`.
3. Inspect fstab for bad device paths:
   ```bash
   cat /etc/fstab
   ```

Common causes
- Devices moved/renamed; using `/dev/mapper/...` raw device name changed.
- LV names changed or VG not activated early in boot.

Safer fstab entry (use UUID and `nofail`):
1. Get UUIDs:
   ```bash
   sudo blkid
   ```
2. Replace device with `UUID=xxxx`:
   ```
   UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /apps xfs defaults,nofail 0 0
   ```
3. Test:
   ```bash
   sudo mount -a
   ```
4. If issue is the LVM not activated early, ensure `lvm2` is available in initramfs:
   ```bash
   sudo dracut -f    # RHEL-based to regenerate initramfs
   ```

Verification
- Reboot and confirm normal boot or run `systemctl --failed`.

Troubleshooting
- If root VG is on LVM and not activating, ensure `rd.lvm.vg` kernel parameters are correct (rare on modern distros).
- If using systemd, check `systemd-cryptsetup` or LVM activation services.

Exercise
- Create a PV/LV/mount entry, then simulate a failure by changing the LV name in fstab and test recovery steps.

---

# 9 — LVM over software RAID (mdadm)

Use case
- Provide redundancy with mdadm RAID, then place LVM on top for flexibility.

Steps

1. Create RAID1:
   ```bash
   sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc --metadata=1.2
   sudo mdadm --detail /dev/md0
   ```

2. Persist array in config:
   ```bash
   sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf
   sudo dracut -f
   ```

3. Create PV / VG / LV on RAID:
   ```bash
   sudo pvcreate /dev/md0
   sudo vgcreate vgraid /dev/md0
   sudo lvcreate -n lvraid -l 100%FREE vgraid
   sudo mkfs.xfs /dev/vgraid/lvraid
   sudo mount /dev/vgraid/lvraid /data
   ```

4. Test failure tolerance:
   - Pull one disk (simulate failure), check mdadm:
     ```bash
     sudo mdadm --detail /dev/md0
     sudo mdadm --fail /dev/md0 /dev/sdb
     sudo mdadm --remove /dev/md0 /dev/sdb
     ```
   - Replace disk, and re-add:
     ```bash
     sudo mdadm --add /dev/md0 /dev/sdb
     ```

Notes
- LVM over RAID ensures LV data remains available if RAID level supports redundancy.
- You can also create PVs on each disk and create mirrored LVs with LVM mirror — more complex.

Exercise
- Build mdadm array, lvm on top, simulate a disk failure and rebuild it.

---

# 10 — Add new disk and migrate LV (pvmove) without downtime

Goal
- Add a new physical disk, move data off old PV to new PV online, then remove old disk.

Steps

1. Add new disk `/dev/sdc`. Create PV:
   ```bash
   sudo pvcreate /dev/sdc
   ```

2. Add to VG:
   ```bash
   sudo vgextend vgdata /dev/sdc
   ```

3. Move data from old PV (/dev/sdb) to new PV:
   ```bash
   sudo pvmove /dev/sdb /dev/sdc
   # or pvmove /dev/sdb1 if partition used
   ```

4. Remove old PV from VG:
   ```bash
   sudo vgreduce vgdata /dev/sdb
   ```

5. Optionally wipe old disk and repurpose:
   ```bash
   sudo pvremove /dev/sdb
   sudo wipefs -a /dev/sdb
   ```

Verification:
```bash
pvs -o+devices
vgs
lvs
```

Notes
- `pvmove` performs live migration of extents; filesystem remains online during operation.
- For very large PVs, monitor progress. `pvmove` can take long time.

Troubleshooting
- If `pvmove` fails halfway, use `pvmove -n` dry-run or check `/var/log/messages` or `journalctl` for I/O errors.
- If new disk too small, pvmove will fail—ensure new PV has enough extents.

Exercise
- Simulate migration by creating small test VG and LV, move PV to another disk and remove old one.

---

## Quick Commands Reference (LVM-focused)

- Inspect: `pvs`, `vgs`, `lvs`, `pvdisplay`, `vgdisplay`, `lvdisplay`
- Scan devices: `lsblk`, `blkid`, `partprobe`, `echo "- - -" | tee /sys/class/scsi_host/host*/scan`
- LVM ops: `pvcreate`, `vgcreate`, `lvcreate`, `lvremove`, `vgextend`, `vgreduce`, `pvmove`, `vgcfgrestore`, `vgchange -ay`
- Filesystem: `mkfs.xfs`, `resize2fs`, `xfs_growfs`
- RAID: `mdadm --create`, `mdadm --detail`, `mdadm --fail`, `mdadm --add`

---

## Troubleshooting Checklist & Best Practices

- Snapshot VM before any metadata operations.
- Always backup important data (rsync/tar) before `vgreduce --removemissing`, `lvremove`, or `vgcfgrestore`.
- Inspect `/etc/lvm/archive` and `/etc/lvm/backup` before restoring.
- Use `--test` or dry-run options where available; read manpages.
- Use UUIDs in `/etc/fstab` to avoid device name issues.
- Monitor disk health (SMART) to preempt PV failures.
- Prefer mirrored or RAID-backed PVs for critical LVs.

---

## Suggested Hands-on Exercises

1. Create VG, LV, fill LV to 90%, extend LV via extra PV and verify writes.
2. Simulate a missing PV: create VG across two disks, remove one disk file-backed device and try `vgchange` and `vgreduce --removemissing`.
3. Corrupt an LVM archive copy (copy then modify) and practice `vgcfgrestore` with the correct archive.
4. Create a snapshot, generate heavy writes, observe snapshot COW growth, and handle snapshot-full event.
5. Build mdadm RAID1, put LVM on top, then simulate rebuild and observe LVM remains active.

---

## Notes on "Design & Effective Font"

- Markdown rendering (GitHub) controls fonts; use headings, consistent code fences, callouts, and tables to improve readability.
- Use monospace for commands and inline code. Use bold for cautions and italic for notes.
- For wider documentation needs, consider MkDocs or GitHub Pages to style fonts and layout.

---
- Add diagrams (ASCII or images) showing PV → VG → LV mappings.

Which next step should I take?
