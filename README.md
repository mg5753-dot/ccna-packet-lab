# Small Enterprise Branch Network Lab

**Tools:** Cisco Packet Tracer | **Cert Level:** CCNA  
**Topics:** OSPF · VLANs · EtherChannel · DHCP/DNS · NAT · ACLs · Port Security · AAA/SSH

---

## Scenario

A small company has two sites — a headquarters and a branch office. As the network engineer, I was tasked with designing and implementing a fully functional network from scratch. The branch office needed department segmentation, secure management access, internal services, and NAT to reach the internet. Both sites needed to exchange routes dynamically.

This lab simulates that environment end-to-end in Cisco Packet Tracer.

---

## Topology

![Network Topology](screenshots/topology.png)

| Site | Devices |
|------|---------|
| HQ | 2911 Router, 2960 Switch, 3x PCs |
| Branch | 2911 Router, 3560 MLS (Core), 2x 2960 Switches, 2x Servers, 4x PCs |
| Simulated Internet | ISP Router with loopback (203.0.113.100) |

---

## IP Addressing

### WAN Links

| Link | Device | Interface | IP Address |
|------|--------|-----------|------------|
| ISP ↔ HQ | ISP-RTR | G0/0 | 203.0.113.1/30 |
| | HQ-RTR | G0/0 | 203.0.113.2/30 |
| ISP ↔ Branch | ISP-RTR | G0/1 | 203.0.113.5/30 |
| | BR-RTR | G0/0 | 203.0.113.6/30 |
| HQ ↔ Branch (OSPF) | HQ-RTR | G0/1 | 10.0.0.1/30 |
| | BR-RTR | G0/1 | 10.0.0.2/30 |

### Branch VLANs

| VLAN | Name | Subnet | Gateway | DHCP Range |
|------|------|--------|---------|------------|
| 10 | Management | 192.168.10.0/24 | 192.168.10.1 | .10 – .50 |
| 20 | Sales | 192.168.20.0/24 | 192.168.20.1 | .10 – .50 |
| 30 | IT | 192.168.30.0/24 | 192.168.30.1 | .10 – .50 |

### Servers

| Device | IP | Services |
|--------|----|----------|
| SRV-1 | 192.168.10.2 | DHCP, DNS |
| SRV-2 | 192.168.10.3 | AAA (TACACS+), Syslog |

---

## Technologies Implemented

### Switching
- VLANs 10, 20, 30 with 802.1Q trunking between all switches
- EtherChannel (LACP) — Port-Channel 1 and 2 between Core and Access switches
- STP configured with CORE-SW as root bridge for all VLANs (`spanning-tree vlan 10,20,30 priority 4096`)

### Routing
- Inter-VLAN routing via Layer 3 SVIs on the 3560 Core switch
- OSPF Area 0 between HQ-RTR and BR-RTR
- Static default routes on both routers pointing to ISP

### Network Services
- DHCP pools on SRV-1 scoped per VLAN
- DNS resolving internal hostnames (e.g. `srv1.branch.local`)
- PAT/NAT overload on BR-RTR — inside hosts reach simulated internet via `203.0.113.6`

### Security
- Named ACL blocking Sales (VLAN 20) from accessing IT (VLAN 30); IT has unrestricted access
- Port security on all access ports — max 2 MAC addresses, shutdown on violation
- SSH v2 enabled on both routers; Telnet disabled on VTY lines
- Local AAA authentication for management access

---

## Key Configuration Snippets

### OSPF (BR-RTR)
```
router ospf 1
 router-id 2.2.2.2
 network 10.0.0.0 0.0.0.3 area 0
 network 192.168.0.0 0.0.255.255 area 0
 default-information originate
```

### NAT/PAT (BR-RTR)
```
ip nat inside source list NAT_ACL interface g0/0 overload

ip access-list standard NAT_ACL
 permit 192.168.0.0 0.0.255.255

interface g0/0
 ip nat outside
interface g0/1
 ip nat inside
```

### Inter-VLAN SVIs (CORE-SW)
```
interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown

interface vlan 30
 ip address 192.168.30.1 255.255.255.0
 no shutdown
```

### ACL — Block Sales from IT
```
ip access-list extended SALES_TO_IT
 deny ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255
 permit ip any any

interface vlan 20
 ip access-group SALES_TO_IT in
```

### SSH + AAA
```
username admin privilege 15 secret Cisco123!
ip domain-name branch.local
crypto key generate rsa modulus 2048
ip ssh version 2

line vty 0 4
 login local
 transport input ssh
```

---

## Verification

| Test | Command | Expected Result |
|------|---------|-----------------|
| VLANs active | `show vlan brief` | VLANs 10/20/30 active on correct ports |
| EtherChannel up | `show etherchannel summary` | Po1, Po2 in P state |
| STP root | `show spanning-tree` | CORE-SW is root for all VLANs |
| OSPF neighbors | `show ip ospf neighbor` | Full adjacency between routers |
| NAT working | `show ip nat translations` | Active entries after pinging ISP |
| DHCP leases | `show ip dhcp binding` | Leases assigned per VLAN |
| ACL — Sales blocked | Ping 192.168.30.x from Sales PC | Request timed out |
| ACL — IT allowed | Ping 192.168.20.x from IT PC | Success |
| Port security | `show port-security interface fa0/1` | Max 2, violation shutdown |
| SSH works | SSH to router from IT PC | Login prompt appears |
| Telnet blocked | Telnet to router from any PC | Connection refused |

---

## Design Decisions

**Why OSPF over EIGRP?**  
OSPF is vendor-neutral and the industry standard for enterprise environments. Since this lab simulates a real-world setup (not a Cisco-only environment), OSPF was the appropriate choice.

**Why Layer 3 SVIs on the Core switch instead of router-on-a-stick?**  
SVIs on a multilayer switch keep inter-VLAN traffic off the router entirely, reducing latency and freeing up router interfaces. Router-on-a-stick is simpler but creates a single bottleneck link.

**Why /30 subnets on WAN links?**  
Point-to-point links only need two usable host addresses. Using /30 conserves address space and is standard practice.

---

## Files

| File | Description |
|------|-------------|
| `enterprise-branch.pkt` | Packet Tracer file |
| `configs/` | Individual device configs exported as .txt |
| `screenshots/` | Verification outputs |

---

*Lab completed as part of post-CCNA practice. Built entirely in Cisco Packet Tracer 8.x.*
