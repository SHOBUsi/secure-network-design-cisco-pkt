# 🔐 OriginBank Secure Network Infrastructure
### MSc Cybersecurity | LD7007 Network Security | Northumbria University London

[![Module](https://img.shields.io/badge/Module-LD7007%20Network%20Security-blue)](.)
[![Tools](https://img.shields.io/badge/Tool-Cisco%20Packet%20Tracer-green)](.)
[![Platform](https://img.shields.io/badge/Platform-Cisco%20IOS%20%7C%20ASA-orange)](.)
[![Status](https://img.shields.io/badge/Status-Completed-brightgreen)](.)

---

## 📋 Overview

This project presents a **full design, configuration, and validation of a secure network infrastructure** for a fictional banking organisation, *OriginBank*, using **Cisco Packet Tracer**. The original network was identified as vulnerable due to weak authentication practices, misconfigured protocols (POP3/ICMP on public-facing interfaces), and insufficient access controls.

The assignment demonstrates how to harden and transform a vulnerable enterprise network into a **secure, segmented, and policy-compliant infrastructure** aligned with the **CIA Triad** (Confidentiality, Integrity, Availability).

---

## 🏗️ Network Architecture

The topology consists of three main zones:

| Zone | Interface | Security Level | Subnet |
|------|-----------|---------------|--------|
| HQ (Internal) | Gig1/1 | 100 (Highest) | 192.168.0.64/26 |
| DMZ | Gig1/3 | 60 | 192.168.0.0/26 |
| Outside (Internet/Branch) | Gig1/2 | 0 (Lowest) | WAN |

The design includes HQ VLANs, a Branch network, a DMZ, and WAN segments — all connected through a **Cisco ASA firewall** and routers running **OSPF** for dynamic routing.

---

## 🛠️ Technologies & Tools

- **Cisco Packet Tracer** — Network simulation
- **Cisco IOS** — Router and switch configuration
- **Cisco ASA Firewall** — Perimeter security and zone-based policy enforcement
- **OSPF** — Dynamic routing protocol
- **IEEE 802.1Q** — VLAN trunking
- **IPSec VPN** — Site-to-site encrypted tunnel
- **SSH v2 + AAA (RADIUS/TACACS+)** — Secure device management
- **IOS IPS** — Intrusion Prevention System
- **DHCP, DNS, HTTP/HTTPS, SMTP, Syslog** — Core network services

---

## 📦 Project Structure

```
originbank-network-security/
│
├── Block_A_Architecture/
│   ├── IP_Allocation_and_Connectivity
│   ├── Device_Hardening
│   ├── DNS_Web_Syslog_Servers
│   └── OSPF_VLAN_InterVLAN_Routing
│
├── Block_B_Secure_Operations/
│   ├── ACL_Firewall_Rules
│   ├── SSH_and_AAA_Configuration
│   ├── Site_to_Site_IPSec_VPN
│   └── IOS_IPS_Configuration
│
├── Block_C_Research/
│   ├── Zero_Trust_Network_Framework
│   ├── VPN_Reliability_Analysis
│   └── IPSec_Cryptographic_Mechanisms
│
└── README.md
```

---

## ✅ Key Implementations

### Block A — Architecture & Communication

- **IP Addressing**: Structured static IP allocation for servers; DHCP pools for clients across HQ and Branch VLANs
- **Device Hardening**: Strong authentication, secure remote management (SSH only), disabled unused ports, legal warning banners, and port security on all routers and switches
- **Core Services**:
  - DNS servers deployed per subnet for internal/external name resolution
  - Web server hosted in the DMZ for controlled public access
  - Syslog server in HQ for centralised log collection from all infrastructure devices
- **Dynamic Routing**: OSPF deployed across all routers (chosen over RIPv2 for faster convergence and scalability)
- **VLAN Trunking & Inter-VLAN Routing**: 802.1Q trunking configured on switches; Router-on-a-Stick with sub-interfaces for inter-VLAN communication

### Block B — Secure Operations & Service Delivery

- **ACLs on ASA Firewall**: Zone-based rules enforcing Zero Trust — DMZ cannot reach internal HQ; outside users can only reach the Web and DNS servers in the DMZ
- **PAT/NAT**: Configured on ASA for DMZ servers to reach the internet using the public-facing IP
- **SSH v2 + AAA**: Telnet replaced with SSH v2; AAA authentication enforced on console and VTY lines; RSA 1024-bit key pairs generated per device
- **Site-to-Site IPSec VPN**: Encrypted tunnel established between HQ and Branch networks using IKEv1 with AES encryption and SHA hashing
- **IOS IPS**: Signature-based intrusion prevention deployed on the HQ router to monitor and block anomalous traffic

### Block C — Research & Development

- **Zero Trust Framework**: Analysis of micro-segmentation, least-privilege access, and continuous verification applied to OriginBank's infrastructure
- **VPN Reliability**: Examination of IPSec tunnel reliability, failover considerations, and protocol limitations
- **IPSec Cryptographic Mechanisms**: Key exchange (IKE Phase 1 & 2), encryption (AES), and integrity verification (HMAC-SHA)

---

## 🔒 Security Policy Highlights

| Policy | Implementation |
|--------|---------------|
| Zero Trust — DMZ to Internal | ASA default deny (lower → higher security level) |
| External access to DMZ | HTTP (80), HTTPS (443), DNS (53) only |
| HQ to DMZ | HTTP, HTTPS, SMTP, POP3, DNS, SSH (admin only), ICMP (admin only) |
| HQ to Outside | HTTP, HTTPS, DNS, ICMP; blocks P2P (6881-6889), SQL (1433), RDP (3389) |
| Management Access | SSH v2 only; Telnet disabled on all devices |
| Encryption | IPSec AES tunnel between HQ and Branch |

---

## 📊 Verification & Testing

Each configuration was validated with:
- `ping` and `traceroute` between zones to confirm ACL enforcement
- `show ip ospf neighbor` to verify dynamic routing adjacency
- `show vlan brief` and `show interfaces trunk` for VLAN verification
- HTTP/DNS connectivity tests from external clients through public IP (NAT)
- IPS signature trigger tests to confirm threat detection
- SSH session establishment from the admin host

---

## 💡 Key Learnings

- How **layered defence** (device hardening + ACL + firewall + VPN + IPS) dramatically reduces attack surface
- The role of **DMZ architecture** in isolating public-facing services from internal networks
- Trade-offs between RIPv2 and **OSPF** in enterprise routing scenarios
- Real-world application of the **Zero Trust** model in network design
- How **IPSec IKE Phase 1 and Phase 2** establish and maintain a secure VPN tunnel

---

## 📚 References

- Cisco Systems. (n.d.). *Cisco ASA Series Firewall CLI Configuration Guide*
- NIST SP 800-77 Rev.1 — Guide to IPsec VPNs
- Shah, S. (2020). *DMZ Network Architecture Best Practices*
- Sheehan, M. (2025). *VLAN Trunking and 802.1Q Encapsulation*

---

## 👤 Author

**Soriful Islam Shoaib**
MSc Cybersecurity — Northumbria University, London Campus
Student ID: 24053223
Module: LD7007 Network Security

---

> *This project was completed as part of an individual coursework assignment. The Cisco Packet Tracer simulation file is available upon request.*
