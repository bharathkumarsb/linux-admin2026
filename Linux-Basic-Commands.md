# 🐧 Linux Commands — Expanded Definitions, Explanations & Detailed Examples

This document expands on common Linux commands and concepts with clearer definitions, rationale, flags you should know, real-world examples, verification commands, and important caveats. Use this as an extended reference for learning, interviews, and sysadmin tasks.

Notes:
- Commands that require elevated privileges include `sudo`. Adjust for your environment.
- Many examples assume bash-compatible shells. Adapt for other shells accordingly.

---

## Table of contents (expanded)
1. System & kernel: definitions, why they matter, and examples
2. Hardware, disks & filesystems: important flags and safe procedures
3. User & group management: security best practices
4. File operations & text processing: idioms and useful pipelines
5. Permissions, ACLs & special bits: when to use what
6. Networking & DNS: diagnostics and packet tools
7. Package management nuances: safe installs, rollbacks
8. Compression & archiving: options, integrity checks
9. Processes, services & systemd: lifecycle and unit files
10. Scheduling & automation: crontab, at, systemd timers
11. Monitoring & performance: interpreting metrics
12. Security: SELinux/AppArmor, firewall basics
13. Shell scripting: robust patterns, traps, debugging
14. LVM & storage operations: grow/shrink safely
15. Useful compound examples and troubleshooting patterns

---

## 1. System & Kernel — What and why

Definition
- Kernel: core of the OS controlling hardware, scheduling, memory, drivers.
- Distribution: userspace (packages, init system) around the kernel.

Key commands, deeper explanation and examples:
- uname
  - What: prints kernel name/version and architecture.
  - Why: determine kernel release (affects module compatibility, CVEs).
  - Examples:
    - `uname -a` -> "Linux host 5.15.0-91-generic #98-Ubuntu SMP..."
    - `uname -m` -> architecture (x86_64, aarch64).
  - Verify kernel modules: `lsmod | grep <module>` and `modinfo <module>`.

- hostnamectl
  - What: systemd-managed host metadata (hostname, chassis, virtualization).
  - Example: `sudo hostnamectl set-hostname web-prod-01`
  - Verify: `hostname`, `cat /etc/hostname`, `hostnamectl status`

- uptime and load averages
  - The 1/5/15 minute load numbers represent processes waiting for CPU or I/O. On a single-core VM, a load of 2 means saturation; on 8 cores, 2 is low.
  - Example: `uptime` -> "load average: 1.23, 0.95, 0.80"
  - Investigate high load: `top` (sort by CPU), `iostat -xz 1` (IO), `ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head`

---

## 2. Hardware, disks & filesystems — safe workflows

- lsblk / blkid / fdisk / parted
  - lsblk shows block devices with mountpoints. Use `-f` to see FS types and labels.
    - `lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,LABEL`
  - Use `blkid` or `/dev/disk/by-uuid/` for persistent UIDs to put in `/etc/fstab`.
  - Example fstab entry using UUID:
    - `UUID=3f8e7c9b-... /data ext4 defaults,noatime 0 2`

- Creating filesystems (safe steps)
  - Always check the device: `lsblk`, `udevadm info -q all -n /dev/sdb`
  - Example:
    - `sudo parted -s /dev/sdb mklabel gpt mkpart primary 0% 100%`
    - `sudo mkfs.ext4 -L data /dev/sdb1`
  - Verify:
    - `sudo blkid /dev/sdb1`
    - `sudo tune2fs -l /dev/sdb1` (ext4 specifics)

- Mount options
  - noatime (reduce writes), nodiratime, ro (read-only), defaults
  - Example: temporary mount:
    - `sudo mount -o noatime /dev/sdb1 /mnt/data`
  - Verify: `mount | grep /mnt/data`

- SMART and disk health
  - `sudo smartctl -a /dev/sda` and `sudo smartctl -t short /dev/sda` to run tests.
  - Look for "SMART overall-health self-assessment test result: PASSED"

- Filesystem check
  - Always unmount before `fsck`: `sudo umount /mnt/data && sudo fsck -f /dev/sdb1`
  - For root filesystem checks, boot to rescue or use initramfs.

---

## 3. User & Group Management — least privilege

- useradd vs adduser
  - `useradd` is low-level; `adduser` (Debian) is friendlier by prompting.
  - Create a user with home and shell:
    - `sudo useradd -m -s /bin/bash -G developers alice`
    - `sudo passwd alice`

- Managing sudo privileges
  - Use `visudo` to avoid syntax errors. Example line to give group sudo without password:
    - `%admins ALL=(ALL) NOPASSWD:ALL`
  - Verification:
    - `sudo -l -U alice` shows effective sudo privileges

- Best practice: use groups like `deploy`, `docker`, `www-data` for service access, rather than adding many users to `sudo`.

---

## 4. File operations & text processing — idioms

- Safe copy & verification
  - `cp -a /src /dst` keeps attributes; verify with `rsync --dry-run -av /src /dst` or `sha256sum` comparisons.
- Common pipelines
  - `journalctl -u nginx -n 200 | grep -i 'error' | awk '{print $5,$6,$7}' | uniq -c`
- Using grep effectively
  - `grep -E` for extended regex, `-P` for Perl regex if supported, `-r` recursive, `-n` show line numbers.
  - Example: find lines with IPv4 addresses:
    - `grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' /var/log/* | sort | uniq -c | sort -nr`

---

## 5. Permissions, ACLs & special bits — deeper meaning

- Numeric permission breakdown
  - Owner/Group/Others, each bit: read(4), write(2), execute(1)
  - `chmod 750 file` -> owner rwx (7), group r-x (5), others none (0)

- setuid and setgid
  - setuid on executable allows it to run with file owner's permissions.
  - Dangerous if applied to scripts or network-exposed programs.
  - Example: `sudo chmod u+s /usr/bin/passwd` (passwds needs to write /etc/shadow as root)
  - Check setuid/setgid files: `find / -perm /6000 -type f -exec ls -l {} \;`

- Sticky bit
  - On directories, `chmod +t dir` prevents users from deleting files they don't own (e.g., /tmp has sticky bit).
  - Example: `ls -ld /tmp` shows `drwxrwxrwt`.

- ACL examples
  - Grant user alice rwx on a directory, without changing the group:
    - `sudo setfacl -m u:alice:rwx /srv/shared`
  - Remove: `sudo setfacl -x u:alice /srv/shared`
  - View: `getfacl /srv/shared`

---

## 6. Networking & DNS — diagnose like a pro

- ip vs ifconfig
  - `ip a` shows interfaces and IPs. `ifconfig` is deprecated on many distros.
- Routes
  - `ip route show` to learn default gateway and policy routing.
- DNS lookup tools
  - `dig` provides raw DNS response. `dig +trace example.com` shows the delegation path.
  - `nslookup` is simple but less flexible.
- Live capture
  - `sudo tcpdump -i eth0 -n -s 0 -w capture.pcap port 80` (capture traffic to file)
  - Open capture in Wireshark for analysis.
- Connectivity troubleshooting sequence
  - `ping` (ICMP) -> confirm host reachable
  - `traceroute`/`tracepath` -> path and hops
  - `dig` or `nslookup` -> DNS resolution
  - `ss -tulnp` -> confirm service listening on port
  - `curl -v http://host:port` -> inspect HTTP exchange
  - `tcpdump` if packets are not reaching/returning

---

## 7. Package Management — safe operations

- APT safe flow (Debian/Ubuntu)
  - Always: `sudo apt update` then `sudo apt upgrade`
  - To install: `sudo apt install -y package`
  - Rollbacks: snapshot the system or use `apt-mark hold packagename` to prevent upgrades.
  - Check packages installed from a local .deb: `dpkg -i package.deb` then `sudo apt-get -f install` to fix dependencies.

- RPM/DNF/YUM (RHEL/CentOS)
  - `sudo dnf history` shows transactional history; you can undo transactions by ID:
    - `sudo dnf history undo <transaction-id>` (where supported)

---

## 8. Compression & archiving — reliability

- Tar tips:
  - Use `-p` to preserve permissions when extracting as root: `tar -xvpf archive.tar.gz`
  - For large backups, creating a compressed stream: `tar -c /var/www | pv | gzip -c > www.tar.gz` (pv shows progress)
- Integrity:
  - Always record checksums: `sha256sum archive.tar.gz > archive.tar.gz.sha256`
  - Verify later: `sha256sum -c archive.tar.gz.sha256`

---

## 9. Processes, services & systemd — lifecycle control

- systemctl unit basics
  - Unit files live in `/lib/systemd/system` or `/etc/systemd/system`.
  - Example minimal service file (/etc/systemd/system/myapp.service):
    ```
    [Unit]
    Description=MyApp service
    After=network.target

    [Service]
    Type=simple
    User=appuser
    WorkingDirectory=/opt/myapp
    ExecStart=/opt/myapp/bin/start.sh
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    ```
  - After creating `systemd` unit:
    - `sudo systemctl daemon-reload`
    - `sudo systemctl enable --now myapp.service`
  - Inspect logs: `journalctl -u myapp -f`

- cgroups and resource limits
  - With systemd, you can limit CPU and memory via `CPUQuota=` and `MemoryMax=` in service files.

---

## 10. Scheduling & automation — crontab format

- Crontab format:
  - `MIN HOUR DOM MON DOW command`
  - Example: every day at 03:30:
    - `30 3 * * * /usr/local/bin/daily-backup.sh >> /var/log/daily-backup.log 2>&1`
- Environment and PATH
  - Cron runs with a minimal environment — use full paths or set PATH at top of crontab:
    - `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`
- systemd timers as an alternative
  - More flexible and integrated with systemd logging/users.

---

## 11. Monitoring & performance — interpreting data

- CPU vs IO bound
  - High `%wa` (I/O wait) in `top` indicates disk IO bottleneck. Check `iostat` and `iotop`.
  - High CPU but low `%wa` suggests CPU-bound processes.
- Memory observations
  - Linux uses free RAM for filesystem cache. Don't panic on "used" memory; check `free -h` and `available`.
- Example triage:
  - High load: `top` -> identify process -> `strace -p PID` (if stuck syscalls) -> `lsof -p PID` -> `tcpdump` if network-bound

---

## 12. Security — SELinux / AppArmor & SSH hardening

- SELinux modes
  - `getenforce` -> Enforcing, Permissive, Disabled
  - Check contexts: `ls -Z /var/www/html`
  - Change context: `sudo chcon -R -t httpd_sys_content_t /var/www/html` (for Apache)
  - Use `semanage fcontext -a -t httpd_sys_content_t "/srv/www(/.*)?"` and `restorecon -Rv /srv/www` for persistent mapping

- SSH best practices
  - Disable root login: `PermitRootLogin no`
  - Disable password auth: `PasswordAuthentication no`
  - Allow only specific users: `AllowUsers deploy ops`
  - Use `ssh-keygen -o -a 100 -t ed25519` for modern keys (argon-like KDF iterations via -a)

---

## 13. Shell scripting — robust patterns

- Strict mode:
  - `#!/usr/bin/env bash`
  - `set -euo pipefail`
  - `IFS=$'\n\t'`
- Trapping signals:
  - ```
    trap 'echo "Interrupted"; cleanup; exit 1' INT TERM
    ```
- Safe temp files:
  - `tmpdir=$(mktemp -d) || exit 1`
  - `rm -rf "$tmpdir"`
- Example: atomic update
  - Write to temp file, verify, then `mv` into place (mv is atomic on same FS).

---

## 14. LVM & storage operations — common tasks

- Adding a physical disk to VG:
  - `sudo pvcreate /dev/sdc1`
  - `sudo vgextend vg_data /dev/sdc1`
  - `sudo lvextend -L +100G /dev/vg_data/lv_data`
  - Resize filesystem:
    - ext4: `sudo resize2fs /dev/vg_data/lv_data`
    - xfs: `sudo xfs_growfs /mountpoint`
- Shrinking logical volumes is risky:
  - Always backup, unmount, fsck, shrink FS then lvreduce.

---

## 15. Compound examples & troubleshooting patterns

1. Web service 502 after deployment:
   - Check unit: `systemctl status myapp`
   - Logs: `journalctl -u myapp -n 200`
   - Confirm listening: `ss -tulnp | grep :8080`
   - Network: `curl -v http://localhost:8080/health`
   - If crash loop: review core dump or `strace` output.

2. Disk full unexpected:
   - `df -h` to find filesystem
   - `du -shx /* | sort -h` to find big directories (use `-x` to stay on same FS)
   - Check large log files: `ls -lh /var/log`
   - Clean rotated logs or truncate: `: > /var/log/large.log` (safe truncation)
   - Temporary space used by deleted but open files: `lsof +L1` shows deleted files still held open by processes; restart those processes.

---

## 16. Final tips & learning resources

- Practice on disposable VMs or containers (Docker/VMs) before touching production.
- Maintain clear, versioned runbooks for common operations.
- Use monitoring & alerting (Prometheus + Grafana, Nagios, etc.) rather than manual checks.
- Read `man` pages: `man tar`, `man systemctl`, and `man 5 fstab` for authoritative details.

---

