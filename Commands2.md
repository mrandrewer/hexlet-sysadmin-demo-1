# Модуль 2 Организация сетевого администрирования операционных систем


## Задание 1 Настройка доменного контроллера Samba на машине BR-SRV.

Установка samba server на BR-SRV
```sh
apt-get install -y task-samba-dc

rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba
rm -rf /var/cache/samba

mkdir -p /var/lib/samba/sysvol

samba-tool domain provision
```

Вводим параметры для сервера (все по умолчанию) и пароль для администратора  
После этого включаем службу
```sh
systemctl enable --now samba
/bin/cp /var/lib/samba/private/krb5.conf /etc/krb5.conf 
```

Подготовим файл с пользователями
```sh
echo 'user1.hq,P@ssw0rd' >> /opt/users.csv
echo 'user2.hq,P@ssw0rd' >> /opt/users.csv
echo 'user3.hq,P@ssw0rd' >> /opt/users.csv
echo 'user4.hq,P@ssw0rd' >> /opt/users.csv
echo 'user5.hq,P@ssw0rd' >> /opt/users.csv
```

Создадим скрипт импорта в файле /opt/import.sh
`
#!/bin/bash
while IFS=',' read -r username password; do
    samba-tool user create "$username" "$password"
    samba-tool group addmembers hq "$username"
done < /opt/users.csv
`

Выполним скрипт импорта
```sh
chmod +x /opt/import.sh
samba-tool group create hq
/opt/import.sh
```

```sh
echo '%hq ALL = (ALL:ALL) NOPASSWD: /bin/cat, /bin/grep, /usr/bin/id' >> /etc/sudoers
```

Домен готов!

Введем машину HQ-CLI в домен

## Задание 2 Сконфигурируйте файловое хранилище на HQ-SRV

Неразмеченные диски добавляем руками через оснастку proxmox

Для очистки можно использовать скрипт
```sh
# Обнуляем суперблоки для добавленных дисков
mdadm --zero-superblock --force /dev/sd{b,c,d}
# Удаляем старые метаданные и подпись на дисках:
wipefs --all --force /dev/sd{b,c,d}
```

Создаем массив
```sh
mdadm --create /dev/md0 -l 5 -n 3 /dev/sd{b,c,d}
/bin/cp /etc/mdadm.conf.sample /etc/mdadm.conf
mdadm --detail --scan | tee -a /etc/mdadm.conf
```

Создаем файловую систему и точку монтирования
```sh
mkfs -t ext4 /dev/md0
mkdir /mnt/raid5
echo "/dev/md0  /mnt/raid5  ext4  defaults  0  0" >> /etc/fstab
mount -a
df -h
```

Устанавливаем и настраиваем сервер nfs
```sh
apt-get install -y nfs-server
mkdir /mnt/raid5/nfs
chmod 766 /mnt/raid5/nfs
echo "/mnt/raid5/nfs 192.168.200.0/28(rw,no_root_squash)" >> /etc/exports
exportfs -a
systemctl enable --now nfs-server
```

Настраиваем доступ к nfs на клиенте
```sh
mkdir /mnt/nfs
echo "192.168.100.2:/mnt/raid5/nfs  /mnt/nfs  nfs  defaults  0  0" >> /etc/fstab
mount -a
df -h
```


## Задание 3 Настройте службу сетевого времени на базе сервиса chrony 

Устанавливаем chrony на hq-rtr
```sh
apt-get install -y chrony
```

Модифицируем файл /etc/chrony.conf
Вписываем следующие настройки вместо использования pool.ntp.org
`
server 127.0.0.1 iburst prefer
local stratum 5

allow 192.168.100.0/26
allow 192.168.200.0/28
allow 192.168.1.0/27
allow 10.0.1.0/30
`

Запускаем и проверяем работу chrony
```sh
systemctl enable --now chronyd
chronyc sources
chronyc tracking | grep Stratum
```

Настраиваем клиентов chrony
```sh
apt-get install -y chrony
sed -i 's/pool pool.ntp.org/server 192.168.100.1/g' /etc/chrony.conf
systemctl enable --now chronyd
chronyc tracking 
```

Для hq-cli
```sh
apt-get install -y chrony
sed -i 's/pool AU-TEAM.IRPO/server 192.168.100.1/g' /etc/chrony.conf
systemctl enable --now chronyd
chronyc tracking 
```