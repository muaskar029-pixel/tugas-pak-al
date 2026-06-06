# Deployment Native Arteri + CIS Hardening di Arch Linux

Panduan ini menjelaskan proses deployment aplikasi **Arteri — Arsip Elektronik Terintegrasi** secara **native** menggunakan **Apache + PHP-FPM + MariaDB** di Arch Linux dengan kernel **linux-hardened** dan hardening dasar berbasis prinsip CIS Server.

Repository aplikasi:

```text
https://github.com/dicarve/arteri.git
```

---

## Daftar Isi

1. [Skenario](#1-skenario)
2. [Ketentuan Deployment](#2-ketentuan-deployment)
3. [Persiapan Sistem](#3-persiapan-sistem)
4. [Install Kernel Linux Hardened](#4-install-kernel-linux-hardened)
5. [Membuat User Administrator Lokal](#5-membuat-user-administrator-lokal)
6. [Install Apache, PHP-FPM, MariaDB, dan Firewalld](#6-install-apache-php-fpm-mariadb-dan-firewalld)
7. [Konfigurasi PHP Legacy](#7-konfigurasi-php-legacy)
8. [Konfigurasi MariaDB](#8-konfigurasi-mariadb)
9. [Clone Project Arteri](#9-clone-project-arteri)
10. [Import Database Arteri](#10-import-database-arteri)
11. [Konfigurasi Database Arteri](#11-konfigurasi-database-arteri)
12. [Konfigurasi Apache Virtual Host](#12-konfigurasi-apache-virtual-host)
13. [Konfigurasi Firewalld](#13-konfigurasi-firewalld)
14. [Akses Arteri dari Jaringan Lokal](#14-akses-arteri-dari-jaringan-lokal)
15. [Disable Kernel Module yang Tidak Diperlukan](#15-disable-kernel-module-yang-tidak-diperlukan)
16. [Hardening Kernel dengan Sysctl](#16-hardening-kernel-dengan-sysctl)
17. [Hardening Mount Point `/tmp`](#17-hardening-mount-point-tmp)
18. [Hardening Apache](#18-hardening-apache)
19. [Audit Service dan Port](#19-audit-service-dan-port)
20. [Verifikasi Akhir](#20-verifikasi-akhir)
21. [Backup dan Restore](#21-backup-dan-restore)
22. [Alur yang Harus Dipahami Presentator](#22-alur-yang-harus-dipahami-presentator)
23. [Kesimpulan](#23-kesimpulan)

---

## 1. Skenario

Anda diminta oleh atasan untuk mempersiapkan server aplikasi arsip/perpustakaan menggunakan aplikasi **Arteri**.

Deployment dilakukan secara langsung melalui terminal server karena tidak diberikan akses SSH. Server harus berjalan secara native tanpa Docker, Podman, LXC, container, ataupun emulasi.

---

## 2. Ketentuan Deployment

Ketentuan yang digunakan:

- Aplikasi: **Arteri**
- Web server: **Apache**
- PHP handler: **PHP-FPM**
- Database: **MariaDB**
- Firewall: **firewalld**
- Kernel: **linux-hardened**
- Sistem operasi: **Arch Linux**
- Metode deployment: **CLI native**
- Akses aplikasi: **jaringan lokal**
- Source aplikasi: **GitHub project**
- Hardening: **adaptasi prinsip CIS Server**
- Tidak menggunakan SSH saat instalasi
- Tidak menggunakan Docker, Podman, LXC, container, atau emulasi
- Tidak menggunakan meta package `base-devel`

---

## 3. Persiapan Sistem

Login langsung ke terminal server sebagai `root` atau user dengan akses `sudo`.

Cek identitas sistem:

```bash
whoami
hostnamectl
uname -a
```

Update sistem:

```bash
sudo pacman -Syu
```

Install tool dasar tanpa `base-devel`:

```bash
sudo pacman -S --needed git curl wget unzip nano vim sudo
```

> [!IMPORTANT]
> Jangan install `base-devel` karena dilarang dalam kriteria tugas.

---

## 4. Install Kernel Linux Hardened

Install kernel hardened:

```bash
sudo pacman -S --needed linux-hardened linux-hardened-headers
```

Jika menggunakan GRUB, update konfigurasi bootloader:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Reboot server:

```bash
sudo reboot
```

Setelah reboot, cek kernel:

```bash
uname -r
```

Pastikan output mengandung kata:

```text
hardened
```

Contoh:

```text
6.x.x-hardened
```

---

## 5. Membuat User Administrator Lokal

Buat user administrator lokal:

```bash
sudo useradd -m -G wheel -s /bin/bash adminserver
sudo passwd adminserver
```

Aktifkan akses sudo untuk grup `wheel`:

```bash
sudo EDITOR=nano visudo
```

Cari baris berikut:

```text
# %wheel ALL=(ALL:ALL) ALL
```

Ubah menjadi:

```text
%wheel ALL=(ALL:ALL) ALL
```

Tes akses sudo:

```bash
su - adminserver
sudo whoami
```

Jika output-nya:

```text
root
```

berarti konfigurasi sudo berhasil.

---

## 6. Install Apache, PHP-FPM, MariaDB, dan Firewalld

Arteri merupakan aplikasi PHP lama berbasis CodeIgniter. Karena itu, pada Arch Linux lebih aman menggunakan `php-legacy`.

Install Apache, MariaDB, dan firewalld:

```bash
sudo pacman -S --needed apache mariadb firewalld
```

Install PHP legacy dan PHP-FPM legacy:

```bash
sudo pacman -S --needed php-legacy php-legacy-fpm php-legacy-gd
```

Aktifkan service utama:

```bash
sudo systemctl enable --now httpd
sudo systemctl enable --now mariadb
sudo systemctl enable --now firewalld
sudo systemctl enable --now php-fpm-legacy
```

Cek status service:

```bash
systemctl status httpd
systemctl status mariadb
systemctl status firewalld
systemctl status php-fpm-legacy
```

Jika service PHP-FPM legacy tidak ditemukan, cek nama unit PHP:

```bash
systemctl list-unit-files | grep -i php
```

---

## 7. Konfigurasi PHP Legacy

Edit konfigurasi PHP legacy:

```bash
sudo nano /etc/php-legacy/php.ini
```

Aktifkan extension berikut dengan menghapus tanda `;` jika masih dinonaktifkan:

```ini
extension=mysqli
extension=pdo_mysql
extension=gd
extension=zip
extension=mbstring
```

Sesuaikan konfigurasi upload dan timezone:

```ini
file_uploads = On
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 300
date.timezone = Asia/Jakarta
```

Tambahkan hardening dasar PHP:

```ini
expose_php = Off
display_errors = Off
log_errors = On
allow_url_include = Off
session.cookie_httponly = 1
session.use_strict_mode = 1
```

Restart PHP-FPM:

```bash
sudo systemctl restart php-fpm-legacy
```

Cek module PHP:

```bash
php-legacy -m | grep -E 'mysqli|pdo_mysql|gd|zip|mbstring'
```

---

## 8. Konfigurasi MariaDB

Jika MariaDB belum pernah diinisialisasi, jalankan:

```bash
sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
```

Aktifkan MariaDB:

```bash
sudo systemctl enable --now mariadb
```

Amankan instalasi MariaDB:

```bash
sudo mariadb-secure-installation
```

Rekomendasi jawaban:

```text
Switch to unix_socket authentication: Y
Change the root password: Y
Remove anonymous users: Y
Disallow root login remotely: Y
Remove test database: Y
Reload privilege tables: Y
```

Masuk ke MariaDB:

```bash
sudo mariadb
```

Buat database dan user untuk Arteri:

```sql
CREATE DATABASE arteri CHARACTER SET utf8 COLLATE utf8_unicode_ci;
CREATE USER 'arteriuser'@'localhost' IDENTIFIED BY 'GantiPasswordKuat';
GRANT ALL PRIVILEGES ON arteri.* TO 'arteriuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

> [!WARNING]
> Ganti `GantiPasswordKuat` dengan password yang kuat.

---

## 9. Clone Project Arteri

Masuk ke direktori web server:

```bash
cd /srv/http
```

Clone repository Arteri:

```bash
sudo git clone https://github.com/dicarve/arteri.git arteri
```

Masuk ke folder Arteri:

```bash
cd /srv/http/arteri
```

Set ownership ke user Apache di Arch Linux, yaitu `http`:

```bash
sudo chown -R http:http /srv/http/arteri
```

Set permission dasar:

```bash
sudo find /srv/http/arteri -type d -exec chmod 755 {} \;
sudo find /srv/http/arteri -type f -exec chmod 644 {} \;
```

Beri permission tulis untuk folder `files`:

```bash
sudo chmod -R 775 /srv/http/arteri/files
sudo chown -R http:http /srv/http/arteri/files
```

---

## 10. Import Database Arteri

Repository Arteri menyediakan file SQL di folder:

```text
/srv/http/arteri/sql/arteri.sql
```

Import database:

```bash
sudo mariadb -u arteriuser -p arteri < /srv/http/arteri/sql/arteri.sql
```

Masukkan password database:

```text
GantiPasswordKuat
```

Cek tabel:

```bash
sudo mariadb -u arteriuser -p -e "USE arteri; SHOW TABLES;"
```

---

## 11. Konfigurasi Database Arteri

File konfigurasi database Arteri berada di:

```text
/srv/http/arteri/application/config/database.php
```

Edit file:

```bash
sudo nano /srv/http/arteri/application/config/database.php
```

Cari bagian konfigurasi database:

```php
'hostname' => 'localhost',
'username' => 'root',
'password' => '',
'database' => 'arteri',
'dbdriver' => 'mysqli',
```

Ubah menjadi:

```php
'hostname' => 'localhost',
'username' => 'arteriuser',
'password' => 'GantiPasswordKuat',
'database' => 'arteri',
'dbdriver' => 'mysqli',
```

Set permission file konfigurasi:

```bash
sudo chown http:http /srv/http/arteri/application/config/database.php
sudo chmod 640 /srv/http/arteri/application/config/database.php
```

---

## 12. Konfigurasi Apache Virtual Host

Edit konfigurasi utama Apache:

```bash
sudo nano /etc/httpd/conf/httpd.conf
```

Pastikan module berikut aktif:

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
LoadModule rewrite_module modules/mod_rewrite.so
```

Aktifkan konfigurasi virtual host:

```apache
Include conf/extra/httpd-vhosts.conf
```

Cek socket PHP-FPM legacy:

```bash
grep -E '^listen' /etc/php-legacy/php-fpm.d/www.conf
```

Biasanya socket PHP-FPM legacy berada di:

```text
/run/php-fpm-legacy/php-fpm.sock
```

Edit file virtual host:

```bash
sudo nano /etc/httpd/conf/extra/httpd-vhosts.conf
```

Isi konfigurasi berikut:

```apache
<VirtualHost *:80>
    ServerName arteri.local
    DocumentRoot "/srv/http/arteri"

    <Directory "/srv/http/arteri">
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
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

Tes konfigurasi Apache:

```bash
sudo apachectl configtest
```

Jika hasilnya:

```text
Syntax OK
```
jika error
```
sudo nano /etc/httpd/conf/httpd.conf
```
pada ServerName masukan port server
restart service:

```bash
sudo systemctl restart php-fpm-legacy
sudo systemctl restart httpd
```

---

## 13. Konfigurasi Firewalld

Aktifkan firewalld:

```bash
sudo systemctl enable --now firewalld
```

Cek zone aktif:

```bash
sudo firewall-cmd --get-active-zones
```

Buka akses HTTP:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

Jika server hanya boleh diakses dari jaringan lokal tertentu, contoh `192.168.1.0/24`:

```bash
sudo firewall-cmd --permanent --new-zone=local-arteri
sudo firewall-cmd --permanent --zone=local-arteri --add-source=192.168.1.0/24
sudo firewall-cmd --permanent --zone=local-arteri --add-service=http
sudo firewall-cmd --reload
```

Cek konfigurasi firewall:

```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --zone=local-arteri --list-all
```

> [!NOTE]
> Karena instalasi tidak menggunakan SSH, port SSH tidak perlu dibuka.

---

## 14. Akses Arteri dari Jaringan Lokal

Cek IP server:

```bash
ip addr
```

Contoh IP server:

```text
192.168.1.10
```

Akses dari browser client lokal:

```text
http://192.168.1.10
```

Login default Arteri:

```text
Username: admin
Password: admin
```

> [!WARNING]
> Setelah berhasil login, segera ganti password default admin.

---

## 15. Disable Kernel Module yang Tidak Diperlukan

Buat file blacklist:

```bash
sudo nano /etc/modprobe.d/cis-blacklist.conf
```

Isi:

```conf
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
```

Reboot:

```bash
sudo reboot
```

Cek module:

```bash
lsmod | grep -E 'cramfs|freevxfs|hfs|hfsplus|jffs2|squashfs|udf|usb_storage|dccp|sctp|rds|tipc'
```

Jika tidak ada output, module tidak aktif.

---

## 16. Hardening Kernel dengan Sysctl

Buat file konfigurasi sysctl:

```bash
sudo nano /etc/sysctl.d/99-cis-hardening.conf
```

Isi:

```conf
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
```

Apply konfigurasi:

```bash
sudo sysctl --system
```

Cek hasil:

```bash
sysctl net.ipv4.tcp_syncookies
sysctl kernel.randomize_va_space
sysctl kernel.dmesg_restrict
```

---

## 17. Hardening Mount Point `/tmp`

Edit `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Tambahkan konfigurasi berikut:

```fstab
tmpfs /tmp tmpfs defaults,rw,nosuid,nodev,noexec,relatime 0 0
```

Apply:

```bash
sudo systemctl daemon-reload
sudo mount -o remount /tmp
```

Cek:

```bash
findmnt /tmp
```

---

## 18. Hardening Apache

Edit konfigurasi default Apache:

```bash
sudo nano /etc/httpd/conf/extra/httpd-default.conf
```

Pastikan nilai berikut:

```apache
ServerTokens Prod
ServerSignature Off
TraceEnable Off
```

Pastikan file tersebut di-include di `/etc/httpd/conf/httpd.conf`:

```apache
Include conf/extra/httpd-default.conf
```

Restart Apache:

```bash
sudo systemctl restart httpd
```

---

## 19. Audit Service dan Port

Cek service yang berjalan:

```bash
systemctl --type=service --state=running
```

Cek port listening:

```bash
ss -tulpn
```

Service minimal yang dibutuhkan:

```text
httpd
php-fpm-legacy
mariadb
firewalld
```

Jika ada service tidak diperlukan, disable:

```bash
sudo systemctl disable --now nama-service
```

---

## 20. Verifikasi Akhir

Cek kernel:

```bash
uname -r
```

Harus mengandung:

```text
hardened
```

Cek service:

```bash
systemctl is-active httpd
systemctl is-active mariadb
systemctl is-active firewalld
systemctl is-active php-fpm-legacy
```

Cek PHP:

```bash
php-legacy -v
php-legacy -m | grep -E 'mysqli|pdo_mysql|gd'
```

Cek Apache:

```bash
curl -I http://localhost
```

Cek firewall:

```bash
sudo firewall-cmd --list-all
```

Cek database:

```bash
sudo mariadb -u arteriuser -p -e "USE arteri; SHOW TABLES;"
```

Cek akses aplikasi dari jaringan lokal:

```text
http://IP-SERVER
```

---

## 21. Backup dan Restore

### Backup Database

Buat folder backup:

```bash
sudo mkdir -p /backup/arteri
```

Backup database:

```bash
sudo mariadb-dump -u arteriuser -p arteri > /backup/arteri/arteri.sql
```

Backup file aplikasi:

```bash
sudo tar czf /backup/arteri/arteri-files.tar.gz /srv/http/arteri
```

### Restore Database

Restore database:

```bash
sudo mariadb -u arteriuser -p arteri < /backup/arteri/arteri.sql
```

Restore file aplikasi:

```bash
sudo tar xzf /backup/arteri/arteri-files.tar.gz -C /
sudo chown -R http:http /srv/http/arteri
```

Restart service:

```bash
sudo systemctl restart php-fpm-legacy
sudo systemctl restart httpd
```

---

## 22. Alur yang Harus Dipahami Presentator

Alur kerja deployment:

1. Server dikonfigurasi secara langsung melalui terminal lokal karena tidak menggunakan SSH.
2. Sistem diperbarui dan hanya package yang dibutuhkan yang diinstall.
3. Kernel diganti ke `linux-hardened` untuk memenuhi requirement hardening.
4. Apache digunakan sebagai web server.
5. PHP dijalankan melalui PHP-FPM legacy.
6. MariaDB digunakan sebagai database backend.
7. Source Arteri diambil dari GitHub.
8. File SQL Arteri diimport ke database MariaDB.
9. Konfigurasi database diatur melalui file `application/config/database.php`.
10. Apache diarahkan ke direktori `/srv/http/arteri`.
11. Direktori sensitif seperti `application`, `system`, dan `sql` diblokir dari akses web langsung.
12. Firewalld hanya membuka service HTTP.
13. Kernel module yang tidak diperlukan diblokir.
14. Sysctl digunakan untuk hardening kernel dan jaringan.
15. Service, port, firewall, database, dan akses aplikasi diverifikasi.

---

## 23. Kesimpulan

Server **Arteri** berhasil dideploy secara native menggunakan **Apache + PHP-FPM Legacy + MariaDB** di atas **Arch Linux linux-hardened**.

Deployment ini memenuhi kriteria:

- Tidak menggunakan Docker, Podman, LXC, container, atau emulasi.
- Tidak menggunakan SSH saat instalasi.
- Tidak menggunakan package meta `base-devel`.
- Menggunakan Apache + PHP-FPM.
- Menggunakan MariaDB sebagai database.
- Menggunakan firewalld untuk kontrol akses jaringan.
- Menggunakan linux-hardened.
- Menerapkan hardening dasar berbasis prinsip CIS.
- Aplikasi dapat diakses dari jaringan lokal.

Dengan demikian, Arteri dapat berjalan sebagai aplikasi arsip elektronik/perpustakaan berbasis web yang dideploy secara native dan lebih aman untuk kebutuhan server lokal.
