### VxLAN. Routing.

### Цель

- Реализовать передачу суммарных префиксов через EVPN route-type 5

## Задачи

- Разместить двух "клиентов" в разных VRF в рамках одной фабрики
- Настроить маршрутизацию между клиентами через внешнее устройство
- Задокументировать работы


### 1. Предварительные условия

Схема стенда:

![img_2](Scheme_Eve_ip.jpg)

В качестве коммутаторов - Nexus 9000v. В качестве роутера - CSR-1000v. На устройствах произведена настройка адресации согласно плана: 

|Device|Interface|IP Address|Description|
|---|---|---|---|
Spine1|Lo0|11.11.11.11/32|
-|Eth1|10.2.1.0/31|Link to Leaf1|
-|Eth2|10.2.1.2/31|Link to Leaf2|
Leaf1|Lo0|1.1.1.1/32|
-|Eth1/1|10.2.1.1/31|Link to Spine1|
-|Eth1/2|VLAN 10|Link to VPC1.1|
-|Eth1/3|VLAN 30|Link to VPC1.2|
Leaf2|Lo0|2.2.2.2/32|
-|Eth1/1|10.2.1.3/31|Link to Spine1|
-|Eth1/2|VLAN 10|Link to VPC2.1|
-|Eth1/3|VLAN 30|Link to VPC2.2|
VPC1.1|Eth0|192.168.10.10/24|TENANT-1|
VPC1.2|Eth0|192.168.30.10/24|TENANT-2|
VPC2.1|Eth0|192.168.10.20/24|TENANT-1|
VPC2.2|Eth0|192.168.20.20/24|TENANT-1|

### 2.1 Настройка L3 VNI

В добавление к лабораторной №5 настраивается дополнительный vlan и vni:

```

Leaf-1(config)#vlan 20
Leaf-1(config)#vn-segment 10020

Leaf-1(config)# int vlan 20
Leaf-1(config-if)# no sh
Leaf-1(config-if)# vrf member TEST
Leaf-1(config-if)# ip address 192.168.20.1/24
Leaf-1(config-if)# fabric forwarding mode anycast-gateway


Leaf-1(config-if)# int nve 1
Leaf-1(config-if-nve)# member vni 10020
Leaf-1(config-if-nve-vni)# ingress-replication protocol bgp
Leaf-1(config-if-nve-vni)# exit

```


### 2.2 Полные конфигурации устройств

Spine1

```
version 9.3(1) Bios:version
switchname Spine-1
vdc Spine-1 id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 248 maximum 248
  limit-resource u6route-mem minimum 96 maximum 96
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8

no feature ssh
nv overlay evpn
feature ospf
feature bgp
feature nv overlay

no password strength-check
username admin password 5 $5$W0y86Dzq$VYqcG2rmHXwwsRgeGsDANo0IkJiGypiqbg.szV0Mt9
9  role network-admin
ip domain-lookup
no system default switchport
snmp-server user admin network-admin auth md5 0x72a08c95012f25aad4ac2eff6e974ef1
 priv 0x72a08c95012f25aad4ac2eff6e974ef1 localizedkey
rmon event 1 description FATAL(1) owner PMON@FATAL
rmon event 2 description CRITICAL(2) owner PMON@CRITICAL
rmon event 3 description ERROR(3) owner PMON@ERROR
rmon event 4 description WARNING(4) owner PMON@WARNING
rmon event 5 description INFORMATION(5) owner PMON@INFO

vlan 1

vrf context management

interface Ethernet1/1
  ip address 10.2.1.0/31
  ip ospf network point-to-point
  ip router ospf 1 area 0.0.0.0
  no shutdown

interface Ethernet1/2
  ip address 10.2.1.2/31
  ip ospf network point-to-point
  ip router ospf 1 area 0.0.0.0
  no shutdown

interface Ethernet1/3

...

interface Ethernet1/128

interface mgmt0
  vrf member management

interface loopback0
  ip address 11.11.11.11/32
  ip router ospf 1 area 0.0.0.0
line console
line vty
router ospf 1
  router-id 11.11.11.11
router bgp 65000
  address-family l2vpn evpn
  neighbor 1.1.1.1
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 2.2.2.2
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client



```

Leaf1

```
version 9.3(1) Bios:version
switchname Leaf-1
vdc Leaf-1 id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 248 maximum 248
  limit-resource u6route-mem minimum 96 maximum 96
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8

no feature ssh
nv overlay evpn
feature ospf
feature bgp
feature fabric forwarding
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay

no password strength-check
username admin password 5 $5$kIg7CrSN$gIjQQRaKfj3wihCghlc69z8zji6.RBlWIqat.xX2at
C  role network-admin
ip domain-lookup
no system default switchport
copp profile strict
snmp-server user admin auth md5 0x899fa8779e43ff9f657e345fe48e5e72 priv 0x899fa8
779e43ff9f657e345fe48e5e72 localizedkey engineID 128:0:0:9:3:80:48:52:2:204:0
rmon event 1 description FATAL(1) owner PMON@FATAL
rmon event 2 description CRITICAL(2) owner PMON@CRITICAL
rmon event 3 description ERROR(3) owner PMON@ERROR
rmon event 4 description WARNING(4) owner PMON@WARNING
rmon event 5 description INFORMATION(5) owner PMON@INFO

fabric forwarding anycast-gateway-mac 0000.0000.0001
vlan 1,10,20,100
vlan 10
  vn-segment 10010
vlan 20
  vn-segment 10020
vlan 100
  vn-segment 100

vrf context TEST
  vni 100
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn
vrf context management

interface Vlan1

interface Vlan10
  no shutdown
  vrf member TEST
  ip address 192.168.10.1/24
  fabric forwarding mode anycast-gateway

interface Vlan20
  no shutdown
  vrf member TEST
  ip address 192.168.20.1/24
  fabric forwarding mode anycast-gateway

interface Vlan100
  no shutdown
  vrf member TEST
  ip forward

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 100 associate-vrf
  member vni 10010
    ingress-replication protocol bgp
  member vni 10020
    ingress-replication protocol bgp

interface Ethernet1/1
  ip address 10.2.1.1/31
  ip ospf network point-to-point
  ip router ospf 1 area 0.0.0.0
  no shutdown

interface Ethernet1/2
  switchport
  switchport access vlan 10

interface Ethernet1/3
  switchport
  switchport access vlan 20


...

interface Ethernet1/128

interface mgmt0
  vrf member management

interface loopback0
  ip address 1.1.1.1/32
  ip router ospf 1 area 0.0.0.0
line console
line vty
router ospf 1
  router-id 1.1.1.1
router bgp 65000
  address-family l2vpn evpn
  neighbor 11.11.11.11
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended


```

Leaf2

```
version 9.3(1) Bios:version
switchname Leaf-2
vdc Leaf-2 id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 248 maximum 248
  limit-resource u6route-mem minimum 96 maximum 96
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8

no feature ssh
nv overlay evpn
feature ospf
feature bgp
feature fabric forwarding
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay

no password strength-check
username admin password 5 $5$xo.rexaD$ITvOJ.j6./AQdpu3w90daeHaW1fYL46KOlgVYmCn/P
.  role network-admin
ip domain-lookup
no system default switchport
snmp-server user admin network-admin auth md5 0x038347264d1f462ed75c0f2ce97b8564
 priv 0x038347264d1f462ed75c0f2ce97b8564 localizedkey
rmon event 1 description FATAL(1) owner PMON@FATAL
rmon event 2 description CRITICAL(2) owner PMON@CRITICAL
rmon event 3 description ERROR(3) owner PMON@ERROR
rmon event 4 description WARNING(4) owner PMON@WARNING
rmon event 5 description INFORMATION(5) owner PMON@INFO

fabric forwarding anycast-gateway-mac 0000.0000.0001
vlan 1,10,20,100
vlan 10
  vn-segment 10010
vlan 20
  vn-segment 10020
vlan 100
  vn-segment 100

vrf context TEST
  vni 100
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn
vrf context management

interface Vlan1

interface Vlan10
  no shutdown
  vrf member TEST
  ip address 192.168.10.1/24
  fabric forwarding mode anycast-gateway

interface Vlan20
  no shutdown
  vrf member TEST
  ip address 192.168.20.1/24
  fabric forwarding mode anycast-gateway

interface Vlan100
  no shutdown
  vrf member TEST
  ip forward

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 100 associate-vrf
  member vni 10010
    ingress-replication protocol bgp
  member vni 10020
    ingress-replication protocol bgp

interface Ethernet1/1
  ip address 10.2.1.3/31
  ip ospf network point-to-point
  ip router ospf 1 area 0.0.0.0
  no shutdown

interface Ethernet1/2
  switchport
  switchport access vlan 10

interface Ethernet1/3
  switchport
  switchport access vlan 20


...

interface Ethernet1/128

interface mgmt0
  vrf member management

interface loopback0
  ip address 2.2.2.2/32
  ip router ospf 1 area 0.0.0.0
line console
line vty
router ospf 1
  router-id 2.2.2.2
router bgp 65000
  address-family l2vpn evpn
  neighbor 11.11.11.11
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended

```

### 3. Проверка L3-связности

Пинг между VPC в разных VLAN, подключенными к разным VTEP
```
VPCS>  ip 192.168.10.10/24 192.168.10.1
Checking for duplicate address...
PC1 : 192.168.10.10 255.255.255.0 gateway 192.168.10.1

VPCS> ping 192.168.10.20

84 bytes from 192.168.10.20 icmp_seq=1 ttl=64 time=20.346 ms
84 bytes from 192.168.10.20 icmp_seq=2 ttl=64 time=23.255 ms
^C
VPCS> ping 192.168.20.10

84 bytes from 192.168.20.10 icmp_seq=1 ttl=63 time=15.235 ms
84 bytes from 192.168.20.10 icmp_seq=2 ttl=63 time=14.456 ms
^C
VPCS> ping 192.168.20.20

84 bytes from 192.168.20.20 icmp_seq=1 ttl=62 time=31.800 ms
84 bytes from 192.168.20.20 icmp_seq=2 ttl=62 time=22.940 ms
```

Таблица маршрутизации на Leaf1

```
Leaf-1# sh ip route vrf TEST
IP Route Table for VRF "TEST"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

192.168.10.0/24, ubest/mbest: 1/0, attached
    *via 192.168.10.1, Vlan10, [0/0], 05:46:50, direct
192.168.10.1/32, ubest/mbest: 1/0, attached
    *via 192.168.10.1, Vlan10, [0/0], 05:46:50, local
192.168.10.10/32, ubest/mbest: 1/0, attached
    *via 192.168.10.10, Vlan10, [190/0], 00:38:04, hmm
192.168.10.20/32, ubest/mbest: 1/0
    *via 2.2.2.2%default, [200/0], 05:16:20, bgp-65000, internal, tag 65000 (evp
n) segid: 100 tunnelid: 0x2020202 encap: VXLAN

192.168.20.0/24, ubest/mbest: 1/0, attached
    *via 192.168.20.1, Vlan20, [0/0], 00:40:16, direct
192.168.20.1/32, ubest/mbest: 1/0, attached
    *via 192.168.20.1, Vlan20, [0/0], 00:40:16, local
192.168.20.10/32, ubest/mbest: 1/0, attached
    *via 192.168.20.10, Vlan20, [190/0], 00:19:07, hmm
192.168.20.20/32, ubest/mbest: 1/0
    *via 2.2.2.2%default, [200/0], 00:15:55, bgp-65000, internal, tag 65000 (evp
n) segid: 100 tunnelid: 0x2020202 encap: VXLAN

```


Таблица Host Mobility Manager
```
Leaf-1# sh fabric forwarding ip local-host-db vrf TEST

HMM host IPv4 routing table information for VRF TEST
Status: *-valid, x-deleted, D-Duplicate, DF-Duplicate and frozen,
        c-cleaned in 00:07:00

    Host                 MAC Address        SVI        Flags      Physical Inter
face
*   192.168.10.10/32     0050.7966.68e5     Vlan10     0x420201   Ethernet1/2
*   192.168.20.10/32     0050.7966.68eb     Vlan20     0x420201   Ethernet1/3
```

Таблица L2RIB 
```
Leaf-1# sh l2route evpn mac-ip all
Flags -(Rmac):Router MAC (Stt):Static (L):Local (R):Remote (V):vPC link
(Dup):Duplicate (Spl):Split (Rcv):Recv(D):Del Pending (S):Stale (C):Clear
(Ps):Peer Sync (Ro):Re-Originated (Orp):Orphan
Topology    Mac Address    Host IP                                 Prod   Flags         Seq No     Next-Hops
----------- -------------- --------------------------------------- ------ ---------- ---------- ---------------------------------------
10          0050.7966.68e5 192.168.10.10                           HMM    L,            0         Local
10          0050.7966.68e6 192.168.10.20                           BGP    --            0         2.2.2.2
20          0050.7966.68eb 192.168.20.10                           HMM    L,            0         Local
20          0050.7966.68ea 192.168.20.20                           BGP    --            0         2.2.2.2

```



