# 📡 NFS — Enterprise Troubleshooting Labs

Realistic, production-style labs for diagnosing and repairing NFS problems on RHEL / CentOS / Rocky / AlmaLinux. These scenarios cover timeouts, stale file handles, boot hangs, permission and SELinux issues, firewall problems, autofs, locking, and performance tuning.

Practice on two VMs (Server + Client) or isolated lab environments. NFS issues can hang clients — always use snapshots and avoid production systems.

---

## Table of Contents

- Prerequisites & environment
- Quick reference commands
- Lab index
- Detailed labs (1–10) with commands, explanation, verification, and exercises
- Troubleshooting checklist & best practices
- Notes on design & documentation

---

## Prerequisites & environment

- Two VMs: nfs-server and nfs-client (same L2 for simplicity).
- On server: nfs-utils, rpcbind, and optionally `nfs-utils-lib`, `nfs-utils` tools installed:
  ```bash
  sudo yum install -y nfs-utils rpcbind
  ```
- On client: `nfs-utils` installed.
- Basic networking connectivity allowed between VMs.
- Use `sudo` for admin commands.

Important: Use NFSv4 when possible (simpler port mapping) but be aware of differences (v3 uses mountd and extra ports).

---

## Quick reference commands

- Exports on server: `exportfs -v`
- Show exported servers: `showmount -e <server>`
- Check services: `systemctl status nfs-server rpcbind nfs-lock nfs-idmapd`
- RPC ports: `rpcinfo -p <server>`
- Mount on client: `mount -t nfs -o vers=4 <server>:/export /mnt`
- Check mounts: `mount | grep nfs`
- Find deleted/open files: `lsof | grep deleted`
- Unmount: `umount /mnt` or `umount -l /mnt` (lazy)
- Check I/O: `iostat -x 2`, `nfsstat -s`, `nfsstat -c`
- SELinux AVC logs: `ausearch -m avc -ts recent`

---

## Lab Index

1. NFS mount hangs — "mount.nfs: connection timed out"
2. Stale NFS file handle — "Stale NFS file handle"
3. Boot stuck due to NFS entry in `/etc/fstab`
4. Permission denied on mount — `access denied by server`
5. NFS mounted but files not visible (wrong export path / bind mounts)
6. SELinux blocking NFS access
7. Firewall blocking NFS ports / rpcbind
8. Autofs failing to mount NFS shares
9. NFS lock problems — `.nfsXXXX` files, rpc.statd issues
10. NFS performance tuning — slow I/O and configuration tuning

---

# Lab 1 — NFS mount hangs / connection timed out

Symptoms
- `mount.nfs: connection timed out`
- Client blocks at mount and hangs or waits a long time

Diagnosis & steps

1. Verify network reachability:
   ```bash
   ping -c3 <nfs-server-ip>
   ```

2. Check NFS service on server:
   ```bash
   sudo systemctl status nfs-server
   sudo exportfs -v
   ```

3. Verify RPC services and ports:
   ```bash
   sudo rpcinfo -p <nfs-server-ip>
   # on client:
   rpcinfo -p <nfs-server-ip>
   ```

4. Test direct port connectivity (NFSv4 uses 2049/TCP):
   ```bash
   # TCP test
   nc -vz <nfs-server-ip> 2049

   # If v3, also check rpcbind/mountd (usually 111 and dynamic ports)
   nc -vz <nfs-server-ip> 111
   ```

5. Restart server services if needed:
   ```bash
   sudo systemctl restart rpcbind nfs-server
   sudo exportfs -rv
   ```

6. On client, try mounting with options that avoid hanging:
   ```bash
   sudo mount -o vers=4,soft,intr,proto=tcp <server>:/shared /mnt
   ```
   Note: `soft` may cause silent failures for writes; use with caution.

Explanation
- Hangs most often mean network/firewall or server not listening. NFSv3 requires extra daemons (mountd, statd) and dynamic ports, which can complicate firewalling.

Verification
- `showmount -e <server>` should list exports.
- `mount | grep /mnt` should show if mount succeeded.

Exercise
- Simulate a server service down (stop nfs-server) and recover it. Practice mounting with `soft` vs `hard` options and observe behaviors.

---

# Lab 2 — Stale NFS File Handle

Symptoms
- `Stale file handle` errors when accessing a file or directory.

Root causes
- File was removed/recreated on the server, server re-exported an altered filesystem (inode change), or server restored FS from backup.

Steps

1. Identify the affected path on client:
   ```bash
   ls -la /mnt/path
   ```

2. Check server export and contents:
   ```bash
   sudo exportfs -v
   sudo ls -la /export/path   # on server
   ```

3. Attempt a normal unmount and remount:
   ```bash
   sudo umount /mnt/path
   sudo mount <server>:/export /mnt
   ```

4. If umount blocks, find processes holding handles:
   ```bash
   sudo lsof +D /mnt/path
   sudo fuser -vm /mnt/path
   # then stop and retry umount or use lazy unmount
   sudo umount -l /mnt/path
   ```

5. If problem persists, restart server NFS services (server-side):
   ```bash
   sudo systemctl restart nfs-server
   sudo exportfs -rv
   ```

Explanation
- Stale handle means client holds an inode that the server no longer recognizes. Remounting clears client-side state.

Pitfalls
- Forceful restarts may impact other clients. Use change-control on production.

Exercise
- Create a file, open it from client, delete/replace it on server, and observe stale handle. Practice recovery steps.

---

# Lab 3 — Boot stuck because of `/etc/fstab` NFS entry

Symptoms
- Boot process delays or drops to emergency shell with messages like “A start job is running for /mnt/data”.

Cause
- Client tries to mount NFS as a required mount before network is up or server unreachable.

Safe `/etc/fstab` pattern
```
<server>:/share  /mnt/data  nfs  defaults,_netdev,bg,nofail,x-systemd.automount  0  0
```

Explanation of options
- `_netdev` — mount after network is up
- `bg` — background retry if mount fails
- `nofail` — allow boot continue if mount fails
- `x-systemd.automount` — let systemd automount on first access, avoiding boot-time waits

Steps
1. Edit `/etc/fstab` on client, add `nofail,_netdev` (and optionally `bg,x-systemd.automount`).
2. Test:
   ```bash
   sudo mount -a
   ```
3. Reboot to confirm.

Exercise
- Configure a non-existent NFS server entry and test that boot continues with `nofail` vs without it.

---

# Lab 4 — Permission denied on NFS mount

Symptoms
- `mount.nfs: access denied by server` or read/write errors after mount

Common causes
- Misconfigured `/etc/exports` on server (host mismatch), `root_squash` restricting root access, or ownership/permissions on filesystem not matching expectations.

Steps

1. Inspect server exports:
   ```bash
   sudo cat /etc/exports
   sudo exportfs -v
   ```

2. Example fix to allow a client (use least-privilege you can):
   ```
   /shared  192.168.1.100(rw,sync,no_subtree_check,no_root_squash)
   ```
   Then:
   ```bash
   sudo exportfs -rv
   ```

3. Verify `showmount -e <server>` from client:
   ```bash
   showmount -e <server>
   ```

4. Check UNIX perms and ownership on server for exported path:
   ```bash
   ls -la /shared
   chown -R nfsnobody:nfsnobody /shared   # depending on export policy
   chmod -R 0755 /shared
   ```

Notes
- `no_root_squash` lets remote root act as root — avoid in production unless necessary.
- Use `anonuid`/`anongid` export options to map root to a safe user.

Exercise
- Export a directory as read-only to a CIDR and verify client cannot create files. Then adjust to rw and test.

---

# Lab 5 — NFS mounted but files not visible

Symptoms
- Mount succeeds, but directory shows empty while server has files.

Root causes
- Wrong exported path, server bind-mount misconfiguration, overlaying or exporting an empty directory.

Steps

1. On server, confirm the exported directory and bind mounts:
   ```bash
   mount | grep /shared
   ls -la /actual_data
   ```

2. If you meant to export `/actual/data` but exported `/shared` (which is empty), correct `/etc/exports` or add a bind:
   ```bash
   sudo mount --bind /actual/data /shared
   echo "/shared  *(rw,sync)" | sudo tee -a /etc/exports
   sudo exportfs -rv
   ```

3. Recheck client mount (remount if needed).

Exercise
- Create a bind mount on server and export it; observe client view.

---

# Lab 6 — SELinux blocking NFS access

Symptoms
- Access fails when SELinux is Enforcing; works when SELinux is Permissive/Disabled.

Diagnosis & steps

1. Check SELinux on server and client:
   ```bash
   getenforce
   sestatus
   ```

2. Inspect AVC denials:
   ```bash
   sudo ausearch -m avc -ts recent
   sudo journalctl -t setroubleshoot
   ```

3. Common server-side booleans and fixes:
   - Allow exporting arbitrary directories (use with caution):
     ```bash
     sudo setsebool -P nfs_export_all_ro 1
     sudo setsebool -P nfs_export_all_rw 1
     ```
   - Correct file contexts for exported data:
     ```bash
     sudo semanage fcontext -a -t public_content_t "/shared(/.*)?"
     sudo restorecon -Rv /shared
     ```

4. Client-side SELinux: If client SELinux denies access to mounted remote nfs home dirs:
   ```bash
   sudo setsebool -P use_nfs_home_dirs 1
   ```

Notes
- Prefer setting correct context and booleans instead of disabling SELinux.

Exercise
- Export a directory with SELinux contexts and practice applying `semanage` + `restorecon` to allow client access.

---

# Lab 7 — Firewall blocking NFS / rpcbind

Symptoms
- Mount or RPC calls blocked; `rpcinfo -p` times out.

Steps

1. On server, check listening ports:
   ```bash
   ss -lntp | egrep '2049|111'
   rpcinfo -p
   ```

2. Open NFS services using firewalld:
   ```bash
   sudo firewall-cmd --permanent --add-service=nfs
   sudo firewall-cmd --permanent --add-service=mountd
   sudo firewall-cmd --permanent --add-service=rpc-bind
   sudo firewall-cmd --reload
   ```

3. If mountd uses dynamic ports, consider fixing ports in `/etc/sysconfig/nfs` (RHEL) or use NFSv4 to minimize ports.

Verification
- From client: `rpcinfo -p <server>` and `nc -vz <server> 2049` should succeed.

Exercise
- Block NFS via firewall and reproduce timeout; reopen ports and verify.

---

# Lab 8 — Autofs not mounting NFS share

Symptoms
- Autofs does not mount on access (hangs or immediate error), while manual mount works.

Steps

1. Install and enable autofs:
   ```bash
   sudo yum install -y autofs
   sudo systemctl enable --now autofs
   ```

2. Configure `/etc/auto.master`:
   ```
   /nfs  /etc/auto.nfs --timeout=60
   ```

3. Create `/etc/auto.nfs` map:
   ```
   data  -rw,soft,intr  server:/shared
   ```

4. Reload and test:
   ```bash
   sudo systemctl reload autofs
   ls /nfs/data  # should trigger automount
   ```

Troubleshooting
- Check `systemctl status autofs` and `/var/log/messages` for failures.
- If the map contains errors, autofs may fail silently.

Exercise
- Set up autofs map for a per-user NFS home and test mount/unmount behaviors.

---

# Lab 9 — NFS File Locking & `.nfs` files

Symptoms
- `.nfsXXXX` temporary files remain, applications hang on locking, database writes fail.

Cause
- Client crashed or lost lease while holding file open; server creates `.nfs` file until client releases.

Steps

1. Ensure lock daemons on server are running:
   ```bash
   sudo systemctl status rpcbind nfs-lock rpc-statd nfs-server
   ```

2. Restart lock services if needed:
   ```bash
   sudo systemctl restart nfs-lock rpcbind
   ```

3. Find `.nfs` files and check who holds them:
   ```bash
   find /shared -name '.nfs*' -ls
   # on client
   lsof | grep '.nfs'
   ```

4. If processes are defunct, kill and restart clients or gracefully stop offending apps to clear locks.

Notes
- NFSv4 introduces stateful locking built into nfsd; v3 relies on rpc.statd.

Exercise
- Simulate a client crash while holding a file open and observe `.nfs` creation and recovery.

---

# Lab 10 — NFS Performance tuning & slow I/O

Symptoms
- Slow reads/writes, high latency, app timeouts.

Investigation & tuning steps

1. Gather baseline:
   ```bash
   nfsstat -s      # server stats
   nfsstat -c      # client stats
   iostat -x 2     # disk throughput
   dstat -nf       # network + NFS metrics (if available)
   ```

2. Check mount options on client:
   ```bash
   mount | grep nfs
   ```
   Typical performance options:
   - `rsize` / `wsize` (e.g., 65536 or up to 1048576 on modern kernels)
   - `noatime,nodiratime` (reduce metadata writes)
   - `async` vs `sync` (server default is `sync`; `async` can improve speed but risk)
   - `actimeo` or `noac` (attribute cache): `noac` reduces caching; may increase latency

3. Try improved options (on client):
   ```bash
   sudo mount -t nfs -o rw,vers=4,proto=tcp,rsize=65536,wsize=65536,noatime <server>:/shared /mnt
   ```

4. Network tuning:
   - Check MTU mismatches, NIC offload issues: `ethtool <iface>`
   - Monitor for errors: `netstat -i` or `ip -s link`

5. Server-side I/O:
   - Check `iostat -x` for disk saturation
   - Use faster storage, tune readahead, or distribute NFS exports across disks

6. Long-running ops:
   - Consider using NFSv4.1 pNFS for large scale deployments or move heavy workloads to block storage.

Exercise
- Run `dd` read/write tests with different `rsize/wsize` to see throughput changes. Observe `nfsstat` before/after.

---

## Troubleshooting checklist & best practices

- Always reproduce in lab before touching production.
- Use `showmount -e`, `rpcinfo -p`, and `exportfs -v` to confirm exports.
- Prefer NFSv4 for simpler port usage (single port 2049).
- Use secure and least-privilege export options; avoid `no_root_squash` unless necessary.
- Use `x-systemd.automount` and `_netdev,nofail` in fstab to prevent boot hangs.
- Keep exports' real filesystem ownership and perms correct; consider `anonuid`/`anongid` mapping.
- Monitor server I/O and network health; NFS problems often root in storage or network.
- Capture AVC denials and fix contexts instead of disabling SELinux.
- Document export changes and use `exportfs -rv` to apply them.
- For firewalls, allow `nfs`, `mountd`, and `rpc-bind` services or lock ports to fixed numbers.

---

## Useful commands summary

- Server: `sudo exportfs -av`, `sudo systemctl status nfs-server rpcbind nfs-lock`
- Client: `showmount -e <server>`, `mount -t nfs -o vers=4 <server>:/share /mnt`
- RPC/ports: `rpcinfo -p <server>`
- Diagnostics: `nfsstat`, `iostat`, `lsof`, `fuser`, `ethtool`
- SELinux: `getenforce`, `ausearch -m avc`, `semanage fcontext`, `restorecon`, `setsebool`
- Firewall: `firewall-cmd --add-service=nfs --permanent && firewall-cmd --reload`

---

## Suggested lab exercises

1. Build NFS server and client; export and mount a directory using fstab and automount. Test recovery when server reboots.
2. Create a stale file-handle scenario: open file from client, delete on server, and observe recovery steps.
3. Simulate firewall blocking `rpcbind` and fix by opening services.
4. Create and recover from lock contention on a database file using `.nfs` artifacts.
5. Tune `rsize/wsize` and network offload settings and measure throughput differences.

---

## Documentation & design notes

- Use clear headings, code blocks, and short checklists for runbooks.
- GitHub renders Markdown fonts; use structure and callouts for emphasis.
- For internal runbooks, include exact commands, examples, and rollback steps.
- Consider creating small helper scripts to reproduce issues safely in labs.

---
- Generate diagrams (SVG) showing NFSv3 vs NFSv4 port flows and processes.

Which option should I do next?
