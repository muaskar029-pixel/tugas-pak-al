Logical Volume Manager (LVM) is a device mapper framework that provides logical volume management for the Linux kernel.


Basic building blocks of LVM:

1. Physical Volume: Physical Volume adalah bahan dasar penyimpanan yang dipakai LVM sebelum ruangnya dikumpulkan ke dalam Volume Group.
2. Volume Group: VG adalah kolam penyimpanan yang berisi kumpulan PV, lalu dari kolam itu dibuat Logical Volume.
3. Logical Volume: LV adalah partisi virtual dari LVM yang bisa diformat dan digunakan seperti partisi biasa.
4. Physical Extent: PE adalah unit kecil penyimpanan di dalam PV yang dipakai LVM untuk membangun Logical Volume.

keuntungna: dinamis, fleksibel, bisa RAID

kekurangan: 

instalasi

