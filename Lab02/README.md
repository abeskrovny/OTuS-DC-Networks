# Lab02: Построение Underlay сети (OSPF)

## Состав работы

### Результирующая конфигурация с предыдущей стадии
В работе [Lab1](../Lab01/README.md) была построена сеть Клоша, разработан IP-план для стеков IPv4 и IPv6, а также выполнены некоторые дополнительные действия, описанные в материале.

Учитывая эквивалентность настройки устройств одного уровня, ниже будут указаны конфигурации только двух устройств разного уровня: swSpine01 и swLeaf01.

#### Стартовая конфигурация коммутуторов уровня Spine
```
! Command: show running-config
! device: swSpine01 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname swSpine01
dns domain local
!
spanning-tree mode mstp
!
vrf instance plnOOB
   description --- Plane OOB: Out-Of-Band
   rd 1:255
!
interface Ethernet1
   description --- L3 (no VRF, no VLAN): p2p connection to swLeaf01/Et1
   load-interval 60
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   ipv6 address 2001:db8:101:201:201::1/64
!
interface Ethernet2
   description --- L3 (no VRF, no VLAN) p2p connection to swLeaf02/Et1
   load-interval 60
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   ipv6 address 2001:db8:101:202:201::1/64
!
interface Ethernet3
   description --- L3 (no VRF, no VLAN) p2p connection to swLeaf03/Et1
   load-interval 60
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   ipv6 address 2001:db8:101:203:201::1/64
!
interface Ethernet4
   description --- SECURITY OFF
   shutdown
   load-interval 60
!
interface Ethernet5
   description --- SECURITY OFF
   shutdown
   load-interval 60
!
interface Ethernet6
   description --- SECURITY OFF
   shutdown
   load-interval 60
!
interface Ethernet7
   description --- SECURITY OFF
   shutdown
   load-interval 60
!
interface Ethernet8
   description --- SECURITY OFF
   shutdown
   load-interval 60
!
interface Loopback0
   description --- Virtual (no VRF, no VLAN): Underlay Control Plane
   load-interval 60
   ip address 10.1.0.1/32
   ipv6 address 2001:db8:101::1/128
!
interface Management1
   description --- L3 (vrf: plnOOB, no VLAN): Management network
   load-interval 60
   vrf plnOOB
   ip address 10.1.255.1/24
!
ip routing
no ip routing vrf plnOOB
!
ipv6 unicast-routing
!
ip route vrf plnOOB 0.0.0.0/0 10.1.255.254
!
end
```

#### Стартовая конфигурация коммутуторов уровня Leaf
```
! Command: show running-config
! device: swLeaf01 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname swLeaf01
dns domain local
!
spanning-tree mode mstp
!
vrf instance plnOOB
   description --- Plane OOB: Out-Of-Band
   rd 1:255
!
interface Ethernet1
   description --- L3 (no VRF, no VLAN): p2p connection to swSpine01/Et1
   load-interval 60
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   ipv6 address 2001:db8:101:201:201::2/64
!
interface Ethernet2
   description --- L3 (no VRF, no VLAN): p2p connection to swSpine02/Et1
   load-interval 60
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   ipv6 address 2001:db8:102:201:201::2/64
!
interface Ethernet3
   description --- SECURITY OFF
   shutdown
   load-interval 60
!
interface Ethernet4
   description --- SECURITY OFF
   shutdown
   load-interval 60
!
interface Ethernet5
   description --- SECURITY OFF
   shutdown
   load-interval 60
!
interface Ethernet6
   description --- SECURITY OFF
   shutdown
   load-interval 60
!
interface Ethernet7
   description --- SECURITY OFF
   shutdown
   load-interval 60
!
interface Ethernet8
   description --- SECURITY OFF
   shutdown
   load-interval 60
!
interface Loopback0
   description --- Virtual (no VRF, no VLAN): Underlay Control Plane
   load-interval 60
   ip address 10.1.2.1/32
   ipv6 address 2001:db8:100:201::1/128
!
interface Loopback1
   description --- Virtual (no VRF, no VLAN): Overlay VTEP Edge
   load-interval 60
   ip address 10.1.102.1/32
   ipv6 address 2001:db8:100:201:100::1/128
!
interface Management1
   description --- L3 (vrf: plnOOB, no VLAN): Management network
   load-interval 60
   vrf plnOOB
   ip address 10.1.255.101/24
!
ip routing
no ip routing vrf plnOOB
!
ipv6 unicast-routing
!
end
```

### Конфигурирование Underlay сети с использованием OSPF
Настройки OSPF версий 2 и 3 несколько отличаются друг от друга, поэтому произведем их раздельно.

В обоих случаях будем производить настройки в контексте интерфейса в глобальной таблице RIB.

#### IPv4 (OSPFv2)
Первоначально поднимем процесс на коммутаторе swSpine01. В качестве номера экземпляра будем использовать `4`, в качестве RID - IPv4 адрес интерфейса `Loopback0`:
```
router ospf 4
   router-id 10.1.0.1
```

Далее, настраиваем интерфейсы, вовлеченные в процесс:
```
interface Ethernet1 - 3
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Loopback0
   ip ospf area 0.0.0.0
```

Аналогичные действия производим на остальных коммутаторах с той лишь разницей, что на коммутаторах уровня Leaf необходимо настраивать только 2 физических интерфейса (которые смотрят в оба Spine'а).

На этом настройку можно считать завершенной.

Произведем проверку. Состояние соседства:
```
swSpine01#show ip ospf neighbor 
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.1.2.1        4        default  0   FULL                   00:00:36    10.1.2.1        Ethernet1
10.1.2.2        4        default  0   FULL                   00:00:37    10.1.2.2        Ethernet2
10.1.2.3        4        default  0   FULL                   00:00:29    10.1.2.3        Ethernet3
```

База данных LSDB:
```
swSpine01#show ip ospf database 

            OSPF Router with ID(10.1.0.1) (Instance ID 4) (VRF default)


                 Router Link States (Area 0.0.0.0)

Link ID         ADV Router      Age         Seq#         Checksum Link count
10.1.2.1        10.1.2.1        372         0x80000006   0x1598   3
10.1.0.2        10.1.0.2        114         0x80000006   0x542d   4
10.1.0.1        10.1.0.1        117         0x80000008   0x562c   4
10.1.2.2        10.1.2.2        214         0x80000004   0xfab1   3
10.1.2.3        10.1.2.3        112         0x80000004   0xd9c    3
```

Маршруты:
```
swSpine01#sh ip route ospf

VRF: default
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

 O        10.1.0.2/32 [110/30] via 10.1.2.1, Ethernet1
                               via 10.1.2.2, Ethernet2
                               via 10.1.2.3, Ethernet3
 O        10.1.2.1/32 is directly connected, Ethernet1
 O        10.1.2.2/32 is directly connected, Ethernet2
 O        10.1.2.3/32 is directly connected, Ethernet3
```

Проверим связанность с остальными коммутаторами фабрики:
```
swSpine01#ping 10.1.0.1 repeat 1
PING 10.1.0.1 (10.1.0.1) 72(100) bytes of data.
80 bytes from 10.1.0.1: icmp_seq=1 ttl=64 time=0.593 ms

--- 10.1.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.593/0.593/0.593/0.000 ms
swSpine01#ping 10.1.0.2 repeat 1
PING 10.1.0.2 (10.1.0.2) 72(100) bytes of data.
80 bytes from 10.1.0.2: icmp_seq=1 ttl=63 time=31.9 ms

--- 10.1.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 31.951/31.951/31.951/0.000 ms
swSpine01#ping 10.1.2.2 repeat 1
PING 10.1.2.2 (10.1.2.2) 72(100) bytes of data.
80 bytes from 10.1.2.2: icmp_seq=1 ttl=64 time=10.7 ms

--- 10.1.2.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 10.708/10.708/10.708/0.000 ms
swSpine01#ping 10.1.2.3 repeat 1
PING 10.1.2.3 (10.1.2.3) 72(100) bytes of data.
80 bytes from 10.1.2.3: icmp_seq=1 ttl=64 time=20.0 ms

--- 10.1.2.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 20.068/20.068/20.068/0.000 ms
```

#### IPv6 (OSPFv3)
Первоначально поднимем процесс на коммутаторе swSpine01. В качестве номера экземпляра будем использовать `6`, в качестве RID - IPv4 адрес интерфейса `Loopback0`:
```
ipv6 router ospf 6
   router-id 10.1.0.1
```

и также в контексте интерфейсов:
```
interface Ethernet1 - 3
   ipv6 ospf network point-to-point
   ipv6 ospf 6 area 0.0.0.0
!
interface Loopback0
   ipv6 ospf 6 area 0.0.0.0
!
```

Аналогичные действия производим на остальных коммутаторах с той лишь разницей, что на коммутаторах уровня Leaf необходимо настраивать только 2 физических интерфейса (которые смотрят в оба Spine'а).

На этом настройку можно считать завершенной.

Произведем проверку. Состояние соседства:
```
swSpine01#sh ipv6 ospf neighbor 
Routing Process "ospf 6":
Neighbor 10.1.2.3 VRF default priority is 0, state is Full
  In area 0.0.0.0 interface Ethernet3
  DR is None BDR is None
  Options is E R V6
  Dead timer is due in 35 seconds
  Graceful-restart-helper mode is Inactive
  Graceful-restart attempts: 0
Neighbor 10.1.2.2 VRF default priority is 0, state is Full
  In area 0.0.0.0 interface Ethernet2
  DR is None BDR is None
  Options is E R V6
  Dead timer is due in 29 seconds
  Graceful-restart-helper mode is Inactive
  Graceful-restart attempts: 0
Neighbor 10.1.2.1 VRF default priority is 0, state is Full
  In area 0.0.0.0 interface Ethernet1
  DR is None BDR is None
  Options is E R V6
  Dead timer is due in 37 seconds
  Graceful-restart-helper mode is Inactive
  Graceful-restart attempts: 0
```

База данных LSDB:
```
swSpine01#sh ipv6 ospf database 
Codes: AEX - AS External, GRC - Grace,
       IAP - Inter Area Prefix, IAR - Inter Area Router,
       LNK - Link, NAP - Intra Area Prefix,
       NSA - Not So Stubby Area, NTW - Network,
       RTR - Router
Routing Process "ospf 6", VRF default

  AS Scope LSDB

Type        Link ID     ADV Router  Age       Seq#   Checksum
 
  Area 0.0.0.0 LSDB
 
Type        Link ID     ADV Router  Age       Seq#   Checksum
 RTR        0.0.0.0       10.1.2.3  357 0x80000003   0x00eed8
 NAP        0.0.0.1       10.1.2.3  371 0x80000003   0x001abd
 RTR        0.0.0.0       10.1.0.1  362 0x80000007   0x009701
 NAP        0.0.0.2       10.1.0.1 2001 0x8000000e   0x006a2c
 RTR        0.0.0.0       10.1.0.2  357 0x80000004   0x00fd99
 NAP        0.0.0.1       10.1.0.2  692 0x80000004   0x00855f
 RTR        0.0.0.0       10.1.2.1  679 0x80000006   0x008c3e
 NAP        0.0.0.1       10.1.2.1  104 0x8000000a   0x0017c3
 RTR        0.0.0.0       10.1.2.2  530 0x80000003   0x00c00a
 NAP        0.0.0.1       10.1.2.2  541 0x80000003   0x009f3d
 
  Interface Lo0 LSDB
 
Type        Link ID     ADV Router  Age       Seq#   Checksum
 
  Interface Et3 LSDB
 
Type        Link ID     ADV Router  Age       Seq#   Checksum
 LNK        0.0.0.1       10.1.2.3  368 0x80000001   0x00ff27
 LNK        0.0.0.3       10.1.0.1  367 0x80000001   0x00ad9c

  Interface Et2 LSDB
 
Type        Link ID     ADV Router  Age       Seq#   Checksum
 LNK        0.0.0.2       10.1.0.1  534 0x80000001   0x0091ba
 LNK        0.0.0.1       10.1.2.2  535 0x80000001   0x00aecc
 
  Interface Et1 LSDB
 
Type        Link ID     ADV Router  Age       Seq#   Checksum
 LNK        0.0.0.1       10.1.0.1 1421 0x80000004   0x006fdb
 LNK        0.0.0.1       10.1.2.1 1654 0x80000004   0x00e541
```

Маршруты:
```
swSpine01#sh ipv6 route ospf 

VRF: default
Displaying 7 of 18 IPv6 routing table entries
Codes: C - connected, S - static, K - kernel, O3 - OSPFv3,
       B - Other BGP Routes, A B - BGP Aggregate, R - RIP,
       I L1 - IS-IS level 1, I L2 - IS-IS level 2, DH - DHCP,
       NG - Nexthop Group Static Route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       RC - Route Cache Route

 O3       2001:db8:100:201::1/128 [110/20]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 O3       2001:db8:100:202::1/128 [110/20]
           via fe80::5200:ff:fe03:3766, Ethernet2
 O3       2001:db8:100:203::1/128 [110/20]
           via fe80::5200:ff:fe15:f4e8, Ethernet3
 O3       2001:db8:102::/64 [110/30]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
           via fe80::5200:ff:fe03:3766, Ethernet2
           via fe80::5200:ff:fe15:f4e8, Ethernet3
 O3       2001:db8:102:201::/64 [110/20]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 O3       2001:db8:102:202::/64 [110/20]
           via fe80::5200:ff:fe03:3766, Ethernet2
 O3       2001:db8:102:203::/64 [110/20]
           via fe80::5200:ff:fe15:f4e8, Ethernet3
```

В протоколе OSPFv3 для IPv6 отображение маршрутов через Link-Local адреса (fe80::...) — это стандартное поведение, жестко зашитое в спецификацию RFC 5340.

В отличие от IPv4, где для следующего шага (Next-Hop) используется глобальный IP-адрес соседа, в IPv6 вся маршрутизация внутри подсети строится исключительно на Link-Local адресах. Даже если вы настроите глобальные IPv6-адреса на стыках, OSPFv3 всё равно будет использовать fe80:: в качестве Next-Hop.

Проверим связанность с остальными коммутаторами фабрики:
```
swSpine01#ping 2001:db8:101::1 repeat 1
PING 2001:db8:101::1(2001:db8:101::1) 52 data bytes
60 bytes from 2001:db8:101::1: icmp_seq=1 ttl=64 time=1.04 ms

--- 2001:db8:101::1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.045/1.045/1.045/0.000 ms
swSpine01#ping 2001:db8:102::1 repeat 1
PING 2001:db8:102::1(2001:db8:102::1) 52 data bytes
60 bytes from 2001:db8:102::1: icmp_seq=1 ttl=63 time=23.5 ms

--- 2001:db8:102::1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 23.547/23.547/23.547/0.000 ms
swSpine01#ping 2001:db8:100:201::1 repeat 1
PING 2001:db8:100:201::1(2001:db8:100:201::1) 52 data bytes
60 bytes from 2001:db8:100:201::1: icmp_seq=1 ttl=64 time=10.6 ms

--- 2001:db8:100:201::1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 10.631/10.631/10.631/0.000 ms
swSpine01#ping 2001:db8:100:202::1 repeat 1
PING 2001:db8:100:202::1(2001:db8:100:202::1) 52 data bytes
60 bytes from 2001:db8:100:202::1: icmp_seq=1 ttl=64 time=10.9 ms

--- 2001:db8:100:202::1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 10.969/10.969/10.969/0.000 ms
swSpine01#ping 2001:db8:100:203::1 repeat 1
PING 2001:db8:100:203::1(2001:db8:100:203::1) 52 data bytes
60 bytes from 2001:db8:100:203::1: icmp_seq=1 ttl=64 time=9.84 ms

--- 2001:db8:100:203::1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 9.841/9.841/9.841/0.000 ms
```