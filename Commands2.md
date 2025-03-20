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

## Задание 4 Сконфигурируйте ansible на сервере BR-SRV 

Устанавливаем ansible на br-srv
```sh
apt-get install -y ansible

echo "[all]" >> /etc/ansible/hosts
echo "hq-rtr ansible-host=192.168.100.1 ansible_connection=local" >> /etc/ansible/hosts
echo "hq-srv ansible-host=192.168.100.2 ansible_connection=local" >> /etc/ansible/hosts
echo "hq-cli ansible-host=192.168.200.2 ansible_connection=local" >> /etc/ansible/hosts
echo "br-rtr ansible-host=192.168.1.1 ansible_connection=local" >> /etc/ansible/hosts
```

## Задание 5 Развертывание приложений в Docker на сервере BR-SRV

Устанавливаем docker на сервере
```sh
apt-get install -y docker-io docker-compose
systemctl enable --now docker
mkdir -p /opt/docker
vim /opt/docker/wiki.yml
```

В файл /opt/docker/wiki.yml записываем конфигурацию для контейнеров
`
# MediaWiki with MariaDB
#
# Access via "http://localhost:8080"
#   (or "http://$(docker-machine ip):8080" if using docker-machine)
version: '3'
services:
  wiki:
    container_name: wiki
    image: mediawiki
    restart: always
    ports:
      - 8080:80
    links:
      - mariadb
    volumes:
      - images:/var/www/html/images
      # - /root/LocalSettings.php:/var/www/html/LocalSettings.php
  # This key also defines the name of the database host used during setup instead of the default "localhost"
  mariadb:
    container_name: mariadb
    image: mariadb
    restart: always
    environment:
      MYSQL_DATABASE: mediawiki
      MYSQL_USER: wiki
      MYSQL_PASSWORD: WikiP@ssw0rd
      MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
    volumes:
      - db:/var/lib/mysql

volumes:
  images:
  db:
`

```sh
docker compose -f /opt/docker/wiki.yml up -d 
docker compose ps
```

Далее открываем адрес вики с CLI и настраиваем в браузере
После этого переносим содержимое загруженного файла LocalSettings.php на BR-SRV
По заданию он должен лежать в домашней папке пользователя, так что это будет /root/LocalSettings.php
Правим файл конфига и перезапускаем docker-compose

## Задание 6 Проброс портов на маршрутизаторах

Команды для br-rtr
```sh
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.2:8080 
iptables -t nat -A PREROUTING -p tcp --dport 2024 -j DNAT --to-destination 192.168.1.2:2024
iptables-save > /etc/sysconfig/iptables
systemctl restart iptables
iptables-save 
```

Команды для hq-rtr
```sh
iptables -t nat -A PREROUTING -p tcp --dport 2024 -j DNAT --to-destination 192.168.100.2:2024
iptables-save > /etc/sysconfig/iptables
systemctl restart iptables
iptables-save 
```

## Задание 7 Установка и настройка moodle на сервере HQ-SRV

Устанавливаем необходимое ПО
```sh
apt-get install -y mariadb-server moodle-apache2 moodle-local-mysql
```

Настраиваем mariadb
```sh
systemctl enable --now mariadb
mysql_secure_installation
```

Создаем БД для moodle
```sh
mysql -u root -p
```

Создаем БД для moodle
```sql
create database moodledb character set utf8mb4 collate utf8mb4_unicode_ci;
create user 'moodle'@'localhost' identified by 'P@ssw0rd';
grant all privileges on moodledb.* to  'moodle'@'localhost';
flush privileges;
exit;
```

Устанавливаем max input vars для php
```sh
nano /etc/php/8.2/apache2-mod_php/php.ini 
```

Инициализируем скрипт установки moodle
```sh
/usr/bin/php /var/www/webapps/moodle/admin/cli/install.php
```

```sh
chown -R apache:apache /var/lib/moodle/default
chmod -R 0777 /var/lib/moodle/default
a2enmod rewrite_

```

## Задание 8 Настройка nginx на HQ-RTR 

```sh
apt-get install -y nginx nano
nano /etc/nginx/sites-available.d/proxy.conf
```

Вписываем содержимое файла
```conf
server {
    listen 80;
    server_name wiki.au-team.irpo;
   
    location / {
        proxy_pass http://192.168.1.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 80;
    server_name moodle.au-team.irpo;
   
    location / {
        proxy_pass http://192.168.100.2;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Включаем проксирование

```sh
ln -s /etc/nginx/sites-available.d/proxy.conf /etc/nginx/sites-enabled.d/
systemctl restart nginx
systemctl enable --now nginx
```

## Задание 9 Установка яндекс браузер
```sh
apt-get install -y yandex-browser-stable
```
