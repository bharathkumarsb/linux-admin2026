# 🚀 Real-Time DevOps Shell Scripts — Detailed Guide (Production-ready)

> A comprehensive, actionable README for a collection of production-style Bash scripts used for Linux administration, monitoring, backup, recovery, CI/CD and automation.  
> This document contains: polished script examples, safe production patterns, verification steps, testing tips, alerting integrations (Slack/email), scheduling, security guidance, CI & linting, and additional recommended scripts you should include.

---

## Quick index

- Purpose & scope
- Safety & testing best practices
- How to use this repo
- Script catalog (short summary + production-ready pattern)
- For each high-value script:
  - Production-grade script example
  - How to run (manual)
  - How to verify (test)
  - Test/CI ideas
  - Common improvements
- Alerting & notification (Slack / Email / Webhook)
- Scheduling (cron & systemd timers)
- Logging, rotation & retention
- Secrets & credentials handling
- Unit testing & CI (bats + shellcheck)
- Deployment/packaging recommendations
- Additional suggested scripts (missing / useful)
- Troubleshooting & FAQ
- References & further reading

---

## Purpose & scope

This README documents a set of Real‑Time DevOps Shell Scripts for admins and SREs. The goal is to provide ready-to-run, safe, and testable scripts with clear verification steps so you can adopt them in staging and production.

These scripts assume a Bash environment on Linux (Ubuntu/CentOS) or macOS where noted. Always test on non-production systems first.

---

## Safety & testing best practices (must-read)

1. NEVER run unfamiliar scripts on production without review.
2. Use a separate test/staging environment; use ephemeral VMs or containers.
3. Add `set -Eeuo pipefail` and `IFS=$'\n\t'` to production scripts (robust defaults).
4. Use `shopt -s nullglob` when iterating globs to avoid literal patterns.
5. Avoid storing secrets in scripts or in plaintext in repos — use env vars, OS keyrings or secret managers.
6. Prefer idempotent operations; detect and short-circuit if no work needed.
7. Add logging and dry-run flags (`--dry-run`) to destructive scripts.
8. Use `flock` (or system lockfiles) to prevent concurrent runs.
9. Lint with `shellcheck` and test with `bats-core`.
10. Prefer explicit command paths or use `PATH` sanitation at top of script.

Recommended script header (paste at top of production scripts):

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'
# Optional: set -x for debugging (don't leave it enabled in production)
```

---

## How to use this repository

1. Clone repository:
   git clone https://github.com/ajaybabubojja/linux-admin-2026.git
2. Inspect scripts under `samples/` (or `scripts/`) — don't run blindly.
3. Run lint and tests locally:
   - Install shellcheck and bats-core
   - Run `shellcheck samples/*.sh`
   - Run `bats tests/`
4. Deploy scripts to `/usr/local/bin` or managed config location and create systemd timers or cron entries.

---

## Script catalog (high-level)

The original collection included 20 scripts. Below are the most useful ones with production-grade patterns and verification instructions:

1. Disk usage monitor & alert
2. CPU usage monitor
3. Memory usage alert
4. Service monitor & auto-restart
5. Log cleanup (old logs)
6. Application log archive
7. MySQL backup & retention
8. Web service health check
9. Website availability monitor (multi-site)
10. Bulk user creation (secure)
11. Kubernetes pod status monitor (kubectl)
12. Docker container auto-restart
13. Access-log parsing — top IPs
14. SSH failed login report
15. File integrity monitor (checksums)
16. Cron job monitoring
17. Deployment helper (git pull + restart)
18. Git repo auto-sync
19. Rsync backup to remote server
20. System audit summary

---

## Detailed Scripts — examples, verification & improvements

Below we provide production-grade versions of the most important scripts with verification steps and suggested enhancements.

---

### A. Disk Usage Monitor (production-ready)

Purpose: Alert when a filesystem exceeds threshold. Should be safe to test.

Production script: samples/disk_alert.sh

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

# Usage: disk_alert.sh <mount_point> <threshold_percent>
# Example: disk_alert.sh / 85
MOUNT=${1:-/}
THRESHOLD=${2:-80}
SIMULATE=${SIMULATE_USAGE:-}

get_usage() {
  if [ -n "$SIMULATE" ]; then
    echo "$SIMULATE"
    return
  fi
  df -P "$MOUNT" | awk 'NR==2 {gsub(/%/,""); print $5}'
}

main() {
  usage_pct=$(get_usage)
  if ! [[ "$usage_pct" =~ ^[0-9]+$ ]]; then
    echo "ERROR: could not parse usage: '$usage_pct'" >&2
    exit 2
  fi

  if [ "$usage_pct" -ge "$THRESHOLD" ]; then
    echo "ALERT: $MOUNT is ${usage_pct}% used (threshold ${THRESHOLD}%)"
    # Call notification hook - customize:
    # notify "Disk alert" "Usage ${usage_pct}% on ${HOSTNAME}"
  else
    echo "OK: $MOUNT ${usage_pct}% (threshold ${THRESHOLD}%)"
  fi
}

main "$@"
```

How to run (manual):
- Dry-run simulated high usage:
  SIMULATE_USAGE=90 bash samples/disk_alert.sh / 80

Real run:
  bash samples/disk_alert.sh / 90

Verification:
- For simulated case expect an "ALERT" line.
- For real run: check `df -h /` to verify filesystem usage matches output.

Improvements:
- Add `--notify` flag to send Slack/email.
- Add hysteresis (avoid alert flapping) — rely on monitoring platform or store last alert timestamp.
- Integrate with Prometheus exporter instead of ad-hoc scripts for long-term monitoring.

Test cases (bats):
- With SIMULATE_USAGE env var set to 85 returns "ALERT".
- With SIMULATE_USAGE 30 returns "OK".

---

### B. CPU Usage Monitor (robust)

Production script:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

THRESHOLD=${1:-85}
SIMULATE=${SIMULATE_CPU:-}

get_cpu_usage() {
  if [ -n "$SIMULATE" ]; then
    echo "$SIMULATE"
    return
  fi
  # using /proc/stat to compute total usage (portable)
  read -r cpu user nice system idle iowait irq softirq steal guest < /proc/stat
  prev_total=$((user+nice+system+idle+iowait+irq+softirq+steal))
  prev_idle=$((idle + iowait))
  sleep 1
  read -r cpu user nice system idle iowait irq softirq steal guest < /proc/stat
  total=$((user+nice+system+idle+iowait+irq+softirq+steal))
  idle_now=$((idle + iowait))
  total_diff=$((total - prev_total))
  idle_diff=$((idle_now - prev_idle))
  usage=$(( (1000 * (total_diff - idle_diff) / total_diff + 5) / 10 ))
  echo "$usage"
}

main() {
  cpu=$(get_cpu_usage)
  if [ "$cpu" -ge "$THRESHOLD" ]; then
    echo "ALERT: CPU usage ${cpu}% >= ${THRESHOLD}%"
    # notify ...
  else
    echo "OK: CPU usage ${cpu}%"
  fi
}

main
```

Verify:
- SIMULATE_CPU=95 bash samples/cpu_monitor.sh => prints ALERT
- Real check: run `top`/`mpstat` to compare results.

Notes:
- This method computes usage from /proc/stat and avoids `top`/`awk` differences across distros.

---

### C. Service monitor & auto-restart (systemd)

Production script:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

SERVICE="${1:-nginx}"
LOG="/var/log/service-monitor.log"
retries=3

log() { echo "$(date '+%F %T') $*" >> "$LOG"; }

if ! systemctl is-enabled --quiet "$SERVICE"; then
  log "Service $SERVICE not enabled."
fi

if ! systemctl is-active --quiet "$SERVICE"; then
  log "Service $SERVICE not running - attempting restart"
  for i in $(seq 1 $retries); do
    if systemctl restart "$SERVICE"; then
      log "Restart succeeded on attempt $i"
      exit 0
    else
      log "Restart attempt $i failed"
      sleep 2
    fi
  done
  log "Failed to restart $SERVICE after $retries attempts"
  # notify ...
  exit 1
else
  log "Service $SERVICE running"
fi
```

Verification:
- Stop service manually: `sudo systemctl stop nginx` then run script; log file should show restart attempts and success.
- In CI, mock `systemctl` via wrapper in PATH to simulate behavior.

Improvements:
- Use `systemd` service unit settings (Restart=on-failure) instead of external script when possible.

---

### D. MySQL Backup with rotation & verification

Production script:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

DB="${1:-mydb}"
DEST="${2:-/var/backups/mysql}"
KEEP_DAYS=${3:-7}
DB_USER="${DB_USER:-root}"
DB_PASS="${DB_PASS:-}"

mkdir -p "$DEST"
ts=$(date -u +%F-%H%M%SZ)
backup_file="$DEST/${DB}-${ts}.sql.gz"

# Export with secure credentials (prefer .my.cnf or secret manager)
mysqldump --single-transaction --quick --lock-tables=false -u"$DB_USER" -p"$DB_PASS" "$DB" | gzip > "$backup_file"
chmod 600 "$backup_file"
# verify
if gzip -t "$backup_file"; then
  echo "Backup OK: $backup_file"
else
  echo "Backup verification failed" >&2
  exit 1
fi

# rotate
find "$DEST" -type f -name "${DB}-*.sql.gz" -mtime +$KEEP_DAYS -print -delete
```

Verification:
- Restore test (on staging): gzcat file | mysql -u user -p password test_restore_db
- Check `gzip -t` returns success.
- CI: run backup to temp dir using a small test DB container, then restore into another test DB.

Security:
- Prefer credential files (`~/.my.cnf`) with 600 perms or use vault solutions.

---

### E. Web health check with Slack alert example

Script:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

URL="${1:-http://localhost}"
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"

status="$(curl -s -o /dev/null -w '%{http_code}' "$URL")"
if [ "$status" -ne 200 ]; then
  msg="ALERT: $URL returned $status on $(hostname) at $(date -Iseconds)"
  echo "$msg"
  if [ -n "$SLACK_WEBHOOK" ]; then
    curl -s -X POST -H 'Content-type: application/json' \
      --data "{\"text\":\"$msg\"}" "$SLACK_WEBHOOK" >/dev/null || true
  fi
fi
```

Verification:
- Run with local web server down and check Slack message (or echo if webhook not provided).
- For testing without Slack, set SLACK_WEBHOOK to a dummy and check `curl` behavior; or mock with a request bin.

Security:
- Store `SLACK_WEBHOOK` as secret in CI or env var in systemd unit; avoid committing.

---

## Notifications & integrations

Common integrations:

- Slack (incoming webhook): simple JSON POST with message.
- Email: sendmail or `mail` / `ssmtp` / `msmtp` (system dependent).
- PagerDuty / Opsgenie / Teams: use their webhooks or CLI.
- Prometheus: prefer to expose metrics (e.g., via node_exporter or custom exporter) and use Alertmanager for notifications.

Example Slack curl (as shown above):

```bash
curl -s -X POST -H 'Content-type: application/json' \
  --data "{\"text\":\"$msg\"}" "$SLACK_WEBHOOK"
```

Tip: URL-encode or JSON-escape message content to avoid JSON syntax problems.

---

## Scheduling: cron vs systemd timers

- Cron: traditional, widely available. Use `crontab -e` to schedule.
  - Example: run disk_alert every 5 minutes:
    ```cron
    */5 * * * * /usr/bin/env bash /opt/scripts/disk_alert.sh / 80 >> /var/log/disk_alert.log 2>&1
    ```
- systemd timers: more robust, easier to manage logs and unit dependencies.
  - Create `service` unit and `.timer` to run on schedule.
  - Example timer: runs every 5 minutes; logs to journalctl.

Prefer systemd timers on systemd-based systems.

---

## Logging and rotation

- Write logs to a dedicated file under `/var/log/<app>/`.
- Use `logger` to send to syslog: `logger -t myscript "message"`.
- Rotate logs using `logrotate` (create a config under `/etc/logrotate.d/`):

Example logrotate snippet:

```
/var/log/myscript/*.log {
    daily
    rotate 14
    compress
    missingok
    notifempty
    create 0640 root adm
    sharedscripts
    postrotate
      systemctl restart myscript.service >/dev/null 2>&1 || true
    endscript
}
```

---

## Secrets & credentials (do not hardcode)

Recommended approaches:
- Environment variables (set via systemd unit or CI secrets).
- `~/.my.cnf` with restricted permissions for MySQL credentials.
- Secret management solutions: Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager.
- Use IAM roles (AWS) for S3 access rather than embedding keys.

Never commit secrets to git. Add secret patterns to `.gitignore` and run pre-commit scanning (truffleHog, git-secrets).

---

## Unit testing & CI

- Unit tests: use `bats-core` (Bash Automated Testing System).
- Lint: `shellcheck` — fail CI on important warnings.
- CI example: GitHub Actions job steps:
  1. checkout
  2. install shellcheck, bats-core
  3. run `shellcheck samples/*.sh`
  4. run `bats tests/`

Example GitHub Actions (snippet):

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: install shellcheck
        run: sudo apt-get update && sudo apt-get install -y shellcheck
      - name: shellcheck
        run: shellcheck samples/*.sh
      - name: install bats
        run: |
          git clone --depth 1 https://github.com/bats-core/bats-core.git /tmp/bats
          sudo /tmp/bats/install.sh /usr/local
      - name: run tests
        run: bats tests
```

Testing tips:
- Use env var injection (SIMULATE_USAGE) to make scripts deterministic for tests.
- Use temporary directories via `mktemp -d` and cleanup in tests.
- Mock external commands (like `systemctl`) by providing a `PATH` with wrappers during tests.

---

## Deployment & packaging

- Place stable scripts in `/usr/local/bin` and make them executable (`chmod 755`).
- For configuration-driven scripts, store configs under `/etc/<app>/` and read them from script.
- Create a simple systemd unit for long-running or scheduled tasks.
- Use configuration management (Ansible/Chef/Puppet) to deploy scripts, set perms, and register timers.

Example systemd service unit (for periodic job managed by systemd timer):

`/etc/systemd/system/disk-alert.service`

```ini
[Unit]
Description=Disk alert job

[Service]
Type=oneshot
ExecStart=/usr/bin/env bash /opt/scripts/disk_alert.sh / 85
```

Timer unit: `/etc/systemd/system/disk-alert.timer`

```ini
[Unit]
Description=Run disk alert every 5 min

[Timer]
OnBootSec=1min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now disk-alert.timer
```

---

## Additional useful scripts you should add (missing from original collection)

1. Log rotation helper (wraps `logrotate` config generation)
2. Backup verification/restore script (test restores automatically)
3. Prometheus metrics exporter (simple textfile exporter to expose metrics)
4. System health dashboard aggregator (aggregate many monitors and produce a report)
5. S3 multipart upload helpers & integrity checkers
6. Salt/Ansible runner wrapper to trigger playbooks safely
7. Alert rate-limiter (avoid alert storms)
8. TLS certificate expiry monitor (check cert days left)
9. IP whitelisting script (update firewall rules from list)
10. Container image CVE scanner wrapper (e.g., `trivy` wrapper)

---

## Troubleshooting & FAQ

Q: Script fails with cryptic errors under cron but works interactively.
- Answer: Cron uses a minimal environment (PATH, HOME). Set full PATH in the script or source a known environment file. Use absolute paths to commands.

Q: My script exits unexpectedly with `set -e`.
- Answer: `set -e` exits on first non-zero status; commands inside `if` or within pipelines behave differently. Use explicit checks or `|| true` where appropriate.

Q: shellcheck shows SC2086 ("Double quote to prevent globbing and word splitting")
- Answer: Fix by quoting expansions, or intentionally leave unquoted when globbing is required and safe.

Q: How do I avoid concurrent runs?
- Answer: Use `flock` or create PID/lock files stored under `/var/lock`.

---

## Example: Full workflow to onboard a script into production

1. Develop script in `samples/` with `set -Eeuo pipefail`.
2. Add `--dry-run` and `--verbose` flags; add help text.
3. Add `bats` tests and mock env var hooks.
4. Run `shellcheck` and fix issues.
5. Add logging to `/var/log/<app>` and configure `logrotate`.
6. Create systemd service or timer to run it.
7. Deploy via Ansible to target hosts; set secrets via vault.
8. Add monitoring (Prometheus exporter or push alerts to Slack).
9. Schedule a restore/verification job monthly to validate backups.

---

## Quick checklist before production rollout

- [ ] Script has `set -Eeuo pipefail` (or equivalent defensive checks)
- [ ] Input validated; no unsafe `eval` usage
- [ ] Credentials are not hard-coded
- [ ] Idempotency or safe to run multiple times
- [ ] Logging exists and logrotate configured
- [ ] Unit & integration tests included
- [ ] CI runs shellcheck and bats
- [ ] Notifications configured (Slack/email/webhook)
- [ ] Locking exists to avoid concurrent runs (`flock`)
- [ ] Systemd timer or cron configured with appropriate user/permissions
- [ ] Restore verification (for backup scripts) implemented

---

## References & further reading

- Bash Reference Manual — https://www.gnu.org/software/bash/manual/
- Advanced Bash-Scripting Guide — https://tldp.org/LDP/abs/html/
- ShellCheck — https://www.shellcheck.net/
- bats-core — https://bats-core.readthedocs.io/
- systemd Timers — https://www.freedesktop.org/software/systemd/man/systemd.timer.html
- logrotate — https://linux.die.net/man/8/logrotate

---
