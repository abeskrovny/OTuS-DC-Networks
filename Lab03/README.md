# Lab03: Построение Underlay сети (ISIS)

## Состав работы
- [Условие задачи](#условие-задачи)
- []

### Условие задачи
В этой самостоятельной работе мы ожидаем, что вы самостоятельно:
1. Настроите ISIS в Underlay сети, для IP связанности между всеми сетевыми устройствами.
2. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств
3. Убедитесь в наличии IP связанности между устройствами в ISIS домене

### Описание решения
Протокол ISIS, с точки зрения построения слоя Underlay, по моему мнению, является наиболее технологичным, гибким, масштабируемым и отказоустойчивым вариантом.

На Underlay-уровне (базовой физической сети) лучше использовать протоколы Link-State (OSPF, IS-IS), а не дистанционно-векторные (RIP, EIGRP), потому что они обеспечивают минимальное время сходимости, отсутствие топологических петель и точную информацию о карте сети, требуемую для overlay-технологий.

Кроме того, дистанционно-векторные протоколы передают только готовую таблицу маршрутизации («вектор» и «расстояние»). Если на каком-то участке сети происходит сбой, маршрутизаторы тратят время на пересчет и обмен слухами. Link-State протоколы передают состояние линков — каждый роутер сам строит точную математическую карту сети и моментально находит лучший путь.

***Можно выделить следующие преимущества ISIS***:
1. **Архитектурная независимость, масштабируемость и живучесть**:
   - Масштабируемость и обратная совместимость: Благодаря использованию структуры TLV (Type-Length-Value), протокол IS-IS обладает исключительной масштабируемостью и обратной совместимостью. При внедрении новых сетевых протоколов или стандартов адресации архитектура IS-IS не требует фундаментальной переработки программного кода (в отличие от OSPF, где для поддержки IPv6 потребовалось создание отдельной версии OSPFv3). Разработчикам сетевого оборудования достаточно специфицировать и добавить новые типы TLV-полей, что существенно упрощает интеграцию инноваций на протяжении всего жизненного цикла технологии.
   - Полная изоляция от IP-стека (Чистый L2-транспорт): IS-IS работает напрямую на канальном уровне (Layer 2), используя кадры CLNP. Если на коммутаторе из-за ошибки в конфигурации, сбоя скрипта автоматизации или аварии полностью «отвалится» IPv4/IPv6-стек, сам IS-IS продолжит жить и видеть соседей.
   - Диагностика и управление через CLNP (Connect over CLNS): Благодаря независимости от IP, сохраняется уникальную возможность подключиться к текстовой консоли удаленного коммутатора через L2-кадры IS-IS (`connect clns <HOSTNAME>`). Однако, не все производители интегрируют полный стек CLNP в свои продукты, что создает некоторые ограничения его использования.
   - Экономия IP-адресов: На физических p2p стыках между уровнями Spine и Leaf L3-адреса носят чисто технический характер — достаточно навесить L3-адрес любым доступным способом через `ipv6 enable` (Link-local) или `ip unnumbered Loopback0` чтобы обработчик ISIS считал порт настроенным (работоспособным) и включил процесс на интерфейсе. IS-IS строит соседство по MAC-адресам, анонсируя только адреса Loopback0.
   -  Dual-stack: В отличие от OSPF, где для одновременной поддержки IPv4 и IPv6 вам необходимо запускать два абсолютно раздельных процесса с разными базами данных (OSPFv2 и OSPFv3), IS-IS может делать это внутри одной сессии. Один процесс маршрутизации параллельно и независимо несет в себе маршруты обоих IPv4/IPv6 семейств.
2. Простота эксплуатации и автоматизация:
   - Единообразие конфигурации: Ввиду индифферентности ISIS к уникальности L3-адресации на p2p линках, конфигурация портов на всех коммутаторах становится абсолютно идентичной, что позволяет более гибко использовать системы автоматизированной настройки.
   - Чистые таблицы маршрутизации: Можно получить работоспособную и управляемую фабрику, очищенную от маршрутов p2p соединений, оставив только легко читаемые адреса конструкта Underlay-сети Loopback0.
   - Динамическое именование узлов (Dynamic Hostname): IS-IS автоматически сопоставляет длинные шестнадцатеричные NET-адреса устройств с их системными именами (hostname), которые проще использовать в диагностических целях.
3. Производительность и быстрота схождения:
   - Структура базы данных IS-IS (LSP) значительно проще, чем, например, перегруженная классификацией база данных OSPF с ее многочисленными типами LSA (Type 1, 2, 3, 5, 7).
   - Отсутствие сложной инфраструктуры и наличие полносвязной топологии распространения маршрутной информации также снижает нагрузку на CPU устройств, обеспечивая более быструю сходимость.
4. Безопасность и Защищенность:
   - Устойчивость к сниффингу и спуффингу: IS-IS работает напрямую поверх канального уровня (L2), используя формат кадров ISO 8473 (Connectionless Network Protocol, CLNP). Кадры IS-IS не имеют IP-заголовка, поэтому их невозможно маршрутизировать. Злоумышленник из внешней сети или интернета физически не может вмешиваться или просматривать такие кадры удаленно.
   - Встроенная крипто-аутентификация: IS-IS имеет собственные механизмы защиты (включая стойкие алгоритмы хэширования HMAC-MD5 и современные SHA), работающие на уровне L2.
   
Я буду использовать современный плоский дизайн, основанный на уровне Level2 ISIS.





Настройка ECMP (Equal-Cost Multi-Path) в протоколе IS-IS сводится к двум основным шагам: включению поддержки многопутевой маршрутизации в самом процессе IS-IS и настройке балансировки на уровне коммутатора/маршрутизатора (так как IS-IS лишь находит равные маршруты, а распределяет трафик само железо).


В операционной системе Arista EOS протокол IS-IS из коробки отлично оптимизирован для работы в Leaf-Spine фабриках. По умолчанию в EOS уже разрешено использование до 16 путей ECMP, однако для полноценной работы многопутевой маршрутизации на аппаратном уровне дата-центра необходимо выполнить несколько обязательных шагов в конфигурации.


но отсутствует весь стек

   
    Современный плоский дизайн Level 2-Only позволяет процессорам Arista EOS легко переваривать топологии на тысячи устройств, моментально рассчитывая кратчайшие пути (ECMP) [Underlay Control Plane].


metric-style wide — это команда в протоколах маршрутизации IS-IS, которая меняет формат заголовков (TLV) для передачи метрик интерфейсов, увеличивая их максимальный размер с 6 бит до 24 бит (а суммарную метрику пути — до 32 бит).


можно ли заблокировать перевыборы DIS?


Команды connect clnp (или telnet clnp) в Arista EOS не существует.
Разработчики Arista заложили в операционную систему поддержку IS-IS только как протокола маршрутизации для IP-underlay.Архитектура Arista EOS обрабатывает пакеты CLNS (к которым относится IS-IS) исключительно процессом маршрутизации (внутри ядра для построения топологии).Операционная система Arista не умеет инкапсулировать пользовательский трафик приложения (такого как SSH, Telnet или API) в CLNP-пакеты. Соответственно, поднять VTY-сервер (терминальные линии) на базе адресов NET невозможно.



??? Почему будем использовать Level-2


- Изоляция базы данных через пассивные интерфейсы: С помощью команды passive-interface default вы намертво закрываете клиентские порты (куда подключены серверы VMware ESXi или корпоративные хосты) [Underlay Control Plane]. Они никогда не смогут отправить фейковый кадр IS-IS и стать частью вашей Underlay-инфраструктуры


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

В протоколе OSPFv3 для IPv6 отображение маршрутов через Link-Local адреса (fe80::...) — это стандартное поведение, определенное спецификацией RFC 5340.

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

#### Настройка BFD
Настройка BFD для OSPFv2/v3 достаточно тиривиальна и осуществляется через контекст экземпляра динамического протокола.

Для IPv4 произведем следующие настройки:
```
router ospf 4
   bfd default
```

Аналогичную настройку производим на всех коммутаторах фабрики.

Состояние соседства проверим с помощью команды:
```
swSpine01#sh bfd peers 
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.2.1  2995816412   822533371        Ethernet1(14)  normal   08/18/26 11:00 
10.1.2.2   852903916   648686176        Ethernet2(15)  normal   08/18/26 11:01 
10.1.2.3   495383476  1248090569        Ethernet3(16)  normal   08/18/26 11:01 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```

Для IPv6 настройки имеют следующий вид:
```
ipv6 router ospf 6
   bfd default
```

Настройки OSPFv3 позволяют гранулярно включать BFD в контексте соответстующего интерфейса, например:
```
interface Ethernet1
   ipv6 ospf bfd
```

> В Arista EOS контекст интерфейса является более приоритетным, чем контекст процесса динамического роутинга, что позволяет гранулярно отключить BFD на интерфейсе, занятом в процессе динамической маршрутизации глобально командой `no ipv6 ospf bfd`. Привести к поведению по-умолчанию можно префиксом  `default`.

Аналогичную настройку производим на всех коммутаторах фабрики.

Проверим соседство:
```
swSpine01#sh bfd peers ipv6
VRF name: default
-----------------
DstAddr                      MyDisc    YourDisc   Interface/Transport     Type 
------------------------ ----------- ----------- --------------------- --------
fe80::5200:ff:fe03:3766   766221159   340361787         Ethernet2(15)   normal 
fe80::5200:ff:fe15:f4e8  2851805199  3549154793         Ethernet3(16)   normal 
fe80::5200:ff:fed5:5dc0  1768109603  3156882599         Ethernet1(14)   normal 

           LastUp       LastDown            LastDiag    State
-------------------- -------------- ------------------- -----
   08/18/26 11:13             NA       No Diagnostic       Up
   08/18/26 11:12             NA       No Diagnostic       Up
   08/18/26 11:13             NA       No Diagnostic       Up
```

В больших сетях можно ускорить построение сети OSPF за счет использования команды `bfd adjacency state any` которая убирает одну из главных задержек на старте: OSPF больше не ждет, пока BFD-сессия выполнит свое «трехстороннее рукопожатие» (Three-way handshake) и перейдет в статус Up. Соседство OSPF начинает строиться немедленно, параллельно с запуском BFD при его любом состоянии (Up, Down или Init).

### Защита Underlay Control Plane
В широком смысле, защита плоскости Underlay должна строиться на защите процесса OSPF от
- сниффинга (прослушивания маршрутов) через прямое подключение к фабрике;
- спуффинга (подмены) данных, отсылаемых участниками в открытом режиме.

#### Защита от сниффинга
Для защиты от сниффинга нам необходимо запретить трансляцию OSPF-трафика на нелегитимных интерфейсах. Я так же стараюсь отключать неиспользуемые интерфейсы административно (что уже было сделано на этапе выполнения лабораторной 1).

В контексте конфигурирования экземпляра OSPFv2 (IPv4) произведем следующие настройки на стороне swSpine01:
```
router ospf 4
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   no passive-interface Loopback0
```

> При конфигурировании в контексте инстанца можно использовать диапазоны, например, `no passive-interface Ethernet1 - 3`

Для защиты экземпляра OSPFv3 (IPv6) настройки будут аналогичны:
```
ipv6 router ospf 6
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   no passive-interface Loopback0
```

На остальных коммутаторах используются те же настройки.

#### Защита от спуффинга
Механизмы защиты от подмены трафика различны для разных версий протокола: если OSPFv2 (IPv4) имеет встроенный механизм защиты, то OSPFv3 (IPv6) полностью полягагется на внешний для него механиз IPSEC.

##### OSPFv2
OSPFv2 поддерживает только механизм аутентификации либо открытым текстом, либо с использованием дайджеста (Message-Digest Authentication).

При аутентификации открытым текстом, пароль передается по сети в открытом виде и может быть скомпроментирован сниффингом, поэтому не может считаться надежным.

В EOS настройка производится в контексте интерфейса:
```
swSpine01(config)#interface ethernet 1
swSpine01(config-if-Et1)#ip ospf authentication
swSpine01(config-if-Et1)#ip ospf authentication-key KEY
```

где "KEY" - применяемый ключ.

> Следуется отметить, что шифрование ключа производится только в конфигурации устройства. По сети он передается в открытом виде.

Использование дайджеста позволяет повысить уровень безопасности. Это специальный криптографический метод, позволяющий на основе напередзаданного ключа сформировать подпись, стойкую к взлому за счет хэширования смеси данных и самого ключа.

> Механизм Message-Digest Authentication обеспечивает только аутентификацию (что и следует из названия) и целостность передаваемых данных. Шифрования самой информации, при этом НЕ происходит.

Настройка также производится в контексте интерфейса:
```
swSpine01(config)#interface ethernet 1
swSpine01(config-if-Et1)#ip ospf authentication message-digest
```

здесь подразумевается использование того же ключа, но его, при желании, можно изменить.

##### OSPFv3
Протокол OSPFv3, как было сказано ранее, не имеет собственных механизмов безопасности, но при этом использует стандартный механизм IPSEC, являющийся модульным набором протоколов и стандартов, разработанных для защиты данных в сетевой среде.

Для защиты чувствительной служебной информации, передаваемой участниками OSPF-процесса может быть использовано 2 механизма: либо только аутентификация между участниками процесса, либо полное шифрования трафика. Нельзя запустить безопасное шифрование, отказавшись от аутентификации, так как если трафик будет просто зашифрован, но устройства не будут проверять, кто его прислал, злоумышленник сможет устроить атаку повтора (Replay Attack) — перехватить зашифрованный пакет и отправить его позже, нарушив работу сети.

В Arista EOS настройка может быть произведена глобально в контексте каждой области процесса, например:
```
swSpine01(config)#ipv6 router ospf 6
swSpine01(config-router-ospf3)#area 0 authentication ipsec spi 1 sha1 passphrase PASSPHRASE
```

где:
- spi (Security Parameter Index) — индекс параметров безопасности, некоторый 32-битный индентификатор ключевой фразы, используемый для быстрого поиска внутри процесса IPSEC. Должен быть одинаковым на уровне всей зоны на всех устройствах при глобальном методе конфигурирования.
- <PASSPHRASE> - кодовая фраза.

Arista EOS поддерживает и более гранулярный метод через контекст интерфейса, например:
```
swSpine01(config)#interface ethernet 1
swSpine01(config-if-Et1)#ipv6 ospf encryption ipsec spi 2 esp aes-256-cbc sha1 passphrase PASSPHRASE
```

где на уровне ESP используется алгоритм AES-256-CBC, алгоритма проверки целостности SHA1 и кодовая фраза <PASSPHRASE> для авторизации участников.

Как и в случае с BFD, в Arista OES приоритет интерфейса выше глобального, что дает возможность изменить поведение средств безопасности, перекрыв настройками контекста конкретного интерфейса.
