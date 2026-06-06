
download packet
```
pacman -Syu
pacman -S --needed git curl wget unzip nano vim sudo
pacman -S apache
```
Install Apache, PHP-FPM, dan MariaDB
```
sudo pacman -S --needed apache php php-fpm php-gd php-intl php-zip mariadb firewalld
```
Aktifkan service
```
sudo systemctl enable --now httpd
sudo systemctl enable --now firewalld
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


