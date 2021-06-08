# VPS with Double Nginx through OpenVPN to Local NAS


Цель проекта: доступ к сетевому хранилищу Synology по собственному доменному имени. К хранилищу нет доступа извне, на роутере закрыты порты, ip раздается провайдером динамически, статический ip невозможен. На хранилище запущено несколько сервисов, к каждому из них должен обеспечивается доступ по собственному имени поддомена. 

# Настройка VPS сервера

Создаем VPS на площадке провайдера. Привязываем доменное имя, создаем нужные поддомены. Получаем доступ по SSH.

Если сервер у вас уже настроен, можете пропустить эту часть и следующую и сразу перейти к установке и настройке OpenVPN.

Подключемся по SSH:

```
ssh root@server_ip
```

Меняем пароль root:

```
passwd root
```

Обновляем систему:

```
apt-get update
apt-get upgrade
```

Добавляем нового пользователя:

```
useradd -m -s /bin/bash USERNAME
```

Добавляем пользователя в sudo:

```
adduser USER sudo
```

Задаем пароль для пользователя:

```
passwd USER
```

Правим конфиг SSH:

```
nano /etc/ssh/sshd_config
```

Закрываем логин root:

```
PermitRootLogin no
```

Рестартуем SSH:

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

Выходим из сессии пользователя:

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

На рабочей машине в терминале вводим (для MacOS):

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

Отключаем вход по паролю:

```
PasswordAuthentication no
```

Рестартуем ssh:

```
sudo systemctl restart sshd
```


## Поднимаем OpenVPN на сервере

Скрипт берем здесь https://github.com/Nyr/openvpn-install. Спасибо ребятам!

```
wget https://git.io/vpn -O openvpn-install.sh && bash openvpn-install.sh
```

После запуска скрипта отвечаем на вопросы (в основном, все по умолчанию) и получаем файл сертификата. Он будет лежать в домашней директории root /root.

Проверяем наличие vpn-тунеля:

```
ip a
```

В выводе ищем интерфейс tun0:. В его параметрах нужен адрес сервера в нашей виртуальной сети (10.8.0.1)

```
inet 10.8.0.1/24 brd 10.8.0.255 scope global tun0
```

## Установка и настройка Nginx на сервере

Устанавливаем nginx:

```
 sudo apt install nginx
```
Теперь можно зайти по своему доменному имени и насладиться дефолтой страничкой nginx! Если не получилось, попробуйте рестартовать nginx

```
sudo nginx -s reload
```
Если вылетает 403 ошибка - у вас, скорее всего закрыт 80 порт на файрволле. 

Проверьте статус файрволла:

```
sudo ufw status
```

Если в выводе нет Nginx, как здесь, его нужно добавить.

```
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                  
OpenSSH (v6)               ALLOW       Anywhere (v6)  
```

Добавьте Nginx. Это откроет 80 (http) и 443 (https) порты:

```
sudo ufw allow 'Nginx Full'
```
Проверьте статус, вывод должен быть таким.

```
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                  
Nginx Full                 ALLOW       Anywhere                  
OpenSSH (v6)               ALLOW       Anywhere (v6)             
Nginx Full (v6)            ALLOW       Anywhere (v6)
```
Если у вас нет файрволла, установите его и проделайте предыдущий шаг по работе с ним. Это точно будет не лишним.

Продолжаем настраивать виртуальный хост. Создаем файл конфигурации для сайта: 

```
sudo nano /etc/nginx/sites-available/domain.ru
```

Вставляем следующий код. Директиву **server_name domain.ru** меняем на имя своего домена. В строке **proxy_pass http://10.8.0.3:80** нужно будет сменить адрес на реальный адрес клиента, который мы получим позднее.

```
server {
        server_name domain.ru;
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
Создаем симлинк в директорию запускаемых хостов:

```
sudo ln -s /etc/nginx/sites-available/domain.ru /etc/nginx/sites-enabled/domain.ru
```

Удаляем ссылку дефолтной конфигурации сервера:

```
sudo rm /etc/nginx/sites-enabled/defauil
```

Проверяем конфигурацию сервера:

```
sudo nginx -t
```

Должно быть так:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

```

Перезагружаем конфигурацию Nginx:

```
sudo nginx -s reload
```

## Установка и настройка клиента OpenVPN

Если получится поднять клиент на Synology - отлично! Значит сертификаты начали импортироваться. Если нет - ставим виртуалку Ubuntu на NAS. На ней поднимаем клиент OpenVPN. Я пробовал делать и на виртуалке и на отдельно стоящей raspberry pi - работало в обоих случаях. В случае с отдельным сервером вы разгружаете NAS от работы виртуальной машины.

Устанавливаем OpenVPN:

```
sudo apt install openvpn
```

Копируем полученный сертификат file.ovpn, который вы создали на сервере, в домашнюю директорию любым способом, sftp например.

Переходим в папку с сертификатом, копируем его в папку openvpn и назначаем его конфигурацией:

```
sudo cp file.ovpn /etc/openvpn/client.conf
```
Запускаем OpenVPN:

```
openvpn --client --config /etc/openvpn/client.conf &
```

Нажимаем еще раз enter.

Проверяем адрес клиента в тунеле:

```
ip a 
```
Находим строку с tun0:

```
tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> 
    link/none 
    inet 10.8.0.4/24 brd 10.8.0.255 
```
Запоминаем адрес **10.8.0.4** (у вас будет свой адрес). Этот адрес нужно вставить в конфигурационный файл nginx на вашем сервере.

Чтобы автоматически стартовал  OpenVPN

```
systemctl enable openvpn-client@.service
```

## Продолжение настройки Nginx на VPS

Открываем конфигурационный файл нашего хоста:

```
sudo nano /etc/nginx/sites-available/domain.ru
```

Меняем адрес нашего клиента в конфигурации на полученный (**10.8.0.4**, у вас свой адрес, помните?):

```
server {
        server_name domain.ru;
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
Проверяем конфигурацию сервера:

```
sudo nginx -t
```

Должно быть:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

```

Перезагружаем конфигурацию Nginx:

```
sudo nginx -s reload
```
Все, мы перенаправили весь трафик с доменного имени в тунель до нашего хоста внутри локальной сети. Осталось перенаправить поток с этого хоста на нужый адрес внутренней сети. Но сначала нужно обзавестись SSL-сертификатом, чтобы соединение было безопасным.

## Установка SSL-сертификата на VPS

Для установки SSL-сертификата воспользуемся https://certbot.eff.org/. В строке **My HTTP website is running** выбираем приложение (Nginx) и операционную систему вашего VPS (у меня Ubuntu 20.04). Робот настроится на ваши данные и выдаст все шаги и команды, которые нужно выполнить. В результате ваш сайт будет защищен SSL-сертификатом. 

Бот добавит сертификаты и изменит конфигурационный файл вашего виртуального хоста, который мы создали и правили ранее.

```
server {
        server_name domain.ru;
        location /{
                proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                        proxy_set_header X-Forwarded-Proto $scheme;

                        client_max_body_size 0;
                        add_header Strict-Transport-Security "max-age=31536000; inclideSubDom>
                        add_header Referrer-Policy "same-origin";
                        proxy_pass http://10.8.0.4:80;
        }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/domain.ru/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/domain.ru/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = domain.ru) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


        server_name domain.ru;
        listen 80;
    return 404; # managed by Certbot
}
```

Чтобы сертификат обновлялся автоматически через 90 дней, добавьте в cron (```crontab -e```) строчку:

```
0 5 */90 * * certbot renew
```
Для каждого дополнительного поддомена вам нужно сделать те же действия, что и для основного:
1. Создать конфиг в папке sites-available
2. Созадть линк в папку sites-enabled
3. Проверить конфиг nginx
4. Перезагрузить nginx
5. Добавить SSL-сертификат для поддомена (certbot)

## Установка и настройка Nginx на клиенте

После установки OpenVPN клиента ставим Nginx и настраиваем его как reverse proxy на локальный адрес NAS. Клиент-это ваша виртуальная машина с linux или другое устройство, работающее как прокси-сервер.

Устанавливаем Nginx:

```
 sudo apt install nginx
```
После этого можно заходить по имени вашего домена и видеть уже дефолтную страницу Nginx на вашем внутреннем прокси. Чтобы понять это, измените индексную страницу Nginx на клиенте. Путь к ней /var/www/html/index.nginx-debian.html 

Теперь создадим конфигурационный файл для форварда трафика на наш NAS:

```
sudo nano /etc/nginx/sites-available/nas
```

Вставляем код. Вместо домена domain.ru вставляете имя своего домена. В строке proxy_pass http://your_nas_ip; указываете внутренний ip вашей машины (192.168.x.x). В моем примере ссылка ведет на папку /nextcloud. Директива сервера ds.domain.ru ведет на страницу управления NAS. Используя http://your_nas_ip:port, можно попасть на любой сервис на вашем NAS, доступный по этому порту.  В одном файле можно собрать все домены и поддомены, относящиеся к данному серверу. 

В качестве дополнительного образования не помешает посмотреть курс по настройке Nginx или почитать документацию на странице продукта (она на русском!)

```
server {
        server_name domain.ru;
        listen 80;
        location /{
                proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                        proxy_set_header X-Forwarded-Proto $scheme;

                        client_max_body_size 0;
                        add_header Strict-Transport-Security "max-age=31536000; inclideSubDomains: preload";
                        add_header Referrer-Policy "same-origin";
                        proxy_pass http://your_nas_ip;
                        root /nextcloud;
        }
}
server {
        server_name ds.domain.ru;
        listen 80;
        location /{
                proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                        proxy_set_header X-Forwarded-Proto $scheme;

                        client_max_body_size 0;
                        add_header Strict-Transport-Security "max-age=31536000; inclideSubDomains: preload";
                        add_header Referrer-Policy "same-origin";
                        proxy_pass http://your_nas_ip:5000;
        }
}
```

Создаем симлинк в директорию запускаемых хостов:

```
sudo ln -s /etc/nginx/sites-available/nas /etc/nginx/sites-enabled/nas
```

Удаляем ссылку дефолтной конфигурации серевера:

```
sudo rm /etc/nginx/sites-enabled/defauil
```

Проверяем конфигурацию сервера:

```
sudo nginx -t
```

Должно быть так:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

```

Перезагружаем конфигурацию Nginx:

```
sudo nginx -s reload
```

По идее, после всех этих действий у вас должны открываться окна тех сервисов, что вы назначили в конфигурационном файле на вашем прокси внутри сети. Помните, один домен (поддомен) - один сервис. Нельзя набрать в строке https://domain.ru:8080 и попасть на порт 8080 локального NAS (азбука, но лучше лишний раз сказать). Если это не сработало, скорее всего, вы где-то допустили ошибку, и это нормально. Или я допустил ошибку (и это тоже нормально) и буду вам очень благодарен, если вы укажете на нее. И помните, работа с open source software это прежде всего хорошее владение командами --help, man и поиском в google. Have fun!

<!-- comment -->



