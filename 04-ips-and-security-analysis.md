# 04 — IPS & Security Analysis

[← Firewall, ACLs & VPN](03-firewall-acl-vpn.md) | [← Back to README](../README.md)

---

## Overview

ACLs and firewalls are reactive — they enforce rules based on what you already know to block. An Intrusion Prevention System (IPS) is proactive — it inspects traffic in real time and blocks known attack patterns before they reach their target.

This section covers:

1. **IOS IPS deployment** on the Border Router — what it does and how it's configured
2. **Zero Trust analysis** — where OriginBank's design aligns with NIST SP 800-207 and where the gaps are
3. **IPSec cryptography deep-dive** — the mechanics behind the VPN security

---

## IOS IPS — Intrusion Prevention System

The IPS sits on the Border Router — the first Cisco IOS device traffic hits after leaving the public internet. Positioning it here means threats are detected and dropped before they reach the ASA, the DMZ, or any internal resource.

### How IOS IPS Works

IPS compares incoming packets against a signature database. Each signature describes a known attack pattern — an ICMP flood, a port scan, a specific exploit payload. When a match is found, the IPS can:

- **Alert** — log the event to Syslog and continue forwarding
- **Drop** — silently discard the packet
- **Reset** — send a TCP RST to terminate the connection

For OriginBank, the default action for matched signatures is **drop + alert** — the attack is blocked and logged centrally.

### Step 1 — Create the IPS Signature Store Directory

```cisco
mkdir ipsdir
ip ips config location flash:ipsdir
```

This tells the router where to store IPS signature data.

### Step 2 — Create an IPS Rule

```cisco
ip ips name ORIGINBANK-IPS

! Apply SDEE event notification (for logging)
ip ips notify sdee
```

### Step 3 — Enable Key IPS Signatures

Not every signature needs to be enabled — enabling everything creates noise and can impact performance. The priority signatures for a banking network:

```cisco
ip ips signature-category
 category all
  retired true
 category ios_ips basic
  retired false
```

This retires all signatures by default, then activates only the curated `ios_ips basic` category — which covers the most critical, high-confidence threats without overwhelming the router's CPU.

### Step 4 — Apply IPS to Interfaces

```cisco
interface Serial0/1/0
 ip ips ORIGINBANK-IPS in
 ip ips ORIGINBANK-IPS out
```

Applying inbound and outbound ensures both incoming attacks from the internet and any anomalous outbound traffic (e.g., a compromised internal host trying to beacon out) are inspected.

### Step 5 — Configure Logging to Syslog

```cisco
logging 192.168.0.67
logging trap warnings
ip ips notify log
```

All IPS events are shipped to the central Syslog server in HQ. This creates a persistent audit trail.

---

## Testing the IPS

### ICMP Flood Test

Send a continuous ping from an external host against the DMZ web server:

```
ping 20.20.0.2 repeat 1000
```

The IPS should detect the flood pattern, drop the excess packets, and log an alert. Verify in Syslog:

```
%IPS-4-SIGNATURE: Sig:2004 Subsig:0 Sev:25 ICMP Echo Request [20.20.0.30:0 -> 20.20.0.2:0]
```

### Verify Active Signatures

```cisco
show ip ips all
```

Shows all enabled signatures and their current status. Look for the basic category signatures showing as active.

```cisco
show ip ips statistics
```

Shows counts of packets inspected, alerts triggered, and packets dropped.

---

## Zero Trust Analysis

NIST SP 800-207 defines Zero Trust as an architecture where **no entity is implicitly trusted** — every access request is verified regardless of where it originates, whether inside or outside the network perimeter.

OriginBank's design aligns with several Zero Trust principles, but not all of them.

### Where the Design Aligns

**Network micro-segmentation**
VLANs divide the network into distinct trust zones. Traffic between zones must pass through the ASA, which applies explicit policy. A compromised workstation in VLAN 20 cannot reach the Syslog server in VLAN 99 without traversing the router and hitting an ACL.

**Least-privilege access**
ACL rules are written with minimum necessary access. The auditor account has privilege 5 — enough to run `show` commands, not enough to make configuration changes. External users can only reach port 80 and 443 on the web server — nothing else.

**Assume breach**
The DMZ exists precisely because OriginBank assumes the web server *will* be compromised at some point. By placing it outside the HQ security level, a web server breach doesn't become a network breach.

**Encrypted communication**
All HQ ↔ Branch traffic is encrypted via IPSec. Management sessions use SSH v2. Plaintext protocols (Telnet, HTTP management) are disabled.

**Continuous monitoring**
Centralised Syslog + IPS provides visibility into traffic events across the network. Every router ships its logs to a single, protected server.

### Where the Gaps Are

| Gap | Current State | Recommended Improvement |
|-----|--------------|--------------------------|
| Authentication | Local user database | Centralised RADIUS/TACACS+ with MFA |
| Identity verification | Username/password only | Certificate-based auth or hardware tokens |
| Application-layer control | Network-layer ACLs only | ZTNA — access control per application, per user |
| Monitoring | Syslog + IPS alerts | SIEM (e.g. Splunk, Microsoft Sentinel) with automated response |
| Device trust | No device posture checking | Endpoint compliance validation before network access |

The current implementation is a strong **network-layer Zero Trust** baseline. Maturing toward a full Zero Trust architecture would require moving access control up the stack — from network zones to individual applications and user identities.

---

## IPSec Cryptography Deep-Dive

Understanding *what* was configured is one thing. Understanding *why it works* is what separates a network engineer from someone who just ran commands.

### IKE Phase 1 — Building the Management Tunnel

Before any data is encrypted, the two routers need to agree on cryptographic parameters and verify each other's identity. This happens in IKE Phase 1.

```
HQ Router                                Border/Branch Router
     |                                           |
     |──── ISAKMP Proposal (AES-256, SHA, DH5) ──→|
     |←─── Proposal Accepted ─────────────────────|
     |                                           |
     |──── DH Public Key ──────────────────────→|
     |←─── DH Public Key ──────────────────────|
     |                                           |
     |  [Both sides derive the same shared key   |
     |   from the DH exchange — without ever     |
     |   transmitting the key itself]            |
     |                                           |
     |──── Encrypted Identity + PSK hash ──────→|
     |←─── Encrypted Identity + PSK hash ──────|
     |                                           |
     |  [ISAKMP SA established]                  |
```

The pre-shared key (`ORIGINBANK_VPN_PSK`) never travels across the network. It's used to authenticate the identity proof — each side proves it knows the key by producing the correct hash. The actual encryption key is derived independently by both sides using the Diffie-Hellman exchange.

### IKE Phase 2 — Building the Data Tunnel

With Phase 1 protecting the management channel, Phase 2 negotiates the actual IPSec parameters:

```cisco
crypto ipsec transform-set TS-AES-SHA esp-aes 256 esp-sha-hmac
```

| Parameter | Value | What it does |
|-----------|-------|-------------|
| `esp-aes 256` | AES-256 | Encrypts the data payload |
| `esp-sha-hmac` | SHA-1 HMAC | Creates a message authentication code — verifies packet integrity |
| `mode tunnel` | Tunnel mode | Encrypts the entire original IP packet, including headers |

In **tunnel mode**, the original packet (headers + payload) is fully encapsulated inside a new ESP packet with new IP headers. Anyone intercepting the traffic on the WAN sees only the outer IP headers (HQ router IP → Branch router IP) and the encrypted ESP payload. The original source/destination IPs and all payload data are invisible.

### Perfect Forward Secrecy (PFS)

```cisco
set pfs group5
```

PFS ensures that even if an attacker eventually compromises the long-term pre-shared key, they cannot use it to decrypt previously captured sessions. Each Phase 2 session uses a fresh Diffie-Hellman exchange to derive an independent session key. Capturing and storing encrypted VPN traffic for later decryption (a "record now, decrypt later" attack) doesn't work when PFS is enabled.

### Why AES-256 and Not AES-128?

AES-128 is computationally secure against all known attacks. But AES-256 provides a larger security margin for long-lived keys and is the standard for government and financial sector compliance (including FIPS 140-2). For a banking network, the marginal performance cost of AES-256 over AES-128 is worth the additional assurance.

### Limitations of the Current VPN Design

| Limitation | Impact | Mitigation |
|-----------|--------|-----------|
| Pre-shared key authentication | If the PSK is leaked, both sites are compromised | Migrate to PKI certificate-based auth with IKEv2 |
| IKEv1 | Older protocol, more complex negotiation | Upgrade to IKEv2 (simpler, more secure, built-in EAP support) |
| SHA-1 HMAC | SHA-1 is considered weak for new deployments | Use SHA-256 or SHA-384 |
| Single VPN peer | No redundancy — WAN link failure drops the tunnel | Deploy dual ISP links with failover |

---

## What This Architecture Achieves

Pulling the full design together, OriginBank's network now provides:

| Security Goal | How It's Achieved |
|---|---|
| **Confidentiality** | IPSec AES-256 encrypts all inter-site traffic; SSH encrypts management sessions |
| **Integrity** | SHA-HMAC verifies packets aren't tampered with in transit |
| **Availability** | OSPF reconverges in seconds on path failure; Syslog enables rapid incident response |
| **Access Control** | ASA security levels + ACLs enforce least-privilege between every zone |
| **Auditability** | Centralised Syslog captures events from all devices; IPS logs attack attempts |
| **Non-repudiation** | AAA logs all authentication events with timestamps |

---

## What Would Come Next (Production Hardening)

This design is a strong foundation, but a production deployment would add:

- **IKEv2 with certificates** — eliminate the pre-shared key, use PKI instead
- **RADIUS/TACACS+ with MFA** — replace local AAA with centralised identity management
- **SIEM integration** — correlate Syslog events across all devices, automate alerting
- **AES-GCM transform set** — combines encryption and authentication in one operation, better performance
- **ZTNA overlay** — application-layer access control per user identity, not just per network zone
- **Redundant WAN links** — eliminate the single point of failure on inter-site connectivity

---

## Final Verification Checklist

- [ ] `show ip ips all` — basic signatures active and enabled
- [ ] ICMP flood test triggers IPS alert in Syslog
- [ ] `show ip ips statistics` — packets inspected counter incrementing
- [ ] Zero Trust gap analysis documented
- [ ] VPN traffic inspected in Packet Tracer — ESP header visible on WAN link
- [ ] `show crypto ipsec sa` — PFS enabled, SA lifetime set to 86400s

---

[← Firewall, ACLs & VPN](03-firewall-acl-vpn.md) | [← Back to README](../README.md)
