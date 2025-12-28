# Networking for Linux / DevOps / Cloud Engineers
A progressive, practical guide from fundamentals to advanced topics — designed for Linux administrators, DevOps engineers, and cloud/networking practitioners. Includes commands, configuration examples, cloud context (AWS/GCP/Azure), troubleshooting workflows, security notes, interview talking points, and a printable cheat-sheet.

Table of contents
- 0. How to use this document
- 1. Fundamentals (Beginner)
  - 1.1 Core concepts
  - 1.2 Addressing: IPv4, IPv6, MAC
  - 1.3 Subnetting & CIDR
  - 1.4 OSI & TCP/IP models
- 2. Common Network Services (Beginner → Intermediate)
  - 2.1 DNS
  - 2.2 DHCP
  - 2.3 ARP
  - 2.4 Routing & NAT
- 3. Linux Networking Essentials (Intermediate)
  - 3.1 Viewing and configuring interfaces
  - 3.2 Routing
  - 3.3 VLANs, Bridges, and Tunnels
  - 3.4 Firewalls (iptables, nftables, firewalld, UFW)
- 4. VPNs, Tunnels & Encryption (Intermediate → Advanced)
  - 4.1 WireGuard
  - 4.2 OpenVPN & IPsec (Strongswan)
  - 4.3 SSH tunnels
- 5. Containers, Kubernetes & CNI (Intermediate → Advanced)
  - 5.1 Docker networking
  - 5.2 Kubernetes networking model & CNI plugins
  - 5.3 Service types, Ingress, and Load Balancing
- 6. Cloud Networking (Advanced)
  - 6.1 AWS VPC essentials
  - 6.2 GCP networking essentials
  - 6.3 Azure networking essentials
  - 6.4 Hybrid connectivity patterns
- 7. Routing Protocols & Enterprise Concepts (Advanced)
  - 7.1 BGP basics
  - 7.2 OSPF, VRRP, HSRP, Anycast
- 8. Performance, QoS & MTU (Advanced)
- 9. Monitoring, Observability & Troubleshooting (All levels)
  - 9.1 Tooling
  - 9.2 Systematic troubleshooting playbook
  - 9.3 Common problems & fixes
- 10. Security & Hardening
- 11. Interview Topics & Example Questions
- 12. Quick Reference / Cheat Sheet (Printable)
- 13. Further reading & resources
- Appendix: Useful commands & config snippets

---

0. How to use this document
- Read sequentially if new to networking.
- Skip to sections relevant to tasks (e.g., "Linux Networking Essentials" for day-to-day ops).
- Copy-paste examples into lab VMs, validate, and adapt to your environment.
- Use the Cheat Sheet (Section 12) for quick lookup.

---

1. Fundamentals

1.1 Core concepts
- Network: connected devices exchanging data.
- Packet: unit of data at Layer 3 (IP).
- Frame: unit at Layer 2 (Ethernet).
- Port: endpoint for transport protocols (e.g., TCP/UDP).

1.2 Addressing: IPv4, IPv6, MAC
- IPv4: 32-bit (e.g., `192.168.1.10`).
- IPv6: 128-bit (e.g., `2001:db8::1`).
- MAC: 48-bit hardware address (e.g., `00:1A:2B:3C:4D:5E`).
- Private IPv4 ranges (RFC1918): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.

1.3 Subnetting & CIDR (practical)
- CIDR example: `10.10.0.0/16` → 65536 addresses, `/24` → 256 addresses (254 usable).
- Tools: `ipcalc`, `sipcalc`.
- Example:
  - `ipcalc 192.168.1.10/24` shows network, broadcast, usable hosts.

1.4 OSI & TCP/IP models
- OSI: 7 layers — helpful for troubleshooting mapping (e.g., DNS is Layer 7; ARP is Layer 2/3 helper).
- TCP/IP: 4 layers used in practice.

---

2. Common Network Services

2.1 DNS (Domain Name System)
- Purpose: map names → IPs (A/AAAA), provide discovery (SRV), mail routing (MX).
- Commands:
  - `dig +short example.com A`
  - `dig @8.8.8.8 example.com ANY +noall +answer`
  - `nslookup example.com`
  - `host example.com`
- System: `/etc/resolv.conf` (may be managed by systemd-resolved/NetworkManager).
- Cloud: Route 53 (AWS), Cloud DNS (GCP), Azure DNS.
- Troubleshoot: `dig +trace`, check DNSSEC, TTLs, delegation.

2.2 DHCP
- Provides IP, subnet mask, gateway, DNS.
- Client: `dhclient` / `systemd-networkd` / `NetworkManager`.
- Release/renew:
  - `sudo dhclient -r eth0`
  - `sudo dhclient eth0`
- Check server logs (e.g., `dhcpd` / cloud provider console for VPC DHCP options).

2.3 ARP (Address Resolution Protocol)
- Maps IPv4 → MAC within a broadcast domain.
- Commands:
  - `ip neigh show`
  - `arp -n`
- Flush ARP:
  - `sudo ip neigh flush all`

2.4 Routing & NAT
- Routing table:
  - View: `ip route show`
  - Add: `sudo ip route add 10.10.0.0/16 via 192.168.1.1`
- NAT:
  - SNAT: change source IP for outbound traffic (e.g., masquarade).
  - DNAT: port-forward inbound traffic to internal hosts.
- Example iptables NAT:
  ```bash
  sudo sysctl -w net.ipv4.ip_forward=1
  sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
  sudo iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination 192.168.1.10:22
  ```

---

3. Linux Networking Essentials

3.1 Viewing & configuring interfaces
- View:
  - `ip addr show`
  - `ip link show`
- Temporary IP:
  - `sudo ip addr add 192.168.1.50/24 dev eth0`
  - `sudo ip link set eth0 up`
- Persistent:
  - Ubuntu (netplan): `/etc/netplan/*.yaml` then `sudo netplan apply`.
  - RHEL (ifcfg): `/etc/sysconfig/network-scripts/ifcfg-eth0`.
  - systemd-networkd: `/etc/systemd/network/*.network`.

3.2 Routing
- Show routing:
  - `ip route show`
- Get next-hop:
  - `ip route get 8.8.8.8`
- Policy-based routing:
  - `ip rule add from 10.0.5.0/24 table 100`
  - `ip route add default via 192.168.1.1 table 100`

3.3 VLANs, Bridges, Tunnels
- Create VLAN:
  ```bash
  sudo ip link add link eth0 name eth0.10 type vlan id 10
  sudo ip link set eth0.10 up
  sudo ip addr add 192.168.10.2/24 dev eth0.10
  ```
- Linux bridge:
  ```bash
  sudo ip link add name br0 type bridge
  sudo ip link set eth1 master br0
  sudo ip link set br0 up
  ```
- GRE/VXLAN tunnels (overlay networks for multi-host L2):
  - VXLAN example: `ip link add vxlan100 type vxlan id 100 dev eth0 dstport 4789 local <local-ip>`

3.4 Firewalls
- iptables (legacy) / nftables (recommended) / firewalld / UFW.
- nftables basic:
  ```bash
  sudo nft add table inet filter
  sudo nft 'add chain inet filter input { type filter hook input priority 0; policy drop; }'
  sudo nft add rule inet filter input iif lo accept
  sudo nft add rule inet filter input ct state established,related accept
  sudo nft add rule inet filter input tcp dport 22 accept
  ```

---

4. VPNs, Tunnels & Encryption

4.1 WireGuard (simple & fast)
- Generate keys:
  - `wg genkey | tee privatekey | wg pubkey > publickey`
- Server config `/etc/wireguard/wg0.conf`:
  ```
  [Interface]
  Address = 10.10.10.1/24
  PrivateKey = <server_privkey>
  ListenPort = 51820
  ```
- Start:
  - `sudo systemctl enable --now wg-quick@wg0`

4.2 OpenVPN & IPsec
- OpenVPN: flexible, cross-platform; server/client model.
- IPsec (Strongswan): robust for site-to-site with IKEv2.
- Common for corporate tunnels and cloud VPN gateways.

4.3 SSH tunnels
- Local port forward:
  - `ssh -L 8080:localhost:80 user@remote`
- Remote port forward:
  - `ssh -R 2222:localhost:22 user@remote`

---

5. Containers & Kubernetes Networking

5.1 Docker networking
- Types:
  - bridge (default), host, none, overlay (swarm).
- Inspect:
  - `docker network ls`
  - `docker network inspect bridge`

5.2 Kubernetes networking model
- Every pod has an IP; pods communicate directly (no NAT between pods).
- CNI plugins implement pod networking: Calico, Flannel, Cilium, Weave.
- kube-proxy provides Service IPs (iptables or IPVS mode).
- Policies: NetworkPolicy (pod-level egress/ingress rules).

5.3 Services & Ingress
- ClusterIP: internal access.
- NodePort: exposes port on nodes.
- LoadBalancer: cloud provider LB.
- Ingress: L7 (HTTP) routing via controllers (NGINX, Traefik, ALB ingress).

---

6. Cloud Networking

6.1 AWS
- VPC = virtual network. Subnets (public/private). IGW = internet gateway.
- NAT Gateway/Instance for private subnets egress.
- Security Groups (stateful, instance-level), NACLs (stateless, subnet-level).
- Route Tables, Elastic IPs, ENIs (Elastic Network Interfaces), Transit Gateway for multi-VPC connectivity.
- Debug checklist: Security Group, NACL, Route Table, subnet CIDR, source/destination check (for NAT instances).

6.2 GCP
- VPC auto or custom subnet mode (global VPC).
- Cloud NAT for private VM egress.
- VPC Peering, Shared VPC, Cloud Router for dynamic routing (BGP).

6.3 Azure
- VNet, Subnets, NSG (Network Security Group), UDR (user-defined route), Azure Firewall, ExpressRoute for private connectivity.

6.4 Hybrid connectivity
- VPN (IPsec) or DirectLink/DirectConnect/ExpressRoute for private on-prem ↔ cloud connectivity.
- Transit gateway architectures for multi-VPC/hybrid designs.

---

7. Routing Protocols & Enterprise Concepts

7.1 BGP basics
- Exterior routing protocol carrying prefix advertisements between ASes.
- Peering types: eBGP (between ASes), iBGP (within AS).
- Attributes: AS path, local-preference, MED.
- Use-cases: multi-homing, CDN, route control.

7.2 Other protocols
- OSPF: internal router protocol in large networks.
- VRRP / HSRP: router redundancy for default gateway failover.
- Anycast: same IP announced from multiple locations (useful for DNS, CDNs).

---

8. Performance, QoS & MTU

- MTU: default 1500. Jumbo frames often 9000 (requires whole path support).
- Path MTU Discovery may fail if ICMP blocked — causes fragmentation issues.
- `ip link set dev eth0 mtu 1400`
- QoS & shaping: `tc` (qdisc, classes, filters). Use cases: rate limit, prioritize SSH/VOIP.

---

9. Monitoring, Observability & Troubleshooting

9.1 Tooling
- Packet capture: `tcpdump`, Wireshark
- Connectivity: `ping`, `traceroute`, `mtr`
- Sockets: `ss`, `netstat`
- Port & host scanning: `nmap`
- Bandwidth: `iperf3`
- Logs: `journalctl`, `/var/log/*`, cloud logs
- Observability: Prometheus, Grafana, ELK/EFK

9.2 Troubleshooting playbook
1. Reproduce the issue (who/what/when/impact).
2. Check physical/link layer: `ethtool eth0` (link, speed, duplex).
3. Interface & addressing: `ip addr`, `ip route`.
4. Ping gateway, DNS, internet (`ping <gateway>`, `ping 8.8.8.8`).
5. DNS resolution: `dig`, check `/etc/resolv.conf`.
6. Check firewall rules: `nft list ruleset` / `iptables -L`.
7. Trace path: `traceroute` / `mtr`.
8. Capture packets: `tcpdump -i eth0 host x.x.x.x -w /tmp/cap.pcap`.
9. Inspect application logs, system logs.
10. Escalate with capture and timeline.

9.3 Common problems & fixes
- No IP: check DHCP, `dhclient`, `systemd-networkd` logs.
- DNS slow: check recursive resolver, timeouts, DNSSEC issues.
- Fragmentation / SSL hang: lower MTU or allow ICMP.
- Intermittent drop: check duplex mismatch, high errors (`ip -s link`), switch logs.
- Cloud egress blocked: Security groups or NACLs misconfigured.

---

10. Security & Hardening
- Principle of least privilege (network segmentation).
- Harden SSH: keys, disable password auth, use MFA for bastion.
- Firewall-first: only open required ports; use host firewall + cloud SGs.
- Encrypt in transit (TLS, VPN).
- Monitor & alert: flow logs, IDS/IPS, anomaly detection.
- Patch regularly; maintain baseline configurations (CIS benchmarks).
- Use bastion hosts or SSM-managed sessions (avoid direct public ssh to instances).

---

11. Interview Topics & Example Questions
- Explain the difference: Switch vs Router vs Layer 3 Switch.
- How does ARP work and how to troubleshoot ARP issues?
- Why does Path MTU Discovery fail when ICMP is blocked?
- Walk me through how DNS resolution works from client to authoritative server.
- How do NAT and masquerading differ? Show an iptables example.
- Describe Kubernetes networking: how does a Service work? What does kube-proxy do?
- Explain BGP attributes: AS_PATH, local-pref, MED, communities — and how to control outbound path selection.

Example succinct answers (prep)
- DNS resolution: client queries stub resolver → recursive resolver → root → TLD → authoritative → response cached.
- NAT types: SNAT (source translation for egress), DNAT (dest translation for ingress); PAT uses ports to multiplex.

---

12. Quick Reference / Cheat Sheet

Common commands
- Interfaces: `ip addr show`, `ip link show`
- Routing: `ip route`, `ip route get 8.8.8.8`
- ARP: `ip neigh show`
- DNS: `dig +short example.com`, `nslookup example.com`
- Capture: `sudo tcpdump -i eth0 -n -s 0 -w capture.pcap`
- Ports: `ss -tulnp`
- Firewall: `sudo nft list ruleset`, `sudo iptables -L -n -v`
- MTU test: `ping -M do -s 1472 8.8.8.8` (adjust for header)
- DHCP: `sudo dhclient -r && sudo dhclient eth0`

Quick iptables NAT (outgoing)
```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

WireGuard quick start
```bash
wg genkey | tee privkey | wg pubkey > pubkey
# configure wg0.conf and start: sudo systemctl enable --now wg-quick@wg0
```

---

13. Further reading & resources
- RFC 791 (IPv4), RFC 2460 (IPv6), RFC 1918 (private IPv4)
- iproute2: `man ip`
- nftables wiki: https://wiki.nftables.org/
- WireGuard: https://www.wireguard.com/
- Kubernetes networking docs: https://kubernetes.io/docs/concepts/cluster-administration/networking/
- Cloud provider docs: AWS VPC, GCP VPC, Azure Virtual Networks

---

Appendix: Useful config snippets

Netplan (Ubuntu) example:
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses: [10.0.5.10/24]
      gateway4: 10.0.5.1
      nameservers:
        addresses: [8.8.8.8,1.1.1.1]
```

systemd-networkd example:
```
[Match]
Name=eth0

[Network]
DHCP=yes
```

nftables simple web server allow:
```bash
sudo nft add table inet filter
sudo nft 'add chain inet filter input { type filter hook input priority 0; policy drop; }'
sudo nft add rule inet filter input ct state established,related accept
sudo nft add rule inet filter input iif "lo" accept
sudo nft add rule inet filter input tcp dport {22,80,443} accept
```

---
