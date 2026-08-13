# CCNA Lab: Spanning Tree Protocol (STP) & Rapid STP (RSTP) Fundamentals

## Overview

Redundancy is critical in modern network design to ensure high availability. However, redundant Layer 2 physical links created in an Ethernet mesh topology naturally form physical loops. Without a loop-prevention protocol, Layer 2 frames circulate infinitely, causing devastating network failures.

This laboratory provides a comprehensive analysis of Layer 2 loop prevention, bridging theory and hands-on implementation. We examine the core mechanics of IEEE 802.1D (STP / PVST+) and demonstrate the evolutionary transition to IEEE 802.1w (Rapid STP / Rapid-PVST+), focusing on convergence timing, port roles, and operational states.

---

## Technical Background

### The Layer 2 Loop Problem

Unlike Layer 3 IP packets—which feature a Time-To-Live (TTL) header field to drop looped packets—Ethernet Layer 2 frames lack a TTL field. In a topology with redundant physical paths and no spanning-tree protocol, three major issues occur:

1. **Broadcast Storms:** Switches continuously flood broadcast frames (such as ARP Requests) out of all interfaces except the receiving one. These frames loop endlessly, consuming 100% of available bandwidth and causing CPU starvation across network devices.
2. **MAC Address Table Instability:** The constant arrival of identical source MAC addresses on different physical interfaces forces switches to continually rewrite their CAM (Content Addressable Memory) tables, crippling frame forwarding logic.
3. **Multiple Frame Transmission:** End hosts receive duplicate copies of unicast frames, leading to protocol stack errors at the upper layers.

---

### How Spanning Tree Protocol (STP) Solves the Problem

The Spanning Tree Protocol constructs a logical loop-free tree topology by strategically blocking redundant physical ports. The decision process follows an exact deterministic order:

1. **Elect One Root Bridge:** The switch with the lowest Bridge ID (BID) is elected as the central reference point for the entire Layer 2 domain.
   $$\text{Bridge ID (BID)} = \text{Bridge Priority} + \text{Extended System ID (VLAN ID)} + \text{MAC Address}$$
2. **Determine Root Ports (RP):** Every non-Root switch elects exactly one Root Port—the interface leading to the Root Bridge with the lowest cumulative Path Cost.
3. **Determine Designated Ports (DP):** Every network segment (collision domain) elects one Designated Port to forward traffic based on the lowest path cost toward the Root Bridge.
4. **Block Remaining Ports (Alternate Ports):** All other interfaces not designated as Root or Designated ports are placed into a Blocking state to break physical loops.

---

### IEEE 802.1D (STP / PVST+) vs IEEE 802.1w (RSTP / Rapid-PVST+)

#### IEEE 802.1D Port States & Timers
Classic STP relies on passive timers to safely transition interfaces without creating temporary loops, resulting in slow convergence times (up to 50 seconds):

| Port State | Function | Timer |
| :--- | :--- | :--- |
| **Blocking** | Drops data frames, listens to BPDUs | Max Age (20s) |
| **Listening** | Processes BPDUs, builds topology without learning MACs | Forward Delay (15s) |
| **Learning** | Populates MAC address table, drops data frames | Forward Delay (15s) |
| **Forwarding** | Sends and receives user data frames | Active Forwarding |
| **Disabled** | Administrative down state | N/A |

#### IEEE 802.1w Port States & Convergence Mechanics
RSTP dramatically improves convergence times down to milliseconds by consolidating port states and introducing an active negotiation mechanism called **Proposal / Agreement**:

| 802.1D State | 802.1w State | Operational Status |
| :--- | :--- | :--- |
| Disabled | **Discarding** | Drops data, populates no MACs |
| Blocking | **Discarding** | Drops data, listens to BPDUs |
| Listening | **Discarding** | Drops data, processes BPDUs |
| Learning | **Learning** | Populates MAC table, drops data |
| Forwarding | **Forwarding** | Active data transmission |

---

## Topology Architecture

![Top]()

##



