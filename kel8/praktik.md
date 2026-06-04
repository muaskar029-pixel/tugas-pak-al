konek ke internet

timedatectl

partisi disk
```
fdisk /dev/nvme0n1
```
sesuaikan dengan disk laptop


buat minimal 1 disk yang mau digunakan untuk pv lvm


buat lvm

di laptop saya


buat volume group
```
pvcreate /dev/nvme0n1p4
```
buat volume group ```myvolgroup``` boleh diubah
```
vgcreate MyVolGroup /dev/sdb1
```
buat logical volume sesuai layout Independet linux bechmark
```
lvcreate -L 4G   vg0 -n swap
lvcreate -L 10G  vg0 -n root
lvcreate -L 2G   vg0 -n tmp
lvcreate -L 10G  vg0 -n var
lvcreate -L 2G   vg0 -n var_tmp
lvcreate -L 5G   vg0 -n var_log
lvcreate -L 2G   vg0 -n var_audit
lvcreate -l 100%FREE vg0 -n home
```
enkripsi disk

install packages

genfstab

masuk ke sistem baru

localization

buat hostname

buat hosts

buat sudo user

nyalakan networkmanager


