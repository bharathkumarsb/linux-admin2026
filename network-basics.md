# Linux & Infrastructure Networking — Comprehensive Reference

A practical, structured networking reference for Linux admins, DevOps engineers, and infrastructure operators. This README covers core concepts, protocols, devices, Linux commands, configuration examples, troubleshooting steps, security best practices, and cloud networking notes (AWS/GCP/Azure). Use this as a cheat sheet and a living document for day-to-day operations.

Table of contents
- 1. Fundamentals
  - IP addresses, MAC addresses, subnets
  - OSI vs TCP/IP models
- 2. Addressing & Name Resolution
  - IPv4 / IPv6, subnetting, CIDR
  - DNS, /etc/resolv.conf, dig/nslookup/host
- 3. Routing & NAT
  - Static vs dynamic routing, ip route examples
  - NAT types: SNAT/DNAT/PAT, iptables/nft examples
- 4. LAN Technologies & ARP
  - Ethernet, VLANs, ARP tables
- 5. Linux Networking Commands (cheat sheet)
- 6. Network Configuration Examples
  - Debian/Ubuntu (netplan), RHEL/CentOS (ifcfg), systemd-networkd
- 7. Firewalls & Security
  - iptables, nftables, firewalld, UFW, best practices
- 8. VPNs & Tunnels
  - WireGuard, OpenVPN, IPsec basics and quick examples
- 9. Containers & Kubernetes Networking
  - CNI concepts, host networking, port mapping, ingress
- 10. Cloud Networking (AWS/GCP/Azure)
  - VPC, subnets, IGW, NAT Gateway, security groups, NACLs
- 11. Monitoring & Troubleshooting
  - Tools, step-by-step troubleshooting checklist
- 12. Performance & MTU
- 13. Quick Reference & Common Tasks
- 14. Further reading & references

---

## 1. Fundamentals

### IP address (Internet Protocol)
- Identifies hosts at Layer 3.
- IPv4: 32-bit (e.g., `192.168.1.10`) — written commonly as dotted-decimal.
- IPv6: 128-bit (e.g., `2001:db8::1`) — required for large-scale addressing and modern networks.

Types:
- Public IP: routable on the Internet (assigned by ISP/cloud provider).
- Private IP: RFC1918 (IPv4) ranges:
  - `10.0.0.0/8`
  - `172.16.0.0/12`
  - `192.168.0.0/16`
- Link-local (IPv4: 169.254.0.0/16, IPv6: `fe80::/10`).

Static vs Dynamic
- Static: configured manually — good for servers and infrastructure.
- Dynamic: assigned via DHCP — common for clients.

CIDR and subnetting
- CIDR notation: `192.168.1.0/24` — prefix length indicates network size.
- Common masks: `/24` => 256 addresses (254 usable), `/16`, `/8`.
- Use `ipcalc` or `sipcalc` to compute ranges:
  - Example: `ipcalc 10.0.5.23/24`

### MAC Address (Layer 2)
- 48-bit hardware address (e.g., `00:1A:2B:3C:4D:5E`), used within a broadcast domain.
- Use `ip link` or `ip addr` to view interface MACs.

---

## 2. OSI vs TCP/IP Models (summary)

OSI (7 layers) — conceptual:
- 7 Application, 6 Presentation, 5 Session, 4 Transport (TCP/UDP), 3 Network (IP), 2 Data Link (Ethernet/MAC), 1 Physical (cables).

TCP/IP (practical / 4 layers):
- Application (HTTP/DNS/SSH), Transport (TCP/UDP), Internet (IP/ICMP), Network Access (Ethernet/ARP).

---

## 3. Addressing & Name Resolution

DNS
- Resolves names to IPs. Use:
  - `dig +short example.com`
  - `nslookup example.com`
  - `host example.com`
- System config: `/etc/resolv.conf` (may be managed by NetworkManager/systemd-resolved).
- Example dig: `dig @8.8.8.8 google.com A +noall +answer`

DHCP
- Assigns IP, gateway, netmask, DNS.
- Client tools: `dhclient`, `dhcpcd`, `systemd-networkd` DHCP client.
- Release and renew:
  - `sudo dhclient -r eth0`
  - `sudo dhclient eth0`

Hosts file
- Local static entries: `/etc/hosts`
  - `192.168.1.10 my-service.local my-service`

ARP (Address Resolution Protocol)
- Maps IP -> MAC on a LAN.
- View ARP table:
  - `ip neigh` or `arp -n`
- Clear ARP cache:
  - `sudo ip neigh flush all`

---

## 4. Routing & NAT

Routing table
- View: `ip route show`
- Example:
  - `default via 192.168.1.1 dev eth0 proto dhcp metric 100`
- Add static route:
  - `sudo ip route add 10.10.0.0/16 via 192.168.1.254 dev eth1`

NAT
- Purpose: translate private IPs to a public IP (and vice versa).
- Types:
  - SNAT (Source NAT): used for outgoing traffic.
  - DNAT (Destination NAT): inbound port forwarding.
  - PAT (Port Address Translation): many-to-one using ports.

iptables example (IPv4 NAT for outgoing):
```bash
# Enable forwarding in sysctl
sudo sysctl -w net.ipv4.ip_forward=1

# Masquerade traffic from private network to public interface eth0
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Allow forwarding between interfaces (basic)
sudo iptables -A FORWARD -i eth1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
```

nftables example (simple NAT):
```bash
# Example: create table and NAT chain
sudo nft add table ip nat
sudo nft 'add chain ip nat postrouting { type nat hook postrouting priority 100; }'
sudo nft add rule ip nat postrouting oifname "eth0" masquerade
```

Port forwarding (DNAT) example (iptables):
```bash
# Forward external port 2222 to internal SSH 22 on host 192.168.1.10
sudo iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination 192.168.1.10:22
sudo iptables -A FORWARD -p tcp -d 192.168.1.10 --dport 22 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```

Routing protocols (when needed):
- BGP for Internet routing, OSPF/IS-IS for internal large networks — often used in data centers or routing appliances.

---

## 5. LAN Technologies & VLANs

VLANs (802.1Q)
- Tag traffic with VLAN ID to separate broadcast domains on same physical switch.
- Linux VLAN example:
```bash
# Create VLAN 10 on eth0
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip addr add 192.168.10.2/24 dev eth0.10
sudo ip link set up dev eth0.10
```

Bridging (common in VMs/containers)
- Create a Linux bridge:
```bash
sudo ip link add name br0 type bridge
sudo ip link set dev eth1 master br0
sudo ip addr add 192.168.5.2/24 dev br0
sudo ip link set up dev br0
```

Switch vs Router
- Switch: Layer 2, MAC learning.
- Router: Layer 3, IP routing between networks.

---

## 6. Linux Networking Commands — Quick Reference

View interfaces and addresses:
- `ip addr show`
- `ip link show`
- `ifconfig -a` (legacy)

Routing:
- `ip route show`
- `ip route add/del`

Connectivity & diagnostics:
- `ping 8.8.8.8`
- `ping -I eth0 -c 4 1.1.1.1` (use specific iface)
- `traceroute google.com`
- `tracepath google.com`
- `mtr -r -c 100 google.com` (interactive hybrid)
- `ss -tulnp` (socket/listening)
- `netstat -tulnp` (legacy)

Packet capture & inspection:
- `sudo tcpdump -i eth0 -n -s 0 -w capture.pcap`
- `sudo tcpdump -i eth0 host 10.0.0.5 and port 80 -vv`

Interface tools:
- `ethtool eth0` (link speed, auto-negotiation)
- `tc` (traffic shaping / qdisc)
- `ip -s link` (statistics)

Neighbour/ARP:
- `ip neigh show`
- `arp -an`

Firewall & NAT:
- `sudo iptables -L -n -v`
- `sudo iptables -t nat -L -n -v`
- `sudo nft list ruleset`

System services:
- `systemctl restart network` (RHEL)
- `sudo systemctl restart NetworkManager` (if used)
- `sudo systemctl restart systemd-networkd`

Logs:
- `sudo journalctl -u NetworkManager -e`
- `sudo journalctl -k` (kernel messages)

---

## 7. Network Configuration Examples

### Debian/Ubuntu (netplan)
Example: /etc/netplan/01-netcfg.yaml
```yaml
network:
  version: 2
  renderer: networkd    # or NetworkManager
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.50.10/24]
      gateway4: 192.168.50.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```
Apply:
```bash
sudo netplan apply
```

### RHEL / CentOS (ifcfg)
Example: /etc/sysconfig/network-scripts/ifcfg-eth0
```
DEVICE=eth0
BOOTPROTO=static
ONBOOT=yes
IPADDR=10.0.0.10
NETMASK=255.255.255.0
GATEWAY=10.0.0.1
DNS1=1.1.1.1
```
Restart:
```bash
sudo systemctl restart network
```

### systemd-networkd (example)
Drop a file in `/etc/systemd/network/10-eth0.network`:
```
[Match]
Name=eth0

[Network]
Address=192.168.100.10/24
Gateway=192.168.100.1
DNS=8.8.8.8
```
Enable:
```bash
sudo systemctl enable --now systemd-networkd
```

---

## 8. Firewalls & Network Security

Options:
- iptables (legacy), nftables (modern), firewalld (RHEL wrapper), UFW (Ubuntu-friendly).
- Principle of least privilege: only allow required traffic.

nftables basic accept ruleset:
```bash
sudo nft add table inet filter
sudo nft 'add chain inet filter input { type filter hook input priority 0; policy drop; }'
sudo nft add rule inet filter input ct state established,related accept
sudo nft add rule inet filter input iif "lo" accept
sudo nft add rule inet filter input tcp dport {22,80,443} ct state new accept
```

Security best practices:
- Close unused ports; use `ss -tulnp` to audit.
- Use SSH hardening: disable password auth, use keys, disable root login.
- Use host-based intrusion detection (e.g., AIDE), centralized logging, and frequent audits.
- Segmentation: use VLANs & subnets to restrict lateral movement.
- Monitor logs for port scans and repeated failures.

---

## 9. VPNs & Tunnels

WireGuard (quick server config)
- Install `wireguard` (kernel module + user tools).
- Key generation:
```bash
wg genkey | tee privatekey | wg pubkey > publickey
```
- Minimal server config `/etc/wireguard/wg0.conf`:
```
[Interface]
Address = 10.10.10.1/24
PrivateKey = <server_private_key>
ListenPort = 51820
PostUp = sysctl -w net.ipv4.ip_forward=1; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_pub_key>
AllowedIPs = 10.10.10.2/32
```
Start:
```bash
sudo systemctl enable --now wg-quick@wg0
```

OpenVPN / IPsec (Strongswan) — widely used for compatibility and site-to-site tunnels.

---

## 10. Containers & Kubernetes Networking

CNI (Container Network Interface)
- Plugins provide pod networking: Calico (policy), Flannel (simple overlay), Weave, Cilium (eBPF).
- Node NAT: kube-proxy can NAT service IPs to backend pods.

Container host networking examples:
- Bridge mode (default Docker): containers behind `docker0`.
- Host networking: `--network host` shares host network namespace.
- Port mapping: `docker run -p 8080:80 ...`

Kubernetes notes:
- Pods get IPs that route across nodes using CNI.
- Services: ClusterIP (internal), NodePort, LoadBalancer.
- Ingress controllers handle HTTP/S routing — often fronted by a cloud LB.

---

## 11. Cloud Networking (AWS/GCP/Azure) — highlights

Common concepts:
- VPC (Virtual Private Cloud) / VNet: isolated network.
- Subnet: IP range within VPC.
- Internet Gateway (IGW) — attach to VPC to allow public internet.
- NAT Gateway/Instance — allow private subnet egress to Internet (no inbound).
- Security Group — stateful instance-level firewall.
- Network ACL (NACL) — stateless subnet-level firewall.

AWS example: allow outbound-only traffic from private subnet
- Create private subnet without auto-assign public IPs.
- Route table points 0.0.0.0/0 to NAT Gateway in public subnet.
- Instances use security groups allowing required outbound ports.

GCP and Azure have analogous constructs (VPC/VNet, Cloud NAT / NAT Gateway, NSGs in Azure).

Cloud troubleshooting tips:
- Check route tables and NACLs in console.
- Verify Security Group rules allow the traffic.
- Check source/destination check on EC2 instances (disable for NAT instances).
- Use cloud provider diagnostics (VPC Flow Logs, CloudTrail).

---

## 12. Monitoring & Troubleshooting

Tools:
- tcpdump / Wireshark (capture/analysis)
- mtr / traceroute / tracepath
- ping / fping
- ss / netstat
- iperf3 (throughput testing)
- Prometheus + Grafana (monitoring)
- tc (traffic control) for shaping and diagnosing queuing

Troubleshooting workflow (step-by-step)
1. Physical & link layer:
   - Is cable connected? `ethtool eth0` shows link status.
2. Interface & addressing:
   - `ip addr show`, `ip link show`.
   - Is IP assigned? `ip route show`.
3. Local service checks:
   - Is service listening? `ss -tulnp`.
4. Connectivity basic:
   - `ping <gateway>` → `ping 8.8.8.8`.
5. DNS:
   - `dig example.com`, `cat /etc/resolv.conf`.
6. Routing & path:
   - `traceroute`, `mtr`.
7. Firewall:
   - `iptables -L -n -v`, `nft list ruleset`, cloud security groups.
8. Packet capture:
   - `sudo tcpdump -i eth0 host <ip> -w /tmp/cap.pcap`.
9. Check ARP / neighbour:
   - `ip neigh show`.
10. Check MTU issues:
   - Symptoms: SSL/TLS hang, large transfer fails — try lowering MTU on interface.
11. Check logs:
   - `journalctl`, application logs.
12. Replicate: run `iperf3` to measure bandwidth/latency.

Common failures & fixes:
- No IP — check DHCP server, `dhclient` logs.
- Intermittent packet loss — check NIC, switch logs, duplex mismatch (`ethtool`).
- DNS failures — point to 8.8.8.8 as a sanity check.

---

## 13. Performance & MTU

MTU (Maximum Transmission Unit)
- Default Ethernet MTU: 1500. Jumbo frames: 9000 (requires switch support).
- Path MTU Discovery can fail if ICMP blocked — causing large packet issues.
- Adjust MTU:
```bash
sudo ip link set dev eth0 mtu 1400
```

QoS / Traffic shaping
- Use `tc` for egress shaping, queuing disciplines (fq_codel, htb).
- Example: limit outbound to 5mbit:
```bash
sudo tc qdisc add dev eth0 root tbf rate 5mbit burst 32kbit latency 400ms
```

---

## 14. Quick Reference & Common Tasks

Change MAC (temporary)
```bash
sudo ip link set dev eth0 down
sudo ip link set dev eth0 address 00:11:22:33:44:55
sudo ip link set dev eth0 up
```

Flush IP addresses on an interface:
```bash
sudo ip addr flush dev eth0
```

Enable IP forwarding:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
# persist in /etc/sysctl.conf: net.ipv4.ip_forward=1
```

Check listening sockets and process:
```bash
sudo ss -ltnp
```

Capture HTTP traffic on port 80:
```bash
sudo tcpdump -i eth0 -n -s 0 -w http.pcap tcp port 80
```

Check MTU and fragmentation:
```bash
ping -M do -s 1472 8.8.8.8   # IPv4, adjust size for header overhead
```

---

## 15. Security Checklist for Networked Linux Hosts

- Harden SSH: use keys, change default port if necessary, enable fail2ban.
- Keep firewall rules minimal and audited.
- Ensure updated packages and kernel.
- Use network segmentation: separate management, app, DB networks.
- Use encrypted protocols (TLS) for services in transit.
- Monitor logs and enable alerting for suspicious traffic.
- Use centralized authentication (LDAP/AD) and strong password policies.

---

## 16. References & Further Reading

- RFCs: RFC 791 (IPv4), RFC 2460 (IPv6), RFC 1918 (private IPv4).
- iproute2 man pages: `man ip`, `man ss`, `man tc`
- nftables reference: https://wiki.nftables.org
- WireGuard: https://www.wireguard.com/
- Linux network troubleshooting guides (various distro docs).

---

## 17. Appendix — Sample Commands & One-liners

Show routes and next-hop:
```bash
ip route get 8.8.8.8
```

List interfaces with stats:
```bash
ip -s link
```

Find which process bound to port 5432:
```bash
ss -ltnp | grep 5432
```

Extract DNS servers used by systemd-resolved:
```bash
resolvectl status
```

Generate pcap of a HTTP exchange and inspect with Wireshark:
```bash
sudo tcpdump -i eth0 -n -s 0 port 80 -w /tmp/http.pcap
```

Throughput test (iperf3):
```bash
# Server
iperf3 -s

# Client
iperf3 -c server.example.com -P 4 -t 60
```

---

This README is intended to be practical and copy-paste friendly. Add or adapt sections to match your environment (data center, cloud provider, Kubernetes, or on-prem). If you'd like, I can:
- Produce a one-page printable cheatsheet,
- Generate examples for a specific distro (Ubuntu 22.04 / RHEL 9),
- Create iptables -> nftables migration examples,
- Draft an incident runbook for network outages.
