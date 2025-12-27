# 🎓 RHCSA / RHCE — Exam‑Style Hands‑On Labs (Detailed Commands & Explanations)

This document collects 40+ exam‑style, hands‑on labs in a structured README.md format. Each lab contains:
- Objective (what you must accomplish)
- Tasks / Example commands (what to run)
- Explanation (why the command is used)
- Persistence (how to make changes survive reboot)
- Verification (how an examiner will check your work)
- Common pitfalls / remediation notes

Important:
- Practice in a VM (CentOS / RHEL / Rocky / Alma).
- Do not run these against production systems.
- No authentication bypass instructions are included.

---

Table of Contents
- Labs 1–10: Core exam tasks (users, networking, LVM, cron, SELinux, HTTPD, NFS, boot targets, processes, firewall)
- Labs 11–20: Extended tasks (archives, ACLs, autofs, swap, systemd services, rich firewall rules, Podman, journald, VDO/LVM thin, find)
- Labs 21–40: Advanced/administrative tasks (aliases, links, SSH keys, YUM repo, snapshots, PAM/password policies, umask, backups, zombies, sudo scoping, hostname, timezone, rsyslog, tmpfs, groups, sysctl, firewalld, autofs timeout, kickstart)

---

## Labs 1–10 — Core RHCSA Tasks

### Lab 1 — Users, Groups & sudo Privileges
Objective: create group, user, lock account, enable passwordless sudo for a user.

Commands & explanation:
```bash
# create devops group
groupadd devops

# create user 'rahul' with home and add to devops
useradd -m -G devops rahul

# set password interactively for rahul (exam may require)
passwd rahul

# lock an existing user (pre-created)
passwd -l tempuser
```
Make sudo entry:
```bash
# create sudoers drop-in for rahul (safer than editing /etc/sudoers)
cat > /etc/sudoers.d/rahul <<'EOF'
rahul ALL=(ALL) NOPASSWD:ALL
EOF
chmod 0440 /etc/sudoers.d/rahul
```
Why: /etc/sudoers.d is used to grant scoped sudo rights safely.

Verification:
```bash
su - rahul
sudo id
```
Pitfalls:
- Wrong file permissions on /etc/sudoers.d will cause sudo to refuse the file. Use 0440.

---

### Lab 2 — Configure Persistent Static Network
Objective: assign static IP permanently for eth0.

Commands & explanation (RHEL-style ifcfg):
```bash
# Edit the ifcfg file
cat > /etc/sysconfig/network-scripts/ifcfg-eth0 <<'EOF'
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.1.50
PREFIX=24
GATEWAY=192.168.1.1
DNS1=8.8.8.8
EOF

# Restart network
systemctl restart network   # or NetworkManager: nmcli con reload; nmcli con up eth0
```
Why: If using NetworkManager, prefer nmcli; ifcfg files are persistent across reboots.

Verification:
```bash
ip addr show eth0
ip route show
ping -c 2 8.8.8.8
```
Pitfalls:
- Using `ifup`/`ifdown` vs NetworkManager; confirm which service manages the interface.

---

### Lab 3 — Partition, LVM & Resize
Objective: create PV → VG `vgdata` → LV `lvapps` 2G; mount on /apps; extend by +1G online.

Commands & explanation:
```bash
# Assume /dev/sdb available
pvcreate /dev/sdb
vgcreate vgdata /dev/sdb
lvcreate -L 2G -n lvapps vgdata

# Make filesystem (XFS recommended on RHEL)
mkfs.xfs /dev/vgdata/lvapps

mkdir -p /apps
mount /dev/vgdata/lvapps /apps

# Persist in /etc/fstab using UUID for safety
blkid /dev/vgdata/lvapps
# add line: UUID=<uuid> /apps xfs defaults 0 0

# Extend online by 1G
lvextend -L +1G -r /dev/vgdata/lvapps
# -r automatically grows filesystem (xfs_growfs) for XFS; for ext4 use resize2fs
```
Why: LVM allows online extension; using UUID in fstab prevents device-name issues.

Verification:
```bash
df -h /apps
lvs -o+lv_size
```
Pitfalls:
- If /apps is on ext4, lvextend requires a separate resize2fs if not using `-r`.

---

### Lab 4 — Schedule Cron Job
Objective: run script every 5 minutes.

Commands & explanation:
```bash
# create script
cat > /usr/local/bin/timelog.sh <<'EOF'
#!/bin/bash
date >> /tmp/time.log
EOF
chmod +x /usr/local/bin/timelog.sh

# edit crontab for root or a specific user
crontab -e
# add the line:
*/5 * * * * /usr/local/bin/timelog.sh
```
Why: crontab schedules recurring tasks; */5 executes every 5 minutes.

Verification:
```bash
tail -n 5 /tmp/time.log
```
Pitfalls:
- Ensure script has absolute paths and proper permissions. Environment differs from interactive shells.

---

### Lab 5 — SELinux Context & Boolean (Serve /web via httpd)
Objective: Allow Apache to serve content from `/web` while SELinux is Enforcing.

Commands & explanation:
```bash
mkdir -p /web
echo "RHCSA Test" > /web/index.html
chown -R apache:apache /web

# Add SELinux file context (persistent)
semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"
restorecon -Rv /web

# If content requires read_user_content or other booleans
setsebool -P httpd_read_user_content 1
```
Why: SELinux contexts control which processes can access which files. `semanage` registers the context and `restorecon` applies it.

Verification:
```bash
curl -sS http://localhost/  # if DocumentRoot is configured to serve /web; else create virtual host
```
Pitfalls:
- semanage may require `policycoreutils-python-utils` package on older distros.

---

### Lab 6 — HTTPD Server Configuration
Objective: Install httpd, create a test index, allow via firewall.

Commands & explanation:
```bash
yum install -y httpd   # or dnf
systemctl enable --now httpd

echo "RHCSA Web Test" > /var/www/html/index.html

# Open firewall
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```
Why: `systemctl enable --now` starts service and ensures it starts on boot.

Verification:
```bash
curl -sS http://localhost
# From remote: curl -sS http://<server-ip>
```
Pitfalls:
- If SELinux denies, check `audit.log` and set httpd contexts for content.

---

### Lab 7 — NFS Server & Client
Objective: Export `/shared` from server; mount on client.

Server commands:
```bash
yum install -y nfs-utils
mkdir -p /shared
chmod 777 /shared
echo "/shared *(rw,sync,no_root_squash)" >> /etc/exports
systemctl enable --now nfs-server
exportfs -rv
firewall-cmd --permanent --add-service=nfs
firewall-cmd --reload
```
Client mount:
```bash
mount server:/shared /mnt
# persistent
echo "server:/shared /mnt nfs defaults 0 0" >> /etc/fstab
```
Why: `exportfs -rv` refreshes export table; firewall must allow NFS ports.

Verification:
```bash
showmount -e server
df -h /mnt
```
Pitfalls:
- NFS versions (v3 vs v4) and id mapping can affect permissions.

---

### Lab 8 — Boot Target & Services
Objective: change default boot target and disable a service.

Commands & explanation:
```bash
# Set text-mode boot (multi-user)
systemctl set-default multi-user.target

# Set graphical target back
systemctl set-default graphical.target

# Disable a service (example: bluetooth)
systemctl disable --now bluetooth
```
Why: systemd targets define default boot mode. `--now` stops immediately and disables on boot.

Verification:
```bash
systemctl get-default
systemctl is-enabled bluetooth
```
Pitfalls:
- Be careful when changing targets remotely; ensure another access method exists.

---

### Lab 9 — Find & Kill Processes
Objective: locate specific processes and terminate them.

Commands & explanation:
```bash
ps -ef | grep httpd
# identify PID and kill gracefully
kill <pid>
# force kill if necessary
kill -9 <pid>

# Kill by name
pkill httpd
```
Why: `kill` sends signals; start with TERM (default), escalate only if necessary.

Verification:
```bash
pgrep httpd || echo "no httpd running"
```
Pitfalls:
- `kill -9` prevents graceful cleanup. Avoid in production without reason.

---

### Lab 10 — Firewall & Port-Based Access
Objective: allow TCP port 8080 persistently.

Commands & explanation:
```bash
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
firewall-cmd --list-ports
```
Why: `--permanent` writes to runtime configuration on disk; reload applies it immediately.

Verification:
```bash
firewall-cmd --list-ports | grep 8080
```
Pitfalls:
- Zone assignment for the interface matters. Use `--zone=public` if needed.

---

## Labs 11–20 — Extended Exam Tasks

### Lab 11 — Create & Extract Archives (tar/gzip)
Objective: create compressed backups and extract.

Commands & explanation:
```bash
mkdir -p /labdata
cp -a /etc /labdata/etc-copy

# create uncompressed tar
tar -cvf /root/etc-backup.tar -C /labdata etc-copy

# create gzipped tar
tar -czvf /root/etc-backup.tar.gz -C /labdata etc-copy

# extract to /opt
mkdir -p /opt
tar -xzvf /root/etc-backup.tar.gz -C /opt
```
Why: `-C` changes directory to avoid storing full paths; gzip reduces size.

Verification:
```bash
ls -l /opt/etc-copy
```
Pitfalls:
- Preserve permissions with `-p` when needed (e.g., `tar -czvpf`).

---

### Lab 12 — File ACL Permissions
Objective: grant ACL rights without changing ownership.

Commands & explanation:
```bash
mkdir -p /securedata
touch /securedata/secret.txt
setfacl -m u:rahul:rwx /securedata
getfacl /securedata
# To remove all ACLs
setfacl -b /securedata
```
Why: ACLs allow fine-grained permissions beyond the classical owner/group/other model.

Verification:
```bash
getfacl /securedata
su - rahul -c 'ls -ld /securedata && touch /securedata/testfile'  # tests access
```
Pitfalls:
- Some filesystems (NFS exports) may not preserve ACLs depending on server config.

---

### Lab 13 — Configure AutoFS Automount
Objective: automount remote NFS `/shared` under `/misc/shared`.

Commands & explanation:
```bash
yum install -y autofs

# /etc/auto.master content (append)
/misc  /etc/auto.misc

# /etc/auto.misc content
echo 'shared  -rw  server:/shared' > /etc/auto.misc

systemctl enable --now autofs
```
Why: autofs mounts on demand and unmounts after idle timeout, useful for intermittent usage.

Verification:
```bash
ls /misc/shared      # triggers mount
mount | grep /misc
```
Pitfalls:
- Incorrect map syntax or missing autofs service will prevent mounting.

---

### Lab 14 — Add and Extend Swap Space
Objective: create a 2GB swap file and persist.

Commands & explanation:
```bash
fallocate -l 2G /swapfile         # fast allocation (or dd if sparse not allowed)
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# persist
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```
Why: swapfile is safer to add on VMs than partition changes.

Verification:
```bash
free -h
swapon --show
```
Pitfalls:
- Use `dd` if `fallocate` not supported on filesystem; ensure permission 600.

---

### Lab 15 — Create a Custom systemd Service
Objective: create a simple systemd unit and enable it.

Commands & explanation:
```bash
# create executable
cat > /usr/local/bin/custom.sh <<'EOF'
#!/bin/bash
echo "Service started via systemd" >> /tmp/custom.log
sleep 2
EOF
chmod +x /usr/local/bin/custom.sh

# create unit
cat > /etc/systemd/system/custom.service <<'EOF'
[Unit]
Description=Custom Example Service
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/custom.sh

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now custom.service
systemctl status custom.service
```
Why: Type=oneshot runs a task and exits; `WantedBy=multi-user.target` enables autostart.

Verification:
```bash
cat /tmp/custom.log
systemctl is-enabled custom.service
```
Pitfalls:
- Units must be reloaded when created or changed; inspect `journalctl -u custom`.

---

### Lab 16 — Rich Firewall Rules (restrict source subnet)
Objective: accept TCP 9000 only from 192.168.1.0/24.

Commands & explanation:
```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="9000" protocol="tcp" accept'
firewall-cmd --reload
firewall-cmd --list-rich-rules
```
Why: rich rules are expressive and support source-based decisions.

Verification:
```bash
firewall-cmd --list-rich-rules | grep 9000
```
Pitfalls:
- Ensure interface zone assignment matches intended behavior.

---

### Lab 17 — Podman Container Basics
Objective: pull and run nginx container and enable autostart as systemd unit.

Commands & explanation:
```bash
yum install -y podman

# pull image
podman pull docker.io/library/nginx:latest

# run in background mapping host 8080 to container 80
podman run -d --name web -p 8080:80 nginx

# generate systemd unit for autostart
podman generate systemd --name web --files --new
# this creates container-web.service in current dir; move to systemd dir
mv container-web.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now container-web.service
```
Why: `podman generate systemd` integrates container lifecycle with systemd.

Verification:
```bash
podman ps -a
curl -sS http://localhost:8080
systemctl status container-web.service
```
Pitfalls:
- SELinux contexts and ports may require adjustments; `podman` uses user namespaces by default.

---

### Lab 18 — journald Log Rotation & Querying
Objective: query journal by boot and service; ensure logs rotate via systemd/journald config.

Commands & explanation:
```bash
journalctl -b         # current boot
journalctl -b -1      # previous boot
journalctl -u sshd    # logs for sshd
journalctl --since "1 hour ago"

# View journald config
cat /etc/systemd/journald.conf
# Example to limit journal size
sed -n '1,200p' /etc/systemd/journald.conf
```
Why: `journalctl` is the primary tool to examine logs on systemd systems.

Verification:
```bash
journalctl -u sshd --since "10 minutes ago"
```
Pitfalls:
- Ensure persistent journal enabled (`/var/log/journal`) if logs must survive reboots.

---

### Lab 19 — LVM Thin Pool (create thin logical volume)
Objective: create a thin pool and thin LV for space-efficient storage.

Commands & explanation:
```bash
# create thinpool (example 5G)
lvcreate -L 5G -T vgdata/thinpool

# create thin LV with virtual size 10G
lvcreate -V 10G -T vgdata/thinpool -n thinlv

mkfs.xfs /dev/vgdata/thinlv
mkdir -p /thin
mount /dev/vgdata/thinlv /thin
```
Why: thin LVs consume physical space only as used; useful for snapshots and test VMs.

Verification:
```bash
lvs -a -o+pv_size,lv_size,data_percent
df -h /thin
```
Pitfalls:
- Monitor thinpool usage to avoid out-of-space; configure `thin_pool_autoextend_threshold`.

---

### Lab 20 — Find Files Based on Conditions
Objective: locate files >100MB, by owner, modified recently or empty files.

Commands & explanation:
```bash
find / -xdev -type f -size +100M 2>/dev/null

find /home -type f -user rahul 2>/dev/null

find /var/log -type f -mtime -1

find /tmp -type f -empty -delete
```
Why: `-xdev` prevents descending into other filesystems; redirect errors to avoid noisy output.

Verification:
```bash
# inspect a sample
ls -lh $(find / -xdev -type f -size +100M 2>/dev/null | head -n1)
```
Pitfalls:
- Running `find /` as root may list system files; use `-xdev` or restrict path.

---

## Labs 21–40 — Advanced / Administrative Tasks

### Lab 21 — Persistent Command Alias for All Users
Objective: create `ll` alias globally.

Commands & explanation:
```bash
cat > /etc/profile.d/alias.sh <<'EOF'
alias ll='ls -lhrt'
EOF
chmod 0644 /etc/profile.d/alias.sh
# apply to current shell
source /etc/profile.d/alias.sh
```
Verification:
```bash
su - someuser -c 'bash -lc "type ll; ll /"'
```
Pitfalls:
- /etc/profile.d scripts must be shell compatible.

---

### Lab 22 — Hard & Soft Links
Objective: create hard and soft links and show inode behavior.

Commands & explanation:
```bash
echo "hello" > file.txt
ln file.txt hardlink.txt
ln -s file.txt softlink.txt

# verify inode numbers
ls -li file.txt hardlink.txt softlink.txt
```
Explanation: hard links share the same inode; soft links reference path and break if original removed.

---

### Lab 23 — SSH Key‑Based Authentication
Objective: setup key auth.

Commands & explanation (client side):
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ''
ssh-copy-id user@server
```
Server optional hardening:
```bash
# in /etc/ssh/sshd_config
PasswordAuthentication no
ChallengeResponseAuthentication no
# restart
systemctl reload sshd
```
Verification:
```bash
ssh -i ~/.ssh/id_ed25519 user@server 'id'
```
Pitfalls:
- Disabling password auth on remote system without key tested can lock you out.

---

### Lab 24 — Restrict SSH Access to Specific Users
Objective: only allow `rahul` and `admin`.

Commands & explanation:
```bash
# edit /etc/ssh/sshd_config add:
AllowUsers rahul admin

systemctl reload sshd
```
Verification:
```bash
ssh otheruser@host  # should fail
```
Pitfalls:
- Ensure at least one allowed account exists with working auth before reloading.

---

### Lab 25 — Local YUM Repository
Objective: create local repo from DVD or package directory.

Commands & explanation:
```bash
mkdir -p /repo
cp -a /media/* /repo/    # example source; adjust
createrepo /repo
cat > /etc/yum.repos.d/local.repo <<'EOF'
[local]
name=LocalRepo
baseurl=file:///repo
enabled=1
gpgcheck=0
EOF
yum clean all
yum repolist
```
Why: `createrepo` builds metadata; `baseurl` points to local path.

Verification:
```bash
yum --disablerepo="*" --enablerepo=local list available
```
Pitfalls:
- SELinux context for /repo may prevent access — restorecon if needed.

---

### Lab 26 — LVM Snapshot & Restore
Objective: create snapshot and restore using merge.

Commands & explanation:
```bash
lvcreate -L 1G -s -n snap_lvapps /dev/vgdata/lvapps
# make changes to origin, then to rollback:
umount /apps
lvconvert --merge /dev/vgdata/snap_lvapps
# after merge, the origin logical volume will be replaced by snapshot content
mount /apps
```
Why: snapshot captures point-in-time state; merging restores original LV.

Verification:
```bash
lvs -a
```
Pitfalls:
- Snapshot space must be sufficient; monitor `lvdisplay`.

---

### Lab 27 — Password Aging Policy
Objective: set max password age to 45 days.

Commands & explanation:
```bash
chage -M 45 user1
chage -d 0 user1    # force immediate change on next login if needed
chage -l user1
```
Why: `chage` manages user password expiry settings.

Verification:
```bash
chage -l user1
```

---

### Lab 28 — Default umask for Users
Objective: set system default umask 027.

Commands & explanation:
```bash
# Add to /etc/profile (applies to login shells)
echo 'umask 027' >> /etc/profile
# For interactive shells also consider /etc/bashrc or /etc/profile.d/umask.sh
source /etc/profile
```
Verification:
```bash
su - someuser -c 'umask; touch /tmp/testfile; ls -l /tmp/testfile'
```
Pitfalls:
- Non-login shells may not read /etc/profile; ensure consistent placement.

---

### Lab 29 — Automated Backup Script with Cron
Objective: create daily backup of /etc at 01:00.

Commands & explanation:
```bash
cat > /usr/local/bin/backup.sh <<'EOF'
#!/bin/bash
tar -czf /backup/etc-$(date +%F).tar.gz /etc
EOF
chmod +x /usr/local/bin/backup.sh

# schedule (root)
crontab -e
# add:
0 1 * * * /usr/local/bin/backup.sh
```
Verification:
```bash
ls -lh /backup | tail
```
Pitfalls:
- Ensure /backup exists and has sufficient space.

---

### Lab 30 — Find & Terminate Zombie Processes
Objective: locate zombies and remediate parent.

Commands & explanation:
```bash
ps aux | awk '{ if ($8=="Z") print $0 }'

# find PIDs of zombies and identify parent PID (PPID)
ps -eo pid,ppid,stat,cmd | grep ' Z '
# kill or restart parent if safe
kill -TERM <ppid>
```
Why: zombies indicate child terminated but not reaped by parent.

Verification:
```bash
ps aux | grep Z || echo "no zombies"
```
Pitfalls:
- Killing parent may have side effects; prefer graceful restart.

---

### Lab 31 — Rescue Admin with Limited sudo
Objective: provide scoped sudo to allow limited ops.

Commands & explanation:
```bash
useradd rescue
passwd rescue

# sudoers file - only allow restart httpd
cat > /etc/sudoers.d/rescue <<'EOF'
rescue ALL=(ALL) NOPASSWD: /bin/systemctl restart httpd
EOF
chmod 0440 /etc/sudoers.d/rescue
```
Verification:
```bash
su - rescue -c 'sudo systemctl restart httpd'
```
Pitfalls:
- Avoid `ALL` privileges; scope to required commands.

---

### Lab 32 — Static Hostname & /etc/hosts
Objective: set static hostname and host mapping.

Commands & explanation:
```bash
hostnamectl set-hostname appserver.local
echo "192.168.1.20 dbserver" >> /etc/hosts
hostnamectl status
cat /etc/hosts
```
Why: `hostnamectl` writes system config and updates /etc/hostname appropriately.

Verification:
```bash
hostname
getent hosts dbserver
```

---

### Lab 33 — Timezone & NTP Sync
Objective: set timezone and enable NTP sync.

Commands & explanation:
```bash
timedatectl list-timezones
timedatectl set-timezone Asia/Kolkata
timedatectl set-ntp true
timedatectl status
```
Why: ensures correct system time for logs and cron.

Verification:
```bash
date
timedatectl show --property=NTPSynchronized
```

---

### Lab 34 — rsyslog Remote Forwarding
Objective: forward all logs to remote collector.

Commands & explanation:
```bash
# append to /etc/rsyslog.conf or create /etc/rsyslog.d/remote.conf
echo '*.* @@192.168.1.30:514' >> /etc/rsyslog.conf
systemctl restart rsyslog
```
Why: `@@` indicates TCP transport, `@` for UDP.

Verification:
```bash
ss -tunpl | grep 514   # check connectivity
```
Pitfalls:
- Network and receiver must support the protocol and firewall rules.

---

### Lab 35 — tmpfs Ephemeral Mount
Objective: create tmpfs at /mnt/cache sized 512M.

Commands & explanation:
```bash
mkdir -p /mnt/cache
echo 'tmpfs /mnt/cache tmpfs size=512M 0 0' >> /etc/fstab
mount -a
df -h /mnt/cache
```
Why: tmpfs stores files in RAM; useful for caches but ephemeral across reboot.

Verification:
```bash
mount | grep /mnt/cache
```
Pitfalls:
- Do not store important persistent data on tmpfs.

---

### Lab 36 — User Private Group Behavior
Objective: create user and set primary group to devops.

Commands & explanation:
```bash
useradd rahul
# change primary group
groupadd devops
usermod -g devops rahul

# verify
id rahul
```
Why: primary group defines default group ownership of newly created files.

Verification:
```bash
su - rahul -c 'touch /tmp/testfile; ls -l /tmp/testfile'
```

---

### Lab 37 — Persistent Kernel Parameters (sysctl)
Objective: enable IP forwarding persistently.

Commands & explanation:
```bash
sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-custom.conf
sysctl --system
sysctl net.ipv4.ip_forward
```
Why: sysctl settings in /etc/sysctl.d persist across reboots.

Verification:
```bash
cat /proc/sys/net/ipv4/ip_forward
```

---

### Lab 38 — Block/Unblock IP Using firewalld (rich rule)
Objective: drop traffic from 10.10.10.10.

Commands & explanation:
```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.10.10" drop'
firewall-cmd --reload

# To remove
firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" source address="10.10.10.10" drop'
firewall-cmd --reload
```
Verification:
```bash
firewall-cmd --list-rich-rules
```

---

### Lab 39 — autofs Timeout & NFS
Objective: set autofs timeout to 60 seconds.

Commands & explanation:
```bash
echo 'timeout = 60' >> /etc/autofs.conf
systemctl restart autofs
```
Why: autofs unmounts idle mounts after timeout; useful to free resources.

Verification:
```bash
# Access mount and observe autofs logs or /proc/mounts after 60s idle
ls /misc/shared
```

---

### Lab 40 — Basic Kickstart File (Generate & Validate)
Objective: prepare a simple Kickstart file.

Commands & explanation:
```bash
# If anaconda created /root/anaconda-ks.cfg after an install, flatten it:
yum install -y pykickstart   # or system-config-kickstart (packages may vary)
# validate syntax (ksvalidator)
ksvalidator /root/ks.cfg
```
Why: Kickstart automates installations; validation ensures correctness.

Verification:
- Use `ksvalidator` or test in a disposable VM.

---

## General Best Practices & Exam Tips

- Always show exact commands and files edited — examiners inspect timestamps and file contents.
- Use persistent configuration methods (e.g., /etc/sysctl.d, /etc/profile.d, systemd unit files, /etc/fstab, /etc/sudoers.d).
- Avoid destructive `rm` or `mkfs` unless required; back up data first.
- When changing network or SSH configs remotely, keep a fallback (console access or secondary network) to avoid lockout.
- Document each step you performed; helpful for exam notes and production-change records.
- For SELinux changes, prefer `semanage` and `restorecon` for persistent fixes rather than disabling SELinux.

---

## Verification Checklist (common examiner validations)

- Users: `id <user>`, `/etc/group`, `/etc/sudoers.d/*`
- Network: `ip addr`, `ip route`, `/etc/sysconfig/network-scripts/ifcfg-*`
- LVM: `pvs`, `vgs`, `lvs`, `lsblk`, `/etc/fstab`
- Systemd: `systemctl status <unit>`, `systemctl is-enabled <unit>`
- Filesystems: `mount`, `df -h`, `blkid`
- SELinux: `getenforce`, `semanage fcontext -l`, `restorecon -v`
- Firewall: `firewall-cmd --list-all`, `firewall-cmd --list-rich-rules`
- Logs: `journalctl -u <service>`, `journalctl -b`
- Containers: `podman ps -a`, `podman logs <name>`

---

Use these labs iteratively: attempt the task, capture evidence (command output or file contents), make minimal persistent changes, verify, and document. Good luck with your RHCSA/RHCE practice — practice performed well is the best preparation.
