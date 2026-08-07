# Cisco Lab: Inter-VLAN Routing (Router-on-a-Stick vs. SVI)

This repository contains the configuration, documentation, and verification for an Inter-VLAN routing lab built in EVE-NG. It covers the initial implementation using **Router-on-a-Stick (RoaS)** and documents the step-by-step transition to **Switched Virtual Interfaces (SVIs)** on a Layer 3 Switch.

---

## Environment & Topology

### Emulator and Nodes
* **Emulator:** EVE-NG
* **Router (GATEWAY):** Cisco 7200 Series
* **Switch (SW1):** Cisco IOL Layer 2 / Layer 3
* **Hosts:** VPCS (Virtual PC Simulator)

### Topology Diagram

![Topology](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/inter-vlan-roas-vs-svi/images/top%20-%20roas.png)

---

## Addressing Table

| Device | Interface / Subinterface | IP Address / CIDR | VLAN | Default Gateway |
| :--- | :--- | :--- | :--- | :--- |
| **GATEWAY** | Fa0/0.10 | 192.168.10.254/24 | 10 | N/A |
| **GATEWAY** | Fa0/0.20 | 192.168.20.254/24 | 20 | N/A |
| **GATEWAY** | Fa0/0.30 | 192.168.30.254/24 | 30 | N/A |
| **VPC1** | eth0 | 192.168.10.1/24 | 10 | 192.168.10.254 |
| **VPC2** | eth0 | 192.168.20.1/24 | 20 | 192.168.20.254 |
| **VPC3** | eth0 | 192.168.30.1/24 | 30 | 192.168.30.254 |

---

## Part 1: Router-on-a-Stick (RoaS) Setup

### 1. Router Configuration (GATEWAY)
Enabling the physical interface and configuring 802.1Q subinterfaces for each VLAN.

```text
enable
configure terminal
hostname GATEWAY

interface FastEthernet0/0
 duplex full
 no shutdown

interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.254 255.255.255.0

interface FastEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.254 255.255.255.0

interface FastEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.254 255.255.255.0
end
```

## 2. Switch Configuration (SW1)
Setting up access ports for end devices and a trunk link toward the router.

```
enable
configure terminal
hostname SW1

interface Ethernet0/0
 duplex full
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface Ethernet0/1
 duplex full
 switchport mode access
 switchport access vlan 10

interface Ethernet0/2
 duplex full
 switchport mode access
 switchport access vlan 20

interface Ethernet0/3
 duplex full
 switchport mode access
 switchport access vlan 30
end
```

3. VPCs IP Configuration

VPC1
```
# VPC1
ip 192.168.10.1/24 192.168.10.254
```
VPC2
```
# VPC2
ip 192.168.20.1/24 192.168.20.254
```
VPC3
```
# VPC3
ip 192.168.30.1/24 192.168.30.254
```

---

## Verification & Validation (RoaS)

1. Gateway Subinterface Status
Verifying subinterface operational status and IP assignments:

```
GATEWAY# show ip interface brief
Interface          IP-Address      OK? Method Status                  Protocol
FastEthernet0/0    unassigned      YES unset  up                      up
FastEthernet0/0.10 192.168.10.254  YES manual up                      up
FastEthernet0/0.20 192.168.20.254  YES manual up                      up
FastEthernet0/0.30 192.168.30.254  YES manual up                      up
FastEthernet1/0    unassigned      YES unset  administratively down   down
FastEthernet2/0    unassigned      YES unset  administratively down   down
```

Filtering Output with Pipe (| exclude)
In environments with multiple unused or unassigned interfaces, standard output can become cluttered. The Cisco IOS pipe operator (|) combined with the exclude keyword filters out irrelevant rows, displaying only active subinterfaces with assigned IP addresses:

```
GATEWAY# show ip interface brief | exclude unassigned
Interface          IP-Address      OK? Method Status Protocol
FastEthernet0/0.10 192.168.10.254  YES manual up     up
FastEthernet0/0.20 192.168.20.254  YES manual up     up
FastEthernet0/0.30 192.168.30.254  YES manual up     up
```


2. Switch Trunk & MAC Address Table
Validating trunk encapsulation and dynamic MAC learning across VLANs:

```
SW1# show interface trunk
Port      Mode         Encapsulation  Status        Native vlan
Et0/0     on           802.1q         trunking      1

Port      Vlans allowed and active in management domain
Et0/0     1,10,20,30
```

```
SW1# show mac address-table
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
1       ca07.500d.0000    DYNAMIC     Et0/0
10      0050.7966.6809    DYNAMIC     Et0/1
10      ca07.500d.0000    DYNAMIC     Et0/0
20      0050.7966.680a    DYNAMIC     Et0/2
20      ca07.500d.0000    DYNAMIC     Et0/0
```


3. ICMP Debugging & ARP Cache
Testing inter-VLAN connectivity from end devices and inspecting real-time packet processing on the gateway:


![debug](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/inter-vlan-roas-vs-svi/images/Debug%20ip%20icmp%20%2B%20ping%20gw.png)

```
GATEWAY# debug ip icmp
ICMP packet debugging is on
*Aug  7 14:53:10.731: ICMP: echo reply sent, src 192.168.10.254, dst 192.168.20.1
```

```
GATEWAY# show arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  192.168.10.1             2  0050.7966.6809  ARPA   FastEthernet0/0.10
Internet  192.168.10.254           -  ca07.500d.0000  ARPA   FastEthernet0/0.10
Internet  192.168.20.1             0  0050.7966.680a  ARPA   FastEthernet0/0.20
Internet  192.168.20.254           -  ca07.500d.0000  ARPA   FastEthernet0/0.20
Internet  192.168.30.1             2  0050.7966.6813  ARPA   FastEthernet0/0.30
Internet  192.168.30.254           -  ca07.500d.0000  ARPA   FastEthernet0/0.30

```

## Technical Observations & Cisco IOS Behavior
Best Practice: Explicit VLAN Naming
When assigning access ports to uncreated VLANs, Cisco IOS generates them automatically with default names (VLAN0010, VLAN0020, etc.). Explicitly naming VLANs improves network maintainability:

```
SW1(config)# vlan 10
SW1(config-vlan)# name SALES
SW1(config-vlan)# exit
```
![vlan vendas + expl](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/inter-vlan-roas-vs-svi/images/VLAN%20NAME%20%2B%20expl.png)

### Cisco IOS Behavior: config-vlan Submode Persistence
During testing, creating vlan 200 with the name EX and executing do show vlan br while still inside subconfiguration mode resulted in the VLAN not appearing in the table output:

```
SW1(config)# vlan 200
SW1(config-vlan)# name EX
SW1(config-vlan)# do show vlan br

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active
10   SALES                            active    Et0/1
20   VLAN0020                         active    Et0/2
30   VLAN0030                         active    Et0/3

SW1(config-vlan)# ex
SW1(config)#
SW1(config)# do show vlan br

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active
10   SALES                            active    Et0/1
20   VLAN0020                         active    Et0/2
30   VLAN0030                         active    Et0/3
200  EX                               active  active
```
![](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/inter-vlan-roas-vs-svi/images/vlan%20200.png)

### Technical Explanation:

In Cisco IOS, modifications made in config-vlan mode remain stored in a temporary process buffer.
Executing exit or end exits the submode, triggering the system to commit changes directly into the vlan.dat database.
The VLAN only becomes active in the output of show vlan brief once this database commit occurs.

---
### Wireshark Packet Capture Analysis (802.1Q Validation)

A packet capture on the trunk link confirms the insertion of the 4-byte 802.1Q tag (`0x8100`) with **VLAN ID = 10** for transit ICMP Echo Request packets:

![802.1Q Frame Tagging in Wireshark](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/inter-vlan-roas-vs-svi/images/wireshark%201.png)

**Key Observations from Packet Capture:**
* **EtherType (`0x8100`):** Identifies the presence of the 802.1Q tag header immediately following the Layer 2 Source MAC address.
* **VLAN Identification (`ID: 10`):** Confirms that traffic originated from VLAN 10 (VPC1) is encapsulated before traversing the trunk.
* **Encapsulated Layer 3 Payload (`Type: IPv4 0x0800`):** The original IP packet (`192.168.10.1` -> `192.168.30.1`) remains intact inside the tagged frame.

### 802.1Q Header Breakdown
* **TPID (Tag Protocol Identifier - 16 bits):** Set to `0x8100` to identify the frame as an 802.1Q-tagged Ethernet frame.
* **PCP (Priority Code Point - 3 bits):** Used for Layer 2 Quality of Service (QoS) prioritization (802.1p).
* **DEI (Drop Eligible Indicator - 1 bit):** Indicates frames that may be dropped during network congestion.
* **VLAN ID (VID - 12 bits):** Uniquely identifies the VLAN (supports up to $4094$ VLANs, from $1$ to $4094$).

---

### Packet Flow & De-Encapsulation Process

#### 1. Frame Ingress (Switch Access Port)
* `VPC1` sends an untagged IP packet destined for `VPC2` (`192.168.20.1`) to its Default Gateway (`192.168.10.254`).
* Switch `SW1` receives the untagged frame on interface `Ethernet0/1` (assigned to VLAN 10).

#### 2. Trunk Forwarding & Tag Insertion
* `SW1` inspects its MAC address table and determines the destination MAC (Gateway) is accessible via trunk interface `Ethernet0/0`.
* `SW1` inserts the 802.1Q tag with **VLAN ID = 10** into the frame header and transmits it across the trunk.

#### 3. Router Processing (Subinterface Match)
* The router receives the tagged frame on interface `FastEthernet0/0`.
* The router's parser inspects the 802.1Q header:
  * Reads **VLAN ID = 10**.
  * Matches the tag with subinterface `FastEthernet0/0.10` configured with `encapsulation dot1Q 10`.
* The router strips (de-encapsulates) the 802.1Q tag and processes the IP packet at Layer 3.

#### 4. Inter-VLAN Forwarding & Re-Encapsulation
* The router inspects its routing table and determines the destination IP (`192.168.20.1`) belongs to the directly connected network on subinterface `FastEthernet0/0.20`.
* The router rewrites the Layer 2 header (Source MAC = Router, Destination MAC = VPC2) and inserts a new 802.1Q tag with **VLAN ID = 20**.
* The frame is transmitted back down the trunk to `SW1`.

#### 5. Frame Egress (Destination Access Port)
* `SW1` receives the frame tagged with **VLAN ID = 20** on port `Ethernet0/0`.
* `SW1` checks its MAC table for `VPC2`, strips the 802.1Q tag, and forwards the original untagged Ethernet frame out access port `Ethernet0/2` to `VPC2`.

---

# Part 2: Migration to Switched Virtual Interfaces (SVI)

## Overview & Updated Topology
In this phase, the dedicated router (`GATEWAY`) is decommissioned. Inter-VLAN routing is migrated directly to SW1 using Switched Virtual Interfaces (SVIs) on a Layer 3 Switch, where forwarding is performed by dedicated switching ASICs.

![SVI Topology](images/top-svi.png)

---

## 1. Baseline Verification (Post-Router Removal)
With the router removed and trunking disabled on interface `Et0/0`, `SW1` acts strictly as a Layer 2 switch. 

```
SW1# show ip interface brief
Interface                  IP-Address      OK? Method Status                  Protocol
Ethernet0/0                unassigned      YES unset  up                      up
Ethernet0/1                unassigned      YES unset  up                      up
Ethernet0/2                unassigned      YES unset  up                      up
Ethernet0/3                unassigned      YES unset  up                      up
```

Attempting to ping across VLANs (VPC1 to VPC2) at this stage fails because no default gateway or Layer 3 boundary exists:

```
VPCS> ping 192.168.20.1
host (192.168.10.254) not reachable
```

## 2. SVI Configuration & State Troubleshooting
Configuring SVIs
Creating the virtual interfaces for VLANs 10, 20, and 30 on SW1:

```
enable
configure terminal

interface Vlan10
 ip address 192.168.10.254 255.255.255.0

interface Vlan20
 ip address 192.168.20.254 255.255.255.0

interface Vlan30
 ip address 192.168.30.254 255.255.255.0
```
Cisco IOS Behavior: administratively down State
By default, newly created SVIs remain disabled until explicitly brought up:

```
SW1(config-if)# do show ip interface brief
Interface                  IP-Address      OK? Method Status                  Protocol
Ethernet0/0                unassigned      YES unset  up                      up
Ethernet0/1                unassigned      YES unset  up                      up
Ethernet0/2                unassigned      YES unset  up                      up
Ethernet0/3                unassigned      YES unset  up                      up
Vlan10                     192.168.10.254  YES manual administratively down   down
Vlan20                     192.168.20.254  YES manual administratively down   down
Vlan30                     192.168.30.254  YES manual administratively down   down
```

Executing no shutdown on each SVI transitions their operational status to up/up:

```
SW1(config)# interface range vlan 10, vlan 20, vlan 30
SW1(config-if-range)# no shutdown
```
## 3. Global IP Routing Activation
Even with SVIs in an up/up state, a Cisco Layer 3 switch will not forward packets between subnets until global IP routing is explicitly enabled.

Before running ip routing, ping requests from VPC1 to VPC2 result in a timeout:

```
VPCS> ping 192.168.20.1
192.168.20.1 icmp_seq=1 timeout
192.168.20.1 icmp_seq=2 timeout
192.168.20.1 icmp_seq=3 timeout
192.168.20.1 icmp_seq=4 timeout
192.168.20.1 icmp_seq=5 timeout
```
Enabling global Layer 3 forwarding:

```
SW1(config)# ip routing
```

## 4. Final Verification & Connectivity Test
With global routing active and ARP tables populated, inter-VLAN traffic achieves full connectivity at wire-speed:

```
VPCS> ping 192.168.20.1
84 bytes from 192.168.20.1 icmp_seq=1 ttl=63 time=0.631 ms
84 bytes from 192.168.20.1 icmp_seq=2 ttl=63 time=0.452 ms
84 bytes from 192.168.20.1 icmp_seq=3 ttl=63 time=0.596 ms
84 bytes from 192.168.20.1 icmp_seq=4 ttl=63 time=0.398 ms
84 bytes from 192.168.20.1 icmp_seq=5 ttl=63 time=0.550 ms
```

_*Note: Latency values reflect measurements observed within this EVE-NG virtual lab environment._

---

## Summary Comparison: RoaS vs. SVI

| Feature | Router-on-a-Stick (RoaS) | Switched Virtual Interface (SVI) |
| :--- | :--- | :--- |
| **Hardware Topology** | L2 Switch + External Router | L3 Switch only |
| **Traffic Path** | Sent up the trunk link to router CPU and back | Forwarded internally on switch backplane |
| **Latency** | Higher (~10-25 ms in simulation) | Sub-millisecond (~0.4-0.6 ms) | *Observed latency in this EVE-NG lab*
| **Bandwidth Limits** | Constrained by physical trunk port capacity | Wire-speed ASIC switching capacity |
| **Use Case** | Small networks / Legacy environments | Enterprise Core, Distribution & Data Center |

---

## Deep Dive: Router-on-a-Stick (RoaS)

### Why Router-on-a-Stick?
Router-on-a-Stick is a traditional method used to route traffic between multiple VLANs using a single physical interface on a Layer 2 switch connected to a Layer 3 router. It relies on 802.1Q trunking and logical subinterfaces to segregate and process inter-VLAN traffic without requiring a high-end Layer 3 switch.

### Advantages
* **Cost-Effective:** Allows inter-VLAN routing using low-cost Layer 2 switches and an existing router.
* **Granular Security Control:** Traffic between subnets must pass through the physical router interface, making it easy to apply Access Control Lists (ACLs) and inspect packets.
* **Simplicity:** Easy to set up in small-scale topology labs and branch network implementations.

### Disadvantages
* **Single Point of Failure (SPOF):** The single physical link between the switch and router creates a vulnerability for the entire inter-VLAN infrastructure.
* **Bandwidth Bottleneck:** All inter-VLAN traffic shares the bandwidth of a single physical link (hairpinning (traffic traverses the same physical trunk twice)), leading to network congestion under heavy load.
* **Higher Latency:** Traffic traveling back and forth over a physical medium to be processed by the router's CPU incurs extra delay compared to hardware switching.

### Typical Use Cases
* Small office / home office (SOHO) setups with minimal inter-VLAN traffic.
* Branch networks with standard Layer 2 access switches connected to an edge router.
* Legacy network designs or budget-constrained enterprise environments.

---

## Deep Dive: Switched Virtual Interfaces (SVI)

### Why SVI?
Switched Virtual Interfaces (SVIs) provide Layer 3 routing directly within a Layer 3 switch (Multilayer Switch). Instead of forwarding inter-VLAN traffic out to an external device, the switch processes packets internally on its backplane using dedicated Application-Specific Integrated Circuits (ASICs).

### Advantages
* **Wire-Speed Performance:** Routing occurs in hardware via ASICs, providing extremely high throughput and sub-millisecond latency.
* **Elimination of Traffic Bottlenecks:** Traffic between VLANs does not leave the switch backplane, preserving physical port bandwidth.
* **High Scalability:** Effortlessly handles dense intra-building and data center subnets without performance degradation.
* **Simplified Cabling:** Reduces physical link count by removing the external router link for local subnet gateways.

### Disadvantages
* **Hardware Cost:** Multilayer switches (Layer 3) are significantly more expensive than standard Layer 2 switches.
* **Resource Constraints on Control Plane:** Advanced routing capabilities or complex stateful inspection (like deep packet inspection) may require specialized hardware modules or dedicated firewalls.

### Requirements
* **Layer 3 Capable Hardware:** A switch supporting Layer 3 operations (e.g., Cisco Catalyst 3650/3850/9300 series or IOL L3 image).
* **Global Routing Activation:** Global IP forwarding must be explicitly enabled on the switch via the `ip routing` command.
* **Active VLAN Database:** The corresponding VLAN must exist in the switch database and have at least one active port (or trunk) associated with it for the SVI to remain in an `up/up` operational state.

---

## Best Practices for Inter-VLAN Routing

### 1. VLAN Management & Trunking
* **Explicit VLAN Naming:** Always assign meaningful names to VLANs (`name SALES`, `name MANAGEMENT`) immediately after creation to prevent auto-generated defaults (`VLAN0010`) and simplify network troubleshooting.
* **Prune Unnecessary VLANs:** Restrict allowed VLANs on trunk links using `switchport trunk allowed vlan` to prevent unnecessary broadcast traffic from flooding links.
* **Native VLAN Security:** Never use `VLAN 1` for production or management traffic. Change the native VLAN on trunks to an unused VLAN ID to mitigate VLAN hopping attacks.

### 2. Layer 3 & SVI Deployment
* **Explicit IP Routing Activation:** Always remember that Cisco Layer 3 switches require global `ip routing` to forward packets across subnets, even if SVIs are configured in `up/up` status.
* **Keep-Alive Port Association:** Ensure at least one access port assigned to the VLAN or an active trunk carrying the VLAN exists; otherwise, the SVI will transition to `down/down`.
* **Subnet Alignment:** Maintain a standardized IP addressing schema where VLAN IDs correspond directly to IP subnets (e.g., `VLAN 10` = `192.168.10.0/24`) to simplify network administration.

---

## References & Official Documentation

* [Cisco Networking Academy (NetAcad) - CCNA Enterprise Networking, Security, and Automation](https://www.netacad.com/)
* [Cisco Official Guide: Configuring Inter-VLAN Routing](https://www.cisco.com/c/en/us/support/docs/lan-switching/inter-vlan-routing/41860-howto-L3-intervlanrouting.html)
* [IEEE 802.1Q Standard Overview & Frame Format](https://www.ieee802.org/1/pages/802.1Q.html)
* [Cisco IOS Command Reference - ip routing & interface vlan](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/command/iri-cr-book.html)
