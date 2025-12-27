# 🐧 Linux Basic Commands – Hands-on Lab Guide

This repository contains step-by-step practical labs to learn and practice basic Linux commands safely in a lab environment. Each lab contains the commands to run, an explanation of what they do, variations and examples, tips for safe usage, and short exercises to reinforce learning.

> ✅ Recommended OS: CentOS 7 / RHEL / Rocky / Ubuntu  
> ✅ Environment: VirtualBox / VMware / Cloud VM  
> ⚠️ Do not run labs on production systems — use snapshots or throwaway VMs.

---

## Table of Contents

- Lab preparation
- 1. File commands
- 2. Searching (grep)
- 3. Process management
- 4. File permissions
- 5. Network commands
- 6. System information
- 7. Compression & archiving
- 8. Keyboard shortcuts & shell tips
- Safety notes and exercises

---

## 📂 Lab Preparation

Create a working directory for all labs. Keep all exercises inside so you can safely remove them later:

```
mkdir -p ~/linux-labs
cd ~/linux-labs
pwd  # verify location
```

Tip: Take a VM snapshot before starting a series of labs so you can revert if needed.

---

# 1. File Commands Labs

This section covers creating, listing, viewing, copying, moving, and removing files and directories.

## Lab 1 — Listing & Navigating Directories

Commands

```
ls            # shows non-hidden files in current directory
ls -l         # long listing: permissions, owner, size, timestamp
ls -a         # include hidden files (names starting with .)
ls -al        # combine long listing and show hidden
pwd           # print current working directory
cd /path      # change directory
cd ..         # go up one directory
cd ~          # go to current user's home
```

Explanation
- `ls` shows contents; `-l` gives details; `-a` shows hidden files such as `.bashrc`.
- `pwd` is helpful when you're not sure where you are.

Example
```
mkdir -p proj1 proj2
touch .secretfile
ls -al
```

Exercise
- Create directories `projA` and `projB`. Enter `projA` and verify `pwd` shows the correct path.

Pitfall
- File names with spaces may be split: use quotes or escape spaces (e.g., `ls "My File.txt"`).

---

## Lab 2 — Create, Copy, Move, Delete Files

Commands

```
touch file1.txt file2.txt        # create empty files or update timestamps
cp file1.txt backup.txt          # copy file1.txt to backup.txt
cp -r sourcedir destinadir       # copy directories recursively
mv file2.txt renamed.txt         # move or rename files
mkdir archive
mv backup.txt archive/           # move file into directory
rm renamed.txt                   # remove file
rm -r archive                    # remove directory recursively
rm -rf dir                        # force remove recursively (dangerous)
```

Explanation
- `touch` is quick to create files or update timestamps.
- `cp` duplicates content. Use `-i` to prompt before overwrite.
- `mv` renames or moves files — it does not make a copy.
- `rm -r` deletes directories and contents. `-f` suppresses prompts; avoid `rm -rf /`.

Examples
```
touch file1.txt file2.txt
cp file1.txt backup.txt
mv file2.txt renamed.txt
mkdir -p archive
mv backup.txt archive/
rm renamed.txt
rm -r archive
```

Best practice
- Use `trash` or move to a temporary directory instead of immediate deletion if unsure.
- Add an alias for safety: `alias rm='rm -i'` (interactive delete).

Exercises
- Create 3 files, copy one to a new name, move another into a directory, then remove only the directory contents.

---

## Lab 3 — View File Contents

Commands

```
cat file.txt                # print entire file
less file.txt               # pager for large files (use q to quit)
more file.txt               # simpler pager
head -n 10 file.txt         # first 10 lines
tail -n 10 file.txt         # last 10 lines
tail -f /var/log/messages   # follow file as new lines are appended (logs)
nl file.txt                 # show line numbers
```

Explanation
- Use `less` for interactive viewing (supports scrolling and searching within the file).
- `tail -f` is commonly used to watch log files as the system writes to them.

Example
```
cat > notes.txt <<'EOF'
This is line1
This is line2
This is line3
This is line4
This is line5
EOF

head -n 2 notes.txt
tail -n 2 notes.txt
less notes.txt
```

Tip
- `less +F file` starts `less` in follow mode (similar to `tail -f`) and pressing `Ctrl-C` returns to normal `less` behavior.

Exercises
- Create a log-like file by appending timestamps every second in the background and watch it with `tail -f`.

---

# 2. Searching Commands Labs

## Lab 4 — grep Search Examples

Commands

```
grep 'pattern' file.txt               # basic match (case-sensitive)
grep -i 'pattern' file.txt            # case-insensitive
grep -n 'pattern' file.txt            # show line numbers
grep -r 'pattern' /path/to/dir        # recursive search in directory
grep -R --include='*.log' 'error' .   # recursive but only *.log files
grep -v 'pattern' file.txt            # invert match (show lines that do NOT match)
grep -E 'err(or)?' file.txt           # extended regex (ERE)
```

Explanation
- `grep` searches text content for patterns. By default, regular expressions are allowed.
- Use `-r` to search directories recursively. `-n` helps locate occurrences by line number.
- Use `-i` for case-insensitive searches; `-v` to invert results.

Example
```
cat > cities.txt <<'EOF'
Mumbai
Delhi
Hyderabad
Chennai
Mumbai
Pune
EOF

grep Mumbai cities.txt
grep -i mumbai cities.txt
grep -n Mumbai cities.txt
```

Use case
- Searching `/var/log` for errors: `sudo grep -R "error" /var/log` (run as root if required).

Exercises
- Search recursively for the word `TODO` in your project directory.
- Find files containing both "Failed" and "ssh" (hint: use `grep -i` piped with another `grep`).

---

# 3. Process Management Labs

View processes and manage them.

Commands

```
ps -ef                         # full-format process list (system-wide)
ps aux                         # alternate common format
top                            # interactive process viewer (press q to quit)
htop                           # improved top (if installed)
pgrep nginx                    # find PID(s) by process name
ps -ef | grep sshd             # combine with grep to find specific process
kill <PID>                     # send SIGTERM (graceful) to PID
kill -9 <PID>                  # send SIGKILL (force kill)
nice -n 10 command             # run a command with lower priority
renice +5 -p <PID>             # change priority of running process
```

Explanation
- `ps` shows a snapshot of current processes. Use `top` or `htop` for dynamic monitoring.
- `kill` without `-9` sends SIGTERM allowing cleanup; `-9` forces termination and may leave resources in inconsistent state.
- `pgrep` and `pkill` are convenient: `pkill sshd` will signal processes matching name.

Example
```
ps -ef | grep sshd
top
# In another terminal:
sleep 300 &
pgrep sleep
kill <PID>
```

Safety
- Avoid killing random PIDs unless you know what the process is — `systemd` or critical services might be affected.

Exercises
- Start a background `sleep 2000` process and then list and kill it gracefully.

---

# 4. File Permissions Labs

Understanding permissions and changing them.

Permissions model: rwx for user (owner), group, others. Example from `ls -l`:
```
-rwxr-xr-- 1 alice staff 1234 Jan  1 12:00 script.sh
```
Breakdown:
- `rwx` owner: read, write, execute
- `r-x` group: read, execute
- `r--` others: read only

Commands

```
touch secure.sh
ls -l secure.sh
chmod +x secure.sh       # add execute permission for owner/group/others
chmod 700 secure.sh      # numeric: owner rwx (7), group 0, others 0
chmod 644 file.txt       # owner rw, group r, others r (common for text files)
chown user:group file    # change owner and group
stat file                # more detailed file status
```

Explanation
- `chmod` can use symbolic (`u`, `g`, `o`, `a`, `+`, `-`, `=`) or numeric (octal) modes.
- `chown` requires root to change file ownership.

Examples
```
touch secure.sh
chmod +x secure.sh
ls -l secure.sh
chmod 700 secure.sh
touch openfile.txt
chmod 777 openfile.txt  # not recommended for real systems
```

Security note
- Avoid `chmod 777` on production systems. Use the least privilege needed.

Exercises
- Create a script `runme.sh`, make it executable only by you, and verify other users cannot execute it.

---

# 5. Network Commands Labs

Basic network troubleshooting and information.

Commands

```
ping -c 3 google.com         # send 3 ICMP echo requests
ip addr                     # show network interfaces and addresses
ip route show               # show routing table
nslookup google.com         # DNS lookup (or dig if available)
dig example.com             # DNS query (if installed)
traceroute google.com       # show path packets take to reach host
ss -tuln                    # show listening sockets (replacement for netstat)
curl -I https://example.com # fetch headers from HTTP server
```

Explanation
- `ping` checks host reachability and round-trip time.
- `ip` replaces older `ifconfig` and `route`. Use `ss` instead of `netstat` for active sockets.
- `traceroute` helps find where packets are delayed or dropped.

Examples
```
ping -c 3 google.com
ip addr
nslookup google.com
traceroute google.com
ss -tuln
```

Exercises
- Use `curl -I` to fetch headers from `https://www.github.com`.
- Use `ss -tuln` to find which service is listening on port 22 (ssh).

---

# 6. System Information Labs

Quick system checks and hardware info.

Commands

```
date                        # show date & time
uptime                      # show system uptime and load averages
whoami                      # show current user
uname -r                    # kernel version
cat /etc/os-release         # distribution and version
lscpu                       # CPU architecture and details
free -m                     # memory usage in MB
vmstat 1 5                  # quick system stats every 1s for 5 iterations
df -h                       # disk usage human readable
du -sh /var/log             # disk usage of directory
```

Explanation
- Load averages (in `uptime`) show system load over 1, 5, and 15 minutes. Compare to number of CPU cores.
- `df -h` shows partition usage; `du` shows folder usage.

Examples
```
date
uptime
uname -r
cat /etc/os-release
lscpu
free -m
df -h
```

Exercises
- Find the top 5 largest directories in `/var` using `du -sh /var/* | sort -h`.

---

# 7. Compression Labs

Create and extract archives.

Commands

```
mkdir -p backupdir
touch backupdir/a.txt backupdir/b.txt

tar -cvf backup.tar backupdir         # create uncompressed tar
tar -xvf backup.tar                   # extract tar
tar -czvf backup.tar.gz backupdir     # create gzip-compressed tarball
tar -xzvf backup.tar.gz               # extract gzip tarball
zip -r backupdir.zip backupdir        # create zip (if zip installed)
unzip backupdir.zip                   # extract zip
```

Explanation
- `tar` is commonly used for packaging directories. `-c` create, `-x` extract, `-v` verbose, `-f` filename, `-z` gzip compress.
- `zip`/`unzip` are common on Windows-interoperable systems.

Examples
```
tar -czvf myproject-$(date +%F).tar.gz myproject
tar -xzvf myproject-2025-12-27.tar.gz
```

Exercises
- Create a tar.gz of a test folder and verify its contents with `tar -tzvf`.

---

# 8. Keyboard Shortcuts Practice

Shell and terminal shortcuts to speed up workflow.

Common shortcuts

- Ctrl + C — stop running command (SIGINT)
- Ctrl + Z — suspend current command (SIGTSTP)
- fg — bring last suspended job to foreground
- bg — send suspended job to background
- Ctrl + L — clear screen (same as `clear`)
- Ctrl + D — send EOF (closes shell if at prompt)
- history — show recent commands
- !n — run history command number n (e.g., `!42`)
- !! — repeat last command
- Ctrl + R — reverse search command history (interactive)

Examples
```
# Start a command
ping google.com

# Press Ctrl+C to stop
# Start a long sleep and background it
sleep 300
Ctrl+Z
bg
jobs
fg
```

Tip
- Use `Ctrl+R` and type part of a previous command to quickly find it.

---

## Safety Notes & Best Practices

- Never run destructive commands (like `rm -rf /`) on a system you care about.
- When learning, prefer creating disposable VMs or containers.
- Use `--dry-run` or `echo` to preview commands where available (e.g., rsync has `--dry-run`).
- Use version control (git) for config files to track changes.
- Learn to read man pages: `man <command>` (e.g., `man tar`, `man grep`).

---

## Quick Reference Cheatsheet

- File ops: `ls`, `cp`, `mv`, `rm`, `mkdir`, `rmdir`
- Viewing: `cat`, `less`, `head`, `tail`
- Searching: `grep`, `find`, `locate`
- Processes: `ps`, `top`, `htop`, `pgrep`, `kill`
- Network: `ping`, `ip`, `ss`, `curl`, `traceroute`, `nslookup`
- System: `date`, `uptime`, `uname`, `lscpu`, `free`, `df`
- Archiving: `tar`, `gzip`, `zip`

---

## Suggested Learning Path (short)

1. Practice navigation and file creation for 30 minutes.
2. Use `grep` and `less` to explore log files.
3. Start/stop simple background processes and practice `kill`, `bg`, `fg`.
4. Change permissions and understand `chmod` numeric modes.
5. Run basic networking checks (`ping`, `curl`, `ss`).
6. Archive and extract test folders with `tar`.

---
