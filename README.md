# Small Office Network — VLAN Segmentation & Secure Management

I built this lab in Cisco Packet Tracer to practice setting up a small office network, from VLANs and routing to access restrictions and remote management.

The network has one router, two Layer 2 switches, four PCs, and a server running DNS and HTTP. Each switch has an employee PC and a guest PC connected to it. This let me test VLAN connectivity across both switches, rather than keeping each VLAN on a single switch.

## Topology

R1 connects to SW1 through a single trunk. SW2 connects to SW1, and the server is connected to SW1 on an access port.

| Connection | Port on first device | Port on second device |
|---|---|---|
| R1 → SW1 | Gi0/0 | Gi0/1 |
| SW1 → SW2 | Gi0/2 | Gi0/1 |
| SW1 → Employee PC (PC1) | Fa0/1 | FastEthernet0 |
| SW1 → Guest PC | Fa0/2 | FastEthernet0 |
| SW1 → SRV1 | Fa0/3 | FastEthernet0 |
| SW2 → Employee PC | Fa0/1 | FastEthernet0 |
| SW2 → Guest PC | Fa0/2 | FastEthernet0 |

## Addressing

| VLAN | Name | Subnet | Gateway |
|---|---|---|---|
| 10 | EMPLOYEES | 192.168.10.0/24 | 192.168.10.1 |
| 20 | GUEST | 192.168.20.0/24 | 192.168.20.1 |
| 30 | SERVERS | 192.168.30.0/24 | 192.168.30.1 |
| 99 | MANAGEMENT | 192.168.99.0/24 | 192.168.99.1 |

The server uses `192.168.30.10`. SW1 and SW2 use `192.168.99.2` and `192.168.99.3` for management.

## Switching and routing

I created all four VLANs on both switches and assigned the endpoint ports as access ports. The trunk links carry VLANs 10, 20, 30, and 99. I configured them manually and disabled DTP negotiation with `switchport nonegotiate`.

For inter-VLAN routing, I used router-on-a-stick. R1 has four subinterfaces on Gi0/0, each with an 802.1Q VLAN assignment and its own gateway address. The physical interface has no IP address.

For example, the employee gateway is configured as:

```text
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
```

## DHCP, DNS, and HTTP

R1 provides DHCP for the employee and guest networks. I excluded addresses `.1` through `.20` in each subnet to leave room for static assignments. Both pools provide the appropriate gateway and `192.168.30.10` as the DNS server.

During testing, the PCs received these leases:

| Network | Assigned addresses |
|---|---|
| Employees | 192.168.10.21, 192.168.10.22 |
| Guests | 192.168.20.21, 192.168.20.22 |

SRV1 has a static address and runs both DNS and HTTP. I added an A record pointing `portal.firma.test` to `192.168.30.10`, then tested the default webpage from a client browser.

## Restricting guest access

Once basic connectivity worked, I applied an extended ACL inbound on R1's VLAN 20 subinterface:

```text
ip access-list extended GUEST_IN
 permit udp any eq 68 any eq 67
 permit udp 192.168.20.0 0.0.0.255 host 192.168.30.10 eq 53
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.30.10 eq 53
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.30.10 eq 80
 deny ip any any
!
interface GigabitEthernet0/0.20
 ip access-group GUEST_IN in
```

This allows guests to obtain an address through DHCP, query the internal DNS server, and open its HTTP webpage. Other IP traffic entering the router from VLAN 20 is denied.

I tested this from a guest PC: the webpage loaded, but pings to the server and PC1 failed. It was a useful example of why a failed ping doesn't necessarily mean a service is unavailable.

This ACL doesn't isolate guests from each other. Traffic between devices in VLAN 20 stays on the switches and doesn't pass through the router.

## SSH management

I configured management SVIs in VLAN 99 on both switches, with `192.168.99.1` as their default gateway. These addresses are for managing the switches; inter-VLAN routing still happens on R1.

SSH version 2 is enabled on R1, SW1, and SW2, using a local account and RSA keys. The VTY lines accept SSH only. A standard ACL applied with `access-class 10 in` restricts remote access to the employee subnet:

```text
access-list 10 permit 192.168.10.0 0.0.0.255
```

From PC1, I successfully connected to both switches and R1 using their VLAN 99 addresses.

## An issue during setup

PC1 initially couldn't ping the server. I tried the same ping from R1, which also failed. I then checked SW1: Fa0/3 was connected and assigned to VLAN 30, so the server port looked correct.

The missing piece was on SRV1 itself: I hadn't assigned its IP address. After setting `192.168.30.10/24` and the gateway `192.168.30.1`, the ping worked. DNS and web access worked after the server services were configured.

## Test results

| Test | Result |
|---|---|
| All four PCs receive DHCP leases | Passed |
| PC1 pings SRV1 across VLANs | Passed |
| PC1 opens `http://portal.firma.test` | Passed |
| Guest PC opens the same webpage after applying the ACL | Passed |
| Guest PC pings SRV1 after applying the ACL | Blocked as intended |
| Guest PC pings PC1 after applying the ACL | Blocked as intended |
| PC1 pings both switch management addresses | Passed |
| PC1 connects through SSH to R1, SW1, and SW2 | Passed |

The lab gave me practice checking each part of the network as I added it: switching first, then routing and services, followed by access restrictions. A next step would be to introduce a wrong VLAN assignment or remove a VLAN from a trunk and troubleshoot the resulting connectivity problem.
