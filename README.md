# Enterprise Network Simulation — Cisco Packet Tracer

A small-office/enterprise-style network built, broken, diagnosed, and fixed from scratch in Cisco Packet Tracer, as hands-on preparation for an IT Support / Help Desk role.

> Built while completing the **Google IT Support Professional Certificate** and **CCNA (Introduction to Networks / Routing & Switching Essentials)** coursework.

---

## Project Overview

This project simulates a small office with three departments and a shared printer, connected across two switches, one router, and a wireless guest network — designed to mirror the kind of environment an entry-level IT Support technician would actually work in.

**What it demonstrates:**
- VLAN segmentation across multiple switches (trunking)
- Inter-VLAN routing using Router-on-a-Stick
- DHCP scoped independently per department
- A wireless guest network isolated via double NAT
- Access restriction to a shared resource using ACLs
- Real troubleshooting: every stage below includes an actual misconfiguration I hit, diagnosed, and resolved — not just a working end state

---

## Network Topology

```
                         [Router0]
                        /    |    \
                       /     |     \
              (VLAN 10-30) (VLAN 40)  (Internet uplink)
                    /          |            \
              [Switch0]----[trunk]----[Switch1]      [WRT300N]
               /   |                    |    \             \
           Sales  Printer            IT PCs  Guest PCs    Laptop
            PCs                                          (Wireless)
```

| VLAN | Department | Subnet |
|---|---|---|
| 10 | Sales | 192.168.10.0/24 |
| 20 | IT | 192.168.20.0/24 |
| 30 | Guest | 192.168.30.0/24 |
| 40 | Printers | 192.168.40.0/24 |
| — | Wireless (behind NAT) | 192.168.100.0/24 |

---

## Build Stages

### 1. VLAN Segmentation + Trunking
Created Sales, IT, and Guest VLANs across two switches, connected via a trunk link so VLAN traffic could pass between them.
*Verification tool used: `show interfaces trunk` — a common gotcha is that `show vlan brief` never lists trunk ports, which can look like a misconfiguration when it isn't.*

### 2. Inter-VLAN Routing (Router-on-a-Stick)
Configured router sub-interfaces with 802.1Q tagging so a single physical router link could route between all three department VLANs.

### 3. DHCP Scoped Per Department
Each VLAN pulls addresses from its own DHCP pool, with the router excluding its own gateway addresses from each range.

**Issue hit & fixed:** A pool was created with the default-router set to the network address instead of the actual gateway IP — an easy but completely invalid config, since network addresses aren't assignable to a device. Diagnosed via `show ip dhcp binding` and corrected by rebuilding the pool.

### 4. Wireless Guest Network (Double NAT)
A wireless router was placed inside the Guest VLAN, obtaining its own address from the office DHCP pool on its WAN side, while running an independent DHCP pool for wireless clients on its LAN side — mirroring how guest Wi-Fi is commonly deployed behind a small office's main network.

**Issue hit & fixed:** The wireless router showed no WAN IP and its admin panel controls appeared unresponsive. Diagnosis by checking the device's physical port view revealed the real cause: no cable was actually connected. A reminder that physical connectivity should be checked early, not assumed.

### 5. ACL-Based Access Restriction
Restricted the shared printer so Guest devices cannot reach it, while Sales and IT retain access.

**Issue hit & fixed:** The ACL was initially applied in the wrong direction (`in` instead of `out`) on the printer's interface, so it silently never matched any Guest traffic. Diagnosed using `show access-lists`, which showed the deny rule's match counter stuck at zero despite active testing — corrected by reapplying the ACL in the outbound direction.

---

## Skills Demonstrated

`VLANs` `Trunking` `Inter-VLAN Routing` `Router-on-a-Stick` `DHCP Configuration` `NAT` `Wireless Networking` `Access Control Lists (ACL)` `Network Troubleshooting` `Cisco IOS CLI`

---

## Full Command Reference & Detailed Notes

See [`revision-notes.md`](./revision-notes.md) for the complete command reference, every configuration step, and a full write-up of each issue encountered and how it was diagnosed.

---

## Screenshots

*(Add screenshots here: VLAN table, DHCP bindings, ACL match counters, ping test results)*

---

## About This Project

Built as part of my transition into IT Support, alongside the Google IT Support Professional Certificate and CCNA coursework. Focused deliberately on breaking things and diagnosing real symptoms — wrong gateways, missing physical links, misapplied ACL direction — rather than just building a network that works on the first try, since that diagnostic process is closer to what real support work actually involves.
