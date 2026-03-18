# 02 — Routing & Segmentation

[← Architecture & IP Design](01-architecture-and-ip-design.md) | [Next: Firewall, ACLs & VPN →](03-firewall-acl-vpn.md)

---

## Overview

With addressing in place, the next challenge is making traffic flow efficiently across the topology — while keeping different network segments logically isolated. This section covers two things:

1. **OSPF** — dynamic routing so routers automatically learn paths to every subnet
2. **VLANs + Inter-VLAN routing** — logical network segmentation within the HQ switch fabric

These aren't just technical requirements. They're the foundation of the security model. You cannot enforce zone-based access control if devices in different security zones are on the same flat network.

---

## Why OSPF Over RIPv2

Both are IGPs (Interior Gateway Protocols), but RIPv2 has no place in a modern enterprise network:

| | RIPv2 | OSPF |
|--|-------|------|
| Convergence | Up to 180 seconds | Seconds |
| Metric | Hop count (max 15) | Cost (bandwidth-based) |
| Scalability | Poor | Excellent |
| Loop prevention | Slow (timers) | Topology-aware (SPF) |
| Authentication | Optional MD5 | Supported |

RIPv2's 30-second update cycle and 180-second holddown timer mean a routing failure can black-hole traffic for three minutes. In a banking network, that's unacceptable. OSPF detects failures within seconds and reconverges immediately.

---

## OSPF Configuration

All routers in the topology run OSPF in a single area (Area 0). Each router advertises its directly connected networks.

### HQ Router

```cisco
router ospf 1
 router-id 1.1.1.1
 network 192.168.0.64 0.0.0.63 area 0
 network 20.20.0.0 0.0.0.3 area 0
 passive-interface GigabitEthernet7/0
```

> `passive-interface` on the LAN-facing interface prevents OSPF hello packets from being sent to end devices — they don't need to participate in routing.

### Branch Router

```cisco
router ospf 1
 router-id 3.3.3.3
 network 192.168.0.128 0.0.0.127 area 0
 network 20.20.0.24 0.0.0.3 area 0
 passive-interface FastEthernet0/0
```

### Verify OSPF Adjacency

```cisco
show ip ospf neighbor
```

Expected output — all neighbours showing `FULL` state:

```
Neighbor ID     Pri   State       Dead Time   Address         Interface
2.2.2.2           1   FULL/DR     00:00:38    20.20.0.2       Serial0/1/0
3.3.3.3           1   FULL/DR     00:00:35    20.20.0.26      Serial0/1/1
```

### Verify Routing Table

```cisco
show ip route ospf
```

You should see `O` (OSPF) entries for every remote subnet — HQ sees Branch, Branch sees HQ, all WAN links are learned automatically.

### End-to-End Connectivity Test

```
ping 192.168.0.130    ← from HQ PC to Branch PC
```

A successful ping here confirms OSPF is working correctly across all routers.

---

## VLAN Segmentation

VLANs divide the HQ switch fabric into logical segments. Without them, all HQ devices share the same broadcast domain — a single compromised host can sniff traffic from every other device on the switch.

The VLAN plan for HQ:

| VLAN ID | Name | Purpose |
|---------|------|---------|
| 10 | DMZ | Public-facing servers |
| 20 | HQ-LAN | Internal workstations |
| 30 | BRANCH | Branch network (routed via WAN) |
| 99 | MGMT | Management access (admin only) |

### VLAN Creation on SW1

```cisco
vlan 10
 name DMZ
vlan 20
 name HQ-LAN
vlan 30
 name BRANCH
vlan 99
 name MGMT
```

### Access Port Configuration

Access ports carry traffic for a single VLAN. Workstations plug into access ports:

```cisco
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
```

> `spanning-tree portfast` skips the 30-second STP listening/learning delay on end-device ports — workstations get connectivity immediately on boot.

### Trunk Port Configuration

Trunk ports carry traffic for multiple VLANs simultaneously, using 802.1Q tags to identify which VLAN each frame belongs to. The uplink between the switch and the router is a trunk:

```cisco
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 10,20,30,99
```

### Verify VLAN Assignment

```cisco
show vlan brief
```

Output should show each VLAN with the correct ports assigned:

```
VLAN Name                Status    Ports
---- -------------------- --------- -------------------
10   DMZ                  active    Fa0/5, Fa0/6
20   HQ-LAN               active    Fa0/2, Fa0/3, Fa0/4
99   MGMT                 active    Fa0/1
```

```cisco
show interfaces trunk
```

Confirms which ports are trunking and which VLANs are active on each trunk.

---

## Inter-VLAN Routing (Router-on-a-Stick)

VLANs create isolation — but HQ workstations (VLAN 20) still need to reach the DMZ servers (VLAN 10). Inter-VLAN routing solves this by creating sub-interfaces on the router, one per VLAN. Each sub-interface acts as the default gateway for that VLAN.

The physical interface connecting the router to the switch is set to trunk mode. The router then processes the 802.1Q tags and routes between VLANs.

### Sub-interface Configuration on HQ Router

```cisco
interface GigabitEthernet7/0
 no ip address
 no shutdown

interface GigabitEthernet7/0.10
 encapsulation dot1Q 10
 ip address 192.168.0.1 255.255.255.192
 description DMZ-Gateway

interface GigabitEthernet7/0.20
 encapsulation dot1Q 20
 ip address 192.168.0.65 255.255.255.192
 description HQ-LAN-Gateway

interface GigabitEthernet7/0.99
 encapsulation dot1Q 99 native
 ip address 192.168.0.193 255.255.255.252
 description MGMT-Gateway
```

Each sub-interface gets the gateway IP for its VLAN. Workstations in VLAN 20 have their default gateway set to `192.168.0.65` — traffic destined for another subnet hits that gateway, gets routed via the sub-interface, and lands in the right VLAN.

### Verify Sub-interfaces Are Up

```cisco
show ip interface brief
```

All sub-interfaces should show `up/up`:

```
Interface              IP-Address      OK? Method Status   Protocol
GigabitEthernet7/0     unassigned      YES unset  up       up
GigabitEthernet7/0.10  192.168.0.1     YES manual up       up
GigabitEthernet7/0.20  192.168.0.65    YES manual up       up
GigabitEthernet7/0.99  192.168.0.193   YES manual up       up
```

### Verify Inter-VLAN Communication

From a PC in VLAN 20:

```
ping 192.168.0.2    ← Web server in VLAN 10 (DMZ)
```

A successful ping here confirms traffic is crossing VLAN boundaries correctly through the router sub-interfaces.

---

## Verification Checklist

- [ ] `show ip ospf neighbor` — all neighbours in `FULL` state
- [ ] `show ip route ospf` — all remote subnets present in routing table
- [ ] Ping from HQ PC to Branch PC succeeds
- [ ] `show vlan brief` — all VLANs active with correct port assignments
- [ ] `show interfaces trunk` — trunk port carrying all required VLANs
- [ ] `show ip interface brief` — all sub-interfaces up/up
- [ ] Inter-VLAN ping from VLAN 20 to VLAN 10 succeeds

---

[← Architecture & IP Design](01-architecture-and-ip-design.md) | [Next: Firewall, ACLs & VPN →](03-firewall-acl-vpn.md)
