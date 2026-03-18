# 03 — Firewall, ACLs & VPN

[← Routing & Segmentation](02-routing-and-segmentation.md) | [Next: IPS & Security Analysis →](04-ips-and-security-analysis.md)

---

## Overview

This is where the security posture becomes real. Routing makes traffic flow — firewalls and ACLs decide what's *allowed* to flow. This section covers:

1. **ASA zone-based firewall policy** — enforcing Zero Trust between DMZ, HQ, and the outside world
2. **ACL rules** — granular traffic control on each zone boundary
3. **PAT/NAT** — allowing internal hosts to reach the internet without exposing private addresses
4. **SSH v2 + AAA** — locking down management access
5. **Site-to-site IPSec VPN** — encrypting all HQ ↔ Branch traffic

---

## Cisco ASA — Zone-Based Firewall

The ASA is the perimeter enforcement point. It operates on security levels — higher levels trust lower levels less. Traffic from a lower security level to a higher one is **blocked by default** unless explicitly permitted.

| Interface | Zone | Security Level | Network |
|-----------|------|---------------|---------|
| Gig1/1 | HQ (Inside) | 100 | 192.168.0.64/26 |
| Gig1/3 | DMZ | 60 | 192.168.0.0/26 |
| Gig1/2 | Outside | 0 | Internet / WAN |

This means:
- **HQ → DMZ**: Permitted by default (100 → 60), but we restrict it further with ACLs
- **DMZ → HQ**: **Blocked by default** (60 → 100) — no explicit config needed
- **Outside → anything**: **Blocked by default** (0 → 60/100) — must be explicitly permitted

### Basic ASA Interface Configuration

```cisco
interface GigabitEthernet1/1
 nameif HQ
 security-level 100
 ip address 192.168.0.65 255.255.255.192
 no shutdown

interface GigabitEthernet1/3
 nameif DMZ
 security-level 60
 ip address 192.168.0.1 255.255.255.192
 no shutdown

interface GigabitEthernet1/2
 nameif OUTSIDE
 security-level 0
 ip address 20.20.0.2 255.255.255.252
 no shutdown
```

---

## Access Control Lists

ACLs are applied **inbound on each interface** — they filter traffic as it enters that zone. The philosophy here is **default deny**: everything is blocked unless there's an explicit permit rule for it.

### HQ → DMZ

HQ users need access to web, mail, and DNS services in the DMZ. The network admin needs SSH access for device management.

```cisco
access-list HQ-DMZ extended permit tcp 192.168.0.64 255.255.255.192 host 192.168.0.2 eq 80
access-list HQ-DMZ extended permit tcp 192.168.0.64 255.255.255.192 host 192.168.0.2 eq 443
access-list HQ-DMZ extended permit tcp 192.168.0.64 255.255.255.192 host 192.168.0.3 eq 25
access-list HQ-DMZ extended permit tcp 192.168.0.64 255.255.255.192 host 192.168.0.3 eq 110
access-list HQ-DMZ extended permit udp 192.168.0.64 255.255.255.192 host 192.168.0.4 eq 53
access-list HQ-DMZ extended permit tcp host 192.168.0.68 192.168.0.0 255.255.255.192 eq 22
access-list HQ-DMZ extended permit icmp host 192.168.0.68 any
access-list HQ-DMZ extended deny ip any any

access-group HQ-DMZ in interface HQ
```

| Rule | Traffic | Reason |
|------|---------|--------|
| HTTP/HTTPS to Web Server | HQ → DMZ:80,443 | Staff browse internal web resources |
| SMTP/POP3 to Mail Server | HQ → DMZ:25,110 | Email send/receive |
| DNS | HQ → DMZ:53 | Public domain resolution |
| SSH | Admin only → DMZ:22 | Management access to DMZ devices |
| ICMP | Admin only | Network diagnostics |
| Default deny | Everything else | Zero Trust baseline |

### Outside → DMZ

The outside world should only reach the web server and DNS server. Nothing else.

```cisco
access-list OUT-DMZ extended permit tcp any host 192.168.0.2 eq 80
access-list OUT-DMZ extended permit tcp any host 192.168.0.2 eq 443
access-list OUT-DMZ extended permit udp any host 192.168.0.4 eq 53
access-list OUT-DMZ extended deny ip any any

access-group OUT-DMZ in interface OUTSIDE
```

### HQ → Outside

HQ workstations need web and DNS access. Dangerous ports (P2P, RDP, SQL) are explicitly blocked.

```cisco
access-list HQ-OUT extended permit tcp 192.168.0.64 0.0.0.63 any eq 80
access-list HQ-OUT extended permit tcp 192.168.0.64 0.0.0.63 any eq 443
access-list HQ-OUT extended permit udp 192.168.0.64 0.0.0.63 any eq 53
access-list HQ-OUT extended permit icmp 192.168.0.64 0.0.0.63 any
access-list HQ-OUT extended deny tcp 192.168.0.64 0.0.0.63 any range 6881 6889
access-list HQ-OUT extended deny tcp 192.168.0.64 0.0.0.63 any eq 3389
access-list HQ-OUT extended deny tcp 192.168.0.64 0.0.0.63 any eq 1433
access-list HQ-OUT extended deny ip any any

access-group HQ-OUT in interface HQ
```

> Blocking RDP (3389) outbound prevents employees from tunnelling out to external remote desktops — a common data exfiltration path. Blocking BitTorrent ports (6881-6889) and SQL (1433) removes unnecessary risk.

---

## PAT/NAT — Address Translation

The DMZ web server uses a private IP (192.168.0.2), but external users reach it via the ASA's public IP. NAT maps incoming connections on the public IP to the private server address. For outbound traffic, PAT translates all internal addresses to the single public IP.

### Static NAT — Web Server

```cisco
object network WEB-SERVER
 host 192.168.0.2
 nat (DMZ,OUTSIDE) static 20.20.0.2
```

This maps the ASA's public IP to the web server — external users browse to `20.20.0.2` and the ASA forwards them to `192.168.0.2`.

### Dynamic PAT — DMZ outbound

```cisco
object network DMZ-SUBNET
 subnet 192.168.0.0 255.255.255.192
 nat (DMZ,OUTSIDE) dynamic interface
```

All DMZ traffic going outbound is translated to the ASA's outside interface IP. Responses return correctly because the ASA maintains a translation table.

---

## SSH v2 + AAA — Secure Management Access

Telnet was removed from every device. SSH v2 with AAA-controlled authentication replaced it. The configuration below was applied to all routers and switches.

### SSH Configuration

```cisco
hostname HQ-Router
ip domain-name hq.originbank.local

! Generate RSA keys — minimum 1024 bits for SSH v2
crypto key generate rsa
! When prompted: How many bits in the modulus? Enter: 1024

ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
```

### AAA Configuration

```cisco
aaa new-model

! Use local user database for authentication
aaa authentication login default local
aaa authentication enable default enable

! Create user accounts with privilege levels
username admin privilege 15 secret StrongAdminP@ss
username auditor privilege 5 secret AuditorP@ss

! Apply to console and VTY lines
line console 0
 login authentication default
 exec-timeout 10 0

line vty 0 4
 login authentication default
 transport input ssh
 exec-timeout 10 0
```

**Privilege levels matter:**
- `privilege 15` — full admin access (all commands)
- `privilege 5` — read-only style access (show commands, limited config)

The auditor account exists for compliance reviews — they can inspect the configuration without being able to change it.

### Verify SSH Access

From an admin PC:

```
ssh -l admin 192.168.0.65
```

A successful SSH login confirms the configuration is working. Attempting Telnet should time out or be refused.

---

## Site-to-Site IPSec VPN

All traffic between HQ (192.168.0.64/26) and Branch (192.168.0.128/25) must be encrypted. A site-to-site IPSec VPN tunnel handles this — it's transparent to end users and encrypts everything at the network layer.

### How IPSec Works Here

```
HQ PC → HQ Router → [encrypt with AES-256] → WAN → [decrypt] → Branch Router → Branch PC
```

The VPN uses two phases:
- **IKE Phase 1** — establishes a secure management channel (ISAKMP SA) using a pre-shared key
- **IKE Phase 2** — negotiates the actual data encryption parameters (IPSec SA) and creates the tunnel

### Step 1 — Define Interesting Traffic (ACL)

Only HQ ↔ Branch traffic should be encrypted. Everything else goes normally.

```cisco
! On HQ Router
access-list 100 permit ip 192.168.0.64 0.0.0.63 192.168.0.128 0.0.0.127
```

### Step 2 — Configure ISAKMP Policy (Phase 1)

```cisco
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400

crypto isakmp key ORIGINBANK_VPN_PSK address 20.20.0.26
```

> Group 5 = 1536-bit Diffie-Hellman. This governs the key exchange — a larger group means more computational work for an attacker trying to break the key derivation.

### Step 3 — Configure Transform Set (Phase 2)

```cisco
crypto ipsec transform-set TS-AES-SHA esp-aes 256 esp-sha-hmac
 mode tunnel
```

`esp-aes 256` = AES-256 encryption for data confidentiality
`esp-sha-hmac` = SHA-1 HMAC for data integrity (ensures packets aren't tampered with in transit)

### Step 4 — Create and Apply Crypto Map

```cisco
crypto map VPN-MAP 10 ipsec-isakmp
 set peer 20.20.0.26
 set transform-set TS-AES-SHA
 set pfs group5
 set security-association lifetime seconds 86400
 match address 100

interface Serial0/1/0
 crypto map VPN-MAP
```

### Branch Router — Mirror Configuration

The Branch router uses the same settings with HQ's WAN IP as the peer:

```cisco
access-list 100 permit ip 192.168.0.128 0.0.0.127 192.168.0.64 0.0.0.63

crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400

crypto isakmp key ORIGINBANK_VPN_PSK address 20.20.0.1

crypto ipsec transform-set TS-AES-SHA esp-aes 256 esp-sha-hmac

crypto map VPN-MAP 10 ipsec-isakmp
 set peer 20.20.0.1
 set transform-set TS-AES-SHA
 set pfs group5
 match address 100

interface Serial0/1/0
 crypto map VPN-MAP
```

### Verify the VPN Tunnel

Trigger interesting traffic first (a ping between HQ and Branch), then check tunnel status:

```cisco
show crypto isakmp sa
```

Expected output:

```
dst            src            state    conn-id  slot  status
20.20.0.26     20.20.0.1      QM_IDLE  1        0     ACTIVE
```

```cisco
show crypto ipsec sa
```

Look for incrementing `#pkts encaps` and `#pkts decaps` counters — this confirms packets are being encrypted and decrypted through the tunnel.

In Packet Tracer, you can also inspect a packet at the WAN interface and observe the **ESP header** — confirming the payload is encrypted before traversing the public network.

---

## Verification Checklist

- [ ] Ping from external host to `192.168.0.2` fails (private IP not routable)
- [ ] Ping from external host to `20.20.0.2` succeeds and reaches the web server
- [ ] HTTP from external host to `20.20.0.2:80` loads the OriginBank page
- [ ] FTP from external host to DMZ is blocked
- [ ] ICMP from outside to HQ is blocked
- [ ] SSH to HQ router succeeds; Telnet is refused
- [ ] Auditor account cannot access privileged config mode
- [ ] `show crypto isakmp sa` shows `QM_IDLE` (active tunnel)
- [ ] `show crypto ipsec sa` shows incrementing encaps/decaps counters
- [ ] Packet capture on WAN link shows ESP header (encrypted payload)

---

[← Routing & Segmentation](02-routing-and-segmentation.md) | [Next: IPS & Security Analysis →](04-ips-and-security-analysis.md)
