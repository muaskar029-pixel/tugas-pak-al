# DUAL BOOT ARCH LINUX KEDUA
> Partisi: `nvme0n1p4` sebagai **boot**, `nvme0n1p5` sebagai **LVM disk**

---

# CONNECT WIFI

```bash
iwctl
```
```bash
device list
```
> Catatan: Untuk cek driver wifi setiap laptop

```bash
station (driver wifi) get-network
```
> Catatan: Untuk melihat jaringan yang tersedia

```bash
station (driver wifi) scan
```
> Catatan: Untuk memindai jaringan yang ada

```bash
station {device wifi} connect "{nama wifi}"
```
```bash
exit
```
> Catatan: Untuk menghubungkan ke jaringan yang sudah ditentukan

## Memeriksa Jaringan

```bash
ping 1.1.1.1
```

---

# CHECKING PARTISI

## Melihat partisi beserta type nya
```bash
lsblk -o name,fstype,size
```

## Melihat partisi saja
```bash
lsblk
```

> **Pastikan** `nvme0n1p4` dan `nvme0n1p5` sudah tersedia. Jika belum, buat dulu dengan `cfdisk`.

## Membuat Partisi (jika belum ada)
```bash
cfdisk /dev/nvme0n1
```

### Layout minimal:
```
nvme0n1p4 = 1G   [EFI system]       → boot
nvme0n1p5 = sisa [Linux filesystem]  → LVM disk
```

> Jika partisi sudah ada dari Windows/OS sebelumnya, **JANGAN hapus partisi lain**. Quit tanpa Write jika salah.

```bash
lsblk
```

---

# PARTITION LVM WITH DISK LAYOUT CIS

> Partisi LVM di dalam `nvme0n1p5`

## Setup LVM

```bash
pvcreate /dev/nvme0n1p5
```
```bash
vgcreate proc /dev/nvme0n1p5
```

## Create Logical Volume

```bash
lvcreate -L <size>G proc -n root
```
```bash
lvcreate -L <size>G proc -n vars
```
```bash
lvcreate -L <size>G proc -n vtmp
```
```bash
lvcreate -L <size>G proc -n vlog
```
```bash
lvcreate -L <size>G proc -n vaud
```
```bash
lvcreate -L <size>G proc -n home
```

---

# FORMATTING

```bash
mkfs.ext4 /dev/proc/root
```
```bash
mkfs.vfat -F32 -n BOOT /dev/nvme0n1p4
```
```bash
mkfs.ext4 /dev/proc/vars
```
```bash
mkfs.ext4 /dev/proc/vtmp
```
```bash
mkfs.ext4 /dev/proc/vlog
```
```bash
mkfs.ext4 /dev/proc/vaud
```
```bash
mkfs.ext4 /dev/proc/home
```

---

# MOUNTING

```bash
mount /dev/proc/root /mnt
```
```bash
mount --mkdir -o uid=0,gid=0,dmask=0077,fmask=0077 /dev/nvme0n1p4 /mnt/boot
```
```bash
mount --mkdir -o rw,nodev,nosuid,relatime /dev/proc/vars /mnt/var
```
```bash
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/proc/vtmp /mnt/var/tmp
```
```bash
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/proc/vlog /mnt/var/log
```
```bash
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/proc/vaud /mnt/var/log/audit
```
```bash
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/proc/home /mnt/home
```

---

# SETUP LUKS

```bash
lvcreate -l 50%FREE proc -n [name]
```
```bash
cryptsetup luksFormat /dev/proc/[name]
```

---

# PACKAGES

## Intel
```bash
pacstrap /mnt intel-ucode linux-lts linux-lts-headers linux-firmware lvm2 base base-devel neovim openssh superfile podman podman-desktop iptables mpd mpc mpv keepassxc secrets booster networkmanager pam_mount
```

## AMD
```bash
pacstrap /mnt amd-ucode linux-lts linux-lts-headers linux-firmware lvm2 base base-devel neovim openssh superfile podman podman-desktop iptables mpd mpc mpv keepassxc secrets booster networkmanager pam_mount
```

---

# FSTAB

```bash
genfstab -U /mnt > /mnt/etc/fstab
```

## Formatting tmpfs ke tmp
```bash
echo "/tmpfs /tmp  tmpfs  defaults,nosuid,nodev,noexec,size=1G  0  0" >> /mnt/etc/fstab
```

---

# CHROOT

```bash
arch-chroot /mnt
```

---

# HOSTNAME

```bash
echo [nama komputer] > /etc/hostname
```
> Jika 1 kata tidak perlu `""`, jika lebih gunakan petik `""`

---

# LOCALTIME

```bash
ln -fs /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
```
```bash
hwclock --systohc
```

---

# LOCALE

```bash
nvim /etc/locale.gen
```

Cari dengan `/` di nvim, lalu **uncomment** kedua baris `en_US`.

```bash
locale-gen
```
```bash
locale > /etc/locale.conf
```

## Config locale
```bash
nvim /etc/locale.conf
```

Isi file:
```
LANG=en_US.UTF-8
LC_ALL=en_US.UTF-8
```

---

# PAM_MOUNT

## Buka LUKS
```bash
cryptsetup luksOpen /dev/proc/[name] [nama device]
```
```bash
mkfs.ext4 /dev/mapper/[nama device]
```

## Useradd
```bash
mkdir /home/user
```
```bash
useradd -d /home/user [username]
passwd [username]
```
```bash
chown -R [username]:[username] /home/user
```
```bash
passwd
```
> Password harus sama dengan password LUKS partisi ini

```bash
echo '[username] ALL=(ALL:ALL) ALL' >> /etc/sudoers.d/none
```

## Config Volume
```bash
nvim /etc/security/pam_mount.conf.xml
```

Samakan dengan isi berikut:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE pam_mount SYSTEM "pam_mount.conf.xml.dtd">

<pam_mount>

<debug enable="0" />

<mntoptions allow="nosuid,nodev,loop,encryption,fsck,nonempty,allow_root,allow_other" />
<mntoptions require="nosuid,nodev" />

<logout wait="0" hup="no" term="no" kill="no" />

<!-- Example entry for a LUKS partition -->
<volume 
    user="[username]" 
    fstype="crypt" 
    path="/dev/proc/[name]" 
    mountpoint="/home/[username]" 
/>

<mkmountpoint enable="1" remove="true" />

</pam_mount>
```

## Update Konfigurasi PAM
```bash
nvim /etc/pam.d/system-login
```

Samakan dengan isi berikut:
```
#%PAM-1.0

auth       required   pam_shells.so
auth       requisite  pam_nologin.so
auth       include    system-auth
auth       required   pam_mount.so

account    required   pam_access.so
account    required   pam_nologin.so
account    include    system-auth

password   include    system-auth

session    optional   pam_loginuid.so
session    optional   pam_keyinit.so       force revoke
session    include    system-auth
session    optional   pam_lastlog2.so      silent
session    optional   pam_motd.so
session    optional   pam_mail.so          dir=/var/spool/mail standard quiet
session    optional   pam_umask.so
session    optional   pam_mount.so
-session   optional   pam_systemd.so
session    required   pam_env.so
```

---

# BOOSTER

```bash
nvim /etc/booster.yaml
```

Tambahkan:
```yaml
network:
  dhcp: on
universal: false
modules: -*,ext4
extra_files: fsck,fsck.ext4
strip: true
enable_lvm: true
```

```bash
cd /boot
```

Cek versi kernel:
```bash
ls /usr/lib/modules
```

```bash
booster build --kernel-version <version> /boot/booster-linux-lts-new.img
```
```bash
rm -fr booster-linux-lts.img
```

---

# SYSTEMD-BOOT

```bash
bootctl --path=/boot install
```

> Jika gagal muncul di boot option, keluar dari chroot dulu:
```bash
exit
bootctl --path=/mnt/boot install
arch-chroot /mnt
```

## Boot Entry

```bash
nvim /boot/loader/entries/booster-arch2.conf
```

> **Gunakan nama file berbeda** dari install pertama agar tidak bentrok, contoh `booster-arch2.conf`

```
title    Arch Linux 2 with Booster
linux    /vmlinuz-linux-lts
initrd   /intel-ucode.img
initrd   /booster-linux-lts-new.img
options  root=/dev/proc/root rw
```

> Ganti `intel-ucode.img` dengan `amd-ucode.img` jika pakai AMD

## Loader Config

```bash
nvim /boot/loader/loader.conf
```

Tambahkan (atau sesuaikan default):
```
default  booster-arch2.conf
```

> Jika ingin tetap bisa memilih OS saat boot, tambahkan `timeout 5` agar menu muncul selama 5 detik

```
timeout  5
default  booster-arch2.conf
```

```bash
bootctl --graceful update
```

---

# BOOTING

```bash
exit
```
```bash
umount -R /mnt
```
```bash
reboot
```

---

> **Catatan Dual Boot:** Kedua instalasi Arch menggunakan `bootctl` pada partisi EFI yang berbeda (`nvme0n1p1` untuk install pertama, `nvme0n1p4` untuk install kedua). Pastikan BIOS/UEFI mendeteksi keduanya, atau gunakan BIOS boot menu untuk memilih entry yang diinginkan.
