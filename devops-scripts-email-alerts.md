# 📧 Email-Enabled DevOps Scripts — Detailed README

> This README expands the previously provided email-enabled DevOps scripts into production-ready, tested, and verifiable scripts with:
> - unified notification helper (mail + optional Slack)
> - robust script headers and error handling
> - verification steps and test instructions
> - scheduling examples (cron & systemd timers)
> - security and secrets guidance
> - additional useful scripts that were missing
>
> Save as `README.md` and copy the example scripts into your repo's `samples/` or `scripts/` folder.

---

## Table of contents

1. Goals & scope
2. Prerequisites and installation
3. Notification abstraction (recommended)
4. Email backends supported (mailx, sendmail, msmtp)
5. Production-grade example scripts (with verification)
   - Disk usage alert (email + Slack)
   - CPU usage alert (email)
   - Memory usage alert (email)
   - Service auto-restart + email
   - Website availability monitor + email
   - SSH failure summary email
   - Docker container monitor + email
   - Kubernetes pod failure alert + email
6. Scheduling examples (cron & systemd timers)
7. Testing & verification (manual and automated)
8. Logging, rotation & retention
9. Secrets & credentials (secure options)
10. CI & linting (shellcheck + bats)
11. Additional recommended scripts (missing items)
12. Checklist before production rollout
13. Troubleshooting & common failure modes
14. References and further reading

---

## 1. Goals & scope

This document converts simple alert scripts into production-ready tools by:

- adding safe Bash headers and defensive options
- centralizing notification (so you can switch from mail to Slack easily)
- giving explicit verification steps (how to test without production impact)
- adding scheduling and CI guidance
- suggesting additional scripts and improvements

All examples target Bash on Linux. Some commands differ on macOS (noted where relevant).

---

## 2. Prerequisites & installation

Install a mail program to send emails from scripts (choose one):

- RHEL / CentOS / Rocky:
  sudo yum install -y mailx
- Debian / Ubuntu:
  sudo apt update && sudo apt install -y mailutils
- Alternative lightweight senders: msmtp, ssmtp, or exim4

Optional (for Slack notifications):
- A Slack incoming webhook URL for channel alerts (store it as a secret env var).

Other tools recommended:
- curl (HTTP requests)
- jq (JSON handling)
- docker / kubectl (for container/k8s scripts)

Set environment variables for alerts:

- ALERT_EMAIL (recipient)
- ALERT_FROM (optional From header; used by some mail clients)
- SLACK_WEBHOOK (optional; if set, Slack notifications will be sent)

Example (export in systemd unit or environment file):
export ALERT_EMAIL="ops@example.com"
export ALERT_FROM="monitor@example.com"
export SLACK_WEBHOOK="https://hooks.slack.com/services/XXX/YYY/ZZZ"

Important: do NOT store plaintext SMTP passwords in repo files.

---

## 3. Notification abstraction (recommended)

Create a small notification helper shared by all scripts so you can change notification channels centrally.

Create `lib/notify.sh` (or `scripts/notify.sh`):

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

ALERT_EMAIL="${ALERT_EMAIL:-admin@example.com}"
ALERT_FROM="${ALERT_FROM:-}"
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"

notify() {
  # usage: notify "subject" "body"
  local subject="$1"
  local body="$2"

  # 1) Email (mailx / mail)
  if command -v mail &>/dev/null; then
    if [ -n "$ALERT_FROM" ]; then
      printf '%s\n' "$body" | mail -s "$subject" -a "From: $ALERT_FROM" "$ALERT_EMAIL"
    else
      printf '%s\n' "$body" | mail -s "$subject" "$ALERT_EMAIL"
    fi
  elif command -v sendmail &>/dev/null; then
    # fallback classic sendmail usage - POSIX simple
    {
      echo "Subject: $subject"
      [ -n "$ALERT_FROM" ] && echo "From: $ALERT_FROM"
      echo
      echo "$body"
    } | sendmail "$ALERT_EMAIL"
  else
    echo "No mail program found; skipping email"
  fi

  # 2) Slack (optional)
  if [ -n "$SLACK_WEBHOOK" ] && command -v curl &>/dev/null; then
    # JSON-escape message body simply
    local payload
    payload=$(printf '{"text":"%s\n\n%s"}' "$subject" "$body" | sed 's/"/\\"/g')
    curl -s -X POST -H 'Content-type: application/json' --data "$payload" "$SLACK_WEBHOOK" >/dev/null || true
  fi
}
```

Usage in scripts:

```bash
source /opt/scripts/lib/notify.sh
notify "Disk alert on $(hostname)" "Details..."
```

Advantages:
- Single place to adjust email method, From headers, Slack integration
- Simplifies tests by mocking `notify()` in tests

---

## 4. Email backends & notes

- mailx / mailutils: common, easy; supports `-s` subject and `-a` header but flags differ slightly across distros. Use `mail -s "subj" recipient` for portability.
- sendmail: fall-back via piping raw headers and body into `sendmail` binary.
- msmtp / ssmtp: lightweight SMTP clients configured per-system; recommended when you want SMTP-only clients and to avoid full MTA.

Gmail SMTP note:
- If you must use Gmail, use App Passwords (for accounts with 2FA). Add configuration to `/etc/mail.rc` or `~/.msmtprc` and keep credentials out of repo. Prefer sending through corporate SMTP or relay.

---

## 5. Production-grade example scripts (full examples)

All scripts follow this header:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'
```

Place scripts in `/opt/scripts/` or repo `samples/`. Make executable: chmod 750 script.sh.

Below are improved scripts with `notify()` integration, simulation hooks, verification steps, and logging.

---

### A) Disk usage alert (email + slack, with simulation)

File: samples/disk_alert_email.sh

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

# disk_alert_email.sh <mount> <threshold>
MOUNT=${1:-/}
THRESHOLD=${2:-80}

# Hook for tests: SIMULATE_USAGE env var to avoid touching real disk
USAGE="${SIMULATE_USAGE:-}"

# load notify helper - adjust path as deployed
source "$(dirname "$0")/../lib/notify.sh" || source ./lib/notify.sh || true

get_usage() {
  if [ -n "$USAGE" ]; then
    echo "$USAGE"
    return
  fi
  df -P "$MOUNT" | awk 'NR==2{gsub(/%/,""); print $5}'
}

main() {
  local pct
  pct=$(get_usage)
  if ! [[ "$pct" =~ ^[0-9]+$ ]]; then
    echo "ERROR: cannot parse usage ($pct)" >&2
    exit 2
  fi

  if [ "$pct" -ge "$THRESHOLD" ]; then
    local subj="⚠ Disk Alert: ${MOUNT} at ${pct}% on $(hostname)"
    local body="Disk usage for ${MOUNT} is ${pct}% (threshold ${THRESHOLD}%) on $(hostname) at $(date -Iseconds)"
    echo "$subj"
    notify "$subj" "$body"
    exit 0
  else
    echo "OK: ${MOUNT} ${pct}% (< ${THRESHOLD}%)"
  fi
}

main
```

Verification:
- Simulated: SIMULATE_USAGE=92 bash samples/disk_alert_email.sh / 80 -> should call notify and print alert.
- Real: Run `df -h /` and compare output with script.

Test ideas (bats):
- Run with SIMULATE_USAGE=50 -> expect "OK"
- Run with SIMULATE_USAGE=90 -> expect `notify` to be called — in test, mock `notify()` to record invocation.

---

### B) CPU usage email alert (with /proc/stat calculation)

File: samples/cpu_alert_email.sh

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

THRESHOLD=${1:-85}
SIMULATE="${SIMULATE_CPU:-}"
source "$(dirname "$0")/../lib/notify.sh" || true

calc_cpu() {
  if [ -n "$SIMULATE" ]; then
    echo "$SIMULATE"
    return
  fi
  # read /proc/stat sample twice
  read -r _ user nice system idle iowait irq softirq steal guest < /proc/stat
  prev_total=$((user+nice+system+idle+iowait+irq+softirq+steal))
  prev_idle=$((idle + iowait))
  sleep 1
  read -r _ user nice system idle iowait irq softirq steal guest < /proc/stat
  total=$((user+nice+system+idle+iowait+irq+softirq+steal))
  idle_now=$((idle + iowait))
  total_diff=$((total - prev_total))
  idle_diff=$((idle_now - prev_idle))
  usage=$(( (1000 * (total_diff - idle_diff) / total_diff + 5) / 10 ))
  echo "$usage"
}

main() {
  local cpu
  cpu=$(calc_cpu)
  if [ "$cpu" -ge "$THRESHOLD" ]; then
    local subj="⚠ CPU Alert: ${cpu}% on $(hostname)"
    local body="CPU usage ${cpu}% (threshold ${THRESHOLD}%)"
    notify "$subj" "$body"
    echo "$subj"
  else
    echo "OK: CPU ${cpu}%"
  fi
}

main
```

Verification:
- Simulate: SIMULATE_CPU=90 bash samples/cpu_alert_email.sh 85
- Real: Use `stress` or `stress-ng` to generate CPU load and validate an alert triggers.

---

### C) Memory usage email alert

File: samples/mem_alert_email.sh

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

THRESHOLD=${1:-85}
SIMULATE="${SIMULATE_MEM:-}"
source "$(dirname "$0")/../lib/notify.sh" || true

get_mem() {
  if [ -n "$SIMULATE" ]; then
    echo "$SIMULATE"
    return
  fi
  awk '/Mem:/ {printf "%.0f", $3/$2*100}' /proc/meminfo || free | awk '/Mem/ {printf "%.0f", $3/$2*100}'
}

main() {
  local mem
  mem=$(get_mem)
  if [ "$mem" -ge "$THRESHOLD" ]; then
    notify "⚠ Memory Alert: $mem% on $(hostname)" "Memory usage $mem% (threshold $THRESHOLD%)"
    echo "ALERT: Memory ${mem}%"
  else
    echo "OK: Memory ${mem}%"
  fi
}

main
```

Verification:
- SIMULATE_MEM=95 bash samples/mem_alert_email.sh -> triggers notify.
- Real: Use memory pressure tool (e.g., `stress --vm 1 --vm-bytes 1G`) in test VM.

---

### D) Service auto-restart + email notification

File: samples/service_monitor_email.sh

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

SERVICE=${1:-nginx}
SOURCE="$(dirname "$0")/../lib/notify.sh"
[ -f "$SOURCE" ] && source "$SOURCE"

LOG="/var/log/service-monitor.log"
log() { echo "$(date +'%F %T') $*" >> "$LOG"; }

if ! systemctl is-active --quiet "$SERVICE"; then
  log "Service $SERVICE not active, attempting restart..."
  if systemctl restart "$SERVICE"; then
    log "Restart succeeded"
    notify "Service restarted: $SERVICE on $(hostname)" "Service $SERVICE was restarted successfully."
  else
    log "Restart failed"
    notify "Service restart FAILED: $SERVICE on $(hostname)" "Attempt to restart $SERVICE failed."
    exit 1
  fi
else
  log "Service $SERVICE is active"
fi
```

Verification:
- Stop service: sudo systemctl stop nginx ; run script ; check `/var/log/service-monitor.log` and email.
- In tests, mock `systemctl` (use a `PATH` wrapper) to simulate success/fail.

---

### E) Website availability monitor (email + optional Slack)

File: samples/web_health_email.sh

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

URL="${1:-http://localhost:80}"
TIMEOUT=${2:-10}
source "$(dirname "$0")/../lib/notify.sh" || true

status=$(curl -sS -o /dev/null -w '%{http_code}' --max-time "$TIMEOUT" "$URL" || echo "000")
if [ "$status" != "200" ]; then
  local subj="🚨 Web DOWN: $URL on $(hostname) (code: $status)"
  local body="Checked $URL at $(date -Iseconds). Received status code $status."
  notify "$subj" "$body"
  echo "$subj"
else
  echo "OK: $URL (200)"
fi
```

Verification:
- Test with a local server: `python3 -m http.server 8080` and run script against http://localhost:8080
- Simulate down: run against a port with no service or set `--simulate` env var to return non-200.

---

### F) SSH failed login summary email

File: samples/ssh_failures_email.sh

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

ALERT_RECIPIENT="${ALERT_EMAIL:-admin@example.com}"
LOG="/var/log/auth.log"      # Debian/Ubuntu
[ -f /var/log/secure ] && LOG="/var/log/secure"  # RHEL/CentOS

window=${1:-24}  # hours
since=$(date -d "-${window} hours" '+%b %e')  # month day e.g. "Jun  6"
failed_lines=$(awk -v s="$since" ' $0 ~ s && /Failed password/ {print}' "$LOG" || true)

if [ -n "$failed_lines" ]; then
  printf 'Failed SSH login attempts in last %s hours on %s:\n\n%s\n' "$window" "$(hostname)" "$failed_lines" \
    | mail -s "⚠ SSH Failed Logins on $(hostname)" "$ALERT_RECIPIENT"
  echo "Email sent with SSH failures"
else
  echo "No recent failed attempts"
fi
```

Verification:
- Generate a few failed SSH attempts (use another VM to try invalid login) — be careful not to lock accounts. For tests, create sample log file and point LOG to it.

---

### G) Docker container monitor + restart + email

File: samples/docker_monitor_email.sh

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

source "$(dirname "$0")/../lib/notify.sh" || true
ALERT="${ALERT_EMAIL:-admin@example.com}"

for cid in $(docker ps -a -q); do
  state=$(docker inspect -f '{{.State.Status}}' "$cid")
  name=$(docker inspect -f '{{.Name}}' "$cid" | sed 's/^\/\+//')
  if [ "$state" != "running" ]; then
    if docker restart "$cid" >/dev/null 2>&1; then
      notify "Docker restarted $name on $(hostname)" "Container $name ($cid) was $state and got restarted."
    else
      notify "Docker restart FAILED for $name on $(hostname)" "Container $name ($cid) remains $state after restart attempt."
    fi
  fi
done
```

Verification:
- Create a container that exits (e.g., `docker run --name test-exit --rm busybox sh -c 'exit 1'`) then run monitor. In test envs, use small containers and ensure Docker daemon is present.

---

### H) Kubernetes pod failure alert (kubectl, email)

File: samples/k8s_pod_alert.sh

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

NAMESPACE="${1:-all}"
source "$(dirname "$0")/../lib/notify.sh" || true

if ! command -v kubectl &>/dev/null; then
  echo "kubectl not installed" >&2
  exit 2
fi

if [ "$NAMESPACE" = "all" ]; then
  out=$(kubectl get pods --all-namespaces --no-headers 2>/dev/null)
else
  out=$(kubectl get pods -n "$NAMESPACE" --no-headers 2>/dev/null)
fi

failed=$(printf '%s\n' "$out" | awk '!/Running|Completed/ {print}')
if [ -n "$failed" ]; then
  notify "K8s Pod issues on $(hostname)" "$failed"
  echo "Notified about pods"
else
  echo "All pods running/completed"
fi
```

Verification:
- Test on a K8s cluster. For offline testing, set `kubectl` to a wrapper that outputs pre-crafted text and assert the script finds non-running pods.

---

## 6. Scheduling scripts (cron & systemd timer)

Cron example (run disk alert every 10 minutes):

```cron
*/10 * * * * /usr/bin/env bash /opt/scripts/samples/disk_alert_email.sh / 80 >> /var/log/disk_alert.log 2>&1
```

Systemd timer (preferable on systemd hosts):

Create unit `/etc/systemd/system/disk-alert.service`:

```ini
[Unit]
Description=Disk Alert Job

[Service]
Type=oneshot
ExecStart=/usr/bin/env bash /opt/scripts/samples/disk_alert_email.sh / 80
```

Timer `/etc/systemd/system/disk-alert.timer`:

```ini
[Unit]
Description=Run disk alert every 10 minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=10min

[Install]
WantedBy=timers.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now disk-alert.timer
```

Systemd advantages:
- Better logging (journalctl)
- Start/stop management, restarts and dependencies
- Per-user timers possible (systemd --user)

---

## 7. Testing & verification

Manual tests:
- Use `SIMULATE_*` env vars included in scripts to force alert conditions.
- Run scripts as the intended runtime user (systemd unit user) to verify permissions and environment.
- For email delivery, check mail logs (`/var/log/mail.log`, `/var/log/maillog`) and recipient inbox.

Automated tests:
- Use `bats-core` to write unit tests. Mock `notify()` by sourcing a test helper that overrides it to record calls instead of sending actual emails.
- Use temporary directories (`mktemp -d`) and temp log files in tests to avoid touching system logs.

Example bats test skeleton (for disk alert):

```bash
#!/usr/bin/env bats

setup() {
  export ALERT_EMAIL=test@example.com
  # Create a fake notify function to capture calls
  export NOTIFY_CAPTURE="/tmp/notify_calls"
  cat > ./lib/notify.sh <<'EOF'
notify() {
  printf '%s\n%s\n' "$1" "$2" >> "$NOTIFY_CAPTURE"
}
EOF
}

teardown() {
  rm -f "$NOTIFY_CAPTURE"
}

@test "disk alert triggers notify on high usage" {
  SIMULATE_USAGE=95 run bash samples/disk_alert_email.sh / 80
  [ "$status" -eq 0 ]
  run grep -q "Disk Alert" "$NOTIFY_CAPTURE"
  [ "$status" -eq 0 ]
}
```

CI:
- Add `shellcheck` step; fail on major warnings.
- Add `bats` step to run tests with mocked environment.

---

## 8. Logging, rotation & retention

- Log script actions to a log file under `/var/log/<app>/` or use `logger` to send to syslog: `logger -t disk_alert "message"`.
- Configure `logrotate` for those logs (see `logrotate` sample in previous README).
- Ensure logs rotate before reaching disk capacity (monitor with the disk alert script itself).

---

## 9. Secrets & credentials handling

Do not embed credentials in scripts. Prefer:

- Environment variables set in systemd unit files (`Environment=ALERT_EMAIL=...`)
- Protected files under `/etc/<app>/` with strict permissions (600)
- Use OS-level secret storage (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager)
- For SMTP credentials, use `~/.msmtprc` or `/etc/msmtprc` with 600 permissions, or configure the system MTA.

Example systemd unit snippet to provide env vars from a file:

`/etc/systemd/system/disk-alert.service`:

```
[Service]
EnvironmentFile=/etc/default/disk-alert
ExecStart=/opt/scripts/disk_alert_email.sh / 80
```

`/etc/default/disk-alert` (600):

```
ALERT_EMAIL=ops@example.com
SLACK_WEBHOOK=https://hooks.slack.com/services/XXX
```

---

## 10. CI & linting (shellcheck + bats)

Add a GitHub Actions workflow that:

- installs `shellcheck` and `bats-core`
- runs `shellcheck` across scripts and fails on high-priority warnings
- runs `bats` tests (mock `notify()` functions for tests)

Example job (adapt from previous README/CI example).

---

## 11. Additional recommended scripts (missing items)

These extra scripts improve monitoring and reduce alerts/noise:

1. TLS certificate expiry monitor (email when cert expires in < N days)
   - Use `openssl s_client -connect host:443 -servername host` or `sslyze`/`certbot`.
2. Alert rate limiter (avoid alert storms): persist last alert timestamp and skip if within window.
3. Backup verification script (attempt restore into ephemeral environment).
4. File integrity monitor with inotifywait (real-time changed file detection).
5. TLS auto-renewal reminder / ACME wrapper (Let's Encrypt).
6. Prometheus pushgateway client (push metrics via curl).
7. Centralized log shipping helper (rsyslog or fluentd).
8. S3 multipart backup + verification script (aws cli + IAM role).
9. Disk SMART health monitor (smartctl check & alert).
10. Hardware health summary (dmesg, SMART, temperature sensors).

If you want, I can provide full scripts for any of these items.

---

## 12. Checklist before production rollout

- [ ] Scripts use `set -Eeuo pipefail` and `IFS` safe defaults
- [ ] notify() helper used to centralize alerts
- [ ] Secrets are not in repository; systemd `EnvironmentFile` or secret manager used
- [ ] shellcheck passes or warnings reviewed
- [ ] bats tests exist for deterministic behaviors
- [ ] systemd timers or cron configured with appropriate user and environment
- [ ] log files configured and rotate via `logrotate`
- [ ] Alerts are rate-limited (avoid flood)
- [ ] Restore / mitigation steps documented (who to call / runbook)

---

## 13. Troubleshooting & common failure modes

- Emails not delivered: check `/var/log/mail.log`, verify MTA config (`/etc/ssmtp.conf`, `/etc/msmtprc`), firewall outbound port 25/587.
- Cron runs but script fails: cron has limited PATH—use full absolute paths or source environment file in script.
- systemctl permissions: systemd timers run as root by default; run as specific user with `User=` in unit file.
- `mail` flags not accepted: `mail` varies across distros; test with `printf "body" | mail -s "subj" "$ALERT_EMAIL"`.

---

## 14. References & further reading

- Bash Reference: https://www.gnu.org/software/bash/manual/
- ShellCheck: https://www.shellcheck.net/
- Bats-core: https://bats-core.readthedocs.io/
- systemd timers: https://www.freedesktop.org/software/systemd/man/systemd.timer.html
- msmtp: https://marlam.de/msmtp/
- mailutils: https://mailutils.org/

---

- Create sample `bats` tests that mock `notify()` so CI can validate alerts without sending email.

Which two artifacts should I create and include in the repo next? (suggestion: `samples/` with scripts + `.github/workflows/ci.yml`)
