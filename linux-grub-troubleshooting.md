# 🧩 GRUB & Bootloader — Hands‑On Troubleshooting Labs (Detailed Commands & Explanations)

These labs cover common GRUB and bootloader failures on Linux virtual machines and physical servers. They focus on diagnosing and restoring normal boot behavior: missing menus, grub prompts, missing kernels, wrong root device mappings, initramfs problems, graphical/console boot issues, and dual‑boot detection.

Important safety notes
- ⚠️ No password‑bypass or authentication‑bypass instructions are included. Do not attempt to modify kernel command lines to gain root shells (e.g., adding `init=/bin/bash`, `single`, `rd.break`, or similar).
- 🧪 Practice only in lab VMs, not production systems.
- Always gather state and logs before making changes. Keep out‑of‑band (console) access available.

---

## Lab Index

1. GRUB menu not showing — enable persistent menu
2. Booting to `grub>` (GRUB prompt)
3. Booting to `grub-rescue>` prompt
4. Missing kernel entry in GRUB menu
5. Wrong default kernel boots — change default
6. GRUB config corrupted — rebuild configuration
7. Disk UUID changed after cloning / moving VM — update root device
8. Graphical boot stuck — switch to text boot and diagnose graphics
9. Black screen after boot (no login) — console target troubleshooting
10. Dual‑boot: GRUB not detecting other OS

Each lab contains: symptoms, commands to gather evidence, safe remediation steps, persistence, verification, and common root causes.

---

## Lab 1 — GRUB menu not showing (hidden or zero timeout)

Symptoms
- System boots too quickly; GRUB menu never appears.
- You want to select older kernels or recovery entries.

Gather evidence
```bash
# Inspect GRUB defaults
cat /etc/default/grub
# See the generated config (BIOS & UEFI paths differ)
ls -l /boot/grub2/grub.cfg /boot/efi/EFI/*/grub.cfg 2>/dev/null
```

Fix (RHEL / CentOS / Fedora)
1) Edit `/etc/default/grub`:
```ini
GRUB_TIMEOUT=5
GRUB_TIMEOUT_STYLE=menu    # show menu explicitly
# GRUB_RECORDFAIL_TIMEOUT and GRUB_DISABLE_SUBMENU might also affect behavior
```
2) Rebuild GRUB configuration:
- BIOS (legacy):
```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```
- UEFI:
```bash
grub2-mkconfig -o /boot/efi/EFI/<distro>/grub.cfg
# e.g. /boot/efi/EFI/centos/grub.cfg or /boot/efi/EFI/fedora/grub.cfg
```

Fix (Debian / Ubuntu)
1) Edit `/etc/default/grub` similarly.
2) Rebuild:
```bash
update-grub
```

Verification
```bash
# Reboot and confirm menu visible and timeout honored.
# Also inspect the new grub.cfg to confirm timeout/menu settings
grep -E "set timeout|menuentry" /boot/grub2/grub.cfg 2>/dev/null || grep -E "set timeout|menuentry" /boot/grub/grub.cfg
```

Common root causes
- GRUB timeout set to 0 or `hidden`.
- Distribution defaults that hide menu unless Shift/Esc pressed.
- Automatic updates that change `/etc/default/grub`.

---

## Lab 2 — System stops at `grub>` prompt (GRUB prompt)

Meaning
- GRUB can run but cannot locate or parse its normal configuration; `grub>` is a minimal prompt with full module functionality.

Gather evidence from the prompt
1) List devices and partitions
```
grub> ls
(hd0) (hd0,gpt1) (hd1,gpt1) ...
```
2) Inspect partitions to find `/boot` or kernel files:
```
grub> ls (hd0,gpt1)/
# look for /boot, grub, or vmlinuz files
```

Safe remediation (conceptual steps)
- Identify the partition with `/boot` or `/grub2`.
- Set `prefix` and `root` so GRUB can load its modules and menu:
```
grub> set root=(hd0,gpt1)
grub> set prefix=(hd0,gpt1)/grub2   # or /boot/grub for some distros
grub> insmod normal
grub> normal
```
Explanation
- These commands allow GRUB to find its `normal` module and present the standard menu. They do not show or teach how to modify kernel command lines to bypass authentication.

After booting the OS, fully repair GRUB configuration (see Lab 6).

If these `insmod` steps fail:
- The filesystem may be missing GRUB modules (corrupt or wrong partition).
- Boot from rescue media and perform a proper reinstall with chroot (see safe chroot steps below).

---

## Lab 3 — System stops at `grub-rescue>` prompt

Meaning
- GRUB's core image could not find required modules or the configuration; `grub-rescue>` is a very limited environment.

Gather evidence
```
grub-rescue> ls
grub-rescue> ls (hd0,gpt1)/
```

Remediation (safe & minimal)
- Identify the partition containing `/boot` or GRUB files using `ls`.
- If you find the partition, set `prefix` and `root` as in Lab 2 and try `insmod normal` / `normal`. This recovers the menu when modules and files are present.

If GRUB files are missing/corrupt
- Boot a rescue/live ISO and use chroot to reinstall GRUB (safe chroot workflow shown later). Do not use rescue prompts to change kernel boot parameters for authentication bypass.

---

## Lab 4 — Missing kernel entry in GRUB menu

Symptoms
- GRUB menu appears but kernel entries are absent or fewer than expected.

Gather evidence
```bash
# Show installed kernel packages (RHEL/CentOS)
rpm -q kernel
# Debian/Ubuntu
dpkg -l 'linux-image-*' | grep ^ii

# Inspect /boot for vmlinuz and initramfs images
ls -l /boot | egrep 'vmlinuz|initramfs|initrd'
```

Common causes
- Kernel package removed or not installed.
- GRUB config not regenerated after new kernel install.
- /boot partition full or mount issues prevented kernel installation.

Remediation
1) Ensure a kernel package is installed; reinstall if missing:
- RHEL/CentOS:
```bash
yum reinstall kernel -y     # or dnf reinstall kernel -y
```
- Debian/Ubuntu:
```bash
apt update
apt install --reinstall linux-image-$(uname -r)   # or a specific kernel version
```

2) Rebuild GRUB menu
- RHEL/CentOS:
```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```
- Debian/Ubuntu:
```bash
update-grub
```

3) If /boot is a separate partition, ensure it's mounted before kernel install:
```bash
mount | grep /boot
# mount it if necessary:
mount /dev/sdXn /boot
```

Verification
```bash
grep ^menuentry /boot/grub2/grub.cfg   # RHEL
grep ^menuentry /boot/grub/grub.cfg    # Debian
ls -l /boot/vmlinuz-* /boot/initramfs-*
```

---

## Lab 5 — Wrong default kernel boots (choose previous kernel)

Scenario
- New kernel is unstable; you want to boot a previous known‑good kernel by default.

Gather evidence
```bash
# List menuentries with indices (RHEL)
awk -F"'" '/menuentry /{print i++ " : " $2}' /boot/grub2/grub.cfg
# Or simply:
grep ^menuentry /boot/grub2/grub.cfg
```

Set default (RHEL & older tools)
- By entry index:
```bash
# Set default by menu entry ID (index)
grub2-set-default 2    # zero-based index
grub2-editenv list     # verify saved entry
```
- By exact menuentry name:
```bash
grub2-set-default "CentOS Linux (4.18.0-xyz) 7 (Core)"
```

Debian/Ubuntu
- Edit `/etc/default/grub`, set:
```ini
GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 5.xx.x-xx-generic"
```
- Then regenerate:
```bash
update-grub
```

Verification
```bash
# Reboot and confirm
uname -r
# Or use grub2-editenv list to show saved default
grub2-editenv list
```

Notes
- Avoid manual edits of grub.cfg; always update defaults and regenerate config.

---

## Lab 6 — GRUB config corrupted — rebuild cleanly

Symptoms
- GRUB presents errors mentioning `/boot/grub2/grub.cfg` or shows malformed menu.

Gather state and remediate
1) Inspect `/etc/default/grub` for syntax errors.
2) Regenerate GRUB configuration:
- RHEL/BIOS:
```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```
- RHEL/UEFI (ensure correct EFI path):
```bash
grub2-mkconfig -o /boot/efi/EFI/<distro>/grub.cfg
```
- Debian/Ubuntu:
```bash
update-grub
```

3) If grub modules or stage files are corrupt, reinstall GRUB bootloader:
- BIOS (install to MBR of disk `/dev/sda`):
```bash
# On Debian/Ubuntu and many distros:
grub-install /dev/sda
# or explicit for RHEL systems:
grub2-install /dev/sda
```
- UEFI:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=<distro>
# then regenerate config as above
```

Safe chroot workflow (when booting from live/rescue ISO is necessary)
1) From live ISO:
```bash
# mount root and boot (adjust device names)
mount /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot   # if /boot separate
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
chroot /mnt /bin/bash
# Inside chroot: regenerate grub and reinstall
grub2-install /dev/sda
grub2-mkconfig -o /boot/grub2/grub.cfg
exit
umount -R /mnt
```
2) Reboot and test.

Do NOT use chroot steps to modify authentication artifacts.

Verification
```bash
# Confirm grub.cfg exists and contains menu entries
ls -l /boot/grub2/grub.cfg
grep ^menuentry /boot/grub2/grub.cfg | sed -n '1,20p'
```

---

## Lab 7 — Disk UUID changed (cloned VM / disk re partitoned)

Symptoms
- Boot fails with messages like `error: no such device: <UUID>` or kernel cannot mount root.

Gather evidence & fix
1) Get current block identifiers
```bash
blkid
lsblk -f
```

2) Update `/etc/fstab`
- Replace stale UUIDs with current ones (prefer LABEL or device path if dynamic).
```bash
# Edit safely
cp /etc/fstab /etc/fstab.bak
vi /etc/fstab
# Replace old UUID=xxxx with correct values from 'blkid'
```

3) Update GRUB config so embedded UUID references are corrected:
- RHEL:
```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```
- Debian:
```bash
update-grub
```

4) If the root device has been moved to another disk (sda → sdb), consider reinstalling GRUB to the correct disk MBR/EFI:
```bash
grub2-install /dev/sda    # ensure you install on the disk used for boot
grub2-mkconfig -o /boot/grub2/grub.cfg
```

Verification
- Reboot and confirm boot completes.
- Use `findmnt /` and `cat /etc/mtab` to confirm the root filesystem is correct.

Common causes
- Cloning VMs without regenerating UUIDs.
- Restoring/restaging disks from backups with different device naming.
- Repartitioning that changes partition numbering.

---

## Lab 8 — Graphical boot stuck on splash (X/Wayland/driver issues)

Symptoms
- System shows vendor or distribution splash, then stalls.
- Switching to a TTY (Ctrl+Alt+F2) may or may not work.

Safe diagnostic steps
1) Temporarily boot to text mode (so you can safely investigate)
```bash
# Make multi-user.target the default (text mode)
systemctl set-default multi-user.target
reboot
```
2) Once booted into console, check display manager and graphics logs:
```bash
systemctl status gdm sddm lightdm  # depending on environment
journalctl -b -p err --no-pager
# Xorg logs:
cat /var/log/Xorg.0.log | sed -n '1,200p'
# or for systemd journal-based logs:
journalctl -b _COMM=Xorg
```

Common fixes
- Proprietary driver mismatch after kernel update: reinstall matching driver for kernel (e.g., NVIDIA driver packages).
- Kernel mode setting (KMS) conflicts: consider using safe driver packages or adjust driver installation instead of persistent kernel cmdline tweaks.
- Corrupt user-specific GPU config: test by creating a new user or moving `~/.config` X/Wayland files.

Re-enable graphical.target after fix:
```bash
systemctl set-default graphical.target
reboot
```

Notes
- Editing kernel command line to add `nomodeset` can be used as a troubleshooting measure to fall back to basic graphics; this is a non‑auth bypass use-case and should be applied only to diagnose and revert when appropriate. Prefer fixing driver/kernel mismatch.

---

## Lab 9 — System boots but you see a black screen (no login prompt)

Symptoms
- Machine appears powered, monitor shows black, no login prompt, or GUI fails to render.

Safe checks
1) Switch to a different VT:
```
Ctrl + Alt + F2
```
2) If you can access a console, check systemd and the display manager:
```bash
systemctl status getty@tty1
systemctl status gdm sddm lightdm
journalctl -b -u gdm -e
dmesg | egrep -i "nvidia|amd|modeset|drm"
```

If you cannot switch VTs and have only remote access, consider rebooting to a console target (use caution in production):
```bash
# From remote shell, set text target and reboot (ensure remote console exists)
systemctl set-default multi-user.target
reboot
```

If display manager is failing, restart it and inspect logs:
```bash
systemctl restart gdm
journalctl -u gdm -b --no-pager -n 200
```

Verification
- After fix and switching back to graphical.target, you should get normal login.

---

## Lab 10 — Dual‑boot: GRUB not detecting other OS (Windows, other Linux)

Symptoms
- After installing Linux, system boots only Linux; Windows/other OS missing from GRUB menu.

Gather evidence
```bash
# Ensure os-prober is installed
# Debian/Ubuntu
apt install os-prober -y
# RHEL/Fedora (os-prober may not be packaged by default)
dnf install os-prober || yum install os-prober

# Run os-prober manually
os-prober
```

Enable OS detection (Debian/Ubuntu)
1) Edit `/etc/default/grub` and set:
```ini
GRUB_DISABLE_OS_PROBER=false
```
2) Regenerate GRUB:
```bash
update-grub
```

RHEL/Fedora
- Re-run grub2-mkconfig; on EFI systems GRUB should pick other OSes if `os-prober` is present:
```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

UEFI considerations
- Ensure that the firmware's boot order retains the GRUB entry and that Windows Boot Manager entries exist in the EFI partition. Use `efibootmgr -v` to inspect boot entries and order.

Verification
- Reboot and check the GRUB menu for other OS entries.
- For UEFI, ensure both GRUB and Windows Boot Manager are visible in the firmware boot menu.

---

## Tools & Commands Cheat Sheet (Common tasks)

- Inspect boot config & versions
  - cat /etc/default/grub
  - grep ^menuentry /boot/grub2/grub.cfg
  - update-grub (Debian), grub2-mkconfig -o /boot/grub2/grub.cfg (RHEL)
- Inspect /boot and kernels
  - ls -l /boot
  - rpm -q kernel | dpkg -l 'linux-image-*'
- Reinstall GRUB
  - grub-install /dev/sda
  - grub2-install /dev/sda
  - grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=<name>
- Recreate initramfs
  - RHEL: dracut --force /boot/initramfs-$(uname -r).img $(uname -r)
  - Debian: update-initramfs -u -k all
- Manage targets
  - systemctl set-default multi-user.target
  - systemctl set-default graphical.target
- Disk identification and UUID fixes
  - blkid
  - lsblk -f
  - tune2fs -l /dev/sdXN  (for ext filesystems)
- UEFI boot entries
  - efibootmgr -v
- Safe rescue/chroot workflow
  - mount root and boot
  - mount --bind /dev /proc /sys
  - chroot /mnt /bin/bash
  - perform grub-install, grub-mkconfig, update-initramfs
  - exit; unmount

---

## Troubleshooting Playbook (Step‑by‑step)

1. Confirm the failure mode and scope: single machine or many.
2. Collect current state before changes: `journalctl -b`, `lsblk`, `blkid`, `cat /etc/fstab`, `/etc/default/grub`, `/boot` contents.
3. If GRUB prompt: try to identify boot partition with `ls` from the prompt; if possible, set `prefix` and `insmod normal` to recover menu.
4. If GRUB config missing/corrupt: boot rescue media, chroot, reinstall GRUB, regenerate config.
5. If kernel/initramfs missing or broken: reinstall kernel packages and rebuild initramfs (`dracut` or `update-initramfs`).
6. If root device changed (UUIDs): update `/etc/fstab` and regenerate grub config. Validate `blkid` values.
7. If graphics boot fails: revert to text mode, collect driver and journal logs, reinstall/upgrade matching driver and kernel modules.
8. Regenerate configs and reboot. Verify and document changes.

---

## Escalation & When to Involve Others

- Hardware suspected (failing disk, controller): collect SMART logs (`smartctl -a`), kernel dmesg, and escalate to hardware/infrastructure team.
- Firmware / BIOS/UEFI issues: collect `efibootmgr -v` output and firmware revision; coordinate with platform owner for firmware updates.
- Complex storage changes (LVM, RAID, SAN): gather LVM metadata, mdadm status, and involve storage team.
- Bootloader that repeatedly fails after configuration changes: capture full boot logs (`journalctl -b -1`), and consider preserving `/boot` contents and full disk images for safe recovery.

---

## Safety & Best Practices

- Always back up `/etc/fstab`, `/etc/default/grub`, and `/boot` before making changes.
- If working remotely, ensure an out‑of‑band or console recovery path exists to avoid lockout.
- Use package manager to reinstall kernel/drivers rather than manual file copies unless you understand all implications.
- Prefer regenerating configs (`grub2-mkconfig`, `update-grub`) over hand‑editing generated files. Only edit generated files if you fully understand consequences and have backups.
- Document any change and testing sequence for rollback.

---

Using these labs and commands you can safely diagnose and remediate common GRUB and bootloader issues while avoiding authentication bypass techniques. Practice in isolated lab VMs and always preserve recovery access before making bootloader changes.
