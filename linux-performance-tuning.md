# 🚀 System Performance Tuning — Hands‑On Labs (Detailed Commands & Explanations)

These labs teach practical, production‑style performance tuning for Linux systems: CPU, memory, I/O, network, kernel parameters, tuned profiles, and incident RCA. Each lab contains: symptoms, what to collect first, commands with explanations, safe tuning changes, how to persist them, and verification steps.

Important safety notes
- 🧪 Practice in lab VMs or isolated test environments.
- ⚠️ Avoid running heavy destructive load tests against production systems without authorization.
- Always collect full baseline information before making changes.
- Keep an out‑of‑band console when modifying network/kernel/boot configuration.

---

## 📌 Lab Index

1. Baseline system performance check
2. CPU bottleneck analysis & tuning
3. Memory pressure, swap & cache tuning
4. Disk I/O bottleneck analysis & testing
5. Filesystem performance tuning
6. Network performance tuning (sysctl & testing)
7. tuned profiles — choose & verify
8. Increase ulimits / file descriptors safely
9. Identify top resource‑consuming processes & threads
10. Build a quick RCA checklist for performance incidents

Each lab below gives commands, what each shows, examples, persistence, and verification.

---

## Lab 1 — Establish a Performance Baseline

Goal
- Capture a reproducible baseline (before/after) so you can measure impact of tuning.

Essential captures (run as root or with sudo)
```bash
# short summary
uptime
hostnamectl

# CPU / load
top -b -n1 | head -n20
vmstat 1 5

# Memory
free -h
cat /proc/meminfo

# Disk
df -h
lsblk -f
iostat -x 2 5      # install sysstat package if missing
blktrace -a read -a write -d /dev/sdX -o - | blkparse -i -   # optional deep trace

# Network
ss -tunapl
ip -s link
sar -n DEV 1 5    # install sysstat

# Processes
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head -n 20

# Kernel & tuning state
sysctl -a | egrep 'vm.|net.|fs.|kernel.threads_max'   # focus on interesting subsystems
tuned-adm active || echo "tuned not active"
```

What each capture tells you
- uptime / load averages: brief view of system pressure over 1/5/15 minutes.
- top / ps: which processes are consuming CPU/memory.
- vmstat: CPU, IO, swap activity, run queue length.
- iostat: per‑device utilization, throughput and latencies.
- sar -n DEV: per‑interface packets, errors, dropped packets.

Save baseline to a timestamped directory:
```bash
mkdir -p /var/log/perf-baseline/$(date -u +"%Y%m%dT%H%MZ")
# run commands above and redirect output into that directory for later comparison
```

---

## Lab 2 — CPU Bottleneck Analysis & Tuning

Symptoms
- High load average, CPU% near 100, many runnable tasks, system sluggish for compute tasks.

Collect & diagnose
```bash
# Run in interactive sampling mode for 30 seconds
top -b -n3 -d 10

# Per-CPU utilization
mpstat -P ALL 1 5           # from sysstat

# Runnable queue & interrupts
vmstat 1 5
cat /proc/interrupts

# Per-thread CPU usage (useful for multi-threaded apps)
ps -eLo pid,tid,pcpu,comm --sort=-pcpu | head -n 30

# Locking / contention (for Java/Python apps, use jstack or py-spy respectively)
perf top                    # requires perf tools
```

Interpretation notes
- run queue > number of cores consistently → CPU scheduling pressure.
- high %sys (system) indicates kernel/IRQ overhead; high %usr indicates userland CPU work.
- many softirqs or interrupts on one CPU → interrupt affinity tuning recommended.

Safe tuning options (lab)
1) CPU frequency governor
```bash
# Check governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Use cpupower (install kernel-tools/cpupower)
cpupower frequency-info
cpupower frequency-set -g performance    # performance vs powersave
```
Explanation: performance prevents frequency down‑scaling (more power, less latency).

2) Set CPU affinity for processes (taskset)
```bash
# Launch or reassign a process to specific cores
taskset -c 0-3 myapp          # pin to CPUs 0..3
taskset -cp 0-3 <pid>         # assign existing pid
```
Explanation: reduces scheduler migrations and cache churn for CPU‑bound apps.

3) Use cgroups / systemd for CPU quota (safe controlled throttle)
```bash
# systemd override for unit (example myapp.service)
systemctl edit myapp.service   # add:
# [Service]
# CPUQuota=75%
# Save and `systemctl daemon-reload && systemctl restart myapp.service`
```

Verification
```bash
mpstat -P ALL 1 5
top
ps -eo pid,comm,psr,pcpu | head
```

When to escalate
- If CPU% in kernel mode is very high due to a driver/interrupt, involve platform/network team.

---

## Lab 3 — Memory Pressure, Swap & Cache Tuning

Symptoms
- High swap usage, OOM events, slow response despite free memory report.

Collect & diagnose
```bash
# Memory overall
free -m
vmstat 1 5

# Kernel OOM logs
journalctl -k | egrep -i 'oom|out of memory' -n

# See swap usage by process
for pid in $(ls /proc | egrep '^[0-9]+$'); do awk '/VmSwap/ {print FILENAME, $0}' /proc/$pid/status 2>/dev/null | sed 's#/proc/##; s#/status##'; done | sort -k3 -n -r | head -n 20

# slab caches & page cache
slabtop -s c
echo 1 > /proc/sys/vm/drop_caches   # DO NOT do on production regularly; lab only
```

Key kernel knobs
- vm.swappiness — determines kernel's tendency to swap
- vm.vfs_cache_pressure — pressure to reclaim inode/dentry caches
- vm.overcommit_memory / vm.overcommit_ratio — controls memory overcommit behavior

Safe tuning examples
```bash
# Inspect
sysctl vm.swappiness
sysctl vm.vfs_cache_pressure

# Set temporarily
sysctl -w vm.swappiness=10
sysctl -w vm.vfs_cache_pressure=50

# Persist
echo "vm.swappiness=10" >> /etc/sysctl.d/99-perf.conf
echo "vm.vfs_cache_pressure=50" >> /etc/sysctl.d/99-perf.conf
sysctl --system
```
Explanation: Lower swappiness reduces swapping at the cost of keeping more pages in RAM.

Add swap (lab safe)
```bash
# 4G swap file example
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

HugePages (for DBs)
```bash
# reserve 2G of 2MB pages (example)
sysctl -w vm.nr_hugepages=1024
echo "vm.nr_hugepages=1024" >> /etc/sysctl.d/99-hugepages.conf
```
Note: HugePages size and count depends on workload; consult DB vendor docs.

Verification
```bash
free -h
swapon --show
cat /proc/meminfo | egrep 'Swap|Free|Buffers|Cached|Active|Inactive'
```

When to avoid changes
- Decreasing swappiness may increase RAM usage for caches; ensure you have headroom.

---

## Lab 4 — Disk I/O Bottleneck Troubleshooting & Testing

Symptoms
- High iowait %, slow writes/reads, database slowness.

Collect & diagnose
```bash
# Core checks
iostat -x 2 5
# Look at %util (device busy), await (avg latency), svctm (service time), r/s, w/s

# Identify processes doing IO
iotop -aoPa  # interactive; use -b for batch output

# Disk latency percentiles with ioping
ioping -c 10 -s 4k /dev/sdX

# Filesystem-level profiling
fio --name=randread --ioengine=libaio --iodepth=32 --bs=4k --direct=1 --size=1G --numjobs=4 --runtime=60 --time_based --rw=randread --filename=/tmp/testfile
```

Explain fio example
- bs=4k, iodepth=32, direct=1 (bypass cache), numjobs=4 simulate random read workload similar to DB.

Safe tuning & optimizations
1) Scheduler:
```bash
# Check and set elevator for devices
cat /sys/block/sdX/queue/scheduler
echo mq-deadline > /sys/block/sdX/queue/scheduler   # example change
```
2) Filesystem mount options
- noatime reduces metadata writes.
3) Use RAID/striping, appropriate queue depth, or NVMe with correct driver and firmware.

Persistence
- Mount options in /etc/fstab
- Elevator settings via udev rules for persistence:
```bash
# /etc/udev/rules.d/60-ioschedulers.rules
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/scheduler}="mq-deadline"
```

Verification
```bash
iostat -x 2 5
fio results: look for IOPS and latencies (latency should be within SLO)
```

When to escalate
- If underlying device (%util near 100) is saturated, request storage team to check backend (SAN / hypervisor).

---

## Lab 5 — Filesystem Performance Tuning

Symptoms
- High metadata overhead, frequent fsync waits, poor small-file performance.

Checks & tips
```bash
# Check mount options
mount | grep '/data' || findmnt /data

# Add noatime (decreases writes on most workloads)
# Example fstab entry:
# /dev/mapper/vg-data /data xfs defaults,noatime,attr2,inode64 0 0

# For XFS tune:
xfs_info /data
# When creating filesystems or tuning online:
xfs_admin -l /dev/xxx   # inspect metadata
```

Application-specific guidance
- Databases: use direct I/O or O_DIRECT where supported, set appropriate fdatasync/fsync behavior per DB docs.
- Web content: use XFS or ext4 with noatime; use tmpfs for ephemeral caches.

Rebalance & defrag (XFS)
```bash
xfs_repair -n /dev/xxx     # dry run repair if corrupt
xfs_fsr /data               # defragment (careful in production)
```

Verification
- Compare small file create/delete latency with find/create loops or fio with small block sizes.

---

## Lab 6 — Network Performance Tuning

Symptoms
- High packet drops, high retransmits, poor TCP throughput under concurrency.

Collect & diagnose
```bash
# Interface stats & counters
ip -s link show eth0
ethtool -S eth0   # vendor-specific counters
ss -s            # summary socket stats
ss -tnm state established '( sport = :http or dport = :http )'

# TCP retransmits & errors
ss -t -i
netstat -s | egrep -i 'tcp|ip'
```

Key kernel knobs (examples)
```bash
# Increase socket backlog and queue sizes (temporary)
sysctl -w net.core.somaxconn=4096
sysctl -w net.core.netdev_max_backlog=5000
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
sysctl -w net.ipv4.tcp_congestion_control=cubic   # or bbr if kernel supports
```

Persist knobs
- Add to `/etc/sysctl.d/99-network-tuning.conf`:
```text
net.core.somaxconn = 4096
net.core.netdev_max_backlog = 5000
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
```
Apply: `sysctl --system`

Network testing tools
- iperf3 for throughput
```bash
# On server
iperf3 -s
# On client
iperf3 -c server -P 8 -t 60    # 8 parallel streams
```
- tcpreplay / tcptrace for reproducing traces
- netperf for small transaction workloads

Verification
- Use iperf3 results and ss to observe congestion, retransmits, and socket buffers.
- `ss -tin` shows TCP info incl. retransmits.

When to tune NIC:
- Use ethtool for offload settings: `ethtool -K eth0 tso off gso off gro off` when offloads cause issues (test first).

---

## Lab 7 — tuned Profiles & Selecting Appropriate Profile

Goal
- Use tuned to apply common workload presets quickly.

Install & use
```bash
# Install tuned
yum install tuned -y   # RHEL/CentOS
apt install tuned -y   # Debian/Ubuntu

# Start & list
systemctl enable --now tuned
tuned-adm list
tuned-adm profile throughput-performance
tuned-adm active
```

Profile mapping (examples)
- web servers: balanced or throughput-performance
- DB servers: latency-performance or throughput-performance (test)
- virtualization host: virtual-host
- guests: virtual-guest

Check what a profile changes:
```bash
# show profile contents
tuned-adm profile throughput-performance
ls /etc/tuned/throughput-performance/
cat /etc/tuned/throughput-performance/tuned.conf
```

Verification
- Compare baseline metrics (cpu, io, net) before/after applying profile.

---

## Lab 8 — Increase File Descriptor Limits (ulimits) Safely

Symptoms
- “Too many open files” errors, app unable to open sockets.

Check current limits
```bash
ulimit -n           # per shell
cat /proc/<pid>/limits  # for running process
```

System-wide and per-user configuration
- Edit `/etc/security/limits.d/90-app.conf`:
```text
# file: /etc/security/limits.d/90-app.conf
appuser soft nofile 100000
appuser hard nofile 200000
```
- For systemd services, set in unit file:
```ini
# /etc/systemd/system/myapp.service.d/override.conf
[Service]
LimitNOFILE=200000
# Then reload and restart:
systemctl daemon-reload
systemctl restart myapp
```

Verify
```bash
# After restart, check process limits
cat /proc/$(pgrep -f myapp)/limits | grep "Max open files"
```

Notes
- Some kernel-wide file descriptor limits exist; check `fs.file-max`:
```bash
sysctl fs.file-max
sysctl -w fs.file-max=2000000
echo "fs.file-max=2000000" >> /etc/sysctl.d/99-fs.conf
sysctl --system
```

---

## Lab 9 — Identify Top Resource‑Consuming Processes & Threads

Useful commands
```bash
# Top processes by CPU
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head -n 20

# Top by memory
ps -eo pid,cmd,%mem --sort=-%mem | head -n 20

# Show per-thread CPU inside a process (replace <pid>)
top -H -p <pid>

# IO heavy processes
iotop -aoPa -d 5 | head -n 30

# Historic per-process samples (pidstat)
pidstat -ru 1 5
```

Interpretation
- Use per-thread view to find a single thread hogging CPU in a multi-thread app.
- Check process children and I/O patterns to determine root cause.

---

## Lab 10 — Performance Incident RCA Checklist (quick runbook)

When a performance incident occurs, run and save the following immediately:

1) Host and time
```bash
date; hostname; uptime
```
2) Capture top snapshot
```bash
top -b -n1 > /tmp/top.$(date +%s).log
```
3) Memory and swap
```bash
free -h > /tmp/free.log
vmstat 1 5 > /tmp/vmstat.log
```
4) Disk stats & top IO
```bash
iostat -x 2 5 > /tmp/iostat.log
iotop -b -o -P -n 5 > /tmp/iotop.log
```
5) Network
```bash
ss -tunapl > /tmp/ss.log
sar -n DEV 1 5 > /tmp/sar_net.log
```
6) Process list & open files
```bash
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head -n 50 > /tmp/ps.log
lsof -nP -p $(pgrep -d, -f myapp) > /tmp/lsof_myapp.log
```
7) Kernel and app logs
```bash
journalctl -b -n 500 > /tmp/journal.log
cp /var/log/myapp/*.log /tmp/  # just copy recent logs
```
8) sysctl & tuned
```bash
sysctl -a > /tmp/sysctl-all.log
tuned-adm active > /tmp/tuned.log
```

Store these logs in a single archive and timestamp them:
```bash
tar czf /tmp/perf-incident-$(date -u +"%Y%m%dT%H%MZ").tgz /tmp/*.log
```

RCA analysis steps (concise)
- Compare current metrics to baseline.
- Correlate the start time of the incident with recent config changes, deployments, or cron jobs.
- Check kernel OOM or disk errors in `dmesg`.
- Check storage and network device counters for errors/drops.

---

## Tools & Commands Quick Cheat Sheet

- CPU: top, htop, mpstat, perf, taskset, cpupower
- Memory: free, vmstat, slabtop, /proc/meminfo, ps, smem
- Disk: iostat, iotop, fio, ioping, blktrace, lsblk, df
- Filesystem: tune2fs, xfs_info, xfs_db, xfs_fsr
- Network: ss, ip -s link, ethtool, iperf3, tcpdump, tc, sysctl (net.*)
- Kernel: sysctl, dmesg, journalctl -k
- System tuning: tuned-adm, systemd resource controls, udev rules
- Profiling: perf, bpftrace, strace, eBPF tools (bcc, bpftrace)

---

## Example: Quick Disk Stress Test (lab only)
```bash
# Create a 4GB test file and run random mixed rw for 2 minutes
fio --name=job1 --ioengine=libaio --iodepth=16 --rw=randrw --rwmixread=70 \
--bs=8k --direct=1 --size=4G --numjobs=2 --runtime=120 --time_based --group_reporting \
--filename=/tmp/fio-testfile
```
Interpret fio output: look at IOPS and latencies (avg/95th/99th) to judge suitability for DB or transactional workloads.

---

## Practical Tuning Examples & Rationale

1) Web server (many concurrent short requests)
- Use `tuned-adm profile balanced` or `throughput-performance`
- Increase somaxconn & tcp backlog; increase file descriptors for worker user
- Use noatime on static content filesystems

2) Database server (low latency, high IOPS)
- Use `tuned-adm profile latency-performance` or custom tuned profile
- Reserve HugePages (if supported)
- Lower swappiness, tune vm.dirty_ratio and vm.dirty_background_ratio to control writeback behavior
- Optimize io scheduler and queue depth per storage vendor recommendations

3) Virtual hosts (hypervisors)
- Use `virtual-host` tuned profile
- Ensure CPU isolation and hugepages if needed for VMs, and avoid CPU frequency scaling that hurts latency.

---

## Final Notes & Best Practices

- Make one change at a time and measure. Use the baseline comparisons to validate improvements.
- Prefer non‑persistent, reversible changes for initial testing (sysctl -w, cpupower set, taskset).
- Persist only after validating across realistic workload patterns.
- Document all tuning changes, reason, expected behavior, and rollback plan.
- In multi‑tier systems, collaborate: database, network, storage and application teams all play a role in performance tuning.

---

You're now equipped with practical commands, safe tuning patterns, testing examples, and an incident playbook to analyze and tune Linux system performance in real‑world, production‑like scenarios. Happy tuning—and remember to always test in a controlled environment first!
