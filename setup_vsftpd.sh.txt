#!/bin/bash

# Обновление системы
sudo apt update
sudo apt install vsftpd -y

# Создание пользователей
sudo useradd -m -s /usr/sbin/nologin -G ftp superuser1
sudo passwd superuser1 <<EOF
your_password_here
your_password_here
EOF

sudo useradd -m -s /usr/sbin/nologin -G ftp ftpuser1
sudo passwd ftpuser1 <<EOF
your_password_here
your_password_here
EOF

sudo useradd -m -s /usr/sbin/nologin -G ftp ftpuser2
sudo passwd ftpuser2 <<EOF
your_password_here
your_password_here
EOF

# Настройка общей директории
sudo mkdir -p /home/exchange
sudo chown superuser1:ftp /home/exchange
sudo chmod 775 /home/exchange

# Настройка монтирования для пользователей
sudo mkdir -p /home/ftpuser1/exchange
sudo mount --bind /home/exchange /home/ftpuser1/exchange
sudo mkdir -p /home/ftpuser2/exchange
sudo mount --bind /home/exchange /home/ftpuser2/exchange

# Добавление записей в fstab
echo "/home/exchange /home/ftpuser1/exchange none bind 0 0" | sudo tee -a /etc/fstab
echo "/home/exchange /home/ftpuser2/exchange none bind 0 0" | sudo tee -a /etc/fstab

# Настройка прав доступа
sudo chmod 775 /home/exchange
sudo chown superuser1:ftp /home/exchange

# Добавление пользователей в список VSFTPD
echo "superuser1" | sudo tee -a /etc/vsftpd/user_list
echo "ftpuser1" | sudo tee -a /etc/vsftpd/user_list
echo "ftpuser2" | sudo tee -a /etc/vsftpd/user_list

# Создание конфигурационных файлов для пользователей
sudo mkdir -p /etc/vsftpd/user_config
echo "anon_world_readable_only=YES" | sudo tee /etc/vsftpd/user_config/ftpuser1
echo "write_enable=NO" | sudo tee -a /etc/vsftpd/user_config/ftpuser1
echo "anon_world_readable_only=YES" | sudo tee /etc/vsftpd/user_config/ftpuser2
echo "write_enable=NO" | sudo tee -a /etc/vsftpd/user_config/ftpuser2

# Перезапуск службы VSFTPD
sudo systemctl restart vsftpd
sudo systemctl status vsftpd

# Дополнительные права доступа
sudo chmod 750 /home/superuser1
sudo chown superuser1:ftp /home/superuser1
sudo chown superuser1:ftp /home/superuser1/exchange
sudo chmod 775 /home/superuser1/exchange
sudo mount --bind /home/exchange /home/superuser1/exchange

# Добавление записи в fstab для superuser1
echo "/home/exchange /home/superuser1/exchange none bind 0 0" | sudo tee -a /etc/fstab
sudo mount -a

# Права для общего каталога
sudo chmod 775 /home/exchange
sudo chown superuser1:ftp /home/exchange
sudo chmod 775 /home/superuser1/exchange
sudo chown superuser1:ftp /home/superuser1/exchange
sudo chmod 755 /home/ftpuser1/exchange
sudo chmod 755 /home/ftpuser2/exchange

# Исправление ошибки 530
echo "/usr/sbin/nologin" | sudo tee -a /etc/shells

# Логирование
sudo touch /var/log/vsftpd.log
sudo chown ftp:ftp /var/log/vsftpd.log
sudo chmod 600 /var/log/vsftpd.log
sudo systemctl restart vsftpd

# Настройка конфигурации VSFTPD
cat <<EOF | sudo tee /etc/vsftpd.conf
listen=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
chroot_local_user=YES
allow_writeable_chroot=YES
userlist_enable=YES
userlist_file=/etc/vsftpd/user_list
userlist_deny=NO
user_config_dir=/etc/vsftpd/user_config
dirlist_enable=YES
file_open_mode=0664
async_abor_enable=YES
xferlog_std_format=YES
xferlog_file=/var/log/vsftpd.log
xferlog_enable=YES
log_ftp_protocol=YES
syslog_enable=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
pasv_min_port=40000
pasv_max_port=50000
rsa_cert_file=/etc/ssl/private/vsftpd.pem
rsa_private_key_file=/etc/ssl/private/vsftpd.pem
ssl_enable=YES
EOF

# Перезапуск службы VSFTPD
sudo systemctl restart vsftpd