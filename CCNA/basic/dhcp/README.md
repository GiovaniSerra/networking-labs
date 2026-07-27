# DHCP Basic Lab
## Overview

This lab configures a DHCP server and validates dynamic IP assignment within a single broadcast domain without relay agents.

The goal is to validate DHCP functionality and understand the dynamic IP assignment process.

## Topology

Single broadcast domain environment.

- Router (R1) acting as DHCP Server
- Layer 2 Switch (SW1)
- Client (PC1)

![Topology](https://github.com/GiovaniSerra/Networking-labs/blob/main/CCNA/Basic/DHCP/topology.png)

## Lab Environment
- Router: 7206VXR (Dynamips)
- Switch: IOL L2 l2-adipservicesk9-m-15.2-20170202.bin
- PC: IOL L2 (configured as DHCP client via switchport)

## IP Addressing
- Network: 192.168.1.0/24
- Default Gateway: 192.168.1.254
- DHCP Range: 192.168.1.101 – 192.168.1.253
- DNS Server: 8.8.8.8

## Configuration

### R1 (DHCP Server)

enable

configure terminal

hostname R1

interface fa0/0

 ip address 192.168.1.254 255.255.255.0
 
 no shutdown

ip dhcp excluded-address 192.168.1.1 192.168.1.100

ip dhcp pool DHCPLAB

 network 192.168.1.0 255.255.255.0
 
 default-router 192.168.1.254
 
 dns-server 8.8.8.8
 
 lease 7
 
end

### PC1 (DHCP Client)

enable

configure terminal

hostname PC1

interface e0/1

 no switchport
 
 ip address dhcp
 
end

## Verification

### DHCP Binding Table (R1)

show ip dhcp binding

### Expected result:

- IP address assigned to the client
- MAC address associated with the lease

![IP BINDING](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/dhcp/ip%20dhcp%20binding.png)

### Client IP Address (PC1)

Verify that the client received a valid IP address from the 192.168.1.0/24 network.

![Show IP Int Brief](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/dhcp/dhcp.png)

The assigned IP address falls within the configured DHCP pool range (192.168.1.101–192.168.1.253), confirming that the excluded-address configuration is working as expected

## Connectivity Test

ping 192.168.1.254

## Expected result:

Successful replies

![Successful Ping](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/dhcp/ping.png)

## DHCP Process (Packet Capture)

The DHCP process follows the DORA sequence:

Discover
- The client broadcasts a request looking for a DHCP server

Offer
- The server responds with an available IP address

Request
- The client requests the offered IP

Acknowledge
- The server confirms the assignment

Packets 154–159 show the full DORA process using Transaction ID 0x16a8.

![DORA Wireshark](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/dhcp/dhcp-dora-wireshark.png)

All packets share the same Transaction ID, confirming they belong to the same DHCP exchange.

The DHCP Discover is sent from 0.0.0.0 to 255.255.255.255 using UDP port 68 → 67, indicating the client does not yet have an IP address.

## DHCP DORA Sequence

The capture shows the complete DHCP exchange using a single Transaction ID, confirming all packets belong to the same process.

### Discover
The client sends a broadcast from 0.0.0.0 to 255.255.255.255 using UDP port 68 → 67, since it does not yet have an IP address
### Offer
R1 responds with an available IP address from the DHCP pool, including network parameters such as gateway and DNS
### Request
The client broadcasts a request to accept the offered IP address, informing all DHCP servers of its selection
### Acknowledge (ACK)
The DHCP server confirms the lease and finalizes the IP assignment

All packets share the same Transaction ID, ensuring consistency in the DHCP exchange. The process uses UDP ports 67 (server) and 68 (client), as defined for DHCP communication.

![DORA Sequence](https://github.com/GiovaniSerra/networking-labs/blob/main/CCNA/basic/dhcp/dora-sequence.png)

## Troubleshooting

- Ensure interfaces are up (no shutdown)
- Verify DHCP pool network matches the interface subnet
- Check if the client is configured for DHCP
- Confirm excluded-address range does not block the pool
