
download packet
```
pacman -Syu
pacman -S --needed git curl wget unzip nano vim sudo
pacman -S apache
```
Install Apache, PHP-FPM, dan MariaDB
```
sudo pacman -S --needed apache php php-fpm php-gd php-intl php-zip mariadb firewalld git nano curl unzip
```
Aktifkan service
```
sudo systemctl enable --now httpd
sudo systemctl enable --now firewalld
sudo systemctl enable --now php-fpm
```
Konfigurasi apache untuk file terletak di
```
/etc/httpd/conf
```

Konfigurasi apache untuk file utama terletak di
```
/etc/httpd/conf/httpd.conf
```
secara default dia akan mengkofigurasi folder di
```
/srv/http
```

These options in /etc/httpd/conf/httpd.conf might be interesting for you:
```
Listen 80
```
masukan ```127.0.0.1:80``` jika ingin local development yang hanya bisa diakses lewat computer

```
"/srv/http"
```
masukan folder web di sini


php-fpm


cek
```
systemctl is-active php-fpm
```
buka
```
sudo nano /etc/php/php.ini
```
Cari dengan Ctrl + W.
```
extension=mysqli
extension=pdo_mysql
extension=gd
extension=zip
extension=mbstring
```
Sesuaikan konfigurasi upload dan timezone:
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
cari
aktifkan module untuk proxy


<img width="492" height="126" alt="image" src="https://github.com/user-attachments/assets/60bd6bcf-88d1-497b-b44c-4dd774ebcdc6" />

Create /etc/httpd/conf/extra/php-fpm.conf with the following content:
```
DirectoryIndex index.php index.html
<FilesMatch \.php$>
    SetHandler "proxy:unix:/run/php-fpm/php-fpm.sock|fcgi://localhost/"
</FilesMatch>
```
And include it at the bottom of /etc/httpd/conf/httpd.conf:
```
Include conf/extra/php-fpm.conf
```

```
sudo systemctl enable --now php-fpm
```
restart apache


Install dan Amankan MariaDB

```
sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/
mysql
```
Aktifkan MariaDB:
```
sudo systemctl enable mariadb
```
Amankan instalasi MariaDB:
```
sudo mariadb-secure-installation
```
```
sudo mariadb
```
buat database dan user arteri
```
CREATE DATABASE arsipDigital CHARACTER SET utf8mb4 COLLATE
utf8mb4_unicode_ci;
CREATE USER 'arteriuser'@'localhost' IDENTIFIED BY 'GantiPasswordKuat';
GRANT ALL PRIVILEGES ON arsipDigital.* TO 'arteriuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
Ambil Project arteri dari GitHub
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
Import struktur database bawaan Arteri:
```
sudo mariadb -u arteriuser -p arteri < /srv/http/arteri/sql/arteri.sql
```
Masukkan password:
```
GantiPasswordKuat
```
Edit database config Arteri
```
sudo nano /srv/http/arteri/application/config/database.php
```
Isi bagian pentingnya harus seperti ini:
```
$active_group = 'default';
$query_builder = TRUE;

$db['default'] = array(
    'dsn'      => '',
    'hostname' => 'localhost',
    'username' => 'arteriuser',
    'password' => 'GantiPasswordKuat',
    'database' => 'arteri',
    'dbdriver' => 'mysqli',
    'dbprefix' => '',
    'pconnect' => FALSE,
    'db_debug' => (ENVIRONMENT !== 'production'),
    'cache_on' => FALSE,
    'cachedir' => '',
    'char_set' => 'utf8',
    'dbcollat' => 'utf8_unicode_ci',
    'swap_pre' => '',
    'encrypt' => FALSE,
    'compress' => FALSE,
    'stricton' => FALSE,
    'failover' => array(),
    'save_queries' => TRUE
);
```
set permission
```
sudo chown http:http /srv/http/arteri/application/config/database.php
```
```
sudo chmod 640 /srv/http/arteri/application/config/database.php
```
