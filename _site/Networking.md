# 🌐 Networking — DevOps Interview Guide

> [← Linux](./Linux.md) | [Main Index](./README.md) | [Git →](./Git.md)

---

## Table of Contents

1. [OSI Model](#osi-model)
2. [TCP/IP Suite](#tcpip-suite)
3. [IP Addressing & Subnetting](#ip-addressing--subnetting)
4. [DNS Deep Dive](#dns-deep-dive)
5. [HTTP/HTTPS](#httphttps)
6. [Load Balancing](#load-balancing)
7. [Firewalls & Security Groups](#firewalls--security-groups)
8. [VPN & Tunneling](#vpn--tunneling)
9. [Network Troubleshooting](#network-troubleshooting)
10. [Interview Questions](#interview-questions)
11. [Scenario-Based Questions](#scenario-based-questions)
12. [Hands-On Labs](#hands-on-labs)

---

## OSI Model

```
Layer 7 — Application   → HTTP, HTTPS, DNS, FTP, SMTP
Layer 6 — Presentation  → SSL/TLS, encoding, compression
Layer 5 — Session       → Session management, NetBIOS
Layer 4 — Transport     → TCP, UDP, ports, segmentation
Layer 3 — Network       → IP, ICMP, routing, packets
Layer 2 — Data Link     → MAC addresses, Ethernet, switches, frames
Layer 1 — Physical      → Cables, fiber, signals, bits
```

### Why DevOps Engineers Need OSI Knowledge

| Layer | DevOps Relevance |
|-------|-----------------|
| L7 | Debugging API failures, reverse proxies (nginx), WAFs |
| L4 | Load balancer health checks, TCP timeouts, port conflicts |
| L3 | Routing between VPCs, subnet design, cloud networking |
| L2 | Container networking (bridge/overlay), VLAN segmentation |

---

## TCP/IP Suite

### TCP Three-Way Handshake

```
Client                    Server
  │                          │
  │──── SYN (seq=x) ────────>│
  │                          │
  │<─── SYN-ACK (seq=y,     │
  │          ack=x+1) ───────│
  │                          │
  │──── ACK (ack=y+1) ──────>│
  │                          │
  │ ── Connection Established ──
```

### TCP vs UDP Comparison

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented | Connectionless |
| Reliability | Guaranteed delivery, ordering | No guarantee |
| Speed | Slower (overhead) | Faster |
| Use cases | HTTP, SSH, FTP, databases | DNS, video streaming, gaming |
| Header size | 20-60 bytes | 8 bytes |
| Flow control | Yes (sliding window) | No |
| Congestion control | Yes | No |

### TCP States

```
CLOSED → LISTEN → SYN_RCVD → ESTABLISHED → FIN_WAIT_1 →
FIN_WAIT_2 → TIME_WAIT → CLOSED

# Check TCP connections by state
ss -an | awk '{print $1}' | sort | uniq -c | sort -rn

# TIME_WAIT: normal, connection recently closed, waiting 2*MSL
# CLOSE_WAIT: application hasn't closed its end (potential bug)
# SYN_RECV: many = potential SYN flood attack
```

---

## IP Addressing & Subnetting

### IPv4 Classes

| Class | Range | Default Subnet | Use |
|-------|-------|----------------|-----|
| A | 1.0.0.0 – 126.255.255.255 | /8 (255.0.0.0) | Large networks |
| B | 128.0.0.0 – 191.255.255.255 | /16 | Medium networks |
| C | 192.0.0.0 – 223.255.255.255 | /24 | Small networks |
| D | 224.0.0.0 – 239.255.255.255 | N/A | Multicast |
| E | 240.0.0.0 – 255.255.255.255 | N/A | Reserved |

### Private IP Ranges (RFC 1918)

| Range | CIDR | Addresses |
|-------|------|-----------|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | ~16.7M |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | ~1M |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | 65,536 |

### CIDR Subnet Quick Reference

| CIDR | Subnet Mask | Hosts | Typical Use |
|------|-------------|-------|-------------|
| /32 | 255.255.255.255 | 1 | Single host |
| /30 | 255.255.255.252 | 2 | Point-to-point links |
| /29 | 255.255.255.248 | 6 | Small teams |
| /28 | 255.255.255.240 | 14 | Small subnets |
| /27 | 255.255.255.224 | 30 | Small subnets |
| /24 | 255.255.255.0 | 254 | Standard LAN |
| /22 | 255.255.252.0 | 1022 | Medium network |
| /16 | 255.255.0.0 | 65,534 | Large VPC |
| /8 | 255.0.0.0 | 16,777,214 | Huge network |

### Subnetting Example

```
Network: 192.168.10.0/24 — Need 4 equal subnets

Borrowing 2 bits: /24 + 2 = /26
Each subnet: 64 addresses, 62 usable

Subnet 1: 192.168.10.0/26   → hosts: .1 to .62,  broadcast: .63
Subnet 2: 192.168.10.64/26  → hosts: .65 to .126, broadcast: .127
Subnet 3: 192.168.10.128/26 → hosts: .129 to .190, broadcast: .191
Subnet 4: 192.168.10.192/26 → hosts: .193 to .254, broadcast: .255
```

---

## DNS Deep Dive

### DNS Resolution Process

```
Browser                Recursive Resolver     Root NS    TLD NS    Auth NS
  │                           │                  │          │         │
  │── query: example.com ────>│                  │          │         │
  │                           │── root query ──>│          │         │
  │                           │<── .com NS ──────│          │         │
  │                           │── .com query ──────────────>│         │
  │                           │<── example.com NS ──────────│         │
  │                           │── example.com query ─────────────────>│
  │                           │<── A: 93.184.216.34 ─────────────────│
  │<── A: 93.184.216.34 ──────│
```

### DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| A | IPv4 address | `example.com → 93.184.216.34` |
| AAAA | IPv6 address | `example.com → 2606:2800::` |
| CNAME | Alias to another name | `www → example.com` |
| MX | Mail server | `@ → mail.example.com (priority 10)` |
| TXT | Text data (SPF, DKIM, verification) | `"v=spf1 include:..."` |
| NS | Authoritative name servers | `@ → ns1.example.com` |
| PTR | Reverse DNS (IP → name) | `34.216.184.93.in-addr.arpa → example.com` |
| SRV | Service location | `_http._tcp.example.com` |
| SOA | Zone authority info | Start of Authority |
| CAA | CA authorization | Controls who can issue TLS certs |

### DNS Troubleshooting Commands

```bash
# Basic lookup
dig example.com
dig example.com A        # Specific record type
dig example.com MX       # Mail records
dig @8.8.8.8 example.com # Query specific DNS server
dig +short example.com   # IP only

# Trace full resolution
dig +trace example.com

# Reverse lookup
dig -x 93.184.216.34

# Check DNS propagation
dig @ns1.example.com example.com  # Query authoritative server
nslookup example.com 8.8.8.8      # Using Google DNS

# TTL info
dig example.com | grep -i ttl
```

---

## HTTP/HTTPS

### HTTP Request/Response

```
GET /api/users HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...
Content-Type: application/json
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json
X-Request-ID: abc123
Cache-Control: no-cache

{"users": [...]}
```

### HTTP Status Codes

| Range | Category | Examples |
|-------|----------|---------|
| 1xx | Informational | 100 Continue, 101 Switching Protocols |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Permanent, 302 Temporary, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Server Error | 500 Internal Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

### TLS Handshake (simplified)

```
Client                              Server
  │                                    │
  │──── ClientHello (TLS version, ────>│
  │      cipher suites, random) ────── │
  │                                    │
  │<─── ServerHello (chosen cipher,    │
  │      Certificate, random) ─────────│
  │                                    │
  │── Verify cert chain                │
  │── Generate pre-master secret       │
  │──── ClientKeyExchange ────────────>│
  │──── ChangeCipherSpec ─────────────>│
  │──── Finished (encrypted) ─────────>│
  │                                    │
  │<─── ChangeCipherSpec ──────────────│
  │<─── Finished (encrypted) ──────────│
  │                                    │
  │ ═══════ Encrypted HTTP ═══════════ │
```

### HTTP/2 vs HTTP/1.1

| Feature | HTTP/1.1 | HTTP/2 |
|---------|----------|--------|
| Multiplexing | No (1 req/connection) | Yes (multiple streams) |
| Header compression | No | Yes (HPACK) |
| Server push | No | Yes |
| Binary protocol | No (text) | Yes |
| Connection | Multiple connections needed | Single connection |

---

## Load Balancing

### Load Balancing Algorithms

| Algorithm | Description | Best For |
|-----------|-------------|---------|
| Round Robin | Requests distributed sequentially | Equal capacity servers |
| Weighted Round Robin | Servers get % of traffic by weight | Different capacity servers |
| Least Connections | Route to server with fewest active connections | Long-lived connections |
| IP Hash | Client IP determines server (sticky) | Sessions without shared state |
| Random | Randomly select server | Simple, low overhead |
| Least Response Time | Fastest responding server | Latency-sensitive apps |

### Layer 4 vs Layer 7 Load Balancing

| Aspect | L4 (Transport) | L7 (Application) |
|--------|----------------|-----------------|
| Operates on | TCP/UDP | HTTP/HTTPS |
| Routing basis | IP + port | URL, headers, cookies |
| Performance | Higher (less inspection) | Lower (more work) |
| SSL termination | No | Yes |
| Content-based routing | No | Yes |
| Examples | AWS NLB, HAProxy (TCP mode) | nginx, AWS ALB, HAProxy (HTTP) |

### Nginx Load Balancer Config

```nginx
upstream backend {
    least_conn;
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080 weight=1;
    server 10.0.0.3:8080 backup;    # Standby
    keepalive 32;                   # Connection pool
}

server {
    listen 80;
    
    location /api/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }
    
    # Health check
    location /health {
        return 200 "healthy\n";
    }
}
```

---

## Firewalls & Security Groups

### iptables

```bash
# View rules
iptables -L -n -v --line-numbers

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT

# Allow HTTP/HTTPS
iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT

# Block an IP
iptables -A INPUT -s 1.2.3.4 -j DROP

# Rate limiting (DDoS protection)
iptables -A INPUT -p tcp --dport 80 -m limit --limit 100/min \
  --limit-burst 200 -j ACCEPT

# NAT (masquerade for outbound traffic)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Save rules
iptables-save > /etc/iptables/rules.v4
```

### Security Groups vs NACLs (AWS)

| Feature | Security Group | Network ACL |
|---------|---------------|-------------|
| Level | Instance (ENI) | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules evaluated | Rules evaluated in order |
| Default | Deny all inbound | Allow all |

---

## VPN & Tunneling

### VPN Types

| Type | Use Case | Protocol |
|------|----------|----------|
| Site-to-Site | Connect two networks | IPSec, GRE |
| Client-to-Site | Remote access | OpenVPN, WireGuard, L2TP |
| SSL VPN | Browser-based | TLS |
| SD-WAN | Enterprise WAN | Proprietary + IPSec |

### SSH Tunneling

```bash
# Local port forwarding (access remote service locally)
# Access remote DB at 5432 via local port 5433
ssh -L 5433:db-server:5432 jump-host -N -f

# Remote port forwarding (expose local service remotely)
ssh -R 8080:localhost:3000 remote-server -N -f

# Dynamic (SOCKS proxy)
ssh -D 1080 remote-server -N -f
# Then configure browser to use SOCKS5 proxy on 127.0.0.1:1080

# Jump host (ProxyJump)
ssh -J jump-host target-host
# Or in ~/.ssh/config:
# Host target
#   ProxyJump jump-host
```

---

## Network Troubleshooting

### Systematic Approach

```
Problem: Can't reach service at http://app.example.com:8080

Step 1 — DNS
  dig app.example.com → resolves to 10.0.1.50?

Step 2 — ICMP reachability
  ping 10.0.1.50 → responds?

Step 3 — Port reachability
  nc -zv 10.0.1.50 8080 → connection refused or success?
  telnet 10.0.1.50 8080

Step 4 — Routing
  traceroute 10.0.1.50 → where does it stop?

Step 5 — Firewall
  iptables -L, security groups, NACLs

Step 6 — Service listening
  ss -tulnp | grep 8080 → is something listening?

Step 7 — Application logs
  journalctl -u myapp -f
```

### Packet Analysis with tcpdump

```bash
# Capture all traffic on eth0
tcpdump -i eth0

# Capture to file (analyze with Wireshark)
tcpdump -i eth0 -w /tmp/capture.pcap

# Filter by host and port
tcpdump -i eth0 host 10.0.0.1 and port 80

# HTTP requests only
tcpdump -i eth0 -A -s0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

# DNS queries
tcpdump -i eth0 port 53

# SYN packets only (new connections)
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between a switch and a router?**

> A **switch** operates at Layer 2 (Data Link), forwarding frames based on MAC addresses within a single network segment (LAN). A **router** operates at Layer 3 (Network), forwarding packets between different networks based on IP addresses. In cloud environments, virtual switches and routers are implemented in software (AWS VPC, Azure VNet).

**Q2: What is NAT and why is it used?**

> **Network Address Translation (NAT)** maps private IP addresses to a public IP for outbound internet traffic. Used because IPv4 addresses are scarce — millions of private devices share a handful of public IPs. In AWS: Internet Gateway + route tables perform NAT for public subnets; NAT Gateway for private subnets to reach the internet without being directly reachable.

**Q3: What is a subnet and why do we use them?**

> A **subnet** is a logical subdivision of an IP network. We use them to:
> - **Organize** resources by function (web, app, DB tiers)
> - **Isolate** resources for security (DB in private subnet)
> - **Control** traffic with routing and firewall rules per subnet
> - **Reduce** broadcast domain size

**Q4: Explain what happens when you type a URL in a browser.**

> 1. **DNS resolution**: Browser checks cache, OS cache, recursive resolver → gets IP
> 2. **TCP connection**: 3-way handshake to port 80/443
> 3. **TLS handshake** (HTTPS): certificate exchange, session key
> 4. **HTTP request**: GET / HTTP/1.1 with headers
> 5. **Server processing**: web server, app server, database
> 6. **HTTP response**: HTML content returned
> 7. **Browser rendering**: parse HTML, CSS, JS; fetch additional resources
> 8. **Page displayed**

---

### 🟡 Intermediate

**Q5: How does BGP work and where is it used in DevOps?**

> **BGP (Border Gateway Protocol)** is the routing protocol of the internet — it exchanges routing information between Autonomous Systems (AS). In DevOps: AWS uses BGP for Direct Connect and Transit Gateways; Kubernetes uses BGP (via Calico/MetalLB) for advertising Pod CIDRs to physical routers; multi-cloud SD-WAN solutions use BGP for routing.

**Q6: Explain VXLAN and why containers use it.**

> **VXLAN (Virtual Extensible LAN)** encapsulates Layer 2 frames inside UDP packets (port 4789). It extends Layer 2 networks over Layer 3 infrastructure. Kubernetes overlay networks (Flannel, Calico overlay mode) use VXLAN to allow pods on different nodes to communicate as if on the same LAN, encapsulating pod traffic in UDP/IP packets.

**Q7: What is the difference between a reverse proxy and a forward proxy?**

> A **forward proxy** sits in front of clients — clients send requests through it to the internet (used for filtering, caching, anonymity). A **reverse proxy** sits in front of servers — clients talk to it, it routes to backend servers (used for load balancing, SSL termination, caching, hiding internal topology). In DevOps: nginx, Traefik, HAProxy are reverse proxies.

**Q8: Explain anycast and how CloudFlare/CDNs use it.**

> In **anycast**, the same IP address is announced from multiple geographic locations simultaneously. BGP routing delivers packets to the nearest (topologically) location. CDNs use this so `1.1.1.1` (CloudFlare DNS) routes you to the nearest CloudFlare datacenter automatically — no DNS-based load balancing needed.

---

### 🔴 Advanced

**Q9: How does Kubernetes networking work end-to-end?**

> Every Pod gets a unique IP. CNI plugin (Calico, Flannel, WeaveNet) is responsible for:
> 1. Allocating IP from the pod CIDR
> 2. Setting up veth pair (pod ↔ node bridge)
> 3. Programming routes/overlay so pods can reach each other across nodes
> 4. kube-proxy programs iptables/IPVS rules for Service IP → Pod IP translation
> 5. NodePort opens port on all nodes → forwards to ClusterIP → Pod
> 6. Ingress controller is a reverse proxy managing L7 routing

**Q10: Describe a CDN and how you'd configure cache invalidation.**

> A **CDN** (Content Delivery Network) caches content at edge nodes globally, reducing latency and origin load. Cache control headers: `Cache-Control: max-age=86400` (1 day). Invalidation strategies:
> - **TTL-based**: short TTL for dynamic, long TTL for static
> - **Versioned URLs**: `app.js?v=abc123` — new version = new cache key
> - **API invalidation**: CloudFront `create-invalidation`, Fastly Purge API
> - **Surrogate keys**: tag content, purge by tag (Fastly, Varnish)

---

## Scenario-Based Questions

### 🔵 Scenario 1: Connection Refused

*"Users report 'Connection refused' on port 443. How do you troubleshoot?"*

```bash
# 1. Is the service running?
systemctl status nginx
ss -tulnp | grep :443

# 2. Is the port open in firewall?
iptables -L INPUT -n | grep 443
# AWS: check security group and NACL

# 3. Is the certificate valid?
openssl s_client -connect yoursite.com:443 -servername yoursite.com

# 4. Test from different network (rule out local issue)
curl -v https://yoursite.com from another server

# 5. Check nginx logs
tail -50 /var/log/nginx/error.log

# Common causes:
# - Service crashed: restart + investigate root cause
# - Certificate expired: renew cert
# - Firewall rule blocking 443: add allow rule
# - Listening on 127.0.0.1 only: change to 0.0.0.0
```

### 🔵 Scenario 2: Intermittent Packet Loss

*"Users report intermittent connectivity to the application. Monitoring shows sporadic packet loss."*

```bash
# 1. Confirm and measure packet loss
ping -c 100 target_host | tail -5
mtr -n -r -c 100 target_host   # traceroute + ping combined

# 2. Check interface errors
ip -s link show eth0
ethtool -S eth0 | grep error

# 3. Check kernel ring buffer for NIC errors
dmesg | grep -i "eth0\|nic\|transmit\|receive"

# 4. Check for duplex mismatch
ethtool eth0 | grep Duplex

# 5. Cloud-specific: check if instance is being throttled
# AWS: CloudWatch NetworkIn/Out, check for burst limits

# 6. MTU issues (black hole routing)
ping -M do -s 1472 target_host  # Don't fragment, size=1472
```

---

## Hands-On Labs

### Lab 1: Subnet Calculator Script

```bash
#!/bin/bash
# Simple subnet calculator
# Usage: ./subnet.sh 192.168.1.0/24

CIDR="$1"
IP="${CIDR%/*}"
PREFIX="${CIDR#*/}"

# Convert IP to integer
ip_to_int() {
    IFS='.' read -r a b c d <<< "$1"
    echo $(( (a << 24) + (b << 16) + (c << 8) + d ))
}

int_to_ip() {
    echo "$(( ($1 >> 24) & 255 )).$(( ($1 >> 16) & 255 )).$(( ($1 >> 8) & 255 )).$(( $1 & 255 ))"
}

ip_int=$(ip_to_int "$IP")
mask=$(( 0xFFFFFFFF << (32 - PREFIX) & 0xFFFFFFFF ))
network=$(( ip_int & mask ))
broadcast=$(( network | (~mask & 0xFFFFFFFF) ))
hosts=$(( broadcast - network - 1 ))

echo "Network:   $(int_to_ip $network)/$PREFIX"
echo "Broadcast: $(int_to_ip $broadcast)"
echo "First Host:$(int_to_ip $((network + 1)))"
echo "Last Host: $(int_to_ip $((broadcast - 1)))"
echo "Hosts:     $hosts"
```

### Lab 2: Port Scanner Script

```bash
#!/bin/bash
# Basic port scanner
HOST="$1"
START="${2:-1}"
END="${3:-1024}"

echo "Scanning $HOST ports $START-$END"
for port in $(seq "$START" "$END"); do
    (echo >/dev/tcp/"$HOST"/"$port") 2>/dev/null && \
        echo "Port $port: OPEN"
done
```

### Lab 3: DNS Checker

```bash
#!/bin/bash
# Check DNS propagation across multiple servers
DOMAIN="$1"
DNS_SERVERS=("8.8.8.8" "1.1.1.1" "208.67.222.222" "9.9.9.9")

for ns in "${DNS_SERVERS[@]}"; do
    result=$(dig +short "@$ns" "$DOMAIN" 2>/dev/null)
    printf "%-20s %s\n" "$ns" "${result:-FAILED}"
done
```

---

> **Cross-links:** [← Linux](./Linux.md) | [Git →](./Git.md) | [Docker networking →](./Docker.md) | [Kubernetes networking →](./Kubernetes.md) | [AWS VPC →](./AWS.md)
