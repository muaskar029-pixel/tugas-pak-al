# Panduan Instalasi Arteri di Arch Linux

Panduan ini menjelaskan instalasi Arteri dari awal menggunakan Apache, PHP 8.3 legacy FPM, MariaDB, dan firewalld. Arteri memakai CodeIgniter 3.1.6 yang tidak cocok langsung dengan PHP 8.5.

Versi yang diuji pada 7 Juni 2026:

```text
Apache 2.4.67 | PHP legacy 8.3.31 | MariaDB 12.3.2
firewalld 2.4.1 | CodeIgniter 3.1.6
```

Paket Arch akan terus berubah. Bagian pentingnya adalah memakai `php-legacy-fpm` 8.3 untuk Arteri, bukan PHP-FPM 8.5.

## 1. Perbarui dan Instal Paket

```bash
sudo pacman -Syu
sudo pacman -S apache mariadb firewalld \
  php-legacy php-legacy-fpm php-legacy-gd
```

Tidak perlu `mod_php`. Apache berkomunikasi dengan PHP melalui `proxy_fcgi` dan socket PHP-FPM.

## 2. Siapkan MariaDB

Jika `/var/lib/mysql` belum diinisialisasi:

```bash
sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
sudo systemctl enable --now mariadb
sudo mariadb-secure-installation
```

Hapus anonymous user dan database test, larang login root jarak jauh, lalu muat ulang privilege.

Karena aplikasi dan database berada pada server yang sama, edit `/etc/my.cnf.d/server.cnf`:

```ini
[mysqld]
bind-address = 127.0.0.1
```

```bash
sudo systemctl restart mariadb
sudo ss -ltnp | grep 3306
```

MariaDB seharusnya mendengarkan pada `127.0.0.1:3306`, bukan `0.0.0.0:3306`.

## 3. Buat Database dan User

```bash
sudo mariadb
```

```sql
CREATE DATABASE arteri CHARACTER SET utf8 COLLATE utf8_unicode_ci;
CREATE USER 'arteri_user'@'localhost'
  IDENTIFIED BY 'GANTI_DENGAN_PASSWORD_KUAT';
GRANT ALL PRIVILEGES ON arteri.* TO 'arteri_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Jangan memakai user MariaDB `root` pada aplikasi.

## 4. Tempatkan Source Arteri

Struktur akhirnya:

```text
/srv/http/arteri/
├── application/
├── files/
├── public/
├── sql/
├── system/
└── index.php
```

Contoh ekstraksi:

```bash
sudo mkdir -p /srv/http/arteri
sudo bsdtar -xf arteri.tar.gz -C /srv/http/arteri --strip-components=1
```

Sesuaikan `--strip-components` dengan struktur arsip. Atur permission:

```bash
sudo chown -R root:http /srv/http/arteri
sudo find /srv/http/arteri -type d -exec chmod 750 {} +
sudo find /srv/http/arteri -type f -exec chmod 640 {} +

sudo chown -R http:http \
  /srv/http/arteri/application/cache \
  /srv/http/arteri/application/logs \
  /srv/http/arteri/files
sudo chmod 750 \
  /srv/http/arteri/application/cache \
  /srv/http/arteri/application/logs \
  /srv/http/arteri/files
```

Hanya direktori runtime yang perlu dapat ditulis oleh PHP-FPM.

## 5. Import Database

```bash
mariadb -u arteri_user -p arteri < /srv/http/arteri/sql/arteri.sql
mariadb -u arteri_user -p -D arteri -e "SHOW TABLES;"
```

Tabel yang semestinya tersedia antara lain `data_arsip`, `master_kode`, `master_lokasi`, `master_media`, `master_pencipta`, `master_pengolah`, `master_user`, `sirkulasi`, dan `system_log`.

## 6. Konfigurasi Database CodeIgniter

Edit `/srv/http/arteri/application/config/database.php`:

```php
$db['default'] = array(
    'dsn'      => '',
    'hostname' => 'localhost',
    'username' => 'arteri_user',
    'password' => 'GANTI_DENGAN_PASSWORD_KUAT',
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

```bash
sudo chown root:http /srv/http/arteri/application/config/database.php
sudo chmod 640 /srv/http/arteri/application/config/database.php
```

## 7. Patch Kompatibilitas CodeIgniter 3.1.6

### Dynamic property PHP 8.3

Pada `/srv/http/arteri/index.php`, ubah:

```php
case 'development':
    error_reporting(-1);
    ini_set('display_errors', 1);
break;
```

menjadi:

```php
case 'development':
    error_reporting(E_ALL & ~E_DEPRECATED & ~E_USER_DEPRECATED);
    ini_set('display_errors', 1);
break;
```

CodeIgniter 3.1.6 dibuat sebelum PHP 8.2. Tanpa patch ini, notifikasi `Creation of dynamic property ... is deprecated` tercetak sebelum session dan redirect, lalu memicu `headers already sent`.

### Path session

Pada `/srv/http/arteri/application/config/config.php`, ubah:

```php
$config['sess_save_path'] = NULL;
```

menjadi:

```php
$config['sess_save_path'] = APPPATH.'cache';
```

```bash
sudo chown http:http /srv/http/arteri/application/cache
sudo chmod 750 /srv/http/arteri/application/cache
```

## 8. Konfigurasi PHP 8.3 Legacy

Edit `/etc/php-legacy/php.ini`. Aktifkan tanpa tanda `;`:

```ini
extension=curl
extension=gd
extension=mysqli
extension=pdo_mysql
extension=zip
```

Konfigurasi produksi yang disarankan:

```ini
display_errors = Off
log_errors = On
expose_php = Off
date.timezone = Asia/Jakarta
upload_max_filesize = 32M
post_max_size = 32M
max_execution_time = 120
```

```bash
php-legacy -m | grep -E 'curl|gd|mysqli|pdo_mysql|mbstring|fileinfo|session|zip'
php-legacy -l /srv/http/arteri/index.php
```

## 9. Konfigurasi PHP-FPM

Edit `/etc/php-legacy/php-fpm.d/www.conf`:

```ini
user = http
group = http
listen = /run/php-fpm-legacy/php-fpm.sock
listen.owner = http
listen.group = http
listen.mode = 0660

pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

```bash
sudo systemctl disable --now php-fpm 2>/dev/null || true
sudo systemctl enable --now php-fpm-legacy
sudo systemctl status php-fpm-legacy
ls -l /run/php-fpm-legacy/php-fpm.sock
```

## 10. Konfigurasi Apache

Edit `/etc/httpd/conf/httpd.conf`. Pastikan MPM Event dan modul berikut aktif:

```apache
LoadModule mpm_event_module modules/mod_mpm_event.so
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
LoadModule rewrite_module modules/mod_rewrite.so
ServerName localhost:80
Include conf/extra/httpd-vhosts.conf
```

Hanya satu MPM boleh aktif. Jangan menambahkan handler PHP global jika server juga melayani aplikasi dengan versi PHP lain.

Tambahkan ke `/etc/httpd/conf/extra/httpd-vhosts.conf`:

```apache
<VirtualHost *:80>
    ServerName arteri.local
    DocumentRoot "/srv/http/arteri"

    <Directory "/srv/http/arteri">
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
        DirectoryIndex index.php index.html
    </Directory>

    <DirectoryMatch "^/srv/http/arteri/(application|system|sql)">
        Require all denied
    </DirectoryMatch>

    <FilesMatch "\.php$">
        SetHandler "proxy:unix:/run/php-fpm-legacy/php-fpm.sock|fcgi://localhost/"
    </FilesMatch>

    ErrorLog "/var/log/httpd/arteri-error.log"
    CustomLog "/var/log/httpd/arteri-access.log" combined
</VirtualHost>
```

Slash setelah `arteri/` pada `DirectoryMatch` wajib ada. Tanpanya folder `sql` dapat terbuka melalui web.

```bash
sudo apachectl configtest
sudo systemctl enable --now httpd
sudo systemctl reload httpd
```

Hasil validasi harus `Syntax OK`.

## 11. Nama Host

Pada server lokal, tambahkan ke `/etc/hosts`:

```text
127.0.0.1 arteri.local
```

Pada komputer klien LAN:

```text
IP_SERVER arteri.local
```

Contoh: `192.168.100.10 arteri.local`. Untuk banyak klien, gunakan DNS internal.

## 12. Konfigurasi firewalld

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

Jika HTTPS sudah dikonfigurasi:

```bash
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

Jangan buka MariaDB ke jaringan:

```bash
sudo firewall-cmd --permanent --remove-service=mysql
sudo firewall-cmd --permanent --remove-port=3306/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

Untuk LAN dasar, services cukup berisi `ssh http`, ditambah `https` jika TLS aktif. Port 3306 tidak boleh muncul.

## 13. Restart dan Verifikasi

```bash
sudo systemctl restart mariadb
sudo systemctl restart php-fpm-legacy
sudo systemctl restart httpd
systemctl is-active mariadb php-fpm-legacy httpd firewalld
sudo ss -ltnp | grep -E ':(80|443|3306)\b'
```

Semua layanan harus `active`. MariaDB hanya boleh mendengarkan pada localhost.

## 14. Pengujian HTTP

```bash
curl -I -H 'Host: arteri.local' http://127.0.0.1/
curl -I http://arteri.local/index.php/home/login
```

Uji direktori sensitif:

```bash
for path in application/config/database.php system/core/CodeIgniter.php sql/arteri.sql; do
  curl -o /dev/null -w "%{http_code} $path\n" "http://arteri.local/$path"
done
```

Ketiganya harus HTTP `403`.

Buka `http://arteri.local/index.php/home/login`. Kredensial bawaan dump adalah `admin/admin`. Segera ganti password default setelah login.

Setelah login, uji daftar arsip, pencarian, tambah/edit, upload/download, user, logout, dan persistensi session.

## 15. Log dan Troubleshooting

```bash
sudo tail -f /var/log/httpd/arteri-error.log
sudo journalctl -u httpd -u php-fpm-legacy -f
```

### Source PHP tampil sebagai teks

`proxy_fcgi` atau handler FPM belum aktif:

```bash
apachectl -M | grep -E 'proxy|proxy_fcgi'
systemctl status php-fpm-legacy
ls -l /run/php-fpm-legacy/php-fpm.sock
```

### HTTP 503 atau `Primary script unknown`

Cocokkan socket `/run/php-fpm-legacy/php-fpm.sock`, `DocumentRoot`, dan permission parent directory.

### `Creation of dynamic property ... is deprecated`

Terapkan filter `E_DEPRECATED` pada `index.php` seperti bagian 7.

### `Session cannot be started after headers have already been sent`

Biasanya deprecation sudah tercetak. Pastikan patch diterapkan dan tidak ada output sebelum `<?php`.

### `Session: Configured save path '' is not a directory`

Gunakan `APPPATH.'cache'` dan pastikan direktori dimiliki `http:http`.

### Database tidak terhubung

```bash
systemctl status mariadb
mariadb -u arteri_user -p -D arteri -e "SELECT 1;"
```

Cocokkan hostname, username, password, dan database pada `database.php`.

### Halaman selain index menghasilkan 404

Arteri memakai `$config['index_page'] = 'index.php'`, sehingga URL berbentuk `/index.php/controller/method`. Penghapusan `index.php` memerlukan rewrite tambahan.

### GD tidak tersedia

```bash
sudo pacman -S php-legacy-gd
grep '^extension=gd' /etc/php-legacy/php.ini
php-legacy -m | grep '^gd$'
sudo systemctl restart php-fpm-legacy
```

## 16. Checklist Produksi

- Ganti password login `admin`.
- Gunakan password database unik dan kuat.
- Batasi MariaDB ke `127.0.0.1` dan jangan buka port 3306.
- Pastikan `application`, `system`, dan `sql` menghasilkan HTTP 403.
- Gunakan `Options -Indexes`.
- Aktifkan HTTPS sebelum menyimpan data nyata.
- Ubah environment ke `production` setelah diagnosis selesai.
- Gunakan `display_errors = Off` dan `log_errors = On`.
- Backup database dan direktori `files` secara rutin.
- Rencanakan upgrade CodeIgniter 3.1.6 ke 3.1.13.

## 17. Backup Dasar

```bash
sudo install -d -m 700 /var/backups/arteri
mariadb-dump -u arteri_user -p arteri \
  | gzip > /var/backups/arteri/arteri-$(date +%F).sql.gz
sudo tar -czf /var/backups/arteri/files-$(date +%F).tar.gz \
  -C /srv/http/arteri files
```

Uji restore secara berkala. Backup yang belum pernah diuji restore belum dapat dianggap aman.
