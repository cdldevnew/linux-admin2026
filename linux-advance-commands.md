# 🔥 Advanced Linux Administration — Hands-On Lab Guide

This README contains scenario-based, production-style labs you can safely practice in a VM (CentOS/RHEL7/8, Rocky, AlmaLinux). Each lab includes objectives, step-by-step commands, explanations, verification steps, common pitfalls, troubleshooting hints, and short exercises.

> ⚠️ IMPORTANT: Do NOT use production systems. Use snapshots or throwaway VMs. Labs assume you have root or sudo privileges on the VM.

---

## Quick Links (click to jump)
- 1 — User, Group & sudo
- 2 — Disk, Partition & LVM
- 3 — Filesystem & fstab
- 4 — Systemd & Service Troubleshooting
- 5 — SELinux Practical Troubleshooting
- 6 — Advanced Networking
- 7 — Cron, Automation & Jobs
- 8 — Log Analysis & Auditing
- 9 — YUM/DNF Repo & Package Management
- 10 — NFS Server & Client

---

## Prerequisites
- VM with a supported distro (CentOS/RHEL7/8, Rocky, Alma, or similar).
- Extra virtual disk attached for LVM labs (or use a loopback file).
- Basic knowledge of shell, vi/vim, and networking concepts.
- Backup snapshots or checkpoints available before destructive labs.

Design notes: This README uses headings, code blocks, tables, and callouts for readability. Use your GitHub renderer or a Markdown viewer for the best experience.

---

# 1. User, Group & sudo Labs
Objective: Manage users, groups, sudoers safely, and enforce expiry/policies.

## Task 1 — Create a group and user, add to group
Commands
```bash
# create group
sudo groupadd devops

# create user with home and add to group
sudo useradd -m -G devops john

# set password interactively
sudo passwd john

# verify user details
id john
getent group devops
```
Explanation
- `groupadd` creates a group.
- `useradd -m` creates a home directory at `/home/john`.
- `-G devops` adds the user to the `devops` supplementary group.
- `id` shows uid, gid, and groups for the user.

Verification
- Confirm `/home/john` exists and permissions are owned by john.
- Run: `su - john` and `groups` to confirm membership.

Common pitfalls & troubleshooting
- If username exists, `useradd` will fail — check with `getent passwd john`.
- To add an existing user to a group: `usermod -aG devops john` (note `-a` is required to append).

Exercise
- Create a user `alice` in group `devops` and verify she can create files in `/home/alice`.

---

## Task 2 — Create a user with expiration
Commands
```bash
sudo useradd -m -e 2025-12-31 tempuser
sudo passwd tempuser
sudo chage -l tempuser
```
Explanation
- `-e` sets account expiration date (format YYYY-MM-DD).
- `chage -l` lists password and expiration info.

Verification
- Attempt to log in after expiration date (use simulation by setting expiration to yesterday) to confirm lock-out.

Tip
- For locking/unlocking: `sudo usermod -L username` / `sudo usermod -U username`.

---

## Task 3 — Grant sudo access (via drop-in, safe)
Commands
```bash
# use visudo to create a file in /etc/sudoers.d/
sudo visudo -f /etc/sudoers.d/john
# add this line:
# john ALL=(ALL) NOPASSWD:ALL

# test as john
su - john
sudo -l          # lists allowed commands
sudo ls /root
```
Explanation
- Editing files in `/etc/sudoers.d/` is safer than changing `/etc/sudoers`.
- Always use `visudo` to validate syntax.

Troubleshooting
- If sudo fails due to syntax error, fix using root or reboot into rescue. `visudo` validates and prevents broken files.

Security note
- Avoid `NOPASSWD:ALL` in production. Instead specify minimal commands, e.g.:
  `john ALL=(ALL) NOPASSWD:/usr/bin/systemctl restart httpd`

Exercise
- Create a `deploy` sudoers file that allows `deploy` user to restart `nginx` without password.

---

# 2. Disk, Partition & LVM Labs
Objective: Add disks, create partitions, build LVM, extend volumes online.

Preparation: Add an extra virtual disk (e.g., /dev/sdb) to the VM before starting.

## Task 1 — Detect new disk
Commands
```bash
lsblk
# If hot-added SCSI, rescan
echo "- - -" | sudo tee /sys/class/scsi_host/host0/scan
# re-run lsblk to confirm
lsblk
```
Explanation
- `lsblk` lists block devices and partitions.
- Rescan triggers kernel to detect newly attached storage.

Troubleshooting
- If `host0` doesn't exist, list hosts: `ls /sys/class/scsi_host/` and rescan each.

---

## Task 2 — Create partition with fdisk (interactive)
Commands (example)
```bash
sudo fdisk /dev/sdb
# within fdisk:
# n  -> new partition
# p  -> primary
# 1  -> partition number
# <enter> for defaults
# w  -> write table and exit

# Verify:
sudo partprobe /dev/sdb   # notify kernel
lsblk
```
Explanation
- `fdisk` manipulates MBR/GPT partition tables (use `gdisk` for GPT).
- `partprobe` asks kernel to read partition table immediately.

Common issues
- If partition not visible, run `sudo partprobe` or reboot if necessary.

---

## Task 3 — Build LVM on partition
Commands
```bash
# create LVM PV on new partition
sudo pvcreate /dev/sdb1

# create volume group
sudo vgcreate vgdata /dev/sdb1

# create logical volume 5G named lvapps
sudo lvcreate -L 5G -n lvapps vgdata

# create filesystem (XFS recommended on RHEL-family)
sudo mkfs.xfs /dev/vgdata/lvapps

# create mountpoint and mount
sudo mkdir -p /apps
sudo mount /dev/vgdata/lvapps /apps

# persist in fstab (see filesystem labs for safe entry)
echo '/dev/vgdata/lvapps /apps xfs defaults 0 0' | sudo tee -a /etc/fstab
```
Explanation
- `pvcreate` initializes physical volume for LVM.
- `vgcreate` creates the VG; `lvcreate` makes LVs.
- XFS filesystems can be grown online with `xfs_growfs`.

Verification
- `sudo pvs`, `sudo vgs`, `sudo lvs` show LVM state.
- `df -h /apps` confirms mount and capacity.

---

## Task 4 — Extend LV online
Commands
```bash
# extend the logical volume by +2G
sudo lvextend -L +2G /dev/vgdata/lvapps

# grow filesystem (XFS)
sudo xfs_growfs /apps

# verify new size
df -h /apps
```
Explanation
- `lvextend` increases the LV size. For XFS, use `xfs_growfs` to extend the filesystem on mounted LV.
- For ext4 use `resize2fs` (unmount or offline extension may be required depending on kernel/tools).

Troubleshooting
- If `xfs_growfs` fails: ensure mountpoint is correct and filesystem is XFS. Use `mount | grep /apps` and `sudo file -sL /dev/vgdata/lvapps`.

Exercise
- Add another 1G thin volume (if thin pool exists) or extend the LV again and confirm resize.

---

# 3. Filesystem & fstab Labs
Objective: Add safe fstab entries, use tmpfs, prevent boot failures.

## Task 1 — Mount tmpfs (ephemeral RAM-backed)
Commands
```bash
sudo mkdir -p /mnt/tmpdata
sudo mount -t tmpfs -o size=512M tmpfs /mnt/tmpdata
df -h /mnt/tmpdata
```
Explanation
- tmpfs uses RAM (swap if configured). Good for caches or ephemeral temp storage.

Warning
- Do not put critical persistent data on tmpfs.

---

## Task 2 — Add safe fstab entry
Best practice: use `_netdev` for network mounts and `nofail` to avoid boot hang.
Example fstab line:
```
/dev/vgdata/lvapps /apps xfs defaults,_netdev,nofail 0 0
```
Safe workflow
1. Add entry to `/etc/fstab`.
2. Test with `sudo mount -a` before reboot.
3. Use UUIDs for device stability:
   ```bash
   sudo blkid /dev/mapper/vgdata-lvapps
   # add line using UUID=...
   ```
Explanation
- `nofail` permits boot to continue if the device is missing.
- `mount -a` re-applies fstab entries and validates syntax.

Troubleshooting
- If `mount -a` errors, check `/var/log/messages` or `journalctl -xe` for mount errors.

Exercise
- Replace `/dev/vgdata/lvapps` fstab entry with a UUID-based entry and test.

---

# 4. Systemd & Service Troubleshooting Labs
Objective: Create services, manage dependencies, troubleshoot failures.

## Task 1 — Create a simple systemd service
Commands
```bash
# create the script
sudo tee /usr/local/bin/hello.sh > /dev/null <<'EOF'
#!/bin/bash
echo "$(date): Hello Service ran" >> /tmp/hello-service.log
sleep 2
EOF
sudo chmod +x /usr/local/bin/hello.sh

# create unit file
sudo tee /etc/systemd/system/hello.service > /dev/null <<'EOF'
[Unit]
Description=Hello Service Example
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/hello.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# enable and start
sudo systemctl daemon-reload
sudo systemctl enable --now hello.service

# check status and logs
sudo systemctl status hello.service
sudo journalctl -u hello.service -b
```
Explanation
- `Type=simple` runs the `ExecStart` process directly.
- `Restart=on-failure` restarts on non-zero exit codes.
- Use `systemctl daemon-reload` after changing unit files.

Troubleshooting
- Permission denied: ensure script is executable and owned by root or correct user.
- Unit fails immediately: inspect `journalctl -u <unit>` for stderr output.

Exercise
- Modify service to run as a non-root user (add `User=someuser` under [Service]) and verify.

---

# 5. SELinux Practical Labs
Objective: Diagnose AVC denials, apply correct contexts, and use boolean settings.

## Task 1 — Check SELinux mode
Commands
```bash
getenforce    # Enforcing, Permissive, Disabled
sestatus -v
```
Explanation
- Enforcing blocks and logs denials.
- Permissive logs denials but does not enforce.

---

## Task 2 — Fix webserver permission denial (safe method)
Scenario: Serve content from `/web` (custom dir) using httpd.

Commands
```bash
# install httpd if needed
sudo yum install -y httpd

# prepare content
sudo mkdir -p /web
echo "test page" | sudo tee /web/index.html
sudo chown -R apache:apache /web
sudo chmod -R 0755 /web

# apply SELinux filecontext and restore
sudo semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"
sudo restorecon -Rv /web

# start webserver
sudo systemctl enable --now httpd
```
Explanation
- `semanage fcontext -a` defines persistent SELinux labeling rules.
- `restorecon` reapplies contexts on files in the path.
- `httpd_sys_content_t` is the type httpd reads by default.

Verification
- Access `http://<server-ip>/` or use `curl http://localhost` on the server.

Troubleshooting
- Check denials: `sudo ausearch -m avc -ts recent` or `sudo journalctl -t setroubleshoot`.
- Use `sudo sealert -a /var/log/audit/audit.log` if setroubleshoot installed (provides suggested fixes).

Tip
- Avoid disabling SELinux; prefer fixing context or booleans (`setsebool`).

---

# 6. Advanced Networking Labs
Objective: Configure static IPs, firewall rules, and troubleshoot connectivity.

## Task 1 — Assign static IP (RHEL/CentOS network-scripts or nmcli)
Network-scripts example (legacy):
```bash
sudo cp /etc/sysconfig/network-scripts/ifcfg-eth0 /tmp/ifcfg-eth0.bak

# edit /etc/sysconfig/network-scripts/ifcfg-eth0:
# BOOTPROTO=none
# IPADDR=192.168.1.50
# PREFIX=24
# GATEWAY=192.168.1.1
# DNS1=8.8.8.8
sudo systemctl restart network
ip addr show dev eth0
```
nmcli example (NetworkManager-controlled)
```bash
sudo nmcli con mod "System eth0" ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 ipv4.dns 8.8.8.8 ipv4.method manual
sudo nmcli con up "System eth0"
```
Verification
- `ip route` and `ping 8.8.8.8` check reachability.
- `dig` or `nslookup` checks DNS.

Troubleshooting
- Race with DHCP: disable DHCP client on the interface if necessary.
- For cloud VMs, use cloud-init settings rather than manual edits.

---

## Task 2 — Open firewall port (firewalld)
Commands
```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```
Explanation
- `--permanent` writes to config; `--reload` applies it.
- `firewall-cmd --list-ports` lists open ports.

Troubleshooting
- If firewalld inactive, start with `sudo systemctl start firewalld`.
- For iptables-based systems use `iptables` commands or legacy tools.

Exercise
- Open port 8443/tcp only for a specific zone or source CIDR.

---

# 7. Cron, Automation & Jobs
Objective: Schedule, audit, and troubleshoot cron jobs.

## Task — Create a cron job that runs every 5 minutes
Commands
```bash
# edit current user's crontab
crontab -e

# add line:
*/5 * * * * /usr/bin/date >> /tmp/time.log 2>&1

# list crontab
crontab -l

# verify the job runs
sleep 310; tail -n 5 /tmp/time.log
```
Explanation
- Redirect stderr to logfile for debugging (`2>&1`).
- Use full paths to binaries in cron.

Troubleshooting
- If job doesn't run, check `/var/log/cron` (RHEL) or `journalctl -u cron` (Debian), and ensure environment variables and PATH are correct. Cron runs with a minimal PATH.

Tip
- For system cron tasks use `/etc/cron.d/` with a file that has an owner and permission 0644.

Exercise
- Create a cron job that executes a script and emails output to you (install and configure `mailx` if needed).

---

# 8. Log Analysis & System Auditing
Objective: Use journalctl, audit logs, and grep to find problems.

## Useful commands
```bash
# systemd journal
sudo journalctl -xe             # extended recent logs (interactive)
sudo journalctl -u sshd -b     # show sshd logs since boot

# system log files (RHEL/CentOS)
sudo tail -f /var/log/messages

# audit logs (requires auditd)
sudo ausearch -m USER_LOGIN --start today
sudo aureport --summary

# search for keywords
sudo grep -i "failed" /var/log/messages | tail -n 50
```
Tips
- Use `journalctl --since "2025-12-26 10:00"` for time-scoped queries.
- `ausearch` is for audit subsystem events (requires `auditd`).

Exercise
- Identify failed SSH login attempts over the past day and count unique source IPs.

---

# 9. YUM/DNF Repo & Package Management
Objective: Manage repos, clear caches, and install packages safely.

## Commands
```bash
# backup repo files
sudo cp /etc/yum.repos.d/*.repo /tmp/

# clean local cache
sudo yum clean all

# list repositories
sudo yum repolist

# install a package
sudo yum install -y vsftpd

# remove package
sudo yum remove -y vsftpd
```
Notes
- On RHEL8 and Fedora, `dnf` is preferred (`dnf clean all`, `dnf install -y package`).
- Use `yum downgrade` or `dnf history` to undo package changes if needed.

Troubleshooting
- If repo metadata fails, check `curl` to baseurl or mirrorlist and DNS resolution.

Exercise
- Create a local YUM repo for a directory of RPMs and configure clients to use it.

---

# 10. NFS Server & Client Labs
Objective: Export, mount NFS shares securely; manage permissions & SELinux contexts.

## Server setup (example RHEL/CentOS)
Commands
```bash
sudo yum install -y nfs-utils
sudo mkdir -p /shared
echo "hello nfs" | sudo tee /shared/test.txt
sudo chown -R nfsnobody:nfsnobody /shared
sudo chmod 755 /shared

# export (append to /etc/exports)
echo "/shared 192.168.1.0/24(rw,sync,no_root_squash)" | sudo tee -a /etc/exports

# apply exports and enable services
sudo exportfs -rav
sudo systemctl enable --now nfs-server
sudo firewall-cmd --add-service=nfs --permanent
sudo firewall-cmd --reload
```
Explanation
- `no_root_squash` maps remote root to local root — use carefully.
- Use CIDR to restrict access.

## Client mount
Commands
```bash
sudo yum install -y nfs-utils
sudo mount -t nfs server-ip:/shared /mnt
ls -la /mnt
```
Persist mount in `/etc/fstab`:
```
server-ip:/shared /mnt nfs defaults,_netdev 0 0
```

Troubleshooting
- If mount fails, check `rpcinfo -p server-ip` and `showmount -e server-ip`.
- SELinux on server may need `semanage` rules for NFS export contexts.

Exercise
- Configure NFS export with `root_squash` and verify root on client cannot create files as root on server.

---

## Common Tools & Commands Cheat Sheet
- Filesystem: lsblk, fdisk, mkfs.xfs, mount, umount, df, du
- LVM: pvcreate, vgcreate, lvcreate, lvextend, lvdisplay
- Systemd: systemctl start/stop/status/enable/disable, journalctl
- SELinux: getenforce, semanage, restorecon, sealert, ausearch
- Network: ip addr/route, nmcli, ping, traceroute, ss, firewall-cmd
- Audit & Logs: ausearch, aureport, journalctl, tail, grep
- Package: yum/dnf, rpm, repoquery

---

## Safety Checklist (Before making changes)
- Take a VM snapshot.
- Backup config files (cp to /tmp or git).
- Test changes in non-production.
- Use `mount -a` to validate fstab.
- Use `visudo` to edit sudoers safely.

---

## Recommended Exercises & Learning Path
1. Build a VM and perform the LVM lab end-to-end (partition, vg, lv, mkfs, mount, fstab).
2. Create a custom systemd service that runs a small web app and enable it.
3. Configure SELinux context for a custom web directory and debug with `ausearch`.
4. Simulate disk full conditions and find large files using `du` and `lsof`.
5. Set up one VM as an NFS server and one as client and persist the mount.

---

## Final Notes on "Effective Font & Design"
- Markdown does not control fonts on GitHub; GitHub renders using its default stylesheet.
- For improved readability:
  - Use headings and subheadings (as done here).
  - Use code blocks for commands.
  - Use tables for quick references (seen in the cheat sheet).
  - Add badges and images to the repo README for visual cues.

---
- Generate a printable PDF or a lightweight HTML site (MkDocs) from these labs.

Which option should I do next?
