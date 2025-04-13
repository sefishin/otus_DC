### VxLAN. L2 VNI


### Цель

- Настроить Overlay на основе VxLAN EVPN для L2 связанности между клиентами.

## Задачи

- Настроить BGP peering между Leaf и Spine в AF l2vpn evpn
- Настроить связанность между клиентами в первой зоне и убедиться в её наличии
- Зафиксировать в документации план работы, адресное пространство, схему сети, конфигурацию устройств


### 1. Предварительные условия

Схема стенда из ДЗ №1:

![img_2](Scheme_Eve_ip.png)

На каждом устройстве произведена настройка адресации согласно плана из ДЗ №1: 

|Device|Interface|IP Address|Description|
|---|---|---|---|
Spine1|Lo1|10.0.1.0/32|
-|Lo2|10.1.1.0/32|
-|Eth1|10.2.1.0/31|Link to Leaf1|
-|Eth2|10.2.1.2/31|Link to Leaf2|
-|Eth3|10.2.1.4/31|Link to Leaf3|
Spine2|Lo1|10.0.2.0/32|
-|Lo2|10.1.2.0/32|
-|Eth1|10.2.2.0/31|Link to Leaf1|
-|Eth2|10.2.2.2/31|Link to Leaf2|
-|Eth3|10.2.2.4/31|Link to Leaf3|
Leaf1|Lo1|10.0.0.1/32|
-|Lo2|10.1.0.1/32|
-|Eth1|10.2.1.1/31|Link to Spine1|
-|Eth2|10.2.2.1/31|Link to Spine2|
-|Eth3|10.4.0.1/16|Service1|
Leaf2|Lo1|10.0.0.2/32|
-|Lo2|10.1.0.2/32|
-|Eth1|10.2.1.3/31|Link to Spine1|
-|Eth2|10.2.2.3/31|Link to Spine2|
-|Eth3|10.5.0.2/16|Service2|
Leaf3|Lo1|10.0.0.3/32|
-|Lo2|10.1.0.3/32|
-|Eth1|10.2.1.5/31|Link to Spine1|
-|Eth2|10.2.2.5/31|Link to Spine2|
-|Eth3|10.6.0.3/16|Service3|
-|Eth4|10.7.0.3/16|Service4|

iBGP наcтроен на всех устройствах.

* На спайнах настроен Route-Reflector и фильтрация prefix-list
* Везде настроен multipath для ECMP
* Везде настроен BFD
* На лифах настроена редистрибуция подключенных сетей
* Настроены таймеры Keepalive = 3 sec , Hold Timer = 9 sec 


### 2.2 Настройка VxLAN EVPN

Spine1

```
S! device: Spine-1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname Spine-1
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.2.1.0/31
   bfd static neighbor 10.2.1.1
!
interface Ethernet2
   no switchport
   ip address 10.2.1.2/31
!
interface Ethernet3
   no switchport
   ip address 10.2.1.4/31
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
!
interface Loopback1
   ip address 10.0.1.0/32
!
interface Loopback2
   ip address 10.1.1.0/32
!
interface Management1
!
ip routing
!
ip prefix-list DENY seq 10 deny 10.2.2.0/29 le 32
ip prefix-list DENY seq 20 deny 10.2.1.0/29 le 32
ip prefix-list DENY seq 40 permit 0.0.0.0/0 le 32
!
router bgp 65100
   router-id 10.0.1.0
   maximum-paths 10
   neighbor PEERS peer group
   neighbor PEERS remote-as 65100
   neighbor PEERS next-hop-self
   neighbor PEERS bfd
   neighbor PEERS timers 3 9
   neighbor PEERS route-reflector-client
   neighbor 10.2.1.1 peer group PEERS
   neighbor 10.2.1.3 peer group PEERS
   neighbor 10.2.1.5 peer group PEERS
   !
   address-family ipv4
      neighbor PEERS prefix-list DENY out
!
end


```
Spine2

```
!! device: Spine2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname Spine2
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.2.2.0/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.2/31
!
interface Ethernet3
   no switchport
   ip address 10.2.2.4/31
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   ip address 10.0.2.0/32
!
interface Loopback2
   ip address 10.1.2.0/32
!
interface Management1
!
ip routing
!
ip prefix-list DENY seq 10 deny 10.2.2.0/29 le 32
ip prefix-list DENY seq 20 deny 10.2.1.0/29 le 32
ip prefix-list DENY seq 30 permit 0.0.0.0/0 le 32
!
router bgp 65100
   router-id 10.0.1.0
   maximum-paths 10
   neighbor PEERS peer group
   neighbor PEERS remote-as 65100
   neighbor PEERS next-hop-self
   neighbor PEERS timers 3 9
   neighbor PEERS route-reflector-client
   neighbor 10.2.2.1 peer group PEERS
   neighbor 10.2.2.3 peer group PEERS
   neighbor 10.2.2.5 peer group PEERS
   !
   address-family ipv4
      neighbor PEERS prefix-list DENY out
!
end

```
Leaf1

```
! Command: show running-config
! device: Leaf-1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
switchport default mode routed
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
!
!
match-list input string ztpFilter
   10 match regex ETH-4
!
hostname Leaf-1
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.2.1.1/31
   bfd static neighbor 10.2.1.0
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet2
   speed 100g-2
   no switchport
   ip address 10.2.2.1/31
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet3
   speed 100g-2
   no switchport
   ip address 10.4.0.1/16
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet4
   speed 100g-2
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet5
   speed 100g-2
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet6
   speed 100g-2
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet7
   speed 100g-2
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet8
   speed 100g-2
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Loopback1
   ip address 10.0.0.1/32
!
interface Loopback2
   ip address 10.1.0.1/32
!
interface Management1
   speed 10full
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
ip routing
!
system control-plane
   no service-policy input copp-system-policy
!
router bgp 65100
   router-id 10.0.0.1
   maximum-paths 10
   neighbor PEERS peer group
   neighbor PEERS remote-as 65100
   neighbor PEERS bfd
   neighbor PEERS timers 3 9
   neighbor 10.2.1.0 peer group PEERS
   neighbor 10.2.2.0 peer group PEERS
   redistribute connected
!
end


```

Leaf2

```
! Command: show running-config
! device: Leaf-2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
switchport default mode routed
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
!
!
match-list input string ztpFilter
   10 match regex ETH-4
!
hostname Leaf-2
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.2.1.3/31
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet2
   no switchport
   ip address 10.2.2.3/31
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet3
   speed 200g-4
   no switchport
   ip address 10.5.0.2/16
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet4
   speed 200g-4
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet5
   speed 200g-4
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet6
   speed 200g-4
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet7
   speed 200g-4
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet8
   speed 200g-4
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Loopback1
   ip address 10.0.0.2/32
!
interface Loopback2
   ip address 10.1.0.2/32
!
interface Management1
   speed 100full
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
ip routing
!
system control-plane
   no service-policy input copp-system-policy
!
router bgp 65100
   router-id 10.0.0.2
   maximum-paths 10
   neighbor PEERS peer group
   neighbor PEERS remote-as 65100
   neighbor PEERS bfd
   neighbor PEERS timers 3 9
   neighbor 10.2.1.2 peer group PEERS
   neighbor 10.2.2.2 peer group PEERS
   redistribute connected
!
end

```

Leaf3

```
! Command: show running-config
! device: Leaf-3 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
switchport default mode routed
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
!
!
match-list input string ztpFilter
   10 match regex ETH-4
!
hostname Leaf-3
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.2.1.5/31
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet2
   no switchport
   ip address 10.2.2.5/31
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet3
   speed 100g-2
   no switchport
   ip address 10.6.0.3/16
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet4
   speed 100g-2
   no switchport
   ip address 10.7.0.3/16
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
   isis enable DC1
!
interface Ethernet5
   speed 100g-2
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet6
   speed 100g-2
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet7
   speed 100g-2
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Ethernet8
   speed 100g-2
   no switchport
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
interface Loopback1
   ip address 10.0.0.3/32
!
interface Loopback2
   ip address 10.1.0.3/32
!
interface Management1
   speed 10full
   ipv6 enable
   ipv6 address auto-config
   ipv6 nd ra rx accept default-route
!
ip routing
!
system control-plane
   no service-policy input copp-system-policy
!
router bgp 65100
   router-id 10.0.0.3
   maximum-paths 10
   neighbor PEERS peer group
   neighbor PEERS remote-as 65100
   neighbor PEERS bfd
   neighbor PEERS timers 3 9
   neighbor 10.2.1.4 peer group PEERS
   neighbor 10.2.2.4 peer group PEERS
   redistribute connected
!
end


```

### 3. Проверка связности

Маршруты на Leaf1

```
Gateway of last resort is not set

 C        10.0.0.1/32 is directly connected, Loopback1
 B I      10.0.0.2/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 B I      10.0.0.3/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 C        10.1.0.1/32 is directly connected, Loopback2
 B I      10.1.0.2/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 B I      10.1.0.3/32 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 C        10.2.1.0/31 is directly connected, Ethernet1
 C        10.2.2.0/31 is directly connected, Ethernet2
 C        10.4.0.0/16 is directly connected, Ethernet3
 B I      10.5.0.0/16 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 B I      10.6.0.0/16 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2
 B I      10.7.0.0/16 [200/0] via 10.2.1.0, Ethernet1
                              via 10.2.2.0, Ethernet2

```

Маршруты на Leaf2

```

Gateway of last resort is not set

 B I      10.0.0.1/32 [200/0] via 10.2.1.2, Ethernet1
                              via 10.2.2.2, Ethernet2
 C        10.0.0.2/32 is directly connected, Loopback1
 B I      10.0.0.3/32 [200/0] via 10.2.1.2, Ethernet1
                              via 10.2.2.2, Ethernet2
 B I      10.1.0.1/32 [200/0] via 10.2.1.2, Ethernet1
                              via 10.2.2.2, Ethernet2
 C        10.1.0.2/32 is directly connected, Loopback2
 B I      10.1.0.3/32 [200/0] via 10.2.1.2, Ethernet1
                              via 10.2.2.2, Ethernet2
 C        10.2.1.2/31 is directly connected, Ethernet1
 C        10.2.2.2/31 is directly connected, Ethernet2
 B I      10.4.0.0/16 [200/0] via 10.2.1.2, Ethernet1
                              via 10.2.2.2, Ethernet2
 C        10.5.0.0/16 is directly connected, Ethernet3
 B I      10.6.0.0/16 [200/0] via 10.2.1.2, Ethernet1
                              via 10.2.2.2, Ethernet2
 B I      10.7.0.0/16 [200/0] via 10.2.1.2, Ethernet1
                              via 10.2.2.2, Ethernet2

```

Маршруты на Leaf3

```
Gateway of last resort is not set

 B I      10.0.0.1/32 [200/0] via 10.2.1.4, Ethernet1
                              via 10.2.2.4, Ethernet2
 B I      10.0.0.2/32 [200/0] via 10.2.1.4, Ethernet1
                              via 10.2.2.4, Ethernet2
 C        10.0.0.3/32 is directly connected, Loopback1
 B I      10.1.0.1/32 [200/0] via 10.2.1.4, Ethernet1
                              via 10.2.2.4, Ethernet2
 B I      10.1.0.2/32 [200/0] via 10.2.1.4, Ethernet1
                              via 10.2.2.4, Ethernet2
 C        10.1.0.3/32 is directly connected, Loopback2
 C        10.2.1.4/31 is directly connected, Ethernet1
 C        10.2.2.4/31 is directly connected, Ethernet2
 B I      10.4.0.0/16 [200/0] via 10.2.1.4, Ethernet1
                              via 10.2.2.4, Ethernet2
 B I      10.5.0.0/16 [200/0] via 10.2.1.4, Ethernet1
                              via 10.2.2.4, Ethernet2
 C        10.6.0.0/16 is directly connected, Ethernet3
 C        10.7.0.0/16 is directly connected, Ethernet4

```
Ping VPC2, VPC3, VPC4 с машины VPC1:

```
VPCS> ping 10.5.0.100

84 bytes from 10.5.0.100 icmp_seq=1 ttl=61 time=106.447 ms
84 bytes from 10.5.0.100 icmp_seq=2 ttl=61 time=31.555 ms
84 bytes from 10.5.0.100 icmp_seq=3 ttl=61 time=35.474 ms
84 bytes from 10.5.0.100 icmp_seq=4 ttl=61 time=31.781 ms
84 bytes from 10.5.0.100 icmp_seq=5 ttl=61 time=96.806 ms

VPCS> ping 10.6.0.100

84 bytes from 10.6.0.100 icmp_seq=1 ttl=61 time=82.501 ms
84 bytes from 10.6.0.100 icmp_seq=2 ttl=61 time=44.211 ms
84 bytes from 10.6.0.100 icmp_seq=3 ttl=61 time=50.197 ms
84 bytes from 10.6.0.100 icmp_seq=4 ttl=61 time=27.734 ms
84 bytes from 10.6.0.100 icmp_seq=5 ttl=61 time=31.171 ms

VPCS> ping 10.7.0.100

84 bytes from 10.7.0.100 icmp_seq=1 ttl=61 time=48.251 ms
84 bytes from 10.7.0.100 icmp_seq=2 ttl=61 time=36.471 ms
84 bytes from 10.7.0.100 icmp_seq=3 ttl=61 time=44.042 ms
84 bytes from 10.7.0.100 icmp_seq=4 ttl=61 time=75.808 ms
84 bytes from 10.7.0.100 icmp_seq=5 ttl=61 time=100.756 ms

```

Проверка ECMP:
Leaf-3#sh ip bgp
```
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65100
Route status codes: * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

         Network                Next Hop            Metric  LocPref Weight  Path
 * >Ec   10.0.0.1/32            10.2.2.4              0       100     0       i Or-ID: 10.0.0.1 C-LST: 10.0.2.0
 *  ec   10.0.0.1/32            10.2.1.4              0       100     0       i Or-ID: 10.0.0.1 C-LST: 10.0.1.0
 * >Ec   10.0.0.2/32            10.2.2.4              0       100     0       i Or-ID: 10.0.0.2 C-LST: 10.0.2.0
 *  ec   10.0.0.2/32            10.2.1.4              0       100     0       i Or-ID: 10.0.0.2 C-LST: 10.0.1.0
 * >     10.0.0.3/32            -                     0       0       -       i
 * >Ec   10.1.0.1/32            10.2.2.4              0       100     0       i Or-ID: 10.0.0.1 C-LST: 10.0.2.0
 *  ec   10.1.0.1/32            10.2.1.4              0       100     0       i Or-ID: 10.0.0.1 C-LST: 10.0.1.0
 * >Ec   10.1.0.2/32            10.2.2.4              0       100     0       i Or-ID: 10.0.0.2 C-LST: 10.0.2.0
 *  ec   10.1.0.2/32            10.2.1.4              0       100     0       i Or-ID: 10.0.0.2 C-LST: 10.0.1.0
 * >     10.1.0.3/32            -                     0       0       -       i
 * >     10.2.1.4/31            -                     1       0       -       i
 * >     10.2.2.4/31            -                     1       0       -       i
 * >Ec   10.4.0.0/16            10.2.2.4              0       100     0       i Or-ID: 10.0.0.1 C-LST: 10.0.2.0
 *  ec   10.4.0.0/16            10.2.1.4              0       100     0       i Or-ID: 10.0.0.1 C-LST: 10.0.1.0
 * >Ec   10.5.0.0/16            10.2.2.4              0       100     0       i Or-ID: 10.0.0.2 C-LST: 10.0.2.0
 *  ec   10.5.0.0/16            10.2.1.4              0       100     0       i Or-ID: 10.0.0.2 C-LST: 10.0.1.0
 * >     10.6.0.0/16            -                     1       0       -       i
 * >     10.7.0.0/16            -                     1       0       -       i
```

Проверка BFD:
```
Leaf-1#sh bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp
--------- ----------- ----------- -------------------- ------- ----------------
10.2.1.0  4095099468   504676901        Ethernet1(13)  normal   03/23/25 16:58
10.2.2.0  2133840112  4225652655        Ethernet2(14)  normal   03/30/25 18:07

         LastDown            LastDiag    State
-------------------- ------------------- -----
   03/23/25 16:50       No Diagnostic       Up
               NA       No Diagnostic       Up
```