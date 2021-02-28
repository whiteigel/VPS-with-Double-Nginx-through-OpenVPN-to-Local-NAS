# VPS-Nginx-OpenVN-to-NAS

Цель проекта: доступ к сетевому хранилищу Synology по собственному доменному имени. Хранилище не имеет доступ наружу, на роутере закртыт порты, ip раздается провайдером динамически, статический ip невозможен. На хрунилище запущено несколько сервисов, к каждому из них обеспечивается свой доступ по именни поддомена.

# Настройка VPS сервера

Создаем VPS на площадке провайдера. Привызяваем доменное имя, создаем нужные поддомены. Получаем доступ по SSH.

Подключемся по SSH

```ssh root@server_ip```

Меняем пароль root

```passwd root```

Проверяем обновления

```apt-get update```

И обновляем систему

```apt-get upgrade```

Добавляем пакеты на сервер

apt install vim net-tools tree ncdu bash-completion curl dnsutils htop iftop pwgen screen sudo wget

Добавляем Fail2ban

apt install fail2ban

Добавляем нового пользователя 

```useradd -m -s /bin/bash USERNAME```

Добавляем пользователя в sudo

```adduser USER sudo```

Задаем пароль для пользователя

```passwd USER```

Правим конфиг SSH

```nano /etc/ssh/sshd_config```

Закрываем логин root

```PermitRootLogin no```

Рестартуем SSH

```systemctl restart sshd```

***Открываем новую сессию и проверяем можем ли войти под новым пользователем***

```ssh USER@server_ip```

***Если все ок, выходим из сессии root. Все, под root больше не войдем!***

```exit```

Выходим из сессии пользователя

```exit```

## Настраиваем пользователя

Подключаемся:

```ssh USER@IP```

Делаем папку .ssh для ключа:

```mkdir -p ~/.ssh```

На сервере создаем файл для ключа:

```nano ~/.ssh/authorized_keys```

На рабочей машине в терминале:

```cat ~/.ssh/id_rsa.pub | pbcopy```

Выставляем права:

```sudo chmod -R 700 ~/.ssh/```

Открываем настройку ssh:

```sudo nano /etc/ssh/sshd_config```

Отключаем вход по паролю

```PasswordAuthentication no```

Рестартуем ssh:

```sudo systemctl restart sshd```

Открываем настройку sudo:

```sudo visudo```

Настраиваем sudo без пароля. Находим в конфигурации нужную строку и изменяем ее на нижеследующую:

```%sudo   ALL=(ALL:ALL) NOPASSWD:ALL```

## Поднимаем OpenVPN на сервере

Скрипт берем здесь https://github.com/Nyr/openvpn-install. Спасибо ребятам!

```wget https://git.io/vpn -O openvpn-install.sh && bash openvpn-install.sh```

После запуска скрипта отвечаем на вопросы и получаем файл сертификата. Он бдует лежать в домашней директории root /root. 

## Установка и настройка Nginx на сервере

# Настройка Synology

## Установка и настройка клиента OpenVPN

Если получится поднять клиент на Synology - отлично! Значит сертификаты начали импортироваться. Если нет - ставим виртуалку Ubuntu на NAS. На ней поднимаем клиент OpenVPN.

## Установка и настройка Nginx на клиенте

После установки OpenVPN клиента ставим Nginx и настраиваем его как reverse proxy на локальный адрес NAS





