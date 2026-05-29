hapus linux lama

konek internet 

timedatectl

partisi, lvm, luks, cis layout

lvm

https://github.com/Dorian-DS/LVM-Cheat-Sheets

http://tldp.org/HOWTO/LVM-HOWTO/


luks on lvm

lvm diatur dulu baru luks jadi akan ada disk yang terenkripsi dan tidak

https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_LVM

cis layoutnya


| Physical Disk  | Partition        | Layer LVM                    | Layer LUKS / Mapper       | Mount Point      | Keterangan                                               |
| -------------- | ---------------- | ---------------------------- | ------------------------- | ---------------- | -------------------------------------------------------- |
| `/dev/nvme0n1` | `/dev/nvme0n1p6` | -                            | -                         | `/boot`          | EFI System Partition, tidak dienkripsi                   |
| `/dev/nvme0n1` | `/dev/nvme0n1p8` | `sawitGrup`                  | -                         | -                | Partisi utama untuk LVM                                  |
| `/dev/nvme0n1` | `/dev/nvme0n1p8` | `/dev/sawitGrup/cryptroot`   | `/dev/mapper/cryptroot`   | `/`              | Root filesystem                                          |
| `/dev/nvme0n1` | `/dev/nvme0n1p8` | `/dev/sawitGrup/crypttmp`    | `/dev/mapper/crypttmp`    | `/tmp`           | CIS separate filesystem                                  |
| `/dev/nvme0n1` | `/dev/nvme0n1p8` | `/dev/sawitGrup/cryptvar`    | `/dev/mapper/cryptvar`    | `/var`           | CIS separate filesystem                                  |
| `/dev/nvme0n1` | `/dev/nvme0n1p8` | `/dev/sawitGrup/cryptvartmp` | `/dev/mapper/cryptvartmp` | `/var/tmp`       | CIS separate filesystem                                  |
| `/dev/nvme0n1` | `/dev/nvme0n1p8` | `/dev/sawitGrup/cryptvarlog` | `/dev/mapper/cryptvarlog` | `/var/log`       | CIS separate filesystem                                  |
| `/dev/nvme0n1` | `/dev/nvme0n1p8` | `/dev/sawitGrup/cryptaudit`  | `/dev/mapper/cryptaudit`  | `/var/log/audit` | CIS separate filesystem                                  |
| `/dev/nvme0n1` | `/dev/nvme0n1p8` | `/dev/sawitGrup/crypthome`   | `/dev/mapper/crypthome`   | `/home`          | CIS separate filesystem                                  |
| `/dev/nvme0n1` | `/dev/nvme0n1p8` | `/dev/sawitGrup/cryptswap`   | `/dev/mapper/cryptswap`   | `[SWAP]`         | Encrypted swap, tambahan keamanan dari panduan Arch Wiki |
```
lsblk
```
buat partisi baru dengan root type file 44 ```linux lvm```

```
fdisk /dev/nvme0n1
```

format EFI linux
```
mkfs.fat -F32 /dev/nvme0n1p6
```

Buat LVM dengan nama sawitGrup
```
pvcreate /dev/nvme0n1p7
vgcreate sawitGrup /dev/nvme0n1p7
```
cek
```
pvs
vgs
```
Buat Logical Volume sesuai CIS + swap terenkripsi
```
lvcreate -L 28G sawitGrup -n cryptroot
lvcreate -L 5G sawitGrup -n crypttmp
lvcreate -L 12G sawitGrup -n cryptvar
lvcreate -L 5G sawitGrup -n cryptvartmp
lvcreate -L 7G sawitGrup -n cryptvarlog
lvcreate -L 4G sawitGrup -n cryptaudit
lvcreate -L 4G sawitGrup -n cryptswap
lvcreate -l 100%FREE sawitGrup -n crypthome
```
cek
```
lvs
```
targetnya ad:
```
cryptroot
crypttmp
cryptvar
cryptvartmp
cryptvarlog
cryptaudit
cryptswap
crypthome
```
Sekarang enkripsi semua LV kecuali swap dulu:
```
cryptsetup luksFormat /dev/sawitGrup/cryptroot
cryptsetup luksFormat /dev/sawitGrup/crypttmp
cryptsetup luksFormat /dev/sawitGrup/cryptvar
cryptsetup luksFormat /dev/sawitGrup/cryptvartmp
cryptsetup luksFormat /dev/sawitGrup/cryptvarlog
cryptsetup luksFormat /dev/sawitGrup/cryptaudit
cryptsetup luksFormat /dev/sawitGrup/crypthome
```
Buka volume LUKS
```
cryptsetup open /dev/sawitGrup/cryptroot cryptroot
cryptsetup open /dev/sawitGrup/crypttmp crypttmp
cryptsetup open /dev/sawitGrup/cryptvar cryptvar
cryptsetup open /dev/sawitGrup/cryptvartmp cryptvartmp
cryptsetup open /dev/sawitGrup/cryptvarlog cryptvarlog
cryptsetup open /dev/sawitGrup/cryptaudit cryptaudit
cryptsetup open /dev/sawitGrup/crypthome crypthome
```
cek
```
lsblk
```
Kalau sudah benar, setiap LV punya anak mapper seperti:
```
cryptroot
crypttmp
cryptvar
cryptvartmp
cryptvarlog
cryptaudit
crypthome
```
format ext4
```
mkfs.ext4 /dev/mapper/cryptroot
mkfs.ext4 /dev/mapper/crypttmp
mkfs.ext4 /dev/mapper/cryptvar
mkfs.ext4 /dev/mapper/cryptvartmp
mkfs.ext4 /dev/mapper/cryptvarlog
mkfs.ext4 /dev/mapper/cryptaudit
mkfs.ext4 /dev/mapper/crypthome
```
jangan format swap dulu

mount root
```
mount /dev/mapper/cryptroot /mnt
```
buat foldernya
```
mkdir -p /mnt/boot
mkdir -p /mnt/tmp
mkdir -p /mnt/var
mkdir -p /mnt/var/tmp
mkdir -p /mnt/var/log
mkdir -p /mnt/var/log/audit
mkdir -p /mnt/home
```
mount semua volume
```
mount /dev/nvme0n1p6 /mnt/boot

mount -o nodev,nosuid,noexec /dev/mapper/crypttmp /mnt/tmp
mount /dev/mapper/cryptvar /mnt/var
mount -o nodev,nosuid,noexec /dev/mapper/cryptvartmp /mnt/var/tmp
mount -o nodev,nosuid,noexec /dev/mapper/cryptvarlog /mnt/var/log
mount -o nodev,nosuid,noexec /dev/mapper/cryptaudit /mnt/var/log/audit
mount -o nodev /dev/mapper/crypthome /mnt/home
```
cek hasilnya
```
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS /mnt /mnt/tmp /mnt/var /mnt/var/tmp /mnt/var/log /mnt/var/log/audit /mnt/home
```

pacstrap -K /mnt base linux linux-firmware amd-ucode lvm2 cryptsetup sudo nano vim networkmanager man-db man-pages texinfo grub efibootmgr os-prober booster neovim

generate fstab
```
genfstab -U /mnt >> /mnt/etc/fstab
```
cek
```
cat /mnt/etc/fstab
```
Nanti bagian ini wajib diedit agar opsi CIS permanen:
```
nano /mnt/etc/fstab
```
Pastikan mount option-nya jadi seperti ini:
```
/tmp            ext4    defaults,nodev,nosuid,noexec    0 2
/var            ext4    defaults                        0 2
/var/tmp        ext4    defaults,nodev,nosuid,noexec    0 2
/var/log        ext4    defaults,nodev,nosuid,noexec    0 2
/var/log/audit  ext4    defaults,nodev,nosuid,noexec    0 2
/home           ext4    defaults,nodev                  0 2
```
Tambahkan juga untuk /dev/shm:
```
tmpfs  /dev/shm  tmpfs  defaults,nodev,nosuid,noexec  0 0
```
masuk ke sistem baru
```
arch-chroot /mnt
```
Konfigurasi timezone
```
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc
```
```
nano /etc/locale.gen
```
uncomment
```
en_US.UTF-8 UTF-8
```
```
locale-gen
```
Buat locale default:
```
nvim /etc/locale.gen
```
Hostname
```
nvim /etc/hostname
```
masukan nama contoh ```arch-sawit```


edit host
```
nvim /etc/hosts
```
```
127.0.0.1   localhost
::1         localhost
127.0.1.1   arch-sawit.localdomain arch-sawit
```
set password root
```
passwd
```
network manager
```
systemctl enable NetworkManager
```
konfigurasi booster
```
nano /etc/booster.yaml
```
isi
```
enable_lvm: true
```
Sekarang generate initramfs Booster:
```
booster build --force
```

Cek hasilnya:
```
ls -lh /boot/booster*
```
Konfigurasi crypttab untuk volume selain root
```
blkid /dev/sawitGrup/cryptroot
blkid /dev/sawitGrup/crypttmp
blkid /dev/sawitGrup/cryptvar
blkid /dev/sawitGrup/cryptvartmp
blkid /dev/sawitGrup/cryptvarlog
blkid /dev/sawitGrup/cryptaudit
blkid /dev/sawitGrup/crypthome
```
Edit:
```
nano /etc/crypttab
```
formatnya
```
crypttmp       UUID=UUID_CRYPTTMP       none  luks
cryptvar       UUID=UUID_CRYPTVAR       none  luks
cryptvartmp    UUID=UUID_CRYPTVARTMP    none  luks
cryptvarlog    UUID=UUID_CRYPTVARLOG    none  luks
cryptaudit     UUID=UUID_CRYPTAUDIT     none  luks
crypthome      UUID=UUID_CRYPTHOME      none  luks
```
Untuk cryptroot, jangan dimasukkan dulu ke crypttab, karena root akan dibuka dari bootloader/kernel parameter.


Temporary encrypted swap

Karena swap kita mau seperti panduan Arch Wiki: temporary encrypted swap, isi /etc/crypttab tambahkan:

```
cryptswap /dev/sawitGrup/cryptswap /dev/urandom swap,cipher=aes-xts-plain64,size=256
```
```
nvim /etc/fstab
```
masukan baris ini
```
/dev/mapper/cryptswap none swap defaults 0 0
```
cek
```
cat /etc/fstab
```
install grub
```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id="Archlinux"
```
Sekarang ambil UUID untuk root LUKS:
```
blkid /dev/sawitGrup/cryptroot
```
Copy nilai UUID-nya

edit grub
```
nano /etc/default/grub
```
cari baris
```
GRUB_CMDLINE_LINUX=""
```
isi
```
GRUB_CMDLINE_LINUX="rd.luks.name=UUID_CRYPTROOT=cryptroot root=/dev/mapper/cryptroot"
```
Ganti UUID_CRYPTROOT dengan UUID dari:
```
blkid /dev/sawitGrup/cryptroot
```
aktifkan os-prober
```
nano /etc/default/grub
```
uncomment
```
GRUB_DISABLE_OS_PROBER=false
```
generate konfigurasi grub
```
grub-mkconfig -o /boot/grub/grub.cfg
```

Cek sebelum keluar chroot
```
cat /etc/fstab
cat /etc/crypttab
ls -lh /boot
```
Pastikan ada:
```
/boot/grub/grub.cfg
/boot/booster-linux.img
```
```
exit
```
```
umount -R /mnt
reboot
```
