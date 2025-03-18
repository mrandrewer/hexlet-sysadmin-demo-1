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

