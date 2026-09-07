# vodko.github.io
Ниже представлено **полное правильное решение демонстрационного экзамена по специальности 09.02.06 «Сетевое и системное администрирование» (КОД 09.02.06-1-2025)** с первого по третий модуль включительно.

Решение разработано специально для используемых операционных систем: **EcoRouterOS** (маршрутизаторы HQ-RTR, BR-RTR) и **ОС «Альт»** (Альт Сервер / Альт Рабочая Станция / Linux JeOS).

---

# РАЗДЕЛ 1. РАСЧЕТ IP-АДРЕСАЦИИ (RFC 1918) И ИТОГОВАЯ ТАБЛИЦА

Согласно требованиям RFC 1918 и условиям задания по вместимости хостов:

1. **Локальная сеть HQ-SRV (VLAN 100)**: не более 64 адресов $\rightarrow$ маска `/26` (64 адреса).
* Сеть: `192.168.100.0/26`. Шлюз (HQ-RTR): `192.168.100.62`, Сервер HQ-SRV: `192.168.100.1`.


2. **Локальная сеть HQ-CLI (VLAN 200)**: не более 16 адресов $\rightarrow$ маска `/28` (16 адресов).
* Сеть: `192.168.100.64/28`. Шлюз (HQ-RTR): `192.168.100.78`, Динамические адреса (DHCP): `192.168.100.65`–`192.168.100.77`.


3. **Сеть управления HQ (VLAN 999)**: не более 8 адресов $\rightarrow$ маска `/29` (8 адресов).
* Сеть: `192.168.100.80/29`. Адрес интерфейса управления HQ-RTR: `192.168.100.86`.


4. **Локальная сеть BR-SRV**: не более 32 адресов $\rightarrow$ маска `/27` (32 адреса).
* Сеть: `192.168.200.0/27`. Шлюз (BR-RTR): `192.168.200.30`, Сервер BR-SRV: `192.168.200.1`.


5. **Межофисный IP-туннель (GRE)**: сеть `/30` (4 адреса).
* Сеть: `10.10.10.0/30`. HQ-RTR: `10.10.10.1`, BR-RTR: `10.10.10.2`.


6. **Провайдерские сети ISP**:
* В сторону HQ-RTR: `172.16.4.0/28` (ISP: `172.16.4.1`, HQ-RTR WAN: `172.16.4.14`).
* В сторону BR-RTR: `172.16.5.0/28` (ISP: `172.16.5.1`, BR-RTR WAN: `172.16.5.14`).



### Таблица IP-адресации для отчета

| Имя устройства | Интерфейс / VLAN | IP-адрес / Маска | Шлюз по умолчанию |
| --- | --- | --- | --- |
| **ISP** | ens19 (WAN)<br>

<br>ens21 (HQ)<br>

<br>ens20 (BR) | DHCP<br>

<br>`172.16.4.1/28`<br>

<br>`172.16.5.1/28` | Динамический (от провайдера)<br>

<br>—<br>

<br>— |
| **HQ-RTR** | ISP (WAN)<br>

<br>vl100 (HQ-SRV)<br>

<br>vl200 (HQ-CLI)<br>

<br>vl999 (MGMT)<br>

<br>tunnel.0 (GRE) | `172.16.4.14/28`<br>

<br>`192.168.100.62/26`<br>

<br>`192.168.100.78/28`<br>

<br>`192.168.100.86/29`<br>

<br>`10.10.10.1/30` | `172.16.4.1`<br>

<br>—<br>

<br>—<br>

<br>—<br>

<br>— |
| **BR-RTR** | ISP (WAN)<br>

<br>LAN (BR-SRV)<br>

<br>tunnel.0 (GRE) | `172.16.5.14/28`<br>

<br>`192.168.200.30/27`<br>

<br>`10.10.10.2/30` | `172.16.5.1`<br>

<br>—<br>

<br>— |
| **HQ-SRV** | ens19 (VLAN 100) | `192.168.100.1/26` | `192.168.100.62` |
| **BR-SRV** | ens19 | `192.168.200.1/27` | `192.168.200.30` |
| **HQ-CLI** | ens19 (VLAN 200) | DHCP (`192.168.100.65/28`) | `192.168.100.78` |

---

# МОДУЛЬ 1. НАСТРОЙКА СЕТЕВОЙ ИНФРАСТРУКТУРЫ

## 1.1. Задание системных имен и временных зон

### На узлах с ОС «Альт» (HQ-SRV, BR-SRV, HQ-CLI) и ISP:

```bash
# На узле ISP:
hostnamectl set-hostname ISP; exec bash
timedatectl set-timezone Europe/Moscow

# На узле HQ-SRV:
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
timedatectl set-timezone Europe/Moscow

# На узле BR-SRV:
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
timedatectl set-timezone Europe/Moscow

# На узле HQ-CLI:
hostnamectl set-hostname hq-cli.au-team.irpo; exec bash
timedatectl set-timezone Europe/Moscow

```

### На маршрутизаторах EcoRouter (HQ-RTR и BR-RTR):

```text
! На HQ-RTR:
enable
configure terminal
hostname hq-rtr
ip domain-name au-team.irpo
clock timezone MSK utc+3
write memory

! На BR-RTR:
enable
configure terminal
hostname br-rtr
ip domain-name au-team.irpo
clock timezone MSK utc+3
write memory

```

---

## 1.2. Настройка интернет-провайдера (ISP)

На сервере ISP настраивается прием адреса по DHCP на внешнем интерфейсе (`ens19`), статические адреса на внутренних интерфейсах (`ens21` к HQ, `ens20` к BR), включается IP forwarding и динамическая трансляция адресов (Masquerade).

1. Редактируем внешнее подключение `/etc/net/ifaces/ens19/options`:
```ini
TYPE=eth
BOOTPROTO=dhcp

```


2. Настройка интерфейса в сторону HQ (`ens21`):
* `/etc/net/ifaces/ens21/options`:
```ini
TYPE=eth
BOOTPROTO=static

```


* `/etc/net/ifaces/ens21/ipv4address`:
```text
172.16.4.1/28

```




3. Настройка интерфейса в сторону BR (`ens20`):
* `/etc/net/ifaces/ens20/options`:
```ini
TYPE=eth
BOOTPROTO=static

```


* `/etc/net/ifaces/ens20/ipv4address`:
```text
172.16.5.1/28

```




4. Включение маршрутизации пакетов в `/etc/net/sysctl.conf`:
```ini
net.ipv4.ip_forward = 1

```


5. Настройка динамического NAT (Masquerade) и перезапуск служб:
```bash
systemctl restart network
apt-get update && apt-get install -y iptables
iptables -t nat -A POSTROUTING -o ens19 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

```



---

## 1.3. Создание локальных учетных записей

### На серверах HQ-SRV и BR-SRV (ОС «Альт»):

Создаем пользователя `sshuser` с UID `1010`, паролем `P@ssword` и возможностью вызова `sudo` без пароля:

```bash
useradd sshuser -u 1010
echo "sshuser:P@ssword" | chpasswd
gpasswd -a sshuser wheel

# Настройка беспарольного sudo для группы wheel
echo "%wheel ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/10-wheel
chmod 440 /etc/sudoers.d/10-wheel

```

### На маршрутизаторах HQ-RTR и BR-RTR (EcoRouter):

Создаем пользователя `net_admin` с максимальными привилегиями (`admin`) и паролем `P@ssword`:

```text
configure terminal
username net_admin
 password P@ssword
 role admin
exit
write memory

```

---

## 1.4. Настройка L2/L3 коммутации и VLAN

### Вариант 1: Через EcoRouter (HQ-RTR)

Если коммутатор виртуализирован средствами EcoRouter, на порту `te1` создаются сервисные инстансы (Service Instances) с удалением тега VLAN (`rewrite pop 1`) и привязкой к L3-интерфейсам:

```text
! HQ-RTR:
configure terminal

! L3 интерфейсы
interface vl100
 ip address 192.168.100.62/26
exit
interface vl200
 ip address 192.168.100.78/28
exit
interface vl999
 ip address 192.168.100.86/29
exit

! Привязка VLAN к физическому порту te1
port te1
 service-instance te1/vl100
  encapsulation dot1q 100
  rewrite pop 1
  connect ip interface vl100
 exit
 service-instance te1/vl200
  encapsulation dot1q 200
  rewrite pop 1
  connect ip interface vl200
 exit
 service-instance te1/vl999
  encapsulation dot1q 999
  rewrite pop 1
  connect ip interface vl999
 exit
exit
write memory

```

### Вариант 2: Если HQ-SW — отдельная виртуальная машина на Open vSwitch (OVS)

На виртуальной машине HQ-SW (ens3 — транк к HQ-RTR, ens4 — access VLAN 100 к HQ-SRV, ens5 — access VLAN 200 к HQ-CLI):

```bash
ovs-vsctl add-br SW
ovs-vsctl add-port SW ens3 trunk=100,200,999
ovs-vsctl add-port SW ens4 tag=100
ovs-vsctl add-port SW ens5 tag=200

```

---

## 1.5. Настройка базовой сетевой адресации на серверах HQ-SRV и BR-SRV

### HQ-SRV (ОС «Альт»):

* Файл `/etc/net/ifaces/ens19/options`:
```ini
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no

```


* Файл `/etc/net/ifaces/ens19/ipv4address`:
```text
192.168.100.1/26

```


* Файл `/etc/net/ifaces/ens19/ipv4route`:
```text
default via 192.168.100.62

```


* Применение: `systemctl restart network`

### BR-SRV (ОС «Альт»):

* Файл `/etc/net/ifaces/ens19/options`:
```ini
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no

```


* Файл `/etc/net/ifaces/ens19/ipv4address`:
```text
192.168.200.1/27

```


* Файл `/etc/net/ifaces/ens19/ipv4route`:
```text
default via 192.168.200.30

```


* Применение: `systemctl restart network`

---

## 1.6. Безопасный удаленный доступ (SSH) на серверах HQ-SRV и BR-SRV

Выполняется на серверах **HQ-SRV** и **BR-SRV**:

1. Редактируем баннер в `/etc/openssh/banner`:
```text
Authorized access only.

```


2. Изменяем конфигурацию службы `/etc/openssh/sshd_config`:
```ini
Port 2024
AllowUsers sshuser
MaxAuthTries 2
PasswordAuthentication yes
Banner /etc/openssh/banner

```


3. Перезапускаем SSH-демон:
```bash
systemctl restart sshd

```



---

## 1.7. Настройка GRE IP-туннеля между HQ-RTR и BR-RTR

### WAN-интерфейсы на роутерах:

**HQ-RTR**:

```text
configure terminal
interface ISP
 ip address 172.16.4.14/28
exit
port te0
 service-instance te0/ISP
  encapsulation untagged
  connect ip interface ISP
 exit
exit
ip route 0.0.0.0/0 172.16.4.1
write memory

```

**BR-RTR**:

```text
configure terminal
interface ISP
 ip address 172.16.5.14/28
exit
port te0
 service-instance te0/ISP
  encapsulation untagged
  connect ip interface ISP
 exit
exit
interface LAN
 ip address 192.168.200.30/27
exit
port te1
 service-instance te1/LAN
  encapsulation untagged
  connect ip interface LAN
 exit
exit
ip route 0.0.0.0/0 172.16.5.1
write memory

```

### Настройка туннеля GRE:

**HQ-RTR**:

```text
configure terminal
interface tunnel.0
 ip address 10.10.10.1/30
 ip tunnel 172.16.4.14 172.16.5.14 mode gre
exit
write memory

```

**BR-RTR**:

```text
configure terminal
interface tunnel.0
 ip address 10.10.10.2/30
 ip tunnel 172.16.5.14 172.16.4.14 mode gre
exit
write memory

```

---

## 1.8. Динамическая маршрутизация (OSPF) с защитой

Настройка Link-State протокола OSPF на маршрутизаторах HQ-RTR и BR-RTR с блокировкой лишних интерфейсов (`passive-interface default`) и парольной защитой туннельного интерфейса:

### HQ-RTR:

```text
configure terminal
router ospf 1
 passive-interface default
 no passive-interface tunnel.0
 network 10.10.10.0/30 area 0
 network 192.168.100.0/26 area 0
 network 192.168.100.64/28 area 0
 network 192.168.100.80/29 area 0
 area 0 authentication
exit
interface tunnel.0
 ip ospf authentication-key P@ssword
exit
write memory

```

### BR-RTR:

```text
configure terminal
router ospf 1
 passive-interface default
 no passive-interface tunnel.0
 network 10.10.10.0/30 area 0
 network 192.168.200.0/27 area 0
 area 0 authentication
exit
interface tunnel.0
 ip ospf authentication-key P@ssword
exit
write memory

```

---

## 1.9. Настройка динамической трансляции адресов (NAT / PAT Overload)

Обеспечение выхода локальных сетей филиалов в сеть Интернет.

### HQ-RTR:

```text
configure terminal
interface ISP
 ip nat outside
exit
interface vl100
 ip nat inside
exit
interface vl200
 ip nat inside
exit
interface vl999
 ip nat inside
exit

ip nat pool HQ_POOL 192.168.100.1-192.168.100.254
ip nat source dynamic inside-to-outside pool HQ_POOL overload interface ISP
write memory

```

### BR-RTR:

```text
configure terminal
interface ISP
 ip nat outside
exit
interface LAN
 ip nat inside
exit

ip nat pool BR_POOL 192.168.200.1-192.168.200.254
ip nat source dynamic inside-to-outside pool BR_POOL overload interface ISP
write memory

```

---

## 1.10. Настройка DHCP-сервера на HQ-RTR для подсети HQ-CLI

HQ-RTR выступает в качестве DHCP-сервера для клиенского сегмента VLAN 200 (`192.168.100.64/28`). Исключается IP маршрутизатора (`192.168.100.78`), передаются адрес DNS-сервера (`192.168.100.1`) и суффикс домена (`au-team.irpo`).

```text
configure terminal
dhcp-server 1
 pool HQ-CLI-POOL 1
  mask 28
  gateway 192.168.100.78
  dns 192.168.100.1
  domain-name au-team.irpo
  range 192.168.100.65 192.168.100.77
 exit
exit

interface vl200
 dhcp-server 1
exit
write memory

```

---

## 1.11. Настройка DNS-сервера (BIND9) на HQ-SRV

1. Установка BIND на HQ-SRV:
```bash
apt-get update && apt-get install -y bind bind-utils

```


2. Настройка глобальных параметров `/etc/bind/options.conf`:
```text
options {
    directory "/var/lib/bind";
    listen-on port 53 { any; };
    allow-query { any; };
    forwarders { 77.88.8.8; 8.8.8.8; };
    dnssec-validation auto;
};

```


3. Добавление зон в `/etc/bind/named.conf`:
```text
zone "au-team.irpo" {
    type master;
    file "zone/au-team.irpo.zone";
};

zone "100.168.192.in-addr.arpa" {
    type master;
    file "zone/192.168.100.rev";
};

```


4. Файл прямой зоны `/var/lib/bind/zone/au-team.irpo.zone`:
```text
$TTL 86400
@   IN  SOA hq-srv.au-team.irpo. admin.au-team.irpo. (
        2025040701 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS      hq-srv.au-team.irpo.

hq-srv  IN  A       192.168.100.1
hq-rtr  IN  A       192.168.100.62
hq-cli  IN  A       192.168.100.65
br-rtr  IN  A       172.16.5.14
br-srv  IN  A       192.168.200.1

moodle  IN  CNAME   hq-rtr.au-team.irpo.
wiki    IN  CNAME   hq-rtr.au-team.irpo.

```


5. Файл обратной зоны `/var/lib/bind/zone/192.168.100.rev`:
```text
$TTL 86400
@   IN  SOA hq-srv.au-team.irpo. admin.au-team.irpo. (
        2025040701 3600 1800 604800 86400 )
    IN  NS      hq-srv.au-team.irpo.

1   IN  PTR     hq-srv.au-team.irpo.
62  IN  PTR     hq-rtr.au-team.irpo.
65  IN  PTR     hq-cli.au-team.irpo.

```


6. Запуск и добавление в автозапуск:
```bash
chown -R root:named /var/lib/bind/zone/
systemctl enable --now bind

```



---

# МОДУЛЬ 2. ОРГАНИЗАЦИЯ СЕТЕВОГО АДМИНИСТРИРОВАНИЯ ОПЕРАЦИОННЫХ СИСТЕМ

## 2.1. Настройка файлового хранилища (NFS)

Организация сетевого файлового ресурса на **HQ-SRV** для монтирования на **BR-SRV**.

### На HQ-SRV (NFS Сервер):

```bash
apt-get install -y nfs-server
mkdir -p /srv/share
chown -R nobody:nobody /srv/share
chmod 777 /srv/share

# Добавление экспорта в /etc/exports
echo "/srv/share 192.168.0.0/16(rw,sync,no_subtree_check)" >> /etc/exports

systemctl enable --now nfs-server
exportfs -a

```

### На BR-SRV (NFS Клиент):

```bash
apt-get install -y nfs-clients
mkdir -p /mnt/hq-share

# Добавление в /etc/fstab для автомонтирования
echo "192.168.100.1:/srv/share /mnt/hq-share nfs defaults 0 0" >> /etc/fstab
mount -a

```

---

## 2.2. Настройка службы сетевого времени Chrony

### На HQ-SRV (NTP Сервер):

1. Редактируем `/etc/chrony.conf`:
```ini
pool pool.ntp.org iburst
allow 192.168.0.0/16
local stratum 10

```


2. Перезапуск службы:
```bash
systemctl enable --now chronyd

```



### На BR-SRV (NTP Клиент):

1. Редактируем `/etc/chrony.conf`:
```ini
server 192.168.100.1 iburst

```


2. Перезапуск службы:
```bash
systemctl enable --now chronyd
chronyc sources

```



---

## 2.3. Настройка автоматизации Ansible

Управляющим узлом выступает **HQ-SRV**.

1. Установка Ansible на HQ-SRV:
```bash
apt-get install -y ansible

```


2. Создание файла инвентаризации `/etc/ansible/hosts`:
```ini
[servers]
br-srv ansible_host=192.168.200.1 ansible_port=2024 ansible_user=sshuser

[all:vars]
ansible_python_interpreter=/usr/bin/python3

```


3. Генерация SSH-ключей и копирование на BR-SRV:
```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
ssh-copy-id -p 2024 sshuser@192.168.200.1

```


4. Проверка работы Ansible:
```bash
ansible all -m ping

```



---

## 2.4. Развертывание приложений в Docker

Развертывание Docker на сервере **BR-SRV** или **HQ-SRV**.

1. Установка Docker в ОС «Альт»:
```bash
apt-get update && apt-get install -y docker-engine docker-cli
systemctl enable --now docker
usermod -aG docker sshuser

```


2. Развертывание тестового контейнера веб-сервера:
```bash
docker run -d --name web-app -p 8080:80 --restart always nginx:alpine

```



---

## 2.5. Настройка трансляции портов (Port Forwarding)

Перенаправление внешних запросов с порта `80` на внутренний сервис Docker (порт `8080`) на **HQ-SRV** с использованием `iptables`:

```bash
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080
iptables-save > /etc/sysconfig/iptables

```

---

## 2.6. Развертывание и настройка сервиса Moodle

Развертывание стека LAMP (Apache, MariaDB, PHP) и Moodle на **HQ-SRV**:

1. Установка пакетов:
```bash
apt-get update
apt-get install -y apache2 mariadb-server php8.1 php8.1-mbstring php8.1-xml php8.1-mysqli php8.1-gd php8.1-curl php8.1-zip php8.1-intl
systemctl enable --now httpd2 mariadb

```


2. Настройка СУБД MariaDB для Moodle:
```bash
mysql -u root -e "CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -e "CREATE USER 'moodleuser'@'localhost' IDENTIFIED BY 'P@ssword';"
mysql -u root -e "GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'localhost';"
mysql -u root -e "FLUSH PRIVILEGES;"

```


3. Размещение каталога данных Moodle:
```bash
mkdir -p /var/www/moodledata
chown -R apache2:apache2 /var/www/moodledata
chmod 777 /var/www/moodledata

```



---

## 2.7. Настройка Nginx в качестве обратного прокси-сервера (Reverse Proxy)

Настройка Nginx на **HQ-SRV** для проксирования внешних запросов `moodle.au-team.irpo` на локальный веб-сервер.

1. Установка Nginx:
```bash
apt-get install -y nginx

```


2. Создание конфигурации `/etc/nginx/sites-available/moodle.conf`:
```nginx
server {
    listen 80;
    server_name moodle.au-team.irpo;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```


3. Активация и запуск:
```bash
ln -s /etc/nginx/sites-available/moodle.conf /etc/nginx/sites-enabled/
systemctl enable --now nginx

```



---

## 2.8. Установка Яндекс Браузера на клиентскую машину (HQ-CLI)

Выполняется на рабочем месте **HQ-CLI** (ОС «Альт Рабочая Станция»):

```bash
apt-get update
apt-get install -y yandex-browser-stable

```

---

# МОДУЛЬ 3. ОБЛАЧНАЯ ИНФРАСТРУКТУРА, OPENSTACK CLI И IaC АВТОМАТИЗАЦИЯ

## 3.1. Подключение к OpenStack CLI и настройка окружения

Для работы с Кибер Инфраструктурой / OpenStack загружается файл переменных окружения `openrc.sh`:

```bash
# Файл /root/openrc.sh
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME="au-team-project"
export OS_USERNAME="admin"
export OS_PASSWORD="P@ssword"
export OS_AUTH_URL="http://10.20.0.1:5000/v3"
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2

```

Активация переменных окружения:

```bash
source /root/openrc.sh
openstack token issue

```

---

## 3.2. Управление ресурсами через OpenStack CLI

Команды пошагового создания сетей, маршрутизатора и виртуальных машин:

```bash
# 1. Создание облачной сети и подсети HQ-NET
openstack network create HQ-NET
openstack subnet create --network HQ-NET --subnet-range 192.168.100.0/26 --gateway 192.168.100.62 HQ-SUBNET

# 2. Создание облачной сети и подсети BR-NET
openstack network create BR-NET
openstack subnet create --network BR-NET --subnet-range 192.168.200.0/27 --gateway 192.168.200.30 BR-SUBNET

# 3. Создание виртуального маршрутизатора
openstack router create ROUTER-1
openstack router add subnet ROUTER-1 HQ-SUBNET
openstack router add subnet ROUTER-1 BR-SUBNET

# 4. Создание виртуальной машины HQ-SRV
openstack server create --flavor m1.small \
                        --image "ALT-Server-10" \
                        --network HQ-NET \
                        --wait hq-srv.au-team.irpo

```

---

## 3.3. Разворачивание всей инфраструктуры единым скриптом автоматизации (IaC)

Создание единого исполняемого Bash-скрипта `/root/deploy_infrastructure.sh`, полностью разворачивающего виртуальную стендовую среду:

```bash
#!/bin/bash
set -e

echo "=== 1. Активация окружения OpenStack ==="
source /root/openrc.sh

echo "=== 2. Создание виртуальных сетей и подсетей ==="
openstack network create net-hq
openstack subnet create --network net-hq --subnet-range 192.168.100.0/26 --dns-nameserver 192.168.100.1 sub-hq

openstack network create net-br
openstack subnet create --network net-br --subnet-range 192.168.200.0/27 sub-br

openstack network create net-mgmt
openstack subnet create --network net-mgmt --subnet-range 192.168.100.80/29 sub-mgmt

echo "=== 3. Создание виртуального маршрутизатора ==="
openstack router create rtr-main
openstack router add subnet rtr-main sub-hq
openstack router add subnet rtr-main sub-br
openstack router add subnet rtr-main sub-mgmt

echo "=== 4. Создание групп безопасности (Security Groups) ==="
openstack security group create secgroup-demo
openstack security group rule create --protocol icmp secgroup-demo
openstack security group rule create --protocol tcp --dst-port 2024 secgroup-demo
openstack security group rule create --protocol tcp --dst-port 80 secgroup-demo
openstack security group rule create --protocol tcp --dst-port 53 secgroup-demo
openstack security group rule create --protocol udp --dst-port 53 secgroup-demo

echo "=== 5. Развертывание виртуальных машин ==="
openstack server create --flavor m1.medium \
                        --image "ALT-Server-10" \
                        --network net-hq \
                        --security-group secgroup-demo \
                        --wait hq-srv.au-team.irpo

openstack server create --flavor m1.medium \
                        --image "ALT-Server-10" \
                        --network net-br \
                        --security-group secgroup-demo \
                        --wait br-srv.au-team.irpo

openstack server create --flavor m1.small \
                        --image "ALT-Workstation-10" \
                        --network net-hq \
                        --security-group secgroup-demo \
                        --wait hq-cli.au-team.irpo

echo "=== Инфраструктура успешно развернута! ==="
openstack server list

```

Запуск скрипта авторазвертывания:

```bash
chmod +x /root/deploy_infrastructure.sh
/root/deploy_infrastructure.sh

# РАЗДЕЛ 4. ПЛАН ТЕСТИРОВАНИЯ И ПРОВЕРКИ ИНФРАСТРУКТУРЫ

Для полной сдачи демонстрационного экзамена требуется провести интеграционное тестирование всех настроенных компонентов.

---

## 4.1. Проверка сетевой связности и маршрутизации

### Проверка статуса OSPF-соседства и таблицы маршрутизации (HQ-RTR / BR-RTR)

```text
! Проверка установления OSPF-соседства через GRE-туннель
show ip ospf neighbor

! Проверка наличия OSPF-маршрутов
show ip route ospf

```

### Проверка работы NAT и прохождения трафика в Интернет

```bash
# Проверка доступности внешних узлов с HQ-CLI / HQ-SRV / BR-SRV
ping -c 4 77.88.8.8
ping -c 4 au-team.irpo

# Проверка трансляции адресов на маршрутизаторах
# (команда выполняется на EcoRouter)
show ip nat translations

```

---

## 4.2. Проверка службы DNS (BIND9)

Выполняется с любого клиента или сервера сети (например, HQ-CLI или BR-SRV):

```bash
# Проверка разрешения прямого имени
dig @192.168.100.1 hq-srv.au-team.irpo +short
dig @192.168.100.1 moodle.au-team.irpo +short

# Проверка обратного резолва (PTR)
dig @192.168.100.1 -x 192.168.100.1 +short
dig @192.168.100.1 -x 192.168.100.62 +short

```

---

## 4.3. Проверка сервисов (SSH, NFS, Chrony, Nginx/Moodle)

### Проверка SSH-доступа по нестандартному порту (2024)

```bash
# Подключение с HQ-SRV к BR-SRV под пользователем sshuser
ssh -p 2024 sshuser@192.168.200.1 "id; hostname"

```

### Проверка монтирования NFS-ресурса на BR-SRV

```bash
# Проверка точки монтирования
df -hT /mnt/hq-share

# Проверка записи/чтения
touch /mnt/hq-share/testfile.txt && ls -l /mnt/hq-share/testfile.txt

```

### Проверка синхронизации времени Chrony

```bash
# На BR-SRV
chronyc sources -v
chronyc tracking

```

### Проверка доступности веб-сервисов

```bash
# Проверка ответа веб-сервера Moodle через обратный прокси Nginx
curl -I -H "Host: moodle.au-team.irpo" http://192.168.100.1

```

---

# РАЗДЕЛ 5. ИНФОРМАЦИОННАЯ БЕЗОПАСНОСТЬ И МЕЖСЕТЕВОЕ ЭКРАНИРОВАНИЕ

## 5.1. Настройка файрвола (Netfilter / Iptables) на серверах HQ-SRV и BR-SRV

Настройка правил фильтрации для закрытия незадействованных портов и защиты от несанкционированного доступа.

### Скрипт настройки брандмауэра для HQ-SRV (`/etc/sysconfig/iptables`):

```text
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

# Разрешаем локальный трафик (loopback)
-A INPUT -i lo -j ACCEPT

# Разрешаем установленные и связанные соединения
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Разрешаем ICMP (ping) внутри локальных сетей
-A INPUT -p icmp -s 192.168.0.0/16 -j ACCEPT
-A INPUT -p icmp -s 10.10.10.0/30 -j ACCEPT

# Разрешаем SSH (порт 2024)
-A INPUT -p tcp --dport 2024 -j ACCEPT

# Разрешаем DNS (UDP/TCP 53)
-A INPUT -p udp --dport 53 -j ACCEPT
-A INPUT -p tcp --dport 53 -j ACCEPT

# Разрешаем HTTP / HTTPS (80, 443, 8080)
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT
-A INPUT -p tcp --dport 8080 -j ACCEPT

# Разрешаем NTP (NTP Server 123/UDP)
-A INPUT -p udp --dport 123 -j ACCEPT

# Разрешаем NFS (2049, 111)
-A INPUT -p tcp --dport 2049 -j ACCEPT
-A INPUT -p udp --dport 2049 -j ACCEPT
-A INPUT -p tcp --dport 111 -j ACCEPT
-A INPUT -p udp --dport 111 -j ACCEPT

COMMIT

```

Применение и автозапуск правил:

```bash
systemctl enable --now iptables
iptables-restore < /etc/sysconfig/iptables

```

---

## 5.2. Защита SSH от подбора паролей (Fail2ban)

Выполняется на **HQ-SRV** и **BR-SRV**:

1. Установка Fail2ban:
```bash
apt-get install -y fail2ban

```


2. Настройка файла конфигурации `/etc/fail2ban/jail.d/sshd.local`:
```ini
[sshd]
enabled = true
port = 2024
filter = sshd
logpath = /var/log/messages
maxretry = 2
findtime = 600
bantime = 3600

```


3. Запуск службы:
```bash
systemctl enable --now fail2ban

```



---

# РАЗДЕЛ 6. ЦЕНТРАЛИЗОВАННЫЙ МОНИТОРИНГ И СБОР ЛОГОВ (SYSLOG)

## 6.1. Настройка сервера сбора логов (Rsyslog) на HQ-SRV

1. Включение приёма сообщений по протоколу UDP/TCP в `/etc/rsyslog.conf`:
```ini
# Раскомментировать линии приёма трафика:
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")

# Выделение логов сетевого оборудования в отдельный файл
$template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
if ($hostname != 'hq-srv') then ?RemoteLogs
& stop

```


2. Перезапуск службы:
```bash
systemctl restart rsyslog

```



---

## 6.2. Отправка системных логов с EcoRouter (HQ-RTR / BR-RTR) на HQ-SRV

```text
configure terminal
logging host 192.168.100.1
logging facility local7
logging trap informational
write memory

```

---

# РАЗДЕЛ 7. РЕЗЕРВНОЕ КОПИРОВАНИЕ И АВТОМАТИЗАЦИЯ ОБСЛУЖИВАНИЯ

## 7.1. Скрипт резервного копирования базы данных и файлов Moodle

Создание скрипта `/usr/local/bin/moodle-backup.sh` на **HQ-SRV**:

```bash
#!/bin/bash
BACKUP_DIR="/var/backups/moodle"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p ${BACKUP_DIR}

# Бэкап БД MariaDB
mysqldump -u moodleuser -p'P@ssword' moodle | gzip > ${BACKUP_DIR}/moodle_db_${DATE}.sql.gz

# Бэкап директории moodledata
tar -czf ${BACKUP_DIR}/moodle_data_${DATE}.tar.gz /var/www/moodledata

# Удаление бэкапов старше 7 дней
find ${BACKUP_DIR} -type f -mtime +7 -delete

```

Права на исполнение и добавление в задание Cron:

```bash
chmod +x /usr/local/bin/moodle-backup.sh

# Добавление в crontab (запуск каждый день в 02:00)
(crontab -l 2>/dev/null; echo "0 2 * * * /usr/local/bin/moodle-backup.sh") | crontab -

```

---

## 7.2. Скрипт автоматического бэкапа конфигурации EcoRouter через Ansible

Добавление задачи в Playbook `/etc/ansible/backup-router.yml` на **HQ-SRV**:

```yaml
---
- name: Backup EcoRouter Configurations
  hosts: routers
  gather_facts: no
  tasks:
    - name: Fetch running configuration
      ansible.builtin.raw: "show running-config"
      register: config_out

    - name: Save configuration to file
      ansible.builtin.copy:
        content: "{{ config_out.stdout }}"
        dest: "/var/backups/routers/{{ inventory_hostname }}_{{ ansible_date_time.date }.cfg"
      delegate_to: localhost

```

---

# РАЗДЕЛ 8. СВОДНАЯ СТАЦИОНАРНАЯ КАРТА УЧЕТНЫХ ДАННЫХ И СЕРВИСОВ

### Сводная таблица параметров системы для демонстрации экспертам

| Компонент / Сервис | Узел | IP-адрес / Порт | Учетная запись / Доступ |
| --- | --- | --- | --- |
| **Панель OpenStack** | Cloud Controller | `[http://10.20.0.1:5000](http://10.20.0.1:5000)` | `admin` / `P@ssword` |
| **SSH Управление ОС** | HQ-SRV, BR-SRV | Port `2024` / TCP | `sshuser` / `P@ssword` |
| **SSH Управление RTR** | HQ-RTR, BR-RTR | Port `22` / TCP | `net_admin` / `P@ssword` |
| **DNS Сервер** | HQ-SRV | `192.168.100.1` (UDP/TCP 53) | Зона: `au-team.irpo` |
| **NFS Файловое хранилище** | HQ-SRV $\rightarrow$ BR-SRV | `192.168.100.1:/srv/share` | Монтирование в `/mnt/hq-share` |
| **Веб-платформа Moodle** | HQ-SRV | `[http://moodle.au-team.irpo](http://moodle.au-team.irpo)` | Nginx Proxy (80) $\rightarrow$ Apache (8080) |
| **Docker Контейнер** | BR-SRV | `192.168.200.1:8080` | Nginx Alpine Container |
| **Chrony NTP Сервер** | HQ-SRV | `192.168.100.1:123` UDP | Stratum 10 Local |
| **Syslog Сервер** | HQ-SRV | `192.168.100.1:514` UDP | Каталог `/var/log/remote/` |

# РАЗДЕЛ 9. ТИПОВЫЕ НЕИСПРАВНОСТИ И ИХ УСТРАНЕНИЕ (TROUBLESHOOTING)

В ходе сборки и пусконаладки стенда могут возникать типовые проблемы межсетевого взаимодействия и работы сервисов.

### 9.1. Проблемы OSPF-соседства поверх GRE-туннеля

* **Симптом:** OSPF-соседство зависает в состоянии `INIT` или `EXSTART`.
* **Причина:** Несовпадение параметров MTU/MSS либо блокировка протокола IP 47 (GRE) или UDP 520 / multicast-трафика на внешних интерфейсах.
* **Решение:**
* Установите фиксированный MTU на туннельном интерфейсе EcoRouter:
```text
interface tunnel.0
 ip mtu 1400
 ip tcp adjust-mss 1360
exit

```


* Проверьте прохождение пакетов с флагом DF через провайдера:
```bash
ping -c 4 -s 1400 172.16.5.14

```





### 9.2. Ошибки разрешения имен BIND9

* **Симптом:** Служба BIND9 перезапускается с ошибкой или не отвечает на запросы к локальной зоне.
* **Причина:** Ошибки синтаксиса в зонах или неверные права доступа к файлам зон.
* **Решение:**
* Выполните синтаксическую проверку конфигурационных файлов:
```bash
named-checkconf /etc/bind/named.conf
named-checkzone au-team.irpo /var/lib/bind/zone/au-team.irpo.zone
named-checkzone 100.168.192.in-addr.arpa /var/lib/bind/zone/192.168.100.rev

```


* Восстановите права владельца:
```bash
chown -R root:named /var/lib/bind/zone/
chmod 640 /var/lib/bind/zone/*
systemctl restart bind

```





### 9.3. Недоступность NFS-ресурса на BR-SRV

* **Симптом:** Команда `mount -a` зависает с ошибкой `Connection timed out` или `Permission denied`.
* **Причина:** Блокировка портов `rpcbind` (111) и `nfs` (2049) брандмауэром или отсутствие записи сети в `/etc/exports`.
* **Решение:**
* Проверьте список экспортируемых директорий с BR-SRV:
```bash
showmount -e 192.168.100.1

```


* Проверьте работу RPC-служб:
```bash
rpcinfo -p 192.168.100.1

```


* Перечитайте таблицу экспорта на HQ-SRV:
```bash
exportfs -rv

```





### 9.4. Ошибки развертывания стек-сервиса Moodle / Docker

* **Симптом:** Nginx возвращает ошибку `502 Bad Gateway`.
* **Причина:** Веб-сервер Apache или контейнер Docker не слушают указанный порт (`127.0.0.1:8080`).
* **Решение:**
* Проверьте активные сокеты и процессы:
```bash
ss -tulpn | grep 8080

```


* Для контейнеров Docker проверьте статус и логи:
```bash
docker ps -a
docker logs web-app

```





---

# РАЗДЕЛ 10. ЧЕК-ЛИСТ ПРОВЕРКИ ГОТОВНОСТИ СТЕНДА К СДАЧЕ ЭКСПЕРТАМ

Перед приглашением экспертов для проверки выполненного модуля пройдите по контрольному списку:

| Компонент / Сервис | Проверяемое условие | Команда / Способ проверки | Статус |
| --- | --- | --- | --- |
| **Физический / L2 уровень** | Статусы интерфейсов в режиме `UP` | `show interfaces description` | [ ] |
| **IP-адресация** | Наличие правильных IP и масок на всех узлах | `ip a` / `show ip interface brief` | [ ] |
| **Маршрутизация OSPF** | Соседство в состоянии `FULL` | `show ip ospf neighbor` | [ ] |
| **GRE-туннель** | Пинг между `10.10.10.1` и `10.10.10.2` | `ping 10.10.10.2` | [ ] |
| **Трансляция NAT** | Доступ с локальных ПК в Интернет | `ping 77.88.8.8` с HQ-CLI / BR-SRV | [ ] |
| **DHCP-сервер** | Выдача адресов клиенту HQ-CLI из VLAN 200 | `ip a show dev ens19` на HQ-CLI | [ ] |
| **DNS-сервер** | Разрешение `moodle.au-team.irpo` в `192.168.100.1` | `nslookup moodle.au-team.irpo` | [ ] |
| **SSH-защита** | Доступ только пользователем `sshuser` по порту `2024` | `ssh -p 2024 sshuser@192.168.100.1` | [ ] |
| **NFS-хранилище** | Автомонтирование ресурса `/mnt/hq-share` | `df -h | grep nfs` | [ ] |
| **Chrony NTP** | Синхронизация времени со стратумом | `chronyc tracking` | [ ] |
| **Docker Контейнер** | Ответ веб-приложения на порту `8080` | `curl -I [http://127.0.0.1:8080](http://127.0.0.1:8080)` | [ ] |
| **Moodle / Nginx** | Открытие главной страницы Moodle по имени | `curl -I [http://moodle.au-team.irpo](http://moodle.au-team.irpo)` | [ ] |
| **OpenStack CLI** | Наличие токена и работоспособность CLI | `openstack server list` | [ ] |
| **IaC Скрипт** | Наличие права `+x` и корректность выполнения | `ls -l /root/deploy_infrastructure.sh` | [ ] |

---

# РАЗДЕЛ 11. ФИНАЛЬНОЕ СОХРАНЕНИЕ КОНФИГУРАЦИЙ И ОФОРМЛЕНИЕ ОТЧЕТА

Для предотвращения потери данных при перезагрузке виртуального стенда обязательно сохраните все настройки.

### 1. Сохранение конфигурации EcoRouter (HQ-RTR, BR-RTR):

```text
write memory

```

### 2. Сохранение конфигурации сети и правил брандмауэра на серверах ОС «Альт»:

```bash
# Сохранение правил iptables
iptables-save > /etc/sysconfig/iptables

# Включение автозапуска всех настроенных служб
systemctl enable --now network bind nfs-server chronyd docker nginx httpd2 mariadb fail2ban iptables

```

### 3. Оформление отчета участника:

1. Создайте текстовый документ `report_090206.txt` или `.pdf` в соответствии с формой, выданной главным экспертом.
2. Внесите в документ:
* Итоговую таблицу IP-адресации из Раздела 1.
* Вывод команды `openstack server list` после исполнения IaC-скрипта.
* Вывод команд `show ip route` и `show ip ospf neighbor` с маршрутизаторов.


3. Сохраните копию отчета в каталоге `/root/` на управляющем сервере HQ-SRV.
