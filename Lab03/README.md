# Lab03: Построение Underlay сети (ISIS)

## Состав работы
- [Условие задачи](#условие-задачи)
- [Описание решения и адресация CNLP](#описание-решения-и-адресация-clnp)
- [Стартовая конфигурация](#стартовая-конфигурация)
   - [Стартовая конфигурация коммутуторов уровня Spine](#стартовая-конфигурация-коммутуторов-уровня-spine)
   - [Стартовая конфигурация коммутуторов уровня Leaf](#стартовая-конфигурация-коммутуторов-уровня-leaf)
- [### Конфигурирование Underlay сети с использованием ISIS](#конфигурирование-underlay-сети-с-использованием-isis)
- [Тюниннг и дополнительные настройки](#тюниннг-и-дополнительные-настройки)
   - [Защита от перегрузки при загрузке](#защита-от-перегрузки-при-загрузке)
   - [Агрессивные таймеры генерации LSP](#агрессивные-таймеры-генерации-lsp)
- [Поддержка BFD](#поддержка-bfd)
- [Поддержка ECMP](#поддержка-ecmp)
- [Защита Underlay Control Plane](#защита-underlay-control-plane)
   - [Защита от сниффинга](#защита-от-сниффинга)
   - [Защита от спуффинга](#защита-от-спуффинга)
- [Отказоустойчивость управляющего плана](#отказоустойчивость-управляющего-плана)

### Условие задачи
В этой самостоятельной работе мы ожидаем, что вы самостоятельно:
1. Настроите ISIS в Underlay сети, для IP связанности между всеми сетевыми устройствами.
2. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств
3. Убедитесь в наличии IP связанности между устройствами в ISIS домене

### Описание решения и адресация CLNP
Протокол ISIS, с точки зрения построения слоя Underlay, по моему мнению, является наиболее технологичным, гибким, масштабируемым и отказоустойчивым вариантом.

На Underlay-уровне (базовой физической сети) лучше использовать протоколы Link-State (OSPF, IS-IS), а не дистанционно-векторные (RIP, EIGRP), потому что они обеспечивают минимальное время сходимости, отсутствие топологических петель и точную информацию о карте сети, требуемую для overlay-технологий.

Кроме того, дистанционно-векторные протоколы передают только готовую таблицу маршрутизации («вектор» и «расстояние»). Если на каком-то участке сети происходит сбой, маршрутизаторы тратят время на пересчет и обмен слухами. Link-State протоколы передают состояние линков — каждый роутер сам строит точную математическую карту сети и моментально находит лучший путь.

***Можно выделить следующие преимущества ISIS***:
1. **Архитектурная независимость, масштабируемость и живучесть**:
   - *Масштабируемость и обратная совместимость*: Благодаря использованию структуры TLV (Type-Length-Value), протокол IS-IS обладает исключительной масштабируемостью и обратной совместимостью. При внедрении новых сетевых протоколов или стандартов адресации архитектура IS-IS не требует фундаментальной переработки программного кода (в отличие от OSPF, где для поддержки IPv6 потребовалось создание отдельной версии OSPFv3). Разработчикам сетевого оборудования достаточно специфицировать и добавить новые типы TLV-полей, что существенно упрощает интеграцию инноваций на протяжении всего жизненного цикла технологии.
   - *Полная изоляция от IP-стека (Чистый L2-транспорт)*: IS-IS работает напрямую на канальном уровне (Layer 2), используя кадры CLNP. Если на коммутаторе из-за ошибки в конфигурации, сбоя скрипта автоматизации или аварии полностью «отвалится» IPv4/IPv6-стек, сам IS-IS продолжит жить и видеть соседей.
   - *Диагностика и управление через CLNP (Connect over CLNS)*: Благодаря независимости от IP, сохраняется уникальную возможность подключиться к текстовой консоли удаленного коммутатора через L2-кадры IS-IS (`connect clns <HOSTNAME>`). Однако, не все производители интегрируют полный стек CLNP в свои продукты, что создает некоторые ограничения его использования.
   - *Экономия IP-адресов*: На физических p2p стыках между уровнями Spine и Leaf L3-адреса носят чисто технический характер — достаточно навесить L3-адрес любым доступным способом через `ipv6 enable` (Link-local) или `ip unnumbered Loopback0` чтобы обработчик ISIS считал порт настроенным (работоспособным) и включил процесс на интерфейсе. IS-IS строит соседство по MAC-адресам, анонсируя только адреса Loopback0.
   -  *Dual-stack архитектура*: В отличие от OSPF, где для одновременной поддержки IPv4 и IPv6 вам необходимо запускать два абсолютно раздельных процесса с разными базами данных (OSPFv2 и OSPFv3), IS-IS может делать это внутри одной сессии. Один процесс маршрутизации параллельно и независимо несет в себе маршруты обоих IPv4/IPv6 семейств.
2. **Простота эксплуатации и автоматизация**:
   - *Единообразие конфигурации*: Ввиду индифферентности ISIS к уникальности L3-адресации на p2p линках, конфигурация портов на всех коммутаторах становится абсолютно идентичной, что позволяет более гибко использовать системы автоматизированной настройки.
   - *Чистые таблицы маршрутизации*: Можно получить работоспособную и управляемую фабрику, очищенную от маршрутов p2p соединений, оставив только легко читаемые адреса конструкта Underlay-сети Loopback0.
   - *Динамическое именование узлов (Dynamic Hostname)*: IS-IS автоматически сопоставляет длинные шестнадцатеричные NET-адреса устройств с их системными именами (hostname), которые проще использовать в диагностических целях.
3. П**роизводительность и быстрота схождения**:
   - Структура базы данных IS-IS (LSP) значительно проще, чем, например, перегруженная классификацией база данных OSPF с ее многочисленными типами LSA (Type 1, 2, 3, 5, 7).
   - Отсутствие сложной инфраструктуры и наличие полносвязной топологии распространения маршрутной информации также снижает нагрузку на CPU устройств, обеспечивая более быструю сходимость.
4. **Безопасность и Защищенность**:
   - *Устойчивость к сниффингу и спуффингу*: IS-IS работает напрямую поверх канального уровня (L2), используя формат кадров ISO 8473 (Connectionless Network Protocol, CLNP). Кадры IS-IS не имеют IP-заголовка, поэтому их невозможно маршрутизировать. Злоумышленник из внешней сети или интернета физически не может вмешиваться или просматривать такие кадры удаленно.
   - *Встроенная крипто-аутентификация*: IS-IS имеет собственные механизмы защиты (включая стойкие алгоритмы хэширования HMAC-MD5 и современные SHA), работающие на уровне L2.

Есть смысл использовать на всей подстилающей сети плоский дизайн, растянутый на весь DC. 

Поскольку маршрутизаторы магистрального уровня IS-IS Level 2 обладают полной информацией обо всех префиксах в пределах домена и, в отличие от Level 1, обеспечивают сквозную маршрутизацию между различными областями, для построения межсайтовых соединений на всех устройствах целесообразно использовать именно этот уровень.

При этом протокол IS-IS на своем магистральном уровне (будь то Level 1 или Level 2) использует именно классический алгоритм Дейкстры (SPF — Shortest Path First).

> Для работы VXLAN/EVPN BGP требуется сквозная связность между всеми Loopback-адресами фабрики. Режим L2-Only обеспечивает ее в максимально «чистом» виде.

Перед тем, как приступить к настройке устройств, необходимо разработать адресацию CLNP. В рамках одного DC будем использовать одну зону с полем Area ID: `0001`. В качестве System ID будем использовать IPv4-адрес интерфейса `Loopback0` коммутаторов, записанный тетрадами.

*Таблица 1: Net-address*

| Hostname | Lb0 IPv4 | AFI | Area ID | System ID | NSEL |
|------------|-----------|-----------|-----------------|
| swSpine01 | 10.1.0.1/32 | 49 | 0001 | 0100.0100.0001 | 00 |
| swSpine02 | 10.1.0.2/32 | 49 | 0001 | 0100.0100.0002 | 00 |
| swLeaf01 | 10.1.2.1/32 | 49 | 0001 | 0100.0100.2001 | 00 |
| swLeaf02 | 10.1.2.1/32 | 49 | 0001 | 0100.0100.2002 | 00 |
| swLeaf03 | 10.1.2.1/32 | 49 | 0001 | 0100.0100.2003 | 00 |

### Стартовая конфигурация
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
   mtu 9214
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

### Конфигурирование Underlay сети с использованием ISIS
Так как ISIS нативно поддерживает Dual-stack, настройку будем производить для наиболее интересного, с моей точки зрения, протокола IPv4 с Unnumbered интерфейсами ребер связи (p2p).

> **Важно!**
> Underlay плоскость настраивается в GRT.

Начнем настройку с коммутатора swSpine01. Создадим процесс ISIS с именем `Underlay` глобально:
```
swSpine01(config)#router isis Underlay
swSpine01(config-router-isis)#net 49.0001.0100.0100.0001.00
swSpine01(config-router-isis)#is-type level-2
swSpine01(config-router-isis)#passive loopback 0               ! Включит ISIS на интерфейсе и переведет их в пассивный режим в контексте соответствующего интерфеса
swSpine01(config-router-isis)#hello padding disabled           ! Отключает раздувание пакетов Hello до максимального MTU
swSpine01(config-router-isis)#log-adjacency-changes            ! Производить журналирование событий сходимости
swSpine01(config-router-isis)#address-family ipv4 unicast      ! Без указания версии стека ISIS не будет запущен
swSpine01(config-router-isis-af)#maximum-paths 64               ! Настройка ECMP: максимальное количество путей
wSpine01(config-router-isis-af)#bfd all-interfaces
```

> По умолчанию, при включении IS-IS на интерфейсе, маршрутизатор пытается отправлять через него Hello-пакеты, чтобы найти соседей. Поскольку Loopback — это виртуальный интерфейс «внутри себя», слать туда Hello-пакеты бессмысленно.

Проверить состояние можно с помощью кломанды:
```
swSpine01#sh isis summary 
 
IS-IS Instance: Underlay VRF: default
  Instance ID: 0
  System ID: 0100.0100.0001, administratively enabled
  Router ID: IPv4: 10.1.0.1
  Hostname: swSpine01
  Multi Topology disabled, not attached
  IPv4 Preference: Level 1: 115, Level 2: 115
  IPv6 Preference: Level 1: 115, Level 2: 115
  IS-Type: Level 2, Number active interfaces: 1
  Routes IPv4 only
  LSP size maximum: Level 1: 1492, Level 2: 1492
                            Max wait(s) Initial wait(ms) Hold interval(ms)
  LSP Generation Interval:     5              50               50
  SPF Interval:                2            1000             1000
  Current SPF hold interval(ms): Level 1: 0, Level 2: 1000
  Last Level 2 SPF run 33 seconds ago
  CSNP generation interval: 10 seconds
  Dynamic Flooding: Disabled
  Authentication mode: Level 1: None, Level 2: None
  Graceful Restart: Disabled, Graceful Restart Helper: Enabled
  Area addresses: 49.0001
  level 2: number DIS interfaces: 0, LSDB size: 1
    Area Leader: None
    Overload Bit is not set. 
  Redistributed Level 1 routes: 0 limit: Not Configured
  Redistributed Level 2 routes: 0 limit: Not Configured
```

Далее, настраиваем контекст интерфейсов. Локальная петля:
```
interface Loopback0
   isis enable Underlay
```

> Перевести режим интерфейса в пассивный можно и в контексте самого интерфейса с помощью команды `isis passive`.

и интерфейсы ребер p2p:
```
swSpine01(config)#interface ethernet 1 - 3
swSpine01(config-if-Et1)#isis enable Underlay
swSpine01(config-if-Et1)#isis network point-to-point
```

Аналогичные настройки распространим на остальные коммутаторы схемы.

После настройки всех коммутаторов, проверим соседство:
```
swSpine01#sh isis neighbors detail 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
Underlay  default  swLeaf01         L2   Ethernet1          P2P               UP    29          0F                  
  Area addresses: 49.0001
  SNPA: P2P
  Router ID: 0.0.0.0
  Advertised Hold Time: 30
  State Changed: 09:59:41 ago at 2026-08-20 21:09:13
  IPv4 Interface Address: 10.1.2.1
  IPv6 Interface Address: none
  Interface name: Ethernet1
  Graceful Restart: Supported 
  BFD IPv4 state is Up
  Supported Address Families: IPv4
  Neighbor Supported Address Families: IPv4
Underlay  default  swLeaf02         L2   Ethernet2          P2P               UP    22          0F                  
  Area addresses: 49.0001
  SNPA: P2P
  Router ID: 0.0.0.0
  Advertised Hold Time: 30
  State Changed: 00:08:59 ago at 2026-08-21 06:59:55
  IPv4 Interface Address: 10.1.2.2
  IPv6 Interface Address: none
  Interface name: Ethernet2
  Graceful Restart: Supported 
  BFD IPv4 state is Up
  Supported Address Families: IPv4
  Neighbor Supported Address Families: IPv4
Underlay  default  swLeaf03         L2   Ethernet3          P2P               UP    22          0F                  
  Area addresses: 49.0001
  SNPA: P2P
  Router ID: 0.0.0.0
  Advertised Hold Time: 30
  State Changed: 00:00:25 ago at 2026-08-21 07:08:29
  IPv4 Interface Address: 10.1.2.3
  IPv6 Interface Address: none
  Interface name: Ethernet3
  Graceful Restart: Supported 
  BFD IPv4 state is Up
  Supported Address Families: IPv4
  Neighbor Supported Address Families: IPv4
```

База данных:
```
swSpine01#sh isis database 

IS-IS Instance: Underlay VRF: default
  IS-IS Level 2 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    swSpine01.00-00              50  57384   985    110 L2 <>
    swSpine02.00-00               3  29942   989     99 L2 <>
    swLeaf01.00-00               48  44526   857     87 L2 <>
    swLeaf02.00-00                3  58056   633     98 L2 <>
    swLeaf03.00-00                3  40456   989     98 L2 <>

```

Как видно, база данных ISIS содержит все устройства фабрики.

Проверим RIB:
```
swSpine01#sh ip route isis 

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

 I L2     10.1.0.2/32 [115/30] via 10.1.2.2, Ethernet2
                               via 10.1.2.3, Ethernet3
 I L2     10.1.2.1/32 is directly connected, Ethernet1
 I L2     10.1.2.2/32 is directly connected, Ethernet2
 I L2     10.1.2.3/32 is directly connected, Ethernet3
```

и связанность:
```
swSpine01#ping 10.1.0.2 repeat 1
PING 10.1.0.2 (10.1.0.2) 72(100) bytes of data.
80 bytes from 10.1.0.2: icmp_seq=1 ttl=63 time=24.9 ms

--- 10.1.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 4ms
rtt min/avg/max/mdev = 24.956/24.956/24.956/0.000 ms
swSpine01#ping 10.1.2.1 repeat 1
PING 10.1.2.1 (10.1.2.1) 72(100) bytes of data.
80 bytes from 10.1.2.1: icmp_seq=1 ttl=64 time=7.89 ms

--- 10.1.2.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 7.892/7.892/7.892/0.000 ms
swSpine01#ping 10.1.2.2 repeat 1
PING 10.1.2.2 (10.1.2.2) 72(100) bytes of data.
80 bytes from 10.1.2.2: icmp_seq=1 ttl=64 time=13.1 ms

--- 10.1.2.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 13.189/13.189/13.189/0.000 ms
swSpine01#ping 10.1.2.3 repeat 1
PING 10.1.2.3 (10.1.2.3) 72(100) bytes of data.
80 bytes from 10.1.2.3: icmp_seq=1 ttl=64 time=9.77 ms

--- 10.1.2.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 9.779/9.779/9.779/0.000 ms
```

На этом основной функционал можно счить настроенным.

Дополнительно следует отметить, что настройку типа связи возможно было произвести и в контексте интерфейса, используя команду `isis circuit-type level-2`. Однако, это имеет смысл, когда роутер работает в режиме Level-1-2.

> ВНИМАНИЕ!
> Необходимо убедиться, что L3 MTU строго совпадает на обоих концах провода, либо отключить дополнение пакетов командой no `isis hello padding` в контексте интерфейса или глобально - в контексте экземпляра процесса ISIS (`hello padding disabled`).
>
> На современных скоростных интерфейсах (10G, 25G, 100G, 400G) стандартных значений от 1 до 63 физически не хватает для адекватного рассчета стоимости путей. Более того, если в сети больше 16 хопов с максимальной метрикой, общая стоимость превысит 1023, и сеть просто перестанет сходиться. Для митигации необходимо использовать расширенные метрики (`metric-style wide`) вместо классических (`narrow`). В текущих версиях, в инженеры Arista полностью убрали из кода поддержку старых 6-битных (narrow) метрик, посчитав их устаревшим рудиментом.

### Тюниннг и дополнительные настройки
Ниже собраны дополнительные настройки, которые могут применяться факультативно для Arista EOS.

#### Защита от перегрузки при загрузке
Когда коммутатор Spine перезагружается, его процесс IS-IS поднимается раньше, чем полностью инициализируются внутренние таблицы коммутации (ASIC). Если Spine сразу начнет анонсировать себя, Leaf-коммутаторы пустят через него трафик, и пакеты начнут дропаться (черная дыра).

Для митигации такого состояния существует механизм перегрузки (Overload), который можно выставить в контексте экземпляра процесса ISIS:
```
router isis Underlay
   set-overload-bit on-startup 300
```

Период указывается в секундах: 300 секунд = 5 минут, отсчитываемых от времени запуска ОС.

EOS позволяет выставить бит перегрузки (Overload bit) до окончания процесса загрузки BGP-процесса.

#### Агрессивные таймеры генерации LSP
При использовании ISIS на подстилающей сети критически важным является быстрая реакция на изменения топологии. Если линк падает, коммутатор должен мгновенно сгенерировать новое объявление (LSP) и разослать его соседям.

Управление таймером производится также в контексте экземплара процесса:
```
router isis Underlay
   lsp-gen-interval 5 50 50
```
где:
- 5 (lsp-max-wait): Максимальное время ожидания в секундах между генерациями LSP. Если сеть долгое время штормит, протокол зажмет отправку обновлений и будет отправлять их не чаще, чем раз в 5 секунд.
- 50 (lsp-initial-wait): Начальная задержка в миллисекундах перед генерацией самого первого LSP после изменения топологии. Это обеспечивает мгновенную сходимость (Fast Convergence). Как только линк упал, роутер ждет всего 50 мс и сразу рассылает об этом информацию.
- 50 (lsp-second-wait): Задержка в миллисекундах между первой и второй генерацией LSP, которая также служит базовым шагом для инкремента.

Данная настройка является аналогом Cisco'вской `timers lsp generation 5 50 50`.

### Поддержка BFD
ISIS, как и большинство протоколов маршрутизации, поддерживает BFD.

Включение протокола может производиться на в контексте экземпляра процесса (как это было сделано при настройке ISIS), так и в контексте интерфейса через использование команды
```
interface Ethernet1
   isis bfd
```

Использование BFD крайне желательно. Вкупе с настройкой таймеров LSP, влияющих на скорость работы самого алгоритма вычисления маршрутов, сигнал от BFD, поступающий в случае падения линка, создаст высокодоступную конфигурацию Underlay-сети.

IS-IS без BFD будет ждать стандартный таймаут (Hold-time), который равен 30 секундам в период которого трафик может уйти в нерабочий линк ("бэкдоится").

Проверка осуществляется следующим образом:
```
swSpine01#sh bfd peers 
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.2.1  1028649901  3486473988        Ethernet1(14)  normal   08/20/26 21:09 
10.1.2.2  2900926268  3179794147        Ethernet2(15)  normal   08/21/26 06:59 
10.1.2.3  1447908443   143109816        Ethernet3(16)  normal   08/21/26 07:08 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```

### Поддержка ECMP
В операционной системе Arista EOS протокол IS-IS из коробки отлично оптимизирован для работы в Leaf-Spine фабриках и включен по-умолчанию (разрешено использование до 16 путей). 

Как было показано выше, настройка производится в контексте семейства протокола экземпляра процесса ISIS

Я не нашел, как в Arista EOS проверить настройки ECPM, кроме присутствия соответствующих строк в конфигурации.

### Защита Underlay Control Plane
В широком смысле, защита плоскости Underlay должна строиться на защите процесса ISIS от
- сниффинга (прослушивания маршрутов) через прямое подключение к фабрике;
- спуффинга (подмены) данных, отсылаемых участниками в открытом режиме.

#### Защита от сниффинга
Для защиты от сниффинга нам необходимо запретить трансляцию ISIS-трафика на нелегитимных интерфейсах. Я так же стараюсь отключать неиспользуемые интерфейсы административно (что уже было сделано на этапе выполнения лабораторной 1).

На оборудовании Arista все интерфейсы по-умолчанию рассматриваются как не включенные в процесс ISIS. В отличии от OSPF, директива `passive-interface` включает на интерфейсе пассивный режим: его префикс анонсируется в сеть, но сам интерфейс не участвует в отправке Hello-пакетов и установлении соседства.

#### Защита от спуффинга
Протокол ISIS поддерживает аутентификации управляющего плана (Control Plane), что определено в самой фундаментальной архитектуре стандарта ISO 10589, специфицирующей протокол IS-IS.

В стандарте производится разделение пакетов для аутентификации:
- **Инфраструктурный уровень**: подразумевает аутентификацию пакетов состояния каналов (LSP), а также пакеты синхронизации базы данных (CSNP и PSNP). Настраивается в контексте экземпляра процесса ISIS.
- **Уровень установления соседства**: подразумевает аутентификацию сторон при обмене Hello-пакетами (IIH - IS-to-IS Hello PDU). Настраивается в контекстах интерфейсов, подразумевающих соседство.

Оба контекста настраиваются единообразно, а опции носят эквивалентный смысл.

ISIS поддерживает следующие режимы аутентификации:
- text (Cleartext): Пароль передается в TLV-поле в абсолютно открытом виде. Поддерживается исключительно для обратной совместимости со старым оборудованием и не рекомендуется к использованию.
- md5 (HMAC-MD5): Классический криптографический стандарт для большинства сетей с передачей пароля в хэшированом виде. Имеет более высокую криптостойкость, но с точки зрения современных требований ИБ  считается устаревшим и спорную математическую стойкость.
- sha (HMAC-SHA): Самый современный и безопасный режим. Обеспечивает максимальную стойкость к атакам методом перебора и коллизий.

Кроме описанного выше стандартного механизма аутентификации, Arista EOS поддерживает более современный 
фреймворк управления улючами - это бесшовной механизм смены паролей (Hitless Authentication Key Rollover).

Смысл этого механизма сводится к следующему. В классическом случае, чтобы поменять пароль, инженеру приходилось одновременно менять его на нескольких роутерах. Это вызывало разрыв соседства и сеть штормило. HAKR предусматривает возможность прописать на коммутаторе несколько ключей одновременно (например, key-id 56 и key-id 57) и настроить расписание, например, до полуночи работает 56-й ключ, после полуночи — 57-й. Коммутатор будет принимать пакеты, зашифрованные любым из валидных на данный момент key-id, что позволяет обновить пароли на всей фабрике фактически без потери данных.

Настройку начнем с инфраструктурного уровня (глобальный контекст экземпляра процесса ISIS):
```
swSpine01(config)#router isis Underlay 
swSpine01(config-router-isis)#authentication key-id 1 algorithm sha-512 key P@ssw0rd1
swSpine01(config-router-isis)#authentication mode sha key-id 1 
```

Далее, настраиваем аутентификацию при установлении соседства в контексте соответствующих интерфейсов:
```
swSpine01(config)#interface ethernet 1 - 3
swSpine01(config-if-Et1-3)#isis authentication key-id 1 algorithm sha-512 P@ssw0rd2
swSpine01(config-if-Et1-3)#isis authentication key-id 2 algorithm sha-512 P@ssw0rd2
swSpine01(config-if-Et1-3)#isis authentication mode sha key-id 2 
```

Если просмотреть конфигурацию:
```
swSpine01#show running-config section isis
interface Ethernet1
   isis enable Underlay
   isis network point-to-point
   isis authentication mode sha key-id 2
   isis authentication key-id 1 algorithm sha-512 key 7 073SzWKz9AcnZmwFcPJUug==
   isis authentication key-id 2 algorithm sha-512 key 7 073SzWKz9AcnZmwFcPJUug==
interface Ethernet2
   isis enable Underlay
   isis network point-to-point
   isis authentication mode sha key-id 2
   isis authentication key-id 1 algorithm sha-512 key 7 073SzWKz9AcnZmwFcPJUug==
   isis authentication key-id 2 algorithm sha-512 key 7 073SzWKz9AcnZmwFcPJUug==
interface Ethernet3
   isis enable Underlay
   isis network point-to-point
   isis authentication mode sha key-id 2
   isis authentication key-id 1 algorithm sha-512 key 7 073SzWKz9AcnZmwFcPJUug==
   isis authentication key-id 2 algorithm sha-512 key 7 073SzWKz9AcnZmwFcPJUug==
interface Loopback0
   isis enable Underlay
   isis passive
router isis Underlay
   hello padding disabled
   net 49.0001.0100.0100.0001.00
   is-type level-2
   log-adjacency-changes
   authentication mode sha key-id 1
   authentication key-id 1 algorithm sha-512 key 7 073SzWKz9AfaVuWj9oGliQ==
   !
   address-family ipv4 unicast
      maximum-paths 64
      bfd all-interfaces
```

видно, что идентификаторы ключей не пересекаются для разных уровней аутентификации, а сам идентификатор имеет локальную область видимости: `key-id` не пересылается между участниками процесса ISIS.

### Отказоустойчивость управляющего плана
Под аварийным режимом будем понимать состояние, возникшее в состоянии конфигурирования коммутатора, когда в результате действий инженера доступность устройства может быть утеряна.

При использовании IP-адресации на гранях соединений p2p у инженера остается возможность подключения через использование IP-адреса интерфейса. Однако, при использовании Unnumbered такая возможность отсутствует. Для обеспечения отказоустойчивости производителями рекомендуется использовать отдельную сеть управления Out-of-Band.

Однако, при использовании ISIS, у бОльшого количества производителей остается возможность доступа через In-Band подлежащей сети через использование протокола CLNS. Например, на оборудовании Cisco Systems можно использовать команды `ping clns swLeaf03` или `ping clns 49.0001.0100.0100.2003.00` для проверки доступности устройства используя имя устройста или его NET-адрес.

Кроме того, на таком оборудовании остается возможность подключения к терминалу удаленного устройства, используя команду `connect clns swLeaf03`.

Однако, разработчики Arista заложили в операционную систему поддержку CLNP только на уровне IS-IS. Т.е., архитектура Arista EOS обрабатывает пакеты CLNS исключительно процессом маршрутизации (внутри ядра для построения топологии), но не умеет инкапсулировать пользовательский трафик приложений, таких как SSH, Telnet или API в CLNP-пакеты. Соответственно, поднять VTY-сервер на базе адресов NET на Arista невозможно. Но есть хинт: можно для этого использовать Link-local адреса IPv6, включив поддержку протокола на интерфейсах. Тогда любые проблемы, возникшие в IPv4, на котором работает слой Underlay не скажутся на возможности аварийного доступа и к оборудованию Arista EOS.

Например доступность коммутатора swLeaf01 со стороны swSpine01 через адрес `ff02::1` (все устройства на канале):
```
swLeaf01#ping ipv6 ff02::1 interface ethernet 1
PING ff02::1(ff02::1) from fe80::5200:ff:fed5:5dc0%et1 et1: 52 data bytes
60 bytes from fe80::5200:ff:fed5:5dc0%et1: icmp_seq=1 ttl=64 time=0.558 ms
60 bytes from fe80::5200:ff:fed5:5dc0%et1: icmp_seq=2 ttl=64 time=0.135 ms
60 bytes from fe80::5200:ff:fed5:5dc0%et1: icmp_seq=3 ttl=64 time=0.134 ms
60 bytes from fe80::5200:ff:fed5:5dc0%et1: icmp_seq=4 ttl=64 time=0.144 ms
60 bytes from fe80::5200:ff:fed5:5dc0%et1: icmp_seq=5 ttl=64 time=0.152 ms
```

Но, к сожалению, Arista не дает возможность подключения таким же образом. Для успешной работы telnet и ssh требуется Unicast-адрес соседа: его глобальный IP или конкретный Link-Local вида fe80::... Получить его мы можем добавив обработку IPv6 в контексте экземпляра процесса, либо, чтобы не загроможность RIB Link-local адресами, следующим образом:
```
swLeaf01#show ipv6 neighbors 
IPv6 Address                                  Age Hardware Addr    State Interface
fe80::5200:ff:fed7:ee0b                   3:22:44 5000.00d7.ee0b   REACH Et1
fe80::5200:ff:fecb:38c2                   1:04:07 5000.00cb.38c2   REACH Et2
```
Проверить доступность можно встроенными средствами OES:
```
swLeaf01#ping ipv6 fe80::5200:ff:fed7:ee0b interface Ethernet 1
PING fe80::5200:ff:fed7:ee0b(fe80::5200:ff:fed7:ee0b) from fe80::5200:ff:fed5:5dc0%et1 et1: 52 data bytes
60 bytes from fe80::5200:ff:fed7:ee0b%et1: icmp_seq=1 ttl=64 time=9.34 ms
60 bytes from fe80::5200:ff:fed7:ee0b%et1: icmp_seq=2 ttl=64 time=6.41 ms
60 bytes from fe80::5200:ff:fed7:ee0b%et1: icmp_seq=3 ttl=64 time=7.95 ms
60 bytes from fe80::5200:ff:fed7:ee0b%et1: icmp_seq=4 ttl=64 time=18.7 ms
60 bytes from fe80::5200:ff:fed7:ee0b%et1: icmp_seq=5 ttl=64 time=14.6 ms

--- fe80::5200:ff:fed7:ee0b ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 50ms
rtt min/avg/max/mdev = 6.417/11.405/18.703/4.575 ms, pipe 2, ipg/ewma 12.512/10.648 ms
```

а вот для соединения, например, по SSH, потребуется уже хинт с коммандным процессором `bash`:
```
swLeaf01#bash ssh admin@fe80::5200:ff:fed7:ee0b%et1
Warning: Permanently added 'fe80::5200:ff:fed7:ee0b%et1' (ECDSA) to the list of known hosts.
Password: 
```

> Номер интерфейса необъодимо задавать строчными буквами.