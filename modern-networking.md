# Building Blocks of Modern Networking — Detailed Reference

This repository contains a comprehensive, practical reference for core networking components and their place in modern architectures. Each component is described with:

- What it is
- How it works technically
- A real-world example / topology
- Typical protocols and ports
- Cloud relevance (AWS / Azure / GCP)
- Best practices, monitoring, and troubleshooting notes
- Short configuration or command examples where applicable

Intended audience: system engineers, network engineers, SREs, cloud engineers, and students.

---

Table of contents

- Core Networking
  - Switch
  - Router
  - SD‑WAN
  - DNS
  - DHCP
  - NTP
- Network Security
  - Firewall
  - VPN
  - IDS / IPS
- Delivery
  - Load Balancer
  - Reverse Proxy
  - API Gateway
- Identity & Trust
  - Identity Provider (IdP)
  - RADIUS / AAA
  - PKI
- Operations
  - SIEM
  - NMS
- Edge
  - Access Point (AP)
  - IoT Gateway
- Infrastructure
  - NFV

---

## CORE NETWORKING

### Switch

What it is
- Layer 2 device that forwards Ethernet frames between connected devices on the same broadcast domain.

How it works (technical)
- Uses MAC addresses in a CAM (Content Addressable Memory) table to map ports → MAC.
- Forwards frames by learning source MACs and updating the CAM table.
- Supports VLAN tagging (802.1Q) to partition traffic.
- Runs STP/RSTP/MSTP to prevent layer-2 loops.
- May provide L3 interfaces (SVIs) on multilayer/managed switches.

Real-world example
- Office floor: PCs → access switch → distribution switch → core switch → router/firewall.

Typical protocols / ports
- No TCP/UDP ports (Ethernet/Layer 2). Protocols: 802.1Q, LLDP/CDP, STP, LACP (802.3ad), 802.1X (port authentication).

Cloud relevance
- Cloud networks use virtual switching: VPC subnets, virtual NICs, security/traffic steering. Concepts map to subnets and virtual network appliances.

Best practices
- Use management VLAN separate from user VLANs.
- Limit STP root bridging to core switch.
- Use port security (MAC limit) and 802.1X for NAC.
- Monitor via SNMP, NetFlow/sFlow, or telemetry (gNMI/RESTCONF).

Troubleshooting tips
- Verify VLAN membership and tagging.
- Check CAM table for MAC learning.
- Use `show spanning-tree` (vendor CLI) for blocked ports.

Example (VLAN tagging concept)
```text
switchport mode trunk
switchport trunk allowed vlan 10,20,30
```

---

### Router

What it is
- Layer 3 device that routes IP packets between networks/subnets.

How it works (technical)
- Maintains routing table (static or dynamic).
- Uses routing protocols (OSPF, BGP, EIGRP, IS‑IS) to exchange prefixes.
- Performs packet forwarding, NAT, ACLs, and sometimes firewalling.
- Interfaces can be physical or logical (sub-interfaces).

Real-world example
- Data center: core router peers with ISP via BGP and advertises internal prefixes; uses route maps for traffic engineering.

Typical protocols / ports
- IP routing protocols: OSPF (89), BGP (179), IS‑IS (IPv6/CLNS), RIP (520). Management: SSH (22), SNMP (161/162), NETCONF/gNMI (various).

Cloud relevance
- Cloud route tables (AWS Route Tables), Cloud Router (GCP BGP), Azure UDR. Virtual routers provide similar functionality.

Best practices
- Use route aggregation and consistent addressing.
- Secure BGP (TTL, MD5/TTL, prefix filters, RPKI where supported).
- Use route reflectors for scaling in large networks.

Troubleshooting tips
- `show ip route`, `traceroute`, `ping`, `show bgp summary`.

Example: Simple static route (Cisco-like)
```text
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

---

### SD‑WAN (Software Defined WAN)

What it is
- Centralized control plane managing connectivity between sites over multiple transports (broadband, LTE, MPLS).

How it works (technical)
- Control plane defines policies; data plane builds encrypted tunnels (IPsec/DTLS/WireGuard-like).
- Path selection based on latency, loss, jitter or application signatures.
- Application-aware steering and QoS enforcement.

Real-world example
- Retail chain branches route voice traffic over a prioritized low-latency ISP and bulk backup over cheaper broadband, with fallback if path fails.

Typical protocols / ports
- IPsec/DTLS tunnels (UDP/TCP depending on implementation), control plane often TLS over TCP/443.

Cloud relevance
- Many SD-WAN vendors provide virtual appliances in cloud marketplaces and integrate with cloud routing and security services (e.g., SASE with cloud on-ramps).

Best practices
- Use local internet breakout for SaaS apps with secure segmentation.
- Monitor tunnel health and application SLAs.

---

### DNS (Domain Name System)

What it is
- Distributed naming system that translates domain names to IP addresses and other records.

How it works (technical)
- Hierarchical namespace (root → TLD → authoritative nameservers).
- Resolver behavior: iterative or recursive queries.
- Record types: A, AAAA, CNAME, MX, TXT, NS, SRV, PTR, SOA, etc.
- Caching reduces lookups; TTL controls cache duration.

Real-world example
- web.example.com → A record → 198.51.100.10

Typical protocols / ports
- UDP 53 (queries), TCP 53 (zone transfers, large responses, DNS over TCP), DNS over HTTPS (DoH: 443), DNS over TLS (DoT: 853).

Cloud relevance
- Route 53, Cloud DNS, Azure DNS provide managed authoritative services and integrations (health checks, private DNS zones).

Best practices
- Use split-horizon/private DNS for internal names.
- Configure DNSSEC for integrity where possible.
- Monitor for DNS poisoning and authoritative server reachability.

Troubleshooting tips
- `dig +trace example.com`, `dig @nameserver example.com`, check TTL and SOA.

Example (BIND zone fragment)
```bind
$ORIGIN example.com.
@   3600 IN SOA ns1.example.com. hostmaster.example.com. (
        2026010101 ; serial
        7200       ; refresh
        3600       ; retry
        1209600    ; expire
        3600 )     ; minimum
@   3600 IN NS ns1.example.com.
@   3600 IN A  198.51.100.10
www 3600 IN CNAME @
```

---

### DHCP (Dynamic Host Configuration Protocol)

What it is
- Automates assignment of IP address, gateway, DNS, and other options.

How it works (technical)
- Four-step DORA: Discover → Offer → Request → Acknowledge.
- Leases have expiry: renew via DHCPREQUEST/ACK.
- Offers options via DHCP options (router/gateway, domain-name-servers, lease time).

Real-world example
- Laptop joins enterprise Wi‑Fi and receives IP from DHCP server, VLAN provided via controller.

Typical protocols / ports
- UDP 67 server, UDP 68 client. DHCP Relay uses IP helper on routers.

Cloud relevance
- Cloud VPCs allocate IPs via cloud-managed DHCP-like services to instances; cloud DHCP options sets (e.g., for DNS servers).

Best practices
- Separate scopes by VLAN/subnet.
- Reserve leases for infrastructure devices.
- Secure via DHCP snooping on switches.

Troubleshooting tips
- `tcpdump -i eth0 port 67 or port 68`, check DHCP server logs.

Example (ISC dhcpd.conf snippet)
```conf
subnet 192.168.10.0 netmask 255.255.255.0 {
  range 192.168.10.100 192.168.10.200;
  option routers 192.168.10.1;
  option domain-name-servers 8.8.8.8, 8.8.4.4;
  default-lease-time 600;
  max-lease-time 7200;
}
```

---

### NTP (Network Time Protocol)

What it is
- Synchronizes clocks across devices; critical for logs, Kerberos, TLS, PKI.

How it works (technical)
- Uses hierarchical stratum model (stratum 0 devices → stratum 1 → clients).
- Protocol exchanges measure round-trip delay and offset; algorithms discipline local clocks.

Typical protocols / ports
- UDP 123

Real-world example
- All server VMs sync to a pool (pool.ntp.org) or internal stratum 1 for a regulated network.

Best practices
- Use a small set of reliable upstream time sources; prefer multiple sources for accuracy.
- Use authenticated NTP (autokey) or use NTP over VPN for security; consider chrony for better VM performance.

Troubleshooting tips
- `ntpq -p`, `chronyc sources`, check offset and jitter metrics.

Chrony example (chrony.conf)
```conf
pool 2.pool.ntp.org iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
```

---

## NETWORK SECURITY

### Firewall

What it is
- Controls which traffic is allowed or denied between network segments.

How it works (technical)
- Packet-filtering layer, stateful inspection (tracking connections), application-layer inspection (NGFW), DPI and TLS inspection on advanced appliances.
- Policies: allow/deny rules, NAT, port forwarding, zone-based policies.

Real-world example
- Edge firewall blocks inbound unsolicited traffic, allows outbound HTTPS, restricts SSH to jump boxes.

Typical protocols / ports
- Depends on rules; management via SSH/HTTPS, logging to syslog/SIEM.

Cloud relevance
- Security Groups (AWS), NSGs (Azure), VPC firewall rules (GCP) provide host- or subnet-level control.

Best practices
- Deny-by-default posture, least privilege, separate management plane.
- Use centralized logging and monitor dropped flows.
- Implement change control for firewall rules.

Troubleshooting tips
- Check rule order (first-match), check NAT, use packet capture on appliance.

iptables example (allow HTTPS, drop others)
```bash
# flush
iptables -F
iptables -P INPUT DROP
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
```

---

### VPN (Virtual Private Network)

What it is
- Secure tunnel between endpoints across an untrusted network.

How it works (technical)
- Tunnels created via IPsec (IKE), SSL/TLS (OpenVPN), or modern crypto (WireGuard).
- Provides encryption, integrity, and often authentication (certs, pre-shared keys, or username/password).

Types and examples
- Site-to-site: connects branch networks.
- Remote access: client-to-site for employees.
- SSL VPN for web/application access.

Typical protocols / ports
- IPsec (IKEv2 UDP 500/4500), OpenVPN (UDP/TCP 1194 default), WireGuard (UDP, port configurable), SSL VPN (TCP 443).

Cloud relevance
- Cloud VPN gateways (AWS Site-to-Site VPN, Azure VPN Gateway, GCP VPN) and transit hub connectivity.

Best practices
- Use strong encryption suites, perfect forward secrecy, split or full tunneling depending on needs.
- Monitor for rekeys and tunnel flaps.

Example: minimal WireGuard peer config
```ini
[Interface]
PrivateKey = <host-private-key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <peer-public-key>
AllowedIPs = 10.0.0.2/32
Endpoint = example.org:51820
```

---

### IDS / IPS

What it is
- IDS: detection and alerting; IPS: inline blocking of malicious traffic.

How it works (technical)
- Signature-based: matching patterns (Snort, Suricata).
- Anomaly-based: behavioral deviations.
- Can use DPI, protocol parsing, and reassembly for HTTP/SMTP inspection.

Real-world example
- IDS detects port scans; IPS blocks attempts and raises alerts to security team.

Best practices
- Tune signatures to avoid false positives.
- Use as part of layered defenses (WAF, firewall, endpoint protections).
- Feed alerts into SIEM for correlation.

Troubleshooting tips
- Validate sensor positioning, check packet loss or performance constraints in inline deployment.

---

## DELIVERY

### Load Balancer

What it is
- Distributes client traffic across multiple backend servers for HA and scaling.

How it works (technical)
- L4 (TCP/UDP): connection forwarding/handover.
- L7 (HTTP/S): content-aware routing, host/path routing, TLS termination, header manipulation.
- Health checks to remove unhealthy targets from rotation.

Real-world example
- ALB routes api.example.com/v1 to service A and /v2 to service B; NLB handles TLS passthrough for Game servers.

Typical protocols / ports
- HTTP/HTTPS: 80/443, TCP/UDP for L4 services.

Cloud relevance
- AWS ALB/NLB, Azure Load Balancer & Application Gateway, GCP Load Balancing with global LB and backend services.

Best practices
- Use health checks tuned to application behavior.
- Use connection draining and graceful shutdown for deployments.
- Offload TLS if needed for performance, but consider end-to-end encryption.

Example (HAProxy simple)
```haproxy
frontend http_in
  bind *:80
  default_backend app_pool

backend app_pool
  balance roundrobin
  server app1 10.0.0.11:80 check
  server app2 10.0.0.12:80 check
```

---

### Reverse Proxy

What it is
- Intermediary for requests from clients to backend servers, often handles TLS, caching, rate-limiting.

How it works (technical)
- Accepts client connections, optionally terminates TLS, forwards requests to internal servers, can rewrite headers and responses.

Real-world example
- Nginx as TLS terminator, static content cache & request routing to upstream app servers.

Best practices
- Offload TLS but use secure backend connections for sensitive data.
- Use caching for static assets and configure correct cache headers.

Example (nginx reverse proxy snippet)
```nginx
server {
  listen 443 ssl;
  server_name www.example.com;
  ssl_certificate /etc/ssl/certs/example.crt;
  location / {
    proxy_pass http://app_pool;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

---

### API Gateway

What it is
- Managed front door for APIs offering routing, authentication, rate-limiting, analytics, and monetization controls.

How it works (technical)
- Performs request validation, auth (JWT/OAuth), throttling, analytics, transformations, and service-level routing.

Real-world example
- AWS API Gateway fronting microservices with Cognito-based auth and usage plans.

Best practices
- Apply authentication and rate limits at the edge.
- Use structured logging and tracing headers for observability.

---

## IDENTITY & TRUST

### Identity Provider (IdP)

What it is
- Central authentication and identity management service.

How it works (technical)
- Implements SAML, OAuth2, and OpenID Connect for federated auth and SSO.
- Issues tokens (SAML assertions, JWTs) used for authentication and authorization.

Real-world example
- Okta or Azure AD providing SSO to cloud apps and VPN authentication.

Best practices
- Enforce MFA (2FA) and conditional access policies.
- Use short-lived tokens and fine-grained scopes.

---

### RADIUS / AAA

What it is
- Remote Authentication Dial-In User Service used for authenticating access (Wi‑Fi, VPN, network device logins).

How it works (technical)
- RADIUS handles Authentication, Authorization, Accounting; integrates with LDAP/AD for credentials and with NPS/RADIUS servers.

Real-world example
- Enterprise Wi‑Fi uses 802.1X with RADIUS backend; accounting logs sessions for auditing.

Ports
- UDP 1812 (auth), UDP 1813 (accounting); older deployments use 1645/1646.

Best practices
- Secure RADIUS traffic with IPsec or dedicated management networks.
- Use redundancy and monitoring; apply strict shared secret management.

---

### PKI (Public Key Infrastructure)

What it is
- Infrastructure for issuing, revoking, and managing certificates and their keys.

How it works (technical)
- Root CA → intermediate CAs → issuance of certificates; CRLs and OCSP for revocation.
- Certificates used in TLS, signing, S/MIME, code signing, client auth.

Real-world example
- Internal CA issues certificates for internal services; public CA for internet-facing domains.

Best practices
- Keep root CA offline; use intermediates for issuing.
- Automate certificate issuance and renewal (ACME/Let's Encrypt for public certs).
- Enforce key protection (HSMs for high-assurance keys).

Troubleshooting tips
- Check certificate chain, expiration, and correct SANs; test with `openssl s_client -connect host:443 -servername host`.

---

## OPERATIONS

### SIEM

What it is
- Security Information and Event Management collects logs/events, correlates alerts, and supports investigation.

How it works (technical)
- Ingests logs (syslog, API, agents), normalizes, applies correlation/rules, generates alerts and dashboards. Supports retention and compliance search.

Real-world example
- Splunk or Elastic SIEM indexing firewall, IDS, authentication logs to detect suspicious login patterns.

Best practices
- Centralize time-synced logs, tune use cases to reduce false positives, automate incident response (SOAR).

---

### NMS (Network Management System)

What it is
- Centralized monitoring for network devices: health, performance, bandwidth, and alerts.

How it works (technical)
- Uses SNMP, ICMP, syslog, sFlow/NetFlow/IPFIX for metrics and flow data. Provides dashboards and alerting.

Real-world example
- SolarWinds or Zabbix polling switch CPU, interface errors, and sending alerts for flapping ports.

Best practices
- Ensure SNMPv3 for secure telemetry; use polling intervals to balance timeliness vs. load.

---

## EDGE

### Access Point (AP)

What it is
- Wireless device bridging clients to wired network (Wi‑Fi).

How it works (technical)
- Implements 802.11 standards (a/b/g/n/ac/ax), provides SSIDs, encryption (WPA2/WPA3), roaming, and QoS for voice.

Real-world example
- Enterprise controller-managed APs distribute SSIDs, enforce VLAN tagging per SSID, and perform RF management.

Best practices
- Use WPA2/WPA3 Enterprise with 802.1X + RADIUS.
- Plan channels and power to minimize co-channel interference.

---

### IoT Gateway

What it is
- Local aggregator and translator for IoT devices bridging to cloud services.

How it works (technical)
- Converts protocols (MQTT, CoAP, Modbus) to cloud-friendly formats, performs local filtering/processing, and enforces device identity.

Real-world example
- Factory sensors publish MQTT to a local gateway which forwards to AWS IoT Core with device credentials.

Best practices
- Isolate IoT networks, apply device identity and certificate-based auth, and do local filtering to reduce cloud costs.

---

## INFRASTRUCTURE

### NFV (Network Function Virtualization)

What it is
- Virtualizing network functions (firewall, router, WAN optimizer) into software running on commodity servers.

How it works (technical)
- VNFs run in VMs or containers. Service chaining and orchestration (MANO) provide lifecycle management and forwarding graphs.

Real-world example
- Running virtual NGFW in cloud or data center to apply consistent security policies without physical appliances.

Cloud relevance
- Cloud native equivalents: managed firewalls, virtual appliances in marketplaces, service mesh for east-west controls.

Best practices
- Monitor per-VNF performance; ensure CPU/network resource reservation for predictable latency.

---

## GENERAL BEST PRACTICES ACROSS LAYERS

- Defense in depth: layered security controls (network, host, application).
- Principle of least privilege for network access and management APIs.
- Centralized logging and observability: correlate network telemetry with application logs and traces.
- Automate provisioning and configuration via IaC (Terraform, Ansible).
- Use version control for configs and apply change control.
- Test failover and DR procedures regularly.
- Maintain updated firmware/software and track CVEs for appliances.

---

## TROUBLESHOOTING CHEAT SHEET

- Connectivity: `ping`, `traceroute`, `mtr`, check ARP/CAM tables.
- Service reachability: `curl`, `telnet host port`, `ss -tuln`, `netstat`.
- DNS: `dig +trace`, `host`, `nslookup`.
- DHCP: `tcpdump -i any port 67 or port 68`.
- Time issues: `chronyc sources`, `ntpq -p`.
- Firewall issues: check rules order, state tables (`conntrack -L`), and NAT translations.
- Logs: centralize and search with timestamps; correlate user/host action with network events.

---

## SAMPLE LAB EXERCISES (suggested)

1. Build a small lab with:
   - Two subnets, a router, a managed switch with 2 VLANs, and an HTTP server.
   - Configure DHCP for both VLANs, DNS, and test inter-VLAN routing and ACLs.

2. Deploy a simple HAProxy load balancer and two web servers; simulate failure and observe health checks and session behavior.

3. Set up a WireGuard site-to-site tunnel between two VMs and route a subnet across it.

4. Configure a basic IDS (Suricata), generate attacks with `nmap`/`sqlmap` and see detections.

---


License
- This documentation is provided as-is for educational and operational reference. Adapt for your environment and validate configs before production use.
