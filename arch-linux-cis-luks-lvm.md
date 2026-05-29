# Instalasi Arch Linux: LUKS on LVM dengan CIS Benchmark Disk Layout

> **Referensi:** CIS Distribution-Independent Linux Benchmark v1.1.0  
> **Target:** UEFI system, disk `/dev/sda`

---

## Partisi Wajib sesuai CIS (Scored)

| Mount Point     | Mount Options          | CIS Reference |
|-----------------|------------------------|---------------|
| `/tmp`          | `nodev,nosuid,noexec`  | 1.1.2–1.1.5   |
| `/var`          | —                      | 1.1.6         |
| `/var/tmp`      | `nodev,nosuid,noexec`  | 1.1.7–1.1.10  |
| `/var/log`      | —                      | 1.1.15        |
| `/var/log/audit`| —                      | 1.1.16        |
| `/home`         | `nodev`                | 1.1.17–1.1.18 |

---

## Arsitektur Disk

```
/dev/sda
├── sda1  →  EFI (512MB, vfat)
└── sda2  →  LVM Physical Volume
             └── Volume Group: vg0
                 ├── lv_swap        →  swap
                 ├── lv_root        →  /             (LUKS)
                 ├── lv_tmp         →  /tmp          (LUKS)
                 ├── lv_var         →  /var          (LUKS)
                 ├── lv_var_tmp     →  /var/tmp      (LUKS)
                 ├── lv_var_log     →  /var/log      (LUKS)
                 ├── lv_var_audit   →  /var/log/audit (LUKS)
                 └── lv_home        →  /home         (LUKS)
```

---

## LANGKAH 1 — Persiapan Awal

```bash
# Set keyboard layout (opsional)
loadkeys us

# Verifikasi mode UEFI
ls /sys/firmware/efi/efivars

# Cek nama disk
lsblk
```

---

## LANGKAH 2 — Partisi Disk

```bash
gdisk /dev/sda
```

Di dalam gdisk:

```
o         # Hapus semua partisi (confirm: y)

n         # Partisi baru
1         # Nomor 1
Enter     # First sector default
+512M     # Size
ef00      # Tipe: EFI System

n         # Partisi baru
2         # Nomor 2
Enter     # First sector default
Enter     # Last sector (sisa semua)
8e00      # Tipe: Linux LVM

w         # Tulis dan keluar (confirm: y)
```

Format EFI:

```bash
mkfs.fat -F32 /dev/sda1
```

---

## LANGKAH 3 — Setup LVM

```bash
# Buat Physical Volume
pvcreate /dev/sda2

# Buat Volume Group
vgcreate vg0 /dev/sda2

# Buat semua Logical Volume
# Sesuaikan ukuran dengan kapasitas disk Anda
lvcreate -L 4G       vg0 -n lv_swap
lvcreate -L 20G      vg0 -n lv_root
lvcreate -L 2G       vg0 -n lv_tmp
lvcreate -L 10G      vg0 -n lv_var
lvcreate -L 2G       vg0 -n lv_var_tmp
lvcreate -L 5G       vg0 -n lv_var_log
lvcreate -L 2G       vg0 -n lv_var_audit
lvcreate -l 100%FREE vg0 -n lv_home
```

---

## LANGKAH 4 — Enkripsi LUKS

```bash
# Enkripsi semua LV (kecuali swap)
for lv in lv_root lv_tmp lv_var lv_var_tmp lv_var_log lv_var_audit lv_home; do
  echo "==> Encrypting /dev/vg0/$lv"
  cryptsetup luksFormat /dev/vg0/$lv
done

# Buka semua partisi LUKS
cryptsetup open /dev/vg0/lv_root      cryptroot
cryptsetup open /dev/vg0/lv_tmp       crypttmp
cryptsetup open /dev/vg0/lv_var       cryptvar
cryptsetup open /dev/vg0/lv_var_tmp   cryptvar_tmp
cryptsetup open /dev/vg0/lv_var_log   cryptvar_log
cryptsetup open /dev/vg0/lv_var_audit cryptvar_audit
cryptsetup open /dev/vg0/lv_home      crypthome
```

---

## LANGKAH 5 — Format Filesystem

```bash
mkfs.ext4 /dev/mapper/cryptroot
mkfs.ext4 /dev/mapper/crypttmp
mkfs.ext4 /dev/mapper/cryptvar
mkfs.ext4 /dev/mapper/cryptvar_tmp
mkfs.ext4 /dev/mapper/cryptvar_log
mkfs.ext4 /dev/mapper/cryptvar_audit
mkfs.ext4 /dev/mapper/crypthome

# Swap tidak dienkripsi LUKS (bisa ditambahkan via /etc/crypttab setelah install)
mkswap /dev/vg0/lv_swap
```

---

## LANGKAH 6 — Mount sesuai CIS

```bash
# Root terlebih dahulu
mount /dev/mapper/cryptroot /mnt

# Buat semua direktori
mkdir -p /mnt/{boot,tmp,home,var/tmp,var/log/audit}

# Mount EFI
mount /dev/sda1 /mnt/boot

# Mount dengan flag CIS (urutan penting!)
mount -o nodev,nosuid,noexec /dev/mapper/crypttmp      /mnt/tmp
mount                         /dev/mapper/cryptvar       /mnt/var
mount -o nodev,nosuid,noexec /dev/mapper/cryptvar_tmp   /mnt/var/tmp
mount                         /dev/mapper/cryptvar_log   /mnt/var/log
mount                         /dev/mapper/cryptvar_audit /mnt/var/log/audit
mount -o nodev               /dev/mapper/crypthome      /mnt/home

# Aktifkan swap
swapon /dev/vg0/lv_swap
```

---

## LANGKAH 7 — Instalasi Sistem Dasar

```bash
pacstrap -K /mnt \
  base linux linux-firmware \
  lvm2 cryptsetup \
  grub efibootmgr \
  networkmanager sudo vim

# Generate fstab
genfstab -U /mnt >> /mnt/etc/fstab

# Verifikasi — pastikan mount options sudah benar
cat /mnt/etc/fstab
```

---

## LANGKAH 8 — Konfigurasi Dasar (arch-chroot)

```bash
arch-chroot /mnt

# Timezone (sesuaikan)
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc

# Locale
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Hostname
echo "archcis" > /etc/hostname

# Set password root
passwd
```

---

## LANGKAH 9 — Konfigurasi mkinitcpio

Edit `/etc/mkinitcpio.conf`:

```bash
vim /etc/mkinitcpio.conf
```

Ubah baris `HOOKS=` menjadi:

```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block lvm2 encrypt filesystems fsck)
```

> **Penting:** urutan `lvm2` harus sebelum `encrypt` dan `filesystems`.

Regenerasi initramfs:

```bash
mkinitcpio -P
```

---

## LANGKAH 10 — Konfigurasi /etc/crypttab

Dapatkan UUID semua LV:

```bash
blkid | grep crypto_LUKS
```

Edit `/etc/crypttab` dan isi dengan UUID masing-masing:

```
# <nama>         <device (UUID)>              <password>  <opsi>
crypttmp          UUID=<UUID-lv_tmp>           none        luks
cryptvar          UUID=<UUID-lv_var>           none        luks
cryptvar_tmp      UUID=<UUID-lv_var_tmp>       none        luks
cryptvar_log      UUID=<UUID-lv_var_log>       none        luks
cryptvar_audit    UUID=<UUID-lv_var_audit>     none        luks
crypthome         UUID=<UUID-lv_home>          none        luks
```

> `cryptroot` tidak perlu di sini — ditangani oleh kernel parameter GRUB.  
> `none` = akan prompt password. Bisa diganti keyfile untuk unlock otomatis setelah root terbuka.

---

## LANGKAH 11 — Install & Konfigurasi GRUB

```bash
# Install GRUB ke EFI
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# Dapatkan UUID dari lv_root
blkid /dev/vg0/lv_root
```

Edit `/etc/default/grub`, tambahkan ke `GRUB_CMDLINE_LINUX`:

```
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<UUID-lv_root>:cryptroot root=/dev/mapper/cryptroot"
```

Generate konfigurasi GRUB:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## LANGKAH 12 — Enable Service & Reboot

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

## Troubleshooting Umum

| Masalah | Kemungkinan Penyebab | Solusi |
|---|---|---|
| Boot gagal, tidak bisa buka LUKS | UUID salah di GRUB | Cek `blkid` dan `/etc/default/grub` |
| `/var/tmp` atau `/tmp` tidak ter-mount | Urutan mount di fstab | Pastikan `/var` di-mount sebelum `/var/tmp` |
| initramfs tidak bisa buka LVM | Hook `lvm2` tidak ada | Cek `HOOKS=` di `mkinitcpio.conf` |
| Partisi lain tidak terbuka saat boot | crypttab salah UUID | Jalankan `blkid` dan cocokkan UUID |
