# 🔐 OriginBank — Secure Network Infrastructure

> Designing, configuring, and validating a hardened enterprise network from scratch using Cisco Packet Tracer — covering firewall zoning, VLANs, OSPF, ACLs, IPSec VPN, AAA, and IPS.

![Platform](https://img.shields.io/badge/Platform-Cisco%20Packet%20Tracer-blue?style=flat-square)
![Protocol](https://img.shields.io/badge/Routing-OSPF-green?style=flat-square)
![VPN](https://img.shields.io/badge/VPN-IPSec%20AES--256-orange?style=flat-square)
![Firewall](https://img.shields.io/badge/Firewall-Cisco%20ASA-red?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)

---

## The Problem

OriginBank's existing network had no meaningful security architecture. POP3 and ICMP were exposed on public-facing interfaces. There was no access control between zones, no encrypted management, no inter-site VPN, and no intrusion detection. Any attacker with basic network access could enumerate devices, pivot laterally, and exfiltrate data unchallenged.

This project redesigns the entire network from the ground up — segmented, hardened, monitored, and encrypted end-to-end.

---

## What Was Built

| Component | Technology |
|---|---|
| Network Simulation | Cisco Packet Tracer |
| Perimeter Security | Cisco ASA Firewall (zone-based) |
| Dynamic Routing | OSPF |
| Network Segmentation | VLANs + 802.1Q Trunking + Inter-VLAN Routing |
| Access Control | Extended ACLs (router + ASA) |
| Encrypted Management | SSH v2 + AAA (local) |
| Site-to-Site Encryption | IPSec VPN — IKEv1, AES-256, SHA-HMAC |
| Threat Detection | IOS IPS (signature-based) |
| Core Services | DHCP, DNS, Web (DMZ), Syslog/NTP |
| Address Translation | PAT/NAT on ASA |

---

## Network Architecture

The topology is divided into four security zones:

```
Internet
    │
[Cisco ASA Firewall]  ← perimeter enforcement
    ├── DMZ (Security Level 60)       192.168.0.0/26
    │     ├── Web Server              192.168.0.2
    │     ├── Exchange (Mail)         192.168.0.3
    │     └── Public DNS Server       192.168.0.4
    │
    ├── HQ LAN (Security Level 100)   192.168.0.64/26
    │     ├── Internal DNS/DHCP       192.168.0.66
    │     ├── Syslog/NTP Server       192.168.0.67
    │     └── Workstations (DHCP)     192.168.0.68–126
    │
    └── Branch LAN (via IPSec VPN)    192.168.0.128/25
          ├── Branch DNS/DHCP         192.168.0.130
          └── Workstations (DHCP)     192.168.0.131–254
```

The ASA enforces **Zero Trust between zones** — no traffic passes unless explicitly permitted.

---

## Walkthrough

The full implementation is broken into four detailed sections:

| # | Section | What's Covered |
|---|---------|----------------|
| 01 | [Architecture & IP Design](/01-architecture-and-ip-design.md) | Topology, IP allocation, DHCP, device hardening, DNS/Web/Syslog setup |
| 02 | [Routing & Segmentation](/02-routing-and-segmentation.md) | OSPF, VLAN trunking, inter-VLAN routing (Router-on-a-Stick) |
| 03 | [Firewall, ACLs & VPN](/03-firewall-acl-vpn.md) | ASA zone policy, ACL rules, PAT/NAT, SSH/AAA, site-to-site IPSec VPN |
| 04 | [IPS & Security Analysis](/04-ips-and-security-analysis.md) | IOS IPS deployment, Zero Trust analysis, IPSec cryptography deep-dive |

---

## Key Security Decisions

**Why OSPF over RIPv2?**
Faster convergence and better scalability for a multi-site topology. RIPv2's 30-second update cycle is a liability in a network where routing changes need to propagate immediately.

**Why host the Web Server in the DMZ?**
If the web server is compromised, the attacker is trapped in the DMZ. They cannot reach the HQ VLAN because the ASA blocks all DMZ → HQ traffic by default (lower security level cannot initiate connections to higher).

**Why IPSec over a simple GRE tunnel?**
GRE provides no encryption. All inter-site traffic between HQ and Branch carries sensitive banking data — AES-256 encryption with SHA-HMAC authentication is non-negotiable.

**Why replace Telnet with SSH + AAA?**
Telnet transmits credentials in plaintext. A single packet capture on the management VLAN exposes every device password. SSH v2 with RSA-1024 eliminates that attack vector entirely.

---

## Skills Demonstrated

- Enterprise network design and IP addressing strategy
- Cisco IOS configuration (routers, switches, ASA)
- Firewall zone policy and layered access control
- VPN tunnel design and cryptographic configuration
- Intrusion prevention and centralised log management
- Security framework analysis (NIST Zero Trust SP 800-207)

---

## Author

**Soriful Islam Shoaib** — Cybersecurity Engineer
[GitHub](https://github.com/) · [LinkedIn](https://linkedin.com/)

---

> *Simulation built in Cisco Packet Tracer. The `.pkt` file is available in this repository.*
