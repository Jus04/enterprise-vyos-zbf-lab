# Multi-Zone Enterprise Gateway & Zero-Trust Firewall Lab

A virtualized, multi-zone enterprise edge network topology deployed in **GNS3** using **VyOS 1.4/1.5** as a Zone-Based Firewall (ZBF) gateway and **NetBox** as the single source of truth for IP Address Management (IPAM).

---

## 📌 Project Overview & Objectives
The goal of this project is to simulate an enterprise perimeter network adhering to Zero-Trust network isolation principles. The environment enforces strict stateful packet filtering, explicit protocol blocking, sub-interface VLAN segmentation, and live connection tracking audit logging.

### Key Technical Achievements
- **Layer 2/3 Segmentation:** Configured 802.1Q VLAN sub-interfaces (`eth0.10` and `eth0.20`) to segment traffic into discrete zones.
- **Zero-Trust Policy Enforcement:** Implemented Zone-Based Firewall (ZBF) rulesets with implicit default drop policies and stateful session inspection (`state-policy established accept`).
- **Kernel & State Table Troubleshooting:** Identified and resolved a complex state bypass issue where existing Linux `conntrack` sessions permitted return traffic across newly applied zone policies.
- **Physical/Logical Interface Alignment:** Diagnosed and corrected a virtual trunking mismatch where zone bindings were applied to inactive interfaces (`eth1.x` vs. active trunk `eth0.x`).
- **IPAM Integration:** Fully documented sites, prefixes, sub-interfaces, and endpoint bindings in NetBox.

---

## 🛠️ Architecture & Tech Stack
- **Virtualization Engine:** GNS3 integrated with Oracle VirtualBox
- **Router / ZBF Gateway:** VyOS 1.4/1.5 (Rolling / Sagitta release)
- **Infrastructure Management (IPAM):** NetBox (Deployed via Docker Compose)
- **Layer 2/3 Protocols:** 802.1Q VLAN Tagging, Stateful Inspection, Linux Connection Tracking (`conntrack`), ICMP Filtering

---

## 📐 Topology & Addressing Schema

### Network Subnets & Prefixes
| Zone Name | Sub-Interface | Subnet Range | Gateway IP | Endpoint Hosts |
| :--- | :--- | :--- | :--- | :--- |
| **GENERAL** | `eth0.10` | `10.10.10.0/24` | `10.10.10.1/24` | PC1 (`10.10.10.10`), PC2 (`10.10.10.20`) |
| **FINANCE** | `eth0.20` | `10.10.20.0/24` | `10.10.20.1/24` | PC4 (`10.10.20.11`), PC5 (`10.10.20.10`) |
| **WAN / NAT** | `eth1` | Dynamic (DHCP) | `10.0.2.2` | VyOS WAN Interface (`10.0.2.15/24`) |

---

## 🔒 Zone-Based Firewall (ZBF) Implementation

### Zone Mappings
- **Zone `GENERAL`:** Member interface `eth0.10`
- **Zone `FINANCE`:** Member interface `eth0.20`

### Applied Rulesets
1. **`GENERAL-TO-FINANCE`**
   - **Default Action:** Drop
   - **Rule 10:** Drop ICMP (`protocol icmp`, `action drop`, `log enabled`)
2. **`FINANCE-TO-GENERAL`**
   - **Default Action:** Drop
   - **State Policy:** Accept established / related traffic

---

## 📝 Key Engineering & Troubleshooting Milestones

### 1. Resolving the `conntrack` Session Persistence Trap
**Problem:** After applying a `default-action drop` policy on traffic from `GENERAL` to `FINANCE`, pings initiated from `PC2` (General) to `PC5` (Finance) continued to succeed.  
**Root Cause:** Prior pings initiated from `PC5` (Finance) to `PC2` established an active session in the kernel's connection tracking table (`nf_conntrack`). Subsequent outbound pings matched the existing `established` state, bypassing the newly created zone policy.  
**Resolution:** Flushed active session memory on VyOS using:
```bash
delete conntrack table ipv4
Subsequent pings from GENERAL to FINANCE were immediately blocked and logged as expected
