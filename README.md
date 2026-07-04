# Enterprise Network Design

This document is the full configuration reference for a 3-site network built in Cisco Packet Tracer:
 
- **ISP-Router** — core/transit router connecting all three sites
- **Office-Router** — Office LAN (VLAN 10 + VLAN 20)
- **ABC-Router** — ABC Company LAN (VLAN 30 + VLAN 40)
- **Home Wireless Router (PT-AC)** — wireless branch with NAT

Routing: **OSPF (single area 0)**.  
Addressing: **VLSM**.  
Switching: **802.1Q trunking + Router-on-a-Stick**.  
Services: **DHCP, DNS, HTTP (Web Server)**. NAT/PAT is handled only by the Home Wireless Router (built-in on the PT-AC device).
 
---
 
## 1. Topology Overview
 
```
                                   ISP-Router
                         Gi0/0     Gi0/1      Gi0/2
                          |          |           |
                    10.0.0.0/30  20.0.0.0/30  30.0.0.0/30
                          |          |           |
                    Office-Router ABC-Router  HomeRouter-PT-AC (NAT)
                       Gi0/1        Gi0/1          |  \
                    (trunk)      (trunk)     Smartphone Tablet Laptop
                        |            |           (192.168.10.0/24, DHCP+NAT)
                    Switch0       Switch1
                  (Fa0/1 trunk) (Fa0/1 trunk)
                   /   |   \       /   |   \
              VLAN10 uplink DNS  VLAN30 uplink WebServer
              PC0,1,2  |   Server PC3,PC4 |
                       |                  |
                   Switch3            Switch2
                 (VLAN 20)          (VLAN 40)
                PC6, PC7, PC8    PC9, PC10, PC11
```
 
---
 
## 2. IP Addressing Plan (VLSM)
 
### 2.1 WAN Links (point-to-point /30)
 
| Link | Network | Mask | Device — Interface | IP Address |
|---|---|---|---|---|
| ISP ↔ Office | 10.0.0.0/30 | 255.255.255.252 | ISP-Router Gi0/0 | 10.0.0.1 |
| | | | Office-Router Gi0/0 | 10.0.0.2 |
| ISP ↔ ABC | 20.0.0.0/30 | 255.255.255.252 | ISP-Router Gi0/1 | 20.0.0.1 |
| | | | ABC-Router Gi0/0 | 20.0.0.2 |
| ISP ↔ Wireless | 30.0.0.0/30 | 255.255.255.252 | ISP-Router Gi0/2 | 30.0.0.1 |
| | | | HomeRouter-PT-AC (Internet port) | 30.0.0.2 |
 
### 2.2 LAN Subnets (VLANs)
 
| VLAN | Site | Switch | Network | Mask | Usable Range | Gateway | Static Hosts |
|---|---|---|---|---|---|---|---|
| 10 | Office | Switch0 | 158.0.10.0/29 | 255.255.255.248 | .1 – .6 | 158.0.10.1 | DNS Server = 158.0.10.5 |
| 20 | Office | Switch3 | 158.0.20.0/28 | 255.255.255.240 | .1 – .14 | 158.0.20.1 | — |
| 30 | ABC | Switch1 | 170.0.10.0/29 | 255.255.255.248 | .1 – .6 | 170.0.10.1 | Web Server = 170.0.10.5 |
| 40 | ABC | Switch2 | 170.0.20.0/29 | 255.255.255.248 | .1 – .6 | 170.0.20.1 | — |
| — | Wireless | (n/a) | 192.168.10.0/24 | 255.255.255.0 | .2 – .254 | 192.168.10.1 | — |
 
PCs (PC0–PC2, PC6–PC8, PC3–PC4, PC9–PC11) and wireless clients (Smartphone0, Tablet PC0, Laptop0) all receive addresses via **DHCP**.
 
---
 
## 3. VLAN Plan
 
| VLAN ID | Name | Switch | Subnet |
|---|---|---|---|
| 10 | OFFICE_DATA | Switch0 | 158.0.10.0/29 |
| 20 | OFFICE_REMOTE | Switch3 | 158.0.20.0/28 |
| 30 | ABC_DATA | Switch1 | 170.0.10.0/29 |
| 40 | ABC_REMOTE | Switch2 | 170.0.20.0/29 |
 
Trunk links (802.1Q):
- Switch0 Fa0/1 ↔ Office-Router Gi0/1 (VLANs 10, 20)
- Switch0 Fa0/6 ↔ Switch3 Fa0/1 (VLAN 20)
- Switch1 Fa0/1 ↔ ABC-Router Gi0/1 (VLANs 30, 40)
- Switch1 Fa0/3 ↔ Switch2 Fa0/1 (VLAN 40)
---
 
## 4. Router Configurations
 
### 4.1 ISP-Router
 
```
enable
configure terminal
interface GigabitEthernet0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 20.0.0.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/2
 ip address 30.0.0.1 255.255.255.252
 no shutdown
router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 20.0.0.0 0.0.0.3 area 0
 network 30.0.0.0 0.0.0.3 area 0
end
write memory
```
 
### 4.2 Office-Router (Router-on-a-Stick + DHCP)
 
```
enable
configure terminal
interface GigabitEthernet0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 no ip address
 no shutdown
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 158.0.10.1 255.255.255.248
interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 158.0.20.1 255.255.255.240
router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 158.0.10.0 0.0.0.7 area 0
 network 158.0.20.0 0.0.0.15 area 0
ip dhcp excluded-address 158.0.10.1 158.0.10.5
ip dhcp excluded-address 158.0.20.1
ip dhcp pool VLAN10
 network 158.0.10.0 255.255.255.248
 default-router 158.0.10.1
 dns-server 158.0.10.5
ip dhcp pool VLAN20
 network 158.0.20.0 255.255.255.240
 default-router 158.0.20.1
 dns-server 158.0.10.5
end
write memory
```
 
### 4.3 ABC-Router (Router-on-a-Stick + DHCP)
 
```
enable
configure terminal
interface GigabitEthernet0/0
 ip address 20.0.0.2 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 no ip address
 no shutdown
interface GigabitEthernet0/1.30
 encapsulation dot1Q 30
 ip address 170.0.10.1 255.255.255.248
interface GigabitEthernet0/1.40
 encapsulation dot1Q 40
 ip address 170.0.20.1 255.255.255.248
router ospf 1
 network 20.0.0.0 0.0.0.3 area 0
 network 170.0.10.0 0.0.0.7 area 0
 network 170.0.20.0 0.0.0.7 area 0
ip dhcp excluded-address 170.0.10.1 170.0.10.5
ip dhcp excluded-address 170.0.20.1
ip dhcp pool VLAN30
 network 170.0.10.0 255.255.255.248
 default-router 170.0.10.1
 dns-server 158.0.10.5
ip dhcp pool VLAN40
 network 170.0.20.0 255.255.255.248
 default-router 170.0.20.1
 dns-server 158.0.10.5
end
write memory
```
 
### 4.4 Home Wireless Router (PT-AC) — GUI Configuration
 
This device is configured through its **GUI (Config / GUI tab)**, not IOS CLI:
 
#### Internet (WAN) setup
- Connection type: Static IP
- IP Address: `30.0.0.2`
- Subnet Mask: `255.255.255.252`
- Default Gateway: `30.0.0.1`
- DNS Server: `158.0.10.5`
#### LAN setup
- Router IP: `192.168.10.1`
- Subnet Mask: `255.255.255.0`
#### DHCP Server
- Enabled
- Start IP: `192.168.10.100`
- Max Users: e.g. `50` (covers Smartphone0, Tablet PC0, Laptop0)
#### NAT
- Built-in on this device (Packet Tracer Home Wireless Router always PATs/overloads LAN traffic to the WAN IP `30.0.0.2`).
---
 
## 5. Switch Configurations
 
### 5.1 Switch0 (Office switch)
 
```
enable
configure terminal
hostname Switch0
vlan 10
 name VLAN10
vlan 40
 name VLAN20
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
interface range FastEthernet0/2-5
 switchport mode access
 switchport access vlan 10
interface FastEthernet0/6
 switchport mode trunk
 switchport trunk allowed vlan 10,20
end
write memory
```
 
### 5.2 Switch3 (Office switch)
 
```
enable
configure terminal
hostname Switch3
vlan 20
 name VLAN20
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 20
interface range FastEthernet0/2-4
 switchport mode access
 switchport access vlan 20
end
write memory
```
 
### 5.3 Switch1 (ABC switch)
 
```
enable
configure terminal
hostname Switch1
vlan 30
 name VLAN30
vlan 40
 name VLAN40
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 30,40
interface range FastEthernet0/2-4
 switchport mode access
 switchport access vlan 30
interface FastEthernet0/5
 switchport mode trunk
 switchport trunk allowed vlan 30,40
end
write memory
```
 
### 5.4 Switch2 (ABC remote switch)
 
```
enable
configure terminal
hostname Switch2
vlan 40
 name VLAN40
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 40
interface range FastEthernet0/2-4
 switchport mode access
 switchport access vlan 40
end
write memory
```
 
---
 
## 6. Server Configurations
 
### 6.1 DNS Server (158.0.10.5, VLAN 10)
 
#### IP Config:
- Static — IP `158.0.10.5`,
- Mask `255.255.255.248`
- Gateway `158.0.10.1`
#### Services
  - DNS = ON
  - A Record: `www.abccompany.com` → `170.0.10.5`

### 6.2 Web Server (170.0.10.5, VLAN 30)
 
#### IP Config:
- Static — IP `170.0.10.5`
- Mask `255.255.255.248`
- Gateway `170.0.10.1`
- DNS Server `158.0.10.5`
#### Services
- HTTP = ON 
---
 
## 7. End Devices 
- **IP Config:** DHCP
  
| Devices | VLAN / Network | Gateway |  
|---|---|---|  
| PC0, PC1, PC2 | VLAN 10 (158.0.10.0/29) | 158.0.10.1 |  
| PC6, PC7, PC8 | VLAN 20 (158.0.20.0/28) | 158.0.20.1 |  
| PC3, PC4 | VLAN 30 (170.0.10.0/29) | 170.0.10.1 |  
| PC9, PC10, PC11 | VLAN 40 (170.0.20.0/29) | 170.0.20.1 |  
| Smartphone0, Tablet PC0, Laptop0 | Wireless (192.168.10.0/24) | 192.168.10.1 |
 
---
 
## 8. Verification Commands
 
Run on routers/switches to validate the build:
 
```
show ip interface brief
show vlan brief
show interfaces trunk
show ip route
show ip ospf neighbor
show ip ospf interface brief
show ip dhcp binding
show ip dhcp pool
 
