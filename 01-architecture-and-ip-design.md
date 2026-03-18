# 01 — Architecture & IP Design

[← Back to README](../README.md) | [Next: Routing & Segmentation →](02-routing-and-segmentation.md)

---

## Overview

Before a single command gets typed, the network needs a coherent design. This section covers the decisions behind the IP addressing strategy, the server placement, and the hardening baseline applied to every device in the topology.

A poorly planned address space creates operational headaches for years. Getting it right upfront — static where it matters, DHCP where it doesn't — makes every subsequent configuration step cleaner.

---

## IP Addressing Strategy

The network is divided into four logical zones, each with its own subnet. The principle is simple: **separate concerns, control the boundaries**.

| Zone | Subnet | Mask | Usable Range |
|------|--------|------|--------------|
| DMZ | 192.168.0.0 | /26 | .1 – .62 |
| HQ LAN | 192.168.0.64 | /26 | .65 – .126 |
| Branch LAN | 192.168.0.128 | /25 | .129 – .254 |
| WAN Links | 20.20.0.0 | /30 per link | Point-to-point |

### Static vs DHCP — the rule of thumb

**Static** for anything that needs to be reliably reachable: servers, gateways, firewall interfaces, network admin devices. If a DNS server's IP changes, everything that depends on it breaks.

**DHCP** for everything else: workstations, user laptops, branch PCs. They don't need a fixed address — they just need one.

### Static IP Configuration (Servers)

On each server in Packet Tracer:

```
Desktop > IP Configuration > Static
```

Set the IP, subnet mask, and default gateway manually. Example for the Internal DNS/DHCP Server:

```
IP Address:      192.168.0.66
Subnet Mask:     255.255.255.192
Default Gateway: 192.168.0.65
```

### DHCP Server Configuration

The DHCP server hands out addresses to workstations within its subnet. Configuration in Packet Tracer:

```
Server > Services > DHCP > ON
```

Define a pool:

```
Pool Name:       HQ-POOL
Default Gateway: 192.168.0.65
DNS Server:      192.168.0.66
Start IP:        192.168.0.68
Subnet Mask:     255.255.255.192
Max Users:       58
```

Workstations on the HQ VLAN are set to DHCP — they request an address on boot and get one from this pool automatically.

### Router Interface Configuration

Routers need IP addresses on each interface to act as gateways. A default route points all unknown traffic toward the ASA:

```cisco
interface GigabitEthernet7/0
 ip address 192.168.0.65 255.255.255.192
 no shutdown

ip route 0.0.0.0 0.0.0.0 20.20.0.2
```

---

## Device Hardening

Every router and switch in the topology gets a hardening baseline before any service is configured. The goal: reduce the attack surface to the absolute minimum.

### What was applied to every device

| Control | Why it matters |
|---------|----------------|
| Strong enable/console passwords | Prevents physical and local access without credentials |
| `service password-encryption` | Encrypts passwords stored in the config file |
| SSH v2 only (Telnet disabled) | Eliminates plaintext credential exposure |
| Login banner | Legal deterrent; establishes authorised-use policy |
| `exec-timeout 10 0` | Auto-terminates idle sessions after 10 minutes |
| Disable unused services | Removes HTTP server, CDP on external interfaces, etc. |
| Shutdown unused ports | Prevents rogue device connections on switch ports |

### Router hardening commands

```cisco
hostname HQ-Router
enable secret StrongP@ss123
service password-encryption

banner motd ^
  AUTHORISED ACCESS ONLY
  Unauthorised access is prohibited and will be prosecuted.
^

line console 0
 password ConP@ss
 login
 exec-timeout 10 0

line vty 0 4
 transport input ssh
 login local
 exec-timeout 10 0

no ip http server
no ip http secure-server
```

### Switch hardening — disabling unused ports

An open switch port is an open door. Any unused interface gets shut down:

```cisco
interface range FastEthernet0/7-24
 shutdown
```

Port security can also be added to active ports to restrict which MAC addresses are allowed:

```cisco
interface FastEthernet0/1
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
```

---

## Core Services: DNS, Web, Syslog

### DNS

Each subnet has its own DNS server. This avoids cross-zone DNS dependency — if the DMZ DNS goes down, HQ devices can still resolve names via their own server.

Configuration in Packet Tracer:

```
Server > Services > DNS > ON
```

Add A records:

| Hostname | Type | IP |
|----------|------|----|
| www.originbank.com | A | 192.168.0.2 |
| ns.originbank.com | A | 192.168.0.4 |
| mail.originbank.com | A | 192.168.0.3 |

Verify from a client PC:

```
nslookup www.originbank.com
```

### Web Server (DMZ placement)

The web server lives in the DMZ — not in the HQ LAN. This is a deliberate architectural decision.

If the web server is compromised (which public-facing servers frequently are), the attacker is contained within the DMZ. The ASA blocks all outbound connections from DMZ to HQ by default because of the security level difference (60 vs 100). The blast radius of a web server compromise is dramatically reduced.

```
Server > Services > HTTP > ON
```

The web server hosts `www.originbank.com` and is reachable from the outside via the ASA's public IP after NAT.

### Syslog Server (HQ placement)

The Syslog server sits inside HQ — away from the public-facing DMZ. All routers and the ASA ship their logs here. Centralised logging means:

- A single pane of glass for security events
- Logs survive even if a device is compromised or rebooted
- Correlation between events across devices is possible

Enable logging on each router:

```cisco
logging 192.168.0.67
logging trap informational
service timestamps log datetime msec
```

On the server side, just enable the Syslog service:

```
Server > Services > Syslog > ON
```

---

## Verification Checklist

Before moving on, confirm:

- [ ] All servers respond to ping from their default gateway
- [ ] DHCP clients on HQ and Branch receive addresses automatically
- [ ] `nslookup www.originbank.com` resolves correctly from a client PC
- [ ] Web server is accessible via browser from the HQ VLAN
- [ ] Syslog server is receiving entries from at least one router

---

[← Back to README](../README.md) | [Next: Routing & Segmentation →](02-routing-and-segmentation.md)
