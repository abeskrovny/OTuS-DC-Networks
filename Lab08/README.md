# Lab08: VxLAN. Оптимизация таблиц маршрутизации

## Состав работы
- [Условие задачи](#условие-задачи)

### Условие задачи
В этой самостоятельной работе мы ожидаем, что вы самостоятельно:
- Разместите двух "клиентов" в разных VRF в рамках одной фабрики.
- Настроите маршрутизацию между клиентами через внешнее устройство (граничный роутер\фаерволл\etc)
- Зафиксируете в документации - план работы, адресное пространство, схему сети, настройки сетевого оборудования

### Описание выбранного решения
С точки зрения схемы сети, я оставлю решение из [Lab06]() и [Lab07](), но произведу следующие действия:
1. Оставлю обе сервисные модели (VLAN-Based и VLAN-Aware Based);
2. Полностью перейду на симметричную модель моршрутизации (Symmetric IRB);
3. Удалю OSPF, оставив на уровне оверлея только один BGP;
4. Переделаю подключение L3 сервера и размещу за ним "внутренний" сегмент.

Схема коммутации сети остается следующая:

![Схема сети](scheme2.png)

Кроме того, я хочу гармонизировать все идентификаторы, их названия и произвести весь необходимый для реальной задачи тюнинг.

### Настройка фабрики

#### Настройка базового функционала Underlay/Overlay
На уровне подстилающей сети я буду использовать приватную сеть А-класса 10.0.0.0/8, генерация адреса в которой будет производиться согласно правила `10.<Pod>.<Type>.<Number>`, где:
- **Pod (Point of Delivery)**: "точка предоставления услуг" - модульный блок сетевой и вычислительной инфраструктуры, который имеет четко определенные границы, предсказуемую производительность и масштабируется как единое целое.
- **Type**: тип оборудования:
   - *0-100*: физическое;
      - 1: коммутатор уровня Leaf;
      - 2: коммутатор уровня Spine;
      - 3: коммутатор уровня Super-Spine;

Согласно схеме, настройки в табличном виде предствалены в таблице:

*Таблица 1: Адресация подстилающей (андерлейной) сети*

| **Hostname** | **Type** | **IFace** | **IPv4** | **NSAP** |
|-----------------|--------------|-----------|---------------|---------------------------|
| swSpine01 | 02 (Spine) | Loopback0 | 10.1.2.1/32 | 49.0001.0100.0100.0001.00 |
| swSpine02 | 02 (Spine) | Loopback0 | 10.1.2.2/32 | 49.0001.0100.0100.0002.00 |
| swSpine03 | 02 (Spine) | Loopback0 | 10.1.2.3/32 | 49.0001.0100.0100.0003.00 |
| swLeaf01 | 01 (Std Leaf) | Loopback0 | 10.1.1.1/32 | 49.0001.0100.0100.1001.00 |
| swLeaf02 | 01 (Std Leaf) | Loopback0 | 10.1.1.2/32 | 49.0001.0100.0100.1002.00 |
| swLeaf03 | 01 (Std Leaf) | Loopback0 | 10.1.1.3/32 | 49.0001.0100.0100.1003.00 |
| swLeaf04 | 01 (Std Leaf) | Loopback0 | 10.1.1.4/32 | 49.0001.0100.0100.1004.00 |
| swBorderLeaf01 | 01 (Std Leaf) | Loopback0 | 10.1.1.251/32 | 49.0001.0100.0100.1251.00 |

Начальные конфигурации коммутаторов уровня Spine (на примере `swSpine01`):
```
swSpine01#sh run
! Command: show running-config
! device: swSpine01 (vEOS-lab, EOS-4.33.1.1F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
no service interface inactive port-id allocation disabled
!
transceiver qsfp default-mode 4x10G
!
interface defaults                     !! Настраиваем сосотояние физических интерфейсов по-умолчанию
   !! Security Off, Jumbo-frame on
   mtu 9000                          
   !
   ethernet
      shutdown                         !! Отключаем по требованиям безопасности
!
service routing protocols model multi-agent     !! Обязательное переключение на многоагентную архитектуру
!
hostname swSpine01
dns domain Underlay.local
!
spanning-tree mode mstp                !! Оставляем, чтобы не нарушить консистентность L2
!
system l1
   unsupported speed action error
   unsupported error-correction action error
!
interface Ethernet1
   description --- L3 p2p: (no VLAN, no VRF): connection to swLeaf01:Ethernet1
   no shutdown
   load-interval 60
   mtu 9000
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   isis enable Underlay
   isis network point-to-point
!
interface Ethernet2
   description --- L3 p2p: (no VLAN, no VRF): connection to swLeaf02:Ethernet1
   no shutdown
   load-interval 60
   mtu 9000
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   isis enable Underlay
   isis network point-to-point
!
interface Ethernet3
   description --- L3 p2p: (no VLAN, no VRF): connection to swLeaf03:Ethernet1
   no shutdown
   load-interval 60
   mtu 9000
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   isis enable Underlay
   isis network point-to-point
!
interface Ethernet4
   description --- L3 p2p: (no VLAN, no VRF): connection to swLeaf04:Ethernet1
   no shutdown
   load-interval 60
   mtu 9000
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   isis enable Underlay
   isis network point-to-point
!
interface Ethernet5
   description --- L3 p2p: (no VLAN, no VRF): connection to swBorderLeaf01:Ethernet1
   no shutdown
   load-interval 60
   mtu 9000
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   isis enable Underlay
   isis network point-to-point
!
interface Ethernet6
   shutdown
!
interface Ethernet7
   shutdown
!
interface Ethernet8
   shutdown
!
interface Loopback0
   description --- Loopback (no VLAN, no VRF): interface for Underlay Control-Plane
   load-interval 60
   ip address 10.1.2.1/32
   isis enable Underlay
   isis passive
!
interface Management1
!
ip routing
!
ipv6 unicast-routing
!
router bgp 65001
   !! Main Layer of Overlay Control-Plane
   router-id 10.1.2.1
   update wait-for-convergence            !! Принудительно удерживать отправку BGP-апдейтов (Updates) соседям до тех пор, пока сеть полностью не сойдется (не спамить сеть промежуточными данными)
   update wait-install                    !! Производить ожидание записи маршрутов в аппаратный FIB (предотвращает блэкхол на время записи)
   no bgp default ipv4-unicast            !! Отключает семейство IPv4 для экономии ресурсов устройства
   timers bgp 3 9                         !! Параметр Keepalive = 3 секунды и Holdtime = 9 секунд
   distance bgp 20 200 200                !! Указываем административные дистанции: eBGP, iBGP, local BGP
   graceful-restart restart-time 300      !! Указывает максимальное время рестарта BGP процесса
   graceful-restart                       !! Требует от соседей не удалять маршруты через локальный коммутатор на время перезагрузки процесса
   maximum-paths 16 ecmp 16
   neighbor grpLEAFS peer group
   neighbor grpLEAFS remote-as 65001
   neighbor grpLEAFS update-source Loopback0
   neighbor grpLEAFS bfd
   neighbor grpLEAFS bfd interval 100 min-rx 100 multiplier 3
   neighbor grpLEAFS route-reflector-client
   neighbor grpLEAFS send-community extended
   neighbor 10.1.1.1 peer group grpLEAFS
   neighbor 10.1.1.2 peer group grpLEAFS
   neighbor 10.1.1.3 peer group grpLEAFS
   neighbor 10.1.1.4 peer group grpLEAFS
   neighbor 10.1.1.251 peer group grpLEAFS
   !
   address-family evpn
      neighbor grpLEAFS activate
      neighbor grpLEAFS next-hop-unchanged
!
router isis Underlay
   hello padding disabled
   net 49.0001.0100.0100.0001.00
   router-id ipv4 10.1.0.1
   is-type level-2
   log-adjacency-changes
   set-overload-bit on-startup 300
   !
   address-family ipv4 unicast
      maximum-paths 16
      bfd all-interfaces
!
router multicast
   ipv4
      software-forwarding kernel
   !
   ipv6
      software-forwarding kernel
!
end
```

Начальные конфигурации коммутаторов уровня Leaf (на примере `swLeaf01`):
```
swLeaf01#sh run
! Command: show running-config
! device: swLeaf01 (vEOS-lab, EOS-4.33.1.1F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
no service interface inactive port-id allocation disabled
!
transceiver qsfp default-mode 4x10G
!
interface defaults
   !! Security Off, Jumbo-frame on
   mtu 9000
   !
   ethernet
      shutdown
!
service routing protocols model multi-agent
!
hostname swLeaf01
dns domain Underlay.local
!
spanning-tree mode mstp
!
system l1
   unsupported speed action error
   unsupported error-correction action error
!
interface Ethernet1
   description --- L3 p2p: (no VLAN, no VRF): connection to swSpine01:Ethernet1
   load-interval 60
   mtu 9000
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   isis enable Underlay
   isis network point-to-point
!
interface Ethernet2
   description --- L3 p2p: (no VLAN, no VRF): connection to swSpine02:Ethernet1
   load-interval 60
   mtu 9000
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   isis enable Underlay
   isis network point-to-point
!
interface Ethernet3
   description --- L3 p2p: (no VLAN, no VRF): connection to swSpine03:Ethernet1
   load-interval 60
   mtu 9000
   no switchport
   ip address unnumbered Loopback0
   ipv6 enable
   isis enable Underlay
   isis network point-to-point
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
   description --- Loopback (no VLAN, no VRF): interface for Underlay Control-Plane
   ip address 10.1.1.1/32
   isis enable Underlay
   isis passive
!
interface Management1
!
ip routing
!
ipv6 unicast-routing
!
router bgp 65001
   !! Main Layer of Overlay Control-Plane
   router-id 10.1.1.1
   update wait-for-convergence
   update wait-install
   no bgp default ipv4-unicast
   timers bgp 3 9
   distance bgp 20 200 200
   graceful-restart restart-time 300
   graceful-restart
   maximum-paths 16 ecmp 16
   neighbor grpLEAFS peer group
   neighbor grpSPINES peer group
   neighbor grpSPINES remote-as 65001
   neighbor grpSPINES update-source Loopback0
   neighbor grpSPINES bfd
   neighbor grpSPINES bfd interval 100 min-rx 100 multiplier 3
   neighbor grpSPINES route-reflector-client
   neighbor grpSPINES send-community extended
   neighbor 10.1.2.1 peer group grpSPINES
   neighbor 10.1.2.2 peer group grpSPINES
   neighbor 10.1.2.3 peer group grpSPINES
   !
   address-family evpn
      neighbor grpSPINES activate
!
router isis Underlay
   hello padding disabled
   net 49.0001.0100.0100.1001.00
   router-id ipv4 10.1.1.1
   is-type level-2
   log-adjacency-changes
   set-overload-bit on-startup 300
   !
   address-family ipv4 unicast
      maximum-paths 16
      bfd all-interfaces
!
router multicast
   ipv4
      software-forwarding kernel
   !
   ipv6
      software-forwarding kernel
```

> Если серверы подключены только к коммутаторая уровня Leaf, то блок `address-family ipv4` в конфигурации уровня Spine рекомендуется полностью удалить. Это разгрузит процессор Spine'а и защитит его от ненужных маршрутов.

Конфигурации всех хостов сброшены.

На уровне наложенной сети (оверлейной) будет использоваться iBGP c ASN:65001 (последняя декада указывает на номер POD'а).

С точки зрения поддержки арендаторов, необходимо подготовить отдельные VRF для изоляции их Data-Plane между друг-другом и Control-Plane самой фабрики.

*Таблица 2: Накладные (оверлейные) сети L3*

| **Арендатор** | **VRF Name** | **ASN** | **Target** | **L3VNI** | **L3VLAN** |
|-|-|-|-|-|-|
| Tenant A | TENANT-A | 65101 | 101 | 1101000 | 4001 |
| Tenant B | TENANT-B | 65102 | 102 | 1102000 | 4002 |

Донстройка потребуется только на стороне коммутаторов уровня Leaf.
```
vlan 4001
   !! VLAN for L3VPN Symmetric IRB
   name L3VPN:TENANT-A
!
vlan 4002
   !! VLAN for L3VPN Symmetric IRB
   name L3VPN:TENANT-B
!
vrf instance TENANT-A
   description --- VRF: RIB for Tenant-A
!
vrf instance TENANT-B
   description --- VRF: RIB for Tenant-B
!
interface Vlan4001
   description --- Virtual (VLAN4001, VRF: TENANT-A): interface for L3 VPN tunneling via Symmetric IRB
   no autostate
   vrf TENANT-A
!
interface Vlan4002
   description --- Virtual (VLAN4002, VRF: TENANT-B): interface for L3 VPN tunneling via Symmetric IRB
   no autostate
   vrf TENANT-B
!
interface Vxlan1
   description --- VxLAN (no VRF): interface for Overlay Control-Plane
   load-interval 60
   vxlan source-interface Loopback0                   !! Справедливо для всех Leaf кроме организующих MLAG-пару (swLeaf01 и swLeaf02)
   vxlan udp-port 4789
   vxlan vrf TENANT-A vni 1101000
   vxlan vrf TENANT-B vni 1102000
   vxlan learn-restrict any                           !! Блокировка Data Plane Learning (противодействие атаке MAC-address Hijacking / VTEP Spoofing, использующей динамическое обучение)
!
ip virtual-router mac-address 00:1c:73:00:00:01       !! Задает Anycast MAC (уникальный для всей фабрики)
!
ip routing vrf TENANT-A
ip routing vrf TENANT-B
!
router bgp 65001
   vrf TENANT-A
      !! VRF for Tenant A
      rd 10.1.1.251:101
      route-target import evpn 65101:101
      route-target export evpn 65101:101
      !
      address-family ipv4
         redistribute connected
   !
   vrf TENANT-B
      !! VRF for Tenant B
      rd 10.1.1.251:102
      route-target import evpn 65102:102
      route-target export evpn 65102:102
      !
      address-family ipv4
         redistribute connected
```

Указанные настройки необходимо произвести на всех Leaf'ах фабрики, на которых присутствует хотя бы один VLAN из арендаторских. Фактически, L3VNI создают инфраструктуру для транспорта изолированного трафика каждого их VRF по фабрике. Количество VLAN, присутствующих на конкретном Leaf'е (подключенных через абонентское подключение с сервоеров) не имеет значения, так как маршрутизация внутри VRF будет производиться с помощью технологии Anycast Gateway.

Пользовательские VLAN могут быть произвольными, поэтому нам надо будет либо производить их мутацию на выходе из VTEP, либо использовать пользовательские где это возможно с перенесением логики нумерации на VNI.

*Таблица 3: Накладные (оверлейные) сети L2*

| **Арендатор** | VLAN ID | IP/MASK | VNI |
|-|-|-|-|
| Tenant A | 137 | 192.168.12.0/24 | 1101137 |
| Tenant A | 1026 | 10.128.14.0/24 | 1101026 |
| Tenant A | 6 | 172.12.23.0/24 | 1101006 |
| Tenant B | 23 | 192.168.23.0/24 | 1102023 |
| Tenant B | 889 | 10.1.1.0/24 | 1102889 |

Настройки, связанные с L2-сервисами имеют следующий вид:
```
vlan 6
   name TENANT-A:VLAN006
!
vlan 23
   name TENANT-B:VLAN023
!
vlan 137
   name TENANT-A:VLAN137
!
vlan 889
   name TENANT-B:VLAN889
!
vlan 1026
   name TENANT-A:VLAN026
!
interface Loopback101
   description --- Loopback (no VLAN, VRF: TENANT-A): interface for Control-Plane for Tenant A
   load-interval 60
   ip address 10.1.0.101/32
!
interface Loopback102
   description --- Loopback (no VLAN, VRF: TENANT-B): interface for Control-Plane for Tenant B
   load-interval 60
   ip address 10.1.0.102/32
!
interface Vlan6
   description --- Virtual (VLAN006, VRF: TENANT-A): interface for L3 termination
   no autostate
   vrf TENANT-A
   ip address unnumbered Loopback101
   ip virtual-router address 172.12.23.1
!
interface Vlan23
   description --- Virtual (VLAN023, VRF: TENANT-B): interface for L3 termination
   no autostate
   vrf TENANT-B
   ip address unnumbered Loopback102
   ip virtual-router address 192.168.23.1
!
interface Vlan137
   description --- Virtual (VLAN137, VRF: TENANT-A): interface for L3 termination
   no autostate
   vrf TENANT-A
   ip address unnumbered Loopback101
   ip virtual-router address 192.168.12.1
!
interface Vlan889
   description --- Virtual (VLAN889, VRF: TENANT-B): interface for L3 termination
   no autostate
   vrf TENANT-B
   ip address unnumbered Loopback102
   ip virtual-router address 10.1.1.1
!
interface Vlan1026
   description --- Virtual (VLAN1026, VRF: TENANT-A): interface for L3 termination
   no autostate
   vrf TENANT-A
   ip address unnumbered Loopback101
   ip virtual-router address 10.128.14.1
!
interface Vxlan1
   vxlan vlan 6 vni 1101006
   vxlan vlan 23 vni 1102023
   vxlan vlan 137 vni 1101137
   vxlan vlan 889 vni 1102889
   vxlan vlan 1026 vni 1101026
!
router bgp 65001
   vlan-aware-bundle vabTENANT-A
      rd 10.1.1.251:101
      route-target both 65101:101
      redistribute learned
      vlan 6,137,1026
   !
   vlan-aware-bundle vabTENANT-B
      rd 10.1.1.251:102
      route-target both 65102:102
      redistribute learned
      vlan 23,889
```

 > Если на отдельном коммутаторе уровня Leaf отсутствует тот или иной пользовательский VLAN, его можно не описывать на соответствующем оборудовании.

#### Переход с Ingress Replication на Multicast
Хотя Ingress Replication невероятно прост в настройке (не требует PIM в Underlay), у него есть критические архитектурные недостатки, которые делают его неприменимым в больших фабриках:
• **Огромная нагрузка на аплинки Ingress-лифа**: Когда хост за Leaf'ом генерирует BUM-пакет (например, ARP-запрос), этот Leaf обязан физически скопировать данный пакет (реплицировать) столько раз, сколько удаленных VTEP-соседей находится в этой сети. Если в фабрике 50 коммутаторов уровня Leaf, коммутатор отправит 49 одинаковых копий пакета в свои аплинки, утилизируя полосу пропускания оверлейными дублями.
• **Линейный рост задержки (Serialization Delay)**: Аппаратный чипсет (ASIC) коммутатора не может вытолкнуть 50 пакетов в кабель одновременно — он отправляет их последовательно, один за другим. В итоге 50-й коммутатор получит свой ARP-запрос значительно позже, чем 1-й, что увеличивает RTT (Round-Trip Time - время приема-передачи) для базовых сетевых процедур.
• **Ограничения аппаратных таблиц (Flood List Limits)**: У любого чипа есть жесткий лимит на размер так называемого Flood List (списка репликации). При достижении определенного количества VTEP-соседей коммутатор просто аппаратно не сможет обслуживать такую конфигурацию.
• **Плохая утилизация ресурсов Spines**: Вместо того чтобы коммутатор уровня Spine один раз принял пакет и сам размножил его по сети, он используется как глупый транзит для десятков одинаковых копий пакетов, которые Leaf наплодил самостоятельно.

Для перевода обработки **BUM-трафика** (Broadcast, Unknown Unicast, Multicast) с Head-End Replication (HER / Ingress Replication) на **Multicast в Underlay-сети** на коммутаторах Arista EOS, необходимо настроить протокол **PIM ASN / PIM SM** в транспортной сети и изменить способ флудинга на интерфейсе `Vxlan1`.

**PIM SM — Protocol Independent Multicast - Sparse Mode** (Независимый от протоколов мультикаст — разреженный режим). Это базовый протокол и зонтичный стандарт (описан в RFC 7761). Сам по себе PIM-SM определяет принципы построения деревьев распределения трафика (MDT) на основе reverse path forwarding (RPF). Внутри PIM-SM существуют две разные архитектурные модели обслуживания: ASM и SSM:
• **PIM ASM — Any Source Multicast** (Мультикаст от любого источника). В этой модели получателю (Receiver) абсолютно всё равно, кто отправляет данные. При этом, он не знает, откуда пойдет поток, поэтому в сети нужен посредник — Rendezvous Point (RP).
• **PIM SSM — Source-Specific Multicast** (Мультикаст от конкретного источника). В этой модели получатель знает не только группу, но и точный IP-адрес источника, который вещает. RP не требуется.

Главное отличие между ними заключается в том, знает ли получатель трафика заранее, кто этот трафик транслирует, и как сеть организует встречу источника и получателя. Использование SSM в Underlay возможно, но требует специфической поддержки со стороны оверлейного Control-Plane. Ввиду этого я буду использовать более простой в конфигурировании вариант PIM ASM.

При использовании PIM-ASM коммутаторы строят так называемое **Shared Tree (Общее дерево)**, обозначаемое в таблицах маршрутизации как **`(*, G)`**, где `*` — это любой источник, а `G` — мультикаст-группа. Точкой сборки (корнем дерева) всегда выступает RP. Источник шлет трафик на RP. RP шлет трафик по общему дереву получателю. Если трафика много, коммутаторы пытаются переключиться на кратчайший путь до источника (Shortest Path Tree / SPT).

При объявлении RP будем использовать статический режим (для EVPN/VxLAN фабрик динамический BSR практически никогда не используется) с выделенным петлевым интерфейсом `loopback255` на стороне коммутаторов уровня Spine.

В рамках данного решения для каждого широковещательного сегмента оверлея (VNI) или группы сегментов (VLAN-Aware Bundle) выделяется уникальный адрес многоадресной рассылки (Multicast Group) из диапазона административно ограничиваемых адресов 239.0.0.0/8.

Каждый VTEP-коммутатор (Leaf) осуществляет аппаратную посадку широковещательного домена на соответствующую мультикаст-группу транспортной сети. Сигнализация и построение деревьев распределения BUM-трафика обеспечиваются протоколом PIM Sparse Mode (PIM-SM).

С точки зрения Underlay-маршрутизации (OSPF/IS-IS/BGP), одинаковый IP-адрес RP анонсируется всеми спайнами одновременно. Ближайшие лифы выбирают кратчайший путь (по метрике IGP или ECMP) до этого адреса.

Однако сам по себе PIM Sparse-Mode не умеет синхронизировать состояние источников мультикаста между разными физическими коммутаторами. Если один Leaf зарегистрировал источник трафика (Source) на Spine01, а другой Leaf запрашивает этот трафик (Receiver) у Spine02, то без дополнительного протокола синхронизации мультикаст работать не будет.

Для решения этой задачи в сетях на Arista EOS применяются два основных протокола: MSDP и PIM Anycast (RFC 4610):
- **Синхронизация через PIM Anycast**: Самый простой и лаконичный способ, встроенный в сам протокол PIM. Спайны общаются между собой, используя свои **уникальные IP-адреса**, а Anycast IP используют только для обслуживания лифов. Когда на `Spine01` приходит запрос регистрации от источника (через Anycast IP), он пересылает сообщение `PIM Register` на все остальные коммутаторы Spine, перечисленные в его peer-листе.
- Синхронизация через MSDP: Является классическим подхобом. Spine'ы устанавливают между собой TCP-соединения. Когда `swSpine01` узнает о новом источнике мультикаста, он генерирует Source-Active (SA) сообщение и отправляет его по TCP всем MSDP-соседям. Таким образом, все Spine'ы ведут идентичную базу данных активных источников.

Итоговый выбор PIM-ASM cо статическим указанием IP-адреса RP на стороне коммутаторов Spine и Anycast-синхронизацией между ними. Сторона Spine'ов:
```
ip multicast-routing
!
interface Loopback255
   description --- Loopback (no VLAN, no VRF): interface for VIP Anycast PIM-ASM
   load-interval 60
   ip address 10.1.255.255/32
   isis enable Underlay
   isis passive
!
interface Ethernet1 - 5
   pim ipv4 sparse-mode
!
router pim sparse-mode
   ipv4
      rp address 10.1.255.255                !! Указание адреса RP PIM Anycast фабирики
      anycast-rp 10.1.255.255 10.1.2.1       !! Настройка синхронизации PIM Anycast c swSpine01
      anycast-rp 10.1.255.255 10.1.2.2       !! Настройка синхронизации PIM Anycast c swSpine02
      anycast-rp 10.1.255.255 10.1.2.3       !! Настройка синхронизации PIM Anycast c swSpine03
```

Проверим работоспособность RP:
```
swSpine01#show ip pim rp
Group: 224.0.0.0/4
  RP: 10.1.255.255
    Uptime: 0:03:18, Expires: never, Priority: 0, Override: False
```

Перенесем данные настройки на все коммутаторы уровня Spine. На коммутаторах уровня Leaf производим следующие настройки:
```
interface Ethernet1 - 3
   pim ipv4 sparse-mode
!
router pim sparse-mode
   ipv4
      rp address 10.1.255.255
!
interface Vxlan1
   vxlan vlan 6 flood group 239.1.0.6
   vxlan vlan 23 flood group 239.1.0.23
   vxlan vlan 137 flood group 239.1.1.37
   vxlan vlan 889 flood group 239.1.8.89
   vxlan vlan 1026 flood group 239.1.10.26
```

Проверяем:
```
swLeaf04(config)#sh ip pim rp
Group: 224.0.0.0/4
  RP: 10.1.255.255
    Uptime: 0:00:49, Expires: never, Priority: 0, Override: False

swLeaf04(config)#sh ip pim neighbor
PIM Neighbor Table for default VRF
Neighbor Address  Interface  Uptime    Expires   Mode    Transport
10.1.2.1          Ethernet1  00:03:03  00:01:35  sparse  datagram
10.1.2.2          Ethernet2  00:03:03  00:01:18  sparse  datagram
10.1.2.3          Ethernet3  00:03:02  00:01:23  sparse  datagram
```

Вся инфраструктура готова. Если посмотреть настройки VxLAN:
```
swLeaf04(config-if-Vx1)#sh vxlan vtep
Remote VTEPS for Vxlan1:

VTEP       Tunnel Type(s)
---------- --------------

Total number of remote VTEPS:  0

swBorderLeaf01#sh vxlan vtep
Remote VTEPS for Vxlan1:

VTEP           Tunnel Type(s)
-------------- --------------
10.1.1.4       unicast

Total number of remote VTEPS:  1
```

Как видно, режим репликации переключился с `flood` на `unicast`.

Однако, несмотря на работоспособность фабрики, у меня не "завелся" Anycast RP в полной мере: не синхронизировались между собой RP расположенные на Spine'ах.

Проверим подключения к мультикаст группам на стороне коммутаторов Spine:
```
swSpine01#sh ip pim upstream joins
Neighbor address: 10.1.1.4
 Via interface: Ethernet4 (10.1.2.1)
  Group: 239.1.8.89
    Joins:
      10.1.1.4/32 SPT
    Prunes:
      No prunes included
  Group: 239.1.0.6
    Joins:
      10.1.1.4/32 SPT
    Prunes:
      No prunes included
  Group: 239.1.10.26
    Joins:
      10.1.1.4/32 SPT
    Prunes:
      No prunes included

swSpine02#sh ip pim upstream joins
Neighbor address: 10.1.1.4
 Via interface: Ethernet4 (10.1.2.2)
  Group: 239.1.0.23
    Joins:
      10.1.1.4/32 SPT
    Prunes:
      No prunes included
Neighbor address: 10.1.1.251
 Via interface: Ethernet5 (10.1.2.2)
  Group: 239.1.0.23
    Joins:
      10.1.1.251/32 SPT
    Prunes:
      No prunes included
  Group: 239.1.1.37
    Joins:
      10.1.1.251/32 SPT
    Prunes:
      No prunes included
  Group: 239.1.10.26
    Joins:
      10.1.1.251/32 SPT
    Prunes:
      No prunes included

swSpine03#sh ip pim upstream joins
Neighbor address: 10.1.1.4
 Via interface: Ethernet4 (10.1.2.3)
  Group: 239.1.1.37
    Joins:
      10.1.1.4/32 SPT
    Prunes:
      No prunes included
Neighbor address: 10.1.1.251
 Via interface: Ethernet5 (10.1.2.3)
  Group: 239.1.8.89
    Joins:
      10.1.1.251/32 SPT
    Prunes:
      No prunes included
  Group: 239.1.0.6
    Joins:
      10.1.1.251/32 SPT
    Prunes:
      No prunes included
```

Разница в выводах команд `show ip pim upstream joins` между тремя Spine-коммутаторами — это штатное поведение отлично работающей фабрики. Эта разница обусловлена тремя фундаментальными сетевыми механизмами: Anycast RP, ECMP-балансировкой в Underlay и логикой RPF (Reverse Path Forwarding) check в протоколе PIM.

Поскольку адрес RP (`10.1.255.255`) настроен как Anycast на всех трех Spine'ах одновременно и анонсируется через IS-IS, каждый Leaf-коммутатор (`swLeaf04` с IP `10.1.1.4` и его сосед `swBorderLeaf01` c IP `10.1.1.251`) видит три одинаковых по стоимости пути (ECMP) до этой точки рандеву.
Когда Leaf генерирует PIM Join-запрос для конкретной мультикаст-группы, он выполняет RPF-выбор: он берет IP-адрес RP, смотрит в свою таблицу юникаст-маршрутизации и выбирает строго одного конкретного соседа (один Spine) для отправки запроса на эту группу. Хеш-алгоритм ECMP на лифах распределяет разные группы по разным аплинкам.

Однако, при этом, если посмотреть маршрут до группы существующего клиента, получим следующее:
```
swSpine01#sh ip mroute 239.1.1.37
PIM Bidirectional Mode Multicast Routing Table
RPF route: U - From unicast routing table
           M - From multicast routing table
PIM Sparse Mode Multicast Routing Table
Flags: E - Entry forwarding on the RPT, J - Joining to the SPT
    R - RPT bit is set, S - SPT bit is set, L - Source is attached
    W - Wildcard entry, X - External component interest
    I - SG Include Join alert rcvd, P - Programmed in hardware
    H - Joining SPT due to policy, D - Joining SPT due to protocol
    Z - Entry marked for deletion, C - Learned from a DR via a register
    A - Learned via Anycast RP Router, M - Learned via MSDP
    N - May notify MSDP, K - Keepalive timer not running
    T - Switching Incoming Interface, B - Learned via Border Router
    V - Source is reachable via Evpn Tenant Domain
    F - Learned via MVPN
RPF route: U - From unicast routing table
           M - From multicast routing table

swSpine02#sh ip mroute 239.1.1.37
PIM Bidirectional Mode Multicast Routing Table
RPF route: U - From unicast routing table
           M - From multicast routing table
PIM Sparse Mode Multicast Routing Table
Flags: E - Entry forwarding on the RPT, J - Joining to the SPT
    R - RPT bit is set, S - SPT bit is set, L - Source is attached
    W - Wildcard entry, X - External component interest
    I - SG Include Join alert rcvd, P - Programmed in hardware
    H - Joining SPT due to policy, D - Joining SPT due to protocol
    Z - Entry marked for deletion, C - Learned from a DR via a register
    A - Learned via Anycast RP Router, M - Learned via MSDP
    N - May notify MSDP, K - Keepalive timer not running
    T - Switching Incoming Interface, B - Learned via Border Router
    V - Source is reachable via Evpn Tenant Domain
    F - Learned via MVPN
RPF route: U - From unicast routing table
           M - From multicast routing table
239.1.1.37
  10.1.1.251, 0:57:37, flags: SP
    Incoming interface: Ethernet5
    RPF route: [U] 10.1.1.251/32 [115/20] via 10.1.1.251
    Outgoing interface list:
      Ethernet4

swSpine03#sh ip mroute 239.1.1.37
PIM Bidirectional Mode Multicast Routing Table
RPF route: U - From unicast routing table
           M - From multicast routing table
PIM Sparse Mode Multicast Routing Table
Flags: E - Entry forwarding on the RPT, J - Joining to the SPT
    R - RPT bit is set, S - SPT bit is set, L - Source is attached
    W - Wildcard entry, X - External component interest
    I - SG Include Join alert rcvd, P - Programmed in hardware
    H - Joining SPT due to policy, D - Joining SPT due to protocol
    Z - Entry marked for deletion, C - Learned from a DR via a register
    A - Learned via Anycast RP Router, M - Learned via MSDP
    N - May notify MSDP, K - Keepalive timer not running
    T - Switching Incoming Interface, B - Learned via Border Router
    V - Source is reachable via Evpn Tenant Domain
    F - Learned via MVPN
RPF route: U - From unicast routing table
           M - From multicast routing table
239.1.1.37
  10.1.1.4, 17:24:36, flags: SP
    Incoming interface: Ethernet4
    RPF route: [U] 10.1.1.4/32 [115/20] via 10.1.1.4
    Outgoing interface list:
      Ethernet5
```

Как видно, синхронизация Anycast не работает.

На данный момент, ввиду работоспособности всей фабрики, я решил отложить эксперементы до окончательного завершения лабораторной.

#### Настройка стыка с сетью Интернет
Начну со стороны стыка с роутером-файрволом `fwBorder01`, на коммутаторе фабрики `swBorderLeaf01`.

Арендаторские VLAN  используются ими для терминирования своих устройств и сервисов. Было бы не очень хорошей идеей "растягивать" какой-нибудь из пользовательских VLAN до "роутера-на-палке". Для этих целей я попробую использовать VLAN предназначенные для L3VPN:
```
interface Vlan4001
   description --- Virtual (VLAN4001, VRF: TENANT-A): interface for L3 VPN tunneling via Symmetric IRB
   no autostate
   vrf TENANT-A
   ip address 10.1.101.1/31
!
interface Vlan4002
   description --- Virtual (VLAN4002, VRF: TENANT-B): interface for L3 VPN tunneling via Symmetric IRB
   no autostate
   vrf TENANT-B
   ip address 10.1.102.1/31
```

Конфигурация роутера `fwBorder01` предполагает терминирование каждого арендатора в собственный VRF и
имеет следующий вид:
```
hostname fwBorder01
!
ip domain name local
!
vrf definition TENANT-A
 description --- VRF: Tenant A RIB
 rd 65101:101
 !
 address-family ipv4
  route-target export 65101:101
  route-target import 65101:101
 exit-address-family
!
vrf definition TENANT-B
 description --- VRF: Tenant B RIB
 rd 65102:102
 !
 address-family ipv4
  route-target export 65102:102
  route-target import 65102:102
 exit-address-family
!
lldp run
!
interface GigabitEthernet1
 description --- Trunk (VLAN001): connection to swBorderLeaf01:Ethernet4
 no ip address
 load-interval 60
 negotiation auto
 no mop enabled
 no mop sysid
!
interface GigabitEthernet1.4001
 description --- Virtual (VLAN4001, no VRF): interconnection to Tenant A
 encapsulation dot1Q 4001
 vrf forwarding TENANT-A
 ip address 10.1.101.0 255.255.255.254
!
interface GigabitEthernet1.4002
 description --- Virtual (VLAN4002, no VRF): interconnection to Tenant B
 encapsulation dot1Q 4002
 vrf forwarding TENANT-B
 ip address 10.1.102.0 255.255.255.254
!
interface GigabitEthernet2
 description --- Access (VLAN001): Connection to Internet
 ip dhcp client client-id ascii fwBorder01
 ip address dhcp
 negotiation auto
 no mop enabled
 no mop sysid
!
ip route 0.0.0.0 0.0.0.0 dhcp
```

Проверим связанность со стороны роутера:
```
fwBorder01#ping vrf TENANT-A 10.1.101.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.101.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/6/11 ms

fwBorder01#ping vrf TENANT-B 10.1.102.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.102.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/5/8 ms
```

Теперь на стороне коммутатора `swBorderLeaf01`, если посмотреть маршруты EVPN, можно увидеть маршруты типа 5 (`ip-prefix`):
```
swBorderLeaf01#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.1.251, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.1.251:101 ip-prefix 10.1.101.0/31
                                 -                     -       -       0       i
 * >      RD: 10.1.1.251:102 ip-prefix 10.1.102.0/31
                                 -                     -       -       0       i
```

У меня `fwBorder01` подключен к сети Интернет через интерфейс `Gi2` (DHCP):
```
fwBorder01#sh ip route
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
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
C        10.1.10.0/24 is directly connected, GigabitEthernet2
S        10.1.10.2/32 [254/0] via 10.1.10.2, GigabitEthernet2
L        10.1.10.90/32 is directly connected, GigabitEthernet2

fwBorder01# ping 8.8.8.8 repeat 3
Type escape sequence to abort.
Sending 3, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!!!
Success rate is 100 percent (3/3), round-trip min/avg/max = 35/35/36 ms
```

Необходимо создать eBGP-соседство между VRF'ами на `swBorderLeaf01` и `fwBorder01`, что позволит добиться связанности вдоль сети фабрики к точке терминирования VRF пользователя на стороне выделенного роутера.

На стороне `swBorderLeaf01` произведем следующие настройки:
```
router bgp 65001
   neighbor grpONEARMSYSTEM peer group
   neighbor grpONEARMSYSTEMS peer group
   neighbor grpONEARMSYSTEMS bfd
   neighbor grpONEARMSYSTEMS bfd interval 100 min-rx 100 multiplier 3
   !
   vrf TENANT-A
      neighbor 10.1.101.0 peer group grpONEARMSYSTEM
      neighbor 10.1.101.0 remote-as 65100
      neighbor 10.1.101.0 update-source Vlan4001
   !
   vrf TENANT-B
      neighbor 10.1.102.0 peer group grpONEARMSYSTEM
      neighbor 10.1.102.0 remote-as 65100
      neighbor 10.1.102.0 update-source Vlan4002
```

На стороне файрвола `fwBorder01`:
```
interface GigabitEthernet1.4001
 bfd interval 100 min_rx 100 multiplier 3    ! В Cisco IOS тюнинг bfd осуществляется в контексте интерфейса
!
interface GigabitEthernet1.4002
 bfd interval 100 min_rx 100 multiplier 3
!
router bgp 65100
 bgp router-id 10.1.100.1
 bgp log-neighbor-changes
 bgp update-delay 1
 bgp graceful-restart restart-time 300
 bgp graceful-restart
 timers bgp 3 9
 !
 address-family ipv4 vrf TENANT-A
  neighbor grpLEAFS peer-group
  neighbor grpLEAFS description --- Peer: connection between System-on-a-Stick and Leaf switches
  neighbor grpLEAFS fall-over bfd
  neighbor 10.1.101.1 remote-as 65001
  neighbor 10.1.101.1 peer-group grpLEAFS
  neighbor 10.1.101.1 update-source GigabitEthernet1.4001
  neighbor 10.1.101.1 activate
  neighbor 10.1.101.1 default-originate
 exit-address-family
!
 address-family ipv4 vrf TENANT-B
  neighbor 10.1.102.1 remote-as 65001
  neighbor 10.1.102.1 description --- Peer: connection between System-on-a-Stick and Leaf switches in VRF TENANT-B
  neighbor 10.1.102.1 update-source GigabitEthernet1.4002
  neighbor 10.1.102.1 fall-over bfd
  neighbor 10.1.102.1 activate
  neighbor 10.1.102.1 default-originate
 exit-address-family
```

На этой стадии можно проверить состояние таблиц маршрутизации:
```
swBorderLeaf01#sh ip route vrf TENANT-B

VRF: TENANT-B
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
 B E      0.0.0.0/0 [20/0]
           via 10.1.102.0, Vlan4002

 C        10.1.102.0/31
           directly connected, Vlan4002
```

Осталось настроить NAT с route-leaking на стороне `fwBorder01`:
```
interface GigabitEthernet1.4001
 ip nat inside
!
interface GigabitEthernet1.4002
 ip nat inside
!
interface GigabitEthernet2
 ip nat outside
!
ip access-list standard aclNAT:TENANT-A
 10 permit any
!
ip access-list standard aclNAT:TENANT-B
 10 permit any
!
ip nat inside source list aclNAT:TENANT-A interface GigabitEthernet2 vrf TENANT-A overload
ip nat inside source list aclNAT:TENANT-B interface GigabitEthernet2 vrf TENANT-B overload
!
ip route vrf TENANT-A 0.0.0.0 0.0.0.0 10.1.10.2 global
ip route vrf TENANT-B 0.0.0.0 0.0.0.0 10.1.10.2 global
```

Проверим со стороны `swBorderLeaf01`:
```
swBorderLeaf01#ping vrf TENANT-A 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 72(100) bytes of data.
80 bytes from 8.8.8.8: icmp_seq=1 ttl=105 time=36.3 ms
80 bytes from 8.8.8.8: icmp_seq=2 ttl=105 time=36.0 ms
80 bytes from 8.8.8.8: icmp_seq=3 ttl=105 time=34.5 ms
80 bytes from 8.8.8.8: icmp_seq=4 ttl=105 time=37.4 ms
80 bytes from 8.8.8.8: icmp_seq=5 ttl=105 time=33.2 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 111ms
rtt min/avg/max/mdev = 33.218/35.468/37.433/1.473 ms, pipe 3, ipg/ewma 27.693/35.818 ms

swBorderLeaf01#ping vrf TENANT-B 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 72(100) bytes of data.
80 bytes from 8.8.8.8: icmp_seq=1 ttl=105 time=33.5 ms
80 bytes from 8.8.8.8: icmp_seq=2 ttl=105 time=34.7 ms
80 bytes from 8.8.8.8: icmp_seq=3 ttl=105 time=35.1 ms
80 bytes from 8.8.8.8: icmp_seq=4 ttl=105 time=36.7 ms
80 bytes from 8.8.8.8: icmp_seq=5 ttl=105 time=34.6 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 104ms
rtt min/avg/max/mdev = 33.502/34.900/36.658/1.026 ms, pipe 3, ipg/ewma 26.107/34.231 ms
```

#### Настройка подключения сервера srvHost03 (L2 Multi-Home)
Переходим к коммутаторам `swLeaf03`, `swLeaf04` и `swBorderLeaf01`. Их конфигурация опирается на описанную в части [Настройка базового функционала Underlay/Overlay](#настройка-базового-функционала-underlayoverlay) с дополнением, связанным с абонентским подключением по технологии Multi-Home:
```
```













---

swLeaf04
```
interface Ethernet5
   description --- Trunk (VLAN001): connection to srvHost03:Gi2
   load-interval 60
   switchport mode trunk
```

srvHost03
```

```








---


```
vlan 4091
   !! Interconnect VLAN for Tenant A
   name TENANT-A:Interconnect
!
vlan 4092
   !! Interconnect VLAN for Tenant B
   name TENANT-B:Interconnect
!
```


!!!

maximum-paths 16

После этого можно будет настроить подключение к роутеру `fwBorder01`:
```

```





Далее, настройки



---

router bgp 65001
   vrf TENANT-A
      rd 10.1.1.1:65101
      route-target import evpn 65101:101
      route-target export evpn 65101:101

---



VLAN могут быть произвольными, как и IP в VRF - это прирогатива заказчика. Для выхода мы используем nat

VNI - 110000 VLAN100
VLAN101 - VNI110011

VNI -120000 VLAN200
VLAN201 - VNI120021

Allowed range is: 1-1677
10010000
1- POD1
001 - Tenant1
0000 - VLAN

куда совать L3VNI для каждого тенанта








---

1. Ускорение сходимости и оптимизация трафика
Помимо уже настроенного advertisement-interval 0, для EVPN оверлея критически важны следующие параметры:
• update-wait-time 0 и update-wait-install 0
По умолчанию Arista после перезагрузки или падения сессии выдерживает паузу перед отправкой и установкой EVPN-маршрутов, чтобы собрать полную картину сети. В лабах и отказоустойчивых фабриках эту задержку отключают для мгновенного старта.
• next-hop-unchanged (только если оверлей на iBGP)
Если ваши Spine и Leaf находятся в одной AS (iBGP), то Spine-коммутаторы при пересылке EVPN Route Type-2/3 не должны менять IP Next-Hop на свой собственный. Иначе Leaf-коммутаторы попытаются построить VXLAN-туннель до Spine, а не до целевого Leaf.
2. Защита от «мигания» сети (Route Flapping)
• bgp convergence-time 0
Ускоряет время, через которое Arista считает сеть сошедшейся, убирая внутренние системные задержки планировщика BGP.
• evpn route-flap-damping
Если на каком-то сервере начнет «мигать» сетевой интерфейс, он завалит всю фабрику миллионами EVPN-апдейтов (MAC/IP Route Type-2). Этот механизм временно штрафует и замораживает нестабильные маршруты на Leaf, защищая Control Plane коммутаторов.
3. Масштабирование и оптимизация памяти
• graceful-restart (или long-lived-graceful-restart)
Позволяет Control Plane (процессу BGP) перезагрузиться (например, при обновлении Arista EOS) без прерывания передачи трафика через Data Plane (чип коммутатора). Соседи будут удерживать маршруты, зная, что коммутатор скоро вернется.

router bgp 65001
   ! -- Общий тюнинг процесса BGP --
   bgp convergence-time 0
   update-wait-time 0
   update-wait-install 0
   graceful-restart restart-time 120
   
   ! -- Настройка группы соседей --
   neighbor OVERLAY-PEERS peer group
   neighbor OVERLAY-PEERS remote-as 65001  <-- (Если iBGP, для eBGP укажите remote-as external)
   neighbor OVERLAY-PEERS update-source Loopback0
   neighbor OVERLAY-PEERS send-community
   neighbor OVERLAY-PEERS fall-over bfd
   neighbor OVERLAY-PEERS advertisement-interval 0



Чтобы при падении (отвале) BGP-соседа маршрутизатор Arista отправлял EVPN-апдейты (а именно — отзывы маршрутов, Withdrawals) незамедлительно, вам нужно отключить задержку, которая называется MRAI (Min Route Advertisement Interval).



===
У каждого из них существуют свои

VLAN, VNI, L3VPI, VRF

*Таблица 1: Настройки арендаторов*

| Hostname | Lb0 IPv4 | AFI | Area ID | System ID | NSEL |

===

Будем считать, что в DC расположены два клиента: TenantA и TenantB и их сервисы могут мигрировать на любой из хостов (в лабораторной - хост-роутер). У каждого из клиентов есть некоторое количество L2 сегментов. Связь между клиентами возможна только через файрвол rtBorder01, стоящим также на стыке с интернетом.

### Настройка фабрики
Настройку фабрики (для удобства проверки) начнем с `swBorderLeaf01`. Удалим все настройки, связанные с OSPF, старые VLAN и все настройки

