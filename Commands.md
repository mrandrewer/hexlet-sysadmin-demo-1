# Модуль 1 Настройка сетевой инфраструктуры 

## Задания 1, 2, 4 и 8

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
echo "192.168.100.1/26" > /etc/net/ifaces/vlan100/ipv4address

ovs-vsctl add-port HQ-SW vlan200 tag=200 -- set interface vlan200 type=internal
mkdir /etc/net/ifaces/vlan200
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/vlan200/options
echo "192.168.200.1/28" > /etc/net/ifaces/vlan200/ipv4address

ovs-vsctl add-port HQ-SW vlan999 tag=999 -- set interface vlan999 type=internal
mkdir /etc/net/ifaces/vlan999
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/vlan999/options
echo "192.168.99.1/29" > /etc/net/ifaces/vlan999/ipv4address
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
echo "192.168.100.2/26" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ens18/ipv4route
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
nmcli con mod "Wired connection 2" ipv4.method "manual" ipv4.addresses "192.168.1.1/27"

#mkdir /etc/net/ifaces/ens19
#cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens19/options
#echo "192.168.1.1/27" > /etc/net/ifaces/ens19/ipv4address
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
echo "192.168.1.2/27" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.1.1" > /etc/net/ifaces/ens18/ipv4route
echo "nameserver 8.8.8.8" > /etc/net/ifaces/ens18/resolv.conf
systemctl restart network

# Обновление индекса пакетов
apt-get update -y

# Вывод hostname и сетевых интерфейсов
hostname
ip -br a
```

## Задание 3

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

## Задание 5

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

## Задание 6

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
```

## Задание 7

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
network 192.168.100.0/26 area 0
network 192.168.200.0/28 area 0
network 192.168.99.0/29 area 0
ex
int gre1
no ip ospf passive
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
network 192.168.1.0/27 area 0
ex
int gre1
no ip ospf passive
ex
ex
wr
ex

cat /etc/frr/frr.conf
systemctl restart frr
```
