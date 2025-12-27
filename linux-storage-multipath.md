# 💾 Storage & Multipath — Hands‑On Troubleshooting Labs (Detailed Commands & Explanations)

These labs focus on enterprise storage workflows you’ll see in production: LUN discovery, iSCSI sessions, multipath (device‑mapper), WWIDs, path failover and recovery, udev naming, filesystem mounts on multipath devices, online resize, and RCA. Practice only in lab VMs or isolated testbeds — never experiment on production storage without change control and backups.

High‑level approach for any storage incident:
- Stop, observe, collect (don’t immediately run destructive commands).
- Identify the device using persistent identifiers (WWID/WWN/UUID), not `/dev/sdX`.
- Correlate host logs (`dmesg`, `journalctl`), multipath state, and SAN/controller events.
- Make minimal, reversible changes; document every step.

---

## 📌 Lab Index

1. Discovering new storage LUNs safely (no reboot)
2. Identify disks reliably — use WWID / WWN / by-id links
3. Install, enable and inspect device‑mapper multipath (multipathd)
4. Configure multipath with friendly aliases and blacklists
5. Simulate path failure and verify automatic failover
6. Path recovery, rescans and multipath reconfigure
7. Resize LUNs online (PV → VG → LV → FS)
8. iSCSI initiator discovery, login and troubleshooting
9. Boot problems when root is on multipath device — initramfs & udev
10. Storage RCA checklist & structured interview answer

Each lab contains: symptoms, commands to gather evidence, exact commands with explanations, persistence options, verification, and common pitfalls.

---

## Lab 1 — Discover new LUNs safely (without reboot)

Goal
- Detect newly presented LUNs and confirm device nodes appear.

What to collect first
```bash
# baseline
lsblk -o NAME,KNAME,SIZE,MODEL,MOUNTPOINT,WWN
multipath -ll        # if multipath configured
blkid
```

Trigger SCSI rescan (safe, non‑disruptive)
```bash
# Rescan all SCSI hosts
for host in /sys/class/scsi_host/host*; do
  echo "- - -" > "$host/scan"
done

# Verify kernel dmesg
dmesg | tail -n 40
# Confirm new block devices
lsblk -o NAME,KNAME,SIZE,MODEL,MOUNTPOINT,WWN
```

What this does
- Writing `- - -` to each host's scan requests a rescan of target, channel and LUN. It causes the kernel to detect new or changed LUNs without rebooting.

Verification pointers
- New devices appear in `lsblk`. The host kernel will log discovered device names in `dmesg`. Never assume `/dev/sdX` ordering — identify by WWN/WWID.

Common pitfalls
- Hypervisor-level storage changes (vSphere/Hyper‑V) might require storage team action to rescan on host side or path changes in host bus adapters.
- Rescans can flood logs; run during maintenance windows on busy hosts.

---

## Lab 2 — Identify disks reliably (WWID / WWN / by-id)

Goal
- Map the correct LUN to persistent identifiers to avoid mistakes when device names change.

Commands & reasoning
```bash
# Show persistent device identifiers
ls -l /dev/disk/by-id/
ls -l /dev/disk/by-wwn/
ls -l /dev/disk/by-uuid/

# For modern udev scsi_id (if available)
/usr/lib/udev/scsi_id -g -u -d /dev/sdX
# or (RHEL path)
scsi_id -g -u -d /dev/sdX
```

Why WWID/WWN?
- WWIDs (3600...) and WWNs are stable identifiers assigned by the storage array and udev; `/dev/sdX` can change after reboot or rescan.

Example mapping inspection
```bash
# Show a device with multipath mapping
multipath -ll
# Show mapper device symlink
ls -l /dev/mapper/data_lun1
readlink -f /dev/mapper/data_lun1
```

Common pitfalls
- Some virtual disk platforms (cloning/templates) may create duplicate WWIDs — always confirm uniqueness with storage team.

---

## Lab 3 — Install & enable device‑mapper multipath

Goal
- Install daemon and verify multipath topology.

Install and start
```bash
# RHEL/CentOS
yum install -y device-mapper-multipath lsscsi

# Debian/Ubuntu
apt update
apt install -y multipath-tools

# Enable/start
systemctl enable --now multipathd
```

Quick verification
```bash
multipath -ll      # show mapped multipath devices and path groups
dmsetup ls --tree  # device-mapper tree of maps
lsblk -o NAME,TYPE,KNAME,MAJ:MIN | egrep 'mpath|dm-'   # mapped devices
```

What multipath -ll shows
- WWID, mapper name, path groups, each path device (e.g., /dev/sdb), path status (ALWAYS, active, failed), I/O policy and queueing.

Safety note
- Do not enable multipath on systems where it would wrap the root device unless you fully understand initramfs integration and bootloader configuration.

---

## Lab 4 — Configure multipath with friendly aliases and blacklists

Goal
- Create stable, human-friendly aliases and exclude unwanted devices.

Edit config
```bash
# Backup and edit
cp /etc/multipath.conf /etc/multipath.conf.bak
vi /etc/multipath.conf
```

Recommended minimal `/etc/multipath.conf` structure
```ini
defaults {
    user_friendly_names yes
    polling_interval 10
}

blacklist {
    # blacklist local disks or certain vendors
    devnode "^sda"
    wwid "3600a09876..."   # example WWID to ignore
}

multipaths {
    multipath {
        wwid 36001405eb123456789abc00011122233
        alias data_lun1
        path_grouping_policy multibus
        path_selector "round-robin 0"
        failback immediate
        rr_weight uniform
    }
}
```

Apply changes
```bash
systemctl reload multipathd
# Or reconfigure and rebuild map
multipath -r         # reload all maps
multipath -ll
```

Explanations
- `user_friendly_names yes` creates `/dev/mapper/mpatha` style names or uses `alias` if provided.
- `blacklist` prevents local disks (e.g., boot disk) from being captured by multipath.

Verification
```bash
ls -l /dev/mapper
multipath -ll | egrep 'data_lun1|36001405'
```

Pitfalls
- A mistaken blacklist or incorrect WWID alias can hide devices. Always maintain backups of `/etc/multipath.conf`.

---

## Lab 5 — Simulate path failure and verify automatic failover

Goal
- Confirm multipath failover works and that I/O continues on surviving paths.

Collect baseline
```bash
multipath -ll
# Note active path group and PV/LV using the multipath device
lsblk -o NAME,KNAME,MOUNTPOINT | grep mapper
```

Simulate path loss
- Methods (lab only):
  - Disable one NIC on the host or on the switch port.
  - Log out a single iSCSI session: `iscsiadm -m node -T <IQN> -p <target> --logout` (lab safe).
  - On the storage front, administratively disable a path in SAN simulator.

Observe multipath behavior
```bash
# Watch multipath
watch -n 1 multipath -ll
# Or tail multipathd log
journalctl -u multipathd -f
```

Expected results
- Multipath marks one path as failed and switches active I/O to another path group (if available).
- No application I/O errors (e.g., database should not show I/O failures).

Verification IO test
```bash
# Run an I/O test before and during failover (lab-friendly)
dd if=/dev/zero of=/mnt/testfile bs=1M count=1024 oflag=direct
# Monitor iostat on the underlying device names
iostat -x 1 5
```

Safety & rollback
- Re-enable NIC or login iSCSI session after test; confirm multipath auto‑recovery.

Common pitfalls
- Incorrect `path_grouping_policy` or path selector may cause suboptimal failover behaviour.
- If all paths to a LUN are through the same physical fabric, “failover” does nothing; coordinate with storage team for redundant fabric.

---

## Lab 6 — Path recovery, rescans & multipath reconfigure

Goal
- Recover paths after they are restored and ensure multipath maps rejoin cleanly.

Rescan SCSI and reconfigure multipath
```bash
# SCSI rescan
for host in /sys/class/scsi_host/host*; do echo "- - -" > "$host/scan"; done

# Ask multipathd to reconfigure maps
multipathd reconfigure

# Alternatively
multipath -r
```

Check state
```bash
multipath -ll
journalctl -u multipathd -n 200
dmesg | tail -n 50
```

If stale paths remain
```bash
# Remove a path from a map (be careful)
multipathd -k "fail path '3600...' ; remove path 'sdb'"

# Re-add path when available (multipathd should re-add on reconfigure)
multipathd -k "reconfigure"
```

Explanation
- `multipathd reconfigure` reloads /etc/multipath.conf, reconciles maps and removes stale device entries. SCSI rescans instruct kernel to re-enumerate LUNs.

Pitfalls
- Don’t remove active paths for a map with I/O in flight. Prefer controlled tests and consult application owners.

---

## Lab 7 — Resize multipath device online (LUN extend → PV/VG/LV → FS grow)

Goal
- Handle storage team resizing a LUN and perform host-side growth without downtime.

Scenario
- Storage team extended LUN; host must detect new size and extend PV / LV / FS.

Steps
1) Rescan SCSI bus
```bash
for host in /sys/class/scsi_host/host*; do echo "- - -" > "$host/scan"; done
dmesg | tail -n 40
```

2) Notify multipath of the change and resize map
```bash
# Resize the multipath map (use alias or WWID)
multipathd resize map data_lun1
multipath -ll | grep -i size
```

3) Confirm block device new size
```bash
# mapper device shows new size
lsblk -o NAME,SIZE,TYPE | grep data_lun1
```

4) Grow PV (LVM) on multipath device
```bash
# If PV is /dev/mapper/data_lun1
pvresize /dev/mapper/data_lun1
pvs -o+pv_size,lv_size   # check free PE space
```

5) Extend LV
```bash
# Use all free extents
lvextend -l +100%FREE /dev/vgdata/lvapps
# Verify logical volume growth
lvs -o+lv_size
```

6) Grow filesystem (examples)
- XFS:
```bash
# Ensure filesystem mounted and XFS; can grow online
xfs_growfs /apps      # mountpoint of LV
```
- ext4:
```bash
resize2fs /dev/vgdata/lvapps
```

Verification
```bash
df -h /apps
lsblk -o NAME,SIZE,MOUNTPOINT | grep vgdata
```

Important notes
- Always confirm filesystem type supports online grow (XFS, ext4 typically do).
- Take backups / snapshots before resizing in production.

---

## Lab 8 — iSCSI initiator discovery, login & troubleshooting

Goal
- Discover iSCSI targets, login, and resolve common errors (auth, firewall).

Install and basic workflow
```bash
# Install (RHEL/CentOS)
yum install -y iscsi-initiator-utils

# Debian/Ubuntu
apt install -y open-iscsi

# Discovery
iscsiadm -m discovery -t sendtargets -p <target-ip>

# Login to all discovered nodes
iscsiadm -m node --login

# Show sessions
iscsiadm -m session -o show
```

Common troubleshooting commands
```bash
# Show initiator IQN (client name)
cat /etc/iscsi/initiatorname.iscsi

# View open iSCSI sessions & targets
iscsiadm -m session

# Logout a session (lab)
iscsiadm -m node -T <target-iqn> -p <target-ip> --logout

# To delete a node record
iscsiadm -m node -o delete -T <target-iqn> -p <target-ip>
```

Common errors & remedies
- Authentication failure (CHAP): confirm initiator/target CHAP secrets and IQNs match.
- Firewall blocks: ensure TCP 3260 allowed between initiator and target.
- Multipath & iSCSI: ensure nodes are logged in on all NICs and that LUNs present on all paths.

Logs to inspect
```bash
journalctl -u iscsid -f
dmesg | egrep -i 'iscsi|scsi|sd'
```

Pitfalls
- Repeated login/logout tests can create node records; clean up stale sessions with `iscsiadm -m node -o delete`.
- For production, coordinate with storage admins to avoid unintended LUN unmapping.

---

## Lab 9 — Boot problems when root is on multipath device

Symptoms
- System drops to emergency shell: “cannot find root device /dev/mapper/…”
- Boot HALTS because initramfs lacks multipath or udev rules.

Safe investigation
1) Check GRUB entries and root specification use UUID/WWID rather than `/dev/sdX`.
2) Boot from rescue media if host unbootable. Then chroot and inspect initramfs.

Ensure initramfs includes multipath and is configured
```bash
# Rebuild initramfs with multipath support (RHEL example)
dracut -f --add 'multipath' /boot/initramfs-$(uname -r).img $(uname -r)

# Or ensure dracut includes multipath in config:
echo 'add_dracutmodules+=" multipath "' >> /etc/dracut.conf.d/multipath.conf
dracut -f

# Regenerate grub config (if device map or UUID changed)
grub2-mkconfig -o /boot/grub2/grub.cfg
```

Check udev & multipath startup ordering
- `/etc/multipath.conf` should be correct and multipathd started early by initramfs.
- For UEFI/GRUB, ensure GRUB references `/dev/mapper/<name>` consistent with aliases and that `lvm`/`multipath` modules are loaded.

Verification
- Reboot and watch kernel boot logs: `journalctl -b -1 -e` if previous boot failed.
- If rescue used, ensure you can mount root and access `/dev/mapper/*` after chroot steps.

Important safety
- Changing initramfs or grub on remote systems without console access risks permanent lockout; ensure out‑of‑band console before making changes.

---

## Lab 10 — Storage incident RCA checklist (what to capture & sample interview answer)

What to capture immediately (collect and archive)
```bash
# Multipath & dmsetup
multipath -ll > /tmp/multipath-ll.log
dmsetup ls --tree > /tmp/dmsetup-tree.log

# Block devices & WWN
lsblk -o NAME,KNAME,SIZE,TYPE,WWN,MOUNTPOINT > /tmp/lsblk.log
ls -l /dev/disk/by-id > /tmp/disk-by-id.log

# iSCSI sessions
iscsiadm -m session -o show > /tmp/iscsi-session.log

# Kernel & multipath logs
dmesg | tail -n 200 > /tmp/dmesg-tail.log
journalctl -u multipathd -n 200 > /tmp/multipathd.log

# I/O stats
iostat -x 1 10 > /tmp/iostat.log

# Recent SAN events (if accessible)
# coordinate with storage team to get array logs or SAN switch events
```

Structured interview-style RCA answer
- Short summary: “At 10:32 UTC we observed I/O errors for /dev/mapper/data_lun1. Multipath showed two failed paths and one active. dmesg contained repeated SCSI transport errors which correlated to a fabric change on the SAN. After storage team re-enabled the path, we rescanned SCSI and ran `multipathd reconfigure`. I/O resumed with no data loss. Root cause: scheduled SAN maintenance that took one path offline; mitigation: ensure redundant fabrics for initiator and update runbook to verify multipath config and alerts for path state changes.”
- Evidence you’d present: multipath output, `dmesg` SCSI errors, SAN maintenance ticket/timestamps, iostat showing latency spike, logs showing multipath failover.

---

## Tools & Commands Cheat Sheet

- Device discovery & identification:
  - lsblk, blkid, /usr/lib/udev/scsi_id (or scsi_id), ls /dev/disk/by-id, by-wwn
- Multipath:
  - multipath -ll, multipath -r, multipathd reconfigure, multipathd show paths
- Device-mapper:
  - dmsetup ls --tree, dmsetup info /dev/mapper/mpathX
- LVM:
  - pvdisplay, pvresize, vgs, lvs, lvextend
- iSCSI:
  - iscsiadm -m discovery -t sendtargets -p <ip>; iscsiadm -m node --login
- Rescan:
  - for host in /sys/class/scsi_host/host*; do echo "- - -" > $host/scan; done
- Logs:
  - dmesg, journalctl -u multipathd, journalctl -k, /var/log/messages
- Filesystem ops:
  - xfs_growfs, resize2fs, mount/umount, xfs_repair (read docs before use)
- Safety & diagnostics:
  - multipath -F (very dangerous — clears maps; use only when instructed)
  - multipath -v3 (verbose debugging)
  - `udevadm monitor --udev` to watch udev events during rescan

---

## Safety & Best Practices

- Always identify LUNs by WWID/WWN before formatting or partitioning.
- Avoid using `/dev/sdX` in fstab; prefer `/dev/mapper/alias` (multipath alias) or UUIDs.
- Coordinate with storage and networking teams before path-failover tests if in production.
- Keep `multipath.conf` under version control with clear comments and change history.
- For boot-on-multipath setups, test initramfs changes in a controlled environment first.
- Backup critical metadata (LVM metadata backup, fstab copy, multipath.conf).

---

## Example: Quick Recovery Playbook (summary)

1. Collect: `multipath -ll`, `dmsetup ls --tree`, `lsblk`, `dmesg`, `journalctl -u multipathd`.
2. Identify if failure is host‑side (NIC/driver), fabric, or array.
3. If a path is down, confirm redundancy and run a scsi rescan after restoring hardware.
4. Use `multipathd reconfigure` and `multipath -r` to rebuild maps.
5. If a device is stale, coordinate with storage team and avoid forcing removals during heavy I/O.
6. Validate I/O with `iostat` and application checks.
7. Document timeline and actions, escalate to storage vendor if hardware/controller issue.

---

You now have a practical, command‑focused set of storage & multipath troubleshooting labs, with safe, repeatable steps and verification checks tailored for real enterprise scenarios. Practice in lab VMs, collect evidence before making changes, and always coordinate risky operations with the storage/infrastructure owners.
