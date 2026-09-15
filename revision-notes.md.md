# Enterprise Network Project — Revision Notes

A complete reference for the small-office/enterprise-style network built in Cisco Packet Tracer: 3 departments (Sales, IT, Guest), a Printers VLAN, inter-VLAN routing, scoped DHCP, a wireless guest network, and ACL-based access restriction.

---

## 1. Topology Overview

- **1 Router** (2911) — the core, does all inter-VLAN routing, DHCP, and ACL filtering
- **2 Switches** (2960-24TT) — connected via a trunk link, carrying multiple VLANs between them
- **1 Server** — general LAN device
- **1 Printer** — isolated in its own VLAN, restricted access
- **6 PCs** — split across Sales (VLAN 10), IT (VLAN 20), Guest (VLAN 30)
- **1 Wireless Router (WRT300N) + 1 Laptop** — Guest Wi-Fi, sitting behind its own NAT layer

**Subnets used:**
| VLAN | Department | Subnet |
|---|---|---|
| 10 | Sales | 192.168.10.0/24 |
| 20 | IT | 192.168.20.0/24 |
| 30 | Guest | 192.168.30.0/24 |
| 40 | Printers | 192.168.40.0/24 |
| — | Wireless (behind WRT300N NAT) | 192.168.100.0/24 |

---

## 2. Stage 1 — VLANs + Trunking

**Concept:** VLANs let multiple "logical" separate networks share the same physical switches, instead of needing separate cabling per department. A trunk link carries traffic for *all* VLANs between two switches over one physical connection.

**Key commands:**
```
vlan 10
 name Sales
exit

interface fastEthernet 0/1
 switchport mode access
 switchport access vlan 10
exit

interface fastEthernet 0/24
 switchport mode trunk
exit
```

**Verification:**
- `show vlan brief` — lists access ports per VLAN. **Trunk ports never appear here** — this is normal, not a bug. If a port is "missing" from the VLAN 1 default list, that's often a sign it became a trunk port.
- `show interfaces trunk` — the real way to confirm trunk status; shows which ports are trunking and which VLANs are allowed across them.

**Lesson learned:** Don't rely on `show vlan brief` to check trunk ports — use `show interfaces trunk` instead.

---

## 3. Stage 2 — Inter-VLAN Routing (Router-on-a-Stick)

**Concept:** VLANs are isolated from each other by default — even on the same router, you need dedicated routing between them. "Router-on-a-stick" means one physical router interface is split into multiple logical **sub-interfaces**, one per VLAN, using 802.1Q tagging.

**Key commands:**
```
interface gigabitEthernet 0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
exit

interface gigabitEthernet 0/0
 no shutdown
exit
```

**Important:** The switch port connecting to the router must ALSO be set to trunk mode — otherwise VLAN tags never reach the router at all.

**Verification:** `show ip interface brief` — confirm each sub-interface shows `up/up`.

---

## 4. Stage 3 — DHCP Scoped Per Department

**Concept:** Each VLAN/subnet gets its own DHCP pool, so devices automatically receive the correct IP range, gateway, and DNS for their department.

**Key commands:**
```
ip dhcp excluded-address 192.168.10.1

ip dhcp pool SALES-POOL
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
exit
```

**Common mistakes made (and why they matter):**
- Using the **network address** (`192.168.10.0`) as the default-router instead of the actual gateway IP (`192.168.10.1`) — network addresses are never assignable to a device.
- Typing `default router` (two words) instead of `default-router` (hyphenated) — causes "Invalid input detected," a classic CLI syntax error, not a real config problem.
- Forgetting a line (e.g., `dns-server`) before typing `exit` — easy to miss, always re-check with `show run` after.

**Verification:**
- `show ip dhcp binding` — lists every leased IP, which pool it came from, and its MAC address. This is the most direct way to confirm scoping is working correctly.

**Sign of broken DHCP on a client:** IP shows as `169.254.x.x` (APIPA — self-assigned fallback) or `0.0.0.0`. This is the clearest single symptom to recognize.

---

## 5. Stage 4 — Wireless Guest Network (Double NAT)

**Concept:** A wireless router (WRT300N) was placed *inside* the Guest VLAN, getting its own address from GUEST-POOL on its **Internet/WAN** side, while running its own separate DHCP pool for wireless clients on its **LAN** side (`192.168.100.0/24`). This creates a realistic "layered" structure — common in real small offices for isolating guest Wi-Fi.

**Two-sided configuration:**
- **WAN side** (Setup → Basic Setup → Internet Connection Type): Automatic Configuration – DHCP → gets an address like `192.168.30.x` from the office network.
- **LAN side** (Setup → Basic Setup → Network Setup): Router IP `192.168.100.1`, own DHCP server enabled with a separate range.

**Real issues hit and how they were diagnosed:**
1. **Mismatched IP/gateway subnets on the laptop** (IP in `192.168.30.x`, gateway in `192.168.100.x`) — a config combination that's *always* invalid, since a gateway must be in the same subnet as the device's own IP. Cause: a **stale DHCP lease** taken before the WRT300N's LAN side was configured. Fixed with `ipconfig /release` + `ipconfig /renew`.
2. **WRT300N's Internet IP stuck at `0.0.0.0`**, GUI buttons doing nothing. Root cause: **no cable was actually connected** to the device at all (checked via Physical tab — all ports empty). Software settings can look perfect and still fail if there's no physical link underneath — always verify physical connectivity when nothing responds at the config level.

**Testing a layered connection — ping hop by hop:**
```
ping 192.168.100.1   → first hop (laptop to its own wireless router)
ping 192.168.30.1    → second hop (through NAT, to the office router)
```
Testing hop by hop (instead of jumping straight to the final destination) is the key technique for isolating *where* in a multi-layer path a failure actually is.

**Note on ping timeouts:** the very first packet in a ping often fails even on a working link — this is normal **ARP delay** (the device pausing to resolve a MAC address before the first packet can go out). 3/4 or higher success right after a single early timeout is a pass, not a fault.

---

## 6. Stage 5 — Restricting Printer Access with an ACL

**Concept:** Access Control Lists (ACLs) filter traffic based on rules — here, blocking Guest from reaching the printer while still allowing Sales and IT.

**Key commands:**
```
access-list 10 deny 192.168.30.0 0.0.0.255
access-list 10 permit any

interface gigabitEthernet 0/0.40
 ip access-group 10 out
```

**Important ACL concepts:**
- **Wildcard mask** (`0.0.0.255`) is the *inverse* of a subnet mask — it tells the ACL "match all addresses in this /24 range," not one specific host.
- **ACLs deny everything by default** — without an explicit `permit any` line at the end, every other department would also lose access, not just the one being blocked.
- **Direction matters — `in` vs `out`:** applying the ACL to the wrong direction on an interface means it inspects the wrong stream of traffic entirely. The real lesson: `in` means "traffic entering the router through this specific interface"; `out` means "traffic leaving the router through this interface." For blocking Guest traffic *arriving* at the printer's subnet, `out` (on the printer's own sub-interface) was correct, since that's genuinely where the traffic exits toward its destination.

**How the mistake was diagnosed:** `show access-lists` showed the **deny** line stuck at 0 matches even though a genuinely blocked device was pinging — proof the ACL itself was never evaluating that traffic at all, pointing to a direction/placement issue rather than a rule-content issue.

**Verification:**
```
show access-lists
```
Watch the match counters — deny count increases only when blocked traffic is actually being caught.

**Real-world signal:** a blocked-by-ACL ping often returns "Destination host unreachable" — this looks identical to a routing failure at first glance, but the underlying cause (explicit policy block vs. no route) is completely different. Always check `show access-lists` before assuming it's a routing issue.

---

## 7. Master Command Reference

| Purpose | Command |
|---|---|
| Enter privileged mode | `enable` |
| Enter global config | `configure terminal` |
| Create a VLAN | `vlan <number>` then `name <name>` |
| Set access port | `switchport mode access` / `switchport access vlan <number>` |
| Set trunk port | `switchport mode trunk` |
| Create router sub-interface | `interface <intf>.<vlan>` / `encapsulation dot1Q <vlan>` / `ip address ...` |
| Enable a physical interface | `no shutdown` |
| Exclude an address from DHCP | `ip dhcp excluded-address <ip>` |
| Create a DHCP pool | `ip dhcp pool <name>` / `network ...` / `default-router ...` / `dns-server ...` |
| Create a standard ACL | `access-list <number> deny/permit <network> <wildcard>` |
| Apply an ACL to an interface | `ip access-group <number> in/out` |
| Save config | `write memory` (or `wr`) |
| Show VLAN-to-port mapping | `show vlan brief` |
| Show trunk status | `show interfaces trunk` |
| Show interface status/IP/ACL | `show ip interface brief` / `show ip interface <intf>` |
| Show current DHCP leases | `show ip dhcp binding` |
| Show ACL rules + match counts | `show access-lists` |
| PC network info | `ipconfig /all` |
| Force new DHCP lease | `ipconfig /release` then `ipconfig /renew` |
| Test reachability | `ping <ip>` |
| Test DNS resolution | `nslookup <name>` |
| Trace path hop by hop | `tracert <ip>` |

---

## 8. Big-Picture Lessons for Interviews

- **VLANs vs. subnets vs. routing** are three distinct but connected concepts: VLANs separate traffic at Layer 2 (switching), subnets define address groupings, and routing (inter-VLAN routing here) is what allows two otherwise-separate VLANs/subnets to actually communicate.
- **A device's IP and its default gateway must always be in the same subnet** — if they're not, it's an immediate, guaranteed failure, and usually a sign of a stale lease or a config mismatch.
- **Physical connectivity should be checked early**, not last — a perfectly correct software configuration will still fail completely if there's no actual cable in place.
- **"Timeout" and "Destination host unreachable" mean different things** — a timeout with no reply at all often points to no route/no response; an explicit "unreachable" message (especially from a specific device, not the final destination) often indicates the *sender* of that message is actively rejecting or blocking the request — worth checking ACLs, not just routing tables.
- **The very first ping packet failing is often normal** (ARP delay), not evidence of a real problem — always look at the full result, not just the first line.
- **`show` commands with match/lease counters** (`show access-lists`, `show ip dhcp binding`) are more reliable than pinging alone, because they tell you definitively whether a rule or pool is actually being used, not just whether traffic happened to succeed or fail.
