# LAb01: Основы проектирования сети

## Состав работы
- [Условие задачи](#условие-задачи)
- [Схема сети](#схема-сети)
- [Расчет емкости ЦОД](#расчет-емкости-цод)
- [Распределение адресного пространства для Underlay сети](#распределение-адресного-пространства-для-underlay-сети)
   - [IPv6-план](#ipv6-план)
   - [IPv4-план](#ipv4-план)
- [Конфигурирование Undelay](#Конфигурирование-Undelay)
   - [Настройка базовой связанности](#Настройка-базовой-связанности)
   - [Настройка соединений фабрики](#настройка-соединений-фабрикии)
      - [Стек IPv4](#стек-ipv4)
      - [Стек IPv6](#стек-ipv6)
- [Настройка плоскости OOB (дополнительно)](#настройка-плоскости-oob-дополнительно)

### Условие задачи
В этой самостоятельной работе мы ожидаем, что вы самостоятельно:
1. Соберете топологию CLOS, как в прило схеме:
2. Распределите адресное пространство для Underlay сети
Зафиксируете в документации план работ, адресное пространство, схему сети, настройки (если перенесли на оборудование)

### Схема сети
![Схема сети](image.png)

### Расчет емкости ЦОД
Будем считать, что DC представляет собой коммерческий микро-ЦОД следующей конфигурации:
- Цод предназначен для размещения 2 одинаковых POD'ов (модульных блоков), состоящих из 8 коммуникационных конструктивов высотой 48U;
- Каждый POD имеет следующую конфигурацию:
    - 6 конструктивов стандартного типа, предназначенных для размещения серверного оборудования. Каждый конструктив оснащен 2 коммутаторами TOP (Top of Rack) уровня Leaf, размещенными с задней стороны конструктива в специализированных местах установки (так называемых Zero Unit);
    - 1  коммуникационный конструктив, содержащий в своем объеме весь необходимый connectivity и предназначенный для установки проектируемого количества коммутаторов уровня Spine;
    - 1 специализированный конструктив распределения энергии.

Опираясь на рассматриваемое ТЗ, общая емкость составляет 288U/POD или 576U проектной емкости (максимальной) всего ЦОД.

Исходя из полученных данных и условия, что на уровне Leaf применяются коммутаторы с 48 downlink портами, нам понадобится 12 пар без учета 1 пары Border Leaf. Итого: 13 пар или 26 коммутаторов уровня Leaf.

При расчете количества Spine коммутаторов, будем исходить из условия заданного коэффициента переподписки 2/1 (oversubscription ratio) и полосы пропускания downlink на стороне Leaf-коммутатора в 10Gbps:

`480Gbps/2 ~ 250Gbps`

Т.о., при использовании 50Gbps аплинтов нам потребуется 5 коммутаторов уровня Spine и по 6 портов аплинков на стороне Leaf с соответствующей полосой пропускания.

### Распределение адресного пространства для Underlay сети
В рамках данного курса хочу, в качестве эксперемента, реализовать Dual-Stack решение, с целью одновременного тестирования различных программных архитектур на базе единой физической модели.

Архитектурное разделение трафика на уровни IPv4 и IPv6 позволяет параллельно реализовать и валидировать принципиально разные методы построения Underlay-сети без развертывания дополнительных стендов. В ходе эксперимента планируется симулировать и сравнить различные подходы к передаче служебного трафика (например, классический Broadcast Flood против оптимизированной Multicast-доставки), а также запустить независимые протоколы динамической маршрутизации для каждого из стеков.

Такой подход позволит наглядно оценить эффективность, ресурсоемкость и особенности эксплуатации различных программных сценариев в идентичных инфраструктурных условиях.

#### IPv6-план
Со своей точки зрения, считаю вариант использования IPv6-стека для подстилающей сети наиболее гибким с точки зрения дальнейшей интеграции или масштабирования.

Учитывая заложенные в протокол IPv6 механизмы масштабирования, будем использовать поддиапазон `2001:db8::/32`, так как он не должен быть маршрутизируемым в сети Интернет, являясь инфраструктурной сетью OOB (Out-Of-Band).

В рамках совместимости будем оперировать префиксами /64.

Структура адреса:
`2001 : db8 : <DC><Spine> : <Leaf-Type><Leaf> : <Service-Type>00 :: <Host>`

где:
- 3 хексет:
    - <DC> - определяет порядковый номер сайта/датацентра (8 бит);
    - <Spine> - порядковый номер коммутатора уровня Spine (8 бит);

- 4 хексет:
    - <Leaf-Type> - тип коммутатора уровня Leaf (8 бит):
        - 00 - зарезервировано для Spine;
        - 01 - зарезервировано для Client Edge;
        - 02 - Server Edge;
        - FF - зарезервировано для Border Leaf;
    - <Leaf> - порядковый номер коммутатора уровня Leaf (8 бит);

- 5 хексет:
    - <Service-Type> - тип сервиса (8 бит):
        - 00 — Underlay Control Plane (Loopback0);
        - 01 — Overlay Data Plane VTEP (Loopback1);
        - 02 — p2p линки между коммутаторами;
        - FF — Зарезервировано для подключения серверов AnyCast;

Т.о., данный IP план позволяет нам достаточно широко масштабироваться до 255 сайтов/ЦОД с 255 коммутаторами на каждом из уровней.

*Таблица 1: Underlay Control Plane IPv6*

| Hostname | Type | If | IPv6/Mask |
|------------|-----------|-----------|-----------------|
| swSpine01 | Spine | Loopback0 | `2001:db8:0101::1/128` |
| swSpine02 | Spine | Loopback0 | `2001:db8:0102::1/128` |
| swLeaf01 | Leaf | Loopback0 | `2001:db8:0100:0201::1/128` |
| swLeaf02 | Leaf | Loopback0 | `2001:db8:0100:0202::1/128` |
| swLeaf03 | Leaf | Loopback0 | `2001:db8:0100:0203::1/128` |

> Для мультитеннантной сети (сети с несколькими слоями Overlay) желательно иметь несколько интерфейсов, на которых терминируются VTEP-туннели. Различные IP адреса и интерфейсы желательны для более технологичного разделения сетей и возможности распределить их, например, в разные VRF.

*Таблица 2: Overlay Data Plane VTEP IPv6*

| Hostname | Type | If | IPv6/Mask |
|------------|-----------|-----------|-----------------|
| swLeaf01 | Leaf | Loopback1 | `2001:db8:0100:0201:0100:1/128` |
| swLeaf01 | Leaf | Loopback2 | `2001:db8:0100:0201:0200:1/128` |
| swLeaf02 | Leaf | Loopback1 | `2001:db8:0100:0202:0100:1/128` |
| swLeaf02 | Leaf | Loopback2 | `2001:db8:0100:0202:0200:1/128` |
| swLeaf03 | Leaf | Loopback1 | `2001:db8:0100:0203:0100:1/128` |
| swLeaf03 | Leaf | Loopback2 | `2001:db8:0100:0203:0200:1/128` |

*Таблица 3: p2p соединения фабрики IPv6*

| Link | Network | Spine IPv6 | Leaf IPv6 |
|----------------------|------------------------------|------------------------------|------------------------------|
| swSpine01 - swLeaf01 | `2001:db8:0101:0201:0201::/64` | `2001:db8:0101:0201:0201::1/64` | `2001:db8:0101:0201:0201::2/64` |
| swSpine01 - swLeaf02 | `2001:db8:0101:0202:0201::/64` | `2001:db8:0101:0202:0201::1/64` | `2001:db8:0101:0202:0201::2/64` |
| swSpine01 - swLeaf03 | `2001:db8:0101:0203:0201::/64` | `2001:db8:0101:0203:0201::1/64` | `2001:db8:0101:0203:0201::2/64` |
| swSpine02 - swLeaf01 | `2001:db8:0102:0201:0201::/64` | `2001:db8:0102:0201:0201::1/64` | `2001:db8:0102:0201:0201::2/64` |
| swSpine02 - swLeaf02 | `2001:db8:0102:0202:0201::/64` | `2001:db8:0102:0202:0201::1/64` | `2001:db8:0102:0202:0201::2/64` |
| swSpine02 - swLeaf03 | `2001:db8:0102:0203:0201::/64` | `2001:db8:0102:0203:0201::1/64` | `2001:db8:0102:0203:0201::2/64` |

> Cуммаризация в BGP будет выглядеть следующим образом:
> - На Border Leaf (при анонсе в Core-сеть): 2001:db8:0100::/40.
> - На Spine-коммутаторах для сборки EVPN-соседств достаточно прописать два суммаризированных маршрута до всех VTEP: 2001:db8:0100:0200:0100::/56

#### IPv4-план
В рамках тестового решения есть смысл проверить работоспособность решения, построенного на IPv4 адресации с использованием механизма unnumbered для p2p линков.

Для подстилающей сети будем использовать приватную сеть класса A 10.0.0.0/8.

Структура адреса:
`10.<DC>.<Type>.<Host>`

где:
- <DC> - определяет порядковый номер сайта/ЦОД (2 байта);
- <Type> - Тип устройства/сервиса (2 байта);
    - 00 - зарезервировано для Spine;
    - 01 - зарезервировано для Client Edge;
    - 02 - Server Edge;
    - 1хх - заразервировано для VTEP Edge;  
    - 255 - зарезервировано для Border Leaf.

*Таблица 4: Underlay Control Plane IPv4*

| Hostname | Type | If | IPv4/Mask |
|------------|-----------|-----------|-----------------|
| swSpine01 | Spine | Loopback0 | `10.1.0.1/32` |
| swSpine02 | Spine | Loopback0 | `10.1.0.2/32` |
| swLeaf01 | Leaf | Loopback0 | `10.1.2.1/32` |
| swLeaf02 | Leaf | Loopback0 | `10.1.2.2/32` |
| swLeaf03 | Leaf | Loopback0 | `10.1.2.3/32` |

*Таблица 5: Overlay Data Plane VTEP IPv4*

| Hostname | Type | If | IPv4/Mask |
|------------|-----------|-----------|-----------------|
| swLeaf01 | Leaf | Loopback1 | `10.1.102.1/32` |
| swLeaf01 | Leaf | Loopback2 | `10.1.202.1/32` |
| swLeaf02 | Leaf | Loopback1 | `10.1.102.2/32` |
| swLeaf02 | Leaf | Loopback2 | `10.1.202.2/32` |
| swLeaf03 | Leaf | Loopback1 | `10.1.102.3/32` |
| swLeaf03 | Leaf | Loopback2 | `10.1.202.3/32` |

### Конфигурирование Undelay
Перед началом настройки необходимо отключить процесс Zero Touch Provisioning (ZTP)командой `zerotouch cancel`, после чего коммутатор будет перегружен.

#### Настройка базовой связанности
Начальная конфигурация для коммутаторов уровня Spine выглядит следующим образом:

```ssh
hostname swSpine01
dns domain local

lldp run

ip routing

ipv6 unicast-routing

interface Loopback0
   description --- Virtual (no VRF, no VLAN): Underlay Control Plane
   load-interval 60
   ip address 10.1.0.1/32
   ipv6 address 2001:db8:101::1/128

end
```

Для второго коммутатора уровня Spine конфигурация аналогичная.

Начальная конфигурация для коммутаторов уровня Leaf выглядит следующим образом:

```config
hostname swLeaf01
dns domain local

lldp run

ip routing

ipv6 unicast-routing

interface Loopback0
   description --- Virtual (no VRF, no VLAN): Underlay Control Plane
   load-interval 60
   ip address 10.1.2.1/32
   ipv6 address 2001:db8:0100:0201::1/128

interface Loopback1
   description --- Virtual (no VRF, no VLAN): Overlay VTEP Edge-01
   load-interval 60
   ip address 10.1.102.1/32
   ipv6 address 2001:db8:100:201:100::1/128

interface Loopback2
   description --- Virtual (no VRF, no VLAN): Overlay VTEP Edge-02
   load-interval 60
   ip address 10.1.202.1/32
   ipv6 address 2001:db8:100:201:200::1/128

end
```

#### Настройка соединений фабрики
##### Стек IPv4
Соберем линки. Рассмотрим в примере соединение swSpine01-swLeaf01.

На стороне swSpine01 предварительно проверим соседство, полученное через LLDP:

```
swSpine01#sh lldp neighbors 
Last table change time   : 0:14:53 ago
Number of table inserts  : 4
Number of table deletes  : 1
Number of table drops    : 0
Number of table age-outs : 1

Port          Neighbor Device ID       Neighbor Port ID    TTL
---------- ------------------------ ---------------------- ---
Et1           swLeaf01.local           Ethernet1           120
Et2           localhost                Ethernet1           120
Et3           localhost                Ethernet1           120
```

Начнем с настройки IPv4 стека:

```config
interface Ethernet1
   description --- L3 (no VRF, no VLAN): p2p connection to swLeaf01/Et1
   load-interval 60
   no switchport
   ip address unnumbered Loopback0
```

Проверим настройки:

```
swSpine01#sh ip int br
                                                                        Address
Interface        IP Address       Status      Protocol           MTU    Owner  
---------------- ---------------- ----------- ------------- ----------- -------
Ethernet1        10.1.0.1/32      up          up                1500    Lo0    
Ethernet2        unassigned       up          up                1500           
Ethernet3        unassigned       up          up                1500           
Loopback0        10.1.0.1/32      up          up               65535           
Management1      unassigned       up          up                1500 
```

Переходим на сторону коммутатора swLeaf01 и проверим соседство:

```
swLeaf01#sh lldp neighbors 
Last table change time   : 0:08:38 ago
Number of table inserts  : 2
Number of table deletes  : 0
Number of table drops    : 0
Number of table age-outs : 0

Port          Neighbor Device ID       Neighbor Port ID    TTL
---------- ------------------------ ---------------------- ---
Et1           swSpine01.local          Ethernet1           120
```

Настраиваем соответствующий интерфейс:

```config
interface Ethernet1
   description --- L3 (no VRF, no VLAN): p2p connection to swSpine01/Et1
   load-interval 60
   no switchport
   ip address unnumbered Loopback0
```

Проверим состояние:

```
swLeaf01#sh ip int br
                                                                        Address
Interface       IP Address         Status      Protocol          MTU    Owner  
--------------- ------------------ ----------- ------------- ---------- -------
Ethernet1       10.1.2.1/32        up          up               1500    Lo0    
Ethernet2       unassigned         up          up               1500           
Loopback0       10.1.2.1/32        up          up              65535           
Loopback1       10.1.102.1/32      up          up              65535           
Management1     unassigned         up          up               1500  
```

и связанность по p2p линку сначала на себя, потом на соседа:

```
swLeaf01#ping 10.1.2.1
PING 10.1.2.1 (10.1.2.1) 72(100) bytes of data.
80 bytes from 10.1.2.1: icmp_seq=1 ttl=64 time=0.558 ms
80 bytes from 10.1.2.1: icmp_seq=2 ttl=64 time=0.199 ms
80 bytes from 10.1.2.1: icmp_seq=3 ttl=64 time=0.178 ms
80 bytes from 10.1.2.1: icmp_seq=4 ttl=64 time=0.175 ms
80 bytes from 10.1.2.1: icmp_seq=5 ttl=64 time=0.172 ms

--- 10.1.2.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 8ms
rtt min/avg/max/mdev = 0.172/0.256/0.558/0.151 ms, ipg/ewma 2.062/0.401 ms

swLeaf01#ping 10.1.0.1
connect: Network is unreachable
```

Такое поведение мы будем наблюдать до тех пор, пока не настроим маршрутизацию между сетями на p2p линке.

Для проверки работоспособности можно, например, поднять временные статические маршруты:

```config
ip route 10.1.2.1 255.255.255.255 ethernet 1    # на стороне swSpine01
ip route 10.1.0.1 255.255.255.255 ethernet 1    # на стороне swLeaf01
```

после чего ситуация изменится:

```
swLeaf01#ping 10.1.0.1
PING 10.1.0.1 (10.1.0.1) 72(100) bytes of data.
80 bytes from 10.1.0.1: icmp_seq=1 ttl=64 time=16.2 ms
80 bytes from 10.1.0.1: icmp_seq=2 ttl=64 time=16.8 ms
80 bytes from 10.1.0.1: icmp_seq=3 ttl=64 time=7.58 ms
80 bytes from 10.1.0.1: icmp_seq=4 ttl=64 time=6.82 ms
80 bytes from 10.1.0.1: icmp_seq=5 ttl=64 time=9.03 ms

--- 10.1.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 69ms
```

##### Стек IPv6
Настройку будем производить аналогично, начиная со стороны swSpine01.

Попробуем, сначала, поработать с адресами Link-local - некоторого рода аналог Unnumbered в IPv4. При таком подходе (отсутствии Unicast-адреса необходимо просто включить протокол на интерфейсе):

```config
interface Ethernet1
ipv6 enable
```

и посмотрим состояние стека:

```
swSpine01#sh ipv6 int br
Interface  Status    MTU   IPv6 Address                 Addr State  Addr Source
---------- ------- ------ ---------------------------- ------------ -----------
Et1        up       1500   fe80::5200:ff:fed7:ee0b/64   up          link local 
Lo0        up      65535   fe80::ff:fe00:0/64           up          link local 
                           2001:db8:101::1/128          up          config  
```

Аналогичную настройку необходимо будет произвести и на стороне swLeaf01.

```
swLeaf01#sh ipv6 interface brief 
Interface  Status    MTU  IPv6 Address                  Addr State  Addr Source
---------- ------- ------ ---------------------------- ------------ -----------
Et1        up       1500  fe80::5200:ff:fed5:5dc0/64    up          link local 
Lo0        up      65535  fe80::ff:fe00:0/64            up          link local 
                          2001:db8:100:201::1/128       up          config     
Lo1        up      65535  fe80::ff:fe00:0/64            up          link local 
                          2001:db8:100:201:100::1/128   up          config   
```

Теперь можно проверить связанность на p2p линке:

```
swLeaf01#ping ipv6 fe80::5200:ff:fed7:ee0b interface ethernet 1
PING fe80::5200:ff:fed7:ee0b(fe80::5200:ff:fed7:ee0b) from fe80::5200:ff:fed5:5dc0%et1 et1: 52 data bytes
60 bytes from fe80::5200:ff:fed7:ee0b%et1: icmp_seq=1 ttl=64 time=12.8 ms
60 bytes from fe80::5200:ff:fed7:ee0b%et1: icmp_seq=2 ttl=64 time=10.6 ms
60 bytes from fe80::5200:ff:fed7:ee0b%et1: icmp_seq=3 ttl=64 time=12.9 ms
60 bytes from fe80::5200:ff:fed7:ee0b%et1: icmp_seq=4 ttl=64 time=12.0 ms
60 bytes from fe80::5200:ff:fed7:ee0b%et1: icmp_seq=5 ttl=64 time=13.7 ms

--- fe80::5200:ff:fed7:ee0b ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 85ms
rtt min/avg/max/mdev = 10.670/12.465/13.749/1.049 ms, ipg/ewma 21.437/12.724 ms
```

Замечательно.

Настроим Unicast согласно плана, на стороне swSpine01:

```config
interface Ethernet1
   description --- L3 (no VRF, no VLAN): p2p connection to swLeaf01/Et1
   load-interval 60
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   ipv6 address 2001:db8:101:201:201::1/64
```

и на стороне swLeaf01:

```config
interface Ethernet1
   description --- L3 (no VRF, no VLAN): p2p connection to swSpine01/Et1
   load-interval 60
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   ipv6 address 2001:fb8:101:201:201::2/64
```

Проверим связанность:

```ssh
swLeaf01#ping ipv6 2001:db8:101:201:201::1
PING 2001:db8:101:201:201::1(2001:db8:101:201:201::1) 52 data bytes
60 bytes from 2001:db8:101:201:201::1: icmp_seq=1 ttl=64 time=51.7 ms
60 bytes from 2001:db8:101:201:201::1: icmp_seq=2 ttl=64 time=54.4 ms
60 bytes from 2001:db8:101:201:201::1: icmp_seq=3 ttl=64 time=51.6 ms
60 bytes from 2001:db8:101:201:201::1: icmp_seq=4 ttl=64 time=68.3 ms
60 bytes from 2001:db8:101:201:201::1: icmp_seq=5 ttl=64 time=64.9 ms

--- 2001:db8:101:201:201::1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 47ms
rtt min/avg/max/mdev = 51.698/58.231/68.352/7.044 ms, pipe 5, ipg/ewma 11.931/55.418 ms
```

При этом есть еще один момент, который может помочь в экономии IPv6 адресов: использование не Unicast

Остальные соединения настраиваются аналогично.

### Настройка плоскости OOB (дополнительно)
С точки зрения безопасности, имеет смысл разделение плоскостей что позволяет полностью изолировать управление (OOB), транспорт (Underlay) и клиентские сервисы друг от друга (Overlays).

Это позволит обеспечить доступ к оборудованию при авариях в остальных плоскостях в сети, связанных с потерей доступности к сетеобразующим компонентам, сервису первоначальной настройки ZTP (Zero Touch Provisioning), сбору телеметрии, серверам автоматизации (Ansible, Terraform), AAA-серверы (TACACS+/RADIUS), серверам журналирования, IDPS.

Для организации плоскости OOB будем использовать "плоскую" сеть, построенную на базе L2-коммутаторов, доступ к которой производится через брандмауэр.

*Таблица 6: OOB IP-plan*

| Hostname | Type | If | IPv4/Mask |
|------------|-----------|-----------|-----------------|
| swSpine01 | Spine | Managment1 | `10.1.254.1/32` |
| swSpine02 | Spine | Managment1 | `10.1.254.2/32` |
| swLeaf01 | Leaf | Managment1 | `10.1.254.101/32` |
| swLeaf02 | Leaf | Managment1 | `10.1.254.102/32` |
| swLeaf03 | Leaf | Managment1 | `10.1.254.103/32` |

Конфигурация будет выглядеть следующим образом:

```ssh
vrf instance plnOOB
   description --- Plane OOB: Out-Of-Band
   rd 1:255

interface Management1
   description --- L3 (vrf: plnOOB, no VLAN): Management network
   load-interval 60
   vrf plnOOB
   ip address 10.1.255.1/24

ip route vrf plnOOB 0.0.0.0/0 10.1.255.254
```