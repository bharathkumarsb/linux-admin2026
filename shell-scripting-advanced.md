# 🐚 Shell Scripting — Advanced Guide (Professional Level)

[![Level](https://img.shields.io/badge/Level-Advanced-orange)](https://www.gnu.org/software/bash/)
[![Platform](https://img.shields.io/badge/Platform-Linux%20%7C%20macOS-lightgrey)](#)
[![Style](https://img.shields.io/badge/Style-Production--Ready-blue)](#)

> This document is a continuation of the beginner guide. It focuses on production-ready patterns, robust error handling, testing, modular design, and automation workflows for real-world systems engineering.

---

## Table of Contents

1. Bash execution modes (safe & debug)
2. Advanced variables, quoting & scope
3. Here-documents & here-strings (practical patterns)
4. Redirection techniques & FD manipulations
5. Signal handling with `trap` (robust cleanup)
6. Logging framework & structured logs
7. Debugging shell scripts (tools & tips)
8. Command-line arguments, `$@` vs `$*`, `$#`
9. Professional argument parsing (getopts & getopt)
10. File processing patterns (safe loops)
11. Interactive menus & `select`
12. Associative arrays (hash maps) in practice
13. Subshell vs grouping & side effects
14. Background jobs, process control & concurrency
15. Parallel execution strategies
16. Working with JSON/CSV/text (jq, awk, csv)
17. Error handling frameworks & stack traces
18. Modular, reusable scripts & libraries
19. DevOps automation examples (production-ready)
20. Testing, CI, linting & security checks
21. Exercises (advanced) and suggested solutions
22. Appendix: common pitfalls & pro tips

---

## 1️⃣ Bash execution modes (safe & debug)

Production scripts must be defensive. Recommended top-of-script:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'
```

Meaning:
- `-e` / `errexit`: exit on first command failure (careful with conditionals)
- `-u` / `nounset`: fail on unset variables
- `-o pipefail`: pipeline returns first non-zero exit code
- `-E`: ERR trap propagates to functions
- `IFS=$'\n\t'`: safer default IFS to avoid word-splitting on spaces

Debugging modes:
- `set -x` prints commands; use `PS4` to add timestamps/line numbers:
  ```bash
  export PS4='+ ${BASH_SOURCE##*/}:${LINENO}:${FUNCNAME[0]}: '
  set -x
  # debug code
  set +x
  ```
- Syntax check: `bash -n script.sh` (no execution)

Caveat: `set -e` behavior is subtle in compound commands — prefer explicit status checks in critical areas.

---

## 2️⃣ Advanced variables, quoting & scope

Key rules:
- Always quote expansions unless deliberate: `"$var"`
- Use parameter expansion defaults: `${var:-default}`, `${var:?error message}`
- Use `local` for function-scoped variables:

```bash
myfunc() {
  local tmp="$(mktemp)"
  echo "Using $tmp"
}
```

Read-only consts:
```bash
readonly APP_NAME="myapp"
```

Arrays and slicing:

```bash
arr=(one "two words" three)
echo "${arr[1]}"      # two words
echo "${arr[@]:1:2}"  # slice
```

Avoid global pollution; prefer clear naming or namespaces: `lib_` prefix.

---

## 3️⃣ Here-documents & here-strings (practical patterns)

Heredoc with and without expansion:

```bash
cat <<EOF > config.ini
[main]
name = $APP_NAME
EOF

cat <<'EOF' > literal.txt
${NOT_EXPANDED}
EOF
```

Here-strings (useful for small input):

```bash
grep -E 'pattern' <<<"$var"
```

Use heredocs for templating small files or embedding scripts; prefer `'EOF'` when you want literal contents.

---

## 4️⃣ Redirection techniques & FD manipulations

Standard redirection patterns:

- Combine stdout & stderr to a log:
  ```bash
  exec > >(tee -a "$LOGFILE") 2>&1
  ```

- Redirect a command's stderr to file:
  ```bash
  cmd 2>errors.log
  ```

- Use file descriptors for granular control:
  ```bash
  exec 3>&1   # save stdout to FD 3
  exec 1>log.txt
  # send message to original stdout
  echo "Notice" >&3
  ```

- Atomic writes: write to temp file and move into place to avoid partial writes.

---

## 5️⃣ Signal handling with `trap` (robust cleanup)

Always clean resources (temp files, locks) and optionally dump debug info.

Example robust trap skeleton:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
trap 'rc=$?; on_exit $rc' EXIT
trap 'on_signal TERM' TERM
trap 'on_signal INT'  INT

on_exit() {
  local rc=${1:-0}
  # cleanup resources
  rm -f "$tmpfile" "$lockfile"
  if [ "$rc" -ne 0 ]; then
    echo "Failure (exit $rc)"
  fi
  return $rc
}

on_signal() {
  echo "Received signal, cleaning..."
  exit 2
}
```

Use `trap '...; exit 130' INT` to standardize exit codes for signals.

---

## 6️⃣ Logging framework & structured logs

Use a small logging library for consistent logs and optional JSON output.

Simple logger:

```bash
log() { printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*"; }
info() { log "[INFO] $*"; }
warn() { log "[WARN] $*" >&2; }
error() { log "[ERROR] $*" >&2; }
```

Structured JSON logging (for machines):

```bash
json_log() {
  local level="$1"; shift
  jq -n --arg lvl "$level" --arg msg "$*" \
    '{timestamp:now|todate, level:$lvl, message:$msg}'
}
```

Rotate logs or send to syslog for production systems.

---

## 7️⃣ Debugging shell scripts (tools & tips)

- `bash -n script.sh` — syntax check
- `set -x` / `PS4` — trace with context
- `shellcheck` — static analysis (fix warnings)
- `strace` / `ltrace` — system call tracing for complex failures
- Insert debug dumps on failure:
  ```bash
  trap 'echo "DEBUG: failed at ${BASH_SOURCE[0]}:${LINENO}"; env' ERR
  ```

When reproducing issues: capture inputs, environment, and steps inside a reproducible test.

---

## 8️⃣ Command-line arguments: `$@` vs `$*`, `$#`

Differences:
- `"$@"` expands to `"$1" "$2" ...` — preserves argument boundaries (preferred)
- `"$*"` expands to `"$1c$2c..."` (where `c` is the first char of IFS) — rarely desired
- `$#` number of arguments
- `$0` script name (may be path)

Proper passing:

```bash
some_command "$@"
```

Avoid `for arg in $@` (loses quoting). Use `for arg in "$@"; do ...; done`.

---

## 9️⃣ Professional argument parsing (getopts & getopt)

`getopts` (POSIX, recommended for short options):

```bash
while getopts ":u:p:hv" opt; do
  case $opt in
    u) user="$OPTARG" ;;
    p) pass="$OPTARG" ;;
    h) show_help; exit 0 ;;
    v) verbose=1 ;;
    \?) echo "Invalid option: -$OPTARG" >&2; exit 1;;
    :)  echo "Option -$OPTARG requires an argument." >&2; exit 1;;
  esac
done
shift $((OPTIND -1))
```

For long options on Linux, use GNU `getopt` carefully:

```bash
PARSED=$(getopt --options "" --longoptions "user:,password:,help" -- "$@") || exit 1
eval set -- "$PARSED"
while true; do
  case "$1" in
    --user) USER="$2"; shift 2 ;;
    --password) PASS="$2"; shift 2 ;;
    --) shift; break ;;
  esac
done
```

Always validate and document accepted options; provide `--help`.

---

## 🔟 File processing patterns (safe loops)

Safe read loop for files with spaces:

```bash
while IFS= read -r line; do
  process_line "$line"
done < "$file"
```

When iterating files by glob:

```bash
shopt -s nullglob
for f in /path/*.log; do
  [ -f "$f" ] || continue
  echo "Processing: $f"
done
```

Prefer `find -print0 | xargs -0` or `while read -d '' -r file` for deep directory traversal with safe filenames.

---

## 1️⃣1️⃣ Interactive menus & `select`

Build lightweight interactive menus for admins:

```bash
PS3="Choose action: "
select opt in "Start" "Stop" "Status" "Quit"; do
  case $opt in
    "Start") start_service; break;;
    "Stop") stop_service; break;;
    "Status") status_service;;
    "Quit") break;;
    *) echo "Invalid option";;
  esac
done
```

`select` auto-numbers items; always validate user selection.

---

## 1️⃣2️⃣ Associative arrays (hash maps) in practice

Create configuration maps:

```bash
declare -A conf=(
  [db_host]="db.example.local"
  [db_port]=3306
)
echo "DB host: ${conf[db_host]}"
```

Iterate:

```bash
for k in "${!conf[@]}"; do
  echo "$k -> ${conf[$k]}"
done
```

Useful for dynamic lookups, command templates, mappings.

---

## 1️⃣3️⃣ Subshell vs grouping & side effects

- Subshell `( ... )` runs in child process; changes to environment don't persist.
- Grouping `{ ... ; }` runs in current shell.

Example difference:

```bash
(cd /tmp; echo "pwd inside subshell: $(pwd)")
echo "pwd after subshell: $(pwd)"   # unchanged

{ cd /tmp; echo "pwd inside group: $(pwd)"; }
echo "pwd after group: $(pwd)"     # changed
```

Use subshells to contain side effects, groups to change working dir deliberately.

---

## 1️⃣4️⃣ Background jobs, process control & concurrency

Use `&`, `wait`, `jobs`, `fg`, `bg`. Example launching tasks and waiting:

```bash
long_task & pid1=$!
another_task & pid2=$!
wait $pid1 $pid2
echo "All tasks done"
```

Use `nohup` or `disown` for persistent background tasks. Use `flock` to enforce single-instance scripts:

```bash
(
  flock -n 9 || { echo "Already running"; exit 1; }
  # critical section
) 9>/var/lock/mytask.lock
```

---

## 1️⃣5️⃣ Parallel execution strategies

Small concurrency pattern (limit to N jobs):

```bash
maxjobs=4
for item in "${items[@]}"; do
  (
    process "$item"
  ) &
  while (( $(jobs -rp | wc -l) >= maxjobs )); do
    sleep 0.2
  done
done
wait
```

Better: use `xargs -P` or `GNU parallel` for heavy use:

```bash
printf "%s\n" "${items[@]}" | xargs -n1 -P4 -I{} bash -c 'process "$@"' _ {}
```

Note: jobs built with `&` may produce interleaved output; consider per-job logs.

---

## 1️⃣6️⃣ Working with JSON / CSV / text

JSON with `jq`:

```bash
jq -r '.items[] | "\(.id) \(.name)"' data.json
```

CSV parsing with `awk` (simple) or `csvkit` for complex CSVs:

```bash
awk -F, 'NR>1 {print $1,$3}' file.csv
```

Text streaming with `awk` and `sed`:

```bash
awk '{if ($3 > 100) print $0}' access.log
sed -n '1,100p' file.txt
```

Prefer specialized tools for structured formats; avoid brittle ad-hoc parsing for production workloads.

---

## 1️⃣7️⃣ Error handling frameworks & stack traces

Add a generic `on_error` handler to show failed command and stack:

```bash
on_error() {
  local rc=$?
  echo "Error: command failed with exit $rc" >&2
  echo "Trace:"
  local i=0
  while caller $i; do ((i++)); done
  exit $rc
}
trap on_error ERR
```

Wrap external commands with a `run()` helper to annotate failures:

```bash
run() {
  "$@" || { echo "Command failed: $*"; return 1; }
}
run cp /src /dest
```

Use exit codes consistently and define them in a header.

---

## 1️⃣8️⃣ Modular, reusable scripts & libraries

Structure repo:

```
bin/         # executable scripts
lib/         # reusable shell functions (sourced)
tests/       # bats tests
ci/          # CI scripts
docs/        # documentation
```

Example `lib/log.sh`:

```bash
# lib/log.sh
log() { printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*"; }
```

Source with defensive checks:

```bash
source "$(dirname "${BASH_SOURCE[0]}")/../lib/log.sh"
```

Avoid `source` of untrusted files. Keep functions small and single-purpose.

---

## 1️⃣9️⃣ DevOps automation examples (production-ready)

1) Restart service with retry & log:

```bash
svc=nginx
if ! systemctl is-active --quiet "$svc"; then
  systemctl restart "$svc" && log "Restarted $svc" || {
    log "Failed to restart $svc"; exit 1
  }
fi
```

2) Backup with retention & lock:

```bash
backup() {
  local src="$1" dest="$2"
  local ts; ts=$(date -u +%Y%m%dT%H%M%SZ)
  local archive="$dest/$(basename "$src")-$ts.tar.gz"
  tar -C "$src" -czf "$archive" .
  find "$dest" -type f -name '*.tar.gz' -mtime +30 -delete
}
# Run with flock to prevent concurrent backups
(
  flock -n 200 || { echo "Another backup running"; exit 1; }
  backup /etc /var/backups
) 200>/var/lock/backup.lock
```

3) Parallel ping scanner using xargs:

```bash
seq 1 254 | xargs -P50 -I{} -n1 bash -c 'ping -c1 -W1 192.168.1.{} >/dev/null && echo 192.168.1.{}'
```

4) MySQL dump with rotation and secure permissions:

```bash
mysqldump --single-transaction --quick --lock-tables=false -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" > "$file"
chmod 600 "$file"
```

Never store plain-text credentials in repo; use secret managers / env vars.

---

## 2️⃣0️⃣ Testing, CI, linting & security checks

Testing:
- Use `bats-core` for unit tests of scripts.
- Write integration tests in ephemeral containers (Docker).

Linting:
- `shellcheck` as a CI step; fix warnings (pay attention to SC2086, SC2046, SC2006, etc.)

CI example (GitHub Actions steps):
- Install `shellcheck`
- Run `shellcheck` on `bin/` and `lib/`
- Install `bats-core`
- Run `bats tests/`

Security checks:
- Avoid `eval` on untrusted input
- Validate external inputs (regex check)
- Use least privilege when performing operations
- Sanitize filenames and user input (avoid `;`, `|`, `&`, backticks)

---

## 2️⃣1️⃣ Advanced Practice Exercises (with hints)

1. Build a script that runs a list of commands in parallel with a concurrency limit and a per-job timeout. Hint: use `xargs -P` or manage background PIDs + `timeout`.

2. Write a CLI tool with `getopts` that supports `--dry-run`, `--config FILE`, `--verbose` and command sub-commands (`start|stop|status`). Hint: parse options first, then subcommand.

3. Create a backup script that uploads archives to S3 (use `aws cli`) with retry/backoff and idempotency. Hint: generate deterministic object keys and use `aws s3 cp`.

4. Implement a JSON-to-CSV converter using `jq`. Hint: produce header row first and then iterate items.

5. Implement a safe `deploy` script that uses `flock` to avoid concurrent deploys, writes logs, can rollback on failure, and notifies via webhook on completion. Hint: use `trap` + atomic symlink switching pattern.

If you want, I can provide full solutions and accompanying `bats` tests and a CI workflow for these exercises.

---

## 2️⃣2️⃣ Appendix — Common pitfalls & pro tips

- Pitfall: unquoted expansions — leads to word splitting/glob issues. Always quote: `"$var"`.
- Pitfall: parsing `ls` output — use `find` or `stat`.
- Pitfall: `set -e` surprises — commands in conditionals or subtleties with pipes can be tricky.
- Pitfall: missing `mktemp` cleanup — always trap and remove temporary artifacts.
- Pro tip: keep scripts idempotent where possible.
- Pro tip: use small functions and keep scripts under ~300 LOC — split libraries for complex logic.
- Pro tip: record environment and versions in logs for reproducibility.

---

## 2️⃣3️⃣ Suggested production-ready script template

```bash
#!/usr/bin/env bash
# myscript.sh - short description
set -Eeuo pipefail
IFS=$'\n\t'

# Metadata
APP_NAME="myscript"
VERSION="0.1.0"

# Constants & defaults
LOGFILE="/var/log/${APP_NAME}.log"

# Helpers
log() { printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*"; }
err() { log "[ERROR] $*" >&2; }

usage() {
  cat <<EOF
Usage: $0 [-n|--dry-run] [-v|--verbose] <command>
Commands: start|stop|status
EOF
}

# parse options (example minimal)
dryrun=0
while getopts ":nv-:" opt; do
  case $opt in
    n) dryrun=1 ;;
    v) set -x ;;
    -) case "${OPTARG}" in
         dry-run) dryrun=1 ;;
         verbose) set -x ;;
       esac ;;
    *) usage; exit 1 ;;
  esac
done
shift $((OPTIND-1))

# Main
main() {
  # application logic
  log "Starting $APP_NAME v$VERSION"
}

trap 'err "Unexpected error"; exit 1' ERR
trap 'log "Exiting";' EXIT

main "$@"
```

---

## 2️⃣4️⃣ Further reading & tools

- Bash Reference Manual — https://www.gnu.org/software/bash/manual/
- Advanced Bash-Scripting Guide — https://tldp.org/LDP/abs/html/
- ShellCheck — https://www.shellcheck.net/
- jq manual — https://stedolan.github.io/jq/manual/
- bats-core (testing) — https://bats-core.readthedocs.io/

---
- Walk through one of the advanced exercises with a complete solution, tests and CI.

Which of the above should I generate next?
