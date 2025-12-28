# 🧠 Shell Scripting — Exercises with Answers (Beginner → Advanced)

> Practice-focused, copy-paste-ready exercises with detailed explanations, example runs, and tips.  
> Works in Bash (Linux / macOS). Save this file as `README.md` and use it as a hands‑on workbook.

---

## Table of Contents

- Overview & How to use this file
- Conventions & safe defaults
- Beginner Exercises (1–10) — tasks, solved scripts, detailed explanations
- Intermediate Exercises (11–20) — tasks, solved scripts, detailed explanations
- Advanced Exercises (21–30) — tasks, solved scripts, detailed explanations
- Practice assignments (extra)
- Further reading & tools

---

## Overview & How to use this file

- Each exercise includes:
  - Task (what to implement)
  - Solution (copy-paste script)
  - Explanation (how it works, important details)
  - Example run (expected input/output)
  - Edge cases / improvements (optional)
- Tips:
  - For safety, run scripts in a test environment (temporary dirs).
  - Use `bash -n file` to syntax-check and `shellcheck` to lint.
  - Make scripts executable with `chmod +x file.sh` for direct runs.

---

## Conventions & safe defaults

- Use `#!/usr/bin/env bash` for portability.
- Prefer `set -Eeuo pipefail` and `IFS=$'\n\t'` in production scripts; for short exercises we show minimal wrappers but explain safety notes.
- Quote variable expansions: `"$var"`.
- Use `read -r` and `IFS=` when reading lines to preserve whitespace.

---

# 🟢 Beginner Exercises (1–10)

Each exercise is intentionally short to teach a core concept.

---

## ✅ Exercise 1 — Print “Hello World”

Task: Print `Hello World`.

Solution:

```bash
#!/usr/bin/env bash
echo "Hello World"
```

Explanation:
- `echo` writes text to stdout followed by newline.
- This is the canonical first script.

Example run:

```
$ bash hello.sh
Hello World
```

Notes:
- `printf` is an alternative with more formatting control: `printf '%s\n' "Hello World"`.

---

## ✅ Exercise 2 — Current working directory & username

Task: Print current directory and current user.

Solution:

```bash
#!/usr/bin/env bash
echo "User: $USER"
echo "Current directory: $(pwd)"
```

Explanation:
- `$USER` is an environment variable (common on Linux).
- `pwd` prints the current working directory; command substitution `$(...)` inserts its output.

Example:

```
User: ajay
Current directory: /home/ajay/projects
```

Edge cases:
- `$USER` may be unset in some shells; you can use `$(whoami)` as fallback: `user=${USER:-$(whoami)}`.

---

## ✅ Exercise 3 — Add two numbers

Task: Read two numbers and print their sum.

Solution:

```bash
#!/usr/bin/env bash
read -rp "Enter a: " a
read -rp "Enter b: " b
echo "Sum = $((a + b))"
```

Explanation:
- `read -rp` prompts without adding a newline.
- Arithmetic expansion `$(( ... ))` evaluates integers.

Example:

```
Enter a: 7
Enter b: 5
Sum = 12
```

Notes:
- This uses integer arithmetic. For floats, use `bc` or `awk`.

---

## ✅ Exercise 4 — Even or odd

Task: Check if a number is even or odd.

Solution:

```bash
#!/usr/bin/env bash
read -rp "Enter number: " n
if (( n % 2 == 0 )); then
  echo "Even"
else
  echo "Odd"
fi
```

Explanation:
- `(( ... ))` does arithmetic; `%` is modulo.
- Returns 0 (true) when expression non-zero? In `(( ))` the expression result determines truthiness.

Example:

```
Enter number: 4
Even
```

Edge:
- Validate input is numeric: `[[ $n =~ ^-?[0-9]+$ ]] || { echo "Not a number"; exit 1; }`.

---

## ✅ Exercise 5 — Multiplication table

Task: Print 1..10 multiplication table for a given number.

Solution:

```bash
#!/usr/bin/env bash
read -rp "Enter number: " n
for i in {1..10}; do
  printf "%d x %d = %d\n" "$n" "$i" "$(( n * i ))"
done
```

Explanation:
- `for i in {1..10}` iterates numbers 1..10 (Bash brace expansion).
- `printf` ensures consistent formatting.

Example:

```
Enter number: 3
3 x 1 = 3
3 x 2 = 6
...
3 x 10 = 30
```

---

## ✅ Exercise 6 — Count files in directory

Task: Count files (non-recursive) in a directory.

Solution (safe):

```bash
#!/usr/bin/env bash
read -rp "Directory (default .): " dir
dir=${dir:-.}
count=$(find "$dir" -maxdepth 1 -type f | wc -l)
echo "Total files: $count"
```

Explanation:
- `find` is safer than parsing `ls` because it handles filenames with spaces/newlines.
- `-maxdepth 1` restricts to directory root (non-recursive).

Example:

```
Directory (default .):
Total files: 7
```

Edge:
- If you want to include hidden files, `find` already does; `ls` would need `-A`.

---

## ✅ Exercise 7 — Check if file exists

Task: Ask for filename and check existence.

Solution:

```bash
#!/usr/bin/env bash
read -rp "Enter filename: " f
if [ -f "$f" ]; then
  echo "File exists: $f"
else
  echo "File not found: $f"
fi
```

Explanation:
- `-f` checks regular file existence. Use `-e` for any file type (including symlinks), `-d` for directories.

---

## ✅ Exercise 8 — Biggest of two numbers

Task: Determine larger of two inputs.

Solution:

```bash
#!/usr/bin/env bash
read -rp "Enter a: " a
read -rp "Enter b: " b
if (( a > b )); then
  echo "$a is greater"
elif (( a < b )); then
  echo "$b is greater"
else
  echo "Both are equal: $a"
fi
```

Explanation:
- `(( ))` numeric comparison; `-gt` is an alternative with `[ ]`.

Edge:
- Validate numeric input.

---

## ✅ Exercise 9 — Check if user is root

Task: Detect whether the script runs as root.

Solution:

```bash
#!/usr/bin/env bash
if [ "${EUID:-$(id -u)}" -eq 0 ]; then
  echo "You are root"
else
  echo "Not root"
fi
```

Explanation:
- `EUID` available in Bash; fallback to `id -u`.
- Root UID is 0.

Security note:
- For dangerous operations, always check UID and refuse to run or confirm.

---

## ✅ Exercise 10 — Check if service is running

Task: Check systemd service status.

Solution:

```bash
#!/usr/bin/env bash
read -rp "Enter service name: " s
if systemctl is-active --quiet "$s"; then
  echo "Service '$s' is running"
else
  echo "Service '$s' is not running"
fi
```

Explanation:
- `systemctl is-active --quiet` sets exit code based on service status — ideal for scripts.
- On non-systemd systems, use `service $s status` or `ps` fallback.

---

# 🟡 Intermediate Exercises (11–20)

More processing, strings, logs and simple automation.

---

## ✅ Exercise 11 — Reverse a string

Task: Reverse user-provided string.

Solution:

```bash
#!/usr/bin/env bash
read -rp "Enter string: " str
echo "$str" | rev
```

Explanation:
- `rev` reverses every line. Works for simple ASCII and UTF-8 compatible reversed bytes (be cautious with multi-byte graphemes).

Alternative (pure bash):

```bash
s="$str"; rev=""
for ((i=${#s}-1;i>=0;i--)); do rev+=${s:i:1}; done
echo "$rev"
```

---

## ✅ Exercise 12 — Sum of digits

Task: Sum digits of a positive integer.

Solution:

```bash
#!/usr/bin/env bash
read -rp "Enter number: " n
if ! [[ $n =~ ^[0-9]+$ ]]; then echo "Enter positive integer"; exit 1; fi
sum=0
while [ "$n" -gt 0 ]; do
  d=$(( n % 10 ))
  sum=$(( sum + d ))
  n=$(( n / 10 ))
done
echo "Sum = $sum"
```

Explanation:
- Uses arithmetic modulo and integer division. Validates input.

---

## ✅ Exercise 13 — Palindrome check

Task: Check whether string reads the same reversed.

Solution:

```bash
#!/usr/bin/env bash
read -rp "Enter string: " str
clean=$(echo "$str" | tr -d '[:space:]' | tr '[:upper:]' '[:lower:]')
rev=$(echo "$clean" | rev)
if [ "$clean" = "$rev" ]; then
  echo "Palindrome"
else
  echo "Not palindrome"
fi
```

Explanation:
- Normalizes by removing spaces and lowercasing for a case-insensitive check.

---

## ✅ Exercise 14 — System info menu

Task: Interactive `select` menu for quick diagnostics.

Solution:

```bash
#!/usr/bin/env bash
PS3="Choose option: "
select opt in "Date" "Uptime" "Users" "Disk Usage" "Quit"; do
  case $opt in
    Date) date;;
    Uptime) uptime;;
    Users) who;;
    "Disk Usage") df -h;;
    Quit) break;;
    *) echo "Invalid";;
  esac
done
```

Explanation:
- `select` prints numbered menu; `PS3` is the prompt.

---

## ✅ Exercise 15 — Backup a directory

Task: Compress a directory to timestamped tar.gz.

Solution:

```bash
#!/usr/bin/env bash
read -rp "Directory to backup: " src
read -rp "Destination dir (default /tmp): " dest
dest=${dest:-/tmp}
if [ ! -d "$src" ]; then echo "Source not found"; exit 1; fi
mkdir -p "$dest"
archive="$dest/$(basename "$src")-$(date +%F-%H%M%S).tar.gz"
tar -C "$(dirname "$src")" -czf "$archive" "$(basename "$src")"
echo "Backup created: $archive"
```

Explanation:
- `tar -C` ensures relative paths in archive.
- Timestamp avoids overwriting.

Improvements:
- Add `flock` to avoid concurrent backups; add retention policy.

---

## ✅ Exercise 16 — Disk usage alert (>80%)

Task: Alert when root filesystem > 80%.

Solution:

```bash
#!/usr/bin/env bash
threshold=80
use=$(df -P / | awk 'NR==2 {gsub(/%/,""); print $5}')
if [ "$use" -gt "$threshold" ]; then
  echo "Disk usage high: ${use}%"
else
  echo "Disk usage OK: ${use}%"
fi
```

Explanation:
- `df -P` ensures POSIX format; `awk` gets the percent, `gsub` removes `%`.

Testing tip:
- For unit tests, allow an env var `SIMULATE_USAGE` to override.

---

## ✅ Exercise 17 — Lowercase → Uppercase

Task: Convert text to uppercase.

Solution:

```bash
#!/usr/bin/env bash
read -rp "Enter text: " t
echo "$t" | tr '[:lower:]' '[:upper:]'
```

Explanation:
- `tr` translates characters; locale-aware `awk`/`perl` can be used for more advanced cases.

---

## ✅ Exercise 18 — Top 5 memory-consuming processes

Task: Show top 5 processes by memory.

Solution:

```bash
#!/usr/bin/env bash
ps -eo pid,user,pmem,pcpu,comm --sort=-pmem | head -n 6
```

Explanation:
- `ps -eo` prints columns; `--sort=-pmem` sorts descending by memory; `head -n 6` includes header.

---

## ✅ Exercise 19 — Remove empty files automatically

Task: Find and delete empty files under `/tmp`.

Solution:

```bash
#!/usr/bin/env bash
find /tmp -type f -empty -print -delete
```

Explanation:
- `-empty` matches empty files; `-print` helps auditing; use with caution.

Dry-run suggestion:

```bash
find /tmp -type f -empty -print
# If OK, then:
find /tmp -type f -empty -delete
```

---

## ✅ Exercise 20 — Log script output with timestamp

Task: Append a timestamped line to a log.

Solution:

```bash
#!/usr/bin/env bash
LOGFILE="${1:-/var/log/custom.log}"
mkdir -p "$(dirname "$LOGFILE")"
echo "$(date '+%F %T') : Log entry" >> "$LOGFILE"
```

Explanation:
- Uses `date` formatting; parameterizable logfile with default.

Permissions:
- Writing to `/var/log` requires root; use a user-writable path in exercises.

---

# 🔴 Advanced Exercises (21–30)

Production-style tasks combining multiple concepts. Pay attention to security and robustness.

---

## ✅ Exercise 21 — Bulk user creation from file

Task: Create system users listed in `users.txt`.

Solution (safe, with checks):

```bash
#!/usr/bin/env bash
set -euo pipefail
file="${1:-users.txt}"
[ -f "$file" ] || { echo "File not found: $file"; exit 1; }
while IFS= read -r user || [ -n "$user" ]; do
  if id "$user" &>/dev/null; then
    echo "Skipping existing user: $user"
  else
    useradd --create-home "$user" && echo "Created: $user"
  fi
done < "$file"
```

Explanation:
- `id` tests existing user.
- `|| [ -n "$user" ]` handles files without trailing newline.
- Requires root privileges to create users. In tests, simulate by printing commands.

Security:
- Validate usernames: `[[ $user =~ ^[a-z_][a-z0-9_-]{0,30}$ ]]`.

---

## ✅ Exercise 22 — Find failed SSH login attempts

Task: Extract failed SSH logins from logs.

Solution (RHEL/CentOS):

```bash
#!/usr/bin/env bash
grep -i "Failed password" /var/log/secure | awk '{print $1, $2, $3, $11, $13}'
```

Debian/Ubuntu logs may be in `/var/log/auth.log`. Explanation:
- `grep` filters lines; `awk` prints fields like date, time, username, IP (field indexes depend on distro/log format).

Advanced:
- Use `journalctl` for systemd: `journalctl -u sshd --since "1 day ago" | grep "Failed password"`.

---

## ✅ Exercise 23 — Parse Apache access log (top IPs)

Task: Show top 10 client IPs by request count.

Solution:

```bash
#!/usr/bin/env bash
log="${1:-/var/log/apache2/access.log}"
awk '{print $1}' "$log" | sort | uniq -c | sort -nr | head -n 10
```

Explanation:
- `$1` is the client IP in common log format.
- Works with large logs by piping to `sort`.

Advanced:
- To include counts per URL or status code, print different fields: `awk '{print $7, $9}'`.

---

## ✅ Exercise 24 — Parallel ping subnet scanner

Task: Find live hosts in `192.168.1.0/24` using parallelism.

Solution (xargs for parallelism):

```bash
#!/usr/bin/env bash
network="192.168.1"
seq 1 254 | xargs -n1 -P50 -I{} bash -c 'ping -c1 -W1 '"$network"'.{} >/dev/null && echo '"$network"'.{}" UP"'
```

Explanation:
- `xargs -P50` runs up to 50 parallel pings.
- `-W1` sets 1-second wait (Linux `ping`). On BSD/macOS use `-W` semantics differ; use `-t` or adjust.

Note:
- Run with care on managed networks; consider ICMP rate limits.

---

## ✅ Exercise 25 — Monitor CPU and send alert

Task: If CPU usage (user+system) > threshold, print alert (or send email/webhook).

Solution (using `mpstat`):

```bash
#!/usr/bin/env bash
threshold=${1:-85}
if ! command -v mpstat &>/dev/null; then echo "mpstat required"; exit 1; fi
idle=$(mpstat 1 1 | awk '/Average/ {print $12}')
# On some systems field may differ; check mpstat output
usage=$(awk "BEGIN {printf \"%d\", 100 - $idle}")
if [ "$usage" -gt "$threshold" ]; then
  echo "ALERT: CPU usage at ${usage}%"
fi
```

Explanation:
- `mpstat 1 1` samples once for 1 second; retrieves idle percentage.
- Alternative: parse `/proc/stat` to compute CPU usage when `mpstat` not available.

Improvements:
- Add notification (sendmail, curl webhook) and rate-limit alerts.

---

## ✅ Exercise 26 — Automatic log rotation (simple)

Task: Compress logs older than 7 days.

Solution:

```bash
#!/usr/bin/env bash
logdir="${1:-/var/log/myapp}"
find "$logdir" -type f -name "*.log" -mtime +7 -print -exec gzip {} \;
```

Explanation:
- `-mtime +7` matches files older than 7 days.
- `gzip` compresses in-place; consider moving to archive directory instead.

Production:
- Use `logrotate` for robust rotation, compression, retention.

---

## ✅ Exercise 27 — Database (MySQL) backup automation

Task: Dump a DB and compress, rotate old backups.

Solution:

```bash
#!/usr/bin/env bash
DB_USER="${DB_USER:-root}"
DB_NAME="${1:-mydb}"
DEST="${2:-/var/backups}"
mkdir -p "$DEST"
ts=$(date -u +%F-%H%M%SZ)
file="$DEST/${DB_NAME}-${ts}.sql.gz"
mysqldump --single-transaction --quick --lock-tables=false -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" | gzip > "$file"
chmod 600 "$file"
find "$DEST" -type f -name "${DB_NAME}-*.sql.gz" -mtime +7 -delete
echo "Backup saved to $file"
```

Explanation:
- Uses `mysqldump` options to avoid long locks.
- `DB_PASS` from env var (do not hardcode passwords).
- `chmod 600` restricts backup file access.

Security:
- Use secrets manager or `my.cnf` credential files; never commit credentials.

---

## ✅ Exercise 28 — Validate URL status codes

Task: Check HTTP status for a URL.

Solution:

```bash
#!/usr/bin/env bash
read -rp "URL: " url
code=$(curl -s -o /dev/null -w "%{http_code}" "$url")
echo "Status code: $code"
if [[ "$code" =~ ^2|3 ]]; then
  echo "OK"
else
  echo "Non-OK status: $code"
fi
```

Explanation:
- `curl -w` prints status code; `-s -o /dev/null` silences body.
- For HTTPS cert issues add `--fail` to treat HTTP errors as non-zero exit.

---

## ✅ Exercise 29 — Detect zombie processes

Task: List processes in `Z` state.

Solution:

```bash
#!/usr/bin/env bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/ {print $0}'
```

Explanation:
- `stat` column contains process state (Z for zombie).
- Zombie processes often indicate parent didn't reap; investigate parent PID.

---

## ✅ Exercise 30 — Real-time service self-healing script

Task: If a service stops, restart it and log action.

Solution (systemd example):

```bash
#!/usr/bin/env bash
service="${1:-httpd}"
if ! systemctl is-active --quiet "$service"; then
  echo "$(date '+%F %T') - $service is down, attempting restart" >> /var/log/selfheal.log
  if systemctl restart "$service"; then
    echo "$(date '+%F %T') - $service restarted successfully" >> /var/log/selfheal.log
  else
    echo "$(date '+%F %T') - Failed to restart $service" >> /var/log/selfheal.log
  fi
else
  echo "Service $service is running"
fi
```

Explanation:
- Logs events with timestamps.
- In production, consider systemd unit templates with `Restart=on-failure` instead of external self-healers.

---

# 🧪 Practice Assignments (Try Yourself)

1. Monitor memory across nodes and aggregate results (use SSH + `free -m`).
2. Build menu-driven calculator supporting add/sub/mul/div with error handling.
3. Generate complex random password (cryptographically secure).
4. Parse CSV into a formatted table (handle quoted fields).
5. Monitor multiple services concurrently and send a single consolidated report.

Reply “need answers” to request solutions and tests for any of the above.

---

# Further reading & tools

- Bash Reference Manual — https://www.gnu.org/software/bash/manual/
- Advanced Bash-Scripting Guide — https://tldp.org/LDP/abs/html/
- ShellCheck — https://www.shellcheck.net/ (linting)
- jq (JSON) — https://stedolan.github.io/jq/
- bats-core (testing) — https://bats-core.readthedocs.io/

---

- Add a printable one-page cheatsheet PDF / SVG with common commands and patterns.

Which would you like next? 👇
