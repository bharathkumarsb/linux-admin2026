# Networking Reference — Practical, Detailed, Effective Design
A comprehensive, structured README.md intended for Linux Administrators, DevOps Engineers, and Cloud Engineers.  
This document combines concise cheat-sheets with drill‑down operational runbooks, tables designed for readability, reproducible config snippets, testing commands, hardening guidance, troubleshooting matrices, and automation tips.

Why this document
- Table-first: quick lookup + detailed follow-up
- Actionable: copy/paste commands and config
- Audit-ready: owner/expiry metadata for firewall rules
- Automation-friendly: includes YAML schema for generation

How to use
- Use the top “Quick cheat-sheet” for interviews and fast lookups.
- Use “Detailed reference” tables for ops and onboarding.
- Use “Troubleshooting Matrix” in incident runbooks.
- Keep the YAML dataset (Appendix) as the canonical source to autogenerate docs and security groups.

Table of contents
- Design principles
- Quick cheat-sheet (must memorize)
- Detailed reference table (operational)
- Troubleshooting matrix (decision tree)
- Firewall rule design table (auditable)
- Kubernetes ports table (cluster mapping)
- Full step-by-step examples (SSH, DNS, HTTP, MySQL, kube-apiserver)
- WireGuard quick example
- Master YAML dataset (sample)
- Automation & CI suggestions
- Formatting, UX & publish tips
- Export & printing suggestions
- Final checklist before publishing
- Appendix: useful commands & references

---

## Design principles (brief)
- Columns answer user questions: What? Ports? Proto? Where configured? How to test? How to secure?
- Two-tier approach:
  - Tier 1: Single-page cheat-sheet (fast recall).
  - Tier 2: Detailed reference (config → test → secure → troubleshoot).
- Keep inline code for one-line commands; long snippets as code blocks below the row.
- Add metadata (Owner, Last Reviewed, Expiry) to firewall rules or exception rows.
- Store master data in YAML/CSV to avoid doc drift.

---

## Quick cheat-sheet (must memorize)
A compact, single-line table for interviews and quick lookup.

| Service | Port(s) | Proto | Purpose |
|--------:|:-------:|:-----:|:--------|
| SSH | 22 | TCP | Secure remote shell, scp, sftp |
| DNS | 53 | UDP/TCP | Name resolution (UDP queries, TCP zone transfer) |
| HTTP | 80 | TCP | Unencrypted web traffic |
| HTTPS | 443 | TCP | Encrypted web traffic (TLS) |
| NTP | 123 | UDP | Time sync |
| SMTP | 25 / 465 / 587 | TCP | Email delivery / submission |
| MySQL | 3306 | TCP | Relational DB |
| PostgreSQL | 5432 | TCP | Relational DB |
| Redis | 6379 | TCP | In-memory cache |
| MongoDB | 27017 | TCP | NoSQL DB |
| Kubernetes API | 6443 | TCP | K8s control plane |
| etcd | 2379–2380 | TCP | K8s key-value store |
| Prometheus | 9090 | TCP | Monitoring server |
| Grafana | 3000 | TCP | Dashboards |
| NodePort | 30000–32767 | TCP | K8s node-exposed services |

---

## Detailed reference table (operational)
An actionable table for day-to-day operations. For each service: ports, short description, config pointer, test commands, security notes, cloud mapping.

| Service | Port(s) | Proto | Short description | Linux config example | Test command(s) | Security notes | Cloud mapping |
|---|---:|:---:|---|---|---|---|---|
| SSH | 22 | TCP | Remote admin & automation (Ansible, Git over SSH) | /etc/ssh/sshd_config: `PermitRootLogin no`, `PasswordAuthentication no` | `ssh -i ~/.ssh/id_rsa user@host` `ss -tln | grep :22` | Use key auth, bastion hosts, restrict SG to admin CIDR, enable MFA | AWS SG: allow 22 from bastion CIDR only |
| DNS (BIND/CoreDNS) | 53 | UDP/TCP | Name resolution; CoreDNS used inside Kubernetes | BIND: /etc/named.conf ; CoreDNS: `/etc/coredns/Corefile` | `dig +short example.com @8.8.8.8` `dig +trace example.com` | Restrict AXFR, enable DNSSEC if needed, monitor queries | Route53 / Cloud DNS for cloud-managed DNS |
| HTTP | 80 | TCP | Unencrypted web app traffic | Nginx: /etc/nginx/nginx.conf | `curl -I http://example.com` | Redirect to HTTPS, minimize info leaks | ALB/NLB healthchecks to 80 |
| HTTPS | 443 | TCP | TLS encrypted web traffic | Nginx config + certs in /etc/letsencrypt | `curl -vk https://example.com` `openssl s_client -connect example.com:443 -servername example.com` | Use TLS 1.2/1.3, strong ciphers, HSTS, renew certs automatically | ALB/Cloud LB termination |
| DHCP | 67/68 | UDP | IP assignment | isc-dhcp-server: /etc/dhcp/dhcpd.conf | `sudo dhclient -v eth0` | Limit DHCP scope; avoid rogue DHCP servers | Cloud VPC provides DHCP behavior |
| MySQL/MariaDB | 3306 | TCP | RDBMS | /etc/mysql/my.cnf `bind-address` | `mysql -h host -P 3306 -u user -p` `ss -tln | grep 3306` | Bind to private interface, use TLS, restrict SG | RDS private subnets |
| PostgreSQL | 5432 | TCP | RDBMS | /var/lib/pgsql/data/pg_hba.conf, postgresql.conf | `psql -h host -p 5432 -U user -d db` | Use TLS, configure pg_hba to restrict hosts | RDS/Aurora private subnets |
| Redis | 6379 | TCP | In-memory datastore | /etc/redis/redis.conf `bind` | `redis-cli -h host -p 6379 ping` | Require AUTH, bind to localhost/private subnet; do not expose | ElastiCache in-accessible public |
| MongoDB | 27017 | TCP | NoSQL DB | /etc/mongod.conf `bindIp` | `mongo --host host --port 27017` | Enable auth, TLS, IP binding | Atlas or private VPC |
| RabbitMQ | 5672 / 15672 | TCP | AMQP broker / management | /etc/rabbitmq/rabbitmq.conf | `ss -tln | grep 5672` | TLS, user auth, vhost restrictions | Private subnets |
| Kafka | 9092 | TCP | Event streaming | server.properties (listeners) | `nc -vz broker 9092` | TLS/SASL for production, restrict access | MSK or private brokers |
| Kubernetes API | 6443 | TCP | Control plane endpoint | /etc/kubernetes/manifests/kube-apiserver.yaml | `kubectl get nodes --kubeconfig admin.conf` `curl -k https://<api>:6443/healthz` | RBAC, OIDC, API audit logs, restrict LB/SG | EKS/AKS/GKE control plane endpoint |
| kubelet | 10250 | TCP | Node agent API | kubelet config at /var/lib/kubelet | `ss -tln | grep 10250` | Limit access to control plane, use client cert auth | Nodes only |
| etcd | 2379–2380 | TCP | Key-value store for control plane | /etc/etcd/etcd.conf | `ETCDCTL_API=3 etcdctl endpoint health --endpoints=<etcd>` | TLS only, auth, only accessible from control plane | Managed etcd in cloud services |

Notes:
- "Linux config example" is a pointer, not exhaustive. Full snippets are below in examples.
- "Cloud mapping" shows likely cloud constructs; adapt to your provider.

---

## Troubleshooting Matrix (decision tree)
Use this matrix at incident start — match symptom to probable layer and run the checks.

| Symptom | Probable Layer | Quick checks | Commands | Likely fix |
|--------|---------------|-------------|---------|-----------|
| Cannot SSH to host | L1-L4 (network/firewall) | Reachability? Port open? Route exists? | `ping host` `ss -tln | grep :22` `nc -vz host 22` `sudo ip route` | Fix SG/NACL, add firewall rule, correct route |
| DNS resolution fails | L7 / recursive | Can resolve via public resolver? Authoritative? | `dig @8.8.8.8 example.com` `dig +trace example.com` | Fix resolver settings, fix NS delegation, check firewall blocking 53 |
| HTTP 502/503 | App or proxy (L7) | Is backend healthy? Proxy logs? | `curl -v http://backend:8080/health` `tail -n 100 /var/log/nginx/error.log` | Restart app, check resource exhaustion, fix upstream binding |
| App cannot reach DB | L3/L4 or auth | Can connect to DB port? DB listening? Credentials? | `nc -vz db 5432` `ss -tln | grep 5432` `tail -n 200 /var/log/postgresql/postgresql.log` | Add SG rule, change bind-address, fix pg_hba.conf, update credentials |
| Intermittent latency/packet loss | L1-L3 | Interface errors? Duplex mismatch? CPU/IO? | `ip -s link` `ethtool eth0` `mtr -r host` | Fix duplex/speed, replace cable, tune MTU, investigate noisy neighbors |

How to use:
1. Reproduce.
2. Run checks in the order given.
3. Capture evidence (tcpdump, logs).
4. Escalate with evidence and recommended fixes.

---

## Firewall rule design table (audit-ready)
Document rules with owner and expiry so exceptions do not linger.

| Rule ID | Source CIDR | Dest IP/Network | Proto | Port(s) | Action | Justification | Owner | Last Reviewed | Expiry |
|--------:|:-----------:|:---------------:|:-----:|:-------:|:------:|:-------------:|:-----:|:-------------:|:------:|
| FW-1001 | 203.0.113.0/24 | 10.0.5.10/32 | TCP | 22 | Allow | Admin access from office | ops-team | 2025-12-01 | 2026-01-01 |
| FW-2004 | 0.0.0.0/0 | 10.0.1.0/24 | TCP | 443 | Allow | Public web LB | infra | 2025-11-10 | - |

Best practices:
- Automate expiration reminders.
- Tie each change to a ticket/approval.
- Keep justification concise: "Why is this needed?" not "What is it."

---

## Kubernetes ports table (component mapping)
Kubernetes clusters require specific ports for control plane and node communication. Use this to determine NACL/Security Group rules.

| Component | Port(s) | Proto | Direction | Purpose | Recommended access control |
|----------|:--------:|:-----:|:---------:|:-------|:--------------------------:|
| kube-apiserver | 6443 | TCP | Inbound | API for kubectl/controllers | Restrict to control plane & CI/CD IPs; use LB with mTLS |
| etcd | 2379–2380 | TCP | Inbound | etcd client + peer | Control plane nodes only; TLS + auth |
| kubelet | 10250 | TCP | Inbound | kubelet API (metrics, logs) | Only control plane and admin IPs |
| kube-scheduler | 10251 | TCP | Local | scheduler internal | localhost only on most setups |
| kube-controller-manager | 10252 | TCP | Local | controller internal | localhost only on most setups |
| NodePort range | 30000–32767 | TCP | Inbound | exposes service on node IP | Avoid unless necessary; use LB instead |

---

## Full step-by-step examples
Below are stepwise guides for configuring, testing, securing, and troubleshooting common services. Use these as runbook entries.

### A. SSH — runbook
1. Configure
   - Edit `/etc/ssh/sshd_config`:
     ```
     PermitRootLogin no
     PasswordAuthentication no
     PubkeyAuthentication yes
     AllowUsers alice bob
     MaxAuthTries 3
     ```
   - Restart:
     ```bash
     sudo systemctl restart sshd
     ```
2. Test
   - From client:
     ```bash
     ssh -i ~/.ssh/id_rsa alice@server.example.com
     ss -tln | grep :22
     sudo journalctl -u sshd -n 200
     ```
3. Secure
   - Place host in private subnet, allow SSH only from bastion SG or office CIDR.
   - Implement fail2ban:
     ```
     # /etc/fail2ban/jail.d/ssh.local
     [sshd]
     enabled = true
     maxretry = 5
     bantime = 3600
     ```
   - Consider certificate-based auth (short-lived certs) or MFA.
4. Troubleshoot
   - If connection refused: check `ss -tln`, `iptables -L`, `aws ec2 describe-security-groups`.
   - Packet capture: `sudo tcpdump -n -i eth0 port 22 -w /tmp/ssh.pcap`.

Interview tip: explain the role of bastion hosts, key rotation, and ephemeral certs vs static keys.

---

### B. DNS (BIND9) — runbook
1. Configure (zone example)
   - /etc/named.conf:
     ```
     options { listen-on port 53 { 127.0.0.1; 10.0.0.10; }; allow-query { any; }; };
     zone "example.com" IN {
       type master;
       file "zones/db.example.com";
       allow-transfer { 192.0.2.10; };   # restrict zone transfer
     };
     ```
   - Zone file: `/var/named/zones/db.example.com`
2. Test
   - Local:
     ```bash
     dig @localhost example.com A +short
     dig @8.8.8.8 example.com
     dig +trace example.com
     ```
3. Secure
   - Restrict AXFR to secondaries.
   - Implement Response Rate Limiting (RRL) to reduce reflection/amplification risk.
   - Consider DNSSEC for critical domains.
4. Troubleshoot
   - UDP blocked: `sudo tcpdump -n -i eth0 port 53`
   - Delegation errors: check parent TLD NS entries and glue records.

Interview tip: be prepared to explain difference between authoritative and recursive resolvers and why DNS uses TCP sometimes.

---

### C. HTTP(S) — runbook (Nginx + Let's Encrypt)
1. Basic Nginx server block
   - `/etc/nginx/sites-available/example.conf`
     ```
     server {
       listen 80;
       server_name example.com www.example.com;
       location / { proxy_pass http://127.0.0.1:8080; }
     }
     ```
2. Obtain TLS cert with Certbot
   ```bash
   sudo apt-get install certbot python3-certbot-nginx
   sudo certbot --nginx -d example.com -d www.example.com
   ```
3. Test
   ```bash
   curl -I http://example.com
   curl -vk https://example.com
   openssl s_client -connect example.com:443 -servername example.com
   ```
4. Secure
   - Use strong TLS config in nginx:
     ```
     ssl_protocols TLSv1.2 TLSv1.3;
     ssl_ciphers 'ECDHE-ECDSA-...:!RC4:!aNULL:!eNULL';
     ssl_prefer_server_ciphers on;
     add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
     ```
   - Use WAF, rate limiting for public endpoints.
5. Troubleshoot
   - 502: check upstream (`journalctl` / app logs)
   - 403: check `root` and `location` permissions, SELinux context on files.

Interview tip: explain TLS handshake and SNI.

---

### D. MySQL — runbook
1. Configure
   - `/etc/mysql/my.cnf` or `/etc/mysql/mysql.conf.d/mysqld.cnf`
     ```
     [mysqld]
     bind-address = 10.0.5.10
     ```
   - Create user with limited privileges:
     ```sql
     CREATE USER 'app'@'10.0.5.%' IDENTIFIED BY 'securepassword';
     GRANT SELECT, INSERT, UPDATE ON appdb.* TO 'app'@'10.0.5.%';
     ```
2. Test
   ```bash
   mysql -h 10.0.5.10 -P 3306 -u app -p
   ss -tln | grep 3306
   ```
3. Secure
   - Prefer `bind-address` to private IP.
   - Use TLS between app and DB.
   - Regular backups and failover (replication or managed RDS).
4. Troubleshoot
   - Connection refused: firewall / `bind-address`.
   - Auth error: check user host part and password, `SHOW GRANTS FOR 'app'@'host'`.

Interview tip: discuss replication types, failover strategies, and how to secure DB access in cloud.

---

### E. kube-apiserver — runbook
1. Manifests (kubeadm static pod):
   - `/etc/kubernetes/manifests/kube-apiserver.yaml` contains certs and flags like `--authorization-mode=Node,RBAC`, `--audit-log-path`.
2. Test
   ```bash
   kubectl --kubeconfig /etc/kubernetes/admin.conf get componentstatuses
   curl -k https://<apiserver>:6443/healthz
   ```
3. Secure
   - RBAC enabled, audit logs configured, API server access restricted by SG/NACL.
   - Use OIDC for auth if integrating with enterprise identity.
   - Encrypt secrets at rest: `EncryptionConfiguration` for API server.
4. Troubleshoot
   - If API server unreachable: check kubelet (static pod) logs, etcd health.
   - etcd issues: check `ETCDCTL_API=3 etcdctl --endpoints=<list> endpoint health`.

Interview tip: explain control plane roles, etcd significance, RBAC concepts, and network segregation for control plane.

---

## WireGuard minimal example
1. Generate keys
```bash
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key
```
2. Server `/etc/wireguard/wg0.conf`
```ini
[Interface]
Address = 10.10.10.1/24
PrivateKey = <server_private_key>
ListenPort = 51820
PostUp = sysctl -w net.ipv4.ip_forward=1; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.10.10.2/32
```
3. Client config `/etc/wireguard/wg-client.conf`
```ini
[Interface]
Address = 10.10.10.2/24
PrivateKey = <client_privkey>

[Peer]
PublicKey = <server_pubkey>
Endpoint = server.example.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```
4. Start
```bash
sudo systemctl enable --now wg-quick@wg0
```
5. Test
```bash
ping -c 3 10.10.10.1
curl --interface 10.10.10.2 http://ifconfig.me
```
Security note: restrict port 51820 to trusted clients in SG or use client auth via keys.

---

## Master YAML dataset (sample)
Store master data in YAML so docs, Terraform, and playbooks can be generated from a single source.

Example `protocols.yml` snippet:
```yaml
- service: ssh
  ports: [22]
  proto: tcp
  description: "Secure Shell"
  linux_config: "/etc/ssh/sshd_config"
  test: "ssh -i ~/.ssh/id_rsa user@host"
  security: "Use key auth, disable password auth, restrict SG"
  cloud: "Allow from bastion CIDR"

- service: http
  ports: [80]
  proto: tcp
  description: "HTTP"
  linux_config: "/etc/nginx/nginx.conf"
  test: "curl -I http://host"
  security: "Redirect to HTTPS"
  cloud: "ALB health checks"
```
Automation idea: a small Python/Go/Node script reads the YAML and renders the Markdown tables, or generates cloud SGs.

---

## Automation & CI suggestions
- Keep master data as YAML; render README.md in CI pipeline.
- Validate the dataset: schema check (ports numeric, proto one of TCP/UDP/BOTH).
- Lint generated Markdown (markdownlint).
- Run tests to ensure commands are valid and place sample smoke tests in CI (e.g., test HTTP endpoint after deployment).
- When changing firewall rules in code (Terraform/CloudFormation), require PR with owner and expiry embedded.

---

## Formatting, UX & publish tips
- Put Cheat-sheet at the top and Detailed Reference below.
- Use anchors for long tables, e.g., `## Detailed reference` so team can link specific sections.
- Keep one sentence justification for firewall rules, not paragraphs.
- Provide copy/paste CLI blocks adjacent to each table row where practical.
- Add "Last reviewed" metadata at the top of README.
- Provide a printable one-page cheatsheet as a separate markdown or PDF.

---

## Export & printing suggestions
- To produce a printable PDF:
  - Render Markdown to HTML (e.g., `pandoc README.md -o README.pdf --pdf-engine=wkhtmltopdf`).
  - Or use GitHub’s "Print" view from the repository page.
- For intranet docs, generate as HTML with collapsible code sections to keep pages compact.

---

## Final checklist before publishing
- [ ] Every table row has port(s), proto, short description, one-line test, at least one hardening note.
- [ ] Firewall exceptions include Owner and Expiry.
- [ ] YAML master dataset exists and is tested to render tables.
- [ ] Runbook entries include commands for configure, test, secure, troubleshoot.
- [ ] Add "Last reviewed" date and author at top of README.
- [ ] Provide links to tickets/PRs for temporary firewall rules.

---

## Appendix: Common commands & snippets
- Interfaces and addresses
  ```bash
  ip addr show
  ip link show
  ifconfig -a     # legacy
  ```
- Routes and policy routing
  ```bash
  ip route show
  ip route add 10.10.0.0/16 via 192.168.1.1
  ip rule add from 10.0.5.0/24 table 100
  ```
- Connectivity/diagnostics
  ```bash
  ping -c 4 8.8.8.8
  traceroute google.com
  mtr -r -c 100 google.com
  ss -tulnp
  lsof -iTCP -sTCP:LISTEN -P -n
  ```
- Packet capture
  ```bash
  sudo tcpdump -i eth0 -n -s 0 host 10.0.5.10 -w /tmp/cap.pcap
  ```
- Firewall examples
  ```bash
  # nftables
  sudo nft add table inet filter
  sudo nft 'add chain inet filter input { type filter hook input priority 0; policy drop; }'
  sudo nft add rule inet filter input iif "lo" accept
  sudo nft add rule inet filter input ct state established,related accept
  sudo nft add rule inet filter input tcp dport {22,80,443} accept
  ```
- Useful parsing & conversion
  - ipcalc / sipcalc for CIDR calculations

---

## References & further reading
- RFCs: RFC 791 (IPv4), RFC 2460 (IPv6), RFC 1035 (DNS), RFC 1918 (private IPv4)
- iproute2 manual: `man ip`
- nftables wiki: https://wiki.nftables.org
- Kubernetes networking: https://kubernetes.io/docs/concepts/cluster-administration/networking/
- WireGuard: https://www.wireguard.com/
- Cloud provider networking docs (AWS VPC / GCP VPC / Azure VNet)

---

Last reviewed: 2025-12-28  
Author: ops/docs-team
