# 🔁 Log Rotation — Production Guide for DevOps Scripts

> A complete, production-ready guide to adding log rotation to the DevOps scripts in this repo.  
> Covers sample logs, multiple `logrotate` configurations (time- and size-based), compression, retention, testing, S3 archival, SELinux & permissions, systemd timers, troubleshooting and verification steps — everything you need to make logging safe and manageable in production.

---

## Why log rotation matters

Alert and monitoring scripts write logs continuously. Without rotation logs grow indefinitely → disk fills → system outages.

Log rotation:
- prevents full disks
- compresses/archives older logs
- enforces retention policies
- enables downstream archival / analytics

This guide shows robust patterns you can adopt quickly and safely.

---

## Quick overview (what you'll get)

- Recommended layout and naming for script logs
- Example `logrotate` configurations for common cases
- How to handle long-running processes that keep file descriptors open
- How to test rotations safely (dry-run / forced)
- Compression, retention, and remote archival (S3) example
- Security & permissions (SELinux, mode, su)
- systemd timer + cron examples to run logrotate
- Troubleshooting checklist and diagnostics

---

## Recommended log file layout

Create a dedicated, central folder for DevOps logs:

- /var/log/devops/                 # central log directory
  - service-health.log
  - disk-usage.log
  - web-monitor.log
  - docker-monitor.log
  - k8s-monitor.log

Use consistent log prefixes and avoid embedding secrets in logs.

Example in a script:

```bash
#!/usr/bin/env bash
LOGDIR=/var/log/devops
mkdir -p "$LOGDIR"
LOGFILE="$LOGDIR/service-health.log"
echo "$(date -Iseconds) - Service restarted: nginx" >> "$LOGFILE"
```

Ensure the directory and files have appropriate ownership and permissions:

```bash
sudo mkdir -p /var/log/devops
sudo chown root:adm /var/log/devops
sudo chmod 0750 /var/log/devops
sudo touch /var/log/devops/service-health.log
sudo chown root:adm /var/log/devops/service-health.log
sudo chmod 0640 /var/log/devops/service-health.log
```

---

## Example `logrotate` configurations

Create a single rotate file for all devops logs:

File: `/etc/logrotate.d/devops-monitoring`

```
/var/log/devops/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root adm
    dateext
    dateformat -%Y-%m-%d
    sharedscripts
    postrotate
        /usr/bin/systemctl reload rsyslog >/dev/null 2>&1 || true
    endscript
}
```

Explanation of important directives:
- `daily` — rotate each day
- `rotate 14` — keep 14 archived logs
- `compress` — gzip old logs
- `delaycompress` — skip compression on the first rotation so processes can finish writing to same file
- `notifempty` — skip rotation if file empty
- `missingok` — do not error if file missing
- `create <mode> <owner> <group>` — create new log file with permissions
- `dateext` + `dateformat` — append date instead of numeric count
- `sharedscripts` — run `postrotate` once per block (not per file)
- `postrotate` — reload rsyslog to make sure it reopens files (or restart services when necessary)

---

## Size-based rotation (for very chatty logs)

If logs are high-throughput, rotate by size (or combined):

```
/var/log/devops/high-traffic.log {
    size 50M
    rotate 10
    compress
    missingok
    notifempty
    create 0640 root adm
    postrotate
        /usr/bin/systemctl reload rsyslog >/dev/null 2>&1 || true
    endscript
}
```

Directives:
- `size 50M` — rotate when file grows beyond 50 MB
- Alternative: `maxsize` rotates only if file exceeds specified size even on scheduled rotation

---

## Handling daemons / long-running processes

When programs keep files open, you have two main strategies:

1. Preferred: Move/rename log and instruct daemon to reopen logs (recommended)
   - Use `postrotate` to `systemctl reload` or send `SIGHUP` so the daemon reopens file descriptors.
   - Example (rsyslog): `systemctl reload rsyslog` or `kill -HUP <pid>`

2. If the program cannot be signaled, use `copytruncate` (not ideal)
   - `copytruncate` copies current file to rotated file then truncates original file so process keeps writing to the same inode (risk of data loss between copy & truncate).
   - Example:

```
/var/log/devops/legacy-app.log {
    daily
    rotate 7
    copytruncate
    compress
    missingok
    notifempty
    create 0640 root adm
}
```

Use copytruncate only when you cannot restart or signal the process.

---

## Per-service rotation with proper user context (`su`)

Some logs are written by non-root processes (e.g., mysql). Rotate them as the owning user:

```
/var/log/mysql/*.log {
    daily
    rotate 14
    compress
    missingok
    notifempty
    create 0640 mysql adm
    su mysql adm
    postrotate
        /usr/bin/systemctl reload mysql >/dev/null 2>&1 || true
    endscript
}
```

`su user group` tells logrotate to run prerotate/postrotate scripts as that user.

---

## Compression modes and alternatives

- Default: `compress` uses gzip (`.gz`) and `compresscmd`/`uncompresscmd` can be overridden.
- Use `compresscmd /usr/bin/xz` + `compressext .xz` if XZ is desired for higher compression (slower).
- Use `delaycompress` if you want last rotation uncompressed (sometimes helpful for quick access).
- Example to switch to xz:

```
compress
compresscmd /usr/bin/xz
compressext .xz
```

---

## Archive rotated logs to S3 (example)

If you want off-host retention, upload rotated files to S3. Use IAM roles (recommended) and `aws-cli`.

Script: `/opt/scripts/archive_rotated_to_s3.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
LOGDIR=/var/log/devops
S3_BUCKET=s3://my-backup-bucket/devops-logs
TMPDIR=$(mktemp -d)
trap 'rm -rf "$TMPDIR"' EXIT

for f in "$LOGDIR"/*.log-*.gz; do
  [ -e "$f" ] || continue
  echo "Uploading $f"
  aws s3 cp "$f" "$S3_BUCKET/" --storage-class STANDARD_IA
  if [ $? -eq 0 ]; then
    echo "Uploaded $f"
    # Optionally delete local rotated file after upload:
    # rm -f "$f"
  else
    echo "Upload failed for $f" >&2
  fi
done
```

Verification:
- Use `aws s3 ls s3://my-backup-bucket/devops-logs/` to check uploaded objects.
- Use `aws s3api head-object --bucket my-backup-bucket --key devops-logs/<filename>` to verify metadata.

Notes:
- Use instance profile/role to avoid storing AWS keys on disk.
- Consider lifecycle rules on S3 to move old logs to Glacier or delete after retention period.

---

## Forcing and testing rotation

Dry-run (no changes):

```bash
sudo logrotate -d /etc/logrotate.d/devops-monitoring
```

Verbose dry-run:

```bash
sudo logrotate -d -v /etc/logrotate.d/devops-monitoring
```

Force rotation now:

```bash
sudo logrotate -f /etc/logrotate.d/devops-monitoring
```

Use a separate state file to test without affecting system state:

```bash
sudo logrotate -s /tmp/logrotate.test.state -f /etc/logrotate.d/devops-monitoring
```

Check rotated files:

```bash
ls -lh /var/log/devops/
# sample output:
# -rw-r----- 1 root adm 1.2K service-health.log
# -rw-r----- 1 root adm  45K service-health.log-2025-12-27.gz
```

Check that new log file exists and new entries append correctly:

```bash
echo "test entry" >> /var/log/devops/service-health.log
tail -n 5 /var/log/devops/service-health.log
```

For processes that need reopen:
- Verify that file descriptor changed: after rotate, check `lsof -p <pid> | grep service-health.log` or compare inode numbers (`stat`).

---

## Scheduling logrotate

Most systems run `logrotate` via cron.daily or systemd timer:

- Cron (classic): `/etc/cron.daily/logrotate` runs daily.
- Systemd timer variant: create `logrotate.service` + `logrotate.timer`.

Example systemd timer to run every hour:

`/etc/systemd/system/logrotate-hourly.service`:

```ini
[Unit]
Description=Run logrotate hourly

[Service]
Type=oneshot
ExecStart=/usr/sbin/logrotate /etc/logrotate.conf
```

`/etc/systemd/system/logrotate-hourly.timer`:

```ini
[Unit]
Description=Hourly logrotate for busy systems

[Timer]
OnBootSec=1min
OnUnitActiveSec=1h

[Install]
WantedBy=timers.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now logrotate-hourly.timer
```

Use hourly timer only for very busy log systems — daily is sufficient for most.

---

## SELinux & AppArmor considerations

- On SELinux-enabled systems, rotated files may have different file contexts. Use `restorecon` in `postrotate` to restore context:

```
postrotate
    /usr/sbin/restorecon -R /var/log/devops >/dev/null 2>&1 || true
endscript
```

- If logrotate cannot rename or create files due to SELinux AVC denials, check `ausearch -m avc` or `journalctl -k` for denials and add appropriate policy or adjust contexts.

---

## Emailing rotated logs (optional)

You may want to email rotated logs (first rotation or summary):

`logrotate` supports `mail` and `mailfirst`/`maillast`:

```
/var/log/devops/*.log {
    daily
    rotate 7
    compress
    mail admin@example.com
    mailfirst
    missingok
    notifempty
    create 0640 root adm
}
```

Caveats:
- Mail servers vary; prefer sending notifications, not full files. For large logs, upload to S3 and send link.

---

## Troubleshooting & diagnostics

Symptom: rotation doesn't happen
- Check `/var/log/messages`, `/var/log/syslog`, `/var/log/cron` for errors
- Run `logrotate -d <config>` to debug
- Ensure `/etc/cron.daily/logrotate` exists and cron is running (or systemd timer enabled)

Symptom: `logrotate` fails on `postrotate` script
- Check scripts for `exit` causing rotation to stop
- Ensure `postrotate` uses full paths and proper permissions (runs as root unless `su` used)
- Add `|| true` where failure should not stop rotation

Symptom: New log file has wrong owner/permissions
- Check `create <mode> owner group` in config
- If rotated as different user, use `su` directive

Symptom: Rotated files not compressed
- Confirm `compress` is present; check availability of `gzip` (`which gzip`)
- Check `delaycompress` behavior; `dateext` names may be present uncompressed on first rotation if `delaycompress` is used

Symptom: Files still grow after rotation (process writes to old inode)
- Either use `copytruncate` or signal the process to reopen logs (preferred)
- Verify process has reopened files using `lsof` or comparing inode numbers (`stat -c %i <file>`)

---

## Audit & monitoring for logging system health

Monitor:
- Free disk space on `/var` and `/var/log` (use the disk alert script)
- Age of rotated files: `find /var/log/devops -type f -name "*.gz" -mtime +14 -print`
- Last run of logrotate: check `/var/lib/logrotate/status` or logs in `/var/log/syslog` for run times
- Email queue (if logs emailed)

Automated health check example:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Check /var/log mount usage and the rotated archive count
DF=$(df -h /var | awk 'NR==2 {print $5}')
COUNT=$(ls /var/log/devops/*.gz 2>/dev/null | wc -l || true)
echo "var usage: $DF, archives: $COUNT"
```

---

## Examples you may want to add (missing / recommended)

- `rotate-and-archive.sh` — runs logrotate, finds newly-rotated files and pushes to S3, and verifies uploads
- `rotate-prune-s3.sh` — prunes S3 logs older than N days using lifecycle or script
- `rotate-notify.sh` — sends a summary message to Slack/email after each rotation run (count rotated files)
- `log-shipper.service` — systemd service to run fluentd/rsyslog forwarder for central logging
- `log-verify-restore.sh` — daily/weekly verification that archived logs can be decompressed and parsed
- `logrotate-validate.sh` — CI script that validates `logrotate` configuration via `logrotate -d` and `shellcheck` for rotation helper scripts

---

## Security & best practices summary

- Do not log secrets (passwords, tokens).
- Restrict log directory permissions (owner root and group `adm` or a dedicated group).
- Use `su` in `logrotate` for logs generated by non-root daemons.
- Use instance roles / key management for S3 upload — never hardcode keys.
- Use `logrotate` `postrotate` to restore SELinux context where required.
- Keep the `logrotate` config under source control and validate via CI.

---

## Practical end-to-end example (rotate → s3 → verify)

1. `logrotate` rotates logs daily into `/var/log/devops/*.log-YYYY-MM-DD.gz`.
2. A cron or systemd timer runs `/opt/scripts/archive_rotated_to_s3.sh` that uploads new rotated files.
3. The script writes a local state file of uploaded object keys (or relies on `aws s3api head-object`).
4. S3 lifecycle deletes objects older than 90 days and moves older archives to Glacier.

This pattern ensures local disk stays bounded while retaining long-term logs remotely.

---

## Final checklist before enabling rotation in production

- [ ] Create `/var/log/devops` and set ownership/permissions
- [ ] Add `/etc/logrotate.d/devops-monitoring` (as above)
- [ ] Test with `logrotate -d` and `logrotate -f` using a test state file
- [ ] Ensure `postrotate` uses full paths and does not fail (add `|| true` where appropriate)
- [ ] Configure systemd timer or cron job as needed
- [ ] Configure S3 archival if required (and IAM role)
- [ ] Add `bats` tests that simulate rotation (mock `notify()` and services)
- [ ] Run SELinux `restorecon` in `postrotate` if needed

---
