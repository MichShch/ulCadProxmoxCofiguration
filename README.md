# Инструкция по развёртыванию Proxmox-кластера с SDN EVPN и OpenFabric

## Назначение

Этот репозиторий описывает порядок установки и настройки вычислительной инфраструктуры для массового запуска задач моделирования. Инфраструктура построена на базе Proxmox VE и рассчитана на масштабирование за счёт добавления физических узлов и развёртывания на них виртуальных машин.

По материалам ВКР целевая конфигурация включает:

- кластер Proxmox из физических серверов;
- централизованное управление виртуальными машинами;
- возможность миграции ВМ между физическими нодами;
- SDN-сеть Proxmox с EVPN/VXLAN;
- OpenFabric как underlay-фабрику для связности между нодами;
- DHCP и DNS для автоматизации подключения ВМ;
- шаблоны виртуальных машин и Terraform для ускоренного развёртывания рабочих узлов.

В описанной инфраструктуре использовалось 11 физических узлов. Каждый типовой узел имел 32 ГБ ОЗУ, 16 вычислительных ядер и накопитель 512 ГБ. После переноса вычислений в Proxmox стало возможно одновременно использовать до 30 виртуальных машин для моделирования.

## Общая архитектура

Инфраструктура делится на несколько уровней:

1. Физические серверы.
   На них устанавливается Proxmox VE. Каждый сервер становится нодой кластера.

2. Proxmox-кластер.
   Объединяет физические ноды в единую административную среду. Управление выполняется через веб-интерфейс одного из узлов или через CLI.

3. SDN underlay.
   Между физическими нодами настраивается OpenFabric. Он обеспечивает маршрутизируемую связность между узлами и используется EVPN-контроллером как транспортная основа.

4. SDN overlay.
   Поверх underlay создаётся EVPN-зона. В ней размещаются VNet-сети для виртуальных машин. ВМ могут находиться на разных физических нодах, но оставаться в одной логической сети.

5. Сервисная подсеть.
   В отдельной сети разворачиваются DHCP, DNS и при необходимости дополнительные служебные сервисы.

6. Рабочие виртуальные машины.
   ВМ создаются из шаблонов и подключаются к VNet. Они используются как рабочие узлы системы распределённых вычислений.

## Сабмодули репозитория

В репозитории подключены вспомогательные сабмодули, которые описывают отдельные части инфраструктуры:

| Сабмодуль | Назначение | Что содержит |
|---|---|---|
| `cad_terraform` | Автоматизированное создание рабочих ВМ в Proxmox | Terraform-конфигурацию, модуль пула `DCWorker`, локальный mirror/cache провайдера `bpg/proxmox` |
| `cad_dhcp` | DHCP для виртуальных сетей кластера | Конфигурацию Kea DHCPv4 с адресными пулами для сетей `10.10.0.0/24` и `10.10.1.0/24` |
| `cad_dns` | DNS для внутренней инфраструктуры | Конфигурацию Unbound, локальную зону `cad.internal` и записи для Proxmox-нод |

После клонирования репозитория сабмодули нужно инициализировать:

```bash
git submodule update --init --recursive
```

Если сабмодули уже были добавлены, но требуется получить свежие изменения:

```bash
git submodule update --remote --recursive
```

### `cad_terraform`

Сабмодуль `cad_terraform` отвечает за развёртывание рабочих виртуальных машин в Proxmox через Terraform. Он использует провайдер `bpg/proxmox` версии `~> 0.103`.

Основная лабораторная конфигурация находится в каталоге:

```text
cad_terraform/envirnoments/lab
```

Внутри описан модуль `dcworker_pool`, который:

- берёт исходный шаблон ВМ `source_template_vm_id = 3000` на ноде `pve1`;
- создаёт полные шаблоны на целевых нодах `pve1` - `pve11`;
- создаёт linked clone рабочих ВМ из локального шаблона на каждой ноде;
- настраивает получение IPv4-адреса по DHCP через cloud-init;
- подключает ВМ к bridge `virt2`;
- использует хранилище `local-zfs`;
- по умолчанию задаёт 2 сокета, 4 ядра на сокет и 20480 МБ ОЗУ на worker.

Текущий пример создаёт 29 worker-ВМ: по одной на `pve1` и `pve3`, по три на остальных нодах `pve2`, `pve4` - `pve11`.

Перед применением нужно заполнить переменные доступа к API Proxmox:

```bash
cd cad_terraform/envirnoments/lab
```

Реальный `terraform.tfvars` не хранится в git. Для старта можно скопировать пример:

```bash
cp terraform.tfvars.example terraform.tfvars
```

Значения `proxmox_api_token_id` и `proxmox_api_token_secret` лучше передавать через переменные окружения, чтобы не хранить секреты в репозитории:

```bash
export TF_VAR_proxmox_api_token_id='terraform@pve!iac'
export TF_VAR_proxmox_api_token_secret='token-secret'
terraform init
terraform plan
terraform apply
```

На Windows PowerShell:

```powershell
$env:TF_VAR_proxmox_api_token_id='terraform@pve!iac'
$env:TF_VAR_proxmox_api_token_secret='token-secret'
terraform init
terraform plan
terraform apply
```

### `cad_dhcp`

Сабмодуль `cad_dhcp` содержит конфигурацию Kea DHCPv4:

```text
cad_dhcp/kea-dhcp4.conf
```

DHCP настроен на работу с интерфейсами:

```text
eth0
eth1
```

В конфигурации описаны две подсети:

| Подсеть | DHCP pool | Gateway | DNS | Domain |
|---|---|---|---|---|
| `10.10.0.0/24` | `10.10.0.100 - 10.10.0.199` | `10.10.0.1` | `10.10.0.2` | `cad.internal` |
| `10.10.1.0/24` | `10.10.1.100 - 10.10.1.199` | `10.10.1.1` | `10.10.0.2` | `cad.internal` |

Файл аренды сохраняется в:

```text
/var/lib/kea/kea-leases4.csv
```

Типовой порядок установки на Debian/Ubuntu:

```bash
apt update
apt install kea-dhcp4-server
cp cad_dhcp/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
systemctl restart kea-dhcp4-server
systemctl status kea-dhcp4-server
```

Перед запуском нужно убедиться, что имена интерфейсов в конфигурации Kea совпадают с интерфейсами DHCP-сервера.

### `cad_dns`

Сабмодуль `cad_dns` содержит конфигурацию Unbound:

```text
cad_dns/unbound.conf
cad_dns/unbound.conf.d/infra.conf
cad_dns/unbound.conf.d/remote-control.conf
cad_dns/unbound.conf.d/root-auto-trust-anchor-file.conf
```

Конфигурация:

- слушает DNS-запросы на `0.0.0.0` и `10.10.0.2`;
- разрешает запросы из `127.0.0.0/8`, `10.10.0.0/24`, `10.10.1.0/24`;
- включает IPv4, UDP и TCP;
- скрывает identity/version сервера;
- включает DNSSEC trust anchor;
- создаёт локальную зону `cad.internal`;
- содержит A-записи для `pve1.cad.internal` - `pve11.cad.internal`.

Типовая установка:

```bash
apt update
apt install unbound
cp cad_dns/unbound.conf /etc/unbound/unbound.conf
cp cad_dns/unbound.conf.d/*.conf /etc/unbound/unbound.conf.d/
unbound-checkconf
systemctl restart unbound
systemctl status unbound
```

Важно проверить согласованность DNS-адреса. DHCP выдаёт клиентам DNS `10.10.0.2`, и Unbound также слушает `10.10.0.2`. Если DNS-сервер фактически размещён на другом IP, нужно одновременно обновить `cad_dhcp/kea-dhcp4.conf` и `cad_dns/unbound.conf.d/infra.conf`.

## Пример адресного плана

Ниже приведён пример. Перед установкой его нужно адаптировать под свою сеть.

| Назначение | Пример | Комментарий |
|---|---:|---|
| Management-сеть Proxmox | `192.168.10.0/24` | Доступ к веб-интерфейсу и SSH |
| Corosync/cluster-сеть | `192.168.20.0/24` | Желательно выделить отдельный интерфейс или VLAN |
| OpenFabric router-id prefix | `10.255.0.0/24` | Loopback/router-id адреса нод |
| EVPN VNet для рабочих ВМ | `10.10.0.0/24` | Сеть виртуальных машин |
| Gateway VNet | `10.10.0.1` | Anycast gateway на всех нодах EVPN-зоны |
| ASN EVPN | `65000` | Частный BGP ASN |
| VRF VXLAN ID | `10000` | Не должен совпадать с VNet VXLAN ID |
| VNet VXLAN ID | `11000` | Уникальный ID конкретной VNet |

Пример имён физических нод:

| Нода | Management IP | Corosync IP | OpenFabric router-id |
|---|---:|---:|---:|
| `pve01` | `192.168.10.11` | `192.168.20.11` | `10.255.0.1` |
| `pve02` | `192.168.10.12` | `192.168.20.12` | `10.255.0.2` |
| `pve03` | `192.168.10.13` | `192.168.20.13` | `10.255.0.3` |

## Требования перед установкой

Перед установкой нужно подготовить:

- ISO-образ Proxmox VE;
- статические IP-адреса для всех физических нод;
- уникальные hostname для всех нод;
- доступ к интернету или локальному зеркалу пакетов;
- включённую виртуализацию в BIOS/UEFI: Intel VT-x/VT-d или AMD-V/IOMMU;
- синхронизацию времени;
- сетевые интерфейсы для управления, кластерной сети и, желательно, отдельного трафика SDN/VM;
- одинаковую или совместимую версию Proxmox VE на всех нодах.

Важно: hostname и основной IP-адрес ноды лучше задать окончательно до создания или присоединения к кластеру. Переименование ноды Proxmox после включения в кластер возможно, но неудобно и повышает риск ошибок.

## Установка Proxmox VE на физический узел

1. Загрузиться с ISO Proxmox VE.

2. Выбрать диск для установки.
   Для тестовой или учебной инфраструктуры можно использовать один локальный SSD. Для production-среды лучше использовать RAID/ZFS mirror.

3. Задать регион, часовой пояс и раскладку.

4. Указать пароль `root` и email администратора.

5. Настроить сеть:

   - hostname: например, `pve01.local`;
   - management IP: например, `192.168.10.11/24`;
   - gateway: например, `192.168.10.1`;
   - DNS: адрес локального или внешнего DNS.

6. Завершить установку и перезагрузить сервер.

7. Открыть веб-интерфейс:

```text
https://192.168.10.11:8006
```

8. Подключиться по SSH:

```bash
ssh root@192.168.10.11
```

## Первичная настройка ноды

Обновить систему:

```bash
apt update
apt full-upgrade
reboot
```

## Настройка репозиториев без подписки

После чистой установки Proxmox VE по умолчанию может быть включён enterprise-репозиторий. Он предназначен для систем с активной подпиской. Если подписки нет, при `apt update` будут появляться ошибки доступа к enterprise-репозиторию. Для учебной, лабораторной или тестовой инфраструктуры можно использовать no-subscription репозиторий.

### Отключение enterprise-репозитория

Проверить список подключённых репозиториев:

```bash
grep -R "enterprise.proxmox.com" /etc/apt/sources.list /etc/apt/sources.list.d/
```

Обычно enterprise-репозиторий находится в одном из файлов:

```text
/etc/apt/sources.list.d/pve-enterprise.list
/etc/apt/sources.list.d/ceph.list
```

Отключить Proxmox enterprise-репозиторий:

```bash
sed -i 's/^deb/# deb/' /etc/apt/sources.list.d/pve-enterprise.list
```

Если используется enterprise-репозиторий Ceph и подписки нет, его также нужно отключить:

```bash
sed -i 's/^deb/# deb/' /etc/apt/sources.list.d/ceph.list
```

Если файла нет, команда может вывести ошибку. Это нормально: значит соответствующий enterprise-репозиторий на ноде не настроен.

### Добавление Proxmox no-subscription

Для Proxmox VE 8 на базе Debian 12 Bookworm добавить:

```bash
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
```

Для Proxmox VE 9 на базе Debian 13 Trixie добавить:

```bash
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
```

Версию Debian можно проверить командой:

```bash
cat /etc/os-release
```

После изменения репозиториев обновить индексы пакетов:

```bash
apt update
```

Если всё настроено правильно, ошибок авторизации enterprise-репозитория быть не должно.

### Добавление Ceph no-subscription при необходимости

Этот шаг нужен только если в кластере планируется использовать Ceph.

Для Proxmox VE 8 и актуальной ветки Ceph, выбранной при установке, no-subscription репозиторий обычно настраивается через веб-интерфейс:

```text
Node -> Ceph -> Install Ceph
```

Если Ceph уже установлен и требуется проверить репозитории:

```bash
grep -R "download.proxmox.com/debian/ceph" /etc/apt/sources.list /etc/apt/sources.list.d/
```

Не нужно подключать Ceph-репозиторий, если Ceph в инфраструктуре не используется.

Проверить hostname:

```bash
hostnamectl
cat /etc/hosts
```

В `/etc/hosts` должна быть запись, связывающая IP ноды с её hostname:

```text
192.168.10.11 pve01.local pve01
```

Проверить время:

```bash
timedatectl
```

Если время не синхронизируется, включить NTP:

```bash
timedatectl set-ntp true
```

## Настройка сетевых интерфейсов Proxmox

Базовая management-сеть обычно создаётся установщиком Proxmox в виде Linux Bridge `vmbr0`.

Пример `/etc/network/interfaces` для management-интерфейса:

```text
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.10.11/24
        gateway 192.168.10.1
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0

source /etc/network/interfaces.d/*
```

Если используется отдельная сеть Corosync, для неё можно настроить отдельный интерфейс без gateway:

```text
auto eno2
iface eno2 inet static
        address 192.168.20.11/24
```

После изменения сетевой конфигурации применить её:

```bash
ifreload -a
```

Проверить адреса:

```bash
ip addr
ip route
```

## Создание Proxmox-кластера

Команды выполняются на первой ноде, например `pve01`.

Создать кластер:

```bash
pvecm create noc-cluster
```

Проверить состояние:

```bash
pvecm status
pvecm nodes
```

В веб-интерфейсе кластер также появится в разделе:

```text
Datacenter -> Cluster
```

## Добавление новых физических узлов в кластер

Перед добавлением новой ноды нужно убедиться, что:

- на ней установлена та же мажорная версия Proxmox VE;
- hostname уникален;
- IP-адрес не конфликтует с другими узлами;
- время синхронизировано;
- нода не содержит важных ВМ, которые могут конфликтовать по VMID;
- все нужные сети доступны с уже существующих нод;
- SSH до существующей ноды работает.

На новой ноде, например `pve02`, выполнить:

```bash
pvecm add 192.168.10.11
```

Если для Corosync используется отдельная сеть, нужно явно указать локальный адрес новой ноды в этой сети:

```bash
pvecm add 192.168.10.11 --link0 192.168.20.12
```

Где:

- `192.168.10.11` - адрес уже существующей ноды кластера;
- `192.168.20.12` - Corosync-адрес добавляемой ноды.

После добавления проверить состояние на любой ноде:

```bash
pvecm status
pvecm nodes
```

Кластер должен быть в состоянии `Quorate: Yes`.

## Рекомендации по quorum

Для устойчивой работы желательно использовать нечётное количество узлов или настроить qdevice. Минимальная практически удобная конфигурация для HA - три узла.

Проверка quorum:

```bash
pvecm status
```

Если кластер потерял quorum, нельзя бездумно править конфигурацию вручную. Сначала нужно восстановить доступность узлов или понять, какая часть кластера является актуальной.

## Установка компонентов SDN и FRR

Для EVPN и OpenFabric на каждой ноде должны быть установлены FRR и вспомогательные пакеты:

```bash
apt update
apt install frr frr-pythontools ifupdown2
```

Проверить, что в `/etc/network/interfaces` есть строка:

```text
source /etc/network/interfaces.d/*
```

Проверить службу FRR:

```bash
systemctl status frr
```

Включить FRR:

```bash
systemctl enable --now frr
```

## Настройка OpenFabric

OpenFabric используется как underlay-сеть между физическими нодами. Он автоматически формирует маршрутизируемую связность между узлами и удобен для EVPN, потому что избавляет от ручного описания всех маршрутов.

Настройка выполняется на уровне Datacenter:

```text
Datacenter -> SDN -> Fabrics
```

### Создание Fabric

1. Нажать `Add Fabric`.
2. Выбрать тип `OpenFabric`.
3. Задать имя, например:

```text
fab0
```

4. Задать IPv4 prefix для router-id, например:

```text
10.255.0.0/24
```

Имя OpenFabric fabric должно быть коротким. В документации Proxmox указано ограничение до 8 символов.

### Добавление нод в Fabric

Для каждой ноды добавить запись:

| Нода | Router-ID | Интерфейс |
|---|---:|---|
| `pve01` | `10.255.0.1` | `eno2` или dedicated fabric NIC |
| `pve02` | `10.255.0.2` | `eno2` или dedicated fabric NIC |
| `pve03` | `10.255.0.3` | `eno2` или dedicated fabric NIC |

В интерфейсе Proxmox это делается через:

```text
Datacenter -> SDN -> Fabrics -> fab0 -> Add Node
```

Для каждой ноды нужно указать:

- node;
- IPv4 router-id;
- интерфейсы, через которые нода связана с другими нодами.

После настройки нажать:

```text
Datacenter -> SDN -> Apply
```

### Проверка OpenFabric

На каждой ноде:

```bash
vtysh -c "show openfabric neighbor"
vtysh -c "show ip route"
ip addr
```

Также нужно проверить связность router-id адресов:

```bash
ping 10.255.0.2
ping 10.255.0.3
```

Если соседства OpenFabric не поднимаются, проверить:

- физическую связность интерфейсов;
- VLAN, если используется коммутатор с VLAN;
- firewall на Proxmox и внешних сетевых устройствах;
- что на всех нодах установлен и запущен FRR;
- что SDN-конфигурация применена через `Apply`;
- что router-id уникален на каждой ноде.

## Настройка EVPN-контроллера

EVPN в Proxmox использует FRR и BGP EVPN. В выбранной схеме EVPN-контроллер опирается на OpenFabric fabric.

Открыть:

```text
Datacenter -> SDN -> Controllers
```

Создать EVPN controller:

| Поле | Значение |
|---|---|
| ID | `evpnctl0` |
| ASN | `65000` |
| SDN Fabric | `fab0` |

Если fabric не используется, нужно вручную указать peers - IP-адреса всех нод EVPN. В данной схеме предпочтительно использовать именно OpenFabric, потому что он автоматически описывает underlay-связность.

После создания контроллера нажать:

```text
Datacenter -> SDN -> Apply
```

Проверить BGP/EVPN:

```bash
vtysh -c "show bgp summary"
vtysh -c "show bgp l2vpn evpn summary"
```

## Создание EVPN-зоны

Открыть:

```text
Datacenter -> SDN -> Zones
```

Создать зону:

| Поле | Значение |
|---|---|
| Type | `EVPN` |
| ID | `evpn-zone0` |
| Controller | `evpnctl0` |
| VRF VXLAN ID | `10000` |
| MTU | `1450` |
| Exit Nodes | `pve01` или `pve01,pve02` |
| Primary Exit Node | `pve01`, если нужен один основной выход |
| SNAT | включить, если ВМ должны выходить наружу через exit node |

VRF VXLAN ID должен отличаться от VXLAN ID всех VNet.

MTU обычно задаётся `1450`, потому что VXLAN добавляет инкапсуляцию. Если физическая сеть поддерживает jumbo frames, можно проектировать MTU иначе, но значение должно быть согласовано на всех участках пути.

После создания зоны применить изменения:

```text
Datacenter -> SDN -> Apply
```

## Создание VNet для виртуальных машин

Открыть:

```text
Datacenter -> SDN -> VNets
```

Создать VNet:

| Поле | Значение |
|---|---|
| ID | `worknet` |
| Zone | `evpn-zone0` |
| Tag/VXLAN ID | `11000` |

Создать subnet для VNet:

| Поле | Значение |
|---|---|
| Subnet | `10.10.0.0/24` |
| Gateway | `10.10.0.1` |
| SNAT | включить при необходимости выхода наружу |

Применить изменения:

```text
Datacenter -> SDN -> Apply
```

После применения на нодах появится bridge/VNet, который можно выбирать в настройках сетевого интерфейса ВМ.

## Подключение ВМ к EVPN VNet

При создании или редактировании ВМ:

```text
VM -> Hardware -> Network Device
```

Выбрать bridge:

```text
worknet
```

Рекомендуемые параметры:

- model: `VirtIO (paravirtualized)`;
- firewall: по необходимости;
- VLAN Tag: не указывать, если изоляция уже задаётся через VNet/VXLAN;
- MTU внутри гостевой ОС: обычно `1450`, если возникают проблемы с фрагментацией.

В гостевой ОС настроить адрес вручную:

```text
IP: 10.10.0.101/24
Gateway: 10.10.0.1
DNS: 10.10.0.10
```

Или использовать DHCP, если он настроен для зоны.

## Создание шаблона виртуальной машины

Шаблон нужен, чтобы быстро разворачивать однотипные рабочие ВМ.

1. Создать новую ВМ.
2. Установить гостевую ОС, например Debian или Ubuntu Server.
3. Установить QEMU guest agent:

```bash
apt update
apt install qemu-guest-agent
systemctl enable --now qemu-guest-agent
```

4. Настроить cloud-init, SSH-ключи и базовые пакеты.
5. Очистить machine-id:

```bash
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id
ln -s /etc/machine-id /var/lib/dbus/machine-id
```

6. Выключить ВМ.
7. В Proxmox нажать:

```text
VM -> More -> Convert to template
```

Или через CLI:

```bash
qm template 9000
```

Где `9000` - VMID базовой ВМ.

## Клонирование рабочих ВМ

Создать linked clone:

```bash
qm clone 9000 101 --name worker-01 --full 0
```

Создать full clone:

```bash
qm clone 9000 101 --name worker-01 --full 1
```

Настроить CPU, RAM и сеть:

```bash
qm set 101 --cores 4 --memory 4096
qm set 101 --net0 virtio,bridge=worknet
qm start 101
```

Проверить, что ВМ получила адрес:

```bash
qm guest cmd 101 network-get-interfaces
```

Конкретный provider и синтаксис могут отличаться в зависимости от выбранного Terraform-провайдера для Proxmox.

## Проверка работоспособности всей схемы

### Проверка кластера

```bash
pvecm status
pvecm nodes
```

Ожидаемо:

- все ноды видны;
- `Quorate: Yes`;
- количество votes соответствует числу голосующих узлов.

### Проверка SDN

```bash
ls /etc/pve/sdn
```

В веб-интерфейсе:

```text
Datacenter -> SDN
```

Не должно быть неприменённых или ошибочных pending changes.

### Проверка FRR и OpenFabric

```bash
systemctl status frr
vtysh -c "show openfabric neighbor"
vtysh -c "show ip route"
```

Ожидаемо:

- FRR запущен;
- OpenFabric-соседи установлены;
- router-id адреса других нод доступны.

### Проверка EVPN/BGP

```bash
vtysh -c "show bgp summary"
vtysh -c "show bgp l2vpn evpn summary"
```

Ожидаемо:

- BGP-сессии подняты;
- EVPN routes распространяются между нодами.

### Проверка ВМ

Создать две ВМ на разных физических нодах и подключить их к `worknet`.

На первой ВМ:

```bash
ping 10.10.0.102
```

На второй ВМ:

```bash
ping 10.10.0.101
```

Проверить выход наружу:

```bash
ping 8.8.8.8
curl https://example.com
```

Если DNS настроен:

```bash
ping master.cluster.local
```

### Проверка миграции ВМ

В веб-интерфейсе:

```text
VM -> Migrate
```

После миграции проверить:

```bash
ping <ip-адрес-мигрированной-вм>
```

В выбранной схеме EVPN/OpenFabric виртуальная машина должна сохранять связность после перемещения между нодами.

## Добавление новой физической ноды с учётом SDN

Краткий чек-лист:

1. Установить Proxmox VE.
2. Задать уникальный hostname.
3. Настроить management IP.
4. Настроить Corosync/fabric интерфейсы.
5. Обновить систему.
6. Установить FRR:

```bash
apt update
apt install frr frr-pythontools ifupdown2
systemctl enable --now frr
```

7. Добавить ноду в кластер:

```bash
pvecm add 192.168.10.11 --link0 192.168.20.14
```

8. Добавить ноду в OpenFabric:

```text
Datacenter -> SDN -> Fabrics -> fab0 -> Add Node
```

9. Указать router-id, например:

```text
10.255.0.4
```

10. Добавить ноду в EVPN-зону, если зона ограничена списком нод.
11. Применить SDN changes:

```text
Datacenter -> SDN -> Apply
```

12. Проверить:

```bash
pvecm status
vtysh -c "show openfabric neighbor"
vtysh -c "show bgp l2vpn evpn summary"
```

13. Перенести или скопировать шаблон ВМ на новую ноду, если используется локальное хранилище.
14. Создать тестовую ВМ на новой ноде и проверить связность в `worknet`.

## Типовые проблемы и диагностика

### Нода не добавляется в кластер

Проверить:

```bash
hostnamectl
cat /etc/hosts
ping <ip-existing-node>
ssh root@<ip-existing-node>
timedatectl
```

Частые причины:

- конфликт hostname;
- неверная запись в `/etc/hosts`;
- разные версии Proxmox;
- проблемы с DNS;
- рассинхронизация времени;
- на новой ноде уже есть ВМ с конфликтующими VMID.

### Кластер не quorate

Проверить:

```bash
pvecm status
```

Причины:

- недоступна часть нод;
- проблемы в Corosync-сети;
- сетевые задержки или потери;
- неверно настроены cluster links.

### OpenFabric не видит соседей

Проверить:

```bash
systemctl status frr
vtysh -c "show openfabric neighbor"
ip link
ip addr
```

Причины:

- выбран неправильный интерфейс fabric;
- интерфейс выключен;
- VLAN не пропущен на коммутаторе;
- router-id повторяется;
- SDN changes не применены.

### EVPN-сессии не поднимаются

Проверить:

```bash
vtysh -c "show bgp summary"
vtysh -c "show bgp l2vpn evpn summary"
```

Причины:

- EVPN controller не привязан к fabric;
- неверный ASN;
- underlay-маршруты не работают;
- FRR не запущен на одной из нод;
- firewall блокирует BGP/EVPN-трафик.

### ВМ на разных нодах не видят друг друга

Проверить:

```bash
ip addr
ip route
ping <gateway-vnet>
ping <ip-other-vm>
```

Причины:

- ВМ подключены к разным VNet;
- не применена SDN-конфигурация;
- неверный gateway;
- MTU mismatch;
- не поднят EVPN control plane.

### У ВМ нет выхода во внешнюю сеть

Проверить:

- задан ли gateway в subnet;
- включён ли SNAT в EVPN-зоне или subnet;
- настроены ли exit nodes;
- доступен ли внешний gateway с exit node;
- не блокирует ли firewall forwarding.

## Эксплуатационные рекомендации

- Не менять hostname ноды после добавления в кластер.
- Для Corosync использовать стабильную сеть с низкой задержкой.
- Не смешивать управление, storage, Corosync и VM-трафик без необходимости.
- Хранить адресный план в репозитории.
- Для VXLAN/EVPN заранее продумать MTU.
- После каждого изменения SDN нажимать `Apply`.
- Перед массовым развёртыванием ВМ проверять схему на двух тестовых ВМ на разных нодах.
- Для шаблонов использовать cloud-init и QEMU guest agent.
- Для больших объёмов результатов моделирования использовать NAS или другое сетевое хранилище.
- Для повторяемого развёртывания ВМ использовать Terraform.

## Полезные команды

Кластер:

```bash
pvecm status
pvecm nodes
```

Сеть:

```bash
ip addr
ip route
bridge link
```

Proxmox VM:

```bash
qm list
qm config <vmid>
qm start <vmid>
qm stop <vmid>
qm migrate <vmid> <target-node>
```

FRR:

```bash
vtysh -c "show running-config"
vtysh -c "show openfabric neighbor"
vtysh -c "show bgp summary"
vtysh -c "show bgp l2vpn evpn summary"
```

Службы:

```bash
systemctl status pve-cluster
systemctl status corosync
systemctl status frr
```

## Источники

- Proxmox VE Cluster Manager: <https://pve.proxmox.com/pve-docs/chapter-pvecm.html>
- Proxmox VE SDN: <https://pve.proxmox.com/pve-docs/chapter-pvesdn.html>
