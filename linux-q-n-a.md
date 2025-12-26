# Linux Administration — Questions 1–145: Detailed Step‑by‑step Explanations, Examples & Verification

This README contains 145 commonly asked Linux administration questions. Each question is answered with:
- What it means (short conceptual explanation)
- Step-by-step commands (how to perform)
- Verification steps (how to confirm expected outcome)
- Common pitfalls / cautions
- Short interview-tip phrasing you can use
- Practical example where applicable

Use a disposable VM or snapshot for any destructive actions. Commands that change disks, LVM, partitions, or filesystems can destroy data.

---

Q1. How to set a password for a user that never expires?
- What: Make a user's password not require periodic change.
- Commands:
  - Set maximum password age to a very large value:
    sudo chage -M 99999 username
  - Alternatively set expiry to "never":
    sudo chage -I -1 -m 0 -M 99999 username
- Verification:
  - sudo chage -l username
    -> Confirm "Maximum number of days between password change: 99999"
- Pitfalls:
  - Avoid for human accounts if policy requires rotation. Service accounts may use this pattern.
- Interview tip:
  - "Use chage -M 99999, then verify with chage -l; prefer key-based auth for service accounts."
- Example:
  - sudo useradd svcbackup && sudo chage -M 99999 svcbackup && sudo chage -l svcbackup

---

Q2. Why `/etc/passwd` and `/etc/shadow` cannot be merged?
- What: Two files separate account metadata and sensitive password hashes.
- Explanation:
  - /etc/passwd: user account info (username, UID, GID, home, shell) — world-readable for many userland utilities.
  - /etc/shadow: password hashes and aging info — readable only by root (secure).
  - Combining them would expose password hashes to non-root processes and users.
- Verification:
  - ls -l /etc/passwd /etc/shadow
    -> /etc/passwd usually -rw-r--r--, /etc/shadow -rw------- (root only)
- Pitfalls:
  - Never change /etc/shadow permissions; editing should be done with passwd/chage/usermod or careful root editing.
- Interview tip:
  - "Separation prevents hash exposure; /etc/shadow restricts access to root, improving security."
- Example:
  - cat /etc/passwd | head -n 3
  - sudo cat /etc/shadow | head -n 3

---

Q3. List all files opened by a particular process
- What: Find files, sockets, pipes held by a process (useful when files are deleted but space not freed).
- Commands:
  - lsof -p <PID>
  - lsof -c sshd  # by process name
- Verification:
  - Confirm expected files listed and modes (r/w), look for "(deleted)" entries.
- Pitfalls:
  - Requires root to see files for processes owned by other users.
- Interview tip:
  - "I use lsof -p PID to find open descriptors and identify deleted-but-held files."
- Example:
  - sudo lsof -p $(pidof httpd | awk '{print $1}')

---

Q4. Unable to unmount filesystem — reasons
- What: Common causes and steps to resolve unmount failures.
- Steps & commands:
  - Check what is using mountpoint:
    lsof +f -- /mountpoint
    fuser -vm /mountpoint
  - Change directory out of mountpoint in all shells; kill or stop processes using it.
  - Lazy unmount:
    sudo umount -l /mountpoint
  - Force unmount (careful, limited):
    sudo umount -f /mountpoint
- Verification:
  - mount | grep /mountpoint or findmnt /mountpoint should show it's unmounted.
- Pitfalls:
  - For NFS, `umount -f` may hang; lazy unmount leaves resources until closed.
- Interview tip:
  - "Use fuser/lsof to find processes, then stop them; if necessary umount -l to detach safely."
- Example:
  - sudo fuser -vm /mnt/data && sudo kill -9 <PID> && sudo umount /mnt/data

---

Q5. Server takes more time after reboot — reasons
- What: Reasons why boot or post-reboot operations slow.
- Common causes:
  - FSCK running for many filesystems
  - Unavailable NFS mounts waiting to timeout
  - Services retrying due to misconfiguration (slow network, DB timeouts)
  - Hardware issues (disk errors, SCSI timeouts)
  - DNS or authentication delays (e.g., LDAP timeouts)
- Commands to investigate:
  - sudo journalctl -b
  - dmesg --ctime | less
  - systemd-analyze blame
  - systemd-analyze critical-chain
- Verification:
  - Identify slow units via systemd-analyze, check timestamps in journalctl -b.
- Pitfalls:
  - Logs may be verbose; filter by time or unit.
- Interview tip:
  - "I check journalctl -b and systemd-analyze blame to find long-starting services and then address the root cause."
- Example:
  - sudo systemd-analyze blame | head -n 20

---

Q6. Cannot create file though space exists — why?
- What: Disk space vs inodes vs quotas vs ro mount.
- Steps to diagnose:
  - Check space:
    df -h /mountpoint
  - Check inodes:
    df -i /mountpoint
  - Check mount options (read-only):
    mount | grep /mountpoint
  - Check quotas:
    quota -u username
- Verification:
  - If df -i shows 100% used, free inodes by removing many small files; if filesystem read-only, remount rw.
- Pitfalls:
  - Running out of inodes is common for many tiny files; resizing or recreation needed.
- Interview tip:
  - "I check both df -h and df -i; often inode exhaustion causes this."
- Example:
  - df -h /var && df -i /var

---

Q7. Check kernel routing table
- What: See system's IP routing decisions.
- Commands:
  - ip route show
  - route -n  (legacy)
- Verification:
  - A default route line like:
    default via 192.168.1.1 dev eth0 proto static
  - Specific network routes present.
- Pitfalls:
  - Multiple default routes or wrong metrics can cause traffic to go out wrong interface.
- Interview tip:
  - "Use ip route, then check `/etc/sysconfig/network-scripts/ifcfg-*` or netplan files for static routes configuration."
- Example:
  - ip route get 8.8.8.8

---

Q8. Sticky bit & s/S meaning
- What: Special permission bits and their effects.
- Explanation:
  - Sticky bit (t) on directories prevents users from deleting others' files (e.g., /tmp). Set with chmod +t /dir or chmod 1777 /tmp.
  - SUID (s on owner) lets an executable run with the file owner's permissions (chmod 4755 file).
  - SGID (s on group) runs file with group privileges or causes directory to inherit group (chmod 2755 dir).
  - 's' lowercase indicates execute bit set and SUID/SGID set; 'S' uppercase indicates SUID/SGID set but execute bit not set (often misconfiguration).
- Commands:
  - chmod +t /shared
  - chmod 4755 /usr/bin/someprog
- Verification:
  - ls -ld /tmp shows drwxrwxrwt for sticky bit.
  - ls -l file shows -rwsr-xr-x for SUID.
- Pitfalls:
  - SUID binaries are security-sensitive; avoid if unnecessary.
- Interview tip:
  - "Explain sticky bit use on world-writable directories and risk of SUID misconfigurations."
- Example:
  - mkdir /tmp/test && chmod 1777 /tmp/test && ls -ld /tmp/test

---

Q9. File for default gateway
- What: Where persistent default gateway is configured.
- Explanation:
  - On RHEL/CentOS with legacy network scripts, /etc/sysconfig/network-scripts/ifcfg-<interface> often contains GATEWAY=.
  - On modern systems with NetworkManager, use nmcli or NetworkManager keyfiles /etc/NetworkManager/system-connections/.
  - Current active route: ip route | grep default
- Commands:
  - grep GATEWAY /etc/sysconfig/network-scripts/ifcfg-*
  - nmcli connection show <name> | grep ipv4.gateway
- Verification:
  - ip route shows default via configured gateway
- Pitfalls:
  - Duplicate gateway entries can cause unpredictable routing.
- Interview tip:
  - "I verify with `ip route` and then check configuration files like ifcfg or NetworkManager profiles for persistence."
- Example:
  - ip route | grep default

---

Q10. Switch between run-levels / targets
- What: Change current system state (multi-user, graphical, rescue).
- Commands:
  - Switch immediately: sudo systemctl isolate multi-user.target
  - Set default permanently: sudo systemctl set-default graphical.target
  - Check current target: systemctl get-default
- Verification:
  - systemctl list-units --type target --state active
  - After isolate, runlevels/targets in effect reflect change (tty login vs GUI).
- Pitfalls:
  - isolate is immediate and can drop network or GUI unexpectedly; use carefully on remote machines.
- Interview tip:
  - "Use systemctl isolate for testing; `set-default` for permanent preference."
- Example:
  - sudo systemctl isolate rescue.target   # drops to rescue

---

Q11. NFS definition, port & configuration
- What: Network File System basics and setup.
- Explanation:
  - NFS typically uses TCP/UDP port 2049. rpcbind and related RPC services may use dynamic ports.
- Server setup (RHEL/CentOS):
  - sudo dnf install -y nfs-utils
  - sudo mkdir -p /srv/nfs/share && sudo chown nobody:nogroup /srv/nfs/share && sudo chmod 0777 /srv/nfs/share
  - Add to /etc/exports:
    /srv/nfs/share 10.0.0.0/24(rw,sync,no_root_squash)
  - sudo exportfs -rav
  - sudo systemctl enable --now nfs-server rpcbind
- Client mount:
  - sudo mount -t nfs server:/srv/nfs/share /mnt
  - Add to /etc/fstab for persistence: server:/srv/nfs/share /mnt nfs defaults 0 0
- Verification:
  - On server: sudo exportfs -v
  - On client: showmount -e server
  - mount | grep nfs on client should show mount
- Pitfalls:
  - Firewalls and SELinux contexts can block NFS; open required services and set booleans if needed.
- Interview tip:
  - "I ensure rpcbind and nfs-server are running and firewall rules allow NFS service; use exportfs -rav to refresh exports."
- Example:
  - sudo mount -t nfs4 server:/srv/nfs/share /mnt && touch /mnt/testfile && ls -l /mnt

---

Q12. nice & renice values
- What: Adjust process scheduling priority.
- Explanation:
  - nice value range: -20 (highest priority) to 19 (lowest priority). Only root can set negative (higher priority).
- Start a process with a nice value:
  - nice -n 10 command   # start with lower priority
- Change priority of running process:
  - renice -n 5 -p <PID>
- Verification:
  - ps -o pid,ni,cmd -p <PID>
- Pitfalls:
  - Setting extreme negative nice without understanding can starve other processes.
- Interview tip:
  - "Use nice for background batch jobs; renice to adjust priority after observing behavior."
- Example:
  - sleep 1000 & ; renice -n 10 -p $!

---

Q13. FTP vs TFTP
- What: Differences between these two file transfer protocols.
- Summary:
  - FTP: TCP-based, supports authentication, directory listings, data and control channels (ports 20/21), secure variants include FTPS/SFTP (SFTP is over SSH).
  - TFTP: UDP-based simple file transfer protocol, no authentication, used for PXE boot or simple device transfers, port 69.
- Verification:
  - ftp server (test with ftp client), tftp client to connect to TFTP server.
- Pitfalls:
  - TFTP is insecure for general use; only used for limited scenarios.
- Interview tip:
  - "TFTP is simple and unauthenticated (UDP/69) while FTP is stateful with authentication over TCP (21/20). Use SFTP for secure transfers."
- Example:
  - sudo apt install -y tftpd-hpa && sudo systemctl start tftpd-hpa ; then tftp localhost get testfile

---

Q14. Extend logical volume size
- What: Grow LV and filesystem to use additional space.
- Steps (XFS example):
  1. Ensure VG has free space: vgs or vgdisplay
  2. lvextend -L +5G /dev/vgname/lvname
  3. xfs_growfs /mountpoint
- Steps (ext4 example):
  1. lvextend -L +5G /dev/vgname/lvname
  2. resize2fs /dev/vgname/lvname
- Verification:
  - df -h /mountpoint should reflect increased size
  - lvs and lvdisplay show LV size changes
- Pitfalls:
  - Always grow LV first then filesystem. Backups recommended.
- Interview tip:
  - "Use lvextend then xfs_growfs for XFS; ext4 uses resize2fs."
- Example:
  - sudo lvextend -L +2G /dev/vgdata/lvdata && sudo xfs_growfs /data

---

Q15. Rollback packages after patching
- What: Undo package updates using package-manager transaction logs.
- Commands (yum):
  - yum history
  - yum history info <ID>
  - yum history undo <ID>
- Verification:
  - yum history list shows a new "undo" transaction; check package versions and service behavior.
- Pitfalls:
  - Undo may not revert database migrations or content changes performed by package scripts; always test in staging.
- Interview tip:
  - "I use yum history to find the transaction ID and yum history undo to roll back; verify services and test."
- Example:
  - sudo yum update -y ; sudo yum history ; sudo yum history undo <id>

---

Q16. rsync command & difference from scp
- What: Sync files efficiently between systems.
- Commands:
  - Basic: rsync -avz /local/dir/ user@remote:/remote/dir/
  - Preserve perms/owners: rsync -a (--archive)
  - Dry-run: rsync -avz --dry-run src dst
- Differences vs scp:
  - rsync transfers only deltas (changed blocks), supports resume, preserves metadata; scp copies full files each time and lacks delta transfer.
- Verification:
  - After initial run, modify small file and run rsync again; observe only that file transmitted.
- Pitfalls:
  - Incorrect trailing slash on source changes behavior (with/without slash).
- Interview tip:
  - "I use rsync for backups because it transfers only changed data and supports --delete and --bwlimit."
- Example:
  - rsync -avz /var/log/ remote:/backup/logs/

---

Q17. Reduce LV size
- What: Shrinking logical volumes and filesystems (dangerous).
- Steps for ext4 (safe sequence):
  1. Backup critical data or snapshot if available.
  2. Unmount: sudo umount /data
  3. fsck: sudo e2fsck -f /dev/vg/lv
  4. shrink filesystem: sudo resize2fs /dev/vg/lv 5G
  5. lvreduce -L 5G /dev/vg/lv
  6. mount and verify: sudo mount /data
- Verification:
  - df -h shows reduced size; e2fsck shows consistency.
- Pitfalls:
  - XFS cannot shrink. Shrinking without correct sequence causes data loss.
- Interview tip:
  - "Always take backups and shrink filesystem before lvreduce; XFS can't be reduced."
- Example:
  - sudo umount /data && sudo e2fsck -f /dev/vg/lvdata && sudo resize2fs /dev/vg/lvdata 8G && sudo lvreduce -L 8G /dev/vg/lvdata

---

Q18. Show line numbers in vim
- What: Enable line numbers for navigation.
- Commands:
  - In vim: :set number or :set nu
  - Make persistent: echo "set number" >> ~/.vimrc
- Verification:
  - Open a file and see line numbers displayed on the left.
- Pitfalls:
  - None significant.
- Interview tip:
  - "I enable line numbers and use ':set number' in vim; put in ~/.vimrc for persistence."
- Example:
  - vim +':set number' /etc/passwd

---

Q19. Fields of /etc/fstab
- What: Format and meaning of fstab fields.
- Fields:
  - <device> <mountpoint> <fstype> <options> <dump> <pass>
  - Example: UUID=abcd-1234 /data xfs defaults 0 0
- Verification:
  - sudo blkid shows UUID; sudo mount -a applies fstab; verify with mount | grep /data
- Pitfalls:
  - Incorrect fstab entries can prevent boot; comment problematic lines and use nofail for optional mounts.
- Interview tip:
  - "Use UUIDs for persistence; test with mount -a before rebooting."
- Example:
  - sudo blkid /dev/vg/lv && echo "UUID=$(blkid -s UUID -o value /dev/vg/lv) /data xfs defaults 0 0" | sudo tee -a /etc/fstab && sudo mount -a

---

Q20. Hard link vs soft link
- What: Differences and use-cases.
- Explanation:
  - Hard link: ln file link1 — points to same inode; deleting original name does not delete data until all hard links removed; cannot cross filesystems, cannot link directories normally.
  - Soft (symbolic) link: ln -s file link2 — stores a path to the target; can cross filesystems; breaks if target removed.
- Commands:
  - ln file file_hardlink
  - ln -s file file_symlink
- Verification:
  - ls -li file file_hardlink file_symlink  # shows inode numbers; hard link shares inode
- Pitfalls:
  - Symlinks can point to nonexistent targets (dangling).
- Interview tip:
  - "Hard links share inodes and are indistinguishable from original file; symlinks are pointers and can point across filesystems."
- Example:
  - echo hello >f && ln f f2 && ln -s f f3 && ls -li f f2 f3

---

Q21. How to check whether a port is listening?
- What: List listening sockets and the owning processes.
- Commands:
  - ss -tulnp   # modern
  - netstat -tulnp   # older systems
  - ss -ltnp | grep 8080  # specific port
- Verification:
  - Confirm output shows LISTEN state, local address:port, and PID/Program name.
- Pitfalls:
  - Requires root to see process names for sockets owned by other users.
- Interview tip:
  - "I use ss -tulnp to inspect listening services and ss -tnp host:port to verify specific ones."
- Example:
  - sudo ss -tulnp | grep ssh

---

Q22. Difference between `df` and `du`
- What: df shows filesystem-level usage; du shows directory/file usage.
- Commands:
  - df -h
  - du -sh /var/log
  - du -sh /*  # find heavy directories
- Verification:
  - Compare df -h and du -sh; if they differ significantly, check for deleted-but-open files (lsof | grep deleted).
- Pitfalls:
  - du can be slow on large directories; use --max-depth to narrow.
- Interview tip:
  - "Use df for partition usage and du to find which directories are consuming that space."
- Example:
  - df -h / && sudo du -shx /* | sort -h

---

Q23. What is journaling? Difference between ext2 and ext4
- What: Journaling filesystem ensures metadata consistency after crashes.
- Explanation:
  - ext2: no journaling — can be corrupted and require long fsck.
  - ext3/4: journaling for metadata (and optionally data) speeds recovery.
  - ext4 adds extents, larger filesystems, delayed allocation, and other enhancements over ext3/ext2.
- Verification:
  - tune2fs -l /dev/sdX | grep 'Filesystem features' shows 'has_journal' on ext3/4.
- Pitfalls:
  - Journaling has small overhead but improves recovery. ext4 defaults are usually optimal.
- Interview tip:
  - "Journaling logs changes, enabling quick and safe recovery; ext4 supports extents and generally better performance and features over ext2."
- Example:
  - sudo mkfs.ext4 /dev/loop0 && tune2fs -l /dev/loop0 | grep 'Filesystem features'

---

Q24. Linux boot process step-by-step
- What: Major boot stages.
- Steps:
  1. BIOS/UEFI POST (hardware init)
  2. Bootloader (GRUB2 on modern Linux) reads config and presents menu
  3. Kernel loads + initramfs (initramfs contains necessary modules and tools)
  4. Kernel mounts root filesystem and invokes init (systemd)
  5. systemd reads unit files, starts services and reaches target (multi-user/graphical)
  6. getty/login prompt or display manager presented
- Commands to inspect:
  - dmesg --ctime | less
  - journalctl -b
  - sudo systemd-analyze blame
- Verification:
  - systemd-analyze critical-chain shows ordering; journalctl -b provides logs per step.
- Pitfalls:
  - Missing drivers in initramfs cause early boot failures; rebuild with dracut if needed.
- Interview tip:
  - "I describe BIOS->GRUB->kernel->initramfs->systemd and mention tools like journalctl and systemd-analyze to debug."
- Example:
  - sudo systemd-analyze critical-chain

---

Q25. What are run-levels? (systemd targets)
- What: Mapping old SysV runlevels to systemd targets.
- Mapping:
  - 0: poweroff.target
  - 1: rescue.target (single-user)
  - 3: multi-user.target (text mode)
  - 5: graphical.target (GUI)
  - 6: reboot.target
- Commands:
  - systemctl get-default
  - systemctl isolate multi-user.target
- Verification:
  - systemctl list-units --type target | grep multi-user
- Pitfalls:
  - Isolating a target changes current state temporarily; set-default is persistent.
- Interview tip:
  - "systemd targets map to runlevels; use systemctl to manage and inspect them."
- Example:
  - sudo systemctl set-default multi-user.target && sudo systemctl get-default

---

Q26. Meaning of PCPU and JCPU in `w` command output
- What:
  - PCPU: percent CPU used by the current process.
  - JCPU: total CPU time used by all processes attached to the tty (the session).
- Commands:
  - w
  - top (shows per-process CPU usage)
- Verification:
  - Compare w output with top for specific PIDs.
- Pitfalls:
  - JCPU can be confusing; it aggregates session CPU time, not a single process.
- Interview tip:
  - "PCPU is process CPU usage; JCPU is session total (useful to see session-wide impact)."
- Example:
  - w && top -b -n1 | head -n 20

---

Q27. What is load average? How is it calculated?
- What: Load average is the average number of runnable or uninterruptible processes over time (1, 5, 15 minutes).
- Explanation:
  - Values are counts; on a single-core system, load avg > 1 means more demand than capacity.
  - On multi-core, compare to number of CPU cores (nproc).
- Commands:
  - uptime
  - cat /proc/loadavg
  - nproc
- Verification:
  - Correlate load averages with top and CPU utilization; high iowait often increases load.
- Pitfalls:
  - Load includes processes waiting for I/O (uninterruptible sleep), not just CPU usage.
- Interview tip:
  - "Compare load average to number of cores; investigate with top/vmstat/iostat to find CPU vs I/O pressure."
- Example:
  - uptime && nproc

---

Q28. What happens when you type URL in browser and press Enter?
- What: High-level network/IP/TCP/HTTP flow.
- Steps:
  1. DNS resolution (hosts file, DNS servers)
  2. TCP 3-way handshake to server on port 80/443
  3. TLS handshake if HTTPS
  4. HTTP request and server response
  5. Browser parses and renders HTML/CSS/JS; may fetch additional assets
- Commands to observe:
  - dig example.com
  - curl -v https://example.com
- Verification:
  - curl -I shows headers and response status; tcpdump or wireshark shows packet flow.
- Pitfalls:
  - DNS caching, firewall, or misconfigured reverse proxies can change observed behavior.
- Interview tip:
  - "Describe DNS→TCP→TLS→HTTP and mention debugging with dig and curl."
- Example:
  - curl -v https://www.google.com

---

Q29. Difference between RPM and YUM & how to check installed package
- What:
  - rpm is a low-level tool for installing/querying individual packages; yum/dnf are higher-level managers with repo and dependency handling.
- Commands:
  - rpm -qa | grep httpd
  - rpm -ql httpd
  - yum list installed | grep package
- Verification:
  - rpm -q <package> shows installed version; yum provides repo-based resolution.
- Pitfalls:
  - Using rpm -Uvh without dependency handling can leave package in inconsistent state; prefer yum/dnf when possible.
- Interview tip:
  - "Use rpm for queries and yum/dnf for full package management; check installed packages with rpm -qa."
- Example:
  - rpm -qa | grep kernel && yum list available | head

---

Q30. What is ACL? Why & how to set?
- What: Access Control Lists extend POSIX permissions to allow per-user/group fine-grained access.
- Commands:
  - setfacl -m u:john:rwx /data/file1   # grant rwx to user john
  - getfacl /data/file1
  - setfacl -b /data/file1  # remove all ACLs
- Verification:
  - getfacl shows ACL entries; test login as that user and attempt access.
- Pitfalls:
  - ACL complexity can confuse admins; keep documentation and use sparingly.
- Interview tip:
  - "Use setfacl to give specific users permissions without changing file owner or group."
- Example:
  - sudo setfacl -m u:$(whoami):rwx /tmp/myfile && getfacl /tmp/myfile

---

Q31. What is inode? What does it store?
- What: Inode is filesystem metadata object representing a file (not the name).
- Stored info:
  - Permissions, UID/GID, timestamps, size, block pointers, link count.
- Commands:
  - ls -i filename  # shows inode number
  - stat filename  # detailed inode info
- Verification:
  - stat outputs inode number, times, and block usage.
- Pitfalls:
  - Creating many small files can exhaust inodes even when space remains.
- Interview tip:
  - "Inodes store metadata; filenames are directory entries pointing to inodes."
- Example:
  - stat /etc/passwd

---

Q32. Can we schedule 2-second cron job?
- What: Cron's minimum granularity is 1 minute; not suitable for sub-minute tasks.
- Alternatives:
  - Use a loop script: while true; do /path/script; sleep 2; done
  - Use systemd timers with AccuracySec and OnUnitActiveSec for sub-minute scheduling.
- Verification:
  - Run loop script in background and monitor with ps/top.
- Pitfalls:
  - Busy-looping scripts can consume CPU; ensure sleep is present and script is efficient.
- Interview tip:
  - "Cron is minute-resolution; for sub-minute tasks use systemd timers or a loop with sleep."
- Example:
  - (while true; do date >> /tmp/log; sleep 2; done) &

---

Q33. How to check who rebooted system?
- What: Find records of reboots and who initiated them.
- Commands:
  - last reboot
  - last -x  # includes shutdown and runlevel changes
  - journalctl -b -1 --no-pager | grep -i reboot  # previous boot logs
- Verification:
  - last shows time and source; check /var/log/secure or audit logs to find the initiating user.
- Pitfalls:
  - If system restarted due to crash, 'who' might not show a manual user; use journal logs to find hints.
- Interview tip:
  - "I use last -x and journalctl to correlate reboot times with user actions."
- Example:
  - last -x | head -n 10

---

Q34. Command to set user password expiry date
- What: Set account expiry date (when account becomes unusable).
- Commands:
  - sudo chage -E 2025-12-31 username
  - sudo chage -l username  # verify
- Verification:
  - chage -l shows Account expires: date.
- Pitfalls:
  - chage -E accepts epoch-like values as well; ensure format correct.
- Interview tip:
  - "Use chage -E YYYY-MM-DD to set an absolute expiry; chage -l to confirm."
- Example:
  - sudo chage -E 2026-01-01 alice && sudo chage -l alice

---

Q35. Extract a single file from tar archive
- What: How to list and extract a specific file.
- Commands:
  - tar -tf backup.tar.gz  # list
  - tar -xzf backup.tar.gz path/inside/archive/file
- Verification:
  - ls path/inside/archive/file after extraction; compare checksum with archived version if available.
- Pitfalls:
  - Ensure the exact path inside archive is used; quoting patterns may be needed.
- Interview tip:
  - "Use tar -tf to get the exact path, then tar -xzf archive path to extract only that file."
- Example:
  - tar -xzf backup.tar.gz etc/hosts && ls -l etc/hosts

---

Q36. Command to check memory usage
- What: Inspect current RAM and swap usage.
- Commands:
  - free -h
  - top or htop for interactive view
  - vmstat 2 5  # sample stats
  - sar -r 1 5   # historical if sar enabled
- Verification:
  - free -h shows total/free/used and buffers/cache; use 'free -h' and subtract buffers/cache to get useful free memory.
- Pitfalls:
  - Misinterpreting cached memory as "used" — buffers/cache is reclaimable.
- Interview tip:
  - "Use free -h for summary and top for per-process consumption; check swap usage as well."
- Example:
  - free -h && vmstat 1 3

---

Q37. Check kernel version & architecture
- What: Get kernel release and machine architecture.
- Commands:
  - uname -r  # kernel version
  - uname -m  # architecture (x86_64, aarch64)
  - uname -a  # full info
- Verification:
  - Compare to /boot entries and installed kernel package lists.
- Pitfalls:
  - Running kernel may differ from installed packages (after upgrade until reboot).
- Interview tip:
  - "uname -r to show active kernel; check /boot for other kernels available."
- Example:
  - uname -r && rpm -qa | grep kernel

---

Q38. Search text inside multiple files
- What: Use grep to find patterns recursively, with options.
- Commands:
  - grep -Rin "error" /var/log/
  - grep -R --exclude-dir=.git -n "TODO" .
- Verification:
  - grep output lists file:line with matches; use -C to show context.
- Pitfalls:
  - Large directories may be slow; use ripgrep (rg) for speed when available.
- Interview tip:
  - "Use grep -Rin for case-insensitive recursive searches, or ack/rg for faster performance."
- Example:
  - grep -Rin --exclude-dir=node_modules "password" /srv 2>/dev/null

---

Q39. What is atime, mtime, ctime?
- What: File timestamps.
  - atime: last access time (reading)
  - mtime: last modification time of file contents
  - ctime: inode change time (permissions, ownership changes, link count updates)
- Commands:
  - stat filename  # shows atime, mtime, ctime
- Verification:
  - cat file -> stat shows updated atime (unless noatime mount)
- Pitfalls:
  - Frequent atime updates can cause performance issues; use noatime or relatime mount options when appropriate.
- Interview tip:
  - "ctime changes for any metadata change; mtime only for content updates; atime for last access and often disabled for performance."
- Example:
  - stat /etc/hosts ; cat /etc/hosts ; stat /etc/hosts

---

Q40. Delete log files older than 30 days
- What: Safe deletion pattern using find.
- Commands:
  - Dry-run: find /var/log -type f -mtime +30
  - Delete: find /var/log -type f -mtime +30 -delete
- Verification:
  - Re-run find to ensure no files older than 30 days remain.
- Pitfalls:
  - Ensure you don't delete files required by some services; prefer compressed archives or logrotate instead.
- Interview tip:
  - "Prefer logrotate with retention policies, but for cleanup use find with -mtime and test with a dry-run before deleting."
- Example:
  - sudo find /var/log -type f -mtime +30 -exec gzip {} \;

---

Q41. Delete logs whose ownership changed 45 minutes ago
- What: Use find with cmin to target files whose inode status (owner, metadata) changed N minutes ago.
- Commands:
  - Dry-run: find /var/log -cmin 45 -type f
  - Delete: find /var/log -cmin 45 -type f -delete
- Verification:
  - Confirm via find that matching files are removed.
- Pitfalls:
  - Be careful — -cmin checks inode change time; use -mmin to check modification time.
- Interview tip:
  - "Use find's time qualifiers (mmin, cmin) to target files by minute resolution."
- Example:
  - find /var/log -cmin 45 -type f -ls

---

Q42. User unable to list or cd into directory — reason?
- What: File permissions model for directories.
- Explanation:
  - To list directory contents you need read (r) on directory; to enter directory you need execute (x) bit. So lacking 'x' prevents cd.
- Commands:
  - ls -ld /path to view permissions
  - chmod +x /path to allow traversal
- Verification:
  - ls -ld /path shows x bit present; cd into directory succeeds.
- Pitfalls:
  - Grant minimal permissions needed (execute without read restricts listing but allows access if filenames known).
- Interview tip:
  - "Explain difference: read allows listing, execute allows traversal."
- Example:
  - mkdir /tmp/dtest && chmod 700 /tmp/dtest && sudo -u nobody cd /tmp/dtest  # fails; then chmod 711 and retry

---

Q43. What is run queue? How to check?
- What: Processes waiting to be scheduled on CPU (runnable queue).
- Commands:
  - vmstat 1 5  # procs r (run queue)
  - top  # look at load average and per-process statuses
- Verification:
  - vmstat's 'r' column indicates number of processes waiting for CPU; compare with nproc for saturation.
- Pitfalls:
  - High run queue compared to CPU count indicates CPU bottleneck; if high and CPU idle is low, it could be io-wait related.
- Interview tip:
  - "Use vmstat and top; if run queue > CPU cores, investigate CPU-bound processes."
- Example:
  - vmstat 1 10

---

Q44. Files responsible for default runlevel
- What: systemd default target configuration files.
- Files:
  - /etc/systemd/system/default.target (symlink to desired target)
  - /lib/systemd/system/ contains target unit files
- Commands:
  - readlink -f /etc/systemd/system/default.target
  - systemctl get-default
- Verification:
  - The symlink points to multi-user.target or graphical.target; changing default target and reboot will persist change.
- Pitfalls:
  - Incorrect default target pointing to reboot.target can cause immediate reboot loops.
- Interview tip:
  - "Default target is a symlink; check link and use set-default to change it."
- Example:
  - sudo systemctl set-default rescue.target && systemctl get-default

---

Q45. Difference between `yum update` and `yum upgrade`
- What: Behavioral differences depending on distro and yum version.
- Explanation:
  - Historically, 'yum update' updated packages but did not remove obsolete packages; 'yum upgrade' removed obsolete packages and could be more aggressive. Newer dnf behaves similarly to upgrade.
- Verification:
  - Run yum help update and yum help upgrade to see specific behavior for your system.
- Pitfalls:
  - upgrade may remove packages or perform more sweeping changes; always test in staging.
- Interview tip:
  - "Use update for safe updates and upgrade when you want to allow package obsolescence handling; read transaction before applying."
- Example:
  - sudo yum update --assumeno

---

Q46. How to downgrade a package?
- What: Install an earlier package version.
- Commands:
  - sudo yum downgrade <package-name>
  - Or sudo yum install package-<version>
- Verification:
  - rpm -q package shows installed version; check logs with yum history.
- Pitfalls:
  - Downgrading can break dependencies and should be tested; database migrations might be incompatible.
- Interview tip:
  - "Use yum downgrade or specify package-version; use yum history to track transactions."
- Example:
  - sudo yum downgrade httpd

---

Q47. Downgrade vs rollback
- What:
  - Downgrade: manually installing an older package version.
  - Rollback: using package manager's transaction history to revert an earlier transaction (e.g., yum history undo).
- Commands:
  - sudo yum history undo <ID>
- Verification:
  - yum history list and yum history info <ID> show transaction details and success of undo.
- Pitfalls:
  - Undo may not revert external changes done during upgrade scripts; verify application compatibility.
- Interview tip:
  - "Rollback is safer when available because it replays inverse operations; always inspect yum history first."
- Example:
  - sudo yum history && sudo yum history undo 45

---

Q48. What is filesystem? Examples
- What: A method of organizing data on storage devices.
- Examples:
  - ext4, xfs, btrfs, vfat, ntfs
- Commands:
  - df -T shows filesystem types
- Verification:
  - mount | column -t shows fs and options; tune2fs -l /dev/sda1 for ext filesystems.
- Pitfalls:
  - Different filesystems have different feature sets (journaling, snapshots); choose accordingly.
- Interview tip:
  - "Choose XFS for large files and ext4 for general use; btrfs offers snapshots and checksums but has trade-offs."
- Example:
  - mkfs.ext4 /dev/loop0 && mount /dev/loop0 /mnt && df -T /mnt

---

Q49. Difference between TCP and UDP
- What: Two core transport protocols.
- Explanation:
  - TCP: connection-oriented, reliable, ordered, flow control (used by HTTP, SSH).
  - UDP: connectionless, best-effort, low-overhead, used by DNS, NTP, VoIP.
- Verification:
  - Use ss -t for TCP sockets and ss -u for UDP; test with netcat: nc -u host port
- Pitfalls:
  - UDP apps must handle packet loss and ordering.
- Interview tip:
  - "TCP is reliable but heavier; UDP is low-latency but requires app-level reliability."
- Example:
  - nc -l 12345  # TCP listener ; nc -u -l 12346  # UDP listener

---

Q50. List directories only inside directory
- What: Commands to enumerate subdirectories in a path.
- Commands:
  - ls -d */
  - find . -maxdepth 1 -type d
- Verification:
  - Output shows only directories in current path.
- Pitfalls:
  - ls -d */ only works in shells with globbing enabled; use find for full control.
- Interview tip:
  - "Use find for scripts and ls -d for quick interactive listing."
- Example:
  - find /etc -maxdepth 1 -type d -ls

---

Q51. Difference between archive & compressed archive
- What:
  - Archive: groups multiple files into one (tar collects, but doesn't compress).
  - Compressed archive: archive plus compression (tar.gz, tar.bz2).
- Commands:
  - Create tar: tar -cf backup.tar /dir
  - Create compressed: tar -czf backup.tar.gz /dir
- Verification:
  - tar -tf archive lists contents; compare sizes with ls -lh.
- Pitfalls:
  - Compressed archives are single-file; random access to individual files requires extraction unless using specialized formats.
- Interview tip:
  - "tar just bundles; gzip/bzip2/xz compress. Use tar -tf to check contents."
- Example:
  - tar -czf etc.tgz /etc && tar -tf etc.tgz | head

---

Q52. Port timeout when connecting to remote host — reasons
- What: Connection attempt times out instead of being refused.
- Common causes:
  - Firewall blocking traffic (server or client side)
  - Service not listening on that port
  - Network routing issues or ACLs
  - SELinux or host-level filtering
- Commands to diagnose:
  - ss -tulnp | grep <port>
  - firewall-cmd --list-all or iptables -L
  - traceroute host
  - tcpdump -n -i any port <port>
- Verification:
  - If ss shows LISTEN and firewall allows port but connection still times out, use tcpdump to see if packets arrive.
- Pitfalls:
  - Some cloud security groups block ports before packets reach host (check cloud console).
- Interview tip:
  - "Check if service listens, firewall rules, cloud security groups, and run tcpdump to confirm packet arrival."
- Example:
  - sudo ss -ltnp | grep 8080 ; sudo firewall-cmd --list-ports

---

Q53. What is RAID? Types?
- What: Redundant Array of Independent Disks provides performance and/or redundancy by combining disks.
- Types (common):
  - RAID0: striping (performance, no redundancy)
  - RAID1: mirroring (redundancy)
  - RAID5: striping with parity (single-disk fault tolerance)
  - RAID6: double parity (two-disk fault tolerance)
  - RAID10: mirror of stripes (best redundancy/performance tradeoff)
- Software tool:
  - mdadm for Linux software RAID
- Commands:
  - Create RAID1: mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
  - Monitor: cat /proc/mdstat ; mdadm --detail /dev/md0
- Verification:
  - /proc/mdstat shows array state; mount md device and verify data mirrored.
- Pitfalls:
  - RAID is not a backup; replace failing disks promptly and rebuild.
- Interview tip:
  - "RAID provides availability/performance but not a substitute for backups; use RAID + backups."
- Example:
  - sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/loop1 /dev/loop2 && cat /proc/mdstat

---

Q54. `df` shows /opt full even after deleting 10GB logs — why?
- What: Open file handles hold deleted files still using space.
- Diagnosis:
  - lsof | grep deleted
  - lsof +L1  # list open files that have been unlinked
- Fixes:
  - Restart process holding file, or truncate file by redirecting to FD: > /proc/<PID>/fd/<FD>
- Verification:
  - df -h should show freed space after process restart or truncation; lsof no longer shows deleted files.
- Pitfalls:
  - Truncating a file without coordination may impact service; restarting is safer albeit causing downtime.
- Interview tip:
  - "I check lsof for deleted files and restart the process or truncate file descriptors to free space."
- Example:
  - sudo lsof | grep deleted ; identify PID, sudo systemctl restart that-service ; df -h

---

Q55. Why IP instead of MAC?
- What: Addressing at OSI layers and routing.
- Explanation:
  - MAC addresses operate at Layer 2 (link-layer) and are only relevant within broadcast domains. Routers forward Layer 3 (IP) packets across networks; IP is routable and supports hierarchical addressing.
- Verification:
  - arp -a shows MAC entries for local subnet; ip route shows routing decisions based on IP.
- Pitfalls:
  - MAC addresses can be spoofed; do not rely solely on MAC addresses for authentication.
- Interview tip:
  - "MAC is local link-layer; IP enables routing across networks."
- Example:
  - arp -n ; ip route show

---

Q56. Types of backups
- What: Common backup strategies and their use-cases.
- Types:
  - Full backup: entire dataset; easiest restore, highest space/time cost.
  - Incremental: backs up changes since last backup (full or incremental) — small and fast, restore needs chain.
  - Differential: backs up changes since last full backup — faster restore than incremental but grows over time.
  - Snapshot: filesystem or volume-level points-in-time (LVM, ZFS, btrfs).
  - Image-based: disk images (e.g., dd, backup software) capturing full disk.
- Verification:
  - Test restores regularly and verify checksums/integrity (e.g., rsync --checksum).
- Pitfalls:
  - Not testing restores leads to undetected failures; also, incremental chains can become long and fragile.
- Interview tip:
  - "Always test restores; use snapshots for quick restores and full backups for long-term retention."
- Example:
  - rsync -av --delete /data/ /backup/data/  # simple file-level backup

---

Q57. Find port number used by service
- What: Determine which ports a service process uses.
- Commands:
  - ss -tulpn | grep <service>
  - systemctl status <service> may include listening ports
  - grep -R "Listen" /etc/<service>/* or check service config files
- Verification:
  - ss output indicates process name and PID; netstat -tulpen shows PID/owner.
- Pitfalls:
  - Some services bind to localhost only (127.0.0.1), so remote connectivity won't work.
- Interview tip:
  - "Use ss or netstat to map listening sockets to services, and inspect config for bound addresses."
- Example:
  - sudo ss -tulpn | grep httpd

---

Q58. Troubleshoot I/O issues
- What: Identify and resolve disk I/O bottlenecks.
- Tools & commands:
  - iostat -xz 1  # per-device I/O stats: await, svctm, util%
  - iotop -oP    # per-process I/O usage
  - dstat -dny 1  # combined stats
  - vmstat 1 5    # overall system wait and run queue
- Steps:
  1. Check device utilization (%util) and await (ms)
  2. Identify heavy processes with iotop
  3. Investigate filesystem-level issues (metadata storms, many small files)
  4. Consider rebalancing, moving hot IO to SSD, or adjusting IO scheduler
- Verification:
  - Reduced await and %util after remedial action; iotop shows lowered per-process I/O.
- Pitfalls:
  - Deleting many files may produce high I/O itself; plan maintenance windows.
- Interview tip:
  - "I use iostat to detect high await and iotop to find heavy processes; then mitigate by throttling or relocating IO."
- Example:
  - sudo iostat -xz 1 10 && sudo iotop -oPa

---

Q59. Display system summary & utilization
- What: High-level overview of CPU, memory, disk, and processes.
- Commands:
  - top / htop
  - vmstat 1
  - sar -u 1 5 (if sysstat is installed)
  - dstat
- Verification:
  - Observe CPU usage, load average, free memory, IO wait over time.
- Pitfalls:
  - Instant snapshots can be misleading; monitor over time for trends.
- Interview tip:
  - "Use top for real-time, sar/vmstat for historical and continuous observation."
- Example:
  - top -b -n1 | head -n 20

---

Q60. Display process details for specific user
- What: List processes owned by a user.
- Commands:
  - ps -u username -f
  - pgrep -u username -l
- Verification:
  - ps output shows UID, PID, %CPU, %MEM, and command used.
- Pitfalls:
  - Be careful with scripts that parse ps output; prefer pgrep for scripts.
- Interview tip:
  - "Use ps -u for a detailed list or pgrep to get just PIDs for automation."
- Example:
  - ps -u syslog -f

---

Q61. How to kill a process inside the `top` interactive screen?
- What: Use top's interactive kill feature.
- Steps:
  1. Run top
  2. Press k
  3. Type PID and press Enter
  4. Enter signal number (15 for SIGTERM, 9 for SIGKILL)
- Commands alternative:
  - kill -15 <PID>  # graceful
  - kill -9 <PID>   # force
- Verification:
  - ps -p <PID> should no longer show process (or show Z for a short moment if being reaped).
- Pitfalls:
  - SIGKILL may leave temporary files or corrupted state.
- Interview tip:
  - "Prefer SIGTERM first; use SIGKILL only when process doesn't exit."
- Example:
  - top -> k -> <PID> -> 15

---

Q62. Gracefully vs forcefully terminating a process
- What: Difference between SIGTERM (15) and SIGKILL (9).
- Explanation:
  - SIGTERM (15): requests process to shutdown gracefully; allows cleanup and releasing resources.
  - SIGKILL (9): immediate termination; cannot be caught; may leave resources in inconsistent state.
- Commands:
  - kill -15 PID
  - kill -9 PID
- Verification:
  - After SIGTERM, process may cleanly stop; after SIGKILL, process disappears immediately and may require manual cleanup.
- Pitfalls:
  - Using kill -9 on daemons that manage important files or sockets can cause data loss; prefer orderly shutdown.
- Interview tip:
  - "Always try SIGTERM first and allow process to exit gracefully; use SIGKILL if it's unresponsive."
- Example:
  - kill -TERM <PID> && sleep 5 && kill -0 <PID> || echo "Process exited"

---

Q63. Find PID and details of running process
- What: Identify process IDs and inspect their environment.
- Commands:
  - pidof httpd  # prints PIDs
  - ps -fp <PID>  # full-format for specific PID
  - ps aux | grep process_name
  - ls -l /proc/<PID> to inspect files
- Verification:
  - ps output includes start time, command line, and resource usage.
- Pitfalls:
  - Grep may match the grep command itself; use pgrep -f or ps aux | grep [p]rocess.
- Interview tip:
  - "Use pidof for PIDs and ps -fp PID to inspect; check /proc/PID for more info like environ and fd."
- Example:
  - pidof sshd && ps -fp $(pidof sshd)

---

Q64. Check routing table
- What: See how packets are routed out of the machine.
- Commands:
  - ip route show
  - ip -6 route show  # IPv6
  - route -n  # legacy
- Verification:
  - Traceroute to a host reveals path uses expected gateway.
- Pitfalls:
  - Multiple default routes can cause asymmetric routing; use metric settings to control precedence.
- Interview tip:
  - "Use ip route to inspect and ip route add/replace to change routes; verify with traceroute and tcpdump."
- Example:
  - ip route get 8.8.8.8

---

Q65. Install new kernel & set default
- What: Add new kernel package and select it in GRUB.
- Commands:
  - sudo dnf install kernel  # installs latest kernel package
  - sudo grep ^menuentry /boot/grub2/grub.cfg  # list entries
  - sudo grub2-set-default 0  # set index 0 as default
  - sudo grub2-mkconfig -o /boot/grub2/grub.cfg  # regenerate config if needed
- Verification:
  - Reboot and uname -r should show the new kernel version.
- Pitfalls:
  - Ensure kernel modules and initramfs are present for the new kernel; third-party drivers may need rebuilding.
- Interview tip:
  - "Install kernel via package manager and use grub2-set-default and grub2-mkconfig to persist selection; reboot to test."
- Example:
  - sudo dnf install kernel-$(uname -r)  # example to reinstall current kernel

---

Q66. What is LVM and why use it?
- What: Logical Volume Manager for flexible, logical storage abstraction.
- Benefits:
  - Resize volumes online (grow), create snapshots, aggregate physical disks into volume groups, migrate extents across physical volumes.
- Commands:
  - pvcreate /dev/sdb ; vgcreate vgdata /dev/sdb ; lvcreate -n lvdata -L 10G vgdata
- Verification:
  - pvs, vgs, lvs show inventory and sizes.
- Pitfalls:
  - Snapshots consume space; plan for snapshot sizing. Reducing volumes is risky and requires proper sequence and backups.
- Interview tip:
  - "LVM offers flexibility for dynamic growth and snapshots; ideal for systems requiring online storage changes."
- Example:
  - sudo pvcreate /dev/loop0 && sudo vgcreate vgtest /dev/loop0 && sudo lvcreate -n data -L 1G vgtest

---

Q67. Components of LVM and how to check them
- What: PV, VG, LV definitions and commands.
- Commands:
  - pvs or pvdisplay
  - vgs or vgdisplay
  - lvs or lvdisplay
- Verification:
  - pvs shows PVs with VG association; lvs shows LV path, size, and attributes.
- Pitfalls:
  - Mislabeling disks can cause data loss; ensure PVs are created on intended devices.
- Interview tip:
  - "Describe PV->VG->LV mapping and use pvs/vgs/lvs to inspect."
- Example:
  - sudo pvs ; sudo vgs ; sudo lvs

---

Q68. Initialize a PV
- What: Prepare a block device for LVM usage.
- Commands:
  - sudo pvcreate /dev/sdb
- Verification:
  - sudo pvs should list /dev/sdb with VG set to none (until added).
- Pitfalls:
  - pvcreate obliterates existing partition data; double-check device path.
- Interview tip:
  - "Use pvcreate on a clean block device and verify with pvs."
- Example:
  - sudo pvcreate /dev/loop1 && pvs

---

Q69. Create VG with specific PE size
- What: Create a volume group specifying physical extent (PE) size (affects allocation granularity).
- Commands:
  - sudo vgcreate -s 16M vgdata /dev/sdb
  - sudo vgdisplay vgdata  # verify 'PE Size'
- Verification:
  - vgdisplay shows "PE Size (KByte)" as specified.
- Pitfalls:
  - PE size affects maximum LV granularity and compatibility with some operations; choose appropriately for expected resize patterns.
- Interview tip:
  - "Set PE size when you need specific allocation granularity; typical sizes are 4M or 16M."
- Example:
  - sudo vgcreate -s 8M vgtmp /dev/loop2 && vgdisplay vgtmp | grep 'PE Size'

---

Q70. What is PE and LE?
- What:
  - PE (Physical Extent): smallest allocatable chunk of space in a VG.
  - LE (Logical Extent): an extent inside an LV; maps 1:1 to a PE.
- Explanation:
  - 1 LE = 1 PE; choosing PE size influences granularity and VG capacity math.
- Verification:
  - vgs shows PE Size and Total PE count.
- Pitfalls:
  - Small PE sizes give more granularity but larger metadata; choose balanced value.
- Interview tip:
  - "PE is VG allocation unit; LE is unit inside LV mapping to PE."
- Example:
  - vgs --units m && vgdisplay vgdata | grep 'PE Size'

---

Q71. Create LV with and without PE specification
- What: Create logical volume using size or percentage of free extents.
- Commands:
  - By size: sudo lvcreate -n lvdata -L 5G vgdata
  - By extents: sudo lvcreate -n lvdata -l 100%FREE vgdata
- Verification:
  - lvs and lvdisplay show LV's size and extents used.
- Pitfalls:
  - Using 100%FREE consumes all VG capacity; be intentional.
- Interview tip:
  - "Use -L for absolute size and -l for extents/percentage of free space."
- Example:
  - sudo lvcreate -n lv1 -l 50%FREE vgdata && lvs

---

Q72. Format LV & mount permanently
- What: Create filesystem on LV and add to fstab by UUID.
- Commands:
  - sudo mkfs.xfs /dev/vgdata/lvdata  # or mkfs.ext4
  - sudo mkdir -p /data
  - sudo blkid /dev/vgdata/lvdata
  - Add to /etc/fstab: UUID=<uuid> /data xfs defaults 0 0
  - sudo mount -a
- Verification:
  - df -h /data shows mounted filesystem; cat /etc/mtab or findmnt.
- Pitfalls:
  - Using wrong UUID can mount incorrect device; use blkid to find correct UUID.
- Interview tip:
  - "Format the LV, use blkid to get UUID, add to fstab and mount -a to verify."
- Example:
  - sudo mkfs.ext4 /dev/vgtest/lvtest && sudo blkid /dev/vgtest/lvtest && echo "UUID=$(blkid -s UUID -o value /dev/vgtest/lvtest) /mnt ext4 defaults 0 0" | sudo tee -a /etc/fstab && sudo mount -a

---

Q73. Extend or reduce VG size
- What: Add or remove physical volumes to/from a VG.
- Extend:
  - sudo vgextend vgdata /dev/sdc
- Reduce:
  - sudo vgreduce vgdata /dev/sdc  # only if pv has no extents in use
  - If extents in use: sudo pvmove /dev/sdc /dev/sdd then vgreduce
- Verification:
  - vgs shows free PE and PVs associated.
- Pitfalls:
  - vgreduce will fail if extents still assigned; always pvmove first.
- Interview tip:
  - "To shrink a VG, migrate extents with pvmove before vgreduce."
- Example:
  - sudo pvcreate /dev/loop3 && sudo vgextend vgdata /dev/loop3 && vgs

---

Q74. Extend LV and resize filesystem
- What: Grow LV and filesystem; process differs by fs type.
- Steps:
  - sudo lvextend -L +5G /dev/vg/lv
  - If XFS: sudo xfs_growfs /mountpoint
  - If ext4: sudo resize2fs /dev/vg/lv
- Verification:
  - df -h /mountpoint; lvs shows LV size increased.
- Pitfalls:
  - For XFS, filesystem must be mounted to grow; for ext4, resizing can be done while mounted (online) in many kernels but safer to unmount for shrink (not grow).
- Interview tip:
  - "lvextend followed by xfs_growfs or resize2fs depending on fs."
- Example:
  - sudo lvextend -L +1G /dev/vgtest/lvtest && sudo xfs_growfs /mnt/test

---

Q75. Migrate LV data from faulty PV to another PV
- What: Move extents off a failing physical volume within a VG.
- Steps:
  - Add new PV to VG: sudo pvcreate /dev/sdx && sudo vgextend vgdata /dev/sdx
  - Migrate extents: sudo pvmove /dev/sdb /dev/sdx
  - Remove bad PV: sudo vgreduce vgdata /dev/sdb
- Verification:
  - pvs shows allocation on new PV; vgreduce succeeds.
- Pitfalls:
  - pvmove can take a long time; ensure source and destination have sufficient free space and IO capacity.
- Interview tip:
  - "Use pvmove to migrate extents; then vgreduce to remove the faulty PV."
- Example:
  - sudo pvmove /dev/loop1 /dev/loop2 && sudo vgreduce vgtest /dev/loop1

---

Q76. Remove LV, VG, PV
- What: Cleanly remove logical volumes and associated groups/volumes.
- Steps:
  - Unmount filesystem: sudo umount /data
  - Remove LV: sudo lvremove /dev/vgdata/lvdata
  - Remove VG: sudo vgremove vgdata
  - Remove PV signature: sudo pvremove /dev/sdb
- Verification:
  - lvs, vgs, pvs no longer list removed entities.
- Pitfalls:
  - lvremove destroys data; backup if needed.
- Interview tip:
  - "Unmount first, lvremove, vgremove, then pvremove to clean device metadata."
- Example:
  - sudo umount /data && sudo lvremove -f /dev/vgtest/lvtest && sudo vgremove -f vgtest && sudo pvremove /dev/loop1

---

Q77. Superuser, regular user, service user
- What: Types of accounts and their purposes.
- Explanation:
  - Superuser (root): UID 0 with full privileges.
  - Regular user: login accounts for humans.
  - Service user: non-login account used to run daemons with minimal privileges (e.g., apache, mysql).
- Verification:
  - getent passwd | egrep 'root|apache'
- Pitfalls:
  - Running services as root increases risk; prefer least-privilege service accounts.
- Interview tip:
  - "Adopt principle of least privilege: services should run as dedicated accounts with minimal rights."
- Example:
  - sudo useradd -r -s /sbin/nologin svcapp  # create a system/service user

---

Q78. File that stores user details & passwords
- What:
  - /etc/passwd: user account info (username, UID, GID, home, shell)
  - /etc/shadow: encrypted password hashes and aging, readable only by root
- Commands:
  - cat /etc/passwd | head
  - sudo cat /etc/shadow | head
- Verification:
  - Permissions: ls -l /etc/passwd /etc/shadow
- Pitfalls:
  - Do not expose /etc/shadow; only root should read it.
- Interview tip:
  - "User metadata in /etc/passwd; password hashes and aging in /etc/shadow."
- Example:
  - grep '^root' /etc/passwd && sudo grep '^root' /etc/shadow

---

Q79. Add, delete, modify user
- What: Basic user account lifecycle commands.
- Commands:
  - Add: sudo useradd -m username ; sudo passwd username
  - Delete with home: sudo userdel -r username
  - Modify shell: sudo usermod -s /bin/bash username
- Verification:
  - id username shows UID & groups; ls /home/username to check home creation.
- Pitfalls:
  - userdel -r permanently removes home and mail; ensure backups if necessary.
- Interview tip:
  - "Use useradd -m to create home; usermod to change attributes; userdel -r to delete with caution."
- Example:
  - sudo useradd -m testuser && sudo passwd testuser && id testuser

---

Q80. Lock and unlock user
- What: Temporarily disable account login.
- Commands:
  - Lock: sudo passwd -l username
  - Unlock: sudo passwd -u username
- Verification:
  - passwd -S username shows status (L for locked, P for password set).
- Pitfalls:
  - passwd -l prepends '!' to password field; some automation tools may require explicit unlocking.
- Interview tip:
  - "Lock with passwd -l to prevent login quickly; unlock with passwd -u when ready."
- Example:
  - sudo passwd -l testuser && sudo passwd -S testuser

---

Q81. Use of `chage` & force password change
- What: chage controls password aging and expiry for users.
- Commands:
  - sudo chage -l user1  # list aging info
  - sudo chage -d 0 user1  # force change at next login
- Verification:
  - chage -l shows last changed and password expiry; logging in should prompt for new password.
- Pitfalls:
  - Forcing password change may lock out scripts using static passwords; account for automation.
- Interview tip:
  - "chage -d 0 forces password change next login; use chage -l to audit user aging."
- Example:
  - sudo chage -d 0 deploy

---

Q82. Restrict login hours using PAM time
- What: Use pam_time.so to enforce allowed login hours for users.
- Steps:
  - Edit /etc/security/time.conf and add rule:
    login;*;user1;!Al2000-0700  # deny 20:00–07:00
  - Ensure /etc/pam.d/sshd includes: account required pam_time.so
- Verification:
  - Attempt login during restricted time; check /var/log/secure or journalctl for PAM denial messages.
- Pitfalls:
  - PAM config syntax errors can block logins; always test with another session open.
- Interview tip:
  - "Use pam_time.so plus /etc/security/time.conf to enforce login time windows."
- Example:
  - echo 'login;*;alice;!Al2000-0700' | sudo tee -a /etc/security/time.conf

---

Q83. Grant sudo privilege
- What: Allow a user to execute commands as root.
- Commands:
  - Add to wheel group (RHEL): sudo usermod -aG wheel user1
  - Edit sudoers (safe): sudo visudo and add user1 ALL=(ALL) ALL
- Verification:
  - sudo -l -U user1 shows allowed commands; su - user1 && sudo whoami returns root.
- Pitfalls:
  - Editing /etc/sudoers incorrectly can break sudo access; use visudo to validate.
- Interview tip:
  - "Prefer group-based sudo (wheel/sudo) for easier management; use visudo to edit."
- Example:
  - sudo usermod -aG wheel devuser && su - devuser -c 'sudo -l'

---

Q84. Difference between groups and id commands
- What:
  - groups username prints the group names a user belongs to.
  - id username shows uid, gid, and numerical group IDs plus supplementary groups.
- Commands:
  - groups alice
  - id alice
- Verification:
  - Compare both outputs; id shows numeric IDs.
- Pitfalls:
  - groups may show only name list; id is more detailed.
- Interview tip:
  - "Use id for UID/GID detail and groups for quick group name list."
- Example:
  - groups $(whoami) && id $(whoami)

---

Q85. Add, modify, delete group
- What: Group management commands.
- Commands:
  - Add: sudo groupadd dev
  - Modify (rename): sudo groupmod -n newname dev
  - Delete: sudo groupdel dev
- Verification:
  - getent group dev or /etc/group entry shows changes.
- Pitfalls:
  - Deleting group that is primary for users is dangerous; reassign group membership before deletion.
- Interview tip:
  - "Use groupmod for renaming; ensure no users rely on group being removed."
- Example:
  - sudo groupadd testgrp && getent group testgrp

---

Q86. Remove user from group
- What: Remove a user's membership in a supplementary group.
- Commands:
  - sudo gpasswd -d user1 dev
  - Or edit /etc/group (careful)
- Verification:
  - id user1 should no longer list the group; groups user1 shows updated list.
- Pitfalls:
  - usermod -G without -a overwrites group list; always use -aG to append.
- Interview tip:
  - "Use gpasswd -d to remove safely; check id afterwards."
- Example:
  - sudo gpasswd -d alice developers && id alice

---

Q87. Add user to secondary group without overwriting
- What: Append a group to a user's supplementary groups.
- Commands:
  - sudo usermod -aG dev user1
- Verification:
  - groups user1 or id user1 includes dev
- Pitfalls:
  - Omitting -a will overwrite supplementary groups; dangerous.
- Interview tip:
  - "Always use -aG when adding groups to avoid clobbering existing memberships."
- Example:
  - sudo usermod -aG docker $(whoami) && id $(whoami)

---

Q88. Use of `/etc/profile` and `/etc/login.defs`
- What:
  - /etc/profile: system-wide shell startup file for login shells; set env vars, PATH, umask.
  - /etc/login.defs: system-wide defaults for useradd and login-related policies (password ageing, UID ranges).
- Commands:
  - grep -E 'UMASK|PATH' /etc/profile
  - grep -E 'PASS_MAX_DAYS|UID_MIN' /etc/login.defs
- Verification:
  - New login shells pick up environment variables set in /etc/profile.
- Pitfalls:
  - Changes to /etc/profile require re-login to take effect; prefer per-user files for user-specific settings.
- Interview tip:
  - "Use /etc/profile for global env settings; /etc/login.defs controls user creation defaults and password policies."
- Example:
  - echo 'export MYVAR=1' | sudo tee -a /etc/profile && su - $USER -c 'echo $MYVAR'

---

Q89. What is SUID, SGID, Sticky bit?
- What:
  - SUID (set-user-ID): executable runs with file owner's privileges.
  - SGID (set-group-ID): executables run with file's group privileges; directories inherit group ID for new files.
  - Sticky bit: on directories prevents deletion by non-owners.
- Commands:
  - chmod 4755 /path/to/exe  # set SUID
  - chmod 2755 /path/to/dir  # set SGID on directory
  - chmod +t /shared  # sticky bit
- Verification:
  - ls -l shows s or t bits set: -rwsr-xr-x or drwxrwxrwt
- Pitfalls:
  - SUID programs can be exploited; audit and limit to necessary binaries.
- Interview tip:
  - "SUID/SGID used to elevate specific program privileges; sticky bit protects shared directories like /tmp."
- Example:
  - sudo chmod 4755 /usr/bin/sudo && ls -l /usr/bin/sudo

---

Q90. Set standard permissions
- What: Typical permission modes for files and scripts.
- Commands:
  - chmod 755 script.sh  # owner rwx, group/others rx
  - chmod 644 file.txt   # owner rw, group/others r
- Verification:
  - ls -l shows permissions; run script as owner/other to confirm execution permission.
- Pitfalls:
  - Avoid setting 777 on sensitive directories; security risk.
- Interview tip:
  - "Use 755 for executable scripts owned by root, 644 for data files."
- Example:
  - touch script.sh && chmod 755 script.sh && ls -l script.sh

---

Q91. What is umask & how to set permanently?
- What: umask defines default permission mask when creating files/directories.
- Explanation:
  - umask 022 results in file perms 644 and directories 755 by default.
- Commands:
  - umask 022
  - To persist: add umask 022 to /etc/profile or ~/.bashrc
- Verification:
  - touch newfile ; ls -l newfile shows permissions influenced by umask.
- Pitfalls:
  - Avoid overly permissive umask (e.g., 000) in multi-user systems.
- Interview tip:
  - "Set umask 027 for more privacy in multi-user servers to prevent group/other access."
- Example:
  - umask 027 ; touch /tmp/testfile ; ls -l /tmp/testfile

---

Q92. Purpose of fdisk command
- What: fdisk is an interactive partition table manipulator for MBR/GPT (legacy prefer parted for GPT).
- Commands:
  - sudo fdisk /dev/sdb  # interactive create/delete partitions
  - sudo fdisk -l /dev/sdb  # list partitions
- Verification:
  - lsblk shows partition layout; parted -l shows partitions with GPT data.
- Pitfalls:
  - fdisk is dangerous; ensure correct device; writing changes modifies partition table.
- Interview tip:
  - "Use fdisk for MBR and parted for GPT; always double-check device paths."
- Example:
  - sudo fdisk -l

---

Q93. List partitions on Linux system
- What: Enumerate block devices and their partitions.
- Commands:
  - lsblk -f
  - fdisk -l
  - parted -l
- Verification:
  - lsblk shows device tree with mountpoints; blkid shows FS UUIDs.
- Pitfalls:
  - loop devices may show temporary partitions used for testing.
- Interview tip:
  - "Use lsblk for quick tree view and blkid to map UUIDs for fstab entries."
- Example:
  - lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,UUID

---

Q94. Detect new disk without rebooting
- What: Force kernel rescan for SCSI devices or partitions.
- Commands:
  - echo "- - -" | sudo tee /sys/class/scsi_host/host0/scan  # replace host0 appropriately
  - sudo partprobe  # ask kernel to re-read partition table
  - Use rescan-scsi-bus script (sg3_utils) for multiple hosts
- Verification:
  - lsblk shows the new device; dmesg shows SCSI device detection logs.
- Pitfalls:
  - host index differs; read /sys/class/scsi_host to find correct hostX; for NVMe, echo 1 > /sys/class/block/nvme0n1/device/rescan
- Interview tip:
  - "Use echo '- - -' > /sys/class/scsi_host/hostX/scan or partprobe; check dmesg for confirmation."
- Example:
  - for h in /sys/class/scsi_host/host*; do echo "- - -" | sudo tee $h/scan; done

---

Q95. Difference between primary & logical partitions
- What: Partitioning concepts in MBR scheme.
- Explanation:
  - MBR supports up to 4 primary partitions; one may be an extended partition containing multiple logical partitions, overcoming the 4-partition limit.
  - Logical partitions exist inside the extended partition and numbered starting from 5 (e.g., /dev/sda5).
- Verification:
  - fdisk -l shows types and numbering; parted prints partition table scheme.
- Pitfalls:
  - GPT does not use primary/logical distinction; GPT supports many partitions.
- Interview tip:
  - "Primary partitions are top-level; logical partitions reside within extended to allow more partitions under MBR limits."
- Example:
  - sudo fdisk -l /dev/sda

---

Q96. Difference between MBR & GPT
- What: Partition table formats.
- Comparison:
  - MBR: 2TB limit, 4 primary partition limit, legacy BIOS boot support.
  - GPT: uses UEFI, supports large disks (ZB scale), many partitions (typically 128 in Linux), includes redundant headers for recovery.
- Verification:
  - parted /dev/sda print or parted -l shows msdos (MBR) vs gpt.
- Pitfalls:
  - Booting specifics depend on BIOS vs UEFI; ensure bootloader supports scheme.
- Interview tip:
  - "Use GPT for modern systems and disks >2TB; MBR for legacy BIOS compatibility."
- Example:
  - sudo parted /dev/sda print

---

Q97. Format partition with ext4
- What: Create an ext4 filesystem on a partition or block device.
- Command:
  - sudo mkfs.ext4 /dev/sdb1
- Verification:
  - sudo blkid /dev/sdb1 shows fstype=ext4
  - mount and check df -h
- Pitfalls:
  - mkfs will destroy existing data; ensure device target is correct.
- Interview tip:
  - "Use mkfs.ext4 for general-purpose filesystems; consider tuning parameters for large files."
- Example:
  - sudo mkfs.ext4 -L data /dev/loop0 && sudo mkdir -p /mnt/test && sudo mount /dev/loop0 /mnt/test && df -h /mnt/test

---

Q98. Permanently mount a partition
- What: Add mount entry to /etc/fstab using UUID for stability.
- Commands:
  - sudo blkid /dev/sdb1  # get UUID
  - Edit /etc/fstab and add: UUID=<uuid> /data ext4 defaults 0 0
  - sudo mount -a
- Verification:
  - mount | grep /data or findmnt /data
- Pitfalls:
  - Invalid fstab entries may prevent boot; use systemd's nofail or comment out when testing.
- Interview tip:
  - "Use UUID to avoid device name shifts across reboots."
- Example:
  - echo "UUID=$(blkid -s UUID -o value /dev/loop0) /data ext4 defaults 0 0" | sudo tee -a /etc/fstab && sudo mount -a

---

Q99. Resize existing partition in Linux
- What: Steps to grow/shrink partition and filesystem.
- Growing:
  - Use parted to resize partition to include free space, then resize filesystem (resize2fs for ext4, xfs_growfs for XFS).
- Shrinking:
  - Unmount, shrink filesystem first (resize2fs), then change partition size, then adjust LV if applicable.
- Commands:
  - sudo parted /dev/sda resizepart <num> <end>
  - sudo resize2fs /dev/sda1
- Verification:
  - lsblk and df -h show new sizes.
- Pitfalls:
  - Shrinking without resizing filesystem first causes data loss; XFS cannot shrink.
- Interview tip:
  - "Always shrink filesystem first when reducing partition size; growing is safer."
- Example:
  - sudo parted /dev/loop0 resizepart 1 100% && sudo resize2fs /dev/loop0p1

---

Q100. Repair filesystem
- What: Use fsck/xfs_repair to fix corrupted filesystems.
- Commands:
  - Unmount device: sudo umount /dev/sdb1
  - For ext: sudo fsck -y /dev/sdb1
  - For XFS: sudo xfs_repair /dev/sdb1 (ensure unmounted)
- Verification:
  - fsck reports fixed errors; mount and test file access.
- Pitfalls:
  - fsck -y auto-answers prompts; can cause unexpected changes; consider running without -y for review if unsure.
- Interview tip:
  - "Use fsck for ext*, xfs_repair for XFS; unmount first and backup if possible."
- Example:
  - sudo umount /mnt/test && sudo fsck -f /dev/loop0

---

Q101. Set up NFS & troubleshoot client mount failure
- What: Full server/client setup and common failures.
- Server steps:
  - sudo dnf install -y nfs-utils
  - sudo mkdir -p /nfsshare && sudo chown nobody:nobody /nfsshare && sudo chmod 0777 /nfsshare
  - Add /etc/exports: /nfsshare 10.0.0.0/24(rw,sync)
  - sudo exportfs -rav
  - sudo systemctl enable --now nfs-server rpcbind
  - sudo firewall-cmd --add-service=nfs --permanent && sudo firewall-cmd --reload
- Client steps:
  - sudo mount -t nfs server:/nfsshare /mnt
  - Add to /etc/fstab for permanent mount
- Troubleshooting:
  - Check server export: sudo exportfs -v
  - showmount -e server
  - Ensure rpcbind and nfs-server running: systemctl status rpcbind nfs-server
  - Check firewall and SELinux: setsebool -P nfs_export_all_rw 1 if SELinux blocks
- Verification:
  - On client, ls /mnt and create a test file; check it on server.
- Pitfalls:
  - NFSv4 specifics (idmapd, domain), and root_squash/default ownership may surprise users.
- Interview tip:
  - "Check exportfs and showmount for exports and use firewall-cmd to open NFS service ports."
- Example:
  - touch /nfsshare/testfile && mount server:/nfsshare /mnt && ls /mnt/testfile

---

Q102. System not booting — possible issues
- What: Common root causes and debugging steps.
- Possible causes:
  - corrupted /etc/fstab, missing root filesystem, GRUB issues, missing kernel/initramfs, hardware failure
- Steps to troubleshoot:
  1. Boot rescue mode via GRUB or live CD.
  2. Mount root partition and check /etc/fstab for invalid entries.
  3. journalctl -xb (if system boots partially) or check /var/log for errors.
  4. Rebuild initramfs: sudo dracut -f
  5. Reinstall GRUB: sudo grub2-install /dev/sda && sudo grub2-mkconfig -o /boot/grub2/grub.cfg
- Verification:
  - After fixes, reboot and confirm system reaches the expected target and services start.
- Pitfalls:
  - Reinstalling bootloader without setting correct device can render system unbootable.
- Interview tip:
  - "I mount the root FS from rescue, inspect fstab, rebuild initramfs or reinstall GRUB as needed, and verify logs."
- Example:
  - sudo mount /dev/mapper/vg-root /mnt/sysimage && sudo chroot /mnt/sysimage && dracut -f

---

Q103. How to perform patching in production?
- What: Steps for safe patching with rollback plan.
- Steps:
  1. Inventory servers and packages to patch.
  2. Take backups/snapshots of critical systems (VM snapshots, filesystem backups).
  3. Test patching on staging environment replicating production.
  4. Schedule maintenance window and inform stakeholders.
  5. Apply patches in a controlled manner (canary servers then rolling updates).
  6. Validate services and metrics; run smoke tests.
  7. If problems, rollback using yum history undo or restore snapshot.
- Commands:
  - sudo yum update -y  # or dnf
  - sudo yum history  # review transaction
  - sudo yum history undo <ID>  # rollback if supported
- Verification:
  - Check application logs, system metrics, functional tests, and user acceptance post-patch.
- Pitfalls:
  - Some updates require reboots and have cascading dependency impacts; maintain backout plan.
- Interview tip:
  - "I prefer staged rollouts with snapshots and smoke tests, and I always document changes and have rollback steps."
- Example:
  - Snapshot VM, sudo yum update -y, run app tests, rollback snapshot if failure

---

Q104. Well-known ports
- What: Common services and TCP/UDP ports to remember.
- Typical list:
  - SSH 22/tcp
  - HTTP 80/tcp
  - HTTPS 443/tcp
  - DNS 53/udp (and tcp)
  - SMTP 25/tcp
  - NTP 123/udp
  - FTP 21/tcp (data channel 20)
  - DHCP 67/68/udp
- Verification:
  - grep service names in /etc/services or ss -tuln to see active.
- Pitfalls:
  - Some services use dynamic ports (e.g., RPC services). Confirm with netstat/ss.
- Interview tip:
  - "I know common ports for SSH, HTTP, DNS, etc., and use tools like ss and nmap to verify."
- Example:
  - ss -tuln | grep -E '22|80|443|53'

---

Q105. What is swap & why used?
- What: Swap is disk-backed virtual memory used when RAM is exhausted.
- Uses:
  - Prevent OOM by providing extra memory; used for hibernation; kernel swap behavior controlled via vm.swappiness.
- Commands:
  - swapon --show
  - free -h
  - Create swapfile:
    sudo fallocate -l 2G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
- Verification:
  - swapon --show displays newly added swap; free -h shows swap size.
- Pitfalls:
  - Swap on slow disk is slow; rely on adequate RAM for performance-critical apps.
- Interview tip:
  - "Swap supports memory pressure but isn't a replacement for RAM; configure swappiness appropriately."
- Example:
  - sudo fallocate -l 1G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile && free -h

---

Q106. What is OOM killer?
- What: Kernel Out-Of-Memory (OOM) killer frees memory by terminating processes when system memory exhausted.
- Diagnosis:
  - dmesg | grep -i oom
  - journalctl -k | grep -i oom
- Mitigation:
  - Add more RAM, tune swappiness, set oom_adj/oom_score_adj for critical processes to avoid being killed.
- Verification:
  - After OOM, check dmesg logs for "Killed process <pid> (name) total-vm"
- Pitfalls:
  - OOM can kill important processes; ensure monitoring alerts when memory usage grows.
- Interview tip:
  - "I check dmesg and journalctl for OOM events and adjust oom_score_adj for critical services."
- Example:
  - dmesg | grep -i 'oom' || echo 'No OOM logged'

---

Q107. Kernel panic — causes & troubleshooting
- What: Kernel panic is a fatal error halting the kernel; requires reboot and investigation.
- Common causes:
  - Bad kernel upgrade, failing hardware (RAM/CPU), corrupted modules, missing initramfs, disk corruption
- Troubleshooting steps:
  1. Inspect serial console or kernel oops logs if available (journalctl -k).
  2. Boot into older kernel via GRUB to see if panic persists.
  3. Test RAM with memtest86.
  4. Rebuild initramfs: sudo dracut -f
  5. Reinstall kernel package if kernel files missing.
- Verification:
  - Boot into stable kernel and confirm system stability; check logs for panic cause.
- Pitfalls:
  - Some panics lack clear stack trace; hardware diagnostics necessary.
- Interview tip:
  - "I boot older kernel, run memtest, and rebuild initramfs, then check kernel logs to find offending module or hardware."
- Example:
  - sudo dracut -f && sudo reboot

---

Q108. `/boot/grub2` deleted — what to do?
- What: GRUB directory deletion leaves system unbootable; recover via rescue media.
- Steps:
  1. Boot from live/rescue ISO.
  2. Mount root filesystem and chroot:
     sudo mount /dev/sda1 /mnt  # root partition
     sudo mount --bind /dev /mnt/dev && mount --bind /proc /mnt/proc && mount --bind /sys /mnt/sys
     sudo chroot /mnt
  3. Reinstall grub: grub2-install /dev/sda
  4. Regenerate config: grub2-mkconfig -o /boot/grub2/grub.cfg
  5. Exit chroot, unmount and reboot.
- Verification:
  - System boots and grub menu displays; kernel entries present.
- Pitfalls:
  - Ensure correct device (e.g., /dev/sda vs /dev/nvme0n1) for grub install.
- Interview tip:
  - "Use live media, chroot, grub2-install and grub2-mkconfig to restore bootloader."
- Example:
  - (As above) then reboot and verify

---

Q109. initramfs corrupted or deleted
- What: initramfs contains modules and scripts needed to mount root filesystem; corruption prevents boot.
- Recovery:
  - Rebuild initramfs:
    sudo dracut -f
  - Or for a specific kernel:
    sudo dracut -f /boot/initramfs-$(uname -r).img $(uname -r)
- Verification:
  - ls -l /boot/initramfs-* shows regenerated files; reboot to test.
- Pitfalls:
  - If required modules are missing from the OS package tree, dracut may fail; ensure kernel-devel and necessary drivers present.
- Interview tip:
  - "I run dracut -f to regenerate initramfs; check dmesg on boot if still failing."
- Example:
  - sudo dracut -f && ls -l /boot/initramfs-*

---

Q110. Kernel deleted & rescue kernel missing
- What: If kernel is removed, system can't boot; reinstall kernel via rescue media or package manager.
- Steps:
  1. Boot rescue iso, mount root, chroot into system.
  2. Reinstall kernel package: sudo yum reinstall kernel (or apt-get install linux-image)
  3. Rebuild initramfs if required: dracut -f
- Verification:
  - ls /boot should list vmlinuz, initramfs; reboot to load kernel.
- Pitfalls:
  - If package manager chroots need network, configure /etc/resolv.conf and mount /proc /sys /dev into chroot.
- Interview tip:
  - "Use rescue media, chroot, and reinstall kernel packages; regenerate initramfs and reinstall GRUB if needed."
- Example:
  - sudo yum reinstall kernel -y (in chroot environment)

---

Q111. Classes of IP addresses
- What: Legacy classful addressing (mostly historical, replaced by CIDR).
- Ranges:
  - Class A: 1.0.0.0 – 126.255.255.255
  - Class B: 128.0.0.0 – 191.255.255.255
  - Class C: 192.0.0.0 – 223.255.255.255
  - Class D: 224.0.0.0 – 239.255.255.255 (multicast)
  - Class E: 240.0.0.0 – 255.255.255.255 (reserved)
- Verification:
  - ipcalc or online tools to compute subnets; use ip addr show to see local addresses.
- Pitfalls:
  - Classful routing is obsolete; use CIDR notation (e.g., 192.168.1.0/24).
- Interview tip:
  - "Explain ranges briefly and emphasize modern usage is CIDR, not classful."
- Example:
  - ipcalc 192.168.1.0/24

---

Q112. LAN, WAN, MAN
- What:
  - LAN: Local Area Network — small geographic area (office/home).
  - MAN: Metropolitan Area Network — covers a city or campus.
  - WAN: Wide Area Network — covers broader areas, interconnects LANs over ISPs.
- Verification:
  - Networking topology diagrams and traceroute can demonstrate scope difference.
- Pitfalls:
  - Terminology can vary; focus on practical differences.
- Interview tip:
  - "LAN is local, MAN covers a city or campus, and WAN spans large geographic distances."
- Example:
  - traceroute to distant host to show WAN hops

---

Q113. Router vs Switch vs Bridge vs Hub
- What:
  - Router: forwards packets between different networks (Layer 3).
  - Switch: switches frames within same network via MAC table (Layer 2).
  - Bridge: connects two network segments at Layer 2 (simple switch behavior).
  - Hub: broadcasts incoming signals to all ports; largely obsolete.
- Verification:
  - Observe TTL increments and different subnets when tracing across routers; MAC table using switch management interface.
- Pitfalls:
  - Misconfiguring routers/switches can create loops; use spanning tree protocol (STP) on layer 2 networks.
- Interview tip:
  - "Routers operate at L3 and handle IP routing; switches forward at L2 using MACs; hubs are dumb repeaters."
- Example:
  - Use managed switch interface to show MAC table entries (if available)

---

Q114. Allow port/service in firewalld
- What: Open ports or services using firewalld.
- Commands:
  - firewall-cmd --add-port=1521/tcp --permanent
  - firewall-cmd --add-service=http --permanent
  - firewall-cmd --reload
- Verification:
  - firewall-cmd --list-all shows ports/services open; ss shows service bound and reachable.
- Pitfalls:
  - Zone assignment matters; ensure the correct zone is modified (public, internal).
- Interview tip:
  - "Use firewall-cmd and reload; confirm with firewall-cmd --list-all and ss -ltnp to confirm listening service."
- Example:
  - sudo firewall-cmd --zone=public --add-service=http --permanent && sudo firewall-cmd --reload && sudo firewall-cmd --list-services

---

Q115. What are rich rules?
- What: Advanced firewalld rules allowing granular policies like source address, port, and accept/reject actions.
- Example command:
  - firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.1" port protocol="tcp" port="22" accept'
  - firewall-cmd --reload
- Verification:
  - firewall-cmd --list-rich-rules shows added entries; test connection from specified source.
- Pitfalls:
  - Rich rules can be complex; ensure syntax correctness.
- Interview tip:
  - "Rich rules let you define fine-grained rules (source, ports, actions); use them when default services/ports are insufficient."
- Example:
  - firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.5.0/24" service name="http" accept' && firewall-cmd --reload

---

Q116. System not booting due to wrong `/etc/fstab`
- What: Incorrect fstab entries cause boot failure when mount -a fails on critical filesystems.
- Steps to fix:
  1. Boot into rescue or single-user mode (append systemd.unit=rescue.target at GRUB).
  2. Edit /etc/fstab and comment problematic lines (#).
  3. Use mount -a to test.
  4. Reboot if mount -a succeeds.
- Verification:
  - mount -a returns success; system boots normally after reboot.
- Pitfalls:
  - Using device paths instead of UUID can cause mismatch after disk reordering; prefer UUID.
- Interview tip:
  - "I boot rescue, carefully edit fstab to comment failing entries, then run mount -a to ensure safe boot."
- Example:
  - sudo sed -i '/^UUID=abcd/s/^/#/' /etc/fstab && sudo mount -a

---

Q117. Explain a production issue faced (example answer)
- What: Structure an incident response story for interviews.
- Suggested STAR template:
  - Situation: Brief description of the outage or issue.
  - Task: Your responsibility (restore service).
  - Action: Commands and steps you performed (diagnostics, fixes).
  - Result: Outcome (service restored, lessons learned).
- Example short answer:
  - "Situation: app was down due to disk full. Task: restore service quickly and prevent recurrence. Action: used df/du to find big logs, lsof to find deleted-but-open file held by process, restarted service to free space, compressed old logs and updated logrotate. Result: service restored in 10 minutes and automated alerting added to prevent recurrence."
- Verification:
  - Show logs, commands used, and metrics that returned to normal.
- Interview tip:
  - "Quantify impact and steps, and explain preventive measures you implemented."
- Example:
  - Describe actual command sequence: df -h, du -sh /var/*, lsof | grep deleted, systemctl restart rsyslog

---

Q118. Different ways to create file
- What: Multiple shell methods to create files.
- Commands:
  - touch file  # create empty file or update timestamp
  - echo "text" > file  # create and write single line
  - cat > file <<EOF ... EOF  # interactive multiline
  - printf 'line\n' > file
  - vi file  # interactive editor
- Verification:
  - ls -l file and cat file show existence and contents.
- Pitfalls:
  - > truncates files; use >> to append when needed.
- Interview tip:
  - "Use touch for empty files; echo/printf for quick one-liners; vi for edits."
- Example:
  - echo "hello" > /tmp/testfile && cat /tmp/testfile

---

Q119. File types in Linux
- What: Types you can see in ls -l output.
- Types and symbols:
  - - regular file
  - d directory
  - l symlink
  - b block device
  - c character device
  - s socket
  - p named pipe (FIFO)
- Commands:
  - ls -l shows type; file filename describes file content.
- Verification:
  - stat filename displays file type and metadata.
- Pitfalls:
  - Some files may appear as special types and require root to manage.
- Interview tip:
  - "Explain major file types and how to identify them with ls -l and file."
- Example:
  - ls -l /dev | head ; file /etc/passwd

---

Q120. Day-to-day activities of Linux admin
- What: Routine operational tasks and priorities.
- Key activities:
  - User and permissions management, patching servers, monitoring and alert handling, backups, incident response, configuration management (Ansible), capacity planning, security audits (SELinux, firewall), performance tuning.
- Verification:
  - Keep logs, tickets, and runbooks as evidence of regular tasks.
- Pitfalls:
  - Overfocusing on reactive activities; invest time in automation and preventative measures.
- Interview tip:
  - "Discuss your balance of daily ops, automation, monitoring, and project work; give concrete examples."
- Example:
  - Describe a week: patching schedule, weekly backups, daily checks of monitoring dashboards.

---

Q121. Boot into non-default run level
- What: Change boot behavior for a single boot via GRUB.
- Steps:
  1. On GRUB menu, press 'e' to edit kernel line.
  2. Append systemd.unit=multi-user.target to linux line.
  3. Boot with Ctrl+x to use that target for this boot only.
- Verification:
  - systemctl get-default still shows default; systemctl isolate shows current target.
- Pitfalls:
  - Use only when comfortable with GRUB editing; incorrect kernel params may prevent boot.
- Interview tip:
  - "I use GRUB edit to temporarily switch target for recovery or debugging without changing default."
- Example:
  - At GRUB, add systemd.unit=rescue.target

---

Q121 (duplicate). SELinux context missing — fix
- What: Restore SELinux contexts to default policy for files and directories.
- Commands:
  - restorecon -Rv /path/to/dir
  - To permanently set context on new paths: semanage fcontext -a -t httpd_sys_content_t "/myapp(/.*)?"
  - restorecon -Rv /myapp
- Verification:
  - ls -Z /path shows SELinux labels; getenforce shows Enforcing/Permissive.
- Pitfalls:
  - Using setenforce 0 disables SELinux; prefer adjusting contexts and booleans.
- Interview tip:
  - "I restore contexts with restorecon and set policy with semanage fcontext for persistent labels."
- Example:
  - sudo restorecon -Rv /var/www/html

---

Q122. What is `journalctl`? Usage?
- What: Tool to view systemd journal logs.
- Useful options:
  - journalctl -b  # current boot
  - journalctl -b -1  # previous boot
  - journalctl -u sshd  # service logs
  - journalctl -f  # follow
  - journalctl --since "2025-01-01" --until "2025-01-02"
- Verification:
  - journalctl -k shows kernel messages; entries include timestamps and unit names.
- Pitfalls:
  - Journal can grow large; configure persistent storage and retention via /etc/systemd/journald.conf
- Interview tip:
  - "I use journalctl -u to debug services and -b to see boot-time logs; use --since to scope time windows."
- Example:
  - sudo journalctl -u httpd --since "10 minutes ago" -f

---

Q123. Logs under `/var/log`
- What: Key log files and their purpose.
- Common logs:
  - /var/log/messages or /var/log/syslog (system messages)
  - /var/log/secure (authentication)
  - /var/log/cron (cron jobs)
  - /var/log/boot.log (boot messages)
  - /var/log/maillog (mail service)
  - /var/log/dmesg (kernel ring buffer)
- Verification:
  - tail -F /var/log/messages while reproducing an event to see logs appear.
- Pitfalls:
  - Log rotation must be configured to prevent uncontrolled disk usage.
- Interview tip:
  - "Know which logs to check for auth (secure/auth.log), boot (journalctl or boot.log), and application-specific logs."
- Example:
  - sudo tail -n 100 /var/log/secure

---

Q124. `dmidecode` command
- What: Read system hardware and BIOS-provided DMI/SMBIOS tables.
- Commands:
  - sudo dmidecode | less
  - sudo dmidecode -t memory  # memory sections
- Verification:
  - Get serial numbers, BIOS version, and hardware vendor strings.
- Pitfalls:
  - Output depends on firmware; some fields may be empty or generic.
- Interview tip:
  - "Use dmidecode for hardware inventory and serial numbers for RMA or asset tracking."
- Example:
  - sudo dmidecode -t system

---

Q125. List system hardware info
- What: Commands for CPU, block devices, PCI, USB info.
- Commands:
  - lscpu
  - lsblk -f
  - lspci
  - lsusb
  - free -h for memory
- Verification:
  - Cross-check lscpu with /proc/cpuinfo; lsblk with blkid.
- Pitfalls:
  - Some virtualized environments present fewer details.
- Interview tip:
  - "I use lscpu, lsblk, lspci, and lsusb to quickly inventory hardware."
- Example:
  - lscpu && lsblk && lspci | head

---

Q126. Process states & zombie vs orphan
- What:
  - Running (R), Sleeping (S), Uninterruptible sleep (D), Stopped (T), Zombie (Z)
  - Zombie: process has terminated but parent hasn't called wait() to collect status — remains in process table.
  - Orphan: process whose parent died; init/systemd adopts it and reaps zombies.
- Commands:
  - ps aux | grep Z
  - ps -eo pid,ppid,stat,cmd | grep -E 'Z|<defunct>'
- Verification:
  - ps shows Z or defunct processes; parent PID listed.
- Pitfalls:
  - Many zombies indicate parent process bug; they consume PID entries.
- Interview tip:
  - "Zombies don't use CPU or memory but consume PIDs; reparenting to init allows reaping."
- Example:
  - Use a small C program to create a zombie (for learning) then kill the parent; observe ps.

---

Q127. What is DNS? Role of `/etc/hosts`?
- What: DNS resolves domain names to IP addresses; /etc/hosts provides local hostname overrides.
- Commands:
  - cat /etc/hosts
  - dig example.com @8.8.8.8
  - host hostname
- Verification:
  - ping hostname resolves via /etc/hosts first before DNS unless nsswitch order changes.
- Pitfalls:
  - Hardcoding IPs in /etc/hosts can cause configuration drift; use for small fixes or local overrides only.
- Interview tip:
  - "Explain /etc/hosts precedence and use dig/nslookup to debug DNS resolution."
- Example:
  - echo "192.168.1.10 dbserver" | sudo tee -a /etc/hosts && ping -c1 dbserver

---

Q128. Check last login info
- What: Commands to show login history and last logins.
- Commands:
  - last  # lists recent logins and reboots
  - lastlog  # last login per user
- Verification:
  - last shows time and source; correlate with logs in /var/log/secure.
- Pitfalls:
  - last reads wtmp file which may have rotated; older history may be archived.
- Interview tip:
  - "Use last for a timeline and lastlog to see accounts without recent logins."
- Example:
  - last | head

---

Q129. What is SLA & task prioritization?
- What: Service Level Agreement and how to prioritize tasks.
- Explanation:
  - SLA defines uptime/response requirements; tasks prioritized by impact, urgency, number of users affected, contractual obligations, and potential revenue impact.
- Verification:
  - Use ticketing systems and agreed SLO/SLA metrics to show meeting or breach status.
- Pitfalls:
  - Overlooking business context can prioritize low-impact tasks erroneously.
- Interview tip:
  - "Discuss an example: prioritize outage affecting payment services over minor UI bug based on SLA and business impact."
- Example:
  - Describe triage process and escalation contacts in a previous role.

---

Q130. Jump to particular line in vim
- What: Navigate directly to a line number.
- Commands:
  - :25  # jump to line 25
  - Or use <line_number>G (e.g., 25G)
- Verification:
  - vim cursor lands on that line.
- Pitfalls:
  - None significant.
- Interview tip:
  - "Use colon command or G movement for efficient navigation."
- Example:
  - vim +25 file

---

Q131. Jump to bottom of file in vim
- What: Go to end of file.
- Commands:
  - G  # capital G
- Verification:
  - Cursor at last line.
- Pitfalls:
  - None.
- Interview tip:
  - "Use G to jump to EOF or gg for start."
- Example:
  - Open file and press G

---

Q132. Replace a string in vim
- What: Use substitute command for global replacement.
- Commands:
  - :%s/old/new/g  # replace globally in file
  - :%s/old/new/gc  # confirm each
- Verification:
  - Use :%s/old//n to show matches before changing; after replacement check with :%s/new//n
- Pitfalls:
  - Regex special characters need escaping.
- Interview tip:
  - "Use g flag for global and c flag to confirm replacements."
- Example:
  - vim -c '%s/foo/bar/g | wq' file  # replace from shell and save

---

Q133. Use awk & sed
- What: awk for field-oriented processing; sed for stream editing.
- Commands:
  - awk -F: '{print $1,$3}' /etc/passwd  # prints username and UID
  - sed -i 's/http/https/g' file  # inline replacement
- Verification:
  - Compare before/after outputs; use sed without -i to preview.
- Pitfalls:
  - sed -i differs across BSD and GNU sed syntax; use -i '' on macOS.
- Interview tip:
  - "Use awk for column extraction and sed for quick transformations; provide an example when asked."
- Example:
  - awk -F: '$3>=1000{print $1,$3}' /etc/passwd

---

Q134. Change owner & group of file
- What: chown to set file owner and group.
- Commands:
  - sudo chown user:group file
  - sudo chown user. group file  # older syntax
- Verification:
  - ls -l file displays updated owner and group.
- Pitfalls:
  - Recursive chown can change many files unintentionally; test without -R first.
- Interview tip:
  - "Use chown user:group file and verify with ls -l."
- Example:
  - sudo chown $(whoami):$(id -gn) /tmp/testfile && ls -l /tmp/testfile

---

Q135. What is NTP server & how to configure?
- What: Network Time Protocol server syncs clocks across systems; chrony is common on modern Linux.
- Commands:
  - sudo dnf install -y chrony
  - Edit /etc/chrony.conf to add pool or server lines (e.g., server 0.pool.ntp.org iburst)
  - sudo systemctl enable --now chronyd
  - chronyc sources -v  # verify sources and sync state
- Verification:
  - chronyc tracking shows system clock skews and stratum.
- Pitfalls:
  - Firewall must allow NTP (123/udp); virtualized environments may need host time sync settings adjusted.
- Interview tip:
  - "Use chrony for better handling of intermittent connections and virtualization."
- Example:
  - sudo chronyc sources -v

---

Q136. Passwordless authentication (SSH keys)
- What: Use public/private key pair for authentication, more secure than password.
- Steps:
  - Generate key: ssh-keygen -t ed25519 -C "user@host"
  - Copy public key to server: ssh-copy-id user@server
  - Ensure correct permissions: chmod 700 ~/.ssh ; chmod 600 ~/.ssh/authorized_keys
  - Optionally disable password auth in /etc/ssh/sshd_config: PasswordAuthentication no ; sudo systemctl reload sshd
- Verification:
  - ssh user@server should log in without prompt; sudo tail -f /var/log/secure for sshd logs if debug needed.
- Pitfalls:
  - Disabling password auth before verifying keys works can lock you out; always test in another session.
- Interview tip:
  - "Use ed25519 keys where possible; use ssh-copy-id and confirm owner/permission on server side."
- Example:
  - ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 && ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host && ssh user@host

---

Q137. Setup `httpd` web server
- What: Install and configure Apache HTTPD on RHEL-based systems.
- Commands:
  - sudo dnf install -y httpd
  - sudo systemctl enable --now httpd
  - echo "hello" | sudo tee /var/www/html/index.html
  - sudo firewall-cmd --add-service=http --permanent && sudo firewall-cmd --reload
- Verification:
  - curl -I http://localhost returns HTTP/1.1 200 OK; remote test using browser or curl from another host.
- Pitfalls:
  - SELinux may prevent content from being served; use restorecon -Rv /var/www/html if necessary.
- Interview tip:
  - "Install httpd, enable it, and open firewall. Check SELinux if access fails."
- Example:
  - curl http://localhost

---

Q138. Cron job backup at 2AM
- What: Schedule a cron job to run at a specific time daily.
- Steps:
  - crontab -e
  - Add line: 0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
- Verification:
  - Check /var/log/cron or use system logs to confirm cron executed; or inspect /var/log/backup.log
- Pitfalls:
  - Use full paths in scripts run from cron; environment is minimal.
- Interview tip:
  - "Use crontab -e and always log output and use absolute paths."
- Example:
  - echo "0 2 * * * /usr/bin/tar -czf /backup/$(date +\%F).tgz /etc" > /tmp/cronfile && crontab /tmp/cronfile

---

Q139. Case-insensitive search using grep
- What: grep supports -i for case-insensitive matching.
- Commands:
  - grep -Ri "error" /var/log
  - grep -Rin "pattern" /path  # -n to show line numbers
- Verification:
  - Output shows matching lines regardless of case.
- Pitfalls:
  - Extensive recursion may produce many matches; use --exclude or --include to limit.
- Interview tip:
  - "Use grep -Ri for recursive, case-insensitive search; combine with -n to show line numbers."
- Example:
  - grep -Rin "failed password" /var/log

---

Q140. Set static IP without nmcli/nmtui
- What: Configure network interface via legacy ifcfg files on RHEL/CentOS.
- File (/etc/sysconfig/network-scripts/ifcfg-ens33) example:
  - BOOTPROTO=none
  - IPADDR=192.168.1.10
  - PREFIX=24
  - GATEWAY=192.168.1.1
  - DNS1=8.8.8.8
  - ONBOOT=yes
- Commands:
  - sudo systemctl restart NetworkManager or network
- Verification:
  - ip addr show ens33 and ip route show
- Pitfalls:
  - On systems managed by NetworkManager, manual edits may be overwritten; use nmcli or connection files under /etc/NetworkManager/system-connections/.
- Interview tip:
  - "If you can't use nmcli, edit the ifcfg file; always restart the manager and verify with ip addr and ip route."
- Example:
  - Create ifcfg file and sudo systemctl restart NetworkManager && ip addr show ens33

---

Q141. Where to check fake login attempts?
- What: Look for authentication failures in system logs.
- Commands:
  - sudo grep "Failed password" /var/log/secure  # RHEL
  - sudo grep "authentication failure" /var/log/auth.log  # Debian/Ubuntu
  - Use journalctl: sudo journalctl _COMM=sshd | grep "Failed password"
- Verification:
  - Count occurrences by IP: sudo grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
- Pitfalls:
  - Logs may rotate; look into archived logs or journald persistent logs.
- Interview tip:
  - "Use grep to find 'Failed password' entries and identify offending IPs to block via firewall or fail2ban."
- Example:
  - sudo grep "Failed password" /var/log/secure | tail -n 50

---

Q142. Start, enable, disable service
- What: Manage systemd services.
- Commands:
  - sudo systemctl start httpd
  - sudo systemctl enable httpd
  - sudo systemctl disable httpd
  - sudo systemctl status httpd
- Verification:
  - systemctl is-enabled httpd; systemctl status shows active/inactive.
- Pitfalls:
  - enable adds symlink for future boots; start only affects current runtime.
- Interview tip:
  - "Use enable to persist across reboots and start to run immediately."
- Example:
  - sudo systemctl enable --now httpd && systemctl status httpd

---

Q143. tar, zip, gzip, bzip2, xz — what are they?
- What: Differences between archiving and compression tools.
- Summary:
  - tar: archiver (bundles files into a single file, no compression)
  - gzip: compression algorithm (.gz)
  - bzip2: higher compression ratio (.bz2), slower
  - xz: very high compression (.xz), slower
  - zip: archive + compress in one tool with directory support and common on Windows
- Commands:
  - tar -czf archive.tar.gz /dir  # gzip
  - tar -cJf archive.tar.xz /dir  # xz
  - zip -r archive.zip /dir
- Verification:
  - tar -tf archive.tar.gz ; unzip -l archive.zip
- Pitfalls:
  - Compression choice depends on CPU time vs disk space trade-offs.
- Interview tip:
  - "Use gzip for speed, xz for maximal compression when CPU/time permits."
- Example:
  - tar -czf etc.tgz /etc && ls -lh etc.tgz

---

Q144. Update timestamp of file
- What: touch updates access and modification times or creates file if not exist.
- Commands:
  - touch file
  - touch -t 202501011200 file  # set timestamp explicitly
- Verification:
  - stat file shows new atime and mtime
- Pitfalls:
  - touch without -a or -m may update both times; use -a or -m to update specific timestamp.
- Interview tip:
  - "Use touch to create placeholder files or update timestamps for build systems."
- Example:
  - touch /tmp/testtouch && stat /tmp/testtouch

---

Q145. Create SOS report
- What: Generate a comprehensive system report for vendor support (RHEL/CentOS).
- Commands:
  - sudo dnf install -y sos  # if not installed
  - sudo sosreport
- Output:
  - Generates an archive with logs, configs, and system info suitable for support engineers.
- Verification:
  - sosreport output path printed; inspect contents with tar -tf <reportfile>
- Pitfalls:
  - sosreport may collect sensitive data; share only with trusted parties or redact.
- Interview tip:
  - "sosreport is used to gather diagnostic information for Red Hat support cases."
- Example:
  - sudo sosreport --batch --all-logs

---

End of Questions 1–145

This document contains step-by-step actions, verification, pitfalls, and interview talking points for each of the 145 items. Use it as an interview study guide and hands-on lab checklist. If you would like, I can:
- Export this content as a downloadable README.md file.
- Break the file into smaller topic-specific files (users, LVM, networking, security).
- Generate lab scripts to automatically run safe, non-destructive examples in a VM.

Which would you like next?
