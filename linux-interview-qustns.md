# 🎯 Linux Admin — Interview Scenario Labs

These labs simulate production incidents and troubleshooting tasks commonly asked in interviews for Linux Admin, DevOps/SRE, Cloud Support and RHCSA/RHCE roles.

Practice each scenario in an isolated VM (VirtualBox / VMware / cloud). Use snapshots and backups. This document gives the problem statement, step‑by‑step commands, explanation, verification, common pitfalls, and a short exercise to practice.

> ⚠️ Never run destructive troubleshooting on production systems. Use snapshots or disposable VMs.

---

## Lab Index (click to jump)
1. Disk 100% + app down  
2. Boot stuck due to bad /etc/fstab  
3. SSH "Permission denied" (publickey)  
4. High CPU utilization troubleshooting  
5. Memory leak / swapping investigation  
6. Service works manually but not on reboot  
7. Time sync / NTP issues  
8. Website not loading — Apache troubleshooting  
9. DNS resolution failure analysis  
10. Accidental file deletion — recovery methods

---

# Scenario 1 — Disk is 100% full, application down

Problem
- Users/app report: "No space left on device" and application stopped.

Quick checklist
1. Identify full mount(s)
2. Find big files/directories
3. Free space safely (truncate, rotate, move)
4. Prevent recurrence

Commands & steps
```bash
# 1. Which filesystem(s) are full?
df -hT

# 2. Find top disk usage at root level
du -sh /* 2>/dev/null | sort -h

# 3. Drill into culprit (e.g., /var)
du -sh /var/* 2>/dev/null | sort -h

# 4. Find very large files
find / -xdev -type f -size +500M -exec ls -lh {} \;

# 5. Truncate large log safely (no delete)
sudo > /var/log/messages
sudo > /var/log/secure

# 6. Remove rotated archives (older .gz) carefully
sudo rm -f /var/log/*.gz

# 7. Clean package caches (RHEL)
sudo yum clean all
```

Explanation
- Truncating (`> file`) keeps file handle active for processes and frees space. Deleting file used by running process does not free space until process closes file descriptor—use `lsof | grep deleted` to find such cases.

Verification
```bash
df -h
lsof +L1     # list open files deleted from filesystem
```

Pitfalls & tips
- Don't immediately delete unknown files; copy or compress first.
- Move large non-critical files to another mount or to /tmp and then remove.
- Set up logrotate with compression and retention.

Exercise
- Fill a test LV to ~95% and practice safely freeing space and extending the LV.

---

# Scenario 2 — Boot stuck / emergency due to /etc/fstab

Problem
- System drops to emergency mode after boot with mount failures.

Commands & steps
```bash
# 1. Inspect recent boot errors
journalctl -xb

# 2. Boot rescue or use live media / press 'e' in grub to edit kernel args and add 'systemd.unit=rescue.target'

# 3. Once in rescue, edit fstab
sudo vi /etc/fstab

# 4. Comment the problematic line and add 'nofail' for non-critical mounts
# Example:
# UUID=xxxx /backup xfs defaults,nofail 0 0

# 5. Test mounts
sudo mount -a

# 6. Reboot
sudo reboot
```

Explanation
- A failing mount in fstab without `nofail` or wrong UUID blocks normal boot. Using UUIDs prevents device name drift (/dev/sdX changes).

Verification
```bash
blkid
cat /etc/fstab
mount | grep /backup
```

Pitfalls & tips
- Use `nofail` and `_netdev` for network filesystems.
- Always test `mount -a` after editing fstab.

Exercise
- Add a fake fstab entry for a non-existent device and practice recovering from rescue mode.

---

# Scenario 3 — SSH "Permission denied (publickey)"

Problem
- User can't log in with publickey auth.

Steps & commands
```bash
# 1. Check sshd status and auth logs
sudo systemctl status sshd
sudo tail -n 200 /var/log/secure     # or /var/log/auth.log on Debian

# 2. Inspect .ssh permissions
ls -ld /home/username /home/username/.ssh
ls -l /home/username/.ssh/authorized_keys

# 3. Fix permissions
sudo chown -R username:username /home/username/.ssh
sudo chmod 700 /home/username/.ssh
sudo chmod 600 /home/username/.ssh/authorized_keys

# 4. Confirm SELinux contexts (if SELinux enabled)
sudo restorecon -Rv /home/username/.ssh

# 5. Test from client with verbose
ssh -vvv username@host
```

Explanation
- SSH requires strict permissions: ~/.ssh must be 700 and authorized_keys 600; otherwise sshd rejects keys.

Verification
- Check `/var/log/secure` for messages such as "Authentication refused: bad ownership or modes for directory /home/username".

Pitfalls
- Home directory with 777 or group-write can break key auth.
- SELinux context may prevent sshd reading authorized_keys.

Exercise
- Intentionally break permissions and fix them while watching auth logs.

---

# Scenario 4 — High CPU utilization

Problem
- CPU usage near 100% and service degradation.

Steps & commands
```bash
# 1. Brief snapshot
uptime
top -b -n1 | head -n20

# 2. Identify top CPU consumers
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -n 10

# 3. Get full command line
ps -p <PID> -o pid,user,etime,args

# 4. If safe, terminate or restart process
sudo systemctl restart servicename
# or
sudo kill -15 <PID>
```

Explanation
- High CPU could be due to a runaway process or high legitimate load. Investigate process purpose before killing—look at command line and owner.

Verification
- Monitor with `top` or `htop` and `sar` if installed.

Pitfalls
- `kill -9` may cause resource leaks—prefer graceful `kill` then `kill -9` if it doesn't stop.

Exercise
- Run a CPU-intensive job (stress-ng or a small busy loop) and practice identifying and throttling it (nice/renice).

---

# Scenario 5 — Memory leak / swapping

Problem
- System swapping, out-of-memory, apps slow.

Steps & commands
```bash
free -m
vmstat 1 5
# processes sorted by memory
ps -eo pid,user,pmem,pcpu,rss,etime,cmd --sort=-rss | head -n 15

# check swap usage
swapon --show

# find processes recently allocating memory
smem -r   # if smem installed
```

Explanation
- Look for processes with growing RSS over time. Swap use is normal but heavy swapping means insufficient RAM.

Remediation
- Restart offending service, add memory, tune application, or adjust oom-killer settings.

Verification
- After restart, `free -m` shows recovered RAM.

Exercise
- Create a small program that leaks memory and monitor its RSS growth; practice safe restart of service.

---

# Scenario 6 — Service works manually but fails after reboot

Problem
- Service can be started manually but doesn't start automatically on boot.

Checks & steps
```bash
systemctl status myapp
systemctl is-enabled myapp
sudo systemctl enable myapp
# check for missing dependencies or ordering issues
systemctl show -p WantedBy,RequiredBy myapp.service
journalctl -u myapp -b
```

Explanation
- Service may not be enabled, or it depends on network, mounts, or LVM that are not ready early in boot. Add proper `After=` or `Requires=` in unit file or enable the service.

Verification
- Reboot test and inspect `systemctl --failed`.

Pitfalls
- Unit file may run as a user that hasn't been created. Ensure proper `User=` and that home/dirs exist.

Exercise
- Create a service that requires a mount; test with and without `After=` ordering and `nofail` in fstab.

---

# Scenario 7 — Time not syncing (NTP/chrony)

Problem
- Clock drift causing cert or auth issues.

Steps & commands
```bash
timedatectl status
sudo timedatectl set-ntp true

# With chrony
sudo systemctl restart chronyd
chronyc sources
chronyc tracking
```

Explanation
- Use `chrony` or `ntpd` to keep time synced. Cloud VMs may use host time or ntp config via cloud-init.

Verification
- `timedatectl` shows `System clock synchronized: yes` and NTP service running.

Pitfalls
- If VM paused/resumed, clock steps may be large—enable slewing or force sync.

Exercise
- Stop chronyd, change system clock manually, then restart and observe how chrony corrects.

---

# Scenario 8 — Website not loading (Apache HTTPD)

Problem
- Browser times out or connection refused.

Steps & commands
```bash
# 1. Service status & logs
sudo systemctl status httpd
sudo journalctl -u httpd -b
sudo tail -n 100 /var/log/httpd/error_log

# 2. Is process listening?
ss -tulnp | grep :80

# 3. Firewall
sudo firewall-cmd --list-all
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload

# 4. SELinux
getenforce
# if content in /home, allow:
sudo setsebool -P httpd_enable_homedirs on
```

Explanation
- Check service, port binding, firewall, and SELinux. Also confirm virtual host configuration and DocumentRoot permission.

Verification
- `curl -I http://localhost` should return HTTP headers.

Pitfalls
- Apache may bind only to 127.0.0.1 when configured incorrectly; confirm `Listen` directive.

Exercise
- Simulate site failing by misconfiguring vhost, debug via error_log, and fix.

---

# Scenario 9 — DNS resolution failing

Problem
- hostname lookup fails; ping returns "unknown host".

Steps & commands
```bash
cat /etc/resolv.conf
ping 8.8.8.8    # test raw IP connectivity
nslookup google.com 8.8.8.8
dig +short @8.8.8.8 www.google.com
```

Explanation
- DNS config (/etc/resolv.conf) may be overwritten by NetworkManager. Verify network connectivity before changing DNS.

Remediation
- Update DNS via NM (`nmcli`) or /etc/resolv.conf, verify with `dig`.

Exercise
- Configure system to use a local DNS resolver (dnsmasq) and test resolution.

---

# Scenario 10 — File deleted accidentally

Problem
- User removed an important file.

Immediate actions
1. If the process still has file open, recover from /proc:
```bash
lsof | grep deleted
# or find fd from pid
sudo ls -l /proc/<PID>/fd | grep '(deleted)'
# copy contents
sudo cp /proc/<PID>/fd/<FD> /tmp/recovered-file
```

2. If not open, restore from backups or snapshots:
- Restore from backups (rsync/tar)
- Use filesystem snapshots (LVM snapshots, btrfs/ZFS) to restore.

3. As last resort, file-carving tools (extundelete, testdisk) on unmounted partitions (risky).

Explanation
- Deleting removes directory entry; data blocks stay allocated until overwritten. Quick action increases chance of recovery.

Interview expectation
- Describe backup cadence, retention, offsite copies, and RTO/RPO.

Exercise
- Delete a test file and recover it using a running process that still holds the FD (create a process to cat file & background it).

---

# Design & Presentation Notes (effective README)

- Use clear headings, code blocks, and short steps.
- GitHub controls fonts; use Markdown structure for readability.
- For printable or styled docs, convert to PDF or MkDocs with a controlled CSS.
- Use tables for quick checks and callouts (⚠️) for warnings.

---
- Generate printable PDF / MkDocs site for interview prep.

Which would you like next?
