# DHCP Snooping Lab - Rogue DHCP Attack and Mitigation

## Overview

This lab demonstrates how a network can be impacted by a rogue DHCP server and how DHCP Snooping can be used to mitigate this threat.

The scenario simulates a typical Layer 2 network where an unauthorized device attempts to provide DHCP services, leading to incorrect IP configuration on clients. The mitigation is implemented using DHCP Snooping on the switches.

---

## Objectives

* Validate normal DHCP operation
* Simulate a rogue DHCP attack
* Observe the impact on clients
* Implement DHCP Snooping
* Verify that the attack is mitigated

---

## Topology

* 1 Router (R1) - Legitimate DHCP Server
* 1 Router (ATTACKER) - Rogue DHCP Server
* 2 Switches (SW1, SW2)
* 4 Clients (VPCS)

Logical structure:

R1 → SW1 → SW2
Clients distributed across both switches
Attacker connected to SW2

![Topology](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/topology.png)

---

## Network Design

* VLAN: 10
* Network: 192.168.10.0/24

All devices operate in the same broadcast domain to allow DHCP behavior and attack simulation.

---

## Devices and Images

* Router: 7206 (Dynamips)
* Switches: IOSvL2
* Clients: VPCS
* Attacker: Router (simulating rogue DHCP server)

---

## Baseline Configuration

### R1 - DHCP Server

```
conf t
hostname R1

interface f0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown

ip dhcp excluded-address 192.168.10.1 192.168.10.50

ip dhcp pool LAN
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
 lease 7
end
```

---

## Attack Configuration

### ATTACKER - Rogue DHCP Server

```
conf t
hostname ATTACKER

interface f0/0
 ip address 192.168.10.254 255.255.255.0
 no shutdown

no ip dhcp ping packets

ip dhcp pool FAKE
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.254
 dns-server 1.1.1.1
end
```

---

## Phase 1 - Normal Operation

Clients request IP addresses:

```
ip dhcp
```

Expected result:

* All clients receive IP addresses from R1
* Default gateway: 192.168.10.1

![DORA PC2 Space](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/dora%20-%20pc2%20.png)

![DORA PC2](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/dora%20-%20pc2.png)
---

## Phase 2 - Rogue DHCP Attack

After enabling the attacker, clients renew their DHCP leases:

```
ip 0.0.0.0
ip dhcp
```

![PC2 Two IPs](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/pc2%20-%20com%20os%202%20ips.png)

![Binding Attacker](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/binding%20-%20attacker.png)

Observed behavior:

* Multiple DHCP servers respond
* Some clients receive incorrect configuration
* Default gateway may become 192.168.10.254

This demonstrates the vulnerability of DHCP in an unprotected Layer 2 network.

---

## Packet Analysis (DORA Process)

Wireshark captures show the DHCP DORA process:

* Discover (broadcast from client)
![PC4 DHCP Attacker](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/pc4%20-%20dhcp%20attacker.png)

* Offer (multiple responses from R1 and attacker)
![PC4 DHCP Attacker Offer](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/pc4%20-%20dhcp%20attacker%20offer.png)

* Request (client selects one offer)
![PC4 DHCP Attacker Request](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/pc4%20-%20dhcp%20att%20request.png)

* Acknowledgment
![PC4 Attacker Ack](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/pc4%20-%20att%20ack.png)

During the attack phase, multiple DHCP Offer packets are visible, indicating competing DHCP servers.

---

## Phase 3 - DHCP Snooping Mitigation

### SW1 Configuration

```
conf t
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option

interface e0/0   ! R1
 ip dhcp snooping trust

interface e0/1   ! trunk to SW2
 ip dhcp snooping trust
```

---

### SW2 Configuration

```
conf t
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option

interface e0/0   ! uplink to SW1
 ip dhcp snooping trust
```

---

## Phase 4 - Validation

Clients renew DHCP again:

```
ip dhcp
```
![PC4 IP R1](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/pc4%20-%20ip%20r1.png)

Expected result:

* All clients receive IP from R1
* Default gateway: 192.168.10.1
* Rogue DHCP server is ignored

---

## Verification Commands

On switches:

```
show ip dhcp snooping
show ip dhcp snooping binding
```
![SW2 Snooping Binding](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/sw2%20snooping%20bind.png)

On R1 & ATTACKER:

```
show ip dhcp binding
```
![Binding R1 and Attacker](https://raw.githubusercontent.com/GiovaniSerra/Networking-labs/main/CCNA/advanced/dhcp-snooping/binding%20-%20r1%20e%20attacker.png)
---

## Important Consideration - DHCP Option 82

During implementation, DHCP failed after enabling DHCP Snooping.

This was caused by the insertion of DHCP Option 82 by the switches.

Option 82 adds relay agent information to DHCP requests. In this lab environment, the DHCP server (R1) does not properly process this information, causing it to ignore requests.

To resolve this:

```
no ip dhcp snooping information option
```

After disabling Option 82, DHCP resumed normal operation while maintaining protection against rogue servers.

---

## Key Findings

* DHCP is inherently vulnerable in Layer 2 networks
* Rogue DHCP servers can disrupt client configuration
* DHCP Snooping effectively blocks unauthorized DHCP responses
* Proper trust configuration is critical
* Option 82 may require adjustment depending on the environment

---

## Conclusion

This lab demonstrates a complete lifecycle:

* Normal DHCP operation
* Attack via rogue DHCP server
* Impact on clients
* Mitigation using DHCP Snooping

It highlights the importance of Layer 2 security mechanisms in enterprise networks.

---

## Related Labs

This lab builds upon fundamental dynamic address assignment concepts. For a detailed breakdown of the initial DORA sequence and baseline server configuration:

* Review the foundational lab: [DHCP Basic Lab](../../basic/dhcp)
