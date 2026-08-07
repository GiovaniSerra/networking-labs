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

![IP INT BR](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/inter-vlan-roas-vs-svi/images/GW%20-%20Sh%20ip%20int%20br.png)

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

![int br - unassigned](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/inter-vlan-roas-vs-svi/images/GW%20-%20Sh%20ip%20int%20br%20exc%20unass.png)

```
GATEWAY# show ip interface brief | exclude unassigned
Interface          IP-Address      OK? Method Status Protocol
FastEthernet0/0.10 192.168.10.254  YES manual up     up
FastEthernet0/0.20 192.168.20.254  YES manual up     up
FastEthernet0/0.30 192.168.30.254  YES manual up     up
```


2. Switch Trunk & MAC Address Table
Validating trunk encapsulation and dynamic MAC learning across VLANs:

![](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/inter-vlan-roas-vs-svi/images/sw%20-%20int%20trunk.png)

```
SW1# show interface trunk
Port      Mode         Encapsulation  Status        Native vlan
Et0/0     on           802.1q         trunking      1

Port      Vlans allowed and active in management domain
Et0/0     1,10,20,30
```

![int status + mac](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/inter-vlan-roas-vs-svi/images/sw%20-%20int%20status%20%2B%20mac%20address%20tab.png)


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

![arp](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/inter-vlan-roas-vs-svi/images/GW%20-%20ARP%20vpc1%2C2%2C3.png)


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

# Part 2: Migration to Switched Virtual Interfaces (SVI)




