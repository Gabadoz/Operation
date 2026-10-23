# Panduan Instalasi Arch Linux: Btrfs, Snapper & Optimasi AMD Ryzen 5 5625U

Dokumen ini merupakan panduan teknis komprehensif instalasi Arch Linux berdasarkan dokumentasi resmi [ArchWiki](https://wiki.archlinux.org), yang disesuaikan secara khusus dengan spesifikasi perangkat keras dan kebutuhan sistem berikut:

- **Prosesor:** AMD Ryzen 5 5625U (6 Cores, 12 Threads, Zen 3 Barcelo)
- **Grafis:** AMD Radeon Vega 7 Mobile Graphics (iGPU)
- **RAM:** 12 GB
- **Penyimpanan:** SSD Ruang Kosong (*Free Space*) 106,49 GB (UEFI Mode)
- **Kernel:** `linux-lts` & `linux-lts-headers`
- **Filesystem:** BTRFS dengan subvolume terisolasi untuk Snapshot (*Snapper* / *Timeshift*)
- **Manajemen Snapshot:** Snapper otomatis dengan Pacman Hooks & integrasi Boot Menu GRUB
- **Optimasi Performa:** AMD P-State (EPP), ZRAM untuk RAM 12GB, dan I/O Btrfs SSD

---

## Daftar Isi
1. [Persiapan Awal & Boot Environment](#1-persiapan-awal--boot-environment)
2. [Skema Partisi SSD (Free Space 106,49 GB)](#2-skema-partisi-ssd-free-space-10649-gb)
3. [Format Filesystem & Konfigurasi Btrfs Subvolume](#3-format-filesystem--konfigurasi-btrfs-subvolume)
4. [Instalasi Sistem Dasar (Pacstrap)](#4-instalasi-sistem-dasar-pacstrap)
5. [Konfigurasi Sistem Dasar](#5-konfigurasi-sistem-dasar)
6. [Konfigurasi Bootloader GRUB & Kernel Parameter](#6-konfigurasi-bootloader-grub--kernel-parameter)
7. [Driver Grafis AMD Radeon Vega 7](#7-driver-grafis-amd-radeon-vega-7)
8. [Optimasi Performa AMD Ryzen 5 5625U & RAM 12GB](#8-optimasi-performa-amd-ryzen-5-5625u--ram-12gb)
9. [Panduan Lengkap Snapshot & Pemulihan Sistem (Snapper & Timeshift)](#9-panduan-lengkap-snapshot--pemulihan-sistem-snapper--timeshift)
10. [Finalisasi & Reboot](#10-finalisasi--reboot)

---

## 1. Persiapan Awal & Boot Environment

### 1.1 Verifikasi Mode Booting (Wajib UEFI)
Jalankan perintah berikut pada live environment Arch Linux:
```bash
cat /sys/firmware/efi/fw_platform_size
```
* **Output `64`:** Sistem berjalan pada mode UEFI 64-bit (Lanjutkan).
* **Output berkas tidak ditemukan:** Sistem berjalan pada mode Legacy BIOS. Masuk ke firmware UEFI/BIOS laptop Anda, nonaktifkan CSM/Legacy Boot, dan pastikan Secure Boot dalam keadaan dimatikan (*disabled*).

### 1.2 Pengaturan Keyboard dan Font Konsol (Opsional)
```bash
loadkeys us
setfont ter-132b
```

### 1.3 Koneksi Internet
* **Kabel LAN (Ethernet):** Otomatis terkonfigurasi via DHCP.
* **Wi-Fi:** Gunakan utilitas `iwctl`:
  ```bash
  iwctl
  [iwd]# station wlan0 get-networks
  [iwd]# station wlan0 connect "NAMA_WIFI"
  [iwd]# exit
  ```
* Verifikasi konektivitas:
  ```bash
  ping -c 3 archlinux.org
  ```

### 1.4 Optimasi Mirrorlist (Reflector)
Agar proses pengunduhan paket berjalan maksimal dengan mirror terdekat:
```bash
reflector --country Indonesia,Singapore --latest 5 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

### 1.5 Sinkronisasi Jam Sistem
```bash
timedatectl set-ntp true
```
---

## 2. Skema Partisi SSD (Free Space 106,49 GB)

Alokasi ruang kosong sebesar 106,49 GB akan dibagi ke dalam skema partisi UEFI/GPT berikut:

| No | Partisi | Tipe Partisi | Ukuran | Mount Point | Keterangan |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | `/dev/nvme0n1pX` | EFI System Partition | 1 GiB (1024 MiB) | `/efi` | FAT32 (ESP - Standar UAPI ArchWiki) |
| 2 | `/dev/nvme0n1pY` | Linux Filesystem | Sisa ruang (~105,49 GiB) | `/` | BTRFS (Root Pool) |

> **Catatan Penting Dual-Boot & Standar ESP:** Sesuai dokumentasi resmi ArchWiki *EFI System Partition*, mount point `/efi` adalah standar modern pengganti `/boot/efi` yang kini telah usang (*discouraged*). Memisahkan `/efi` dan membiarkan direktori `/boot` tetap berada di dalam subvolume Btrfs `@` memastikan kernel dan modul initramfs ikut tersimpan di dalam snapshot secara konsisten. Jika pada SSD sudah ada partisi EFI Windows, **jangan diformat**, cukup pasang ke `/efi`.
### Langkah Partisi Menggunakan `cfdisk` atau `fdisk`
Identifikasi nama disk SSD Anda:
```bash
lsblk
```
*(Contoh asumsi drive NVMe adalah `/dev/nvme0n1`, atau `/dev/sda` jika menggunakan SATA SSD).*

Buka alat partisi pada ruang kosong:
```bash
cfdisk /dev/nvme0n1
```
1. Pilih area `Free space` (106.49 GB).
2. Buat partisi 1: Ukuran `1G`, pilih type `EFI System`.
3. Buat partisi 2: Gunakan sisa ruang kosong (~105.49 GB), pilih type `Linux filesystem`.
4. Pilih **[ Write ]**, ketik `yes`, lalu pilih **[ Quit ]**.

---

## 3. Format Filesystem & Konfigurasi Btrfs Subvolume

### 3.1 Format Partisi
Format partisi EFI menjadi FAT32 (lewati langkah ini jika menggunakan partisi EFI yang sudah ada):
```bash
mkfs.fat -F 32 /dev/nvme0n1p1
```

Format partisi Root menjadi BTRFS dengan label `ARCH_ROOT`:
```bash
mkfs.btrfs -L "ARCH_ROOT" -f /dev/nvme0n1p2
```

### 3.2 Pembuatan Struktur Subvolume Btrfs
Struktur subvolume dirancang mengikuti standar openSUSE/Arch Linux agar proses snapshot sistem tidak mencemari berkas log, berkas temporer, cache pacman, maupun dokumen personal user di `/home`.

1. Mount root BTRFS sementara ke `/mnt`:
   ```bash
   mount /dev/nvme0n1p2 /mnt
   ```

2. Buat subvolume yang diperlukan:
   ```bash
   btrfs subvolume create /mnt/@
   btrfs subvolume create /mnt/@home
   btrfs subvolume create /mnt/@snapshots
   btrfs subvolume create /mnt/@var_log
   btrfs subvolume create /mnt/@var_cache
   btrfs subvolume create /mnt/@var_tmp
   ```

3. Lepas mount sementara:
   ```bash
   umount /mnt
   ```

### 3.3 Mount Subvolume dengan Opsi Optimasi SSD
Opsi mount yang diterapkan:
- `noatime`: Mencegah penulisan metadata akses berkas setiap kali berkas dibaca (mengurangi beban I/O SSD secara signifikan).
- `compress=zstd:1`: Kompresi transparan Zstandard level 1. Memberikan rasio kompresi tinggi dengan beban CPU terendah pada arsitektur Zen 3, meningkatkan throughput I/O SSD dan menghemat kapasitas dari disk 106 GB.
- `ssd`: Mengaktifkan alokasi blok khusus memori flash/SSD.
- `space_cache=v2`: Caching blok bebas generasi kedua yang lebih stabil dan cepat.

Eksekusi perintah mount hierarkis berikut:
```bash
# 1. Mount subvolume root (@)
mount -o noatime,compress=zstd:1,ssd,space_cache=v2,subvol=@ /dev/nvme0n1p2 /mnt

# 2. Buat direktori mount point
mkdir -p /mnt/{efi,home,.snapshots,var/log,var/cache,var/tmp}

# 3. Mount subvolume lainnya
mount -o noatime,compress=zstd:1,ssd,space_cache=v2,subvol=@home /dev/nvme0n1p2 /mnt/home
mount -o noatime,compress=zstd:1,ssd,space_cache=v2,subvol=@snapshots /dev/nvme0n1p2 /mnt/.snapshots
mount -o noatime,compress=zstd:1,ssd,space_cache=v2,subvol=@var_log /dev/nvme0n1p2 /mnt/var/log
mount -o noatime,compress=zstd:1,ssd,space_cache=v2,subvol=@var_cache /dev/nvme0n1p2 /mnt/var/cache
mount -o noatime,compress=zstd:1,ssd,space_cache=v2,subvol=@var_tmp /dev/nvme0n1p2 /mnt/var/tmp

# 4. Mount partisi EFI ke /efi (standar modern ArchWiki)
mount /dev/nvme0n1p1 /mnt/efi

Verifikasi struktur mount dengan:
```bash
lsblk
```

---

## 4. Instalasi Sistem Dasar (Pacstrap)

Gunakan kernel **`linux-lts`** dan firmware prosesor **`amd-ucode`** sesuai instruksi target:

```bash
pacstrap -K /mnt base base-devel linux-lts linux-lts-headers linux-firmware amd-ucode btrfs-progs nano sudo networkmanager git
```

### Generate berkas `fstab`
Buat tabel partisi otomatis berdasarkan UUID:
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Periksa hasil `/mnt/etc/fstab`:
```bash
cat /mnt/etc/fstab
```
Pastikan seluruh subvolume Btrfs terdaftar dengan opsi `compress=zstd:1,ssd,space_cache=v2,subvol=@...`.

---

## 5. Konfigurasi Sistem Dasar

Masuk ke dalam instalasi baru melalui chroot:
```bash
arch-chroot -S /mnt
```
*(Atau gunakan `arch-chroot /mnt`)*
### 5.1 Zona Waktu dan Jam Sistem
```bash
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc
```
*(Ganti `Asia/Jakarta` sesuai zona waktu Anda: `Asia/Makassar` untuk WITA atau `Asia/Jayapura` untuk WIT).*

### 5.2 Lokalisasi (*Locale*)
Edit `/etc/locale.gen` dan hilangkan tanda komentar (`#`) pada:
```text
en_US.UTF-8 UTF-8
id_ID.UTF-8 UTF-8
```
Generate locale:
```bash
locale-gen
```
Tetapkan variabel sistem di `/etc/locale.conf`:
```text
LANG=en_US.UTF-8
```

### 5.3 Konfigurasi Jaringan & Hostname
Tentukan nama komputer (misalnya `arch-ryzen`):
```bash
echo "arch-ryzen" > /etc/hostname
```

Buat entri `/etc/hosts`:
```text
127.0.0.1   localhost
::1         localhost
127.0.1.1   arch-ryzen.localdomain arch-ryzen
```

Aktifkan NetworkManager agar internet langsung aktif saat reboot:
```bash
systemctl enable NetworkManager
```

### 5.4 Password Root dan Akun Pengguna
Set password root:
```bash
passwd
```

Buat user reguler dengan hak akses sudo:
```bash
useradd -m -G wheel -s /bin/bash userarch
passwd userarch
```
Buka hak akses sudo dengan menjalankan `EDITOR=nano visudo`, lalu hilangkan tanda pagar (`#`) pada baris:
```text
%wheel ALL=(ALL:ALL) ALL
```

### 5.5 Konfigurasi Mkinitcpio untuk Btrfs
Buka berkas konfigurasi initramfs:
```bash
nano /etc/mkinitcpio.conf
```
Pastikan baris `HOOKS` menggunakan urutan standar berikut (hook `microcode` telah terintegrasi langsung di dalam mkinitcpio modern):
```text
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block filesystems fsck)
```
Generate awal image kernel LTS:
```bash
mkinitcpio -p linux-lts
```

---

## 6. Konfigurasi Bootloader GRUB & Kernel Parameter

Pemasangan GRUB dengan dukungan Btrfs dan isolasi kernel di root subvolume `@` memungkinkan snapshot menyertakan modul kernel secara sinkron.

### 6.1 Instalasi Paket GRUB & EFIBootMgr
```bash
pacman -S grub efibootmgr dosfstools mtools
```

Pasang GRUB ke direktori EFI:
```bash
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=ArchLinux --recheck
```

### 6.2 Konfigurasi Kernel Parameter untuk AMD Ryzen
Edit berkas `/etc/default/grub`:
```bash
nano /etc/default/grub
```
Cari parameter `GRUB_CMDLINE_LINUX_DEFAULT` dan tambahkan parameter performa AMD & penonaktifan zswap:
```text
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet amd_pstate=active zswap.enabled=0"
```
* **`amd_pstate=active`**: Memaksa driver CPU frequency scaling AMD P-State EPP aktif pada Ryzen 5000 Mobile.
* **`zswap.enabled=0`**: Sesuai rekomendasi ArchWiki *Zram*, fitur kernel `zswap` wajib dinonaktifkan agar tidak mencegat halaman memori sebelum masuk ke ZRAM.
Generate konfigurasi GRUB:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 7. Driver Grafis AMD Radeon Vega 7

AMD Ryzen 5 5625U terintegrasi dengan AMD Radeon Vega 7 Graphics (arsitektur GCN 5th Gen / GFX9), yang didukung penuh oleh driver kernel `amdgpu` dan stack Mesa.

### 7.1 Instalasi Paket Penggerak & Vulkan
```bash
pacman -S mesa vulkan-radeon libva-mesa-driver mesa-vdpau libva-utils vdpauinfo
```

* **`mesa`**: Driver OpenGL (RadeonSI).
* **`vulkan-radeon`**: Driver Vulkan berperforma tinggi (RADV).
* **`libva-mesa-driver` & `mesa-vdpau`**: Akselerasi decoding/encoding video perangkat keras (Hardware Video Acceleration - VA-API).

Jika menggunakan repositori `[multilib]` (aplikasi 32-bit seperti Steam/Wine), aktifkan `[multilib]` pada `/etc/pacman.conf` dan pasang paket 32-bit terkait:
```bash
pacman -S lib32-mesa lib32-vulkan-radeon lib32-libva-mesa-driver
```

Verifikasi driver hardware acceleration nantinya dapat dilakukan pasca-boot dengan perintah:
```bash
vainfo
```

---

## 8. Optimasi Performa AMD Ryzen 5 5625U & RAM 12GB

### 8.1 Optimasi CPU Scaling: AMD P-State (EPP)
Driver `amd_pstate=active` memungkinkan CPU mengatur frekuensi core secara otonom dalam level milidetik.

Pasang `power-profiles-daemon` untuk mengelola profil daya CPU (sangat kompatibel dengan GNOME, KDE, atau manajemen CLI):
```bash
pacman -S power-profiles-daemon
systemctl enable power-profiles-daemon
```

* **Cek Profil Daya (setelah reboot):**
  ```bash
  powerprofilesctl list
  ```
* **Beralih ke Mode Performa Tinggi:**
  ```bash
  powerprofilesctl set performance
  ```
* **Beralih ke Mode Seimbang (Rekomendasi Laptop):**
  ```bash
  powerprofilesctl set balanced
  ```

### 8.2 Konfigurasi ZRAM untuk RAM 12 GB
Pada kapasitas RAM 12 GB dan sisa SSD 106,49 GB, menggunakan partisi swap konvensional di disk akan membuang 4–12 GB ruang berharga dan menambah keausan tulis SSD (*write wear*). Solusi standar terbaik dari ArchWiki adalah **ZRAM** (RAM swap terkompresi dengan algoritma super cepat Zstandard).

1. Pasang paket generator:
   ```bash
   pacman -S zram-generator
   ```

2. Buat berkas konfigurasi `/etc/systemd/zram-generator.conf`:
   ```ini
   [zram0]
   zram-size = ram / 2
   compression-algorithm = zstd
   swap-priority = 100
   ```
   *Alokasi `ram / 2` akan menciptakan perangkat swap virtual berkapasitas 6 GB di dalam memori dengan kompresi data rasio ~2:1 hingga 3:1.*

3. Konfigurasi Kernel Virtual Memory (VM) Sysctl untuk ZRAM:
   Buat berkas `/etc/sysctl.d/99-vm-zram-parameters.conf`:
   ```ini
   # Optimasi agresivitas penggunaan ZRAM
   vm.swappiness = 180
   vm.watermark_boost_factor = 0
   vm.watermark_scale_factor = 125
   vm.page-cluster = 0
   ```
   *Nilai `vm.swappiness = 180` merupakan rekomendasi resmi kernel untuk swap terkompresi di memori agar sistem secara proaktif memindahkan halaman memori pasif ke dalam ZRAM, menjaga RAM bebas tetap lega untuk aplikasi aktif.*

### 8.3 Pemeliharaan SSD Berkala (TRIM)
Untuk menjaga kecepatan transfer cell NAND SSD kapasitas 106 GB tetap prima:
```bash
systemctl enable fstrim.timer
```

---

## 9. Panduan Lengkap Snapshot & Pemulihan Sistem (Snapper & Timeshift)

### OPSI UTAMA: SNAPPER (Sangat Direkomendasikan untuk Integrasi Btrfs & Pacman)

Snapper dikembangkan secara native untuk Btrfs. Dalam panduan ini, Snapper dikombinasikan dengan `snap-pac` (snapshot otomatis setiap instalasi/update paket) dan `grub-btrfs` (membuat entri boot menu otomatis dari snapshot).

#### A. Instalasi Paket Snapper & Utilitas Terkait
Jalankan di dalam chroot:
```bash
pacman -S snapper snap-pac grub-btrfs inotify-tools
```

#### B. Konfigurasi Snapper untuk Subvolume Root
Karena sebelumnya kita telah me-mount `@snapshots` ke `/.snapshots`, kita harus melakukan initialisasi konfigurasi dengan urutan berikut:

1. Unmount `/.snapshots` sementara:
   ```bash
   umount /.snapshots
   rm -rf /.snapshots
   ```

2. Buat konfigurasi Snapper untuk root:
   ```bash
   snapper -c root create-config /
   ```

3. Hapus subvolume default yang otomatis dibuat oleh Snapper:
   ```bash
   btrfs subvolume delete /.snapshots
   ```

4. Buat ulang direktori `/.snapshots` dan mount kembali subvolume `@snapshots`:
   ```bash
   mkdir /.snapshots
   mount -a
   chmod 750 /.snapshots
   ```

#### C. Pengaturan Retensi Snapshot (Penting untuk SSD 106 GB)
Agar snapshot tidak memenuhi ruang penyimpanan SSD 106 GB yang terbatas, edit `/etc/snapper/configs/root`:
```bash
nano /etc/snapper/configs/root
```
Sesuaikan parameter limit timeline berikut:
```text
ALLOW_GROUPS="wheel"
TIMELINE_CREATE="yes"
TIMELINE_CLEANUP="yes"

# Batas kuota snapshot berkala untuk SSD 106GB
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="5"
TIMELINE_LIMIT_WEEKLY="2"
TIMELINE_LIMIT_MONTHLY="1"
TIMELINE_LIMIT_YEARLY="0"
```

Aktifkan timer otomatis Snapper:
```bash
systemctl enable snapper-timeline.timer
systemctl enable snapper-cleanup.timer
```

#### D. Integrasi Otomatis ke Menu Booting GRUB (`grub-btrfs`)
1. Aktifkan daemon `grub-btrfsd` agar setiap kali ada snapshot baru (baik manual maupun otomatis dari `pacman`), menu boot GRUB langsung diperbarui secara otomatis:
   ```bash
   systemctl enable grub-btrfsd
   ```

2. **Dukungan Booting Read-Only Snapshot (OverlayFS):**
   Snapshot bawaan Snapper bersifat *read-only*. Sebagian layanan sistem (seperti display manager desktop) memerlukan direktori `/var` yang dapat ditulisi saat start. Sesuai rekomendasi resmi ArchWiki *Snapper*, tambahkan hook `grub-btrfs-overlayfs` di akhir baris `HOOKS` pada `/etc/mkinitcpio.conf`:
   ```text
   HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block filesystems fsck grub-btrfs-overlayfs)
   ```
   Perbarui initramfs:
   ```bash
   mkinitcpio -p linux-lts
   ```
#### E. Penggunaan Sehari-hari Snapper (CLI)

1. **Membuat Snapshot Manual (misal sebelum utak-atik konfigurasi):**
   ```bash
   snapper -c root create -d "Sebelum modifikasi kernel / driver"
   ```

2. **Melihat Daftar Snapshot:**
   ```bash
   snapper -c root list
   ```

3. **Memeriksa Perbedaan Berkas antar Snapshot:**
   ```bash
   # Melihat status file yang berubah antara snapshot 10 dan 11:
   snapper -c root status 10..11

   # Melihat diff isi berkas:
   snapper -c root diff 10..11 /etc/fstab
   ```

4. **Menghapus Snapshot Tertentu:**
   ```bash
   snapper -c root delete 10
   # Atau hapus rentang:
   snapper -c root delete 10-15 --sync
   ```

5. **Mengembalikan Berkas Individual dari Snapshot:**
   ```bash
   # Mengembalikan file /etc/default/grub dari snapshot nomor 5:
   snapper -c root undochange 5..0 /etc/default/grub
   ```

#### F. Prosedur Pemulihan Sistem (Rollback)

##### Skenario 1: Sistem Rusak / Gagal Booting Pasca Update
1. Nyalakan laptop, pada menu GRUB pilih submenu: **"Arch Linux snapshots"**.
2. Pilih snapshot kondisi sebelum sistem bermasalah (tersedia versi read-only).
3. Sistem akan berhasil boot ke dalam desktop/terminal dari snapshot tersebut.
4. Buka terminal dan lakukan rollback menggunakan `snapper-rollback` (atau restore subvolume `@` secara manual).

##### Skenario 2: Rollback Darurat Menggunakan Arch Live USB
Jika sistem tidak bisa masuk ke GRUB sama sekali:
1. Boot menggunakan Flashdisk Live Arch Linux.
2. Mount toplevel BTRFS (subvolid=5):
   ```bash
   mount -t btrfs -o subvolid=5 /dev/nvme0n1p2 /mnt
   ```
3. Cek daftar snapshot yang tersedia:
   ```bash
   ls /mnt/@snapshots/*/snapshot
   ```
4. Pindahkan subvolume `@` yang rusak ke backup:
   ```bash
   mv /mnt/@ /mnt/@_broken
   ```
5. Buat subvolume `@` baru yang merupakan clone read-write dari snapshot pilihan (contoh snapshot nomor 5):
   ```bash
   btrfs subvolume snapshot /mnt/@snapshots/5/snapshot /mnt/@
   ```
6. Lepas mount dan restart:
   ```bash
   umount /mnt
   reboot
   ```
*Sistem Anda kini kembali 100% normal seperti pada snapshot nomor 5 tanpa memengaruhi data pada `/home`.*

---

### OPSI ALTERNATIF: TIMESHIFT (Jika Lebih Memilih GUI)

Bagi pengguna yang menginginkan tampilan grafis (*Graphical User Interface*), Timeshift dapat digunakan dengan tata letak subvolume yang telah kita buat.

1. **Persyaratan:**
   Timeshift pada mode BTRFS mewajibkan struktur subvolume bernama `@` (untuk `/`) dan opsional `@home` (untuk `/home`). Struktur yang telah kita pasang pada Bab 3 sudah memenuhi standar ini secara tepat.

2. **Instalasi Paket (dijalankan setelah desktop environment terpasang):**
   ```bash
   # Dari repositori resmi:
   pacman -S timeshift

   # Otomatisasi snapshot saat pacman update (tersedia via AUR):
   # yay -S timeshift-autosnap grub-btrfs
   ```

3. **Konfigurasi Timeshift:**
   - Buka aplikasi **Timeshift**.
   - Pilih tipe snapshot: **BTRFS**.
   - Pilih drive partisi BTRFS Anda.
   - Pada tab **Schedule**, pilih jadwal penyimpanan: misalnya 3 Daily dan 2 Boot snapshots (jangan terlalu banyak untuk menghemat disk 106 GB).
   - Klik **Create** untuk membuat snapshot perdana.

4. **Prosedur Restore via Timeshift:**
   - Buka Timeshift -> Pilih tanggal snapshot yang diinginkan -> Klik **Restore**.
   - Timeshift akan secara otomatis mengganti subvolume `@` lama dengan snapshot yang dipilih, lalu meminta reboot sistem.

---

## 10. Finalisasi & Reboot

1. Buat ulang konfigurasi GRUB final untuk memastikan seluruh hook terdaftar:
   ```bash
   grub-mkconfig -o /boot/grub/grub.cfg
   ```

2. Keluar dari lingkungan chroot:
   ```bash
   exit
   ```

3. Lepas semua partisi yang ter-mount secara bersih:
   ```bash
   umount -R /mnt
   ```

4. Nyalakan ulang komputer dan cabut Flashdisk instalasi:
   ```bash
   reboot
   ```

Sistem Arch Linux berbasis kernel LTS dengan sistem berkas BTRFS, perlindungan snapshot Snapper, dan optimasi hardware AMD Ryzen 5 5625U siap digunakan.
