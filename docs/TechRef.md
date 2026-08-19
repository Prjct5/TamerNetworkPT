# Tamer Topology — Technical Reference

Addressing plan, OSPF design, and the verification commands used to confirm each
build stage, exactly as run during the build.

## Site & Device Inventory

| Site               | Devices                              | Role                                          |
| ------------------ | ------------------------------------ | --------------------------------------------- |
| HQ Core            | SW-CORE1, SW-CORE2 (3560-24PS)       | Inter-VLAN routing, HSRP redundancy, STP root |
| HQ Access          | SW-ACC1, SW-ACC2 (2960)              | End-device access, port security              |
| HQ Edge            | R-EDGE-HQ (2911)                     | Routes HQ core to the Data Center and WAN     |
| HQ End Devices     | 8 PCs across HR/IT/Sales VLANs       | HSRP/VLAN/security verification               |
| Data Center        | R-DC, SW-DC, SRV-WEB, SRV-CORE       | Backend services site                         |
| Branch A           | R-BR-A (1941), SW-BR-A (2960), 4 PCs | Remote site, physically deployed              |
| Branch B           | R-BR-B (1941), SW-BR-B (2960), 4 PCs | Remote site, physically deployed              |
| Simulated Internet | R-ISP (2911)                         | Transit point between HQ and branches         |

## IP Addressing

**HQ VLANs (SVI + HSRP on SW-CORE1/SW-CORE2):**

| VLAN | Name       | Network         | HSRP VIP | SW-CORE1 (Active) | SW-CORE2 (Standby) |
| ---- | ---------- | --------------- | -------- | ----------------- | ------------------ |
| 10   | HR         | 192.168.10.0/24 | .1       | .2                | .3                 |
| 20   | IT         | 192.168.20.0/24 | .1       | .2                | .3                 |
| 30   | Sales      | 192.168.30.0/24 | .1       | .2                | .3                 |
| 50   | Wireless   | 192.168.50.0/24 | .1       | .2                | .3                 |
| 60   | IoT        | 192.168.60.0/24 | .1       | .2                | .3                 |
| 99   | Management | 192.168.99.0/24 | .1       | .2                | .3                 |

**Branch LANs:**

| Site     | Network          | Gateway     |
| -------- | ---------------- | ----------- |
| Branch A | 192.168.110.0/24 | .1 (R-BR-A) |
| Branch B | 192.168.120.0/24 | .1 (R-BR-B) |

**Data Center:**

| Device             | IP               |
| ------------------ | ---------------- |
| DC LAN             | 192.168.200.0/24 |
| SRV-WEB            | 192.168.200.10   |
| SRV-CORE           | 192.168.200.11   |
| R-DC LAN interface | 192.168.200.1    |

**WAN / Transit:**

| Link                 | Network         |
| -------------------- | --------------- |
| R-EDGE-HQ ↔ R-ISP    | 203.0.113.0/30  |
| R-ISP ↔ R-BR-A       | 203.0.113.4/30  |
| R-ISP ↔ R-BR-B       | 203.0.113.8/30  |
| R-EDGE-HQ ↔ SW-CORE1 | 10.255.255.0/30 |
| R-EDGE-HQ ↔ R-DC     | 10.255.255.4/30 |

## OSPF Design

- **Area 0 (Backbone):** SW-CORE1, SW-CORE2, R-EDGE-HQ, R-DC — fully converged.
- **Router IDs:** `1.1.1.1` = SW-CORE1, `2.2.2.2` = SW-CORE2, `10.0.0.1` = R-EDGE-HQ,
  `20.0.0.1` = R-DC.
- Branch A and Branch B are physically deployed with their LAN-side interfaces
  addressed and ready for OSPF, but are not yet joined to the backbone — they show
  zero neighbors until the WAN connection between HQ and the branches is completed.

## Verification Commands Used at Each Stage

Base device hardening (every device):
```
show running-config | include hostname
show ip ssh
```

HQ Core — VLANs, trunking, HSRP:
```
show vlan brief
show standby brief
show interfaces trunk
```
`show standby brief` confirms SW-CORE1 as Active and SW-CORE2 as Standby across
every VLAN — the actual proof that HSRP failover is configured correctly, not
just that the commands were typed.

HQ Access — port assignment and security:
```
show vlan brief
show interfaces status
```

Routed links (HQ Edge, Data Center):
```
show ip interface brief
ping <remote interface IP>
```

OSPF backbone convergence:
```
show ip ospf neighbor
show ip route ospf
```
`show ip ospf neighbor` should report `FULL` state for every adjacency; `show ip
route ospf` confirms each router is actually learning the others' subnets, not
just forming a neighbor relationship.
