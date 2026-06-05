login user di linux baru dengan tty1

install luks jika belum
```
pacman -S cryptsetup
```
cek isi disk 
```
lsblk
```

<img width="1440" height="960" alt="WhatsApp Image 2026-06-05 at 20 11 04" src="https://github.com/user-attachments/assets/f6169f71-f247-4d22-aba5-c89695495e04" />

liat lvm
```
sudo lvdisplay
```
buat enkripsi
```
sudo cryptsetup luksFormat /dev/sawit/lvroot
```
liat status
```
sudo cryptsetup status /dev/sawit/lvroot
```
open untuk membuat virtual device yang mapping enkripsi (cryptroot=nama virtual) 
```
sudo cryptsetup status /dev/sawit/lvroot cryptroot
```
untuk lihat luks header information
```
sudo cryptsetup luksDump /dev/sawit/lvroot
```
formatting
```
sudo mkfs.ext4 /dev/mapper/cryptroot
```
mounting
```
sudo mkdir -p /mnt/cryptroot
sudo mount /dev/mapper/cryptroot /mnt/cryptroot
```
edit crypttab agar bisa terbaca ketika proses booting
```
echo "cryptroot UUID=$(cryptsetup luksUUID /dev/nvme0n1p5) none luks" >> /etc/crypttab
```
cek lagi
```
sudo nano /etc/crypttab
```
update fstab file


masukan
```
/dev/mapper/lroot / ext4 defaults 0 1
```

update grub
```
echo "cryptsetup luksUUID /dev/mapper/lvroot" >> /etc/crypttab
```

``` 
nano /etc/default/grub
```
buat seperti ini
cut uuidnya lalu dibagian ini ketik seperti itu
```
cryptdevice=UUID= device-UUID :root root=/dev/mapper/root
```
paste uuid dibagian ini ```device-UUID```

