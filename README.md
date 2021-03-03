# VPS-Nginx-OpenVPN-to-SynologyNAS

Цель проекта: доступ к сетевому хранилищу Synology по собственному доменному имени. К хранилищу нет доступа извне, на роутере закртыт порты, ip раздается провайдером динамически, статический ip невозможен. На хранилище запущено несколько сервисов, к каждому из них должен обеспечивается доступ по собственному именни поддомена. В качестве примера будет использоваться домен meltan.ru. На момент написания статьи (28.02.2021) домен был свободен.

# Настройка VPS сервера

Создаем VPS на площадке провайдера. Привязываем доменное имя, создаем нужные поддомены. Получаем доступ по SSH.

Подключемся по SSH

```
ssh root@server_ip
```

Меняем пароль root

```
passwd root
```

Обновляем систему

```
apt-get update
apt-get upgrade
```

Добавляем пакеты на сервер

```
apt install vim net-tools tree ncdu bash-completion curl dnsutils htop iftop pwgen screen sudo wget
```

Добавляем Fail2ban

```
apt install fail2ban
```

Добавляем нового пользователя 

```
useradd -m -s /bin/bash USERNAME
```

Добавляем пользователя в sudo

```
adduser USER sudo
```

Задаем пароль для пользователя

```
passwd USER
```

Правим конфиг SSH

```
nano /etc/ssh/sshd_config
```

Закрываем логин root

```
PermitRootLogin no
```

Рестартуем SSH

```
systemctl restart sshd
```

***Открываем новую сессию и проверяем можем ли войти под новым пользователем***

```
ssh USER@server_ip
```

***Если все ок, выходим из сессии root. Все, под root больше не войдем!***

```
exit
```

Выходим из сессии пользователя

```
exit
```

## Настраиваем пользователя

Подключаемся:

```
ssh USER@IP
```

Делаем папку .ssh для ключа:

```
mkdir -p ~/.ssh
```

На сервере создаем файл для ключа:

```
nano ~/.ssh/authorized_keys
```

На рабочей машине в терминале:

```
cat ~/.ssh/id_rsa.pub | pbcopy
```

Выставляем права:

```
sudo chmod -R 700 ~/.ssh/
```

Открываем настройки ssh:

```
sudo nano /etc/ssh/sshd_config
```

Отключаем вход по паролю

```
PasswordAuthentication no
```

Рестартуем ssh:

```
sudo systemctl restart sshd
```

Открываем настройку sudo:

```
sudo visudo
```

Настраиваем sudo без пароля. Находим в конфигурации нужную строку и изменяем ее на нижеследующую:

```
%sudo   ALL=(ALL:ALL) NOPASSWD:ALL
```

## Поднимаем OpenVPN на сервере

Скрипт берем здесь https://github.com/Nyr/openvpn-install. Спасибо ребятам!

```
wget https://git.io/vpn -O openvpn-install.sh && bash openvpn-install.sh
```

После запуска скрипта отвечаем на вопросы и получаем файл сертификата. Он бдует лежать в домашней директории root /root.

Проверяем наличие vpn-тунеля.

```
ip a
```

В выводе ищем интефейс tun0:. В его параметрах нужен адрес сервера в нашей виртуальной сети (10.8.0.1)

```
inet 10.8.0.1/24 brd 10.8.0.255 scope global tun0
```

## Установка и настройка Nginx на сервере

Создаем файл конфигурации для сайта. 

```
sudo nano /etc/nginx/sites-available/meltan.ru
```

Вставляем следующий код. В строке proxy_pass http://10.8.0.3:80 нужно будет сменить адрес на реальный адрес клиента, который мы получим позднее.

```
server {
        server_name meltan.ru;
        listen 80;
        location /{
                proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                        proxy_set_header X-Forwarded-Proto $scheme;

                        client_max_body_size 0;
                        add_header Strict-Transport-Security "max-age=31536000; inclideSubDomains: preload";
                        add_header Referrer-Policy "same-origin";
                        proxy_pass http://10.8.0.3:80;
        }
}
```
Создаем симлинк в директорию запускаемых хостов

```
sudo ln -s /etc/nginx/sites-available/meltan.ru /etc/nginx/sites-enabled/meltan.ru
```

Удаляем ссылку дефолтной конфигурации серевера

```
sudo rm /etc/nginx/sites-enabled/defauil
```

Проверяем конфигурацию сервера

```
sudo nginx -t
```

Должно быть

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

```

Перезагружаем конфигурацию Nginx

```
sudo nginx -s reload
```

## Установка и настройка клиента OpenVPN

Если получится поднять клиент на Synology - отлично! Значит сертификаты начали импортироваться. Если нет - ставим виртуалку Ubuntu на NAS. На ней поднимаем клиент OpenVPN.

Устанавливаем OpenVPN

```
sudo apt install openvpn
```

Копируем полученный сертификат file.ovpn в домашнюю директорию любым способом

Копируем сертификат в папку openvpn и назначаем его конфигурацией

```
sudo cp file.ovpn /etc/openvpn/client.conf
```
Запускаем OpenVPN

```
openvpn --client --config /etc/openvpn/client.conf &
```

Проверяем адрес клиента в тунеле

```
ip a 
```
Находим строку с примерно таким содержанием

```
tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> 
    link/none 
    inet 10.8.0.4/24 brd 10.8.0.255 
```
Запоминаем адрес 10.8.0.4 (у вас будет свой адрес). Этот адрес нужно вставить в конфигурационный файл nginx на вашем сервере.

## Продолжение настройки Nginx на VPS

Открываем конфигурацинный файл нашего хоста

```
sudo nano /etc/nginx/sites-available/meltan.ru
```

Добавляем адрес нашего клиента в конфигурацию

```
server {
        server_name meltan.ru;
        listen 80;
        location /{
                proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                        proxy_set_header X-Forwarded-Proto $scheme;

                        client_max_body_size 0;
                        add_header Strict-Transport-Security "max-age=31536000; inclideSubDomains: preload";
                        add_header Referrer-Policy "same-origin";
                        proxy_pass http://10.8.0.4:80;
        }
}
```
Проверяем конфигурацию сервера

```
sudo nginx -t
```

Должно быть

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

```

Перезагружаем конфигурацию Nginx

```
sudo nginx -s reload
```


## Установка и настройка Nginx на клиенте

После установки OpenVPN клиента ставим Nginx и настраиваем его как reverse proxy на локальный адрес NAS





