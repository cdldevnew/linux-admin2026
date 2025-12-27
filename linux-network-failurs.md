# 🌐 Networking Failure — Hands‑On Troubleshooting Labs (Detailed Commands & Explanations)

These labs simulate real production networking failures on Linux servers and provide step‑by‑step commands, what each command tells you, verification checks, persistence changes, and common root causes. Use in VM / lab environments only. Do not run these against production systems without change control.

High‑level checklist for any network issue:
- Confirm the problem scope: single host, subnet, data center, or internet.
- Reproduce the failure from multiple vantage points (localhost, another host on same subnet, remote host).
- Collect logs and state before changing anything.
- Make only incremental changes and verify after each step.

---

## Lab 1 — Interface is DOWN / No connectivity

Symptoms
- `ip addr` shows interface state DOWN
- Cannot ping gateway, no ARP entry

What to gather first
```bash
ip addr show dev eth0
ip link show eth0
ethtool eth0            # link negotiation, speed, duplex
dmesg | grep -i eth0    # kernel messages for link events
journalctl -u NetworkManager -r --no-pager  # if using NetworkManager
```

Common causes
- Admin or boot config set interface down
- Cable or switch port down
- Broken driver or hardware
- Misconfigured NetworkManager / ifcfg

Immediate actions
1) Try to bring the interface up
- NetworkManager (modern RHEL/CentOS/Fedora):
```bash
nmcli device status
nmcli device connect eth0
nmcli device show eth0
```

- sysv / ifup (older distros):
```bash
ifup eth0         # may use /etc/sysconfig/network-scripts/ifcfg-eth0
ip link set dev eth0 up
```

What these do:
- `nmcli device connect` asks NetworkManager to activate device.
- `ip link set up` directly sets link state; this might conflict with NetworkManager.

2) If link shows no carrier
```bash
ethtool eth0
# key fields: Link detected: yes/no, Speed, Duplex, Auto-negotiation
```
- If `Link detected: no` — check cable, switch port, SFP, or virtual NIC settings.

Making persistent fixes
- RHEL/CentOS ifcfg file:
```ini
# /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
ONBOOT=yes
BOOTPROTO=dhcp   # or none/static
# for static:
# IPADDR=192.0.2.10
# NETMASK=255.255.255.0
# GATEWAY=192.0.2.1
```
- Restart:
```bash
systemctl restart network    # or NetworkManager
```

Verification
```bash
ip addr show dev eth0
ip route show
ping -c3 <gateway-ip>
arp -n
```

If still failing: capture packets and logs
```bash
tcpdump -i eth0 -n -vv
journalctl -k | tail -n 200
```

---

## Lab 2 — LAN OK, No Internet (Gateway/Routing Issue)

Symptoms
- Can ping gateway, cannot reach external IPs (e.g., 8.8.8.8)

What to gather
```bash
ip route show
ip -4 rule show
curl -sI https://8.8.8.8 || ping -c3 8.8.8.8
traceroute -n 8.8.8.8
```

Common causes
- Missing or wrong default route
- Upstream gateway has no internet or is misconfigured (NAT missing)
- Firewall/NAT blocking outbound

Fix: Add default route
```bash
# temporary
ip route add default via 192.0.2.1 dev eth0

# verify
ip route show | grep default
ping -c3 8.8.8.8
```

Persist default route
- RHEL/CentOS ifcfg:
```ini
GATEWAY=192.0.2.1
```
- Debian/Ubuntu (if using /etc/network/interfaces):
```ini
gateway 192.0.2.1
```
- Netplan (Ubuntu >=18.04)
```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.0.2.10/24]
      gateway4: 192.0.2.1
```
Apply:
```bash
netplan apply
```

Check NAT/masquerade if server is a gateway
```bash
iptables -t nat -L -n -v
# or nftables
nft list ruleset
```

If gateway reachable but internet not:
- Try pinging 8.8.8.8 FROM the gateway to confirm upstream
- Check provider outage

Verification
```bash
ip route get 1.1.1.1
curl -sI https://example.com
```

---

## Lab 3 — DNS Not Resolving (IPs OK)

Symptoms
- `ping 8.8.8.8` works, but `ping google.com` fails with `unknown host`

Collect system configuration
```bash
cat /etc/resolv.conf
systemd-resolve --status        # if using systemd-resolved
nmcli device show eth0 | grep DNS
```

Common causes
- /etc/resolv.conf overwritten or mispointed
- systemd-resolved misconfigured or not working
- DNS server unreachable or firewall blocking UDP/53

Quick fix (temporary)
```bash
# Overwrite /etc/resolv.conf (careful with systemd-resolved)
echo -e "nameserver 8.8.8.8\nnameserver 1.1.1.1" > /etc/resolv.conf
# or use nmcli to set DNS for connection:
nmcli con modify "System eth0" ipv4.dns "8.8.8.8 1.1.1.1"
nmcli con up "System eth0"
```

Test DNS
```bash
dig +short google.com @8.8.8.8
dig +trace google.com
nslookup google.com
```

If systemd-resolved in use
```bash
# Check status
systemctl status systemd-resolved
resolvectl status
# To configure
resolvectl dns eth0 8.8.8.8
resolvectl domain eth0 example.com
```

Firewall check for DNS
```bash
iptables -L -n | grep 53
firewall-cmd --list-services
firewall-cmd --list-ports
```

Persist configuration
- Use NetworkManager or netplan to set DNS so /etc/resolv.conf is managed correctly.

---

## Lab 4 — Duplicate IP Conflict

Symptoms
- Intermittent disconnects, ARP flaps, system logs show duplicate address messages

Detecting duplicate IP
```bash
# show ARP table
ip neigh show

# observe ARP / DHCP collisions
tcpdump -i eth0 arp -n -vv
# or monitor for gratuitous ARP announcements
arping -I eth0 -c 5 192.0.2.10
```

Look for conflicting MACs:
```bash
arp -an | grep 192.0.2.10
```

If using DHCP
- Check DHCP server leases for duplicate assignment
- Verify static IPs do not overlap DHCP pool

Mitigation
- Change server IP to a known-good unused address
```bash
nmcli con modify "System eth0" ipv4.addresses 192.0.2.50/24
nmcli con up "System eth0"
```
- Or update DHCP reservation for the MAC

Monitoring & alerts
- Install arpwatch or use switch/mac table inspections
```bash
sudo apt install arpwatch
# start and check /var/log/syslog for reports
```

---

## Lab 5 — Ping works but Application Unreachable (Layer‑7 issue)

Symptoms
- Server responds to ICMP, but the service port (e.g., 8080) is unreachable or resets

Gather evidence
```bash
# Check if process listening
ss -tulpen | grep :8080
lsof -i :8080

# Check service status and logs
systemctl status myapp
journalctl -u myapp -n 200

# From remote host test TCP
nc -v -w 3 192.0.2.10 8080
curl -v http://192.0.2.10:8080/health
```

Common causes
- Service crashed or stuck (zombie, high CPU, blocked threads)
- Service bound to localhost (127.0.0.1) instead of 0.0.0.0
- Local firewall blocking traffic
- SELinux denying access to network port or socket
- Proxy / load balancer misconfiguration

Fixes/Checks
1) Ensure service is listening on desired address:
```bash
ss -ltnp | grep 8080
# Output shows LISTEN 0 128 127.0.0.1:8080 0.0.0.0:* users:(("myapp",pid=1234,...))
```
- If bound to 127.0.0.1, update the app config to listen on 0.0.0.0 or specific interface.

2) Firewall:
```bash
firewall-cmd --list-all
firewall-cmd --query-port=8080/tcp
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
# iptables alternative:
iptables -L INPUT -n | grep 8080
```

3) SELinux:
```bash
getenforce
ausearch -m avc -ts recent | tail -n 50
# If denial shown for port:
semanage port -a -t http_port_t -p tcp 8080
# If semanage not available:
yum install policycoreutils-python-utils  # RHEL/CentOS
apt install policycoreutils-python-utils -y   # Debian/Ubuntu
```

4) Logs / stack traces
- Check app logs; consider transient resource exhaustion (ulimits, FD leaks)

5) If behind a load balancer or proxy:
- Verify health checks
- Check proxy config (Nginx, HAProxy) and ensure upstream is reachable from proxy host.

Verification
```bash
curl -v http://127.0.0.1:8080/    # local
curl -v http://192.0.2.10:8080/ # remote
ss -tnp | grep 8080
```

---

## Lab 6 — Wrong Route Table / Asymmetric Routing

Symptoms
- Outbound packets leave via one interface but return path goes via another router => replies dropped
- Application flows fail (TCP) though pings might work

Collect state
```bash
ip route show
ip rule show
ip route get <peer-ip> from <local-ip>
# Example:
ip route get 198.51.100.12 from 192.0.2.10
```

Diagnose with packet capture
```bash
# On host suspected of asymmetric path
tcpdump -i eth0 host 198.51.100.12 -w /tmp/flow.pcap
# Also capture on upstream routers / firewalls when possible
```

Common causes
- Multiple interfaces with different default gateways
- Policy routing (ip rule) or VRF tables misconfigured
- Incorrect SNAT / NAT on one path only

Fixes
1) Ensure default route consistent:
```bash
ip route add default via 192.0.2.1 dev eth0
# or delete wrong route:
ip route del default via 10.0.0.1
```

2) For multi‑homed hosts, use source based policy routing
```bash
# Create table in /etc/iproute2/rt_tables: add "200 to-providerB"
ip rule add from 192.0.2.10 table 200
ip route add default via 10.0.0.1 dev eth1 table 200
```

3) On routers ensure return path exists; check routing table and asymmetry with traceroute from both ends:
```bash
traceroute -n <client>
traceroute -n <server>
```

Verification
```bash
ip route get 8.8.8.8 from 192.0.2.10
# test full application flow
curl --interface 192.0.2.10 http://198.51.100.12/health
```

---

## Lab 7 — MTU / Fragmentation Problems

Symptoms
- Small transfers or SSH fine; large HTTP downloads, TLS handshakes, or VPN tunneled traffic hang or fail
- Path MTU discovery failing or ICMP blocked

Tests
```bash
# Test large pings with DF (do not fragment)
# reduce size until success; 28 bytes overhead for IP+ICMP
ping -M do -s 1472 198.51.100.12   # 1500 - 28 = 1472
# If it fails, reduce size (e.g., 1400)
```

Diagnose TCP MSS and PMTUD issues
```bash
# Observe packets with tcpdump, see MSS option in handshake
tcpdump -i eth0 -s 0 -w /tmp/tcp.pcap tcp and host 198.51.100.12
# Or watch for ICMP fragmentation needed:
tcpdump -n icmp and 'icmp[0] == 3 and icmp[1] == 4'   # fragmentation needed
```

Temporary mitigation
```bash
# Lower MTU on interface
ip link set dev eth0 mtu 1400
# Or adjust TCP MSS on the server (iptables mangle)
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

Persistent setting
- ifcfg:
```ini
MTU=1400
```
- netplan:
```yaml
mtu: 1400
```

Verification
```bash
ping -M do -s 1372 198.51.100.12   # for mtu 1400
curl -v --limit-rate 1M http://198.51.100.12/largefile
```

Root causes
- VPN/GRE/IPIP tunnels reduce effective MTU.
- Firewalls blocking ICMP "Fragmentation Needed" notifications.
- Misconfigured intermediate device (jumbo frames mismatch).

---

## Lab 8 — Firewalld / iptables / nftables Blocking Traffic

Symptoms
- Service running and listening, but remote connection attempts time out or are rejected.

Gather facts
```bash
ss -ltnp | grep :8080
firewall-cmd --state
firewall-cmd --list-all
firewall-cmd --zone=public --list-ports
iptables -L -n -v
nft list ruleset    # if nftables used
```

Typical commands to open a port (firewalld):
```bash
# Temporary (until reload)
firewall-cmd --add-port=8080/tcp
# Permanent
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
```

Check zone assignment for interface:
```bash
firewall-cmd --get-active-zones
firewall-cmd --zone=public --list-interfaces
```

If using iptables legacy:
```bash
iptables -A INPUT -p tcp --dport 8080 -m conntrack --ctstate NEW -j ACCEPT
service iptables save   # distro specific
iptables-save > /etc/iptables/rules.v4
```

If nftables:
```bash
nft add rule inet filter input tcp dport 8080 accept
nft list ruleset
```

Debugging tips
- Use `conntrack -L` to inspect connection tracking state.
- Temporarily stop firewalld to confirm firewall is the source of blocks (only in lab):
```bash
systemctl stop firewalld
# test connectivity, then restart
systemctl start firewalld
```

Verification
```bash
# From remote host
nc -vz 192.0.2.10 8080
telnet 192.0.2.10 8080
curl -v http://192.0.2.10:8080/health
```

---

## Lab 9 — SELinux Blocking Network Service

Symptoms
- Service appears to be listening, firewall open, yet connections fail; stops working when SELinux is Enforcing, works when disabled.

Check SELinux mode
```bash
getenforce
sestatus
```

Inspect AVC denials
```bash
ausearch -m avc -ts recent
ausearch -m avc -i | tail -n 50
# or check /var/log/audit/audit.log
grep avc /var/log/audit/audit.log | tail -n 50
```

Common SELinux fixes
- Allow a port type (example: HTTP port 8080)
```bash
# add tcp port 8080 as http_port_t
semanage port -a -t http_port_t -p tcp 8080
# if port exists already, use -m to modify:
semanage port -m -t http_port_t -p tcp 8080
```
- Restore file contexts for web content
```bash
restorecon -Rv /var/www/html
```

If semanage missing:
```bash
yum install policycoreutils-python-utils   # RHEL/CentOS
apt install policycoreutils-python-utils -y
```

Testing
- Put SELinux into permissive to capture denials without enforcing (lab only):
```bash
setenforce 0
# reproduce the failure and check audit logs, then re-enable
setenforce 1
```

---

## Lab 10 — NIC Bonding / Teaming Failure

Symptoms
- Packet loss, high latency, link flapping, or degraded throughput after configuring bond/team.

Gather status
```bash
cat /proc/net/bonding/bond0
ip link show bond0
ethtool eth0
dmesg | grep -i bond
# For teamd:
teamdctl team0 state
```

What to look for in /proc/net/bonding/bond0
- Slave interfaces and their state
- Link status for each slave
- Transmit hash policy
- MII status and link monitoring

Common issues
- Misaligned LACP configuration on switch (LACP vs active-backup mismatch)
- MTU mismatch between slaves and bond
- Using wrong bond mode for switch config (mode 4 requires LACP on switch)
- Spanning Tree or switch port misconfiguration blocking traffic

Troubleshooting steps
1) Verify slave link states
```bash
cat /proc/net/bonding/bond0
# look for "MII Status: up" on slaves
```

2) Check switch side for LACP (use switch CLI to view port channel)
3) If using NetworkManager:
```bash
nmcli con show "bond0"
nmcli con show --active
```

4) Restarting bond safely
```bash
ip link set dev bond0 down
ip link set dev bond0 up
# or restart network stack:
systemctl restart NetworkManager
```

5) Ensure consistent configuration on both ends:
- Bond mode: active-backup (mode=1) works without switch config.
- LACP (mode=4 / 802.3ad) requires switch aggregation.

Verification
```bash
ethtool -S bond0
cat /proc/net/bonding/bond0
ping -I bond0 <peer>
```

---

# General Tools & Commands Cheat Sheet

- Interface & link
  - ip addr show; ip link set dev eth0 up/down
  - ethtool eth0; mii-tool eth0 (older)
- Routing & policy
  - ip route show; ip rule show; ip route add/del; ip route get
- DNS & name resolution
  - cat /etc/resolv.conf; dig; nslookup; systemd-resolve/resolvectl
- Firewalls
  - firewall-cmd --list-all; iptables -L -n; nft list ruleset
- SELinux
  - getenforce; audit logs: ausearch -m avc; semanage; restorecon
- Socket/process
  - ss -tulnp; lsof -i :PORT; systemctl status UNIT; journalctl -u UNIT
- Packet capture
  - tcpdump -i eth0 -nn -s0 -w /tmp/capture.pcap
- Diagnostic networking
  - traceroute, mtr, ping -M do -s SIZE, arping, arp -n, ip neigh show
- Bonding/team
  - cat /proc/net/bonding/bond0; teamdctl team0 state

---

# Safety & Best Practices

- Always gather state and logs before making changes.
- Make changes during maintenance windows in production.
- For persistent changes, update the appropriate network configuration management tool (NetworkManager, Netplan, ifcfg, or your orchestration config).
- Keep a remote console (out‑of‑band) when changing network config so you don't lock yourself out.

---

# Troubleshooting Playbook (Short)

1. Define scope — single host or network-wide.
2. Reproduce from multiple places (localhost, same subnet, remote).
3. Collect: ip addr, ip route, ss, firewall rules, logs (journalctl).
4. Isolate: bring up/down interface, temporarily disable firewall/SELinux (only to test, not long term).
5. Capture traffic with tcpdump for both sides.
6. Apply minimal fix and verify.
7. Persist configuration and document the change.

---

# Escalation Checklist

- If hardware suspected: gather NIC logs (ethtool -S), replace cable, test alternate port, open ticket with infrastructure team.
- If switch/aggregator misconfigured: gather switch LAG/LACP state and escalate to network team.
- If provider upstream outage: collect traceroutes and provider contact details.
- If application-level issue: collect application logs, stack traces, core dumps and escalate to application owners.

---

Using these detailed commands, explanations, and verification steps will give you a practical, production‑oriented toolkit for diagnosing and resolving common Linux networking failures safely and effectively.
