# 🐚 Shell Scripting — Beginner → Practical (Complete README)

[![Shell](https://img.shields.io/badge/Shell-Bash-blue)](https://www.gnu.org/software/bash/)
[![Platform](https://img.shields.io/badge/Platform-Linux%20%7C%20macOS-lightgrey)](#)
[![Level](https://img.shields.io/badge/Level-Beginner--Intermediate-green)](#)

> An "Awesome" beginner-friendly, practical Bash shell scripting guide.  
> Start from zero knowledge and move step-by-step into writing real scripts you can reuse and test.

---

## Table of Contents

1. What this guide covers
2. How to use this README
3. Quick Start — Hello World (copy & run)
4. Running scripts (3 ways)
5. Shebang, permissions & execution flow
6. Comments & documentation style
7. Variables — rules, quoting & examples
8. Reading input (interactive and non-interactive)
9. Command substitution, pipes & redirection
10. Arithmetic (ints & floats)
11. Conditional tests: `[ ]`, `[[ ]]`, `(( ))`
12. If / elif / else, and `case`
13. Loops — for, while, until
14. Functions, scope & return values
15. Arrays & associative arrays
16. Strings & pattern matching
17. Files, file tests & safe file handling
18. Error handling, `set` flags & `trap`
19. Useful I/O patterns (heredocs, process substitution)
20. Best practices & recommended template
21. Real-world scripts (copy-ready)
22. Cheatsheet (common commands)
23. Exercises with solutions
24. Troubleshooting & FAQ
25. Further reading

---

## 1 — What this guide covers
- Core Bash features needed to write safe, maintainable scripts.
- Practical examples you can copy/paste.
- Real scripts (backups, disk alerts, user checks) ready to use.
- Exercises + solutions for practice.

This guide assumes access to a Unix-like terminal (Linux/macOS). All examples use Bash.

---

## 2 — How to use this README
- Read sections in order if you're new.
- Copy code blocks into `.sh` files, `chmod +x`, and run.
- Experiment by changing variables and inputs.
- Use the Cheatsheet as a quick reference.

---

## 3 — Quick Start — Hello World

Create `hello.sh`:

```bash
#!/usr/bin/env bash
# hello.sh — prints a greeting
echo "Hello, World!"
```

Make executable & run:

```bash
chmod +x hello.sh
./hello.sh
```

Expected output:
```
Hello, World!
```

Notes:
- `#!/usr/bin/env bash` finds the `bash` interpreter in PATH (portable).

---

## 4 — Running scripts (3 ways)

1. Direct execution (script must be executable and have a valid shebang):
   ```bash
   ./script.sh
   ```
2. Call interpreter explicitly:
   ```bash
   bash script.sh
   sh script.sh
   ```
3. Source into current shell (no subshell — useful when exporting variables):
   ```bash
   source script.sh
   . script.sh
   ```

Use explicit interpreter (`bash script.sh`) when you want reproducible behavior regardless of shebang.

---

## 5 — Shebang, permissions & execution flow

- Shebang chooses the interpreter: `#!/usr/bin/env bash` or `#!/bin/bash`.
- Grant execute permission: `chmod +x file.sh`.
- Scripts run in a subshell; to change the current shell environment, `source` them.

---

## 6 — Comments & documentation style

Comment at top with purpose, usage, options:

```bash
#!/usr/bin/env bash
# backup.sh — create tar.gz backup of /etc
# Usage: ./backup.sh /path/to/dest
# Example: ./backup.sh /mnt/backup
```

Use inline comments sparingly and document non-obvious logic.

---

## 7 — Variables — rules, quoting & examples

Assignment:

```bash
name="Ajay"
age=30
```

Do NOT add spaces around `=`.

Access: `$name` or `${name}`.

Quoting:
- Use double quotes to prevent word-splitting: `"$var"`
- Use single quotes when you want literal text: `'literal $var'`

Examples:

```bash
path="/home/$USER/my dir"
echo "Path: $path"         # safe with quotes
echo $path                 # may split on whitespace
```

Environment variable:

```bash
export MYVAR="value"
```

Read-only:

```bash
readonly PI=3.14
```

Unset safe default:

```bash
value="${SOME_VAR:-default}"  # default if SOME_VAR empty or unset
```

---

## 8 — Reading input (interactive & non-interactive)

Interactive prompt:

```bash
read -rp "Enter your name: " name
echo "Hello, $name"
```

Default values:

```bash
read -rp "Region [us-east-1]: " region
region=${region:-us-east-1}
```

Read file line-by-line (safe for spaces):

```bash
while IFS= read -r line; do
  echo "Line: $line"
done < file.txt
```

Notes:
- `IFS=` prevents trimming leading/trailing whitespace.
- `-r` prevents backslash escaping.

---

## 9 — Command substitution, pipes & redirection

Command substitution:

```bash
now=$(date '+%F %T')
echo "Now: $now"
```

Pipes:

```bash
ps aux | grep sshd | grep -v grep
```

Redirection:
- `>` overwrite, `>>` append
- `2>` redirect STDERR, `2>&1` combine
- `&>` redirect both STDOUT & STDERR (bash-specific)

Capture both outputs:

```bash
if some_command >out.log 2>&1; then
  echo "OK"
else
  echo "Failed (check out.log)"
fi
```

---

## 10 — Arithmetic (ints & floats)

Integer math: `$(( ))`

```bash
a=7; b=3
echo $(( a + b ))   # 10
echo $(( a / b ))   # 2 (integer division)
```

C-style:

```bash
(( a++ ))
if (( a > b )); then echo "a>b"; fi
```

Floating point (use `bc` or `awk`):

```bash
result=$(awk 'BEGIN {printf "%.3f\n", 10/3}')
echo "$result"
```

---

## 11 — Conditional tests: `[ ]`, `[[ ]]`, `(( ))`

- `[ ... ]` POSIX test — portable
- `[[ ... ]]` Bash extended test — safer (no globbing/word-splitting), supports regex
- `(( ... ))` arithmetic evaluation

Examples:

```bash
if [ -f "/etc/passwd" ]; then echo "exists"; fi
if [[ $name =~ ^[A-Za-z]+$ ]]; then echo "alpha only"; fi
if (( num % 2 == 0 )); then echo "even"; fi
```

Comparison operators:
- Numeric: `-eq -ne -gt -lt -ge -le` or use `(( ))`
- String: `=` or `==` (in `[[ ]]`), use `!=` for not equal

Always quote variables in `[ ]`: `[ "$x" = "$y" ]`

---

## 12 — If / elif / else, and `case`

If structure:

```bash
if [[ -z "$1" ]]; then
  echo "Usage: $0 arg"
  exit 1
elif [[ "$1" == "start" ]]; then
  echo "Starting"
else
  echo "Unknown command"
fi
```

Case (good for multiple choices):

```bash
case "$opt" in
  start)  echo "start";;
  stop)   echo "stop";;
  *)      echo "unknown";;
esac
```

---

## 13 — Loops — for, while, until

For loop over list:

```bash
for i in 1 2 3; do
  echo "Item: $i"
done
```

For over files (safe):

```bash
for file in *.log; do
  [ -e "$file" ] || continue  # handle no-match
  echo "Processing: $file"
done
```

While loop:

```bash
n=1
while [ $n -le 5 ]; do
  echo "$n"
  n=$((n+1))
done
```

Until:

```bash
n=1
until [ $n -gt 5 ]; do
  echo "$n"
  n=$((n+1))
done
```

---

## 14 — Functions, scope & return values

Define:

```bash
greet() {
  local name="$1"
  echo "Hello, $name"
}
```

Notes:
- Use `local` to avoid global pollution.
- `return` only returns an integer status (0-255). To return strings, `echo` and capture.

Example:

```bash
get_date() { date '+%F'; }
today=$(get_date)
```

Check function exit status:

```bash
if do_something; then
  echo "ok"
else
  echo "failed"
fi
```

---

## 15 — Arrays & associative arrays

Indexed arrays:

```bash
arr=("one" "two" "three")
echo "${arr[0]}"        # one
echo "${arr[@]}"        # all elements
echo "${#arr[@]}"       # length
```

Associative arrays (Bash 4+):

```bash
declare -A cap
cap[India]="New Delhi"
cap[USA]="Washington"
echo "${cap[India]}"
```

Iterate:

```bash
for key in "${!cap[@]}"; do
  echo "$key -> ${cap[$key]}"
done
```

---

## 16 — Strings & pattern matching

Length, substring, replace:

```bash
s="LinuxShell"
echo "${#s}"            # length
echo "${s:0:5}"         # substring "Linux"
echo "${s/Shell/SCRIPT}"# "LinuxSCRIPT"
```

Globbing vs regex:
- Globbing: `*.txt` (filename matching)
- Regex: `[[ $s =~ ^[0-9]+$ ]]`

---

## 17 — Files, file tests & safe file handling

Common tests:
- `-f` regular file, `-d` directory
- `-r` readable, `-w` writable, `-x` executable
- `-s` non-empty file

Example:

```bash
if [ -d "$dir" ]; then
  echo "dir ok"
else
  mkdir -p "$dir"
fi
```

Safe temp file:

```bash
tmp=$(mktemp) || exit 1
trap 'rm -f "$tmp"' EXIT
```

Avoid parsing `ls`. Use `find` for robust file lists.

---

## 18 — Error handling, `set` flags & `trap`

Recommended beginning of scripts:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'
```

Meaning:
- `-e` exit on error
- `-u` treat unset variables as errors
- `-o pipefail` pipeline fails if any command fails
- `-E` ensures ERR trap inherits in functions

`trap` for cleanup:

```bash
cleanup() { echo "Cleaning..."; rm -f "$tmp"; }
trap cleanup EXIT
```

Be cautious: `set -e` has nuances inside conditionals and pipelines.

---

## 19 — Useful I/O patterns (heredocs, process substitution)

Heredoc:

```bash
cat <<'EOF' > /tmp/msg.txt
Hello $USER
Multi-line message
EOF
```

Note `'EOF'` prevents variable expansion inside heredoc.

Process substitution:

```bash
diff <(sort file1) <(sort file2)
```

Redirect both stdout+stderr:

```bash
cmd >out.log 2>&1
```

---

## 20 — Best practices & recommended template

- Use `#!/usr/bin/env bash`
- Add `set -Eeuo pipefail` and `IFS=$'\n\t'`
- Use functions and `local` variables
- Quote expansions: `"$var"`
- Validate inputs and exit with meaningful messages
- Use `mktemp` for temporary files
- Prefer `[[ ]]` for tests in Bash scripts
- Provide `--help` and usage examples

Starter template:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

usage() {
  cat <<EOF
Usage: $0 <dest-dir>
Example: $0 /backup
EOF
  exit 1
}

main() {
  local dest="${1:-}"
  [ -n "$dest" ] || usage
  mkdir -p "$dest"
  echo "Created $dest"
}

main "$@"
```

---

## 21 — Real-world scripts (copy-ready)

1) Check if user exists

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
read -rp "Enter username: " user
if id "$user" &>/dev/null; then
  echo "User '$user' exists"
else
  echo "User '$user' not found"
  exit 2
fi
```

2) Disk usage alert (80% threshold)

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
threshold=80
use=$(df / | awk 'NR==2 {gsub(/%/,""); print $5}')
if (( use > threshold )); then
  echo "ALERT: Disk usage is ${use}% on $(hostname) at $(date)"
fi
```

3) Backup `/etc` safely

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
dest_dir="${1:-/tmp}"
dest_file="$dest_dir/etc-$(date +%F).tar.gz"
mkdir -p "$dest_dir"
tar -czf "$dest_file" /etc
echo "Backup saved to $dest_file"
```

4) Batch rename `.txt` → `.bak` (safe for spaces)

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
shopt -s nullglob
for f in *.txt; do
  mv -- "$f" "${f%.txt}.bak"
done
```

---

## 22 — Cheatsheet (common commands)

- Make executable: `chmod +x script.sh`
- Run: `./script.sh` or `bash script.sh`
- Quote variables: `"$var"`
- Exit status: `$?`
- Print formatted: `printf "%s\n" "$var"`
- Read: `read -rp "Prompt: " var`
- Integer math: `$((a + b))`
- Array: `arr=(a b c)` / `${arr[@]}`

Quick file tests:
- `[ -f file ]`, `[ -d dir ]`, `[ -r file ]`, `[ -w file ]`, `[ -x file ]`

Useful `set`:
```bash
set -Eeuo pipefail
IFS=$'\n\t'
```

---

## 23 — Exercises with solutions

Exercise 1 — Multiplication table
Problem: Print table for user-provided number (1..10)

Solution:

```bash
#!/usr/bin/env bash
read -rp "Number: " n
for ((i=1;i<=10;i++)); do
  printf "%d x %d = %d\n" "$n" "$i" "$((n*i))"
done
```

Exercise 2 — Max of three numbers

```bash
#!/usr/bin/env bash
read -rp "a: " a
read -rp "b: " b
read -rp "c: " c
max="$a"
for v in "$b" "$c"; do
  if (( v > max )); then max="$v"; fi
done
echo "Max is $max"
```

Exercise 3 — Rename `.txt` to `.bak` (safe)

```bash
#!/usr/bin/env bash
shopt -s nullglob
for f in *.txt; do
  mv -- "$f" "${f%.txt}.bak"
done
```

Exercise 4 — Count files in a directory

```bash
#!/usr/bin/env bash
dir="${1:-.}"
count=$(find "$dir" -maxdepth 1 -type f | wc -l)
echo "Files in $dir: $count"
```

Exercise 5 — Simple CPU usage check (example using `top` or `mpstat`)

A portable quick check:

```bash
#!/usr/bin/env bash
idle=$(mpstat 1 1 | awk '/Average/ {print $12+0}')
if (( $(awk "BEGIN {print ($idle < 20)}") )); then
  echo "High CPU usage — idle ${idle}%"
else
  echo "CPU idle ${idle}%"
fi
```

---

## 24 — Troubleshooting & FAQ

- Script shows `[: missing ']'` → Check spacing: `[ "$x" = "y" ]`
- Globs fail for spaces → Always quote filenames: `"$file"`
- `set -u` causes exit on unset variable → use `${var:-default}` for defaults
- `set -e` exits unexpectedly inside `if` or `&&` → be careful and test commands

Debugging tips:
- Add `set -x` to trace execution (remove in production)
- Use `shellcheck` (https://www.shellcheck.net/) to lint scripts

---

## 25 — Further reading & tools

- Bash Reference Manual — https://www.gnu.org/software/bash/manual/
- Advanced Bash-Scripting Guide — https://tldp.org/LDP/abs/html/
- ShellCheck — https://www.shellcheck.net/
- Bats (Bash Automated Testing System) — https://bats-core.readthedocs.io/

---


- Generate colorized terminal demo examples (copy/paste with ANSI colors).

Which of these shall I produce next?
