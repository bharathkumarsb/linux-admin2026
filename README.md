# linux-admin2026 — Practical Linux Administration & DevOps Runbooks

Comprehensive, hands‑on collection of Linux administration notes, command references, runbooks, troubleshooting guides, and shell scripting resources — curated for Linux admins, DevOps engineers, SREs, and cloud engineers.

Last updated: 2025-12-28

---

Table of contents
- About this repository
- Quick start
- Repository structure (file index & short descriptions)
- How to use these notes (learning, ops, interviews)
- Recommended workflows (runbook patterns, incident steps)
- Generating printable / PDF output
- Contributing guidelines
- Automation & maintenance suggestions
- License & contact

---

About this repository
This repository is a single-place knowledge base for day-to-day Linux administration and DevOps tasks. It contains cheat-sheets, step-by-step runbooks, troubleshooting matrices, interview Q&A, and production-ready shell scripts. Use it to onboard quickly, prepare for incidents, and study for interviews.

Quick start
1. Clone the repo:
   git clone https://github.com/bharathkumarsb/linux-admin2026.git
2. Open the most relevant file for your task, e.g.:
   - Basic Linux commands: [linux-basic-commands.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-basic-commands.md)
   - Networking notes: [linux-networking-notes.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-networking-notes.md)
   - Troubleshooting & runbooks: [linux-service-failures-systemctl.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-service-failures-systemctl.md)
3. Search quickly with ripgrep:
   rg 'ssh' -n

---

Repository structure (file index & short descriptions)
Below is the canonical file index with concise descriptions — click each filename to open on GitHub.

- [README.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/README.md) — this file (overview & navigation).
- [linux-basic-commands.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-basic-commands.md) — essential day‑to‑day Linux commands, quick examples and usage patterns.
- [linux-advance-commands.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-advance-commands.md) — advanced iproute2, tc, systemd-networkd, nsenter, strace examples and deep dives.
- [linux-networking-notes.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-networking-notes.md) — practical networking concepts (IP, NAT, DNS, ARP), commands and troubleshooting steps.
- [network-basics.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/network-basics.md) — beginner-friendly network fundamentals and foundational theory.
- [network-protocols-ports.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/network-protocols-ports.md) — protocol/port cheat-sheets (SSH, HTTP, DB ports, Kubernetes ports, etc.).
- [realtime-devops-shellscripts.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/realtime-devops-shellscripts.md) — collection of production-ready scripts for real-time tasks and alerts.
- [devops-scripts-email-alerts.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/devops-scripts-email-alerts.md) — email alert scripts, SMTP examples and operational tips.
- [shell-scripting-basics.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/shell-scripting-basics.md) — beginner shell scripting patterns and idioms.
- [shell-scripting-basics-practical.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/shell-scripting-basics-practical.md) — practical scripting exercises and examples.
- [shell-scripting-advanced.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/shell-scripting-advanced.md) — advanced Bash features, safety, idempotence, testing.
- [shell-scripting-exercises-1.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/shell-scripting-exercises-1.md) — hands-on exercises to practice scripting.
- [realtime-devops-shellscripts.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/realtime-devops-shellscripts.md) — scripts for monitoring, alerting and automating fixes.
- [linux-service-failures-systemctl.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-service-failures-systemctl.md) — runbook for investigating and recovering systemd services.
- [linux-nfs-issues.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-nfs-issues.md) — NFS mounting, troubleshooting, performance and locking issues.
- [linux-kernel-panic-issues.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-kernel-panic-issues.md) — diagnosing kernel panics, kernel oops, crash dumps, and kexec workflows.
- [lvm-troubleshooting.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/lvm-troubleshooting.md) — LVM management and common recovery patterns for PV/VG/LV.
- [log-rotation.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/log-rotation.md) — best practices for logrotate, systemd journald rotation and retention.
- [linux-interview-qustns.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-interview-qustns.md) — curated interview questions and model answers.
- [linux-q-n-a.md](https://github.com/bharathkumarsb/linux-admin2026/blob/main/linux-q-n-a.md) — Q&A knowledge dump for quick review.

---

How to use these notes
Choose an objective and follow the suggested paths below.

1. Learn Linux basics (newcomer)
   - Read: [linux-basic-commands.md], then [shell-scripting-basics.md].
   - Practice: tutorial exercises in [shell-scripting-exercises-1.md].

2. Prepare for interviews (engineer)
   - Read: [linux-interview-qustns.md] and [linux-q-n-a.md]
   - Memorize: quick ports & commands in [network-protocols-ports.md] and [linux-basic-commands.md].

3. Daily operations & runbooks (on-call SRE)
   - Keep: [linux-service-failures-systemctl.md], [linux-networking-notes.md], [lvm-troubleshooting.md] in an accessible location (bastion, wiki).
   - Use: [realtime-devops-shellscripts.md] & [devops-scripts-email-alerts.md] for common automation.

4. Deep troubleshooting (senior engineer)
   - Reference: [linux-advance-commands.md], [linux-kernel-panic-issues.md], [linux-nfs-issues.md], [log-rotation.md].

Recommended runbook pattern (for each incident)
- Triage: What is failing, blast radius, who is affected.
- Quick tests: ping, ss/sshd, journalctl, docker/k8s status.
- Capture evidence: tcpdump, strace, system logs, /proc, vmstat/iostat.
- Contain & mitigate: restart service, scale up, failover.
- Root cause analysis: reproduce, trace, fix, postmortem.

Examples of one‑line diagnostics
- Show listening sockets: ss -tulnp
- Find high CPU users: ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
- Capture packets for port 22: sudo tcpdump -i eth0 port 22 -w /tmp/ssh.pcap

---

Generating printable / PDF output
You can render any of these Markdown files to PDF or HTML for printing.

- Using pandoc:
  pandoc linux-basic-commands.md -o linux-basic-commands.pdf --pdf-engine=wkhtmltopdf

- Using GitHub web UI:
  Open the file in the repository and use the browser Print → Save as PDF.

- For site generation:
  - Use MkDocs or Hugo to generate an internal docs site and publish to GitHub Pages.

Suggested one‑page cheat sheet
- Extract top items from:
  - [linux-basic-commands.md]
  - [network-protocols-ports.md]
  - [linux-interview-qustns.md]
- Use a small template (A4 / Letter) and export as PDF.

---

Contributing guidelines
We welcome improvements — documentation, scripts, corrections, or new runbooks.

- Fork the repo and create a feature branch.
- Add or update a Markdown file with clear examples and test commands.
- If adding firewall or production scripts, accompany them with a tested dry-run and README section showing safe usage.
- Open a PR with:
  - Summary of changes.
  - Files changed.
  - Test instructions to validate the change (commands).
  - Tag with labels: docs, runbook, script.

Commit message convention (recommended)
- feat(docs): add NFS locking notes
- fix(script): handle missing dependency in alert script
- chore(ci): add mkdocs build

Maintainers will review for clarity, safety, and reproducibility.

---

Automation & maintenance suggestions
- Keep a canonical YAML or JSON dataset for any repeated table data (ports, services, runbook metadata). Example fields:
  service, ports, proto, config_path, test_cmd, security_note, owner, last_reviewed
- Add a CI job to:
  - Validate Markdown linting (markdownlint).
  - Render a test HTML site and ensure no broken links.
  - Run shellcheck on scripts in `realtime-devops-shellscripts.md`.
- Consider moving scripts into a `scripts/` directory and adding small unit tests.

---

Security note (important)
- Treat scripts that perform changes as privileged: test on non-production instances first.
- Do not store secrets (passwords, private keys, tokens) in this repo. Use vault/secret manager and reference secure retrieval in scripts.
- Review and sanitize any example that exposes private endpoints or credentials.

---

License & contact
- If this repository does not include a LICENSE file, please add one to clarify reuse (MIT is common for learning repos).
- Author / Maintainer: Bharath Kumar S B (see repo owner)
- For questions or contribution requests, open an Issue in this repository.

---

