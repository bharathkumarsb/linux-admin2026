# 💥 Kernel Panic — Hands-On Troubleshooting Labs

These labs teach diagnosing, reproducing safely in lab environments, and fixing common kernel panic causes: corrupted or missing initramfs, wrong fstab/NFS entries, missing kernel modules, disk/filesystem corruption, incompatible kernel updates, bad drivers, and simulated hardware faults.

Practice only on lab VMs. Do NOT attempt on production systems. This guide contains no password-bypass or destructive production instructions; always use snapshots and backups.

---

## Contents

- Goals & safety
- Quick commands & artifacts to check
- Lab index
- Detailed labs (1–10) with steps, commands, verification, and notes
- Troubleshooting checklist & best practices
- Suggested exercises and next steps

---

## Goals & Safety

Goals:
- Recognize kernel panic symptoms and root causes
- Use rescue/recovery procedures to restore bootability
- Rebuild/initramfs and fix module-related panics
- Recover from fstab, NFS, and filesystem-related panics
- Gather crash data (kdump) for RCA

Safety:
- Always snapshot or clone VMs before experiments.
- Mount filesystems read-only in rescue scenarios where possible.
- Avoid destructive actions on production; rehearse fixes on lab images.
- Do not include password-bypass methods or insecure permanent changes.

---

## Quick commands & artifacts

- Inspect last kernel messages (if system booted again):
  ```bash
  journalctl -b -1 -k --no-pager
  dmesg -T | tail -n 200
  cat /var/log/messages | grep -i panic
  ```
- List kernels:
  ```bash
  rpm -q kernel
  awk -F\' '/menuentry / {print $2}' /boot/grub2/grub.cfg
  ```
- Initramfs files:
  ```bash
  ls -l /boot/initramfs-*.img
  ```
- Rebuild initramfs (RHEL/CentOS/Alma/Rocky):
  ```bash
  sudo dracut -f
  sudo dracut -f /boot/initramfs-<version>.img <version>
  ```
- Filesystem checks (unmounted):
  ```bash
  sudo fsck -fy /dev/sdXN      # ext4
  sudo xfs_repair /dev/sdXN    # xfs (unmounted)
  ```
- GRUB set default (RHEL/CentOS):
  ```bash
  sudo grub2-set-default "<menuentry-string-or-index>"
  sudo grub2-mkconfig -o /boot/grub2/grub.cfg
  ```
- kdump & crash tool:
  ```bash
  sudo yum install -y kexec-tools
  sudo systemctl enable --now kdump
  ls /var/crash
  sudo crash /var/crash/<timestamp>/vmcore /usr/lib/debug/lib/modules/<kernel>/vmlinux
  ```

---

## Lab Index

1. Identify kernel panic messages from console/logs  
2. Kernel panic after kernel upgrade — boot older kernel  
3. Initramfs missing/corrupted — rebuild (dracut)  
4. Wrong /etc/fstab causes boot failure or panic  
5. Corrupted filesystem → run appropriate fsck/xfs_repair  
6. Missing kernel modules / drivers not in initramfs  
7. Kernel panic caused by NFS/unavailable network root  
8. Out-of-memory (OOM) panic — identify and mitigate (lab)  
9. Hardware-fault simulation (disk/RAM) — lab-safe simulation  
10. Collect crash dump & produce RCA evidence (kdump/crash)

---

# Lab 1 — Identify kernel panic properly

Objective
- Recognize panic types and collect initial evidence.

Symptoms & example messages
- "Kernel panic - not syncing: VFS: Unable to mount root fs"  
- "Kernel panic - not syncing: Attempted to kill init"  
- "Kernel panic - not syncing: Fatal exception"

Steps
1. Capture the console screenshot or notes (important if VM kernel panic shows only on console).
2. On a successfully-booted system (if possible), inspect previous boot logs:
   ```bash
   sudo journalctl -b -1 -p err --no-pager
   sudo dmesg -T | grep -i panic -n
   sudo awk '/panic/,/Call Trace/' /var/log/messages 2>/dev/null
   ```
3. Record: kernel version, last operations (kernel update, disk changes, fstab changes), and time.

Verification
- Ensure you have the exact panic text; it guides the root cause (initramfs vs missing root device vs driver fault).

Notes
- If the VM remains stuck in panic and does not reboot, power-cycle the VM and boot a rescue ISO to inspect disks and logs.

---

# Lab 2 — Kernel panic after kernel upgrade — boot older kernel

Objective
- Recover system by booting a working kernel and clean up the faulty update.

Scenario
- New kernel causes boot panic; previous kernel boots normally.

Steps
1. At GRUB menu choose "Advanced options" and select the last working kernel.
2. After booting, verify running kernel:
   ```bash
   uname -r
   rpm -q kernel
   ```
3. Make the working kernel the default (example using index 0 for the first entry):
   ```bash
   sudo awk -F"'" '/menuentry /{print ++i " : " $2}' /boot/grub2/grub.cfg
   # pick correct index (0..n)
   sudo grub2-set-default <index>
   sudo grub2-mkconfig -o /boot/grub2/grub.cfg
   ```
4. Investigate why new kernel failed: missing modules, incompatible drivers (see Labs 3 & 6).
5. Remove the bad kernel after confirming stable boot:
   ```bash
   sudo yum remove kernel-<bad-version>
   ```

Verification
- Reboot and ensure system boots into selected kernel and services are normal.

Notes
- Keep at least one older kernel installed for quick rollback.

---

# Lab 3 — Initramfs missing/corrupted — rebuild with dracut

Objective
- Fix "root device not found" or initramfs-related panics by regenerating initramfs.

Symptoms
- "dracut: FATAL: root device not found"  
- "VFS: Unable to mount root fs"

Steps
1. Boot rescue media or boot an older kernel (if possible).
2. Check initramfs existence:
   ```bash
   ls -l /boot/initramfs-$(uname -r).img
   ```
   or list all `/boot/initramfs-*`.
3. Recreate initramfs for the active kernel:
   ```bash
   sudo dracut -f
   ```
   To target a specific kernel:
   ```bash
   sudo dracut -f /boot/initramfs-<kernel-version>.img <kernel-version>
   ```
4. If initramfs must include special drivers, add them:
   ```bash
   sudo dracut --force --add-drivers "driver1 driver2"
   ```
5. Re-generate grub (if needed) and reboot:
   ```bash
   sudo grub2-mkconfig -o /boot/grub2/grub.cfg
   sudo reboot
   ```

Verification
- System boots; `dmesg -T` shows normal root mount and init progression.

Notes
- Missing CPU/memory/storage drivers in initramfs are common after custom kernels or kernel updates. Use `dracut --include` or `--add-drivers` as required.

---

# Lab 4 — Wrong /etc/fstab causes boot failure

Objective
- Fix boot hang/panic caused by bad fstab entries (missing device, wrong UUID, network fs).

Symptoms
- Boot drops to emergency mode or shows "cannot mount /dev/mapper/..." or start job for mount runs long.

Steps
1. Boot into rescue, single-user, or use live ISO.
2. Inspect block devices and UUIDs:
   ```bash
   lsblk -f
   sudo blkid
   ```
3. Edit `/etc/fstab` to use correct UUIDs, add `nofail` and `_netdev` for non-critical/remote filesystems:
   ```
   UUID=<uuid>  /data  xfs  defaults,nofail,_netdev 0 0
   ```
4. Test fstab without reboot:
   ```bash
   sudo mount -a
   ```
5. Reboot and verify normal boot.

Verification
- `mount | grep /data` shows mount or absence without blocking boot; `systemctl --failed` should be empty.

Notes
- Use `x-systemd.automount` for automounting on first access to avoid boot waits.

---

# Lab 5 — Corrupted filesystem → run fsck / xfs_repair

Objective
- Repair filesystem corruption that causes kernel panic or boot failure.

Caveat
- Run repair tools on unmounted filesystems or in rescue environment.

Steps (ext4)
1. Boot rescue or unmount filesystem:
   ```bash
   sudo umount /dev/sdXN
   sudo fsck -fy /dev/sdXN
   ```
2. Read fsck output carefully and allow fixes. After repair, mount and test.

Steps (XFS)
1. XFS requires `xfs_repair` and unmounted filesystem:
   ```bash
   sudo umount /dev/sdXN
   sudo xfs_repair /dev/sdXN
   ```
2. If `xfs_repair` suggests recovery steps or `-L`, use caution—read docs.

Verification
- Mount filesystem and run `dmesg -T` to ensure no errors; `df -h` to check space.

Notes
- Back up data before dangerous repairs. For critical corruption, consider imaging the block device first.

---

# Lab 6 — Missing kernel modules / driver panic

Objective
- Recover when kernel panic due to missing driver modules required early in boot.

Symptoms
- Initramfs or kernel messages: "could not load module ..." or "unknown filesystem" or missing controller.

Steps
1. Boot rescue/working kernel and check installed modules:
   ```bash
   lsmod
   modinfo <module_name>
   ```
2. Rebuild initramfs including required modules:
   ```bash
   sudo dracut --force --add-drivers "<module1> <module2>"
   sudo dracut -f
   ```
3. If module is missing from the kernel package, install kernel-devel or appropriate driver package, or use DKMS to build out-of-tree modules.

Verification
- Reboot into updated initramfs/kernel and confirm modules loaded early (dmesg shows module load).

Notes
- If using third-party drivers (e.g., vendor NICs), ensure matching kernel module version or use vendor-supplied initramfs/drivers.

---

# Lab 7 — Kernel panic due to NFS / network root unavailable

Objective
- Avoid panics when rootfs or required filesystems are over NFS and server unavailable.

Symptoms
- Kernel panics or long boot waits with messages about NFS server not responding.

Steps & best practices
1. Avoid using NFS as root unless absolutely required. If used, ensure robust initramfs and network early init.
2. Use safe `/etc/fstab` options for NFS:
   ```
   server:/export/rootfs  /mnt/root  nfs  defaults,_netdev,bg,nofail 0 0
   ```
3. For root-on-NFS setups, ensure initramfs built with network/NFS support and correct kernel cmdline (advanced).
4. To recover a boot blocked by NFS mount, boot rescue and comment out NFS entries or add `nofail` then reboot.

Verification
- After adjustments, reboot and ensure system proceeds without blocking on unavailable NFS exports.

Notes
- NFS v4 simplifies port usage but does not eliminate dependency issues. Use `systemd` options like `x-systemd.device-timeout=` and automount to reduce boot impact.

---

# Lab 8 — Out-of-memory (OOM) kernel panic (lab simulation)

Objective
- Identify OOM events and mitigate (for lab scenario).

Symptoms
- System kills processes with "OOM-killer invoked" or becomes unresponsive.

Investigation
1. Check kernel logs:
   ```bash
   sudo dmesg -T | grep -i -e oom -e kill
   sudo journalctl -k | grep -i oom
   ```
2. Inspect process memory use:
   ```bash
   ps aux --sort=-rss | head -n 20
   top    # press M to sort by memory
   ```
Mitigations (lab-only)
1. Add temporary swap:
   ```bash
   sudo fallocate -l 2G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```
2. Restart or limit offending process (use `systemctl restart` or `kill`).

Verification
- `free -h` shows swap usage; kernel stops invoking OOM if memory pressure relieved.

Notes
- For production, plan capacity, use cgroup limits (`MemoryMax=` in systemd unit), and ensure OOM handling (oom_score_adj).

---

# Lab 9 — Hardware-fault simulation (RAM / disk) — lab-safe

Objective
- Simulate hardware faults in VM and observe kernel messages; practice diagnosing with logs.

Disk simulation (safe VM approach)
1. Add and remove a virtual disk:
   - Detach a secondary disk in VM settings and monitor `dmesg -T` for errors:
     ```bash
     sudo dmesg -T | tail -n 50
     ```
2. Observe degraded RAID (if configured) or I/O errors.

RAM simulation (VM-only safe method)
- Use stress tools to force memory pressure (do this only in lab):
  ```bash
  sudo yum install -y stress-ng
  stress-ng --vm 2 --vm-bytes 80% --timeout 60s
  ```
- Watch for OOM or kernel logs.

Verification
- `dmesg -T` shows hardware or I/O errors; `smartctl` (on physical systems) reveals SMART failures:
  ```bash
  sudo smartctl -a /dev/sda
  ```

Notes
- For physical hardware, follow vendor guidance; use diagnostics like memtest86+ and SMART tools.

---

# Lab 10 — Collect crash dump & RCA evidence (kdump + crash)

Objective
- Configure kdump, collect vmcore, and use `crash` for root-cause analysis.

Steps
1. Install and enable kdump (RHEL/CentOS example):
   ```bash
   sudo yum install -y kexec-tools
   sudo systemctl enable --now kdump
   sudo systemctl start kdump
   sudo systemctl status kdump
   ```
2. Trigger/test crash dump (lab-only; do not do in production):
   - As root, echo to sysrq trigger (only in disposable lab VMs):
     ```bash
     echo c | sudo tee /proc/sysrq-trigger
     ```
   - VM will reboot and kdump should capture `/var/crash/<timestamp>/vmcore`.
3. Analyze vmcore:
   ```bash
   ls /var/crash
   sudo crash /var/crash/<timestamp>/vmcore /usr/lib/debug/lib/modules/$(uname -r)/vmlinux
   # inside crash:
   bt     # backtrace
   ps     # process list at crash
   dmesg  # kernel log snapshot
   ```
4. Collect logs and evidence: `/var/log/messages`, `journalctl -b -1`, kernel config, installed kernel RPM.

Verification
- Successful `crash` inspection yields stack traces and pointers to modules or kernel functions involved.

Notes
- Configure kdump kernel memory reservation (`crashkernel=`) if required by distro.
- For production RCA, coordinate with support teams and preserve vmcore securely.

---

## Troubleshooting checklist & best practices

- Snapshot/backup before attempting repairs.
- Keep at least one working kernel installed.
- Always rebuild initramfs after kernel changes or when adding drivers needed for boot.
- Use UUIDs in `/etc/fstab` and `nofail/_netdev/x-systemd.automount` for non-critical mounts.
- For XFS, use `xfs_repair` on unmounted partitions only.
- Check SELinux AVC denials (not commonly direct cause of kernel panic but can cause app failures leading to panic scenarios).
- Use kdump to capture vmcore for post-mortem; preserve evidence.
- Document all remediation steps and change control entries (what was changed, why, and rollback).
- Test kernel and initramfs changes on staging before production.

---

## Suggested exercises

1. Simulate a bad initramfs (remove or rename `/boot/initramfs-<version>.img`) in a disposable VM and practice rebuilding with `dracut`. Reproduce and recover.
2. Install a newer kernel package in a lab VM, test booting both kernels, revert to previous kernel if panic occurs.
3. Create a deliberate fstab entry for a non-existent device and practice rescue-mode recovery and adding `nofail`.
4. Simulate heavy memory usage using `stress-ng` and observe OOM behavior; configure `MemoryMax=` for a service and test behavior.
5. Enable `kdump` on a lab VM, trigger a test crash, and run `crash` to extract a backtrace.

---

## Documentation & presentation notes

- Use descriptive headings, command blocks, and short verification steps in runbooks.
- GitHub's Markdown renderer controls fonts; ensure clear structure and monospace code blocks for commands.
- For runbooks in production, include rollback steps and impact assessment.
- Consider adding small diagrams or sequence diagrams for complex causes (initramfs -> kernel modules -> rootfs) in supplementary docs.

---

- Produce a printable PDF or an MkDocs site for this material.

Which would you like next?
