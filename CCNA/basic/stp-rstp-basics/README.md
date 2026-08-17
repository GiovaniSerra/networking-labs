# CCNA Lab: Spanning Tree Protocol (STP) & Rapid STP (RSTP) Fundamentals

## Overview

Redundancy is critical in modern network design to ensure high availability. However, redundant Layer 2 physical links created in an Ethernet mesh topology naturally form physical loops. Without a loop-prevention protocol, Layer 2 frames circulate infinitely, causing devastating network failures.

This laboratory provides a comprehensive analysis of Layer 2 loop prevention, bridging theory and hands-on implementation. We examine the core mechanics of IEEE 802.1D (STP / PVST+) and demonstrate the evolutionary transition to IEEE 802.1w (Rapid STP / Rapid-PVST+), focusing on convergence timing, port roles, and operational states.

---

## Technical Background

### The Layer 2 Loop Problem

Unlike Layer 3 IP packets—which feature a Time-To-Live (TTL) header field to drop looped packets—Ethernet Layer 2 frames lack a TTL field. In a topology with redundant physical paths and no spanning-tree protocol, three major issues occur:

1. **Broadcast Storms:** Switches generate excessive, looping Layer 2 traffic that can severely degrade network performance or saturate available bandwidth, potentially leading to a denial-of-service condition across the broadcast domain.
2. **MAC Address Table Instability:** The constant arrival of identical source MAC addresses on different physical interfaces forces switches to continually rewrite their CAM (Content Addressable Memory) tables, crippling frame forwarding logic.
3. **Multiple Frame Transmission:** End hosts receive duplicate copies of unicast frames, leading to protocol stack errors at the upper layers.

---

### How Spanning Tree Protocol (STP) Solves the Problem

The Spanning Tree Protocol constructs a logical loop-free tree topology by strategically blocking redundant physical ports. The decision process follows an exact deterministic order:

1. **Elect One Root Bridge:** The switch with the lowest Bridge ID (BID) is elected as the central reference point for the entire Layer 2 domain.
   $$\text{Bridge ID (BID)} = \text{Bridge Priority} + \text{Extended System ID (VLAN ID)} + \text{MAC Address}$$
2. **Determine Root Ports (RP):** Every non-Root switch elects exactly one Root Port, representing the local interface receiving the best (lowest value) BPDU toward the Root Bridge. The election evaluates the following criteria in sequential order:
   * **Lowest Root Bridge ID (Root BID):** Identifies the target Root Switch.
   * **Lowest Root Path Cost (RPC):** Cumulative cost of the path to reach the Root Bridge.
   * **Lowest Sender Bridge ID (Sender BID):** Evaluates the BID of the upstream neighbor switch transmitting the BPDU.
   * **Lowest Sender Port ID (Sender PID):** Evaluates the port priority and port index of the transmitting switch interface.
   * **Lowest Local Port ID:** Final local tie-breaker applied when multiple local interfaces receive identical BPDUs from the same upstream port.
3. **Determine Designated Ports (DP):** Exactly one Designated Port is elected per collision domain / network segment to forward traffic toward the Root Bridge. The switch that advertises the most attractive BPDU onto the segment wins the Designated role based on the following hierarchy:
   * **Lowest Root Path Cost:** The switch advertising the lowest cumulative path cost to the Root Bridge.
   * **Lowest Sender Bridge ID:** If path costs are equal, the switch with the lower Bridge ID wins the designated role for the segment.
   * **Lowest Sender Port ID:** If a single switch has multiple ports connected to the same shared segment, the local interface with the lowest Port ID (Priority + Port Number) becomes the Designated Port.
4. **Block Remaining Ports (Alternate Ports):** All other interfaces not designated as Root or Designated ports are placed into an Alternate/Blocking state to break physical loops.

---

### IEEE 802.1D (STP / PVST+) vs IEEE 802.1w (RSTP / Rapid-PVST+)

#### IEEE 802.1D Port States & Timers

Classic STP relies on passive timers to safely transition interfaces without creating temporary loops, resulting in slow convergence times. Under classic 802.1D Spanning Tree, standard topology reconvergence can take up to 50 seconds based on default timers: **Max Age (20s)** in the event of direct/indirect failure detection, followed by **Listening (15s)** and **Learning (15s)** states before reaching Forwarding.

| Port State | Function | Timer |
| :--- | :--- | :--- |
| **Blocking** | Drops data frames, listens to BPDUs | Max Age (20s) |
| **Listening** | Processes BPDUs, builds topology without learning MACs | Forward Delay (15s) |
| **Learning** | Populates MAC address table, drops data frames | Forward Delay (15s) |
| **Forwarding** | Sends and receives user data frames | Active Forwarding |
| **Disabled** | Administrative down state | N/A |

#### IEEE 802.1w Port States & Convergence Mechanics

RSTP achieves sub-second convergence primarily through the explicit **Proposal/Agreement handshake** rather than passive timer expiration. Immediate transition to forwarding depends on specific port and link classifications:

1. **Edge Ports (PortFast):** Ports connected directly to end-user devices (hosts, servers) that will never receive BPDUs. They transition immediately to the Forwarding state without undergoing negotiation.
2. **Point-to-Point Links:** Inter-switch links operating in **Full-Duplex** mode. RSTP assumes full-duplex links are point-to-point connections between two switches, enabling the bidirectional Proposal/Agreement handshake. (Half-duplex links revert to shared media mode and fallback to 802.1D timers).
3. **Alternate & Backup Roles:** Switches maintain immediate pre-computed backup paths. If the Root Port fails, an Alternate Port immediately transitions to Root Port without recalculation delays.

| 802.1D State | 802.1w State | Operational Status |
| :--- | :--- | :--- |
| **Disabled** | Discarding | Drops data, populates no MACs |
| **Blocking** | Discarding | Drops data, listens to BPDUs |
| **Listening** | Discarding | Drops data, processes BPDUs |
| **Learning** | Learning | Populates MAC table, drops data |
| **Forwarding** | Forwarding | Active data transmission |

---

### Wireshark Protocol Analysis & Interoperability Note

#### 1. IEEE 802.1D / PVST+ Configuration BPDU Deep Dive

* **Destination Multicast MAC:** `01:80:c2:00:00:00` (Nearest-Customer-Bridge / Spanning Tree Multicast)
* **Source MAC Address:** `50:00:00:01:00:00` (Originating interface from Root Bridge SW1)
* **Protocol Version Identifier:** `0` (Classic IEEE 802.1D Spanning Tree)
* **BPDU Type:** `0x00` (Configuration BPDU)
* **BPDU Flags (0x00):**
  * Topology Change Acknowledgment (TCA) = No (Bit 7)
  * Topology Change (TC) = No (Bit 0)
* **Root Identifier:** `32768 / 1 / 50:00:00:01:00:00` (Priority 32768 + Sys-ID-Ext 1 = BID 32769)
* **Root Path Cost:** `0` (Originating directly from Root Bridge)
* **Protocol Timers:** Message Age: 0, Max Age: 20s, Hello Time: 2s, Forward Delay: 15s

> **Technical Note on BPDU Multicast Addressing:**  
> In Cisco PVST+ and Rapid-PVST+, on the native VLAN (VLAN 1 by default), switches generate two types of BPDUs across trunks:
> 1. An untagged standard IEEE BPDU sent to `0180.c200.0000` for backward compatibility with non-Cisco/CST switches.
> 2. A Cisco-proprietary PVST+ BPDU encapsulated via SNAP and sent to `0100.0ccc.cccd` carrying the VLAN tag/TLV.  
> For all non-native/other VLANs, BPDUs are transmitted exclusively to `0100.0ccc.cccd`.

---
## Deep Dive: BPDU Frame Structure & Header Evolution

Bridge Protocol Data Units (BPDUs) are the fundamental Layer 2 management frames switches exchange to compute the loop-free topology. Comparing the byte structure of classic 802.1D and 802.1w BPDUs reveals how RSTP achieves rapid convergence without changing the basic frame size.

---

### 1. IEEE 802.1D Classic Configuration BPDU Header

A standard Configuration BPDU is encapsulated directly inside an IEEE 802.3 LLC (Logical Link Control) frame with a payload length of **35 bytes**:

```
+-----------------------------------------------------------------------+
| Field Name               | Size (Bytes) | Description                 |
+--------------------------+--------------+-----------------------------+
| Protocol Identifier      | 2            | Always 0x0000 (IEEE 802.1D) |
| Protocol Version ID      | 1            | 0x00 for Classic STP        |
| BPDU Type                | 1            | 0x00 (Configuration BPDU)   |
| BPDU Flags               | 1            | Topology Change bits only   |
| Root Identifier (Root ID)| 8            | Priority (2B) + MAC (6B)    |
| Root Path Cost           | 4            | Cumulative path cost to Root|
| Bridge Identifier (BID)  | 8            | Priority (2B) + MAC (6B)    |
| Port Identifier (Port ID)| 2            | Port Priority (1B) + Number |
| Message Age              | 2            | Time since Root originated  |
| Max Age                  | 2            | Max time to store BPDU (20s)|
| Hello Time               | 2            | BPDU transmission interval  |
| Forward Delay            | 2            | Listening/Learning time(15s)|
+-----------------------------------------------------------------------+
```
## The 802.1D Flags Byte (1 Byte):
Classic STP only utilizes the two extreme bits of the 8-bit flags field, leaving the middle 6 bits entirely unused:

```
Bit 7      Bit 6      Bit 5      Bit 4      Bit 3      Bit 2      Bit 1      Bit 0
+----------+----------+----------+----------+----------+----------+----------+----------+
|   TCA    | Reserved | Reserved | Reserved | Reserved | Reserved | Reserved |    TC    |
+----------+----------+----------+----------+----------+----------+----------+----------+
```

Bit 7 (TCA - Topology Change Acknowledgment): Set by the upstream switch to confirm receipt of a Topology Change Notification (TCN).
Bit 0 (TC - Topology Change): Set by the Root Bridge to notify all downstream switches to reduce their MAC address table aging timer from 300s to Forward Delay (15s).
---
### 2. IEEE 802.1w Rapid Spanning Tree BPDU Header
IEEE 802.1w introduces the RST BPDU, keeping backwards compatibility with 802.1D frame parsers while utilizing previously reserved fields for deterministic negotiation:

Protocol Version ID: Incremented to 0x02 (RSTP).

BPDU Type: Updated to 0x02 (Rapid / Multiple Spanning Tree).

Version 1 Length (1 Byte): Set to 0x00 (appended to indicate no additional protocol TLVs).

The 802.1w Flags Byte Breakdown (Full 8-Bit Utilization):
Instead of wasting bits, RSTP re-engineers the entire 1-byte Flags field to handle the Proposal / Agreement handshake and active role negotiation:


```
Bit 7      Bit 6      Bit 5      Bit 4      Bit 3      Bit 2      Bit 1      Bit 0
+----------+----------+----------+----------+----------+----------+----------+----------+
|   TCA    |Agreement |Forwarding| Learning |   Port Role (2 bits)| Proposal |    TC    |
+----------+----------+----------+----------+----------+----------+----------+----------+
```

Bit 7 (TCA): Topology Change Acknowledgment.
Bit 6 (Agreement): Sent in response to a Proposal to immediately transition the designated port to Forwarding.
Bit 5 (Forwarding): Set if the sending port is currently in the Forwarding state.
Bit 4 (Learning): Set if the sending port is actively learning MAC addresses.
Bits 3-2 (Port Role): Identifies the exact functional role of the transmitting port:
00 = Unknown
01 = Alternate / Backup Port
10 = Root Port
11 = Designated Port
Bit 1 (Proposal): Transmitted by a Designated port to initiate an active synchronization handshake with the neighboring switch.
Bit 0 (TC): Topology Change bit (RSTP handles TC locally and flushes CAM tables immediately rather than using TCN BPDUs).

### 1. IEEE 802.1D Classic Configuration BPDU Header Fields

| Field Name | Size (Bytes) | Description |
| :--- | :--- | :--- |
| **Protocol Identifier** | 2 | Always `0x0000` (IEEE 802.1D) |
| **Protocol Version ID** | 1 | `0x00` for Classic STP |
| **BPDU Type** | 1 | `0x00` (Configuration BPDU) |
| **BPDU Flags** | 1 | Topology Change bits only (TC / TCA) |
| **Root Identifier (Root ID)** | 8 | Root Priority (2B) + Root MAC Address (6B) |
| **Root Path Cost** | 4 | Cumulative path cost to the Root Bridge |
| **Bridge Identifier (BID)** | 8 | Sender Priority (2B) + Sender MAC Address (6B) |
| **Port Identifier (Port ID)** | 2 | Port Priority (1B) + Interface Number (1B) |
| **Message Age** | 2 | Time elapsed since Root Bridge generated BPDU |
| **Max Age** | 2 | Maximum time to store BPDU before discarding (20s) |
| **Hello Time** | 2 | Periodic BPDU transmission interval (2s) |
| **Forward Delay** | 2 | Time spent in Listening and Learning states (15s) |

---

### 2. IEEE 802.1w (RSTP) BPDU Flags Byte Mapping

| Bit Position | Flag Name | Description |
| :--- | :--- | :--- |
| **Bit 7** | `TCA` | Topology Change Acknowledgment |
| **Bit 6** | `Agreement` | Confirms Proposal to transition Designated port to Forwarding immediately |
| **Bit 5** | `Forwarding` | Set when the transmitting port is in the Forwarding state |
| **Bit 4** | `Learning` | Set when the transmitting port is in the Learning state |
| **Bits 3-2** | `Port Role` | `00`: Unknown \| `01`: Alternate/Backup \| `10`: Root \| `11`: Designated |
| **Bit 1** | `Proposal` | Initiates rapid link synchronization handshake with downstream switch |
| **Bit 0** | `TC` | Topology Change flag (triggers direct MAC address table flushing) |

---

### 3. Key Architectural & Protocol Differences

| Characteristic | IEEE 802.1D (STP / PVST+) | IEEE 802.1w (RSTP / Rapid-PVST+) |
| :--- | :--- | :--- |
| **BPDU Generation** | Only Root Bridge originates; non-roots relay | Every switch originates BPDUs every Hello interval |
| **Missing BPDU Timeout** | 20 seconds (Max Age timer expiration) | 6 seconds (3 consecutive missed Hello intervals) |
| **Topology Change Handling** | Transmits TCN to Root; Root sets TC flag | Switch detecting change directly flushes MAC table & floods TC |
| **Transition Mechanism** | Passive timers (`30 - 50 seconds`) | Active Proposal/Agreement handshake (`< 1 second`) |
| **Port Roles** | Root, Designated, Non-Designated (Blocked) | Root, Designated, Alternate, Backup |
| **Port Operational States** | Disabled, Blocking, Listening, Learning, Forwarding | Discarding, Learning, Forwarding |

---

### Verification & Empirical Validation

#### 1. Verifying Spanning Tree Topology on the Root Bridge (SW1)

```
SW1# show spanning-tree vlan 1

VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     5000.0001.0000
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     5000.0001.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Gi0/0               Desg FWD 4         128.1    P2p 
Gi0/1               Desg FWD 4         128.2    P2p 
Gi0/2               Desg FWD 4         128.3    P2p
```
#### 2. Identifying Root Bridge Information Across the Domain

```
SW4# show spanning-tree root

                                       Root    Hello Max Fwd
Vlan                   Root ID          Cost    Time  Age Dly  Root Port
---------------- -------------------- --------- ----- --- ---  ------------
VLAN0001         32769 5000.0001.0000         4     2   20  15  Gi0/1
```
#### 3. Inspecting Blocked Ports (SW4)

```
SW4# show spanning-tree blockedports

Name                 Blocked Interfaces List
-------------------- ------------------------------------
VLAN0001             Gi0/0, Gi0/2

Number of blocked ports (segments) in the system : 2
```

#### 4. Detailed Interface State Breakdown (SW3 Alternate Port)

```
SW3# show spanning-tree interface GigabitEthernet 0/0 detail

 Port 1 (GigabitEthernet0/0) of VLAN0001 is Alternate Alternate-Blocking 
   Port path cost 4, Port priority 128, Port Identifier 128.1.
   Designated root has priority 32769, address 5000.0001.0000
   Designated bridge has priority 32769, address 5000.0002.0000
   Designated port id is 128.2, designated path cost 4
   Timers: message age 1, forward delay 0, hold 0
   Number of transitions to forwarding state: 0
   Link type is point-to-point by default
   BPDU: sent 0, received 142
```

## Topology Architecture

![Topo](./images/topo.png)

### Interconnection Matrix

| Source Device | Source Port | Target Device | Target Port | Connection Type |
| :--- | :--- | :--- | :--- | :--- |
| **SW1** | `Gi0/0` | **SW2** | `Gi0/0` | Trunk (802.1Q) |
| **SW1** | `Gi0/1` | **SW4** | `Gi0/1` | Trunk (802.1Q) |
| **SW1** | `Gi0/2` | **SW3** | `Gi0/2` | Trunk (802.1Q) |
| **SW2** | `Gi0/1` | **SW3** | `Gi0/0` | Trunk (802.1Q) |
| **SW2** | `Gi0/2` | **SW4** | `Gi0/2` | Trunk (802.1Q) |
| **SW2** | `Gi1/0` | **VPCS5** | `eth0` | Access (VLAN 1) |
| **SW3** | `Gi0/1` | **SW4** | `Gi0/0` | Trunk (802.1Q) |
| **SW4** | `Gi1/0` | **VPCS6** | `eth0` | Access (VLAN 1) |

---

## Configuration Guide

### 1. Initial Configuration

SW1

```
! Baseline Configuration applied to all switches (adjust hostname accordingly)
enable
configure terminal
hostname SW1
no ip domain-lookup

! Disable all unused interfaces to avoid unintended topology interference
interface range GigabitEthernet0/3, GigabitEthernet1/1 - 3
 shutdown
 exit

! Configure active Inter-Switch Links as 802.1Q Trunks
interface range GigabitEthernet0/0 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
 exit

! Console line hardening and productivity settings
line con 0
 exec-timeout 0 0
 logging synchronous
 exit
end
write memory
```

###Configuration Rationale & Best Practices:
- no ip domain-lookup: By default, Cisco IOS treats any typo or unrecognised CLI command as a domain name and attempts a broadcast DNS lookup via port 53. This causes the CLI to freeze for up to 30 seconds while waiting for the DNS resolver to time out. Disabling domain lookup keeps the console responsive.

- exec-timeout 0 0: Sets the console session inactivity timer to infinity (minutes seconds). This prevents the switch from automatically logging out the administrator during extended lab testing and packet capture sessions. (Note: Strictly recommended for lab environments only; production devices should maintain a strict timeout for security).

- logging synchronous: Prevents unsolicited system log messages (syslog notifications, interface up/down alerts) from interrupting and splitting commands currently being typed into the CLI prompt. The IOS redisplays the half-typed command on a clean new line immediately after outputting the system message.

- shutdown on Unused Interfaces: Layer 2 security and operational best practice. Unused switch ports left enabled can cause accidental spanning-tree topology recalibrations if connected incorrectly, or introduce security vulnerabilities (e.g., unauthorized switch attachments).

---
### 2. Default PVST+ Execution & Observations

Under factory default settings, all switches run Cisco PVST+ with the default priority of `32768` (resulting in a Bridge Priority of `32769` when including the Extended System ID for VLAN 1).

![Default STP Topology and Show Commands](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/stp-rstp-basics/images/stp%20-%20bef.png)

#### Baseline MAC & Bridge IDs:
* **SW1:** `32769 : 5000.0001.0000` (Lowest MAC Address $\rightarrow$ **Elected Root Bridge**)
* **SW2:** `32769 : 5000.0002.0000`
* **SW3:** `32769 : 5000.0003.0000`
* **SW4:** `32769 : 5000.0004.0000`

---

### Decision Breakdown (Why Each Port is Forwarding or Blocking):

1. **SW1 (Root Bridge):**
   * Since SW1 possesses the lowest system MAC address (`5000.0001.0000`), it wins the Root Bridge election.
   * All active operational interfaces (`Gi0/0`, `Gi0/1`, `Gi0/2`) assume the **Designated Port (Desg / FWD)** role.

2. **SW2:**
   * **Root Port:** `Gi0/0` (Direct link to SW1 with the lowest cumulative Root Path Cost = `4`).
   * **Designated Ports:** `Gi0/1` (toward SW3), `Gi0/2` (toward SW4), and `Gi1/0` (access port to VPCS5) remain in **FWD** state because SW2 has a lower Bridge ID than SW3 and SW4 on those segments.

3. **SW3:**
   * **Root Port:** `Gi0/2` (Direct link to SW1 with Root Path Cost = `4`).
   * **Designated Port:** `Gi0/1` (toward SW4, because SW3's MAC `5000.0003.0000` is lower than SW4's MAC `5000.0004.0000`).
   * **Alternate / Blocked Port:** `Gi0/0` (**Altn / BLK**) is placed into the Blocking state to prevent a loop with SW2.

4. **SW4:**
   * **Root Port:** `Gi0/1` (Direct link to SW1 with Root Path Cost = `4`).
   * **Alternate / Blocked Ports:**
     * `Gi0/0` (**Altn / BLK**) is blocked toward SW3.
     * `Gi0/2` (**Altn / BLK**) is blocked toward SW2.
   * **Designated Port:** `Gi1/0` (Access port to VPCS6) remains in **FWD** state.

---
### Deep Dive: Analyzing a Non-Root Switch (SW4 Perspective)

To truly understand spanning tree mechanics, we must analyze the protocol from the perspective of a switch that is furthest from the Root Bridge. The output from **SW4** provides a perfect example of a switch aggressively blocking ports to break physical loops.

![SW4 Spanning Tree Output](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/stp-rstp-basics/images/wireshark%20-%20config%20bpdu%20packet%20capt.png)

#### 1. Root ID vs. Bridge ID
The `show spanning-tree` command splits the switch's identity into two distinct sections:
* **Root ID (The Boss):** Displays the information of the elected Root Bridge for the VLAN. SW4 knows the Root has a MAC of `5000.0001.0000` (SW1) and calculates a cumulative **Cost of 4** to reach it. It also identifies that its local interface `Gi0/1` is the best path to get there.
* **Bridge ID (Local Identity):** Displays SW4's own information. It has the default priority (`32769`) but the highest MAC address in the topology (`5000.0004.0000`).

#### 2. Interface State Analysis
Because SW4 has the worst (highest) Bridge ID in the core topology, it loses the segment elections against both SW2 and SW3.

* **`Gi0/1` (Root Port / FWD):** This is SW4's lifeline to the Root Bridge. It connects directly to SW1. The STP cost for a GigabitEthernet link is `4`, making it the lowest-cost path.
* **`Gi0/0` (Alternate / BLK):** This port connects to SW3. On this specific link, SW3 and SW4 compare Bridge IDs. Since SW3 (`5000.0003.0000`) is lower than SW4 (`5000.0004.0000`), SW3 becomes the Designated port for the segment. SW4 is forced to place its port into the **Alternate (Blocking)** state to prevent a loop.
* **`Gi0/2` (Alternate / BLK):** This port connects to SW2. The exact same logic applies: SW2's MAC (`5000.0002.0000`) beats SW4. SW4 must block this port.
* **`Gi1/0` (Designated / FWD):** This interface connects to the end host (`VPCS6`). Since PCs do not generate BPDUs, SW4 automatically wins the election on this collision domain and places the port in a **Forwarding** state.

---

### Wireshark Protocol Analysis & Interoperability Note

#### 1. IEEE 802.1D / PVST+ Configuration BPDU Deep Dive

* **Destination Multicast MAC:** `01:80:c2:00:00:00` (Nearest-Customer-Bridge / Spanning Tree Multicast)
* **Source MAC Address:** `50:00:00:01:00:00` (Originating interface from Root Bridge SW1)
* **Protocol Version Identifier:** `0` (Classic IEEE 802.1D Spanning Tree)
* **BPDU Type:** `0x00` (Configuration BPDU)
* **BPDU Flags (0x00):**
  * Topology Change Acknowledgment (TCA) = No (Bit 7)
  * Topology Change (TC) = No (Bit 0)
* **Root Identifier:** `32768 / 1 / 50:00:00:01:00:00` (Priority 32768 + Sys-ID-Ext 1 = BID 32769)
* **Root Path Cost:** `0` (Originating directly from Root Bridge)
* **Protocol Timers:** Message Age: 0, Max Age: 20s, Hello Time: 2s, Forward Delay: 15s

> **Technical Note on BPDU Multicast Addressing:**
> In Cisco PVST+ and Rapid-PVST+, on the native VLAN (VLAN 1 by default), switches generate two types of BPDUs across trunks:
> 1. An untagged standard IEEE BPDU sent to `0180.c200.0000` for backward compatibility with non-Cisco/CST switches.
> 2. A Cisco-proprietary PVST+ BPDU encapsulated via SNAP and sent to `0100.0ccc.cccd` carrying the VLAN tag/TLV.
> For all non-native/other VLANs, BPDUs are transmitted exclusively to `0100.0ccc.cccd`.

---

### IEEE 802.1w (RSTP / Rapid-PVST+) Rapid Convergence Mechanics

RSTP achieves sub-second convergence primarily through the explicit **Proposal/Agreement handshake** rather than passive timer expiration. However, immediate transition to forwarding depends on specific port classifications:

1. **Edge Ports (PortFast):** Ports connected directly to end-user devices (hosts, servers) that will never receive BPDUs. They transition immediately to the Forwarding state without undergoing negotiation.
2. **Non-Edge / Point-to-Point Links:** Inter-switch links operating in **Full-Duplex** mode. RSTP assumes full-duplex links are point-to-point connections between two switches, enabling the bidirectional Proposal/Agreement handshake. (Half-duplex links revert to shared media mode and fallback to 802.1D timers).
3. **Alternate & Backup Roles:** Switches maintain immediate pre-computed backup paths. If the Root Port fails, an Alternate Port immediately transitions to Root Port without recalculation delays.

---

![PVST+ Configuration BPDU Packet Capture](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/stp-rstp-basics/images/wireshark%20-%20config%20bpdu%20packet%20capt.png)


#### Packet Analysis Breakdown:
* **Destination Multicast MAC:** `01:80:c2:00:00:00` (*Nearest-Customer-Bridge / Spanning Tree Multicast*).
* **Source MAC Address:** `50:00:00:01:00:00` (Originating interface from Root Bridge **SW1**).
* **Protocol Version Identifier:** `0` $\rightarrow$ Indicates Classic IEEE 802.1D Spanning Tree.
* **BPDU Type:** `0x00` $\rightarrow$ Standard **Configuration BPDU** (used for tree topology maintenance).
* **BPDU Flags (`0x00`):** 
  * `Topology Change Acknowledgment (TCA) = No` (Bit 7).
  * `Topology Change (TC) = No` (Bit 0).
  * *Note:* Classic STP utilizes only the two outer bits of the 8-bit flag byte, leaving intermediate bits unused.
* **Root Identifier:** `32768 / 1 / 50:00:00:01:00:00` (Priority `32768` + System ID Extension `1` = Total BID `32769`).
* **Root Path Cost:** `0` (Since the frame originated directly from the Root Bridge).
* **Protocol Timers:** 
  * `Message Age: 0`
  * `Max Age: 20`
  * `Hello Time: 2`
  * `Forward Delay: 15`


---

---

### Verification & Empirical Validation

#### 1. Verifying Spanning Tree Topology on the Root Bridge (SW1)

```
SW1# show spanning-tree vlan 1

VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     5000.0001.0000
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     5000.0001.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Gi0/0               Desg FWD 4         128.1    P2p 
Gi0/1               Desg FWD 4         128.2    P2p 
Gi0/2               Desg FWD 4         128.3    P2p
```
#### 2. Identifying Root Bridge Information Across the Domain

```
SW4# show spanning-tree root
                                       Root    Hello Max Fwd
Vlan                   Root ID          Cost    Time  Age Dly  Root Port
---------------- -------------------- --------- ----- --- ---  ------------
VLAN0001         32769 5000.0001.0000         4     2   20  15  Gi0/1
```


#### 3. Inspecting Blocked Ports (SW4)

```
SW4# show spanning-tree blockedports

Name                 Blocked Interfaces List
-------------------- ------------------------------------
VLAN0001             Gi0/0, Gi0/2

Number of blocked ports (segments) in the system : 2
```

#### 4. Detailed Interface State Breakdown (SW3 Alternate Port)

```
SW3# show spanning-tree interface GigabitEthernet 0/0 detail

 Port 1 (GigabitEthernet0/0) of VLAN0001 is Alternate Alternate-Blocking 
   Port path cost 4, Port priority 128, Port Identifier 128.1.
   Designated root has priority 32769, address 5000.0001.0000
   Designated bridge has priority 32769, address 5000.0002.0000
   Designated port id is 128.2, designated path cost 4
   Timers: message age 1, forward delay 0, hold 0
   Number of transitions to forwarding state: 0
   Link type is point-to-point by default
   BPDU: sent 0, received 142
```
   
---

### Empirical Results Comparison

| Metric | PVST+ (802.1D) | Rapid-PVST+ (802.1w) |
| :--- | :--- | :--- |
| **Observed Downtime / Failover** | ~30 - 50 seconds | ~1 - 3 seconds |
| **Packet Loss Count** | 30+ lost ICMP packets | 1-2 lost ICMP packets |
| **Transition Mechanics** | Timer-driven | Proposal/Agreement |

> **Note on Convergence Measurements:** The failover times and packet loss metrics listed above reflect empirical data captured within this emulated lab environment. In production networks, actual convergence times may vary depending on link speed, hardware platform processing, physical distance (propagation delay), and total network diameter.

---

## Next Steps in This Repository

Now that foundational Layer 2 loop prevention and rapid convergence concepts are established, continue exploring the evolution of Spanning Tree implementation:

* **Next Level Advanced:** [PVST+ & Rapid-PVST+ Per-VLAN Load Balancing]()
* **Enterprise Level:** [MSTP (802.1s) & L2 Hardening Security]()

---

## References & Further Reading

* [IEEE 802.1D Standard: Media Access Control (MAC) Bridges](https://standards.ieee.org/)
* [IEEE 802.1w Standard: Rapid Reconfiguration of Spanning Tree](https://standards.ieee.org/)
* [Cisco Technical Support: Understanding Spanning-Tree Protocol State Transitions](https://www.cisco.com/)
* Cisco Networking Academy: CCNA: Switching, Routing, and Wireless Essentials (SRWE)



