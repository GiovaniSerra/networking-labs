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

Classic STP relies on timer-based state transitions to safely reconverge without creating temporary Layer 2 loops, resulting in relatively slow convergence. Under classic IEEE 802.1D Spanning Tree, worst-case convergence can reach approximately 50 seconds with default timers: **Max Age (20s)** may be required for stale BPDU information to expire, followed by **Listening (15s)** and **Learning (15s)** before the port reaches the Forwarding state.

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

## Topology Architecture

![Topo](./images/topo.png)

### Interconnection Matrix

| Source Device | Source Port | Target Device | Target Port | Connection Type |
| :--- | :--- | :--- | :--- | :--- |
| **SW1** | Gi0/0 | **SW2** | Gi0/0 | Trunk (802.1Q) |
| **SW1** | Gi0/1 | **SW4** | Gi0/1 | Trunk (802.1Q) |
| **SW1** | Gi0/2 | **SW3** | Gi0/2 | Trunk (802.1Q) |
| **SW2** | Gi0/1 | **SW3** | Gi0/0 | Trunk (802.1Q) |
| **SW2** | Gi0/2 | **SW4** | Gi0/2 | Trunk (802.1Q) |
| **SW2** | Gi1/0 | **VPCS5** | eth0 | Access (VLAN 1) |
| **SW3** | Gi0/1 | **SW4** | Gi0/0 | Trunk (802.1Q) |
| **SW4** | Gi1/0 | **VPCS6** | eth0 | Access (VLAN 1) |

---

## Deep Dive: BPDU Frame Structure & Header Evolution

Bridge Protocol Data Units (BPDUs) are the fundamental Layer 2 management frames switches exchange to compute the loop-free topology. Comparing the byte structure of classic 802.1D and 802.1w BPDUs reveals how RSTP achieves rapid convergence without changing the basic frame size.

---

### 1. IEEE 802.1D Classic Configuration BPDU Header
A standard Configuration BPDU is encapsulated directly inside an IEEE 802.3 LLC (Logical Link Control) frame with a payload length of 35 bytes:

| Field Name | Size (Bytes) | Description |
| :--- | :--- | :--- |
| Protocol Identifier | 2 | Always 0x0000 (IEEE 802.1D) |
| Protocol Version ID | 1 | 0x00 for Classic STP |
| BPDU Type | 1 | 0x00 (Configuration BPDU) |
| BPDU Flags | 1 | Topology Change bits only (TC / TCA) |
| Root Identifier (Root ID) | 8 | Root Priority (2B) + Root MAC Address (6B) |
| Root Path Cost | 4 | Cumulative path cost to the Root Bridge |
| Bridge Identifier (BID) | 8 | Sender Priority (2B) + Sender MAC Address (6B) |
| Port Identifier (Port ID) | 2 | Port Priority (1B) + Interface Number (1B) |
| Message Age | 2 | Time elapsed since Root Bridge generated BPDU |
| Max Age | 2 | Maximum time to store BPDU before discarding (20s) |
| Hello Time | 2 | Periodic BPDU transmission interval (2s) |
| Forward Delay | 2 | Time spent in Listening and Learning states (15s) |

#### The 802.1D Flags Byte (1 Byte)
Classic STP utilizes only the two extreme bits of the 8-bit flags field, leaving the middle 6 bits entirely unused:

```
Bit 7      Bit 6      Bit 5      Bit 4      Bit 3      Bit 2      Bit 1      Bit 0
+----------+----------+----------+----------+----------+----------+----------+----------+
|   TCA    | Reserved | Reserved | Reserved | Reserved | Reserved | Reserved |    TC    |
+----------+----------+----------+----------+----------+----------+----------+----------+
```

Bit 7 (TCA - Topology Change Acknowledgment): Set by the upstream switch to confirm receipt of a Topology Change Notification (TCN).
Bit 0 (TC - Topology Change): Set by the Root Bridge to notify downstream switches to reduce their MAC address table aging timer from 300s to Forward Delay (15s).

---

### 2. IEEE 802.1w Rapid Spanning Tree BPDU Header
IEEE 802.1w introduces the RST BPDU while maintaining backward compatibility with 802.1D frame parsers:

- Protocol Version ID: Incremented to 0x02 (RSTP).
- BPDU Type: Updated to 0x02 (Rapid / Multiple Spanning Tree).
- Version 1 Length (1 Byte): Set to 0x00 (appended to indicate no additional protocol TLVs).


#### The 802.1w Flags Byte Breakdown (Full 8-Bit Utilization)
Instead of leaving bits reserved, RSTP re-engineers the entire 1-byte Flags field to handle the Proposal / Agreement handshake and active role negotiation:

```
Bit 7      Bit 6      Bit 5      Bit 4      Bit 3      Bit 2      Bit 1      Bit 0
+----------+----------+----------+----------+----------+----------+----------+----------+
|   TCA    |Agreement |Forwarding| Learning | Port Role (2 bits)  | Proposal |    TC    |
+----------+----------+----------+----------+----------+----------+----------+----------+
```

| Bit Position | Flag Name | Description |
| :--- | :--- | :--- |
| **Bit 7** | TCA | Topology Change Acknowledgment |
| **Bit 6** | Agreement | Confirms Proposal to transition Designated port to Forwarding immediately |
| **Bit 5** | Forwarding | Set when the transmitting port is in the Forwarding state |
| **Bit 4** | Learning | Set when the transmitting port is in the Learning state |
| **Bits 3-2** | Port Role | `00`: Unknown \| `01`: Alternate/Backup \| `10`: Root \| `11`: Designated |
| **Bit 1** | Proposal | Initiates rapid link synchronization handshake with downstream switch |
| **Bit 0** | TC | Topology Change flag (triggers direct MAC address table flushing and fast TC flooding) |

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
| **Transition Mechanism** | Timer-based state transitions (Listening/Learning) | Active Proposal/Agreement and rapid role transitions |
| **Port Roles** | Root, Designated, Non-Designated (Blocked) | Root, Designated, Alternate, Backup |
| **Port Operational States** | Disabled, Blocking, Listening, Learning, Forwarding | Discarding, Learning, Forwarding |

---

## Configuration Guide

### 1. Initial Configuration

SW1

Baseline Configuration applied to all switches
```
enable
configure terminal
hostname SW1
no ip domain-lookup
```

Disable all unused interfaces to avoid unintended topology interference
```
interface range GigabitEthernet0/3, GigabitEthernet1/1 - 3
 shutdown
 exit
```

Configure active Inter-Switch Links as 802.1Q Trunks
```
interface range GigabitEthernet0/0 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
 exit
```

Console line productivity settings
```
line con 0
 exec-timeout 0 0
 logging synchronous
 exit
end
write memory
```



---

### Configuration Rationale & Best Practices:
- no ip domain-lookup: Prevents Cisco IOS from interpreting unrecognized CLI commands as hostnames and attempting DNS resolution via UDP port 53, which freezes the console.

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

![SW4 Spanning Tree Output](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/stp-rstp-basics/images/Expl%20sw4%20-%20stp%20outp.png)

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

![PVST+ Configuration BPDU Packet Capture](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/stp-rstp-basics/images/wireshark%20-%20config%20bpdu%20packet%20capt.png)

#### Packet Analysis Breakdown:
* **Destination Multicast MAC:** `01:80:c2:00:00:00` (*Nearest-Customer-Bridge / Spanning Tree Multicast*).
* **Source MAC Address:** `50:00:00:01:00:00` (Originating interface from Root Bridge **SW1**).
* **Protocol Version Identifier:** `0` $\rightarrow$ Indicates Classic IEEE 802.1D Spanning Tree.
* **BPDU Type:** `0x00` $\rightarrow$ Standard **Configuration BPDU** (used for tree topology maintenance).
* **BPDU Flags (`0x00`):** * `Topology Change Acknowledgment (TCA) = No` (Bit 7).
  * `Topology Change (TC) = No` (Bit 0).
  * *Note:* Classic STP utilizes only the two outer bits of the 8-bit flag byte, leaving intermediate bits unused.
* **Root Identifier:** `32768 / 1 / 50:00:00:01:00:00` (Priority `32768` + System ID Extension `1` = Total BID `32769`).
* **Root Path Cost:** `0` (Since the frame originated directly from the Root Bridge).
* **Protocol Timers:** * `Message Age: 0`
  * `Max Age: 20`
  * `Hello Time: 2`
  * `Forward Delay: 15`

---

## Rapid PVST+ (RSTP) Implementation & Root Bridge Manipulation

### 1. Migrating to Rapid-PVST+
By default, Cisco Catalyst switches run PVST+ (IEEE 802.1D). To achieve sub-second convergence, all switches in the Layer 2 domain must be migrated to Rapid-PVST+ (IEEE 802.1w).

#### Configuration (Apply to all switches: SW1, SW2, SW3, SW4)
```
configure terminal
spanning-tree mode rapid-pvst
end
write memory
```

---

#### Verification on SW1

```
SW1# show spanning-tree summary
Switch is in rapid-pvst mode
Root bridge for: VLAN0001
Extended system ID           is enabled
Portfast Default             is disabled
Portfast Edge BPDU Guard Default is disabled
Portfast Edge BPDU Filter Default is disabled
Loopguard Default            is disabled
PVST Simulation              is enabled
Etherchannel misconfig guard is enabled
```
---
#### Verification on SW2

```
SW2# show spanning-tree vlan 1

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    4097
             Address     5000.0002.0000
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4097   (priority 4096 sys-id-ext 1)
             Address     5000.0002.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Gi0/0               Desg FWD 4         128.1    P2p 
Gi0/1               Desg FWD 4         128.2    P2p 
Gi0/2               Desg FWD 4         128.3    P2p 
Gi1/0               Desg FWD 4         128.4    P2p Edge
```

---

### 2. Automated Root Bridge Manipulation (Root Primary & Secondary Macro)
Cisco IOS provides dynamic macro commands to automatically adjust bridge priority without manually calculating the numerical value:

* - **root primary:** Forces the switch to become the Root Bridge. If the current root priority is greater than 24576, the switch sets its priority to 24576. If the current root priority is 24576 or lower, the switch sets its own priority to 4096 less than the current root priority.  
* - **root secondary:** Sets the switch priority to 28672, positioning it as the immediate backup if the primary Root Bridge fails.

#### Setting SW1 as Primary Root and SW2 as Secondary Root

Configuration on SW1 (Primary Root)
```
configure terminal
spanning-tree vlan 1 root primary
end
write memory
```


Configuration on SW2 (Secondary Root)
```
configure terminal
spanning-tree vlan 1 root secondary
end
write memory
```

---
Verification on SW1 (Primary)

```
SW1# show spanning-tree vlan 1 | include Priority
  Root ID    Priority    24577
  Bridge ID  Priority    24577  (priority 24576 sys-id-ext 1)
```

Verification on SW2 (Secondary)

```
SW2# show spanning-tree vlan 1 | include Priority
  Root ID    Priority    24577
  Bridge ID  Priority    28673  (priority 28672 sys-id-ext 1)
```

---

### 3. Manual Root Bridge Configuration (Explicit Priority)
By default, the bridge priority is 32768. The extended system ID adds the VLAN ID (VLAN 1 = 32769). Priority values must be configured in multiples of 4096 (0, 4096, 8192, ..., 32768 (default), ..., 61440).

To deterministically make a specific switch (e.g., SW2) the Root Bridge with the lowest priority:

Configuration on SW2
```
configure terminal
spanning-tree vlan 1 priority 4096
end
write memory
```

### 4. Tuning Spanning Tree Timers (Hello Time, Forward Delay & Max Age)
Spanning Tree timers control the transmission frequency of BPDUs and the transition intervals between port states. By default, IEEE 802.1D / PVST+ uses the following baseline timers:

* - **Hello Time:** 2 seconds (the periodic interval at which the Root Bridge generates Configuration BPDUs).
* - **Forward Delay:** 15 seconds (the time spent in both the Listening and Learning states before reaching Forwarding).
* - **Max Age:** 20 seconds (the maximum duration a switch stores BPDU information before discarding it due to inactivity).

#### Operational Rules and Scope
Configured Strictly on the Root Bridge: Spanning Tree timers must be altered only on the active Root Bridge. Non-root switches dynamically inherit and enforce the operational timers advertised within the Root Bridge's BPDUs to maintain domain-wide consistency.

Network Diameter Impact: Manually changing individual timer parameters can lead to transient bridging loops or false-positive topology recalculations in large networks. Cisco strongly recommends tuning timers using the automated diameter macro instead of setting manual values.

#### Method 1: Recommended Diameter-Based Tuning (Cisco Best Practice)
The diameter macro automatically recalculates and applies optimized Hello Time, Forward Delay, and Max Age values based on the maximum number of switch hops between any two endpoints (default network diameter is 7):

```
SW1(config)# spanning-tree vlan 1 root primary diameter 4
```

#### Method 2: Manual Per-VLAN Timer Configuration
If specific custom timer values are strictly required, configure them directly on the designated Root Bridge:

```
SW1# configure terminal
SW1(config)# spanning-tree vlan 1 hello-time 1
SW1(config)# spanning-tree vlan 1 forward-time 10
SW1(config)# spanning-tree vlan 1 max-age 14
SW1(config)# end
SW1# write memory
```

##### Valid Cisco IOS configurable ranges:
* - **hello-time:** 1 to 10 seconds
* - **forward-time:** 4 to 30 seconds
* - **max-age:** 6 to 40 seconds
 
### Verification Commands
To confirm that the custom timers are active on the Root Bridge and propagated throughout the Layer 2 domain:

```
SW1# show spanning-tree vlan 1
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    24577
             Address     5000.0001.0000
             This bridge is the root
             Hello Time   1 sec  Max Age 14 sec  Forward Delay 10 sec

  Bridge ID  Priority    24577  (priority 24576 sys-id-ext 1)
             Address     5000.0001.0000
             Hello Time   1 sec  Max Age 14 sec  Forward Delay 10 sec
             Aging Time  300 sec
```

On downstream non-root switches (such as SW2), verify that the Root ID section reflects the updated timers received via upstream BPDUs, while the local Bridge ID section retains default values until that switch becomes the root:

```
SW2# show spanning-tree vlan 1 | include (Root ID|Bridge ID|Hello Time)
  Root ID    Priority    24577
             Hello Time   1 sec  Max Age 14 sec  Forward Delay 10 sec
  Bridge ID  Priority    32769
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
```

---

### Empirical Results Comparison


### Empirical Results Comparison

| Metric | PVST+ (802.1D) | Rapid-PVST+ (802.1w) |
| :--- | :--- | :--- |
| **Observed Downtime / Failover** | ~30 - 50 seconds | ~1 - 3 seconds |
| **Packet Loss Count** | 30+ lost ICMP packets | 1 - 2 lost ICMP packets |
| **Transition Mechanics** | Timer-driven | Proposal / Agreement |

> **Note on Convergence Measurements:** The failover times and packet loss metrics listed above reflect empirical data captured within this emulated lab environment. In production networks, actual convergence times may vary depending on link speed, hardware platform processing, physical distance (propagation delay), and total network diameter.

---

## Spanning Tree Toolkit (STP Security)

### Spanning Tree Toolkit: Theoretical Deep Dive

The standard Spanning Tree Protocol design inherently trusts every device connected to the network topology. If any interface receives a Bridge Protocol Data Unit (BPDU), the switch processes it and dynamically shifts its roles and port states to adapt to what it assumes is a valid network alteration. This open architecture introduces critical operational vulnerabilities: misconfigurations (such as connecting a generic consumer switch to a wall jack) or malicious activities (such as executing an STP spoofing attack) can force a complete root bridge recalculation, degrading performance or creating active bridging loops. 

To mitigate these architecture risks, the Spanning Tree Toolkit offers granular, proactive security features to enforce topology boundaries and guard the active forwarding paths.

---

### 1. BPDU Guard: Enforcing the Access Layer Boundary

In a well-designed hierarchical network, access layer ports configured with **PortFast** (or designated as Edge Ports in RSTP) connect exclusively to end hosts like workstations, IP phones, and printers. These end devices operate at Layer 3 and above; they do not generate or expect Layer 2 topology management frames. 

**BPDU Guard** acts as an enforcement mechanism on these edge interfaces. When enabled, the switch port continues its fast-forwarding behavior, but the background processes constantly monitor the ingress queue for BPDUs. If a user connects a rogue switch, a bridging router, or runs a software-based bridge simulator on a host, a BPDU will enter the interface. 

Upon receiving even a single standard or configuration BPDU, BPDU Guard immediately steps in to isolate the threat. It transitions the operational state of the interface into **`err-disabled`** (error-disabled), effectively shutting down the physical link down to the hardware level. This action drops all user traffic and cuts off potential loop paths or spoofing vectors instantly. To recover the port, an administrator must either manually issue a `shutdown` followed by a `no shutdown` on the interface CLI or configure an automated `errdisable recovery` timer.

#### Configuration

##### Global Configuration (Applies to all PortFast-enabled ports automatically)
```
configure terminal
spanning-tree portfast bpduguard default
end
write memory
```

##### Interface-Level Configuration

```
configure terminal
interface GigabitEthernet1/0
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
end
write memory
```


##### Verification

```
SW1# show spanning-tree summary
Switch is in rapid-pvst mode
Root bridge for: VLAN0001
Extended system ID           is enabled
Portfast Default             is enabled
Portfast Edge BPDU Guard Default is enabled
Portfast Edge BPDU Filter Default is disabled
Loopguard Default            is disabled
PVST Simulation              is enabled
Etherchannel misconfig guard is enabled
```

```
SW1# show interfaces status err-disabled
Port      Name  Status       Reason               Err-disabled Vlans
Gi1/0           err-disabled bpduguard

```

---

### 2. BPDU Filter: Controlling BPDU Propagation
While BPDU Guard disables an interface upon detecting an anomaly, BPDU Filter alters the actual transmission mechanics of the protocol. It allows administrators to stop BPDUs from leaving or entering an interface, but its operational behavior shifts drastically depending on whether it is activated globally or applied explicitly at the interface level.

* - **Global Configuration (PortFast Default):** When enabled globally, the feature integrates smoothly with edge behaviors. When a link state transitions to up, the interface transmits exactly 10 BPDUs to probe the segment for other switches. If no return BPDUs are detected, the port stops transmitting management frames completely, reducing unnecessary CPU utilization and link overhead. However, if a BPDU is ever received during operations, the port instantly disables BPDU Filter, strips its PortFast edge status, and returns to a traditional, fully reactive spanning-tree state.

* - **Interface-Level Configuration:** This is an explicit override that completely removes the interface from the Spanning Tree instance. The port sends zero BPDUs and silently drops any inbound BPDUs without taking any protective action (like err-disabling). Warning: Because this effectively blinds the switch to any downstream topology loops, using interface-level BPDU Filter on cross-connected switch links will cause a catastrophic, uncontained broadcast storm.


#### Configuration
##### Global Configuration (Recommended for edge environments)
```
configure terminal
spanning-tree portfast bpdufilter default
end
write memory
```

##### Interface-Level Configuration
```
configure terminal
interface GigabitEthernet1/0
 spanning-tree bpdufilter enable
end
write memory
```

##### Verification
```
SW1# show spanning-tree summary
Switch is in rapid-pvst mode
Root bridge for: VLAN0001
Extended system ID           is enabled
Portfast Default             is enabled
Portfast Edge BPDU Guard Default is disabled
Portfast Edge BPDU Filter Default is enabled
Loopguard Default            is disabled
```

```
SW1# show running-config interface GigabitEthernet 1/0
Building configuration...

Current configuration : 132 bytes
!
interface GigabitEthernet1/0
 switchport mode access
 spanning-tree portfast
 spanning-tree bpdufilter enable
end
```

---
### 3. Root Guard: Protecting the Root Bridge Authority
The Root Bridge election is purely deterministic, dictated by the lowest Bridge ID (BID). If an administrator leaves the core switches at default priority parameters, any newly provisioned access switch or foreign device possessing a marginally lower factory MAC address will automatically usurp the Root Bridge role. This structural shift forces the entire enterprise network to reroute traffic through an inadequate, lower-capacity access device, collapsing the optimized distribution forwarding tree.

**Root Guard** enforces a top-down structural hierarchy. It is deployed exclusively on downstream-facing **Designated Ports** (e.g., core-to-distribution or distribution-to-access links). Root Guard allows the port to transmit and process local BPDUs normally, provided they are inferior to the current Root Bridge.

If a downstream device transmits a superior BPDU (claiming a better priority or lower MAC address), Root Guard refuses to relay the configuration frame or accept the new root identity. Instead, it instantly places the local interface into a **'root-inconsistent'** state. This specialized state acts as a structural block: it halts the forwarding of user data frames and drops incoming control packets, completely isolating the rogue superior switch. Unlike BPDU Guard, Root Guard does not down the link; it continuously listens. The moment the downstream device stops advertising the superior path, the port automatically moves through standard convergence states back into Forwarding.

#### Configuration (Applied per interface on Designated Ports)
```
configure terminal
interface range GigabitEthernet0/1 - 2
 spanning-tree guard root
end
write memory
```

#### Verification
```
SW1# show spanning-tree inconsistentports

   Name                 Interface              Inconsistency
   -------------------- ---------------------- ------------------
   VLAN0001             GigabitEthernet0/1     Root Inconsistent

Number of inconsistent ports (segments) in the system : 1
```

```
SW1# show spanning-tree interface GigabitEthernet 0/1 detail
 Port 2 (GigabitEthernet0/1) of VLAN0001 is Designated Root-Inconsistent
   Port path cost 4, Port priority 128, Port Identifier 128.2.
   Designated root has priority 24577, address 5000.0001.0000
   Designated bridge has priority 24577, address 5000.0001.0000
   Designated port id is 128.2, designated path cost 0
   Timers: message age 0, forward delay 0, hold 0
   Number of transitions to forwarding state: 1
   Root guard is enabled on the interface
```

---

### 4. Loop Guard: Mitigating Unidirectional Failures
Modern networks rely heavily on fiber-optic links and high-speed transceivers. These links can suffer from a structural failure known as a unidirectional link condition, where the physical path breaks in one direction (e.g., RX breaks while TX remains functional). If a blocking/alternate port on a downstream switch stops receiving periodic BPDUs due to a unilateral fiber cut upstream, it assumes the link is free of loops. Once its Max Age timer expires, it transitions through Listening and Learning directly into a Designated Forwarding state. Because the reverse path is still physically open, this creates an active, silent Layer 2 loop.

**Loop Guard** addresses this by tracking the continuous arrival of BPDUs on non-designating interfaces (**Root Ports** and **Alternate Ports**). It introduces a state check: if BPDUs disappear on a monitored port without a corresponding link-down notification, Loop Guard prevents the interface from transitioning to a forwarding role.

Instead of moving toward a Designated status, the switch isolates the port by placing it into a **'loop-inconsistent'** state. This state completely blocks Layer 2 traffic forwarding on that interface, ensuring a unidirectional link failure cannot break the active loop-free logical path. Like Root Guard, this feature features automated recovery: the moment a valid, periodic BPDU is successfully received on the interface again, the inconsistency flag clears, and the port returns to normal operation.

#### Configuration
Global Configuration (Recommended for all non-edge point-to-point links)
```
configure terminal
spanning-tree loopguard default
end
write memory
```

#### Interface-Level Configuration
```
configure terminal
interface GigabitEthernet0/0
 spanning-tree guard loop
end
write memory
```

#### Verification
```
SW2# show spanning-tree summary
Switch is in rapid-pvst mode
Root bridge for: none
Extended system ID           is enabled
Portfast Default             is disabled
Portfast Edge BPDU Guard Default is disabled
Portfast Edge BPDU Filter Default is disabled
Loopguard Default            is enabled
PVST Simulation              is enabled
Etherchannel misconfig guard is enabled
```

```
SW2# show spanning-tree inconsistentports

   Name                 Interface              Inconsistency
   -------------------- ---------------------- ------------------
   VLAN0001             GigabitEthernet0/0     Loop Inconsistent

Number of inconsistent ports (segments) in the system : 1
```

---

## Troubleshooting Spanning Tree Protocol Issues

### 1. Protocol Mismatch: STP (802.1D) vs. RSTP (802.1w)
When one switch in the topology runs legacy STP (or standard PVST+) while the remaining switches run RSTP (or Rapid-PVST+), the protocol boundary defaults to backward compatibility mode. 

#### Operational Impact
RSTP features backward compatibility by falling back to 802.1D mechanics on interfaces where legacy BPDUs are detected. The affected interface falls back to 802.1D-compatible behavior and no longer uses the full RSTP Proposal/Agreement mechanism. As a result, convergence on that segment can become significantly slower and may follow the legacy Listening and Learning timer sequence.

#### Commands & Verification
To detect an active protocol fallback on an interface, inspect the spanning-tree link type output.

```
SW1# show spanning-tree vlan 1 interface GigabitEthernet 0/0
Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Gi0/0               Root FWD 4         128.1    P2p Peer(STP)
```

* **Note: The Peer(STP) designation explicitly signals that the local RSTP switch has detected 802.1D BPDUs and downgraded its transition mechanics on that link.**

### 2. Protocol Mismatch Across Different VLANs
A silent and complex issue occurs when a switch runs legacy PVST+ for one specific VLAN while running Rapid-PVST+ for others, or when a native VLAN mismatch exists across an inter-switch trunk.

#### Operational Impact
If a switch processes standard PVST+ timers for an isolated VLAN while neighboring switches expect sub-second RSTP synchronization, standard frame forwarding can resume before the legacy switch finishes its 30-second timer-based transition. This timing desynchronization results in intermittent connectivity loss, unidirectional frame loops, and temporary MAC address table instability restricted to that specific VLAN.

#### Commands & Verification
Identify the protocol operational differences by verifying the spanning-tree execution mode per VLAN.

```
SW1# show spanning-tree vlan 10 | include protocol
  Spanning tree enabled protocol ieee

SW1# show spanning-tree vlan 20 | include protocol
  Spanning tree enabled protocol rstp
```
* **Note: protocol ieee indicates standard 802.1D (PVST+), whereas protocol rstp confirms the execution of 802.1w (Rapid-PVST+).**

#### 3. Misconfigurations & Issues with Loop Guard
Loop Guard prevents bridging loops caused by unidirectional link failures by keeping an alternate or root port blocked when periodic BPDUs disappear.

#### Operational Impact
If an administrator incorrectly enables Loop Guard on a Designated Port that naturally transmits BPDUs rather than receiving them, the configuration remains inactive. Conversely, if legitimate network congestion or high control-plane CPU utilization drops or delays three consecutive inbound BPDUs on a valid Root Port, Loop Guard triggers a false positive, locking the interface into a `loop-inconsistent` state and breaking valid backup paths.

#### Commands & Verification

```
SW2# show spanning-tree inconsistentports

   Name                 Interface              Inconsistency
   -------------------- ---------------------- ------------------
   VLAN0001             GigabitEthernet0/0     Loop Inconsistent
```
* **Note:** Once the upstream neighbor recovers control-plane stability and resumes transmitting periodic, valid BPDUs, Loop Guard automatically restores the port to its normal STP operational state without requiring manual intervention.

### 4. Misconfigurations & Issues with Root Guard
Root Guard prevents downstream devices from forcing a Root Bridge recalculation by ignoring superior BPDUs on designated interfaces.

#### Operational Impact
If Root Guard is accidentally applied to the legitimate path facing the intended core Root Bridge, the port detects the core's superior BPDUs and locks up into a root-inconsistent blocking state. This breaks network transit paths and splits the Spanning Tree domain.

```
SW3# show spanning-tree inconsistentports

   Name                 Interface              Inconsistency
   -------------------- ---------------------- ------------------
   VLAN0001             GigabitEthernet0/1     Root Inconsistent
```

Verify if Root Guard was applied to an incorrect path by looking at the detailed interface configuration:

```
SW3# show spanning-tree interface GigabitEthernet 0/1 detail
   Port 2 (GigabitEthernet0/1) of VLAN0001 is Designated Root-Inconsistent
   Root guard is enabled on the interface
```

### 5. Misconfigurations & Issues with BPDU Filter
BPDU Filter stops a switch from transmitting or receiving BPDUs.

#### Operational Impact
When enabled explicitly at the interface level **(spanning-tree bpdufilter enable)**, the switch completely disables STP on that port. If this interface is accidentally cross-connected to another operational switch, the link forms a silent physical path with no loop protection. The switches will forward data frames unconditionally, resulting in an uncontained, catastrophic broadcast storm that can saturate interface links and freeze management access.

#### Commands & Verification
Because the port stops processing BPDUs entirely, it will not log an inconsistency or error-disabled state. You must find the interface-level misconfiguration by inspecting the active operational configuration or checking for high interface utilization.

```
SW1# show running-config interface GigabitEthernet 1/0
interface GigabitEthernet1/0
 spanning-tree bpdufilter enable

SW1# show interfaces GigabitEthernet 1/0 | include input rate|output rate
  5 minute input rate 998241000 bits/sec, 124310 packets/sec
  5 minute output rate 999124000 bits/sec, 124402 packets/sec
```

### 6. Misconfigurations & Issues with BPDU Guard
BPDU Guard error-disables an access interface the moment any inbound BPDU is detected.

#### Operational Impact
The primary issue stems from connecting an authorized infrastructure device (such as a downstream switch, an IP phone with an internal switch, or a hypervisor running virtual switches) to a port where an administrator left **spanning-tree bpduguard enable** active. The incoming topology or management frames immediately trip the feature, placing the port into **err-disabled** and cutting off connection for all downstream end users.

#### Commands & Verification
```
SW1# show interfaces status err-disabled
Port      Name  Status       Reason               Err-disabled Vlans
Gi1/0           err-disabled bpduguard
```

To resolve the issue permanently, remove the structural guard or move the interface configuration to an inter-switch trunk. To recover the port interface state automatically without a manual administrative shutdown, configure the automatic recovery sequence:
```
configure terminal
errdisable recovery cause bpduguard
errdisable recovery interval 30
end
write memory
```


### 7. Other Common Spanning Tree Failures

#### Duplex Mismatch Creating False Forwards
When one side of an inter-switch trunk is hardcoded to Full-Duplex and the other side defaults to Half-Duplex, the half-duplex side uses Carrier Sense Multiple Access with Collision Detection (CSMA/CD) to listen before transmitting. If the full-duplex side transmits data continuously, the half-duplex switch experiences constant collisions and fails to receive inbound BPDUs. Its Max Age timer eventually expires, causing it to incorrectly transition its alternate port into a Designated Forwarding state, creating a devastating data loop.

Duplex mismatches can also interfere with BPDU delivery and STP operation. Therefore, both ends of an inter-switch link should use compatible speed and duplex settings, preferably through consistent auto-negotiation or explicit matching configuration.

#### EtherChannel Misconfigurations
If a bundle of physical links is cross-connected between two switches without an active aggregation protocol (such as LACP or PAGP), Spanning Tree treats each physical link as an independent operational path. Because the links are physically bundled on one side but unaggregated on the control plane, BPDUs loop back into adjacent links of the same switch, resulting in severe MAC table flapping and constant topology alterations.

#### Verification

```
SW1# show spanning-tree summary | include Etherchannel
  Etherchannel misconfig guard is enabled
```

```
SW1# show interfaces status err-disabled
Port      Name  Status       Reason               Err-disabled Vlans
Gi0/1           err-disabled channel-misconfig
```

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



