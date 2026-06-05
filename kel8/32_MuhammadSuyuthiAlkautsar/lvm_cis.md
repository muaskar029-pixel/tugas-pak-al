# TUTORIAL INSTALASI BASE ARCH LINUX

## konek internet
jika menggunakan kabel lan tinggal colok


jika menggunakan wifi:
iwctl
```
iwctl
```
```
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect NamaWifi
```
alternatif jika phy0 belum dinyalakan
Keluar dulu dari iwctl:
```
exit
```
```
rfkill list
```
Kalau ada tulisan Soft blocked: yes atau Hard blocked: yes
```
rfkill unblock all
rfkill unblock wifi
```
```
ip link set wlan0 up
```
Update sistem waktu
```
timedatectl
```

## Partisi Disk
Di live environment anda dapat melihat disk yang dibagi menjadi *block device* seperti /dev/sda, /dev/nvme0n1. Untuk mengidentifikasi device ini gunakan lsblk atau fdisk
```
fdisk /dev/nvme0n1
```
buat satu disk untuk lvm dan boot jika dibutuhkan


### lalu format disk

jika buat boot, jangan diformat ulang kalau sebelumnnya sudah ada karena dapat merusak sistem
### untuk boot
```
mkfs.fat -F 32 /dev/nvme0n1p6
```
lalu buat folder dan mount bootnya


lalu buat lvm

untuk pv contoh di disk saya
```
pvcreate /dev/nvme0n1p5
```
untuk vg
```
vgcreate sawit /dev/nvme0n1p5
```
buat lv sesuai layout cis
```
lvcreate -L 20G sawit -n lroot
lvcreate -L 8G  sawit -n lvar
lvcreate -L 4G  sawit -n lvtmp
lvcreate -L 4G  sawit -n lvlog
lvcreate -L 2G  sawit -n lvaud
lvcreate -l 100%FREE sawit -n home
```

format partisi
```
mkfs.ext4 /dev/sawit/lroot 
mkfs.ext4 /dev/sawit/lvar
mkfs.ext4 /dev/sawit/lvtmp
mkfs.ext4 /dev/sawit/lvlog
mkfs.ext4 /dev/sawit/lvaud
mkfs.ext4 /dev/sawit/home
```
tambahkan format boot jika perlu


## Mount partisi

```
mount /dev/sawit/lroot /mnt
mount --mkdir /dev/nvme0n1p4 /mnt/boot
mount --mkdir -o rw,nodev,nosuid,relatime /dev/proc/lvar /mnt/var
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/sawit/ltmp /mnt/var/tmp
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/sawit/lvlog /mnt/var/log
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/sawit/lvaud /mnt/var/log/audit
mount --mkdir -o rw,nodev,nosuid,noexec,relatime /dev/sawit/home /mnt/home
```

## Mirror
jika diperlukan semisalnya downloadnya masih lambat
```
nano /etc/pacman.d/mirrorlist

## Install packages penting
paket yang sudah dikustomisasi untuk laptop saya
```
pacstrap -K /mnt base linux-hardened amd-ucode mesa vulkan-radeon xf68-video-amdgpu xf68-video-ati linux-firmware networkmanager nano sudo grub efibootmgr os-prober
```
### fstab
agar Agar nanti setelah Arch Linux di-boot, sistem tahu format partisi disk yang disk ada dimana
dibuatlah file
```
/etc/fstab
```
```fstab``` adalah daftar partisi yang harus otomatis dimount saat Linux menyala
```
genfstab -U /mnt >> /mnt/etc/fstab
```

### chroot
### untuk ke system environmt 
```
arch-chroot /mnt
```
set time
```
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc
```

### localization
karena sudah install nano

masuk ke folder
```
nano /etc/locale.gen
```
dibuka komentarnya 
```
en_US.UTF-8 UTF-8
```
jalankan
```
locale -gen
```

ketik
```
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```
hostname
```
nano /etc/hostname
```
adduser
```
useradd -m -G wheel -s /bin/bash kautsar
```
masuk
```
EDITOR=nano visudo
```
uncomment
```
# %wheel ALL=(ALL:ALL) ALL
```
EDITOR=nano visudo
```
passwd
```
nano

### ketik nama hostname
### start networkmanager
```
systemctl enable NetworkManager.service
```
root password
```
passwd
```
### boot loader

### install grub package
contoh
```
grub-install --target=x86_64-efi --efi-directory=esp --bootloader-id=GRUB
```
sesuaikan dengan folder boot
```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```
karena saya sudah buat jadi tidak perlu

### initframs
```
mkinitcpio -P
```

Use the ```grub-mkconfig tool``` to generate ```/boot/grub/grub.cfg```
nyalakan os-prober
```
nano /etc/default/grub
```
setting sendiri malas ngetik


update bootloader
```
grub-mkconfig -o /boot/grub/grub.cfg
```
setting sendiri malas ngetik
Reboot
```
exit
```
```
umount -R /mnt
```
```
reboot
```
