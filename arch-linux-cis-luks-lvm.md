# Instalasi Arch Linux: LUKS on LVM dengan CIS Benchmark Disk Layout

> **Referensi:** CIS Distribution-Independent Linux Benchmark v1.1.0  
> **Target:** Dual boot dengan Windows, disk `/dev/nvme0n1`, initramfs menggunakan `booster`

---

## Kondisi Disk Awal (Existing)

| Partisi         | Size    | Type                      | Status            |
|-----------------|---------|---------------------------|-------------------|
| nvme0n1p1       | 260M    | EFI System                | ✅ Dipakai bersama (Windows + GRUB Arch) |
| nvme0n1p2       | 16M     | Microsoft reserved        | 🚫 Jangan disentuh |
| nvme0n1p3       | 123.2G  | Microsoft basic data (C:) | 🚫 Jangan disentuh |
| nvme0n1p4       | 251.2G  | Microsoft basic data (D:) | 🚫 Jangan disentuh |
| nvme0n1p5       | 2G      | Windows recovery          | 🚫 Jangan disentuh |
| nvme0n1p6       | 1G      | EFI Linux lama            | 🗑️ Hapus           |
| nvme0n1p7       | 4G      | Linux swap lama           | 🗑️ Hapus           |
| nvme0n1p8       | 95.3G   | Linux root lama           | 🗑️ Hapus           |

---

## Partisi Wajib sesuai CIS (Scored)

| Mount Point      | Mount Options         | CIS Reference |
|------------------|-----------------------|---------------|
| `/tmp`           | `nodev,nosuid,noexec` | 1.1.2–1.1.5   |
| `/var`           | —                     | 1.1.6         |
| `/var/tmp`       | `nodev,nosuid,noexec` | 1.1.7–1.1.10  |
| `/var/log`       | —                     | 1.1.15        |
| `/var/log/audit` | —                     | 1.1.16        |
| `/home`          | `nodev`               | 1.1.17–1.1.18 |

---

## Arsitektur Disk Akhir

```
/dev/nvme0n1
├── p1   260M    EFI System           ← shared: Windows + GRUB Arch (JANGAN format ulang)
├── p2    16M    Microsoft reserved   ← Windows, jangan sentuh
├── p3  123.2G   Windows C:           ← Windows, jangan sentuh
├── p4  251.2G   Windows D:           ← Windows, jangan sentuh
├── p5     2G    Windows recovery     ← Windows, jangan sentuh
└── p6  ~100G    Linux LVM (vg0)
                 ├── lv_swap     4G   → swap (plain)
                 ├── lv_root    20G   → /              (LUKS)
                 ├── lv_tmp      2G   → /tmp           (LUKS)
                 ├── lv_var      8G   → /var           (LUKS)
                 ├── lv_var_tmp  2G   → /var/tmp       (LUKS)
                 ├── lv_var_log  4G   → /var/log       (LUKS)
                 ├── lv_var_audit 2G  → /var/log/audit (LUKS)
                 └── lv_home   ~58G   → /home          (LUKS)
```

---

## LANGKAH 1 — Persiapan Awal

```bash
# Set keyboard layout (opsional)
loadkeys us

# Verifikasi mode UEFI
ls /sys/firmware/efi/efivars

# Cek layout disk
lsblk
fdisk -l /dev/nvme0n1
```

---

## LANGKAH 2 — Hapus Partisi Linux Lama

> ⚠️ **HATI-HATI:** Hanya hapus p6, p7, p8. Jangan sentuh p1–p5 (Windows).

```bash
gdisk /dev/nvme0n1
```

Di dalam gdisk:

```
d
8         # Hapus nvme0n1p8 (Linux root lama)

d
7         # Hapus nvme0n1p7 (Linux swap lama)

d
6         # Hapus nvme0n1p6 (EFI Linux lama)

p         # Verifikasi — pastikan p1-p5 masih ada dan p6-p8 sudah hilang

w         # Tulis perubahan (confirm: y)
```

---

## LANGKAH 3 — Buat Partisi LVM Baru

```bash
gdisk /dev/nvme0n1
```

```
n                 # Partisi baru
Enter             # Nomor otomatis (akan jadi p6)
Enter             # First sector otomatis (setelah p5)
Enter             # Last sector (pakai semua sisa ~100GB)
8e00              # Tipe: Linux LVM

p                 # Verifikasi

w                 # Simpan (confirm: y)
```

Hasil: **`/dev/nvme0n1p6`** ~100GB untuk LVM.

---

## LANGKAH 4 — Setup LVM

```bash
# Buat Physical Volume
pvcreate /dev/nvme0n1p6

# Buat Volume Group
vgcreate vg0 /dev/nvme0n1p6

# Buat semua Logical Volume (~100GB total)
lvcreate -L 4G       vg0 -n lv_swap
lvcreate -l 50%FREE  vg0 -n lv_root
lvcreate -L 2G       vg0 -n lv_tmp
lvcreate -L 8G       vg0 -n lv_var
lvcreate -L 2G       vg0 -n lv_var_tmp
lvcreate -L 4G       vg0 -n lv_var_log
lvcreate -L 2G       vg0 -n lv_var_audit
lvcreate -l 100%FREE vg0 -n lv_home
# lv_home mendapat sisa 

# Verifikasi
lvs
```

---

## LANGKAH 5 — Enkripsi LUKS

```bash
cryptsetup luksFormat /dev/vg0/cryptroot
# Buka partisi LUKS
cryptsetup open /dev/vg0/lv_root cryptroot
```
masukin password dua kali

## LANGKAH 6 — Format Filesystem

```bash
mkfs.ext4 /dev/mapper/cryptroot
mkfs.ext4 /dev/mapper/crypttmp
mkfs.ext4 /dev/mapper/cryptvar
mkfs.ext4 /dev/mapper/cryptvar_tmp
mkfs.ext4 /dev/mapper/cryptvar_log
mkfs.ext4 /dev/mapper/cryptvar_audit
mkfs.ext4 /dev/mapper/crypthome
mkswap /dev/vg0/lv_swap
```

## LANGKAH 7 — Mount sesuai CIS

```bash
# Root terlebih dahulu
mount /dev/mapper/cryptroot /mnt
```
# Buat semua direktori mount point
```
mkdir -p /mnt/{boot,tmp,home,var/tmp,var/log/audit}
```
# Mount EFI Windows yang sudah ada (JANGAN format ulang!)
```
mount /dev/nvme0n1p1 /mnt/boot

# Mount dengan flag CIS (urutan penting!)
mount -o nodev,nosuid,noexec /dev/mapper/crypttmp      /mnt/tmp
mount                         /dev/mapper/cryptvar       /mnt/var
mount -o nodev,nosuid,noexec /dev/mapper/cryptvar_tmp   /mnt/var/tmp
mount                         /dev/mapper/cryptvar_log   /mnt/var/log
mount                         /dev/mapper/cryptvar_audit /mnt/var/log/audit
mount -o nodev               /dev/mapper/crypthome      /mnt/home
```
# Aktifkan swap
```
swapon /dev/vg0/lv_swap
```

---

## LANGKAH 8 — Instalasi Sistem Dasar

> `booster` digunakan sebagai pengganti `mkinitcpio` untuk generate initramfs.

```bash
pacstrap -K /mnt \
  base linux linux-firmware \
  lvm2 cryptsetup \
  booster \
  grub efibootmgr os-prober \
  networkmanager sudo vim

# Generate fstab
genfstab -U /mnt >> /mnt/etc/fstab

# Verifikasi mount options sudah benar
cat /mnt/etc/fstab
```

---

## LANGKAH 9 — Konfigurasi Dasar (arch-chroot)

```bash
arch-chroot /mnt
```
# Timezone
```
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc
```

# Locale
```
nano /etc/locale.gen

locale-gen
```
```
nano /etc/locale.conf
```
```
"LANG=en_US.UTF-8"
```
# Hostname
```
nano /etc/hostname
```
# Set password root
passwd

## LANGKAH 10 — Konfigurasi Booster
cek image
```
ls -lh /boot/booster*
```
kalau ga ada
Build initramfs dengan booster:
```
booster build mybooster.img
```
atau
```bash
booster build --force /boot/booster-linux.img
```
cek lagi
konfigurasi luks untuk memberitahu booster root


cari tahu UUID root
```
blkid /dev/mapper/cryptroot
blkid -s UUID -o value /dev/mapper/cryptroot
```
salin atau foto uuidnya lalu masukan disini
```
nano /etc/default/grub
```
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```
```
rd.luks.name=device-UUID=root root=/dev/mapper/root
```
```
rd.luks.name=masukanUUIDdisini=root root=/dev/mapper/cryptroot
```
```
cryptdevice=UUID=device-UUID:root root=/dev/mapper/root
cryptdevice=UUID=masukanUUID:cryptroot root=/dev/mapper/cryptroot

ctrl o
enter
ctrl x

konfigurasi 
> `booster` otomatis mendeteksi kernel yang terinstall. File initramfs akan dibuat di `/boot/booster-linux.img`.

Verifikasi file terbuat:

```bash
ls -lh /boot/booster-linux.img
ls -lh /boot/vmlinuz-linux
```

---

## LANGKAH 11 — Konfigurasi /etc/crypttab

Dapatkan UUID semua LV:

```bash
blkid | grep crypto_LUKS
```

Edit `/etc/crypttab`:

```
# <nama>          <device UUID>                <password>  <opsi>
crypttmp           UUID=<UUID-lv_tmp>           none        luks
cryptvar           UUID=<UUID-lv_var>           none        luks
cryptvar_tmp       UUID=<UUID-lv_var_tmp>       none        luks
cryptvar_log       UUID=<UUID-lv_var_log>       none        luks
cryptvar_audit     UUID=<UUID-lv_var_audit>     none        luks
crypthome          UUID=<UUID-lv_home>          none        luks
```

> `cryptroot` tidak perlu di sini — ditangani oleh kernel parameter GRUB.  
> `none` = prompt password saat boot. Bisa diganti keyfile untuk auto-unlock setelah root terbuka.

---

## LANGKAH 12 — Install & Konfigurasi GRUB

```bash
# Install GRUB ke EFI (pakai bootloader-id berbeda dari Windows)
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Arch

# Dapatkan UUID lv_root
blkid /dev/vg0/lv_root
```

Edit `/etc/default/grub`:

```bash
vim /etc/default/grub
```

Ubah/tambahkan baris berikut:

```
# Kernel parameter untuk buka LUKS root
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<UUID-lv_root>:cryptroot root=/dev/mapper/cryptroot"

# Gunakan booster sebagai initramfs
GRUB_EARLY_INITRD_LINUX_STOCK=""

Generate konfigurasi GRUB:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Verifikasi entry booster terdeteksi di grub.cfg:

```bash
grep booster /boot/grub/grub.cfg
```

konfigurasi fstab
```
nano /etc/fstab
```

---

## LANGKAH 13 — Enable Service & Reboot

```bash
systemctl enable NetworkManager

# Keluar dari chroot
exit

# Unmount semua
umount -R /mnt

# Reboot
reboot
```

---

## Post-Install: Hardening Tambahan sesuai CIS

### Disable Filesystem Tidak Dipakai (CIS 1.1.1)

```bash
cat >> /etc/modprobe.d/cis.conf << 'EOF'
install cramfs /bin/true
install freevxfs /bin/true
install jffs2 /bin/true
install hfs /bin/true
install hfsplus /bin/true
install squashfs /bin/true
install udf /bin/true
EOF
```

### Hardening /dev/shm (CIS 1.1.19–1.1.21)

Tambahkan ke `/etc/fstab`:

```
tmpfs  /dev/shm  tmpfs  defaults,nodev,nosuid,noexec  0 0
```

Terapkan tanpa reboot:

```bash
mount -o remount /dev/shm
```

---

## Referensi Mount Options CIS

| Mount Point      | nodev | nosuid | noexec |
|------------------|:-----:|:------:|:------:|
| `/tmp`           | ✅    | ✅     | ✅     |
| `/var/tmp`       | ✅    | ✅     | ✅     |
| `/home`          | ✅    | —      | —      |
| `/dev/shm`       | ✅    | ✅     | ✅     |

---

## Perbandingan booster vs mkinitcpio

| Aspek             | booster                        | mkinitcpio                    |
|-------------------|--------------------------------|-------------------------------|
| Konfigurasi       | `/etc/booster.yaml`            | `/etc/mkinitcpio.conf`        |
| Kecepatan build   | Jauh lebih cepat               | Lebih lambat                  |
| Kecepatan boot    | Lebih cepat                    | Standar                       |
| LUKS support      | ✅ Native                      | ✅ via hook `encrypt`         |
| LVM support       | ✅ Native                      | ✅ via hook `lvm2`            |
| Perintah build    | `booster build --force <path>` | `mkinitcpio -P`               |

---

## Troubleshooting Umum

| Masalah | Kemungkinan Penyebab | Solusi |
|---|---|---|
| Boot gagal, tidak bisa buka LUKS | UUID salah di GRUB | Cek `blkid` dan `/etc/default/grub` |
| `/var/tmp` atau `/tmp` tidak ter-mount | Urutan mount di fstab | Pastikan `/var` di-mount sebelum `/var/tmp` |
| booster gagal build | Package belum terinstall | Jalankan `pacman -S booster` |
| GRUB tidak detect Windows | os-prober tidak aktif | Pastikan `GRUB_DISABLE_OS_PROBER=false` dan jalankan ulang `grub-mkconfig` |
| Partisi lain tidak terbuka saat boot | crypttab salah UUID | Jalankan `blkid` dan cocokkan UUID |
| `/boot/booster-linux.img` tidak ada | Belum di-build | Jalankan `booster build --force /boot/booster-linux.img` |
