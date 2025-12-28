# 🐚 Shell Scripting — Beginner → Practical (Complete README)

> A friendly, **step-by-step** Bash shell scripting guide for absolute beginners.
>
> Works on Linux, macOS, and other Unix-like systems.  
> Copy this file as `README.md` and use it as your learning reference.

---

[TOC]

- 1 What is a Shell & Shell Scripting?
- 2 Your First Script — Hello World
- 3 Ways to Run a Script
- 4 Shebangs, Permissions & Execution Flow
- 5 Comments & Documentation
- 6 Variables — Types, Naming & Quoting
- 7 Reading Input (interactive & non-interactive)
- 8 Command Substitution, Pipes & Redirection
- 9 Arithmetic & Floating-point
- 10 Tests: [ ] vs [[ ]] vs (())
- 11 Conditional Logic — if / elif / else
- 12 Case statements
- 13 Loops — for, while, until
- 14 Functions, Return Values & Scope
- 15 Arrays & Associative Arrays
- 16 Strings & Pattern Matching
- 17 Files & File Tests
- 18 Error handling, Exit Codes & set options
- 19 Useful I/O patterns & here-documents
- 20 Best Practices & Script Template
- 21 Real-world Examples
- 22 Practice Lab with Solutions
- 23 Troubleshooting Tips & FAQ
- 24 Further Reading & Resources

---

## 1️⃣ What is a Shell & Shell Scripting?

- A *shell* is a command-line interpreter — a program that accepts commands and runs them.
  - Common shells: `bash`, `sh`, `zsh`, `ksh`.
- A *shell script* is a text file with a sequence of shell commands. Scripts automate repetitive tasks.

Why learn shell scripting?
- Automate system tasks (backup, monitoring, installs)
- Glue different programs together
- Useful for DevOps, SRE, admin tasks, CI/CD, quick prototyping

---

## 2️⃣ Your First Script — Hello World

Create a file named `hello.sh`:

```bash
#!/bin/bash
# hello.sh — first script
echo "Hello, World!"
```

Make it executable and run:

```bash
chmod +x hello.sh
./hello.sh
```

Output:
```
Hello, World!
```

Explanation:
- `#! /bin/bash` (shebang) tells OS which interpreter to use.
- `echo` prints a line to STDOUT.

---

## 3️⃣ Ways to Run a Script

- Direct execution (requires executable flag and valid shebang):
  ```bash
  ./script.sh
  ```
- Using an interpreter explicitly:
  ```bash
  bash script.sh
  sh script.sh
  ```
- Sourcing (runs in the current shell — useful for setting env vars):
  ```bash
  source script.sh
  . script.sh
  ```

Note: `sh` may invoke a more minimal shell — some Bash features won't work.

---

## 4️⃣ Shebangs, Permissions & Execution Flow

Common shebangs:
- `#!/bin/bash` — use Bash
- `#!/usr/bin/env bash` — more portable (finds `bash` in PATH)

Permissions:
- `chmod +x script.sh` — make executable
- The OS executes the interpreter in the shebang, passing the script path.

Execution flow tip:
- Scripts run in a subshell by default. To modify the current shell environment (like `export`), source the script.

---

## 5️⃣ Comments & Documentation

- Single-line comments start with `#`:
  ```bash
  # This is a comment
  ```
- Document the script purpose, usage, options, and examples at the top:

```bash
#!/usr/bin/env bash
# backup.sh — archive /etc
# Usage: ./backup.sh /path/to/dest
# Example: ./backup.sh /mnt/backup
```

---

## 6️⃣ Variables — Types, Naming & Quoting

Basics:

```bash
name="Ajay"
age=30
echo "User: $name, Age: $age"
```

Rules:
- No spaces around `=` (wrong: `a = 1`)
- Variable names: letters, numbers, underscore; avoid starting with digit
- Convention: shell variables often lowercase for user variables

Quoting:
- Double quotes `"${var}"` allow expansions inside but prevent word splitting & globbing.
- Single quotes `'literal'` prevent any expansion.
- Unquoted variables can lead to bugs due to word splitting:
  ```bash
  files=$(ls *.txt)
  for f in $files; do echo "$f"; done  # WRONG for filenames with spaces
  for f in $files; do ...; done        # better: use loops over glob directly
  ```

Environment variables:
- `export VAR=value` — makes a variable available to child processes.

Read-only:
- `readonly VAR="value"`

---

## 7️⃣ Reading Input (interactive & non-interactive)

Interactive:

```bash
read -p "Enter your name: " name
echo "Hello, $name"
```

Read with default:

```bash
read -rp "Value [default]: " val
val=${val:-default}
```

Read from file/pipe:

```bash
while IFS= read -r line; do
  echo "Line: $line"
done < file.txt
```

Notes:
- `IFS=` and `-r` avoid trimming whitespace and backslash interpretation.

---

## 8️⃣ Command Substitution, Pipes & Redirection

Command substitution:

```bash
now=$(date '+%F %T')
echo "Now: $now"
```

Older form: `` `cmd` `` — prefer `$(...)` for nesting clarity.

Pipes:

```bash
ps aux | grep sshd | grep -v grep
```

Redirects:
- `>` overwrite
- `>>` append
- `<` read file into STDIN
- `2>` redirect STDERR
- `&>` redirect both STDOUT & STDERR (bash)
- `cmd >out.txt 2>&1` portable way to combine

Here's capturing both outputs:

```bash
if cmd >out.log 2>&1; then
  echo "OK"
else
  echo "Failed"
fi
```

---

## 9️⃣ Arithmetic & Floating-point

Integer arithmetic (Bash built-in):

```bash
a=10
b=3
echo $(( a + b ))   # 13
echo $(( a / b ))   # 3   (integer division)
```

`let` and `(( ))`:

```bash
((a++))
if (( a > b )); then echo "a>b"; fi
```

Floating point — use `bc` or `awk`:

```bash
result=$(awk "BEGIN {print 10/3}")
echo "$result"
```

---

## 1️⃣0️⃣ Tests: [ ] vs [[ ]] vs (())

- `[ ... ]` — POSIX `test`, requires careful quoting
- `[[ ... ]]` — Bash extended test (regex, no word-splitting, safer)
- `(( ... ))` — arithmetic evaluation

Examples:

```bash
if [ -f "/etc/passwd" ]; then echo "ok"; fi
if [[ $name == "Ajay" ]]; then echo "match"; fi
if (( 5 > 3 )); then echo "true"; fi
```

String equality:
- `[ "$a" = "$b" ]` or `[[ $a == $b ]]`

Regex in `[[ ]]`:
```bash
if [[ $input =~ ^[0-9]+$ ]]; then echo "numeric"; fi
```

---

## 1️⃣1️⃣ Conditional Logic

Classic IF:

```bash
read -p "Number: " n
if (( n > 0 )); then
  echo "Positive"
elif (( n < 0 )); then
  echo "Negative"
else
  echo "Zero"
fi
```

Tips:
- Always quote variables in `[ ... ]` to avoid syntax errors: `[ "$x" = "y" ]`

---

## 1️⃣2️⃣ Case Statements

Cleaner for multiple discrete choices:

```bash
read -p "Enter y/n: " ans
case "$ans" in
  y|Y|yes) echo "Proceed";;
  n|N|no)  echo "Cancel";;
  *) echo "Unknown";;
esac
```

---

## 1️⃣3️⃣ Loops

For loop (list):

```bash
for i in 1 2 3; do echo "$i"; done
```

For loop (globbed files — safe):

```bash
for file in *.log; do
  [ -e "$file" ] || continue
  echo "Processing: $file"
done
```

C-style Bash loop:

```bash
for ((i=0;i<5;i++)); do echo $i; done
```

While loop:

```bash
n=1
while [ $n -le 5 ]; do
  echo "n=$n"
  n=$((n+1))
done
```

Until loop:

```bash
n=1
until [ $n -gt 5 ]; do
  echo $n
  n=$((n+1))
done
```

---

## 1️⃣4️⃣ Functions, Return Values & Scope

Define:

```bash
greet() {
  local name="$1"
  echo "Hello, $name"
}
greet "Ajay"
```

Notes:
- Use `local` to avoid polluting global variables.
- Functions can `return` an integer (0-255) as exit status.
  ```bash
  is_even() {
    local n=$1
    (( n % 2 == 0 ))
  }
  if is_even 4; then echo "even"; fi
  ```
- To return strings, echo and capture:
  ```bash
  get_date() { date '+%F'; }
  today=$(get_date)
  ```

---

## 1️⃣5️⃣ Arrays & Associative Arrays

Indexed arrays:

```bash
fruits=("apple" "banana" "mango")
echo "${fruits[0]}"      # apple
echo "${fruits[@]}"      # all items
for f in "${fruits[@]}"; do echo "$f"; done
```

Length:

```bash
echo "${#fruits[@]}"
```

Associative arrays (Bash 4+):

```bash
declare -A capitals
capitals[India]="New Delhi"
capitals[USA]="Washington"
echo "${capitals[India]}"
```

---

## 1️⃣6️⃣ Strings & Pattern Matching

Length, substring, replace:

```bash
s="LinuxShell"
echo "${#s}"           # length
echo "${s:0:5}"        # substr "Linux"
echo "${s/Shell/SCRIPT}"  # substitution
```

Globbing vs regex:
- Globbing: `*.txt` matches filenames
- Regex (within `[[ ... =~ ... ]]`) matches patterns

---

## 1️⃣7️⃣ Files & File Tests

Common file tests:

- `-f` file exists and is regular file
- `-d` directory exists
- `-r` readable
- `-w` writable
- `-x` executable
- `-s` file exists and not empty

Example:

```bash
if [ -d "$backup_dir" ]; then
  echo "Backup dir exists"
else
  mkdir -p "$backup_dir"
fi
```

Permissions check:

```bash
if [ -w "$file" ]; then echo "Can write"; fi
```

---

## 1️⃣8️⃣ Error Handling, Exit Codes & set options

Every command has an exit status (0 success, non-zero failure). Check via `$?`.

Useful options (place near top of scripts):

```bash
set -o errexit   # same as set -e  -> exit on any command failure
set -o nounset   # same as set -u  -> error on unset variables
set -o pipefail  # pipeline returns non-zero if any command fails
```

Combined:

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

`trap` handles cleanup:

```bash
cleanup() { rm -f "$tmpfile"; }
trap cleanup EXIT
```

Be careful with `set -e` and commands inside conditionals.

---

## 1️⃣9️⃣ Useful I/O Patterns & Here-documents

Here-doc:

```bash
cat <<'EOF' > /tmp/message.txt
Hello, $USER
This is a multi-line message.
EOF
```

Process substitution:

```bash
diff <(sort file1) <(sort file2)
```

Temporary files safely:

```bash
tmpfile=$(mktemp) || exit 1
trap 'rm -f "$tmpfile"' EXIT
```

---

## 2️⃣0️⃣ Best Practices & Script Template

Best practices:
- Use `#!/usr/bin/env bash` for portability
- `set -euo pipefail` and `IFS=$'\n\t'`
- Quote variable expansions: `"$var"`
- Use functions & `local` variables
- Validate user input
- Use `mktemp` for temp files
- Avoid parsing `ls` output
- Prefer `[[ ]]` in Bash
- Check return codes and handle errors gracefully

Starter template:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

usage() {
  echo "Usage: $0 <dest-dir>"
  exit 1
}

if [ $# -lt 1 ]; then usage; fi

main() {
  local dest="$1"
  mkdir -p "$dest"
  echo "Created $dest"
}

main "$@"
```

---

## 2️⃣1️⃣ Real-world Examples

1) Check if a user exists:

```bash
#!/usr/bin/env bash
read -rp "Enter username: " user
if id "$user" &>/dev/null; then
  echo "User $user exists"
else
  echo "User $user not found"
fi
```

2) Disk usage alert (email-friendly):

```bash
#!/usr/bin/env bash
threshold=80
use=$(df / | awk 'NR==2 {gsub(/%/,""); print $5}')
if [ "$use" -gt "$threshold" ]; then
  echo "Disk usage is ${use}% on $(hostname) at $(date)"
fi
```

3) Backup `/etc`:

```bash
#!/usr/bin/env bash
dest="/backup/etc-$(date +%F).tar.gz"
tar -czf "$dest" /etc
echo "Backup saved to $dest"
```

4) Batch rename `.txt` → `.bak` safely:

```bash
for f in *.txt; do
  [ -e "$f" ] || continue
  mv -- "$f" "${f%.txt}.bak"
done
```

---

## 2️⃣2️⃣ Practice Lab — Exercises & Solutions

Exercise 1: Table of a number (user input)
Solution:
```bash
#!/usr/bin/env bash
read -rp "Number: " n
for ((i=1;i<=10;i++)); do
  printf "%d x %d = %d\n" "$n" "$i" "$((n*i))"
done
```

Exercise 2: Biggest of 3 numbers
Solution:
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

Exercise 3: Rename `.txt` → `.bak`
Solution: see section 21 (batch rename)

Exercise 4: Monitor CPU or Memory usage
Solution (Memory example):
```bash
#!/usr/bin/env bash
mem_used=$(free -m | awk '/^Mem:/ {print $3}')
mem_total=$(free -m | awk '/^Mem:/ {print $2}')
percent=$(( mem_used * 100 / mem_total ))
echo "Memory: ${percent}% used (${mem_used}MB/${mem_total}MB)"
```

Exercise 5: Count files in a directory
Solution:
```bash
#!/usr/bin/env bash
dir="${1:-.}"
count=$(find "$dir" -maxdepth 1 -type f | wc -l)
echo "Files in $dir: $count"
```

---

## 2️⃣3️⃣ Troubleshooting Tips & FAQ

- "Script fails with `[: missing ']'`" → Ensure spaces around `[` `]` and quote variables.
- "Globs break on filenames with spaces" → loop with `for f in *.txt; do ...; done` and use `"$f"`.
- "Why does my variable expand empty?" → Check `set -u` (nounset) or ensure it was assigned; use `${var:-default}`.
- "Why `set -e` exits unexpectedly?" → `set -e` stops on non-zero exit; commands in `if` or `&&` chains may behave differently.

---

## 2️⃣4️⃣ Further Reading & Resources

- Bash reference manual: https://www.gnu.org/software/bash/manual/
- Advanced Bash-Scripting Guide: https://tldp.org/LDP/abs/html/
- ShellCheck (linting): https://www.shellcheck.net/
- Learn Bash Scripting — online tutorials & examples

---
Happy scripting! 🧑‍💻
