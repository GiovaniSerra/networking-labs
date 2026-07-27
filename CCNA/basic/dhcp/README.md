# DHCP Basic Lab

## Overview

This lab configures a DHCP server and validates dynamic IP assignment within a single broadcast domain without relay agents.

The goal is to validate DHCP functionality, verify configuration parameters, and analyze the dynamic IP assignment process (DORA).

---

## Topology

Single broadcast domain environment consisting of:
* **R1**: Cisco Router acting as DHCP Server
* **SW1**: Layer 2 Switch
* **PC1**: Client host

![Topology](./topology.png)

---

## Lab Environment

| Component | Description / Model |
|----------|-------------|
| **Platform** | EVE-NG / PNETLab |
| **Router (R1)** | Cisco 7206VXR (Dynamips) |
| **Switch (SW1)** | Cisco IOL L2 (`l2-adipservicesk9-m-15.2-20170202.bin`) |
| **Client (PC1)** | Cisco IOL L2 (configured as host via switchport) |
| **Packet Analysis** | Wireshark Capture |

---

## IP Addressing Plan

| Parameter | Value |
|-----------|-------|
| **Network Subnet** | 192.168.1.0/24 |
| **Default Gateway** | 192.168.1.254 |
| **Excluded IP Range** | 192.168.1.1 – 192.168.1.100 |
| **DHCP Pool Range** | 192.168.1.101 – 192.168.1.253 |
| **DNS Server** | 8.8.8.8 |
| **Lease Time** | 7 days |

---

## Configuration

### R1 (DHCP Server)

```cisco
enable
configure terminal
hostname R1

interface FastEthernet0/0
 ip address 192.168.1.254 255.255.255.0
 no shutdown
exit

ip dhcp excluded-address 192.168.1.1 192.168.1.100

ip dhcp pool DHCPLAB
 network 192.168.1.0 255.255.255.0
 default-router 192.168.1.254
 dns-server 8.8.8.8
 lease 7
end
```

### PC1 (DHCP Client)

```cisco
enable
configure terminal
hostname PC1

interface Ethernet0/1
 no switchport
 ip address dhcp
 no shutdown
end
```

---


## Verification

### 1. DHCP Binding Table (R1)

Verify active IP assignments on the DHCP server:

```cisco
show ip dhcp binding
```

### Expected Result:

IP address assigned to the client (192.168.1.101).
Hardware address (MAC) associated with the dynamic lease.

![IP BINDING](./ip%20dhcp%20binding.png)

### Client IP Address Verification (PC1)
Verify that the client received a valid IP address from the server:

```cisco
show ip interface brief
```

![Show IP Int Brief](./dhcp.png)

The assigned IP address (192.168.1.101) falls strictly within the configured DHCP pool range (192.168.1.101–192.168.1.253), confirming that the ip dhcp excluded-address range (192.168.1.1–192.168.1.100) is working as expected.


## Connectivity Test
Validate connectivity from the client to the default gateway:

```cisco
ping 192.168.1.254
```

Expected Result: Successful ICMP replies (100% success rate).

![Successful Ping](./ping.png)

## DHCP Process & Packet Analysis (DORA)

### DORA Sequence Summary

1. **Discover**: The client broadcasts a request looking for an available DHCP server (`0.0.0.0:68` → `255.255.255.255:67`).
2. **Offer**: The router (R1) responds with an available IP address from the pool, including network parameters such as gateway and DNS.
3. **Request**: The client broadcasts a request accepting the offered IP address, informing all DHCP servers of its selection.
4. **Acknowledge (ACK)**: The DHCP server confirms the lease and finalizes the IP assignment.

### Wireshark Traffic Capture

Packets 154–159 show the complete DORA process using a single Transaction ID (`0x16a8`):

![Wireshark DORA Capture](./dhcp-dora-wireshark.png)

![DORA Sequence Detail](./dora-sequence.png)

* **Transaction ID**: All packets share the same Transaction ID (`0x16a8`), confirming they belong to the same exchange sequence.
* **Ports**: The exchange utilizes standard UDP ports 67 (Server) and 68 (Client).
* **Initial State**: The Discover packet uses `0.0.0.0` as the source IP, as the host does not yet hold a valid network configuration.

---

## Troubleshooting Checklist

* Ensure interfaces are administratively up (`no shutdown`).
* Verify that the DHCP pool network subnet matches the router interface IP subnet.
* Confirm that the client interface is explicitly set to acquire an address dynamically (`ip address dhcp`).
* Check if the `ip dhcp excluded-address` range is not accidentally blocking the entire active pool scope.

---

## Next Steps & Related Labs

After mastering basic DHCP operation and analyzing the DORA process, the next step is securing dynamic address allocation against attacks like DHCP Spoofing and Rogue DHCP Servers.

* Proceed to the advanced security lab: [DHCP Snooping Lab](../../advanced/dhcp-snooping)
