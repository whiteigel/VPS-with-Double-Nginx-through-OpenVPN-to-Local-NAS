# VPS-Nginx-OpenVN-to-NAS

Создаем VPS на площадке провайдера. Получаем доступ по SSH.

Подключемся по SSH

ssh root@server_ip

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

ssh root@server_ip

Если все ок, выходим из сессии root. Все под root больше не войдем!

exit
