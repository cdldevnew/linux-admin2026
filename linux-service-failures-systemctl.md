# ⚙️ Systemctl / Service Failure — Hands-On Troubleshooting Labs

Hands‑on scenario labs focused on systemd: services stuck in failed/activating states, units missing, permission and ExecStart errors, dependency ordering, cgroup and resource limit issues, boot-time failures, and recovery patterns you will encounter in production and interviews.

Practice all labs in disposable VMs. Take snapshots before performing destructive or invasive changes.

---

## Contents

- Prerequisites & quick commands
- Lab Index (10 scenarios)
- Detailed labs (steps, commands, explanations, verifications)
- Systemd unit file examples & useful snippets
- Troubleshooting checklist & best practices
- Suggested exercises

---

## Prerequisites & quick commands

Install tools (RHEL/CentOS example):
```bash
sudo yum install -y systemd lvm2 util-linux
```

Common inspection commands
```bash
# service state & unit info
systemctl status myapp.service --no-pager -l
systemctl is-enabled myapp.service
systemctl --failed
systemctl list-units --type=service --state=failed

# logs
journalctl -u myapp.service -b --no-pager -n 200
journalctl -p err -b

# dependency & boot analysis
systemctl list-dependencies myapp.service
systemd-analyze blame
systemd-analyze critical-chain

# unit file & properties
systemctl cat myapp.service
systemctl show myapp.service
systemctl show -p ExecMainPID,ExecMainStatus,ActiveState myapp.service

# cgroups and resource usage
systemd-cgtop
systemctl status --property=CGroup myapp.service

# socket status
ss -lntp
```

Tip: use `-l` to avoid truncated output and `--no-pager` to prevent paging in scripts.

---

## Lab Index

1. Service shows "failed" — analyze with logs  
2. Service runs manually but fails at boot  
3. "Unit not found" — missing/incorrect unit file  
4. Permission denied on ExecStart  
5. Service stuck in "activating" forever  
6. Wrong ExecStart path or script error  
7. Dependency ordering — `After=` / `Requires=` troubleshooting  
8. Restart policy, start limits & auto-recovery configuration  
9. Create custom service & simulate crash/restart behavior  
10. Boot delay due to failed/slow service (start job)

---

# Lab 1 — Service is in "failed" state

Symptoms
- `systemctl status myapp.service` shows `Active: failed`.
- Application stopped, logs show non-zero exit codes.

Steps

1. Inspect journal for the service
```bash
sudo journalctl -u myapp.service -b --no-pager -n 500
```

2. Check unit status and last exit code
```bash
systemctl status myapp.service --no-pager -l
systemctl show -p ExecMainStatus -p ExecMainPID myapp.service
```

3. Search system logs for related errors
```bash
sudo journalctl -p err -b
sudo journalctl -t systemd | tail -n 200
```

4. Restart and observe immediately
```bash
sudo systemctl restart myapp.service
sudo systemctl status myapp.service
sudo journalctl -u myapp.service -f
```

Common causes
- Crash due to uncaught exception, missing dependency, wrong permissions, resource limits, port-in-use, SELinux denial.

Verification
- After fix, `systemctl status` should show `active (running)` and `journalctl -u` should show healthy start messages.

Notes
- If logs are sparse, increase service logging (e.g., app-level stdout/stderr to journal) or inspect application logs.

---

# Lab 2 — Service runs manually but fails at boot

Symptoms
- `systemctl start myapp` works.
- After reboot the service is inactive/failed or never started.

Steps

1. Ensure unit is enabled for boot:
```bash
sudo systemctl enable myapp.service
sudo systemctl is-enabled myapp.service
```

2. Inspect boot ordering and dependencies:
```bash
systemctl list-dependencies myapp.service
systemd-analyze critical-chain myapp.service
```

3. If service needs network, update unit:
```ini
[Unit]
After=network-online.target
Wants=network-online.target
```
Then:
```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp.service
```

4. Check for timing/race with mounts, LVM, or other units. Use `Requires=` when the service must not start unless the dependency is active.

Explanation
- At boot the network, filesystems, or other resources may not yet be available. `After=` controls ordering; `Requires=` ensures the dependency is started (and the dependent will fail if it doesn't start).

Verification
- Reboot and watch:
```bash
sudo reboot
# after boot
systemctl status myapp.service
sudo journalctl -u myapp.service -b
```

---

# Lab 3 — "Unit not found" — missing service file

Symptoms
- `Failed to start foo.service: Unit foo.service not found.`

Steps

1. Verify where unit files live:
```bash
ls -l /etc/systemd/system /usr/lib/systemd/system /lib/systemd/system
```

2. If unit file exists but systemd doesn't know it:
```bash
sudo systemctl daemon-reload
```

3. If you created a new unit file, enable & start:
```bash
sudo systemctl enable --now demo.service
```

Tip
- Use `systemctl cat demo.service` to view the unit file systemd is using. To override shipped units prefer `systemctl edit --full <unit>` or place drop-in in `/etc/systemd/system/<unit>.d/`.

---

# Lab 4 — Permission denied while starting service

Symptoms
- Journal shows: `Permission denied` or `ExecStart` can't execute.

Steps

1. Inspect unit and ExecStart:
```bash
systemctl cat myapp.service
```

2. Check script binary permissions and ownership:
```bash
ls -l /opt/app/start.sh
chmod +x /opt/app/start.sh
chown appuser:appgroup /opt/app/start.sh
```

3. Check SELinux denial (if enabled):
```bash
getenforce
sudo ausearch -m avc -ts recent
sudo journalctl -t setroubleshoot
```
If SELinux is blocking, prefer fixing context:
```bash
sudo semanage fcontext -a -t bin_t "/opt/app(/.*)?"
sudo restorecon -Rv /opt/app
```

4. If service uses `User=` and `Group=`, ensure user exists and has permission for files and directories.

Verification
- After fixes, `sudo systemctl restart myapp` should succeed and `sudo journalctl -u myapp -b` should show successful startup.

Warning
- Avoid `setenforce 0` in production unless debugging; revert to `setenforce 1` and apply permanent fixes.

---

# Lab 5 — Service stuck in "activating" state

Symptoms
- `Active: activating (start)` for long time.

Steps

1. Inspect the unit for `Type=` and timeouts:
```bash
systemctl show -p Type,TimeoutStartUSec myapp.service
```

2. Check dependency list:
```bash
systemctl list-dependencies myapp.service
```

3. View live logs while trying to start:
```bash
sudo journalctl -u myapp.service -f
```

4. If the service creates a socket or waits on another unit, inspect those (socket units or mounts).

5. Increase TimeoutStartSec when the process legitimately takes longer:
```ini
[Service]
TimeoutStartSec=300
```
Then:
```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp.service
```

Troubleshooting
- If `Type=notify`, the service is expected to notify systemd. If it never does, systemd will wait until timeout. For `Type=forking` ensure the parent exits correctly.

Verification
- After fixes, service should reach `Active: active (running)`.

---

# Lab 6 — Wrong ExecStart path or script errors

Symptoms
- `ExecStart=/opt/app/run.sh: No such file or directory`
- `ExecStart` failed with interpreters errors

Steps

1. Verify path:
```bash
ls -l /opt/app/run.sh
```

2. Verify shebang line (`#!/bin/bash`) exists and points to a valid interpreter:
```bash
head -n1 /opt/app/run.sh
which bash
```

3. Ensure script is executable:
```bash
chmod +x /opt/app/run.sh
```

4. Test script manually as the service user:
```bash
sudo -u appuser /opt/app/run.sh
```

5. Redirect stdout/stderr to journal if needed (systemd captures stdout/stderr by default).

Verification
- Once script runs manually, `systemctl start myapp` should succeed.

---

# Lab 7 — Dependency failure (Requires / After)

Symptoms
- `Dependency failed for myapp.service` during boot.

Steps

1. Inspect dependency tree and failed dependency:
```bash
systemctl list-dependencies myapp.service --reverse
systemctl --failed
```

2. Fix the failing dependency (e.g., failed network, mount, or another unit).

3. If the unit must wait for the network to be fully configured:
```ini
[Unit]
Requires=network-online.target
After=network-online.target
```

Then:
```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
```

Notes
- `After=` only changes ordering—not whether the dependency must be up. Use `Requires=` or `Wants=` depending on strictness.

Verification
- `systemd-analyze critical-chain myapp.service` shows pre-requisites and timings.

---

# Lab 8 — Restart policy & auto-recovery configuration

Goal
- Configure service to auto-restart on failures but protect from rapid crash loops.

Unit add-ons:
```ini
[Service]
Restart=on-failure
RestartSec=5s

# limit restart storms
StartLimitBurst=5
StartLimitIntervalSec=60
```

Also useful:
```ini
[Service]
# resource limits
MemoryMax=1G
CPUQuota=50%    # limit CPU to 50% of one CPU (on cgroups v2 semantics depends)
TasksMax=500
```

Commands
```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
systemctl status myapp
```

Check restart history:
```bash
journalctl -u myapp -o cat | grep -i "Starting\|Started\|Failed"
```

Notes
- `Restart=always` restarts even after clean exit—use carefully.
- `StartLimitBurst` and `StartLimitIntervalSec` protect the systemd unit from infinite restarts.

---

# Lab 9 — Create a custom service and simulate crash

Create a crash script:
```bash
sudo tee /usr/local/bin/crash.sh > /dev/null <<'EOF'
#!/bin/bash
echo "$(date): crash start" >> /tmp/crash.log
sleep 2
exit 1
EOF
sudo chmod +x /usr/local/bin/crash.sh
```

Create unit:
```bash
sudo tee /etc/systemd/system/crash.service > /dev/null <<'EOF'
[Unit]
Description=Crash Demo Service

[Service]
Type=simple
ExecStart=/usr/local/bin/crash.sh
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
```

Enable & watch:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now crash.service
sudo systemctl status crash.service --no-pager -l
sudo journalctl -u crash.service -f
```

Observe auto-restart behavior and StartLimit handling.

Cleanup:
```bash
sudo systemctl disable --now crash.service
sudo rm /etc/systemd/system/crash.service /usr/local/bin/crash.sh
sudo systemctl daemon-reload
```

---

# Lab 10 — Boot delay due to failed/slow service ("A start job is running")

Symptoms
- Boot waits with “A start job is running for … 1min 30s” or similar.

Investigation

1. Who/what delays boot?
```bash
systemd-analyze blame | head -n 30
```

2. Find critical chain:
```bash
systemd-analyze critical-chain
```

3. If a service times out during start, reduce timeout or fix service to start faster:
```ini
[Service]
TimeoutStartSec=30
```

4. If service is non-essential and causing delays, mask it:
```bash
sudo systemctl mask problematic.service   # prevents it from being started
sudo systemctl daemon-reload
```

Explanation
- Masking links unit to /dev/null; only use when you intend to permanently disable a unit or during troubleshooting.

Verification
- After change, reboot and ensure boot time improved:
```bash
systemd-analyze blame
systemctl --failed
```

---

## Systemd Unit File Examples & Tips

Minimal unit:
```ini
[Unit]
Description=My App
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/start.sh
Restart=on-failure
RestartSec=5
Environment=ENV=production
EnvironmentFile=/etc/myapp/env
# Limits
MemoryMax=512M
CPUQuota=50%

[Install]
WantedBy=multi-user.target
```

Drop-in overrides (recommended for ops):
```bash
sudo systemctl edit --full myapp.service
# or create /etc/systemd/system/myapp.service.d/override.conf
```

Useful options
- `ExecStartPre=` / `ExecStartPost=` — run pre/post commands
- `KillMode=` — control killing behavior of processes
- `LimitNOFILE=` — file-descriptor limits
- `PrivateTmp=yes` — isolates /tmp
- `ProtectSystem=full` — read-only /usr and /boot
- `AmbientCapabilities=` / `CapabilityBoundingSet=` — control capabilities

Transient one-off units
```bash
sudo systemd-run --unit=oneshot --description="temp job" /bin/echo hello
```

---

## Troubleshooting Checklist & Best Practices

- Always inspect `journalctl -u <unit> -b` first.
- Check `systemctl --failed` to find broken units across the system.
- Prefer `systemctl edit` to create drop-ins rather than editing vendor unit files.
- Use `systemctl daemon-reload` after changing unit files.
- Use `systemd-analyze blame` and `critical-chain` for boot debugging.
- Check SELinux (ausearch, setroubleshoot) when permission errors are obscure.
- Do not disable SELinux as the first fix — fix contexts or booleans instead.
- Use `systemd-cgtop` and `systemctl show` for cgroup and resource limit debugging.
- Avoid `kill -9` unless graceful stop fails; allow systemd to stop units (`systemctl stop`).
- For production, use health checks, proper Restart/StartLimit settings, and resource limits to prevent noisy neighbors.

---

## Suggested Exercises

1. Build a sample web app service that depends on network-online.target and test boot-time behavior.
2. Create a service that intentionally fails 3 times and observe StartLimit behavior, then tune StartLimit.
3. Simulate SELinux denial on a service and fix with `semanage fcontext` and `restorecon`.
4. Configure `MemoryMax` for a misbehaving service and observe OOM handling by systemd.
5. Create a socket-activated service (socket unit) and observe how systemd starts the service on first connection.

---

## Design & Presentation Suggestions

- Use clear headings and code blocks (as in this README).
- For published docs: use MkDocs or GitHub Pages to control fonts and layout; on GitHub the font is controlled by their stylesheet.
- Keep unit templates in `/examples/` and one file per lab for quick copy/paste.
- Add small diagrams when explaining ordering (e.g., Unit A → Unit B arrows).

---
