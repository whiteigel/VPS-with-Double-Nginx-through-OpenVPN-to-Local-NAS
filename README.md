# VPS-Nginx-OpenVN-to-NAS

Создаем VPS на площадке провайдера. Получаем доступ по SSH.

Подключемся по SSH

``` ssh root@server_ip ```

Меняем пароль root

passwd root

Обновляем систему

apt-get update
apt-get upgrade

Добавляем нового пользователя 

adduser USER

Добавляем пользователя в sudo

adduser USER sudo

Задаем пароль для пользователя

passwd USER

Правим конфиг SSH

nano /etc/ssh/sshd_config

Закрываем логин root

PermitRootLogin no

Рестартуем SSH

systemctl restart sshd

Открываем новую сессию и проверяем можем ли войти под новым пользователем

ssh USER@server_ip

Если все ок, выходим из сессии root. Все, под root больше не войдем!

exit

Выходим из сессии пользователя

' exit '

# Настраиваем пользователя

Подключаемся

ssh USER@IP

Делаем папку .ssh для ключа

mkdir -p ~/.ssh

Вводим ключ

nano ~/.ssh/authorized_keys
cat ~/.ssh/id_rsa.pub | pbcopy

Выставляем права

sudo chmod -R 700 ~/.ssh/

Открываем настройку ssh

sudo nano /etc/ssh/sshd_config

Отключаем вход по паролю

PasswordAuthentication no

Рестартуем ssh

sudo systemctl restart sshd

Открываем настройку sudo

sudo visudo

Настраиваем sudo без пароля. 

%sudo   ALL=(ALL:ALL) NOPASSWD:ALL




