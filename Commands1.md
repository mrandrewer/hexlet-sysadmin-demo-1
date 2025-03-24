# Модуль 1 Настройка сетевой инфраструктуры 

## Заданиe 1 Произведите базовую настройку устройств
## Заданиe 2 Настройка ISP
## Заданиe 4 Настройте на интерфейсе HQ-RTR в сторону офиса HQ виртуальный коммутатор
## Заданиe 8 Настройка динамической трансляции адресов.

Настройка ISP
```sh
# Установка hostname
hostnamectl set-hostname ISP.au-team.irpo

# Включение пересылки пакетов
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf

# Обновление индекса пакетов
apt-get update -y

# Включение NAT
apt-get install iptables -y
iptables -t nat -A POSTROUTING -j MASQUERADE -o ens18 
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

# Установка и настройка NetworkManager
apt-get install NetworkManager-tui -y
systemctl enable --now NetworkManager
nmcli con mod "Wired connection 1" ipv4.method "manual" ipv4.addresses "172.16.4.1/28"
nmcli con mod "Wired connection 2" ipv4.method "manual" ipv4.addresses "172.16.5.1/28"

# Вывод hostname и сетевых интерфейсов
hostname
ip -br a
iptables-save
```

Настройка HQ-RTR
```sh
# Установка hostname
hostnamectl set-hostname HQ-RTR.au-team.irpo

# Включение пересылки пакетов
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf

# Настройка адаптера к ISP
sed -i 's/BOOTPROTO=dhcp4/BOOTPROTO=static/g' /etc/net/ifaces/ens18/options
sed -i 's/BOOTPROTO=dhcp/BOOTPROTO=static/g' /etc/net/ifaces/ens18/options
echo "172.16.4.2/28" > /etc/net/ifaces/ens18/ipv4address
echo "default via 172.16.4.1" > /etc/net/ifaces/ens18/ipv4route
echo "nameserver 8.8.8.8" > /etc/net/ifaces/ens18/resolv.conf
systemctl restart network

# Обновление индекса пакетов
apt-get update -y

# Установка и настройка NetworkManager
apt-get install NetworkManager-tui -y
systemctl enable --now NetworkManager

# Включение NAT
apt-get install iptables -y
iptables -t nat -A POSTROUTING -j MASQUERADE -o ens18 
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

# Включение виртуального коммутатора
apt-get install -y openvswitch
systemctl enable --now openvswitch
ovs-vsctl add-br HQ-SW

ovs-vsctl add-port HQ-SW ens19
mkdir /etc/net/ifaces/ens19
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens19/options

ovs-vsctl add-port HQ-SW vlan100 tag=100 -- set interface vlan100 type=internal
mkdir /etc/net/ifaces/vlan100
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/vlan100/options
echo "192.168.10.1/26" > /etc/net/ifaces/vlan100/ipv4address

ovs-vsctl add-port HQ-SW vlan200 tag=200 -- set interface vlan200 type=internal
mkdir /etc/net/ifaces/vlan200
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/vlan200/options
echo "192.168.10.97/28" > /etc/net/ifaces/vlan200/ipv4address

ovs-vsctl add-port HQ-SW vlan999 tag=999 -- set interface vlan999 type=internal
mkdir /etc/net/ifaces/vlan999
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/vlan999/options
echo "192.168.10.113/29" > /etc/net/ifaces/vlan999/ipv4address
systemctl restart network

# Вывод hostname и сетевых интерфейсов
hostname
ip -br a
iptables-save
```

Настройка HQ-SRV
```sh
# Установка hostname
hostnamectl set-hostname HQ-SRV.au-team.irpo

# Настройка адаптера к HQ-RTR
sed -i 's/BOOTPROTO=dhcp4/BOOTPROTO=static/g' /etc/net/ifaces/ens18/options
sed -i 's/BOOTPROTO=dhcp/BOOTPROTO=static/g' /etc/net/ifaces/ens18/options
echo "192.168.10.2/26" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.10.1" > /etc/net/ifaces/ens18/ipv4route
echo "nameserver 8.8.8.8" > /etc/net/ifaces/ens18/resolv.conf
systemctl restart network

# Обновление индекса пакетов
apt-get update -y

# Вывод hostname и сетевых интерфейсов
hostname
ip -br a
```

Настройка HQ-CLI
```sh
# Установка hostname
hostnamectl set-hostname HQ-CLI.au-team.irpo

# Настройка адаптера к HQ-RTR
# Вручную через собственный аль-линукса центр управления системой

# Обновление индекса пакетов
apt-get update -y

# Вывод hostname и сетевых интерфейсов
hostname
ip -br a
```


Настройка BR-RTR
```sh
# Установка hostname
hostnamectl set-hostname BR-RTR.au-team.irpo

# Включение пересылки пакетов
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf

# Настройка адаптера к ISP
sed -i 's/BOOTPROTO=dhcp4/BOOTPROTO=static/g' /etc/net/ifaces/ens18/options
sed -i 's/BOOTPROTO=dhcp/BOOTPROTO=static/g' /etc/net/ifaces/ens18/options
echo "172.16.5.2/28" > /etc/net/ifaces/ens18/ipv4address
echo "default via 172.16.5.1" > /etc/net/ifaces/ens18/ipv4route
echo "nameserver 8.8.8.8" > /etc/net/ifaces/ens18/resolv.conf
systemctl restart network

# Обновление индекса пакетов
apt-get update -y

# Включение NAT
apt-get install iptables -y
iptables -t nat -A POSTROUTING -j MASQUERADE -o ens18 
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

# Настройка сети в сторону BR-SRV
# Установка и настройка NetworkManager
apt-get install NetworkManager-tui -y
systemctl enable --now NetworkManager
#nmcli con mod "Wired connection 2" ipv4.method "manual" ipv4.addresses "192.168.1.1/27"

mkdir /etc/net/ifaces/ens19
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens19/options
echo "192.168.10.65/27" > /etc/net/ifaces/ens19/ipv4address
systemctl restart network

# Вывод hostname и сетевых интерфейсов
hostname
ip -br a
iptables-save
```

Настройка BR-SRV
```sh
# Установка hostname
hostnamectl set-hostname BR-SRV.au-team.irpo

# Настройка адаптера к HQ-RTR
sed -i 's/BOOTPROTO=dhcp4/BOOTPROTO=static/g' /etc/net/ifaces/ens18/options
sed -i 's/BOOTPROTO=dhcp/BOOTPROTO=static/g' /etc/net/ifaces/ens18/options
echo "192.168.10.66/27" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.10.65" > /etc/net/ifaces/ens18/ipv4route
echo "nameserver 8.8.8.8" > /etc/net/ifaces/ens18/resolv.conf
systemctl restart network

# Обновление индекса пакетов
apt-get update -y

# Вывод hostname и сетевых интерфейсов
hostname
ip -br a
```

## Задание 3 Создание локальных учетных записей

Создание пользователя на HQ-SRV и BR-SRV
```sh
# Создание пользователя
groupadd sshuser -g1010
useradd sshuser -u1010 -g1010 -m -s/bin/bash
passwd sshuser

grep sshuser /etc/passwd
# Настройка прав суперпользователя
usermod -aG wheel sshuser
echo 'WHEEL_USERS ALL = (ALL:ALL) NOPASSWD: ALL' >> /etc/sudoers
```

Создание пользователя на HQ-RTR и BR-RTR
```sh
# Создание пользователя
useradd net_admin -m -s/bin/bash -Gwheel
passwd net_admin

grep net_admin /etc/passwd

# Настройка прав суперпользователя
echo 'WHEEL_USERS ALL = (ALL:ALL) NOPASSWD: ALL' >> /etc/sudoers
grep net_admin /etc/group
```

## Задание 5 Настройка безопасного удаленного доступа на серверах HQ-SRV и BR-SRV

Настройка удаленного доступа на HQ-SRV и BR-SRV
```sh
echo "Port 2024" >> /etc/openssh/sshd_config
echo "AllowUsers sshuser" >> /etc/openssh/sshd_config
echo "MaxAuthTries 2" >> /etc/openssh/sshd_config
echo "Banner /etc/openssh/banner" >> /etc/openssh/sshd_config
echo "**************************" >> /etc/openssh/banner
echo "* Authorized access only *" >> /etc/openssh/banner
echo "**************************" >> /etc/openssh/banner
systemctl restart sshd
```

## Задание 6 Между офисами HQ и BR необходимо сконфигурировать ip туннель

Настройка туннеля на HQ-RTR
```sh
apt-get install NetworkManager-tui -y
systemctl enable --now NetworkManager
nmcli connection add type ip-tunnel ip-tunnel.mode gre ip-tunnel.parent ens18 con-name gre1 ifname gre1 remote 172.16.5.2 local 172.16.4.2
nmcli connection modify gre1 ipv4.addresses '10.0.1.1/30'
nmcli connection modify gre1 ipv4.method manual
nmcli connection modify gre1 ip-tunnel.ttl 64

#nmcli connection up gre1

systemctl restart network
systemctl restart NetworkManager
```

Настройка туннеля на BR-RTR
```sh
apt-get install NetworkManager-tui -y
systemctl enable --now NetworkManager
nmcli connection add type ip-tunnel ip-tunnel.mode gre ip-tunnel.parent ens18 con-name gre1 ifname gre1 remote 172.16.4.2 local 172.16.5.2
nmcli connection modify gre1 ipv4.addresses '10.0.1.2/30'
nmcli connection modify gre1 ipv4.method manual
nmcli connection modify gre1 ip-tunnel.ttl 64
#nmcli connection up gre1

systemctl restart network
systemctl restart NetworkManager
```

## Задание 7 Обеспечьте динамическую маршрутизацию

Установка frr
```sh
apt-get install frr -y
sed -i 's/ospfd=no/ospfd=yes/g' /etc/frr/daemons
systemctl enable --now frr
```

Настройка ospf HQ-RTR через vtysh
```sh
vtysh
conf t
ip forwarding
router ospf
passive-interface default
network 10.0.1.0/30 area 0
network 192.168.10.0/26 area 0
network 192.168.10.96/28 area 0
network 192.168.10.112/29 area 0
ex
int gre1
no ip ospf passive
ip ospf authentication
ip ospf authentication-key P@ssw0rd
ex
ex
wr
ex

cat /etc/frr/frr.conf
systemctl restart frr
```

Настройка ospf BR-RTR через vtysh
```sh
vtysh
conf t
ip forwarding
router ospf
passive-interface default
network 10.0.1.0/30 area 0
network 192.168.10.64/27 area 0
ex
int gre1
no ip ospf passive
ip ospf authentication
ip ospf authentication-key P@ssw0rd
ex
ex
wr
ex

cat /etc/frr/frr.conf
systemctl restart frr

vtysh
show ip ospf interface gre1
ex
```


## Задание 9 Настройка протокола динамической конфигурации хостов

Настройка dhcp server на HQ-RTR
```sh
apt-get install -y dhcp-server
sed -i 's/DHCPDARGS=/DHCPDARGS=vlan200/g' /etc/sysconfig/dhcpd
cp /etc/dhcp/dhcpd.conf.sample /etc/dhcp/dhcpd.conf
vim /etc/dhcp/dhcpd.conf
```

Пример содержимого файла
```
ddns-update-style interim;

subnet 192.168.10.96 netmask 255.255.255.240 {
        option routers                  192.168.10.97;
        option subnet-mask              255.255.255.240;

        option nis-domain               "au-team.irpo";
        option domain-name              "au-team.irpo";
        option domain-name-servers      192.168.10.2;

        range dynamic-bootp 192.168.10.99 192.168.10.110;
        default-lease-time 21600;
        max-lease-time 43200;
}

host hq-cli {
        hardware ethernet BC:24:11:A0:14:E5;
        fixed-address 192.168.10.98;
}
```
```sh
systemctl enable --now dhcpd
```

После этого на клиенте переключить настройку на DHCP

## Задание 10 Настройка DNS для офисов HQ и BR

Настройка dns server на HQ-RTR

Устанавливаем сервер
```sh
apt-get install -y bind
```

Настраиваем bind в файле /var/lib/bind/etc/options.conf
```sh
vim /var/lib/bind/etc/options.conf
```
Опции, которые нужно прописать:
```
listen-on { any; };
forwarders { 8.8.8.8; };
allow-query { any; };
```

Настраиваем зоны в файле /var/lib/bind/etc/rfc1912.conf
```sh
echo "zone \"au-team.irpo\" {" >> /var/lib/bind/etc/rfc1912.conf
echo "  type master;" >> /var/lib/bind/etc/rfc1912.conf
echo "  file \"au-team\";" >> /var/lib/bind/etc/rfc1912.conf
echo "};" >> /var/lib/bind/etc/rfc1912.conf
echo "" >> /var/lib/bind/etc/rfc1912.conf

echo "zone \"10.168.192.in-addr.arpa\" {" >> /var/lib/bind/etc/rfc1912.conf
echo "  type master;" >> /var/lib/bind/etc/rfc1912.conf
echo "  file \"local\";" >> /var/lib/bind/etc/rfc1912.conf
echo "};" >> /var/lib/bind/etc/rfc1912.conf
echo "" >> /var/lib/bind/etc/rfc1912.conf

echo "zone \"200.168.192.in-addr.arpa\" {" >> /var/lib/bind/etc/rfc1912.conf
echo "  type master;" >> /var/lib/bind/etc/rfc1912.conf
echo "  file \"vlan200\";" >> /var/lib/bind/etc/rfc1912.conf
echo "};" >> /var/lib/bind/etc/rfc1912.conf
echo "" >> /var/lib/bind/etc/rfc1912.conf
```

Настраиваем файлы привязки 
```sh
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/au-team
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/vlan100
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/vlan200
```

Содержимое файла au-team
```
; BIND reverse data file for empty rfc1918 zone
;
; DO NOT EDIT THIS FILE - it is used for multiple zones.
; Instead, copy it and use that copy.
;
$TTL    1D
@       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
                                2025020600      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
        IN      NS      hq-srv.au-team.irpo.
        IN      A       192.168.10.2
hq-rtr  IN      A       192.168.10.1
br-rtr  IN      A       192.168.10.65
hq-srv  IN      A       192.168.10.2
hq-cli  IN      A       192.168.10.98
br-srv  IN      A       192.168.10.66

moodle  IN      CNAME   hq-rtr.
wiki    IN      CNAME   hq-rtr.
```

Содержимое файла vlan100
```
$TTL    1D
@       IN      SOA     10.168.192.in-addr.arpa. root.10.168.192.in-addr.arpa. (
                                2025020600      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
        IN      NS      10.168.192.in-addr.arpa.
        IN      A       192.168.10.2
1       IN      PTR     hq-rtr.au-team.irpo.
2       IN      PTR     hq-srv.au-team.irpo.
65      IN      PTR     br-rtr.au-team.irpo.
66      IN      PTR     br-srv.au-team.irpo.
97      IN      PTR     hq-rtr.au-team.irpo.
98      IN      PTR     hq-cli.au-team.irpo.
113     IN      PTR     hq-rtr.au-team.irpo.

```

Содержимое файла vlan200
```
$TTL    1D
@       IN      SOA     200.168.192.in-addr.arpa. root.200.168.192.in-addr.arpa. (
                                2025020600      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
        IN      NS      200.168.192.in-addr.arpa.
        IN      A       192.168.200.2
2       IN      PTR     hq-cli.au-team.irpo.
1       IN      PTR     hq-rtr.au-team.irpo.
```

Настраиваем rndc
```sh
rndc-confgen > /var/lib/bind/etc/rndc.key
sed -i '6,$d' /var/lib/bind/etc/rndc.key
chgrp -R named /var/lib/bind/etc/zone/
named-checkconf -z
systemctl enable --now bind
```

Прописываем на всех машинах resolv.conf
```sh
echo "nameserver 192.168.10.2" > /etc/net/ifaces/ens18/resolv.conf
echo "nameserver 8.8.8.8" >> /etc/net/ifaces/ens18/resolv.conf
echo "domain au-team.irpo" >> /etc/net/ifaces/ens18/resolv.conf
systemctl restart network
```


## Задание 11 Настройка часового пояса

```sh
apt-get install tzdata
timedatectl set-timezone Europe/Moscow
timedatectl
```