# Cisco CCNA Lab: Link Aggregation (LACP, PAgP & Static EtherChannel) - L2/L3 Implementation & Packet Analysis

## Overview
This lab demonstrates the implementation, verification, and packet-level analysis of Link Aggregation Groups (LAG / EtherChannel) operating across Layer 2 and Layer 3 boundaries. The topology implements standard IEEE 802.3ad (LACP), Cisco PAgP, static link bundling, load-balancing hash algorithms, and interaction with the Spanning Tree Protocol (STP).

## Topology

![]()

## Interface Mapping

---

## Device Configurations

SW-ACCESS-01 (L2 Access Switch)
```
! Access peer-link to SW-ACCESS-02
interface range GigabitEthernet1/0 - 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 no shutdown
exit

! Primary uplink to SW-CORE-01
interface range GigabitEthernet0/0 - 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 2 mode active
 no shutdown
exit
```
SW-ACCESS-02 (L2 Access Switch)
```
! Access peer-link to SW-ACCESS-01
interface range GigabitEthernet1/0 - 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode passive
 no shutdown
exit
```

SW-CORE-01 (L3 Multilayer Switch)
```
! Downlink to SW-ACCESS-01 (Layer 2 Trunk)
interface range Ethernet0/0 - 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode passive
 no shutdown
exit

! Core-to-Core Interconnect (Layer 3 Routed Port-Channel)
interface range Ethernet0/2 - 3
 no switchport
 channel-group 3 mode active
 no shutdown
exit

interface Port-channel 3
 no switchport
 ip address 10.0.0.1 255.255.255.252
 no shutdown
exit
```
---

## Verification & State Inspection

### 1. Dual-Stack L2 and L3 EtherChannel Validation

Running show etherchannel summary on SW-CORE-01 verifies both Layer 2 and Layer 3 Port-Channels running simultaneously:
```
SW-CORE-01# show etherchannel summary
Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      N - not in use, no aggregation
        
Group  Port-channel  Protocol  Ports
------+-------------+----------+-----------------------------------------------
1      Po1(SU)         LACP     Et0/0(P)    Et0/1(P)    
3      Po3(RU)         LACP     Et0/2(P)    Et0/3(P)

```

* **Po1 (SU):** S (Layer 2), U (In use) with member ports Et0/0 and Et0/1 flagged as P (bundled in port-channel).
* **Po3 (RU):** R (Layer 3), U (In use) with member ports Et0/2 and Et0/3 bundled.

---

## Deep Dive: LACP Packet Dissection (Wireshark)

Frame captures highlight the structure of IEEE 802.3ad control packets:

* Multicast Destination: 01:80:c2:00:00:02 (Slow Protocols Multicast MAC, EtherType 0x8809, Subtype 0x01).
* Actor State (0x3d - Active):
  * Activity: Active (1)
  * Aggregation: Aggregatable (1)
  * Synchronization: In Sync (1)
  * Collecting: Enabled (1)
  * Distributing: Enabled (1)
* Actor State (0x3c - Passive):
  * Activity: Passive (0)
  * Synchronized, Collecting, and Distributing are enabled after initial negotiation by the active peer.
* TLV Exchange: Both peers exchange System Priority (default 32768), System ID (base MAC address), Port Priority, and Port Key to ensure matching bundle parameters prior to forwarding data frames.

## EtherChannel Load-Balancing Algorithms
The load-balancing hash distribution was tested and configured at the global configuration level:

```
! Inspect default hash configuration (Source XOR Destination IP)
SW-CORE-01# show etherchannel load-balance
EtherChannel Load-Balancing Configuration:
        src-dst-ip

Non-IP: Source XOR Destination MAC address
  IPv4: Source XOR Destination IP address
  IPv6: Source XOR Destination IP address

! Configure source-IP based load balancing
SW-CORE-01(config)# port-channel load-balance src-ip
SW-CORE-01# show etherchannel load-balance
EtherChannel Load-Balancing Configuration:
        src-ip
```

### Spanning Tree Protocol (STP) Integration
The Spanning Tree Protocol treats each bundle as a single point-to-point link:
```
SW-ACCESS-02# show spanning-tree

VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     5000.0001.0000
             Cost        3
             Port        65 (Port-channel1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Po1              Root FWD 3         128.65   P2p
```
* Reduced Cost: The Port-Channel receives an STP cost reflecting aggregated bandwidth (Cost = 3 for 2x Gigabit interfaces).
* Loop Prevention: STP blocks or unblocks redundant Port-Channel logical interfaces as a single unit without shutting down individual member links.

---

## Summary of Findings

LACP Stability: Full convergence requires at least one active peer (active + active or active + passive).
Layer 3 Bundling: Applying no switchport on member ports allows native IP assignment and routed forwarding across the Port-Channel.
Traffic Integrity: Hash algorithms determine link distribution across member ports based on frame headers without packet reordering.

