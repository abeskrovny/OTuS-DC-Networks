# Lab07: VxLAN. Аналоги VPC

## Состав работы
- [Условие задачи](#условие-задачи)

### Условие задачи
В этой самостоятельной работе мы ожидаем, что вы самостоятельно:
1. Подключите клиентов 2-я линками к различным Leaf
2. Настроите агрегированный канал со стороны клиента
3. Настроите multihoming для работы в Overlay сети. Если используете Cisco NXOS - vPC, если иной вендор - то ESI LAG (либо MC-LAG с поддержкой VXLAN)
4. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств
5. Опционально - протестировать отказоустойчивость - убедиться, что связнность не теряется при отключении одного из линков

### Описание выбранного решения
Продолжу работать со схемой, собранной в [предудущей лабораторной](https://github.com/abeskrovny/OTuS-DC-Networks/blob/main/Lab06/README.md).

Схема сети имеет следующий вид:

![Схема сети](scheme2.png)

### Подключение MLAG парой

#### Краткое описание технологии
Arista MLAG — это фирменная технология агрегации каналов, которая позволяет объединить два физических (или виртуальных vEOS) коммутатора Leaf-уровня в одну логическую сущность для подключенных к ним серверов или других коммутаторов.

Главные преимущества и архитектура MLAG в Arista EOS:
- **Active/Active топология**: В отличие от классического Spanning Tree (STP), который блокирует резервные каналы, MLAG позволяет использовать 100% полосы пропускания всех аплинков.
- **Независимый Control Plane**: В отличие от технологий стекирования (где два коммутатора превращаются в один с общим Control Plane), в паре Arista MLAG каждый коммутатор сохраняет свой независимый Control Plane. Если на одном коммутаторе упадет ОС или обновится прошивка, второй продолжит передавать трафик без штормов и простоев.
- **Стандартизация для клиентов**: Серверу не нужно знать про протоколы Arista. Он подключается к двум Leaf'ам по стандартному протоколу LACP (802.3ad).

Для сборки этой схемы в классическом варианте понадобятся три базовые вещи:
- **Peer Link** (Коммутатор–Коммутатор): Физический стык между двумя коммутаторами Leaf (обычно Port-Channel из 2-4 линков). По нему синхронизируются таблицы MAC-адресов, ARP и состояния портов.
- **Peer-VLAN и SVI**: Специальный изолированный служебный VLAN (обычно используют VLAN 4094), в котором настраиваются IP-адреса для общения Лифов друг с другом (heartbeat).
- **MLAG ID**: Идентификатор, который присваивается клиентскому Port-Channel. По этому ID оба Leaf'а понимают, что этот конкретный порт подключен к одному и тому же серверу.

Внедрение MLAG в оверлейную сеть (EVPN-VxLAN) кардинально меняет поведение фабрики и является золотым стандартом для ЦОД. В терминологии Arista и стандартов RFC такая топология называется All-Active EVPN-MLAG.

Главное изменение заключается в том, что пара Лифов начинает работать как один виртуальный VTEP, что подразумевает следующие архитектурные изменения в технологии MLAG:
- **Единый (Shared) VTEP IP (Anycast VTEP)**: В классическом решении (без многослойной структуры сети) у каждого коммутатора Leaf имеется свой уникальный VTEP IP. В схеме с MLAG на обоих коммутаторах настраивается дополнительный одинаковый Loopback-интерфейс (`Loopback1`), который используется в качестве точки терминирования VxLAN. Вся остальная фабрика (включая и коммутаторы уровня Spine) видят пару Leaf'ов как одно устройство.
- **Маршруты BGP EVPN (ESI — Ethernet Segment Identifier)**: При анонсировании MAC-адресов, прилетающих c абонентских интерфейсов через MLAG-порт, оба Leaf ставят в BGP-маршрут специальный маркер — ESI. Благодаря этому удаленные Leaf'ы знают, что хост подключен к "All-Active" группе, и могут балансировать VxLAN-трафик на оба Leaf одновременно (ECMP).
- **Локальная маршрутизация через Peer-Link**: Если пакет прилетит из VxLAN-туннеля на `swLeaf01`, а нужный линк MLAG до роутера в этот момент активен на `swLeaf02`, пакет пройдет через внутренний Peer-Link стык коммутаторов.

#### Настройка MLAG на стороне фабрики
Начнем с определения VRF:
```
vrf instance vrfASYM-IRB01
   description --- VRF: RIB for Overlay Data-Plane of Assymetric IRB
!
vrf instance vrfSYM-IRB01
   description --- VRF: RIB for Overlay Data-Plane of Symmetric IRB
```

Опишем L2: пользовательские, транзитный и Peer-Link MLAG:
```
vlan 11
   name VLAN-BASED01
!
vlan 12
   name VLAN-BASED02
!
vlan 21
   name VLAN-AWARE01
!
vlan 22
   name VLAN-AWARE02
!
vlan 4001
   name L3VNI01
!
vlan 4094
   name MLAG-PEER-CONTROL
   trunk group MLAG-Peer-Link
```

Здесь следует обратить внимание на директиву `trunk group MLAG-Peer-Link`. Она изолирует указанный VLAN (или, вообще говоря, диапазон VLAN) и запрещает его автоматическую передачу по всем остальным транк-портам коммутатора даже с выставленным `switchport trunk allowed vlan all`, разрешая его прохождение строго внутри конкретной транк-группы.

Хорошим решением является использование для Peer-Link'а агрегированного порта - это позволит масштабировать полосу пропускания для состояния отказа одной из "голов" пары. Производитель крайне не рекомендует "смешивать" данный трафик ввиду его чувствительности к случайному пользовательсному трафику, который будет восприниматься парой как мусорный. 

Зададим виртуальный MAC-адрес фабрики:
```
ip virtual-router mac-address 00:1c:73:00:00:01
```

он уникален для всех коммутаторов фабрики.

Соберем агрегат для Peer-Link'а на основе портов `Ethernet8` на обоих коммутаторах:
```
interface Ethernet8
   description --- Port-channel 4094 (LACP): Channel-group for MLAG Peer-Link
   channel-group 4094 mode active
!
interface Port-Channel4094
   description --- Trunk (VLAN001): Channel-group for MLAG Peer-Link
   switchport mode trunk
   switchport trunk group MLAG-Peer-Link     ! Ограничиваем разрешенные VLAN только Peer-Link VLAN
```

Если сейчас посмотреть состояние интерфейсов (я делаю изменения на обоих "головах одновременно"), увидим их состояние:
```
swLeaf01#sh int status
Port       Name                                                             Status       Vlan      Duplex Speed  Type            Flags Encapsulation
Et1        --- L3 p2p (no VLAN, no VRF): connection for swSpine01:Ethernet1 connected    routed    full   1G     EbraTestPhyPort
Et2        --- L3 p2p (no VLAN, no VRF): connection for swSpine02:Ethernet1 connected    routed    full   1G     EbraTestPhyPort
Et3        --- L3 p2p (no VLAN, no VRF): connection for swSpine03:Ethernet1 connected    routed    full   1G     EbraTestPhyPort
Et4                                                                         connected    1         full   1G     EbraTestPhyPort
Et5                                                                         connected    1         full   1G     EbraTestPhyPort
Et6                                                                         connected    1         full   1G     EbraTestPhyPort
Et7                                                                         connected    1         full   1G     EbraTestPhyPort
Et8        --- Port-channel 4094 (LACP): Channel-group for MLAG Peer-Link   connected    in Po4094 full   1G     EbraTestPhyPort
Ma1                                                                         disabled     routed    a-full a-1G   10/100/1000
Po4094     --- Trunk (VLAN001): Channel-group for MLAG Peer-Link            connected    trunk     full   1G     N/A
```

Если L2 "поднят" и работоспособен, можно продолжать дальше и переходить к настройкам L3.

На уровне подстилающей сети нам нужно создать дополнительный петлевой интерфейс с общим IP-адресом на паре коммутаторов уровня Leaf, составляющих MLAG-пару, и включить его в процесс маршрутизации:
```
interface Loopback1
   description --- Loopback 1 (no VRF): interface for vPC instance
   load-interval 60
   ip address 10.1.1.1/32
   isis enable Underlay
   isis passive
```

Данная конфигурация должна быть произведена на обоих коммутаторах MLAG-пары.

В результате мы получим следующую картину (со стороны коммутатора уровня Spine):
```
swSpine01#sh ip route 10.1.1.1

VRF: default
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

 I L2     10.1.1.1/32 [115/20]
           via 10.1.2.1, Ethernet1
           via 10.1.2.2, Ethernet2
```

Экземпляр iBGP процесса при соседстве со Spine'ами будут продолжать использовать петлю `loopback0` в качестве `update-source`, строя соседство для каждого Leaf'а участника MLAG раздельно.

Настроим SVI-интерфейсы:
```
interface Vlan11
   description --- Virtual (VLAN011:VLAN-BASED01, VRF:vrfASYM-IRB01): L3 termination point
   vrf vrfASYM-IRB01
   ip address 192.168.11.201/24
   ip virtual-router address 192.168.11.253
!
interface Vlan12
   description --- Virtual (VLAN012:VLAN-BASED02, VRF:vrfSYM-IRB01): L3 termination point
   vrf vrfSYM-IRB01
   ip address 192.168.12.201/24
   ip virtual-router address 192.168.12.253
!
interface Vlan21
   description --- Virtual (VLAN021:VLAN-AWARE01, VRF:vrfASYM-IRB01): L3 termination point
   vrf vrfASYM-IRB01
   ip address 192.168.21.201/24
   ip virtual-router address 192.168.21.253
!
interface Vlan22
   description --- Virtual (VLAN022:VLAN-AWARE02, VRF:vrfSYM-IRB01): L3 termination point
   vrf vrfSYM-IRB01
   ip address 192.168.22.201/24
   ip virtual-router address 192.168.22.253
!
interface Vlan4001
   description --- Virtual (VLAN:L3VNI01, VRF:vrfL3VNI01): L3 transport interface
   no autostate
   vrf vrfSYM-IRB01
```

Я разделю L3-функционал p2p связей двух MLAG-голов: выделенный канал через интерфейс `Management1` будет использоваться исключетельно для Heartbeat-фунционала. Этот подход является корректным и соответствующим рекомендациям Arista Design Guide для предотвращения сценария Split-Brain.
```
interface Vlan4094
   description --- Virtual (VLAN:MLAG-PEER-CONTROL, no VRF): L2 MLAG Peer-Link Control IP
   no autostate                     ! Оставляем интерфейс включенным постоянно
   ip address 172.16.1.0/31
```

> В терминологии и конфигурации Arista MLAG резервный канал связи для проверки доступности соседа чаще всего называют одним из трех вариантов: `mlag-peer-check` (или `mlag-heartbeat`) — это техническое название самого функционала в документации (сервис отправки Heartbeat-пакетов). Peer-Link Backup — официальный термин Arista для этой роли.

IP-адрес на Peer-Link (`interface Vlan4094`) остается обязательным, так как он обслуживает основной Control Plane MLAG и синхронизирует служебные данные через протокол *MlagApp*, в то время как `Management`-порт рассчитан исключительно на легкие UDP-пакеты проверки состояния.
```
interface Management1
   description --- L3 p2p (no VLAN, no VRF): interface for MLAG heard-beat
   ip address 172.16.2.0/31
```

> На физических интерфейсах (в том числе и `Management1`) отсутствует опция `no autostate`. Ввиду этого, не очень хорошей идеей видится коммутация его через третье активное оборудование.

Сразу проверим связанность с техническими интерфейсами:
```
swLeaf01#ping 172.16.1.1
PING 172.16.1.1 (172.16.1.1) 72(100) bytes of data.
80 bytes from 172.16.1.1: icmp_seq=1 ttl=64 time=29.3 ms
80 bytes from 172.16.1.1: icmp_seq=2 ttl=64 time=26.9 ms
80 bytes from 172.16.1.1: icmp_seq=3 ttl=64 time=18.7 ms
80 bytes from 172.16.1.1: icmp_seq=4 ttl=64 time=18.9 ms
80 bytes from 172.16.1.1: icmp_seq=5 ttl=64 time=30.2 ms

--- 172.16.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 112ms
rtt min/avg/max/mdev = 18.660/24.784/30.175/5.036 ms, pipe 4, ipg/ewma 28.090/27.057 ms

swLeaf01#ping 172.16.2.1
PING 172.16.2.1 (172.16.2.1) 72(100) bytes of data.
80 bytes from 172.16.2.1: icmp_seq=1 ttl=64 time=7.83 ms
80 bytes from 172.16.2.1: icmp_seq=2 ttl=64 time=1.46 ms
80 bytes from 172.16.2.1: icmp_seq=3 ttl=64 time=5.36 ms
80 bytes from 172.16.2.1: icmp_seq=4 ttl=64 time=2.23 ms
80 bytes from 172.16.2.1: icmp_seq=5 ttl=64 time=3.00 ms

--- 172.16.2.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 107ms
rtt min/avg/max/mdev = 1.456/3.976/7.834/2.330 ms, pipe 2, ipg/ewma 26.869/5.846 ms

swLeaf01#sh arp
Address         Age (sec)  Hardware Addr   Interface
10.1.0.1          0:00:13  5000.00d7.ee0b  Ethernet1
10.1.0.2          0:00:31  5000.00d5.5dc0  Ethernet2
10.1.0.3          0:00:42  5000.00f6.ad37  Ethernet3
172.16.1.1        0:00:56  5000.0003.3766  Vlan4094, Port-Channel4094
172.16.2.1        0:00:07  5000.0004.0000  Management1
```

Включим маршрутизацию во всех VRF:
```
ip routing vrf vrfASYM-IRB01
ip routing vrf vrfSYM-IRB01
```

Состояние таблиц RIB во всех VRF:
```
swLeaf01#sh ip route vrf all

VRF: default
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

Gateway of last resort is not set

 I L2     10.1.0.1/32
           directly connected, Ethernet1
 I L2     10.1.0.2/32
           directly connected, Ethernet2
 I L2     10.1.0.3/32
           directly connected, Ethernet3
 C        10.1.1.1/32
           directly connected, Loopback1
 C        10.1.2.1/32
           directly connected, Loopback0
 I L2     10.1.2.2/32 [115/30]
           via 10.1.0.1, Ethernet1
           via 10.1.0.2, Ethernet2
           via 10.1.0.3, Ethernet3
 I L2     10.1.2.3/32 [115/30]
           via 10.1.0.1, Ethernet1
           via 10.1.0.2, Ethernet2
           via 10.1.0.3, Ethernet3
 I L2     10.1.2.4/32 [115/30]
           via 10.1.0.1, Ethernet1
           via 10.1.0.2, Ethernet2
           via 10.1.0.3, Ethernet3
 I L2     10.1.255.1/32 [115/30]
           via 10.1.0.1, Ethernet1
           via 10.1.0.2, Ethernet2
           via 10.1.0.3, Ethernet3
 C        172.16.1.0/31
           directly connected, Vlan4094
 C        172.16.2.0/31
           directly connected, Management1


VRF: vrfASYM-IRB01
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

Gateway of last resort is not set

 C        192.168.11.0/24
           directly connected, Vlan11
 C        192.168.21.0/24
           directly connected, Vlan21


VRF: vrfSYM-IRB01
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

Gateway of last resort is not set

 C        192.168.12.0/24
           directly connected, Vlan12
 C        192.168.22.0/24
           directly connected, Vlan22
```

Все готово для активации домена MLAG:
```
mlag configuration
   domain-id mlagLeaf01
   peer-link Port-Channel4094             ! L2 интерфейс Data Plane MLAG
   local-interface Vlan4094               ! L3 интерфейс управления Control Plane MLAG 
   peer-address 172.16.1.1                ! L3 адрес соседа интерфейса управления Control Plane MLAG
   peer-address heartbeat 172.16.2.1      ! L3 адрес для сервиса Heart-beat через интерфейс management1
```

После симметричной настройки (изменятся только IP адреса `peer-address`), проверим состояние MLAG-домена:
```
swLeaf01#show mlag detail
MLAG Configuration:
domain-id                          :          mlagLeaf01
local-interface                    :            Vlan4094
peer-address                       :          172.16.1.1
peer-link                          :    Port-Channel4094
hb-peer-address                    :          172.16.2.1
peer-config                        :          consistent

MLAG Status:
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:00:00:03:37:66
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False

MLAG Ports:
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   0

MLAG Detailed Status:
State                           :              secondary
Peer State                      :                primary
State changes                   :                      2
Last state change time          :            0:02:09 ago
Hardware ready                  :                   True
Failover                        :                  False
Failover Cause(s)               :                Unknown
Last failover change time       :                  never
Secondary from failover         :                  False
Peer MAC address                :      50:00:00:03:37:66
Peer MAC routing supported      :                  False
Reload delay                    :            300 seconds
Non-MLAG reload delay           :            300 seconds
Ports errdisabled               :                  False
Lacp standby                    :                  False
Configured heartbeat interval   :                4000 ms
Effective heartbeat interval    :                4000 ms
Heartbeat timeout               :               60000 ms
Last heartbeat timeout          :                  never
Heartbeat timeouts since reboot :                      0
UDP heartbeat alive             :                   True
Heartbeats sent/received        :                 240/70
Peer monotonic clock offset     :   49235.149939 seconds
Agent should be running         :                   True
P2p mount state changes         :                      1
Fast MAC redirection enabled    :                  False
Interface activation interlock  :            unsupported
```

Как видно из вывода, коммутатор swLeaf02 выбран в роли 'primary'.

В классическом понимании технологии Arista MLAG оба коммутатора всегда работают в режиме `Active/Active` (`All-Active`) с точки зрения передачи данных (Data Plane). Технологически не подразумевается конфигурации, в которой можно программно заставить трафик идти только через один коммутатор, а второй держать в холодном резерве — это нарушило бы саму суть агрегации каналов LACP.

Однако в паре MLAG всегда выбирается один главный коммутатор для управления служебными протоколами (Control Plane COORDINATOR). Его роли распределяются как `Primary` (основной координатор) и `Secondary` (ведомый). В EOS нет команды выставления приоритета одному из коммутаторов - Arista сознательно убрала возможность вручную выставлять приоритеты «Primary/Secondary» для самого процесса MLAG. По логике производителя, оба коммутатора абсолютно равноправны (Active/Active), а выбор того, кто станет управляющим координатором, происходит автоматически по наименьшему MAC-адресу пары

Включаем туннельный интерфейс с точкой терминирования в созданном петлевом интерфейсе и маппим пользовательские VLAN:
```
interface Vxlan1
   description --- VxLAN (no VRF): interface for Overlay Control-Plane
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 11 vni 10011
   vxlan vlan 12 vni 10012
   vxlan vlan 21 vni 10021
   vxlan vlan 22 vni 10022
   vxlan vrf vrfSYM-IRB01 vni 14001
```

Состояние туннельного интерфейса:
```
swLeaf01#sh interfaces vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Description: --- VxLAN (no VRF): interface for Overlay Control-Plane
  Source interface is Loopback1 and is active with 10.1.1.1
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: CLI
  Remote MAC learning is disabled
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [11, 10011]       [12, 10012]       [21, 10021]       [22, 10022]

  Dynamic VLAN to VNI mapping for 'evpn' is
    [4097, 14001]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is
   [vrfSYM-IRB01, 14001]
  MLAG Shared Router MAC is 0000.0000.0000
```

Control Plane Overlay полностью аналогичен настройкам коммутатором `swBorderLeaf01` и `swLeaf04`:
```
router bgp 65000
   router-id 10.1.2.1
   neighbor grpSPINES peer group
   neighbor grpSPINES remote-as 65000
   neighbor grpSPINES update-source Loopback0
   neighbor grpSPINES send-community
   neighbor 10.1.0.1 peer group grpSPINES
   neighbor 10.1.0.2 peer group grpSPINES
   neighbor 10.1.0.3 peer group grpSPINES
   !
   vlan 11
      rd auto
      route-target both 65000:11
      redistribute learned
   !
   vlan 12
      rd auto
      route-target both 65000:12
      redistribute learned
   !
   vlan-aware-bundle vabBUNDLE01
      rd auto
      route-target both 65000:20
      redistribute learned
      vlan 21-22
   !
   address-family evpn
      neighbor grpSPINES activate
   !
   vrf vrfSYM-IRB01
      rd 10.1.2.1:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      redistribute connected
```

Настроим клиентское подключение. Сначала, на стороне инфраструктуры:
```
interface Ethernet4
   description --- Port-channel 4 (MLAG): connection to srvHost01:Gi1
   load-interval 60
   channel-group 4 mode active
!
interface Port-Channel4
   description --- Trunk (VLAN001): connection to srvHost01:Gi1
   load-interval 60
   switchport trunk allowed vlan 1-999
   switchport mode trunk
   mlag 4
```

Эта конфигурация эквивалентна для обоих сторон MLAG-пары.

#### Абонентское подключение и намтройка оверлэй-маршрутизации
Настроим LACP-подключение со стороны роутера-сервера srvHost01:
```
hostname srvHost01
!
ip domain name local
!
lldp run
!
vrf definition vrfVLAN-AWARE01
 !
 address-family ipv4
 exit-address-family
!
vrf definition vrfVLAN-AWARE02
 !
 address-family ipv4
 exit-address-family
!
vrf definition vrfVLAN-BUNDLE01
 !
 address-family ipv4
 exit-address-family
!
vrf definition vrfVLAN-BUNDLE02
 !
 address-family ipv4
 exit-address-family
!
interface Port-channel1
 description --- Trunk (VLAN001): connection to MLAG: swLeaf01, swLeaf02
 no ip address
 load-interval 60
 no negotiation auto
 no mop enabled
 no mop sysid
!
interface GigabitEthernet1
 description --- Port-channel 1 (LACP): connection to swLeaf01:Ethernet4
 no ip address
 negotiation auto
 no mop enabled
 no mop sysid
 channel-group 1 mode active
!
interface GigabitEthernet2
 description --- Port-channel 1 (LACP): connection to swLeaf02:Ethernet4
 no ip address
 load-interval 60
 negotiation auto
 no mop enabled
 no mop sysid
 channel-group 1 mode active
!
interface Port-channel1.11
 description --- Virtual (VLAN011): VLAN-BUNDLE01
 encapsulation dot1Q 11
 vrf forwarding vrfVLAN-BUNDLE01
 ip address 192.168.11.1 255.255.255.0
!
interface Port-channel1.12
 description --- Virtual (VLAN012): VLAN-BUNDLE02
 encapsulation dot1Q 12
 vrf forwarding vrfVLAN-BUNDLE02
 ip address 192.168.12.1 255.255.255.0
!
interface Port-channel1.21
 description --- Virtual (VLAN021): VLAN-AWARE01
 encapsulation dot1Q 21
 vrf forwarding vrfVLAN-AWARE01
 ip address 192.168.21.1 255.255.255.0
!
interface Port-channel1.22
 description --- Virtual (VLAN022): VLAN-AWARE02
 encapsulation dot1Q 22
 vrf forwarding vrfVLAN-AWARE02
 ip address 192.168.22.1 255.255.255.0
!
ip route vrf vrfVLAN-BUNDLE01 0.0.0.0 0.0.0.0 192.168.11.253
ip route vrf vrfVLAN-BUNDLE02 0.0.0.0 0.0.0.0 192.168.12.253
ip route vrf vrfVLAN-AWARE01 0.0.0.0 0.0.0.0 192.168.21.253
ip route vrf vrfVLAN-AWARE02 0.0.0.0 0.0.0.0 192.168.22.253
```

Проверим связанность внутри своих пар IRB:
```
srvHost01#ping vrf vrfVLAN-BUNDLE01 192.168.11.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.11.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 56/79/114 ms
srvHost01#ping vrf vrfVLAN-BUNDLE01 192.168.21.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.21.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 102/214/514 ms
srvHost01#ping vrf vrfVLAN-BUNDLE02 192.168.12.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.12.3, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 67/96/134 ms
srvHost01#ping vrf vrfVLAN-BUNDLE02 192.168.22.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.22.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 55/116/315 ms
```

что указывает на работоспособность всех сервисных моделей.

Осталось настроить маршрутизацию на уровне оверлейной сети: она аналогична конфигурации `swLeaf01`:
```
interface Vlan11
   ip ospf area 0.0.0.0
interface Vlan12
   ip ospf area 0.0.0.0
interface Vlan21
   ip ospf area 0.0.0.0
interface Vlan22
   ip ospf area 0.0.0.0
router ospf 10 vrf vrfASYM-IRB01
   router-id 10.1.11.1
   passive-interface default
   no passive-interface Vlan11
   no passive-interface Vlan21
   max-lsa 12000
router ospf 20 vrf vrfSYM-IRB01
   router-id 10.1.12.1
   passive-interface default
   no passive-interface Vlan12
   no passive-interface Vlan22
   max-lsa 12000
```

Аналогичная и на `swLeaf02`, но с уникальными `router-id`.

Теперь можно проверить таблицы RIB во всех VRF:
```
swLeaf02#sh ip route vrf all

VRF: default
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

Gateway of last resort is not set

 I L2     10.1.0.1/32
           directly connected, Ethernet1
 I L2     10.1.0.2/32
           directly connected, Ethernet2
 I L2     10.1.0.3/32
           directly connected, Ethernet3
 C        10.1.1.1/32
           directly connected, Loopback1
 I L2     10.1.2.1/32 [115/30]
           via 10.1.0.1, Ethernet1
           via 10.1.0.2, Ethernet2
           via 10.1.0.3, Ethernet3
 C        10.1.2.2/32
           directly connected, Loopback0
 I L2     10.1.2.3/32 [115/30]
           via 10.1.0.1, Ethernet1
           via 10.1.0.2, Ethernet2
           via 10.1.0.3, Ethernet3
 I L2     10.1.2.4/32 [115/30]
           via 10.1.0.1, Ethernet1
           via 10.1.0.2, Ethernet2
           via 10.1.0.3, Ethernet3
 I L2     10.1.255.1/32 [115/30]
           via 10.1.0.1, Ethernet1
           via 10.1.0.2, Ethernet2
           via 10.1.0.3, Ethernet3
 C        172.16.1.0/31
           directly connected, Vlan4094
 C        172.16.2.0/31
           directly connected, Management1


VRF: vrfASYM-IRB01
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

Gateway of last resort:
 O E2     0.0.0.0/0 [110/1]
           via 192.168.11.254, Vlan11

 C        192.168.11.0/24
           directly connected, Vlan11
 O        192.168.12.0/24 [110/11]
           via 192.168.11.254, Vlan11
 C        192.168.21.0/24
           directly connected, Vlan21
 O        192.168.22.0/24 [110/21]
           via 192.168.11.254, Vlan11


VRF: vrfSYM-IRB01
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

Gateway of last resort:
 O E2     0.0.0.0/0 [110/1]
           via 192.168.12.254, Vlan12

 O        192.168.11.0/24 [110/11]
           via 192.168.12.254, Vlan12
 B I      192.168.12.254/32 [200/0]
           via VTEP 10.1.255.1 VNI 14001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        192.168.12.0/24
           directly connected, Vlan12
 O        192.168.21.0/24 [110/21]
           via 192.168.12.254, Vlan12
 C        192.168.22.0/24
           directly connected, Vlan22
```

Проверим связанность со стороны роутера `rtBorder01`:
```
rtBorder01#sh ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, m - OMP
       n - NAT, Ni - NAT inside, No - NAT outside, Nd - NAT DIA
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       H - NHRP, G - NHRP registered, g - NHRP registration summary
       o - ODR, P - periodic downloaded static route, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR
       & - replicated local route overrides by connected

Gateway of last resort is 10.1.10.2 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 10.1.10.2
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.10.0/24 is directly connected, GigabitEthernet1
L        10.1.10.101/32 is directly connected, GigabitEthernet1
      192.168.11.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.11.0/24 is directly connected, GigabitEthernet1.11
L        192.168.11.254/32 is directly connected, GigabitEthernet1.11
      192.168.12.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.12.0/24 is directly connected, GigabitEthernet1.12
L        192.168.12.254/32 is directly connected, GigabitEthernet1.12
O     192.168.21.0/24
           [110/11] via 192.168.11.250, 23:31:02, GigabitEthernet1.11
           [110/11] via 192.168.11.204, 23:31:02, GigabitEthernet1.11
           [110/11] via 192.168.11.202, 00:02:03, GigabitEthernet1.11
           [110/11] via 192.168.11.201, 00:03:51, GigabitEthernet1.11
O     192.168.22.0/24
           [110/11] via 192.168.12.250, 23:56:10, GigabitEthernet1.12
           [110/11] via 192.168.12.204, 23:30:55, GigabitEthernet1.12
           [110/11] via 192.168.12.202, 00:01:45, GigabitEthernet1.12
           [110/11] via 192.168.12.201, 00:03:43, GigabitEthernet1.12

rtBorder01# ping 192.168.11.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.11.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 56/109/196 ms

rtBorder01# ping 192.168.12.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.12.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 85/143/266 ms

rtBorder01# ping 192.168.21.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.21.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 63/105/242 ms

rtBorder01# ping 192.168.22.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.22.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 127/280/481 ms
```

#### Команды дополнительной диагностики MLAG
Произведем полноценную диагностику MLAG-пары. Проверим конфигурацию:
```
swLeaf01#show mlag config-sanity
No global configuration inconsistencies found.

No per interface configuration inconsistencies found.
```

Вывод команды `show mlag config-sanity` показывает, что глобальные настройки и параметры интерфейсов на обоих коммутаторах полностью идентичны и корректны. В терминологии Arista EOS это означает, что кластер находится в состоянии полной синхронизации, а критические параметры (такие как MTU, членство VLAN в транках и настройки STP) не имеют расхождений, способных заблокировать трафик.

Просмотрим расширенную информацию об MLAG-домене, включая точное количество переданных/принятых служебных пакетов (MLAG control packets) через Peer-Link. Помогает понять, нет ли дропов на самом межкоммутаторном стыке.
```
swLeaf01#show mlag detail
MLAG Configuration:
domain-id                          :          mlagLeaf01
local-interface                    :            Vlan4094
peer-address                       :          172.16.1.1
peer-link                          :    Port-Channel4094
hb-peer-address                    :          172.16.2.1
peer-config                        :          consistent

MLAG Status:
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:00:00:03:37:66
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False

MLAG Ports:
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   0

MLAG Detailed Status:
State                           :              secondary
Peer State                      :                primary
State changes                   :                      2
Last state change time          :            4:07:02 ago
Hardware ready                  :                   True
Failover                        :                  False
Failover Cause(s)               :                Unknown
Last failover change time       :                  never
Secondary from failover         :                  False
Peer MAC address                :      50:00:00:03:37:66
Peer MAC routing supported      :                  False
Reload delay                    :            300 seconds
Non-MLAG reload delay           :            300 seconds
Ports errdisabled               :                  False
Lacp standby                    :                  False
Configured heartbeat interval   :                4000 ms
Effective heartbeat interval    :                4000 ms
Heartbeat timeout               :               60000 ms
Last heartbeat timeout          :                  never
Heartbeat timeouts since reboot :                      0
UDP heartbeat alive             :                   True
Heartbeats sent/received        :              7502/7410
Peer monotonic clock offset     :   49234.919719 seconds
Agent should be running         :                   True
P2p mount state changes         :                      1
Fast MAC redirection enabled    :                  False
Interface activation interlock  :            unsupported
```

К диагностическим командам есть смысл добавить следующий вывод:
```
swLeaf01#show interfaces status
Port       Name                                                             Status       Vlan      Duplex Speed  Type            Flags Encapsulation
Et1        --- L3 p2p (no VLAN, no VRF): connection for swSpine01:Ethernet1 connected    routed    full   1G     EbraTestPhyPort
Et2        --- L3 p2p (no VLAN, no VRF): connection for swSpine02:Ethernet1 connected    routed    full   1G     EbraTestPhyPort
Et3        --- L3 p2p (no VLAN, no VRF): connection for swSpine03:Ethernet1 connected    routed    full   1G     EbraTestPhyPort
Et4        --- Port-channel 4 (MLAG): connection to srvHost01:Gi1           connected    in Po4    full   1G     EbraTestPhyPort
Et5                                                                         connected    1         full   1G     EbraTestPhyPort
Et6                                                                         connected    1         full   1G     EbraTestPhyPort
Et7                                                                         connected    1         full   1G     EbraTestPhyPort
Et8        --- Port-channel 4094 (LACP): Channel-group for MLAG Peer-Link   connected    in Po4094 full   1G     EbraTestPhyPort
Ma1        --- L3 p2p (no VLAN, no VRF): interface for MLAG heard-beat      connected    routed    a-full a-1G   10/100/1000
Po4        --- Trunk (VLAN001): connection to srvHost01:Gi1                 connected    trunk     full   2G     N/A
Po4094     --- Trunk (VLAN001): Channel-group for MLAG Peer-Link            connected    trunk     full   1G     N/A
```

А также список интерфейсов, вовлеченных в MLAG:
```
swLeaf01#show mlag interfaces
                                                                   local/remote
mlag  desc                                  state  local   remote        status
----- ------------------------------- ------------ ------ -------- ------------
   4  --- Trunk (VLAN001): connectio  active-full    Po4      Po4         up/up
```

Есть также команда просмотра таблицы MAC-адресов, ассоциированных с MLAG:
```
swLeaf01#show mac address-table mlag
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    5000.0009.0000    DYNAMIC     Po4094     1       4:17:49 ago
  11    001c.7300.0001    STATIC      Po4094
  11    5000.0003.3766    STATIC      Po4094
  12    001c.7300.0001    STATIC      Po4094
  12    5000.0003.3766    STATIC      Po4094
  21    001c.7300.0001    STATIC      Po4094
  21    5000.0003.3766    STATIC      Po4094
  22    001c.7300.0001    STATIC      Po4094
  22    5000.0003.3766    STATIC      Po4094
4001    001c.7300.0001    STATIC      Po4094
4001    5000.0003.3766    STATIC      Po4094
4094    001c.7300.0001    STATIC      Po4094
4094    5000.0003.3766    STATIC      Po4094
Total Mac Addresses for this criterion: 13
```

#### Тестирование сбоя
Проверим работоспособность MLAG при условии нарушения следующих соединений:

![Тестирование сбоя](scheme_bad.png)

Отключим указанные на схеме линии на стороне коммутаторов уровня Leaf:
```
```