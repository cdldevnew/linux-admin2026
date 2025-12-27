# Linux Admin 2026 — Practical Labs & Troubleshooting

Practical, hands‑on Linux administration labs, troubleshooting scenarios, and interview preparation materials for sysadmins, SREs, DevOps engineers, and candidates preparing for RHCSA/RHCE interviews.

This repository collects scenario-based guides and step‑by‑step commands covering basics through advanced topics: user and group management, disk/LVM, filesystems, systemd, NFS, kernel panic recovery, service troubleshooting, and common interview questions.

---

## Quick links (Table of Contents)

- [linux-basic-commands.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-basic-commands.md) — Core command-line labs: file ops, searching, process management, permissions, networking, compression, and shell tips.
- [linux-advance-commands.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-advance-commands.md) — Hands-on advanced labs: systemd, SELinux, advanced networking, automation, repo/package management, and NFS server/client topics.
- [linux-q-n-a.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-q-n-a.md) — 145 commonly asked Linux admin Q&A with explanations, commands, verification, pitfalls, and interview tips.
- [linux-interview-qustns.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-interview-qustns.md) — Interview scenario labs: realistic incident simulations and step-by-step troubleshooting (disk full, boot problems, SSH, high CPU, etc.).
- [linux-service-failures-systemctl.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-service-failures-systemctl.md) — systemd troubleshooting: failed units, logs, dependencies, ExecStart issues, start limits, and useful commands.
- [lvm-troubleshooting.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/lvm-troubleshooting.md) — LVM recovery and maintenance labs: expanding/reducing LVs, metadata restore, missing PVs, snapshots, and migration.
- [linux-nfs-issues.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-nfs-issues.md) — Enterprise NFS troubleshooting labs: timeouts, stale handles, SELinux, firewall, autofs, performance, and boot hangs.
- [linux-kernel-panic-issues.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-kernel-panic-issues.md) — Kernel panic diagnosis and recovery: initramfs, fsck, GRUB, missing modules and kdump analysis.

---

## Who is this for?

- Engineers learning or refreshing practical Linux administration skills.
- Candidates preparing for technical interviews (SRE/DevOps/Linux admin roles).
- Anyone who wants reusable lab-style exercise steps to practice safely in VMs.

---

## Recommended workflow / Learning path

1. Start with [linux-basic-commands.md] for command fundamentals and safe lab setup.
2. Move to [linux-advance-commands.md] to practice system-level topics and automation.
3. Use [linux-q-n-a.md] and [linux-interview-qustns.md] for interview-style scenarios and quick Q&A review.
4. Focus on topic-specific guides (LVM, NFS, systemd, kernel panic) when you encounter or want to practise those exact scenarios.

---

## Safety & prerequisites / Important notes

- Always practice in disposable VMs or snapshots. Many labs involve destructive operations (disks, LVM, partitions).
- Recommended distros: CentOS/RHEL 7–8, Rocky, AlmaLinux, Ubuntu (adapt commands where needed).
- Use `sudo` where required.
- Read each lab's "Prerequisites & Safety" note before executing commands.

---

## How to use this repository

1. Clone or browse the repository:
   - Clone: git clone https://github.com/bharathkumarsb/linux-admin2026.git
   - Browse files on GitHub using the links in the Table of Contents above.
2. Create a lab VM and snapshot before starting any exercise.
3. Work through the labs in order or pick topics relevant to your learning goals.
4. Use the verification steps in each lab to confirm outcomes and understand pitfalls.

---

## Contributing

- Found a bug, typo, or have an improvement? Please open an issue or submit a pull request.
- When submitting PRs:
  - Keep changes limited to a single topic (typo fixes, clarity improvements, added steps).
  - Include tested commands and verification steps.
  - Mark any distribution-specific instructions (e.g., Ubuntu vs RHEL).

---

## About / Attribution

- Author: bharathkumarsb (repository owner)
- This README was generated to consolidate and make the repository easier to navigate; the authoritative, detailed labs are the individual Markdown files linked above.

---

## Next steps (suggested)

- Add a LICENSE file (if you want public reuse — e.g., MIT or CC BY).
- Add CONTRIBUTING.md with PR/issue templates.
- Add small examples or scripts for commonly used safe tasks (non-destructive), and badges (build/status/license) if desired.

---
