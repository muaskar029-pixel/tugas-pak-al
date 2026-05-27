# penjelasan 

## booster

Booster adalah alat cepat untuk membuat initramfs. Ia mirip mkinitcpio dan dracut, terinspirasi dari proyek distri, bertujuan membuat init image yang kecil dan cepat, menggunakan program ```/usr/bin/booster```, dan hasil file-nya secara default disimpan di ```/boot/```.

Teks itu menjelaskan bahwa setelah kamu install Booster, sistem akan otomatis membuat file initramfs untuk setiap kernel yang terpasang.

Contoh kalau kernel kamu adalah:
```
linux
```
maka Booster akan membuat file:
```
/boot/booster-linux.img
```
Perintah ini:
```
ls -lh /boot/booster*
```
dipakai untuk mengecek apakah file Booster sudah ada di folder /boot.
Hasil ini:
```
-rwxr-xr-x 1 root root 4.0M Dec 16 16:20 /boot/booster-linux.img
```
artinya file booster-linux.img sudah berhasil dibuat, ukurannya sekitar 4 MB, dan lokasinya ada di:
```
/boot/
```
Jadi intinya: install Booster → otomatis dibuatkan file initramfs → file-nya disimpan di /boot/.

## Usage Booster

Setelah konfigurasi Booster selesai atau diubah, file **initramfs image** perlu dibuat ulang agar perubahan tersebut bisa digunakan saat proses booting.

### Membuat Ulang Initramfs Secara Otomatis

Booster menyediakan script praktis untuk membuat ulang initramfs image untuk semua kernel yang sudah terinstall.

Jalankan perintah berikut:

```bash
sudo /usr/lib/booster/regenerate_images
```

Perintah ini akan membuat ulang image Booster untuk setiap kernel yang ada, misalnya:

- `linux`
- `linux-lts`

Hasil file biasanya akan berada di direktori:

```bash
/boot/
```

Contoh hasilnya:

```bash
/boot/booster-linux.img
```

### Membuat Initramfs Secara Manual

Selain menggunakan script otomatis, initramfs juga bisa dibuat manual dengan perintah:

```bash
booster build mybooster.img
```

Perintah tersebut akan membuat file initramfs bernama:

```bash
mybooster.img
```

File ini akan dibuat di direktori tempat terminal sedang dijalankan.

Jika direktori tersebut membutuhkan izin root, gunakan:

```bash
sudo booster build mybooster.img
```

## Configuration Booster

File konfigurasi Booster berada di:

```bash
/etc/booster.yaml
```

Jika file konfigurasi tidak ada, Booster akan memakai konfigurasi bawaan, yaitu:

- image dibuat sesuai host/sistem yang digunakan
- tanpa dukungan network di initramfs

File konfigurasi ini digunakan untuk mengubah perilaku bawaan Booster.

---

## Early Module Loading

Booster bisa memaksa beberapa modul kernel agar dimuat lebih awal saat tahap initramfs.

Contoh konfigurasi:

```yaml
modules_force_load: nvidia,nvidia_modeset,nvidia_uvm,nvidia_drm
```

Konfigurasi ini berguna jika menggunakan NVIDIA dan ingin GPU diinisialisasi lebih awal sebelum sesi grafis berjalan.

Setelah mengubah konfigurasi, buat ulang image Booster:

```bash
sudo /usr/lib/booster/regenerate_images
```

---

## Encryption

Booster mendukung enkripsi disk berbasis **LUKS** secara bawaan.

Generator Booster tidak membutuhkan konfigurasi tambahan untuk LUKS. Namun, bootloader harus diberi informasi tentang partisi LUKS yang berisi root system.

Parameter yang bisa digunakan:

```bash
rd.luks.uuid=LUKSUUID
```

atau:

```bash
rd.luks.name=LUKSUUID=LUKSNAME
```

Keterangan:

- `LUKSUUID` adalah UUID dari partisi LUKS yang akan dibuka oleh Booster
- `LUKSNAME` adalah nama partisi setelah dibuka, misalnya di `/dev/mapper/LUKSNAME`

Setelah konfigurasi bootloader selesai, tidak perlu rebuild image.

Cukup reboot komputer. Saat booting, akan muncul prompt seperti:

```text
Enter passphrase for YOURROOT:
```

Artinya sistem meminta password untuk membuka partisi root yang terenkripsi.

---

## systemd Style Binding

Booster juga mendukung partisi yang terikat dengan systemd, seperti:

- `systemd-fido2`
- `systemd-tpm2`

Jika menggunakan `systemd-fido2`, install paket berikut:

```bash
sudo pacman -S libfido2
```

Lalu tambahkan konfigurasi ini ke `/etc/booster.yaml`:

```yaml
extra_files: fido2-assert
```

Setelah itu, buat ulang image Booster:

```bash
sudo /usr/lib/booster/regenerate_images
```

Booster akan mendeteksi konfigurasi ini saat booting dan menggunakan YubiKey untuk membuka drive.

Jika FIDO2 tidak terbaca saat booting, tambahkan modul berikut:

```yaml
modules_force_load: usbhid,hid_sensor_hub
extra_files: fido2-assert
```

---

## Removing Modules

Booster bisa menghapus modul tertentu dari initramfs menggunakan tanda `-`.

Contoh:

```yaml
modules: -*,btrfs,nvme
```

Artinya:

- semua modul dihapus terlebih dahulu dengan `-*`
- hanya modul `btrfs` dan `nvme` yang dimasukkan

Konfigurasi ini bisa digunakan untuk membuat initramfs lebih minimal dan meningkatkan performa booting.

---

## Stripping Modules

Untuk mengurangi ukuran initramfs, aktifkan opsi `strip`.

Contoh:

```yaml
strip: true
```

Namun, opsi ini dapat menghapus signature modul yang dibutuhkan oleh kernel lockdown.

---

## Compression

Booster mendukung kompresi initramfs.

Secara default, Booster menggunakan algoritma:

```yaml
compression: zstd
```

Algoritma kompresi yang didukung:

- `zstd`
- `xz`
- `gzip`
- `lz4`
- `none`

Contoh konfigurasi:

```yaml
compression: zstd
```

---

## Kesimpulan

File konfigurasi Booster berada di:

```bash
/etc/booster.yaml
```

Beberapa hal yang bisa dikonfigurasi antara lain:

- modul kernel yang dimuat lebih awal
- dukungan enkripsi LUKS
- dukungan FIDO2/YubiKey
- penghapusan modul yang tidak diperlukan
- pengurangan ukuran initramfs
- algoritma kompresi

Jika konfigurasi Booster diubah, jalankan:

```bash
sudo /usr/lib/booster/regenerate_images
```

agar perubahan diterapkan ke initramfs.
