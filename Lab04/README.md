# Lab04: Построение Underlay сети (BGP)

## Состав работы
- [Условие задачи](#условие-задачи)
- [Описание решения](#описание-решения)
- [Стартовая конфигурация](#стартовая-конфигурация)
   - [Стартовая конфигурация коммутуторов уровня Spine](#стартовая-конфигурация-коммутуторов-уровня-spine)
   - [Стартовая конфигурация коммутуторов уровня Leaf](#стартовая-конфигурация-коммутуторов-уровня-leaf)
- [Конфигурирование Underlay сети с использованием BGP](#конфигурирование-underlay-сети-с-использованием-bgp)
   - [Вариант eBGP](#вариант-ebgp)
   - [Вариант iBGP](#вариант-ibgp)
   - [Вариант iBGP с конфедерацией](#вариант-ibgp-с-конфедерацией)
   - [Выводы и выбор варианта](#выводы-и-выбор-варианта)
- [Тюниннг и дополнительные настройки](#тюниннг-и-дополнительные-настройки)
   - [Оптимизация сходимости (Convergence & Timers)](#оптимизация-сходимости-convergence--timers)
   - [Тюнинг безопасности и защиты CPU (Security)](#тюнинг-безопасности-и-защиты-cpu-security)
- [Поддержка BFD](#поддержка-bfd)
- [Поддержка ECMP и UCMP](#поддержка-ecmp-и-ucmp)
- [Средства безопасности](#средства-безопасности)
- [Graceful shutdown](#graceful-shutdown)

### Условие задачи
В этой самостоятельной работе мы ожидаем, что вы самостоятельно:
1. Настроите BGP в Underlay сети, для IP связанности между всеми сетевыми устройствами. iBGP или eBGP - решать вам!
2. Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств.
3. Убедитесь в наличии IP связанности между устройствами в BGP домене.

### Описание решения
Со стороны сторонников BGP в подстилающей сети звучит следующий аргумент "зачем при необходимости использования BGP на уровне оверлея использовать другой протокол на уровне Underlay?".

Вероятно, из-за недостатка опыта, я не рассматриваю данный вариант своим фаворитом. Причины следующие:
1. BGP - дистанционно-векторный протокол, спроектированный для функционирования сети Интернет с достаточно инерционными механизмами, и как следствие, «маршрутизации по слухам» (Routing by Rumor) - маршрутизатор BGP верит на слово своим соседям о том, какие сети существуют в мире, не проверяя топологию всей сети лично. Такой подход, с моей точки зрения, не совсем подходит для развертывания подстилающей (Underlay) сети.
2. BGP - протокол маршрутизации, ориентирован на соединение (179/TCP). И сам этот факт порождает дополнительные проблемы, связанные как с таймерами транспортного уровня (L4/TCP), так и необходимостью наличия связанности между соседями.
3. Нагруженность конфигурации в сетях с большим количеством соседей - потребуется создавать BGP-соседство по количеству соединений p2p, что значительно раздует конфигурацию и усложнит ее чтение (нивелируется современным динамическим способом описания соседства).
4. Адаптация стандартного BGP-процесса для подстилающей сети потребует измения системных таймеров.

В любом случае, есть интерес рассмотреть все варианты конфигурирования сети Underlay на основе как iBGP, так и eBGP.

Основное различие между iBGP и eBGP заключается в механизмах предотвращения петель (кольцевых маршрутов) и правилах распространения сетевой информации.
- В eBGP (Exterior BGP) для защиты от петель используется атрибут AS-Path. Если маршрутизатор видит в этом списке номер своей локальной автономной системы (AS), он отклоняет такой апдейт.
- В iBGP (Interior BGP) атрибут AS-Path внутри одной автономной системы не меняется. Поэтому для предотвращения петель применяется правило расщепления горизонта (Split Horizon): маршрутизатор не передает iBGP-соседям маршруты, которые сам получил от другого iBGP-соседа. Из-за правила Split Horizon в iBGP по умолчанию требуется полносвязная топология (Full Mesh) — каждый iBGP-маршрутизатор должен иметь сессию со всеми остальными.

Для обхода этого ограничения в крупных сетях используют два метода: Route Reflectors (RR) — отражатели маршрутов, которые позволяют обходить правило Split Horizon и переход на eBGP через дробление одной большой AS на несколько мелких суб-AS. Более правильно даже сказать, что внутри конфедерации между суб-AS используется не чистый eBGP, а его особый гибридный вариант — intra-confederation eBGP.

Таким образом, начнем с eBGP и перейдем через iBGP к intra-confederation iBGP, раскрыв все возможные варианты применения.

### Стартовая конфигурация
В работе [Lab1](../Lab01/README.md) была построена сеть Клоза, разработан IP-план для стеков IPv4 и IPv6, а также выполнены некоторые дополнительные действия, описанные в материале.

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
   mtu 9214
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   ipv6 address 2001:db8:101:202:201::1/64
!
interface Ethernet3
   description --- L3 (no VRF, no VLAN) p2p connection to swLeaf03/Et1
   load-interval 60
   mtu 9214
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
   mtu 9214
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   ipv6 address 2001:db8:101:201:201::2/64
!
interface Ethernet2
   description --- L3 (no VRF, no VLAN): p2p connection to swSpine02/Et1
   load-interval 60
   mtu 9214
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
ip route vrf plnOOB 0.0.0.0/0 10.1.255.254
!
end
```

### Конфигурирование Underlay сети с использованием BGP

#### Вариант eBGP
Конструкт использования eBGP на уровне Underlay выглядит следующим образом:
- Все коммутаторы уровня Spine принадлежат одной автономной системе (в нашем случае - 65000)
- Коммутаторы уровня Leaf принадлежат своим собственным AS, с последовательным ASN: swLeaf01 - 65001, swLeaf02 - 65002, swLeaf03 - 65003 и так далее при необходимости.

![Схема AS](eBGP.png)

Такая архитектура позволяет избежать петель маршрутизации за счет применения механизма подавления петель (AS-Path Loop Prevention) eBGP между коммутаторами уровня Leaf, но потенциально создает проблему связанности между коммутаторами уровня Spine.

В свете примера, озвученного Андреем Блиновым:

![Пример](telegram-cloud-photo-size-2-5219960332686664353-y.jpg)

Давайте рассмотрим детально. Исходя из конфигурации связей, трафик от Host1 к Host4 должен пойти по следующей траектории: Host1 -> Leaf1 -> Spine2 -> Leaf2 -> Spine1 -> Leaf3 -> Host4. Однако, даже коммутатор Leaf1 не будет ничего знать о сетях за коммутатором Leaf2. Причина кроется в том, что коммутаторы Spine отвергнут апдейты друг от друга, так как сработает защита механизма подавления петель. Речь об апдейтах пришедших в Spine со стороны Leaf'ов, содержащих префиксы интерфейсов обратных петель от других Spine'ов и, очевидно, имеющих в последовательности AS-Path свою же ASN.

Для митигации этой проблемы можно воспользоваться двумя механизмами:
1. На стороне коммутаторов уровня Spine, используя команду `allowas-in`, разрешить принимать маршруты со своей AS в пути AS-Path;
2. На Leaf-коммутаторах настроить опцию `as-override`, изменяющей AS-Path для соответствующих префиксов (с моей точки зрения не самый элегантный метод).

Я буду использовать механизм `allowas-in`, реализуя вариант минимализации расходования IPv4 через Link-local адреса (IPv6) p2p ребер.

Начнем с подготовки анонсируемых префиксов. Стандартно, для этого используется директива `network` в контексте соответствующего стека (`address-family`), но более гибким вариантом является получение анонсируемых префиксов через редистрибьюцию по имени интерфейса. Кроме того, мы можем обеспечить чистоту RIB за счет фильтрации списка редистрибьюции - отобрать только используемые выше интерфейсы, используя для этого маршрутную карту:

```config
route-map rmapBGPRedistributeConnected permit 10
   description --- BGP: Redistribute connection list
   match interface Loopback0     ! Указываем список редистрибьюции через имя интерфейса
   set origin igp                ! Меняем origin с incomplete на igp для редистрибьюированных интерфейсов
```

Я буду использовать его только для стека IPv4, но устанавливать соседство через IPv6 Link-local адрес. Теперь можно настроить контекст процесса:

```ssh
swSpine01(config)#router bgp 65000
swSpine01(config-router-bgp)#router-id 10.1.0.1          ! Используем для идентификации Lo0
swSpine01(config-router-bgp)#bgp log-neighbor-changes    ! Включаем журналирование сходимости
swSpine01(config-router-bgp)#address-family ipv4
swSpine01(config-router-bgp-af)#redistribute connected route-map rmapBGPRedistributeConnected
```

> **Важно!**
> Underlay плоскость настраивается в GRT.

В операционной системе Arista EOS BGP-шаблоны реализуются через механизм Peer-Group. Шаблоны (Peer-Groups) позволяют сгруппировать общие настройки (например, номер автономной системы, политики фильтрации маршрутов, параметры таймеров) и применять их сразу к множеству соседей. Шаблон создается в контексте соответствующего экземпляра BGP-процесса:

```ssh
swSpine01(config-router-bgp)#neighbor tmpUnderlay2Leaf peer group                ! Создаем шаблон
```

Я только создал шаблон (он пока пуст), чтобы потом было удобно реконфигуровать все соседства одновременно.

Теперь любые реконфигурации шаблона будут производиться и в настройках соседства, созданных через этот шаблон. Останется только перезапустить процесс для запуска процесса обмена апдейтами с измененными настройками.

Так как я буду использовать IPv6 Link-local адреса для установления соседства, необходимо их определить:

```
swSpine01(config-router-bgp)#sh ipv6 neighbors
IPv6 Address                                  Age Hardware Addr   Interface
fe80::5200:ff:fed5:5dc0                   2:32:08 5000.00d5.5dc0  Et1
fe80::5200:ff:fe03:3766                   2:39:57 5000.0003.3766  Et2
fe80::5200:ff:fe15:f4e8                   2:33:50 5000.0015.f4e8  Et3
```

Теперь мы может описать соседство, указая не только Link-local адрес соседа (на всех портах удаленного коммутатора они будут одинаковые), но и физический порт, через который осуществляется соседство:

```
swSpine01(config-router-bgp)#neighbor fe80::5200:ff:fed5:5dc0%Et1 peer group tmpUnderlay2Leaf
swSpine01(config-router-bgp)#neighbor fe80::5200:ff:fed5:5dc0%Et1 remote-as 65001
swSpine01(config-router-bgp)#address-family ipv4
swSpine01(config-router-bgp-af)#neighbor fe80::5200:ff:fed5:5dc0%Et1 activate
```

где суффикс `Et1` указывает на интерфейс `Ethernet1`.

> В этой конфигурации стоит отметить следующий момент: не следует изменять интерфейс-источник для апдейтов в сторону соседа на интерфейс (например, на интерфейс локальной петли через команду `update-source loopback 0`), так как апдейт будет отброшен из-за строгого механизма валидации TCP-соединения: несоответствия на своей стороне поля `neighbor` и `update-source`, которое мы не сможем устранить без дополнительного указания маршрута к удаленному интерфейсу локальной петли.

Теперь мы можем проверить функционирование соседства:

```
swSpine01#sh bgp summary
BGP summary information for VRF default
Router identifier 10.1.0.1, local AS number 65000
Neighbor                             AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------------------------- ----------- ------------- ----------------------- -------------- ---------- ----------
fe80::5200:ff:fed5:5dc0%Et1       65001 Established   IPv4 Unicast            Negotiated              0          0
```

Здесь мы видим, что соседство установлено (`Established`), но коммутатор не получает ни одного NLRI (Network Layer Reachability Information), несмотря на то, что редистрибьюция настроена.

Проверим, попадает ли она в локальную таблицу BGP:

```
swSpine01#sh ip bgp
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            -                     -       -          -       0       i
```

Как видно, префикс локального интерфейса Lo0 присутствует, он валиден и активен, но у него нет параметра `Next-Hop`, так как в рамках самого коммутатора он и не нужен. И вот что здесь происходит: когда Spine пытается отправить маршрут 10.1.0.1/32 в сторону Leaf, по стандарту BGP он обязан указать в качестве Next Hop свой собственный IP-адрес. Но у Spine на интерфейсе в сторону Leaf есть только Link-local IPv6-адрес. Он не может просто так передать IPv6-адрес в качестве Next Hop для IPv4-сети внутри стандартного обновления, потому что Leaf ожидает там увидеть 32-битный IPv4-адрес. Так как Spine «не знает», какой IPv4 Next Hop поставить для этого анонса, он блокирует отправку маршрута:

```
swSpine01#sh bgp neighbors fe80::5200:ff:fed5:5dc0%Et1 advertised-routes
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Queued for advertisement
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
```

Этот момент решается использованием директивы "next-hop address-family ipv6 originate" в конфигурации соответствующего соседства.

Также чтобы не тянуть IPv6 адреса в настройке соседства, Arista позволяет описывать его через интерфейс соответствующего ребра:

```
swSpine01#sh run sec bgp
route-map rmapBGPRedistributeConnected permit 10
   description --- BGP: Redistribute connection list
   match interface Loopback0
   set origin igp
router bgp 65000
   router-id 10.1.0.1
   neighbor tmpUnderlay2Leaf peer group
   neighbor interface Et1 peer-group tmpUnderlay2Leaf remote-as 65001
   !
   address-family ipv4
      neighbor tmpUnderlay2Leaf activate
      neighbor tmpUnderlay2Leaf next-hop address-family ipv6 originate  ! Разрешить анонсировать IPv6 адрес в качестве параметра next-hop
      redistribute connected route-map rmapBGPRedistributeConnected
```

Теперь проверим анонсы:

```
swSpine01#sh bgp neighbors fe80::5200:ff:fed5:5dc0%Et1 advertised-routes
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Queued for advertisement
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            fe80::5200:ff:fed7:ee0b%Et1 -       -          -       -       65000 i
```

Аналогичные настройки имеет и Leaf-коммутатор:

```
swLeaf01#sh run section bgp
route-map rmapBGPRedistributeConnected permit 10
   description --- BGP: Redistribute connection list
   match interface Loopback0
   set origin igp
router bgp 65001
   router-id 10.1.2.1
   neighbor tmpUnderlay2Spine peer group
   neighbor interface Et1 peer-group tmpUnderlay2Spine remote-as 65000
   !
   address-family ipv4
      neighbor tmpUnderlay2Spine activate
      neighbor tmpUnderlay2Spine next-hop address-family ipv6 originate
      redistribute connected route-map rmapBGPRedistributeConnected
```

Проверим состояние соседства:

```
swSpine01#show bgp summary
BGP summary information for VRF default
Router identifier 10.1.0.1, local AS number 65000
Neighbor                             AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------------------------- ----------- ------------- ----------------------- -------------- ---------- ----------
fe80::5200:ff:fed5:5dc0%Et1       65001 Established   IPv4 Unicast            Negotiated              1          1
```

Перенесем настройки на все коммутаторы фабрики.

Теперь если сравнить вывод RIB на стороне Spine и Leaf, увидим следующее:

```
swSpine01#sh ip route

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

 C        10.1.0.1/32 [0/0]
           via Loopback0, directly connected
 B E      10.1.2.1/32 [200/0]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 B E      10.1.2.2/32 [200/0]
           via fe80::5200:ff:fe03:3766, Ethernet2
 B E      10.1.2.3/32 [200/0]
           via fe80::5200:ff:fe15:f4e8, Ethernet3

swLeaf01#sh ip route

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

 B E      10.1.0.1/32 [200/0]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 B E      10.1.0.2/32 [200/0]
           via fe80::5200:ff:fecb:38c2, Ethernet2
 C        10.1.2.1/32 [0/0]
           via Loopback0, directly connected
 B E      10.1.2.2/32 [200/0]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 B E      10.1.2.3/32 [200/0]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
```

Коммутаторы Spine не знают ничего друг о друге: это результат работы BGP AS-Path Loop Prevention, о котором шла речь ранее. Если посмотреть на списки анонсированных со стороны Leaf:

```
swLeaf01#sh bgp neighbors fe80::5200:ff:fed7:ee0b%Et1 advertised-routes
BGP routing table information for VRF default
Router identifier 10.1.2.1, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Queued for advertisement
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.2/32            fe80::5200:ff:fed5:5dc0%Et1 -       -          -       -       65001 65000 i
 * >      10.1.2.1/32            fe80::5200:ff:fed5:5dc0%Et1 -       -          -       -       65001 i
```

и полученных на стороне Spine маршрутов:

```
swSpine01#sh bgp neighbors fe80::5200:ff:fed5:5dc0%Et1 received-routes
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.2.1/32            fe80::5200:ff:fed5:5dc0%Et1 -       -          -       -       65001 i
```

мы увидим `10.1.0.2/32` в полученных отсутствует, несмотря на то, что со стороны Leaf он анонсирован. Т.е., видно, что Spine его фильтрует, так как AS-Path содержит собственную ASN 65000.

Разрешим принимать апдейты для своей же зоны на обоих Spine'ах, добавив в темплейт команду `neighbor tmpUnderlay2Leaf allowas-in 1`, где последняя `1` указывает на разверешнное количество включений ASN в полученный AS-Path префикса. Для архитектуры Clos в данной конфигурации - это один уровень.

После мягкого перезапуска процессов (`lear bgp * soft`), проверим RIB:

```
swSpine01#sh ip route

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

 C        10.1.0.1/32 [0/0]
           via Loopback0, directly connected
 B E      10.1.0.2/32 [200/0]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 B E      10.1.2.1/32 [200/0]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 B E      10.1.2.2/32 [200/0]
           via fe80::5200:ff:fe03:3766, Ethernet2
 B E      10.1.2.3/32 [200/0]
           via fe80::5200:ff:fe15:f4e8, Ethernet3
```

и связанность с ним:

```
swSpine01#ping 10.1.0.2
PING 10.1.0.2 (10.1.0.2) 72(100) bytes of data.
80 bytes from 10.1.0.2: icmp_seq=1 ttl=64 time=5.20 ms
80 bytes from 10.1.0.2: icmp_seq=2 ttl=64 time=3.52 ms
80 bytes from 10.1.0.2: icmp_seq=3 ttl=64 time=3.55 ms
80 bytes from 10.1.0.2: icmp_seq=4 ttl=64 time=3.63 ms
80 bytes from 10.1.0.2: icmp_seq=5 ttl=64 time=3.25 ms

--- 10.1.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 25ms
rtt min/avg/max/mdev = 3.248/3.830/5.201/0.697 ms, ipg/ewma 6.346/4.486 ms
```

Будем считать основной функционал настроенным. Полная конфигурация Spine-коммутаторов:
```
route-map rmapBGPRedistributeConnected permit 10
   description --- BGP: Redistribute connection list
   match interface Loopback0
   set origin igp
!
router bgp 65000
   router-id 10.1.0.1
   neighbor tmpUnderlay2Leaf peer group
   neighbor tmpUnderlay2Leaf allowas-in 1
   neighbor interface Et1 peer-group tmpUnderlay2Leaf remote-as 65001
   neighbor interface Et2 peer-group tmpUnderlay2Leaf remote-as 65002
   neighbor interface Et3 peer-group tmpUnderlay2Leaf remote-as 65003
   !
   address-family ipv4
      neighbor tmpUnderlay2Leaf activate
      neighbor tmpUnderlay2Leaf next-hop address-family ipv6 originate
      redistribute connected route-map rmapBGPRedistributeConnected
```

и на стороне Spine'ов:

```
route-map rmapBGPRedistributeConnected permit 10
   description --- BGP: Redistribute connection list
   match interface Loopback0
   set origin igp
!
router bgp 65001
   router-id 10.1.2.1
   neighbor tmpUnderlay2Spine peer group
   neighbor interface Et1-2 peer-group tmpUnderlay2Spine remote-as 65000
   !
   address-family ipv4
      neighbor tmpUnderlay2Spine activate
      neighbor tmpUnderlay2Spine next-hop address-family ipv6 originate
      redistribute connected route-map rmapBGPRedistributeConnected
```


#### Вариант iBGP
В случае iBGP, все коммутаторы фабрики принадлежат одной и той же AS.

![Схема AS](iBGP.png)

При использовании внутреннего механизма предотвращения петель, мы сталкиваемся с проблемой фильтрации апдейтов на принимающей стороне с ASN своей же системы. Чтобы обойти эту особенность, нам на уровне Spine необходимо создать "отражатель маршрутов" в сторону коммутаров уровня Leaf.

Как и в предыдущем примере eBGP, будем использовать динамический метод указанания соседства. Работающая конфигурация для коммутаторов уровня Spine приведена ниже:

```
swSpine01#sh run sec bgp
route-map rmapBGPRedistributeConnected permit 10
   description --- BGP: Redistribute connection list
   match interface Loopback0
   set origin igp
router bgp 65000
   router-id 10.1.0.1
   bgp listen range fe80::/10 peer-group tmpUnderlay2Leaf remote-as 65000
   neighbor tmpUnderlay2Leaf peer group
   !
   address-family ipv4
      neighbor tmpUnderlay2Leaf activate
      neighbor tmpUnderlay2Leaf next-hop address-family ipv6 originate
      redistribute connected route-map rmapBGPRedistributeConnected
```

и коммутаторов Leaf:

```
swLeaf01#sh run sec bgp
route-map rmapBGPRedistributeConnected permit 10
   description --- BGP: Redistribute connection list
   match interface Loopback0
   set origin igp
router bgp 65000
   router-id 10.1.2.1
   neighbor tmpUnderlay2Spine peer group
   neighbor tmpUnderlay2Spine next-hop-self
   neighbor interface Et1-2 peer-group tmpUnderlay2Spine remote-as 65000
   !
   address-family ipv4
      neighbor tmpUnderlay2Spine activate
      neighbor tmpUnderlay2Spine next-hop address-family ipv6 originate
      redistribute connected route-map rmapBGPRedistributeConnected
```

Проверим после настройки состояние RIB, получим:

```
swSpine01#sh ip route

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

 C        10.1.0.1/32 [0/0]
           via Loopback0, directly connected
 B I      10.1.2.1/32 [200/0]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 B I      10.1.2.2/32 [200/0]
           via fe80::5200:ff:fe03:3766, Ethernet2
 B I      10.1.2.3/32 [200/0]
           via fe80::5200:ff:fe15:f4e8, Ethernet3
```

видно, что Spine'ы не получают апдейты со Spine'ами,

```
swLeaf01#sh ip route

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

 B I      10.1.0.1/32 [200/0]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 B I      10.1.0.2/32 [200/0]
           via fe80::5200:ff:fecb:38c2, Ethernet2
 C        10.1.2.1/32 [0/0]
           via Loopback0, directly connected
```

а Leaf'ы - от остальных Leaf'ов.

Воспользуемся, сначала, рекоменацией, настройке на стороне коммутаторов Spine отражателей маршрутов:

```
swSpine01(config-router-bgp)#neighbor tmpUnderlay2Leaf route-reflector-client
```

Если теперь посмотреть, анонсы в сторону swLeaf01:

```
swSpine01#sh bgp neighbors fe80::5200:ff:fed5:5dc0%Et1 advertised-routes
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Queued for advertisement
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            fe80::5200:ff:fed7:ee0b%Et1 -       -          100     -       i
```

и увидим только префикс его петлевого интерфейса при том, что его RIB содержит префиксы всех спайнов. То есть отражения не происходит. И происходит это потому, он по стандартам BGP обязан сохранить оригинальный Next-Hop `next-hop` адрес - Link-local IPv6 адреса за интерфейсами самого Spine'а и он сам считает, что со стороны другого оборудования такие адреса будут недоступны.

Особенность команды `next-hop address-family ipv6 originate` в Arista EOS: эта команда заставила Spine генерировать IPv6 Next-Hop для локальных маршрутов (поэтому на его собственном интерфейсе локальной петли 10.1.0.1/32 оно работает). Но для отражаемых iBGP-маршрутов эта команда не применима, так как Route Reflector не имеет права модифицировать Next-Hop клиентов, если не включены специальные политики.

Чтобы решить проблему недостижимости чужих Link-Local адресов, нужно заставить Spine при отражении маршрутов стирать оригинальный Next-Hop лифов и подставлять вместо него себя:

```
swSpine01(config-router-bgp)#address-family ipv4
swSpine01(config-router-bgp-af)#neighbor tmpUnderlay2Leaf next-hop-self
```

В результате чего, анонсы заработают правильно:

```
swSpine01#sh bgp neighbors fe80::5200:ff:fed5:5dc0%Et1 advertised-routes
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Queued for advertisement
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            fe80::5200:ff:fed7:ee0b%Et1 -       -          100     -       i
 * >      10.1.2.2/32            fe80::5200:ff:fed7:ee0b%Et1 -       -          100     -       i Or-ID: 10.1.2.2 C-LST: 10.1.0.1
 * >      10.1.2.3/32            fe80::5200:ff:fed7:ee0b%Et1 -       -          100     -       i Or-ID: 10.1.2.3 C-LST: 10.1.0.1
```

И лифы получат желанное после распространения указанной настройки на все коммутаторы уровня Spine:

```
swLeaf01#sh ip route

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

 B I      10.1.0.1/32 [200/0]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 B I      10.1.0.2/32 [200/0]
           via fe80::5200:ff:fecb:38c2, Ethernet2
 C        10.1.2.1/32 [0/0]
           via Loopback0, directly connected
 B I      10.1.2.2/32 [200/0]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 B I      10.1.2.3/32 [200/0]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
```

Теперь можно проверить работоспособность всей схемы, проведя ping между крайними Leaf'ами:

```
swLeaf01#ping 10.1.2.3
PING 10.1.2.3 (10.1.2.3) 72(100) bytes of data.
80 bytes from 10.1.2.3: icmp_seq=1 ttl=64 time=5.54 ms
80 bytes from 10.1.2.3: icmp_seq=2 ttl=64 time=4.33 ms
80 bytes from 10.1.2.3: icmp_seq=3 ttl=64 time=3.49 ms
80 bytes from 10.1.2.3: icmp_seq=4 ttl=64 time=3.53 ms
80 bytes from 10.1.2.3: icmp_seq=5 ttl=64 time=3.75 ms

--- 10.1.2.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 26ms
rtt min/avg/max/mdev = 3.487/4.128/5.535/0.764 ms, ipg/ewma 6.539/4.796 ms
```

У нас осталась только проблема связанности между коммутаторами уровня Spine.

```
swLeaf01#sh bgp neighbors fe80::5200:ff:fed7:ee0b%Et1 advertised-routes
BGP routing table information for VRF default
Router identifier 10.1.2.1, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Queued for advertisement
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.2.1/32            fe80::5200:ff:fed5:5dc0%Et1 -       -          100     -       i
```

Лифы не настроены как RR, и делать из них RR нельзя. Это полностью сломает логику Spine-Leaf фабрики.

Spine'ы не соединены между собой напрямую, а iBGP Split Horizon запрещает Leaf'у передавать маршрут, полученный от одного спайна, в сторону другого. Из-за этого спайны никогда не узнают друг о друге (и о сетях за ними) через обычный iBGP.

При этом Правило `Split Horizon` гласит: маршрутизатор, получивший апдейт от одного iBGP-соседа, никогда не передаст его другому iBGP-соседу. Протокол BGP блокирует эту отправку на корню, чтобы защитить сеть от петель. Так как Leaf не является рефлектором, никакими командами фильтрации или тюнинга `next-hop` маршрут, принятый внутри iBGP не может быть передан далее (в сторону Spine'а).

Эта проблема, как раз и является аргументом выбора eBGP (рассмотренного ранее), либо использования конфедерации (рассматриваемого далее).

Полная конфигурация коммутаторов уровня Spine выглядит следующим образом:

```
swSpine01#sh run sec bgp
route-map rmapBGPRedistributeConnected permit 10
   description --- BGP: Redistribute connection list
   match interface Loopback0
   set origin igp
router bgp 65000
   router-id 10.1.0.1
   bgp listen range fe80::/10 peer-group tmpUnderlay2Leaf remote-as 65000
   neighbor tmpUnderlay2Leaf peer group
   neighbor tmpUnderlay2Leaf route-reflector-client
   !
   address-family ipv4
      neighbor tmpUnderlay2Leaf activate
      neighbor tmpUnderlay2Leaf next-hop address-family ipv6 originate
      neighbor tmpUnderlay2Leaf next-hop-self
      redistribute connected route-map rmapBGPRedistributeConnected
```

для Leaf:

```
swLeaf01#sh run sec bgp
route-map rmapBGPRedistributeConnected permit 10
   description --- BGP: Redistribute connection list
   match interface Loopback0
   set origin igp
router bgp 65000
   router-id 10.1.2.1
   maximum-paths 64
   neighbor tmpUnderlay2Spine peer group
   neighbor interface Et1-2 peer-group tmpUnderlay2Spine remote-as 65000
   !
   address-family ipv4
      neighbor tmpUnderlay2Spine activate
      neighbor tmpUnderlay2Spine next-hop address-family ipv6 originate
      redistribute connected route-map rmapBGPRedistributeConnected
```

Решить вопрос связанности между Spine'ами можно с помощью костыля - статических маршрутов через все соседние Leaf'ы:

```
ip route 10.1.0.2/32 10.1.2.1
ip route 10.1.0.2/32 10.1.2.2
ip route 10.1.0.2/32 10.1.2.3
```

#### Вариант iBGP с конфедерацией
BGP Confederation (Конфедерация BGP) — это метод масштабирования протокола iBGP внутри одной автономной системы (AS), стандартизированный в RFC 5065. Он позволяет обойти ограничение правила iBGP Split Horizon и полностью отказаться от построения полносвязной топологии (Full Mesh) путем логического деления одной большой глобальной AS (Public AS) на несколько мелких приватных систем — Member-AS (суб-AS). При этом, для внешнего мира вся фабрика останется в единой AS 65000.

Основная цель конфедерации — устранение требования полносвязной топологии iBGP (Full Mesh) и обход ограничения правила iBGP Split Horizon без использования Route Reflectors (RR). 

При использовании конфедерации одна крупная глобальная автономная система (Public AS) дробится на несколько мелких, приватных систем — Member-AS (или суб-AS). Внутри конфедерации выделяют два типа соседства, которые работают по разным правилам:
- Intra-Member-AS iBGP: Классический iBGP внутри одной суб-AS. Здесь по-прежнему действует правило Split Horizon и требуется Full Mesh (или локальные рефлекторы - RR).
- Inter-Member-AS eBGP (Intra-Confederation eBGP): Гибридный тип соседства между разными суб-AS. Протокольно и по механизмам предотвращения петель он работает как eBGP, но при этом сохраняет ключевые свойства iBGP.

*Таблица 1. Различия механизмов BGP*

| Механизм/Атрибут | eBGP | iBGP | Confederation eBGP |
| ------------------ | ------------------ | ------------------ | ------------------ |
| **Предотвращение петель (Loop Prevention)** | На базе AS-Path: Анонс отклоняется, если в списке есть собственный ASN | На базе Split Horizon: маршрут, полученный от iBGP-соседа, не передается другим iBGP-соседям | На базе AS-Path (суб-AS): Анонс отклоняется, если в списке суб-AS есть своя |
| **Логика Next-Hop (по умолчанию)** | Автоматически **меняется** на IP-адрес исходящего интерфейса апдейта | **Не меняется**. Передается оригинальный IP-адрес инициатора (требуется IGP для достижимости) | **Не меняется** (поведение iBGP). Сохраняется оригинальный Next-Hop инициатора для сквозного форвардинга |
| **Кому нужен next-hop-self** | Обычно **не требуется** | **Необходим** на пограничных роутерах для обеспечения достижимости eBGP-маршрутов внутри AS | **Требуется**, если нужно принудительно терминировать Data Plane на стыке суб-AS в BGP Unnumbered фабриках |
| **Передача Local Preference** | **Запрещена**. Атрибут сбрасывается при выходе из AS | **Разрешена**. Свободно передается внутри всей AS | **Разрешена** (поведение iBGP). Атрибут прозрачно проходит сквозь границы суб-AS |
| **Передача MED** | **Разрешена** только между соседними AS (по умолчанию не передается дальше) | **Разрешена**. Транслируется внутри всей AS | **Разрешена** (наследуется от iBGP). Сохраняется и передается между суб-AS без сброса |
| **Значение TTL по-умолчанию** | **TTL = 1**. Соседи должны быть соединены физические | **TTL = 255**. Соседи могут находиться в разных концах сети, связность обеспечивает IGP | **TTL = 255** (наследуется от iBGP) |
| **Ограничение топологии (Full Mesh)** | **Не требуется**. Топология может быть любой | **Требуется** Full Mesh или обход через Route Reflectors | **Не требуется** (наследуется от eBGP) |
| **Отображение в AS-Path** | Номер AS пишется как обычное число в списке | Номер AS не добавляется в путь при передаче внутри системы | Номера суб-AS пишутся внутри сегментов AS_CONFED_SEQUENCE в скобках (например, (65001) |
| **Поведение при выходе из сети** | Остается в неизменном виде в глобальном BGP-апдейте | При выходе наружу в eBGP роутер дописывает номер локальной Public AS | Все скобки и номера суб-AS строго вырезаются, а вместо них подставляется одна внешняя Public AS |

Схема распредеделния ASN повторяет решение для eBGP:

![Схема AS](eBGP.png)

Начнем со следуюшей конфигурации на swSpine01:

```
swSpine01(config-router-bgp-af)#sh run sec bgp
route-map rmapBGPRedistributeConnected permit 10
   description --- BGP: Redistribute connection list
   match interface Loopback0
   set origin igp
router bgp 65000
   router-id 10.1.0.1
   bgp log-neighbor-changes
   neighbor tmpUnderlay2Leaf peer group
   !
   address-family ipv4
      neighbor tmpUnderlay2Leaf activate
      neighbor tmpUnderlay2Leaf next-hop address-family ipv6 originate
      redistribute connected route-map rmapBGPRedistributeConnected
```

Так как в конфидерации параметр nex-hop не меняется (и BGP ведет себя как iBGP), произведем замену на свой адрес при передаче соседу:

```
swSpine01(config-router-bgp)#address-family ipv4
swSpine01(config-router-bgp-af)#neighbor tmpUnderlay2Leaf next-hop-self
swSpine01(config-router-bgp-af)#exit
```

Далее, необходимо описать инфраструктуру конфидерации:

```
swSpine01(config-router-bgp)#bgp confederation identifier 65500      ! Указываем идентификатор AS - это виртуальная AS, которая будет включать все ASN Spine'ов и Leaf'ов
swSpine01(config-router-bgp)#bgp confederation peers 65001 - 65100   ! Указываем валидные sub-AS
```

Зададим соседство динамически: Spine'ы будут отвечать любым Leaf'ам, запрашивающим соединение. В формате Arita, передать в динамический "слушатель" можно либо одну ASN, либо несколько через фильтр. Создадим его:

```
swSpine01(config-router-bgp)#peer-filter pfrBGPLeafs
swSpine01(config-peer-filter-pfrBGPLeafs)#description --- List of valide Leafs
swSpine01(config-peer-filter-pfrBGPLeafs)#match as-range 65001-65100 result accept
```

и передадим в "слушатель":

```
swSpine01(config-peer-filter-pfrBGPLeafs)#router bgp 65000
swSpine01(config-router-bgp)#bgp listen range fe80::/10 peer-group tmpUnderlay2Leaf peer-filter pfrBGPLeafs
```

После этого, можно перейти к настройке первого лифа swLeaf01, начальная конфигурация которого следующая:

```
swLeaf01(config-router-bgp-af)#sh run sec bgp
route-map rmapBGPRedistributeConnected permit 10
   description --- BGP: Redistribute connection list
   match interface Loopback0
   set origin igp
router bgp 65001
   router-id 10.1.2.1
   bgp log-neighbor-changes
   neighbor tmpUnderlay2Spine peer group
   !
   address-family ipv4
      neighbor tmpUnderlay2Spine activate
      neighbor tmpUnderlay2Spine next-hop address-family ipv6 originate
      redistribute connected route-map rmapBGPRedistributeConnected
```

Далее, необходимо произвести настройки конфидерации, указав Public AS (65500) и соседа, имеющего собственный ASN:

```
swLeaf01(config-router-bgp)#bgp confederation identifier 65500
swLeaf01(config-router-bgp)#bgp confederation peers 65000
```

Можно попробовать установить соседство:

```
swLeaf01(config-router-bgp)#neighbor interface Et1 peer-group tmpUnderlay2Spine remote-as 65000
```

Соседство установится, но будет проблема, связанная с блокировкой путей к Spine'ам со стороны транзитных Leaf'ов, когда изолированные Spine'ы находятся в одной суб-AS.

Если посмотреть, что swLeaf01 получает от Spine01 и что анонсирует в ответ:

```
swLeaf01#sh bgp neighbors fe80::5200:ff:fed7:ee0b%Et1 received-routes
BGP routing table information for VRF default
Router identifier 10.1.2.1, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            fe80::5200:ff:fed7:ee0b%Et1 -       -          100     -       (65000) i
 * >      10.1.2.2/32            fe80::5200:ff:fed7:ee0b%Et1 -       -          100     -       (65000 65002) i
 * >      10.1.2.3/32            fe80::5200:ff:fed7:ee0b%Et1 -       -          100     -       (65000 65003) i

swLeaf01#sh bgp neighbors fe80::5200:ff:fed7:ee0b%Et1 advertised-routes
BGP routing table information for VRF default
Router identifier 10.1.2.1, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Queued for advertisement
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.2.1/32            fe80::5200:ff:fed5:5dc0%Et1 -       -          100     -       (65001) i
```

видно, он что он получает от swSpine1 префикс на локальную петлю, но не передает его в сторону второго спайна (swSpine02).

И Leaf ее всё равно не отдаст, потому что ядро Arista EOS выполняет Loop Prevention еще до того, как роутер вообще посмотрит на любой route-map, будь он на входе или на выходе. Программными хаками этот встроенный в код механизм предотвращения петель конфедерации обойти невозможно.

Технически, можно собрать следующий костыль на стороне Spine'ов:
```
swSpine01(config)#ip route 10.1.0.2/32 10.1.2.1
swSpine01(config)#ip route 10.1.0.2/32 10.1.2.2
swSpine01(config)#ip route 10.1.0.2/32 10.1.2.3
```

как и в варианте iBGP.

#### Выводы и выбор варианта
> **Официальная позиция Arista Networks**:
>
> Arista категорически не рекомендует использовать BGP Конфедерации (RFC 5065) для построения современных Underlay-сетей в Spine-Leaf фабриках.
>
> Если в iBGP-дизайне два Спайна изолированы друг от друга (нет Inter-Spine линка), стандартный механизм конфедераций создает мертвую петлю фильтрации на Лифах. Arista рекомендует решать это единственным архитектурным способом — полным переходом на eBGP Underlay (дизайн RFC 7938).

Со своей стороны хочу еще раз подчеркнуть, что при всех возможных выгодах, использование BGP в Underlay-слое лично с моей точки зрения является спорным.

Далее буду использовать версию eBGP.

### Тюниннг и дополнительные настройки
Чтобы выжать максимум из eBGP-фабрики в топологии Clos, Arista Networks рекомендует внедрить дополнительный тюнинг. Эти настройки делятся на категории: ускорение сходимости и безопасность.

Настройки необходимо производить на всей фабрике, но я буду использовать коммутатор swSpine01 для указания контекста команды.

#### Оптимизация сходимости (Convergence & Timers)
Поскольку в сетях Clos есть вполне определенные требования обработке трафика, eBGP должен реагировать на падение линков мгновенно. Стандартно, BGP использует следующие таймеры: keepalive 60, holdtime 180, заданные в секундах. Arista рекомендует агрессивные таймеры: keepalive 3, holdtime 9.

```
swSpine01(config-router-bgp)#timers bgp 3 9
```

Если сессия BGP падает не из-за физического обрыва кабеля, а из-за ошибок конфигурации (например, превышен лимит префиксов по команде maximum-routes), Arista переводит соседа в состояние Idle. Чтобы коммутатор автоматически пытался поднять упавшего по ошибке соседа каждые 60 секунд (вместо дефолтного ожидания ручного сброса), используют команду (может быть задана и в контексте определенного соседа):
```
swSpine01(config-router-bgp)#neighbor tmpUnderlay2Leaf idle-restart-timer 60
```

#### Тюнинг безопасности и защиты CPU (Security)
 Поскольку Spine и Leaf соединены напрямую, TTL eBGP пакета должен быть строго равен 255 (если пакет летит от хакера через три хопа, его TTL уменьшится). Коммутатор будет аппаратно дропать любые BGP-пакеты с TTL меньше заданного, защищая процессор (Supervisor). Фактически, это защита от защита от спуфинга BGP-пакетов согласно [RFC 5082]: BGP TTL Security (GTSM — Generalized TTL Security Mechanism).

```
swSpine01(config-router-bgp)#neighbor tmpUnderlay2Leaf ttl maximum-hops 1
```

Защита от переполнения таблицы маршрутов:

```
swSpine01(config-router-bgp)#neighbor tmpUnderlay2Leaf maximum-routes 1000 warning-only
```

### Поддержка BFD
BGP, как и любой современный протокол, поддерживает BFD. Включение BFD на всех eBGP-стыках Spine-Leaf является обязательным. Это снижает время фиксации аварии до миллисекунд.

```
swLeaf01(config-router-bgp)#neighbor tmpUnderlay2Spine bfd
```

После включения можно проверить состояние и таймеры на работающих соседях следующим образом:

```
swLeaf01(config-router-bgp)#show bfd peers detail
VRF name: default
-----------------
Peer Addr fe80::5200:ff:fecb:38c2, Intf Ethernet2, Type normal, Role active, State Down
...
TxInt: 1000 ms, RxInt: 300 ms, Multiplier: 3
...
```

Здесь мы видим настройки по-умолчанию:
- RxInt: 300 ms - проверка соседства производится 300 мс;
- Multiplier: 3 - признавать соседа недоступным чере 3 потерянных пакета;
- TxInt: 1000 ms - поскольку сессия не согласована, коммутатор автоматически сбросил скорость отправки служебных пакетов до 1000 мс, чтобы не спамить в пустой линк.

Для сетей Clos, может быть применены следующие тайминги:

```
neighbor tmpUnderlay2Leaf bfd interval 100 min-rx 50 multiplier 3
```

где:
- interval 100: отправка Hello-пакетов каждые 100 мс;
- min-rx 100: готовность принимать пакеты каждые 100 мс;
- multiplier 3 — падение линка фиксируется при потере 3 пакетов подряд.

Итоговое время обнаружения аварии (Detection Time): 100 мс × 3 = 300 миллисекунд.

### Поддержка ECMP и UCMP
Clos-топология строится ради горизонтального масштабирования полосы пропускания. В операционной системе Arista EOS механизмы балансировки трафика ECMP (Equal-Cost Multi-Path) и UCMP (Unequal-Cost Multi-Path) являются фундаментом для построения сетей Clos (Spine-Leaf) [RFC 7938]. Они позволяют утилизировать пропускную способность всех доступных аплинков.

#### ECMP — балансировка по равной стоимости
По умолчанию BGP выбирает только один лучший путь к префиксу. ECMP позволяет устанавливать в таблицу маршрутизации (RIB) и аппаратную таблицу коммутации (FIB) несколько одинаковых маршрутов до одной сети.

Включение ECMP производится директивой `maximum-paths` с указанием максимального числа параллельных путей:

```
swLeaf03(config-router-bgp)#maximum-paths 64
```

BGP Multipath Relax (As-Path Multipath Relax): Поскольку Лиф получает маршруты до других Лифов от разных Спайнов, AS-Path у них будет одинаковой длины, но контент путей может отличаться. Чтобы роутер балансировал трафик по обоим Спайнам, включается команда bgp bestpath as-path multipath-relax.

В eBGP-фабриках Leaf'ы получает маршруты до удаленных Leaf'ов от всех SPine'ов фабрики. Длина AS-Path у них одинаковая, но сами пути отличаются (например, у одного 65000 65002, у другого 65000 65003). По стандарту BGP такие маршруты не считаются равными. Arista лечит это одной критически важной командой, которая заставляет BGP сравнивать только длину пути, игнорируя конкретные ASN:

```
swLeaf01(config-router-bgp-af)#bgp bestpath as-path multipath-relax
```

Эту команду распространяют на все Leaf'ы фабрики.

После распространения указанных настроек, таблица BGP будет иметь следующий вид:

```
swSpine01(config-unit-bgp)#sh ip bgp
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            -                     -       -          -       0       i
 * >Ec    10.1.0.2/32            fe80::5200:ff:fe03:3766%Et2 0       -          100     0       65002 65000 i
 *  ec    10.1.0.2/32            fe80::5200:ff:fe15:f4e8%Et3 0       -          100     0       65003 65000 i
 *  ec    10.1.0.2/32            fe80::5200:ff:fed5:5dc0%Et1 0       -          100     0       65001 65000 i
 * >      10.1.2.1/32            fe80::5200:ff:fed5:5dc0%Et1 0       -          100     0       65001 i
 *  E     10.1.2.1/32            fe80::5200:ff:fe03:3766%Et2 0       -          100     0       65002 65000 65001 i
 *  e     10.1.2.1/32            fe80::5200:ff:fe15:f4e8%Et3 0       -          100     0       65003 65000 65001 i
 * >      10.1.2.2/32            fe80::5200:ff:fe03:3766%Et2 0       -          100     0       65002 i
 *        10.1.2.2/32            fe80::5200:ff:fed5:5dc0%Et1 0       -          100     0       65001 65000 65002 i
 * >      10.1.2.3/32            fe80::5200:ff:fe15:f4e8%Et3 0       -          100     0       65003 i
 *        10.1.2.3/32            fe80::5200:ff:fed5:5dc0%Et1 0       -          100     0       65001 65000 65003 i
```

а таблица маршрутизации:

```
swSpine01(config-unit-bgp)#sh ip route

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

 C        10.1.0.1/32 [0/0]
           via Loopback0, directly connected
 B E      10.1.0.2/32 [200/0]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
           via fe80::5200:ff:fe03:3766, Ethernet2
           via fe80::5200:ff:fe15:f4e8, Ethernet3
 B E      10.1.2.1/32 [200/0]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 B E      10.1.2.2/32 [200/0]
           via fe80::5200:ff:fe03:3766, Ethernet2
 B E      10.1.2.3/32 [200/0]
           via fe80::5200:ff:fe15:f4e8, Ethernet3
```

#### UCMP — Балансировка по несовпадающей стоимости
Классический ECMP делит трафик строго поровну (50/50). Eсли коммутатор swSpine01 подключен портом 40G, а swSpine02 — через линк 10G. ECMP пустит туда одинаковый объем трафика, что мгновенно перегрузит и «уронит» 10G-линк, в то время как 40G-линк будет простаивать.

UCMP решает эту проблему, распределяя пакеты пропорционально пропускной способности интерфейсов (в данном примере 4:1). В BGP для этого используется механизм BGP Link Bandwidth.

Как работает UCMP в Arista под капотом: коммутаторы eBGP-соседи обмениваются специальным неканоническим атрибутом (Extended Community) — Link Bandwidth. В этом атрибуте роутер передает реальную скорость своего физического интерфейса (например, 10 Гбит/с или 40 Гбит/с). Принимающий Лиф считывает эти веса и на аппаратном уровне (в ASIC) нарезает хэш-таблицу (ECMP buckets) пропорционально этим значениям.

Чтобы запустить этот механизм, необходимо использовать следующую команду:

```
swSpine01(config-router-bgp)#neighbor tmpUnderlay2Leaf link-bandwidth <mode>
```

где параметр `mode` может принимать следующие значения:
- **auto** - заставляет коммутатор автоматически брать значение пропускной способности (bandwidth) с физического интерфейса, на котором висит сосед, и упаковывать его в BGP-апдейте;
- **adjust auto percent <pr>** - позволяет уменьшить реальное значение Link Bandwidth на указанное значение, полученному от конкретного пира. Используется для сложной инженерии трафика (Traffic Engineering). Например, если необходимо искусственно снизить приоритет линков через определенного соседа, не выключая его полностью.
- **default <bw>** - принудительно прописывает фиксированное значение пропускной способности (в байтах/бит в секунду) для всех маршрутов, полученных или отправляемых этой группе, например, `neighbor tmpUnderlay2Leaf link-bandwidth default 20000000000`;
- **update-delay <delay>** - задает таймер ожидания (в секундах) перед тем, как коммутатор отправит соседям обновленное значение Link Bandwidth, если на физическом интерфейсе изменилась скорость.

### Средства безопасности
В протоколе BGP есть встроенные средства безопасности обеспечивающие аутентификацию соседей.

Что касается шифрования — сам по себе протокол BGP (пакеты Update, Open и т.д.) исторически передает данные в открытом (Clear Text) виде [RFC 4271].

Авторизация на оборудовании Arista может осуществляться с использованием задаваемого пароля в контексте соседства (или темплейта), либо с использованием профиля безопасности.

Конфигурирование паролем осуществляется следующим образом:
```
swSpine01(config-router-bgp)#neighbor tmpUnderlay2Leaf password <PASSWORD>
```

Другой вариант - использование профиля безопасности. Вместо того чтобы прописывать пароли вручную под каждым BGP-соседом, создается глобальный централизованный профиль безопасности в системе. А в настройках BGP просто производится ссылка на имя этого профиля.

```
swSpine01(config-router-bgp)#management security
swSpine01(config-mgmt-security)#session shared-secret profile secBGPProfile
swSpine01(config-mgmt-sec-sh-sec-profile-secBGPProfile)#secret kidBGP <PASSWORD> <date>
```

где параметр `date` определяет сроки использования отдельного пароля и может быть:
- **infinite** - Ключ активируется сразу и работает бессрочно. Сессия BGP зашифрована и не упадет по таймеру.
- **receive-lifetime <date> / transmit-lifetime <date>** - Определяет период времени, когда коммутатор использует пароль для подписи своих исходящих пакетов и когда он готов принимать этот пароль от соседа. Могут в качестве аргументов получать и `infinite` для перманентного развешения апдейтов в соответствующем направлении.
- **yyyy-mm-dd / mm/dd/yyyy** - определяют формат даты.

После описания профиля, он привязывается в контексте соседства (или шаблона) экземпляра процесса BGP:

```
swSpine01(config-router-bgp)#neighbor tmpUnderlay2Leaf password shared-secret profile secBGPProfile algorithm hmac-sha-256
```

> Алгоритм хэширование может быть следующим: AES-128-CMAC-96, HMAC-SHA-256-128 и HMAC-SHA-1-96. Arista рекомендует использование `hmac-sha-256` как наиболее надежного.

### Graceful shutdown
Механизм BGP Graceful Shutdown (определенный в RFC 8326) в Arista EOS — это инструмент для планового, безаварийного вывода маршрутизатора из эксплуатации на время техобслуживания.

В отличие от жесткого выключения сессии (которое вызывает потерю пакетов, пока фабрика Clos пересчитывает ECMP-хэши), Graceful Shutdown заблаговременно и плавно уводит трафик на альтернативные пути, сводя потери к нулю (Zero Packet Loss).

```
swSpine01(config)# maintenance
swSpine01(config-maintenance)# unit bgp
swSpine01(config-maintenance-unit-bgp)# quiesce
```

Ключевое слово `quiesce` запускает плавный увод трафика (BGP Graceful Shutdown). Трафик иссякнет за секунды. Чтобы вернуть Спайн в работу после обслуживания, необходимо использовать команду `no quiesce`
