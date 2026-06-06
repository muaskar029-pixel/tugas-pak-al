```
whoami
hostnamectl
uname -a
```
Update sistem:
```
pacman -Syu
```
```
pacman -S --needed git curl wget unzip nano vim sudo
```

## Install Apache, PHP-FPM, dan MariaDB

Install package utama:
```
sudo pacman -S --needed apache php php-fpm php-gd php-intl php-zip mariadb firewalld
```
Aktifkan service utama:
```
sudo systemctl enable --now httpd
sudo systemctl enable --now php-fpm
sudo systemctl enable --now firewalld
```
Cek status:
```
sudo systemctl status httpd
sudo systemctl status php-fpm
sudo systemctl status firewalld
```

Konfigurasi PHP untuk Arteri

Edit file PHP:
```
sudo nano /etc/php/php.ini
```
Aktifkan extension berikut dengan menghapus tanda ; jika ada:
```
extension=mysqli
extension=pdo_mysql
extension=gd
extension=intl
extension=zip
extension=gettext
extension=mbstring
```
Cari dan sesuaikan konfigurasi upload:
```
file_uploads = On
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 300
date.timezone = Asia/Jakarta
```
Restart PHP-FPM:
```
sudo systemctl restart php-fpm
```
———

## 6. Konfigurasi Apache + PHP-FPM

Aktifkan module Apache yang diperlukan.

Edit konfigurasi Apache:
```
sudo nano /etc/httpd/conf/httpd.conf
```
Pastikan module berikut aktif:
```
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
LoadModule rewrite_module modules/mod_rewrite.so
```
Cari baris berikut:
```
#Include conf/extra/httpd-vhosts.conf
```
Ubah menjadi:
```
Include conf/extra/httpd-vhosts.conf
```
Buat konfigurasi virtual host Arteri:
```
sudo nano /etc/httpd/conf/extra/httpd-vhosts.conf
```
Isi:
```
<VirtualHost *:80>
    ServerName arteri.local
    DocumentRoot "/srv/http/arteri"

    <Directory "/srv/http/arteri">
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler
        "proxy:unix:/run/php-fpm/php-fpm.sock|fcgi://localhost/"
    </FilesMatch>

    ErrorLog "/var/log/httpd/arteri-error.log"
    CustomLog "/var/log/httpd/arteri-access.log" combined
</VirtualHost>
```
Tes konfigurasi Apache:
```
sudo apachectl configtest
```
Jika hasilnya:
```
Syntax OK
```
restart Apache:
```
sudo systemctl restart httpd
```
———

## Install dan Amankan MariaDB

Inisialisasi database MariaDB:
```
sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/
mysql
```
Aktifkan MariaDB:
```
sudo systemctl enable --now mariadb
```
Amankan instalasi MariaDB:
```
sudo mariadb-secure-installation
```
Rekomendasi jawaban:
```
Switch to unix_socket authentication: Y
Change the root password: Y
Remove anonymous users: Y
Disallow root login remotely: Y
Remove test database: Y
Reload privilege tables: Y
```
Masuk ke MariaDB:
```
sudo mariadb
```
Buat database dan user arteri:
```
CREATE DATABASE arsipDigital CHARACTER SET utf8mb4 COLLATE
utf8mb4_unicode_ci;
CREATE USER 'arteriuser'@'localhost' IDENTIFIED BY 'GantiPasswordKuat';
GRANT ALL PRIVILEGES ON arsipDigital.* TO 'arteriuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
## Ambil Project arteri dari GitHub

Masuk ke direktori web:
```
cd /srv/http
```
Clone project arteri:
```
sudo git clone https://github.com/dicarve/arteri.git arteri
```
Masuk folder arteri:
```
cd arteri
```
Set ownership ke user Apache:
```
sudo chown -R http:http /srv/http/arteri
```
Set permission dasar:
```
sudo find /srv/http/arteri -type d -exec chmod 755 {} \;
sudo find /srv/http/arteri -type f -exec chmod 644 {} \;
```
Set permission folder yang perlu ditulis aplikasi:
```
sudo chmod -R 775 /srv/http/arteri/files
sudo chown -R http:http /srv/http/arteri/files
```
———

## 9. Import Database Arteri

Import struktur database bawaan Arteri:
```
sudo mariadb -u arteriuser -p arteri < /srv/http/arteri/sql/arteri.sql
```
Masukkan password:
```
GantiPasswordKuat
```
Jika ingin menggunakan sample data:
```
sudo nano /srv/http/arteri/application/config/database.php
```
———

## 10. Konfigurasi Database SLiMS

Masuk folder config:
```
cd /srv/http/slims/config
```
Copy file sample:
```
sudo cp database.sample.php database.php
sudo cp env.sample.php env.php
```
Edit database:
```
sudo nano database.php
```
Contoh isi konfigurasi:
```
<?php
return [
    'default_profile' => 'SLiMS',
    'proxy' => false,
    'nodes' => [
        'SLiMS' => [
            'host' => 'localhost',
            'database' => 'senayan',
            'port' => '3306',
            'username' => 'slimsuser',
            'password' => 'GantiPasswordKuat',
            'options' => [
                'storage_engine' => 'MyISAM'
            ]
        ]
    ]
];
```
Edit environment:
```
sudo nano env.php
```
Isi:
```
<?php
$env = 'production';
$conditional_environment = 'production';
$based_on_ip = false;
$range_ip = [''];
```
Set permission config:
```
sudo chown http:http /srv/http/slims/config/database.php /srv/http/
slims/config/env.php
sudo chmod 640 /srv/http/slims/config/database.php /srv/http/slims/
config/env.php
```
Restart service:
```
sudo systemctl restart php-fpm
sudo systemctl restart httpd
```
———

## 11. Konfigurasi Firewalld

Aktifkan firewalld:
```
sudo systemctl enable --now firewalld
```
Cek zone aktif:
```
sudo firewall-cmd --get-active-zones
```
Buka akses HTTP untuk jaringan lokal:
```
sudo firewall-cmd --permanent --add-service=http
```
Jika server hanya boleh diakses dari subnet lokal tertentu, contoh
subnet 192.168.1.0/24:
```
sudo firewall-cmd --permanent --new-zone=local-slims
sudo firewall-cmd --permanent --zone=local-slims --add-
source=192.168.1.0/24
sudo firewall-cmd --permanent --zone=local-slims --add-service=http
```
Reload firewall:
```
sudo firewall-cmd --reload
```
Cek hasil:
```
sudo firewall-cmd --list-all
sudo firewall-cmd --zone=local-slims --list-all
```
> Karena instalasi tidak memakai SSH, service SSH tidak perlu dibuka
> jika memang tidak digunakan.

———

## 12. Akses SLiMS dari Jaringan Lokal

Cek IP server:
```
ip addr
```
Misalnya IP server:
```
192.168.1.10
```
Akses dari browser client lokal:
```
http://192.168.1.10
```
———

## 13. Disable Kernel Module yang Tidak Diperlukan

Buat file blacklist CIS hardening:

sudo nano /etc/modprobe.d/cis-blacklist.conf

Isi:

install cramfs /bin/false
blacklist cramfs

install freevxfs /bin/false
blacklist freevxfs

install hfs /bin/false
blacklist hfs

install hfsplus /bin/false
blacklist hfsplus

install jffs2 /bin/false
blacklist jffs2

install squashfs /bin/false
blacklist squashfs

install udf /bin/false
blacklist udf

install usb-storage /bin/false
blacklist usb-storage

install dccp /bin/false
blacklist dccp

install sctp /bin/false
blacklist sctp

install rds /bin/false
blacklist rds

install tipc /bin/false
blacklist tipc

Apply dengan reboot:

sudo reboot

Cek module:

lsmod | grep -E 'cramfs|freevxfs|hfs|hfsplus|jffs2|squashfs|udf|
usb_storage|dccp|sctp|rds|tipc'

Jika tidak ada output, module tidak aktif.

———

## 14. Hardening Kernel dengan Sysctl

Buat konfigurasi sysctl:

sudo nano /etc/sysctl.d/99-cis-hardening.conf

Isi:

# IP spoofing protection
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable source routed packets
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Disable ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Disable secure ICMP redirects
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0

# Log suspicious packets
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Ignore broadcast ICMP
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ignore bogus ICMP errors
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Enable SYN flood protection
net.ipv4.tcp_syncookies = 1

# Disable IP forwarding
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Kernel pointer restriction
kernel.kptr_restrict = 2

# Restrict dmesg
kernel.dmesg_restrict = 1

# Restrict ptrace
kernel.yama.ptrace_scope = 1

# ASLR
kernel.randomize_va_space = 2

# Hardlink and symlink protection
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.protected_fifos = 2
fs.protected_regular = 2

Apply:

sudo sysctl --system

Cek beberapa nilai:

sysctl net.ipv4.tcp_syncookies
sysctl kernel.randomize_va_space
sysctl kernel.dmesg_restrict

———

## 15. Hardening Mount Point Sementara

Edit /etc/fstab:

sudo nano /etc/fstab

Tambahkan atau sesuaikan mount option untuk /tmp:

tmpfs /tmp tmpfs defaults,rw,nosuid,nodev,noexec,relatime 0 0

Apply:

sudo systemctl daemon-reload
sudo mount -o remount /tmp

Cek:

findmnt /tmp

———

## 16. Hardening Service Apache

Sembunyikan informasi versi Apache.

Edit:

sudo nano /etc/httpd/conf/extra/httpd-default.conf

Pastikan nilai:

ServerTokens Prod
ServerSignature Off
TraceEnable Off

Pastikan file tersebut di-include di /etc/httpd/conf/httpd.conf:

Include conf/extra/httpd-default.conf

Restart Apache:

sudo systemctl restart httpd

———

## 17. Hardening PHP

Edit:

sudo nano /etc/php/php.ini

Set nilai berikut:

expose_php = Off
display_errors = Off
log_errors = On
allow_url_fopen = Off
allow_url_include = Off
session.cookie_httponly = 1
session.use_strict_mode = 1

Restart:

sudo systemctl restart php-fpm
sudo systemctl restart httpd

———

## 18. Audit Service yang Berjalan

Cek service aktif:

systemctl --type=service --state=running

Cek port listening:

ss -tulpn

Minimal yang dibutuhkan:

httpd
php-fpm
mariadb
firewalld

Jika ada service tidak diperlukan, disable:

sudo systemctl disable --now nama-service

———

## 19. Verifikasi Akhir

Cek kernel:

uname -r

Harus mengandung:

hardened

Cek Apache:

systemctl is-active httpd

Cek PHP-FPM:

systemctl is-active php-fpm

Cek MariaDB:

systemctl is-active mariadb

Cek firewall:

systemctl is-active firewalld
sudo firewall-cmd --list-all

Cek akses lokal:

curl -I http://localhost

Dari client jaringan lokal, buka:

http://IP-SERVER

———

## 20. Backup Database SLiMS

Buat folder backup:

sudo mkdir -p /backup/slims

Backup database:

sudo mariadb-dump -u slimsuser -p senayan > /backup/slims/senayan.sql

Backup file aplikasi:

sudo tar czf /backup/slims/slims-files.tar.gz /srv/http/slims/files /
srv/http/slims/images /srv/http/slims/repository

———

## 21. Restore Database SLiMS

Restore database:

sudo mariadb -u slimsuser -p senayan < /backup/slims/senayan.sql

Restore file:

sudo tar xzf /backup/slims/slims-files.tar.gz -C /
sudo chown -R http:http /srv/http/slims/files /srv/http/slims/images /
srv/http/slims/repository

———

## 22. Alur yang Harus Dipahami Presentator

Alur kerja deployment:

1. Server disiapkan secara langsung melalui terminal lokal karena tidak
   ada akses SSH.

2. Sistem diperbarui dan hanya package yang dibutuhkan yang diinstall.
3. Kernel diganti ke linux-hardened untuk memenuhi requirement
   hardening.

4. Apache digunakan sebagai web server.
5. PHP dijalankan melalui PHP-FPM, bukan module PHP Apache.
6. MariaDB digunakan sebagai database backend.
7. Source SLiMS diambil dari GitHub agar sesuai requirement project.
8. Apache diarahkan ke /srv/http/slims.
9. SLiMS dikonfigurasi agar terkoneksi ke database lokal.
10. Firewalld membatasi akses hanya pada service yang dibutuhkan.
11. Kernel module yang tidak diperlukan diblokir.
12. Sysctl digunakan untuk hardening layer kernel dan network.
13. Service, port, firewall, dan akses aplikasi diverifikasi.

———

## 23. Ringkasan Perintah Utama

pacman -Syu
pacman -S --needed git curl wget unzip nano vim sudo
pacman -S --needed linux-hardened linux-hardened-headers
grub-mkconfig -o /boot/grub/grub.cfg
reboot

sudo pacman -S --needed apache php php-fpm php-gd php-intl php-zip
mariadb firewalld
sudo systemctl enable --now httpd php-fpm firewalld
sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/
mysql
sudo systemctl enable --now mariadb

cd /srv/http
sudo git clone https://github.com/slims/slims9_bulian.git slims
sudo chown -R http:http /srv/http/slims

sudo systemctl restart php-fpm
sudo systemctl restart httpd

sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload

———

## Kesimpulan

Server SLiMS berhasil dideploy secara native menggunakan Apache + PHP-
FPM + MariaDB, berjalan di atas kernel linux-hardened, dapat diakses
dari jaringan lokal, dan sudah menerapkan hardening dasar berbasis
prinsip CIS seperti firewall, disable module kernel, sysctl hardening,
pembatasan service, serta hardening Apache dan PHP.
