# Instalasi dan Konfigurasi Apache CloudStack
Kelompok 8:
- Hafizyah Rayhan Zulikhram (2206029185)
- Monica Vierin Pasman (2206029405)
- Muhammad Fahish Haritsah (2206059616)
- Valentino Farish Adrian (2206825896)
- Stefanus Simon Rilando (2206830422)

## Dasar Teori
### CloudStack
Apache CloudStack merupakan perangkat lunak open-source yang dirancang untuk membangun, menjalankan, dan mengelola jaringan besar sebagai platform komputasi awan IaaS (Infrastructure as a Service). CloudStack digunakan oleh banyak penyedia layanan dan perusahaan untuk menawarkan layanan cloud publik, layanan cloud on-premises/private, atau sebagai solusi dari hybrid cloud. Pengguna dapat mengelola cloud-nya sendiri melalui antarmuka web (GUI), command line (CLI), dan full-featured RESTful API. API yang disediakan telah kompatibel dengan AWS EC2 dan S3 bagi organisasi yang ingin men-deploy hybrid cloud. Platform ini menyediakan lapisan orkestrasi yang otomatis membuat dan mengatur komponen-komponen infrastruktur virtual, serta mengubah infrastruktur virtual tersebut menjadi platform cloud IaaS yang siap digunakan. Fitur-fitur yang disediakan CloudStack terdiri dari:
- pengelolaan otomatis dalam pembuatan dan penghapusan VM;
- manajemen jaringan virtual (router, firewall, load balancer, dan IP publik);
- dukungan multi-tenant (banyak pengguna dan organisasi); serta
- perhitungan penggunaan sumber daya (CPU, RAM, dan penyimpanan) per pengguna.


### TailScale
Tailscale adalah solusi VPN berbasis WireGuard yang memudahkan pembuatan jaringan privat antar perangkat secara otomatis dan aman, tanpa perlu konfigurasi firewall atau port forwarding yang rumit. Dalam konteks ini, Tailscale dapat digunakan untuk perangkat administrator secara aman melalui jaringan privat, sehingga memudahkan akses remote, monitoring, dan manajemen infrastruktur cloud dari mana saja.

### KVM
KVM (Kernel-based Virtual Machine) adalah teknologi virtualisasi open-source yang terintegrasi langsung dengan kernel Linux. KVM memungkinkan satu server fisik menjalankan banyak mesin virtual (VM) dengan performa mendekati bare-metal. Dalam penerapan cloud IaaS menggunakan Apache CloudStack, KVM berperan sebagai hypervisor utama untuk menjalankan dan mengelola VM yang disediakan kepada pengguna, sehingga memungkinkan isolasi, alokasi sumber daya dinamis, dan skalabilitas infrastruktur cloud.

## Computing Environment
### System Requirement
```
RAM: 24 GB
Storage: 250GB
CPU: Intel Core i5 gen 8
Network: Ethernet 100GB/s
OS: Ubuntu Live Server 22.04
```

### IP Address
```
Alamat IP jaringan: 192.168.1.0/24
Alamat IP host: 192.168.1.10/24
Alamat IP gateway: 192.168.1.1
Alamat IP TailScale: 100.66.37.47/32
Alamat IP manajemen:
Alamat IP sistem:
Alamat IP publik:
```

## Konfigurasi Jaringan

### Mendefinisikan konfigurasi jaringan dengan Netplan pada /etc/netplan
```
cd /etc/netplan
sudo nano
```
### Edit file dengan menetapkan Bridge 
```
# Updated in 28/4/2025
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: true
      dhcp6: false # Disable IPv6, could change it for later uses
      optional: true
  bridges:
    cloudbr0:
      addresses: [192.168.1.10/24] # host IP address
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1,8.8.8.8]
      interfaces: [enp1s0]
      dhcp4: false
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```
### Terapkan Konfigurasi Jaringan

```
sudo -i  #buka shell baru dengan hak akses root
netplan generate #menghasilkan file konfigurasi untuk renderer
netplan apply  #menerapkan konfigurasi jaringan ke sistem
reboot #restart sistem
```
> Untuk memeriksa apakah konfigurasi jaringan sudah diterapkan, gunakan perintah ifconfig dan cari interface "br0", pastikan alamat IP-nya sesuai dengan yang sudah Anda atur.

### Uji Jaringan, pastikan konfigurasi sudah diterapkan

```
ifconfig     #cek alamat IP dan interface yang ada
ping -c 20 google.com  #pastikan Anda dapat terhubung ke internet
```

> Jika Anda tidak bisa ping ke google.com, coba ping ke gateway dan 8.8.8.8.
> Langkah ini akan membantu Anda mengetahui masalah koneksi antara komputer dan internet, pastikan tidak ada masalah karena Anda akan mengunduh paket dari internet.

### Masuk ke sistem sebagai root

```
su -
```

### Instalasi alat monitoring sumber daya hardware

```
apt update -y
apt upgrade -y
apt install htop lynx duf -y
apt install bridge-utils
```

* htop adalah alat monitoring penggunaan CPU
* duf adalah alat monitoring penggunaan disk
* lynx adalah browser web berbasis CLI

### Konfigurasi LVM (opsional)

```
#tidak wajib kecuali menggunakan logical volume
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```

### Instalasi layanan jaringan

```
apt-get install openntpd openssh-server sudo tar -y
apt-get install intel-microcode -y
passwd root
#ganti password root, misal: Pa$$w0rd
```

* openntpd adalah klien NTP untuk sinkronisasi waktu antara host dan internet
* openssh-server adalah server SSH untuk akses remote
* tar adalah alat kompresi dan dekompresi file, sering digunakan untuk file yang diunduh
* intel-microcode adalah kumpulan prosedur untuk meningkatkan modularitas tingkat rendah pada prosesor Intel

### Aktifkan login root via SSH

```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
#restart layanan ssh
service ssh restart
#atau
systemctl restart sshd.service
```

### Periksa Konfigurasi SSH
```
nano /etc/ssh/sshd_config
```
Cari baris 'PermitRootLogin' dan pastikan nilainya 'yes'

## Instalasi Tailscale

```
curl -fsSL https://tailscale.com/install.sh | sh   #unduh dan instalasi Tailscale
tailscale up                                       #aktifkan Tailscale dan login dengan akun Anda
```

Setelah menjalankan perintah di atas, ikuti instruksi pada terminal untuk login ke akun Tailscale Anda melalui browser. Setelah berhasil login, server Anda akan terhubung ke jaringan privat Tailscale dan dapat diakses dari perangkat lain yang juga terhubung ke jaringan yang sama.

## Instalasi CloudStack

### Import Key Repository dari CloudStack

```
sudo -i
mkdir -p /etc/apt/keyrings 
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list
```

* Baris pertama membuat direktori untuk menyimpan kunci publik cloudstack.
* Perintah wget -O digunakan untuk mengunduh URL dan mengarahkan output ke perintah `gpg --dearmor`.
* Perintah `gpg --dearmor` mengubah format ASCII menjadi format biner.
* Perintah `sudo tee` mengarahkan hasil ke file `/etc/apt/keyrings/cloudstack.gpg`.

### Periksa Repositori yang Ditambahkan

```
nano /etc/apt/sources.list.d/cloudstack.list
```
Pastikan terdapat baris berikut pada bagian akhir file:
```
deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 /
```

### Instalasi CloudStack dan MySQL Server

```
apt-get update -y
apt-get install cloudstack-management mysql-server
```

### Konfigurasi MySQL

#### Buka file konfigurasi MySQL

```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

#### Tambahkan baris berikut di bawah bagian [mysqld]

```
server-id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'
```

#### Restart layanan MySQL

```
systemctl restart mysql
```

#### Cek status layanan MySQL

```
systemctl status mysql
```
Pastikan statusnya 'active'.

### Deploy Database sebagai Root dan Buat User "cloud" dengan Password "cloud"

```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:Pa$$w0rd -i 192.168.1.10
```

### Konfigurasi Primary dan Secondary Storage

```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

### Konfigurasi NFS Server

```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

**Penjelasan:**
- Perintah `sed` digunakan untuk mengubah konfigurasi port dan opsi pada file konfigurasi NFS.
- `echo "NEED_STATD=yes"` menambahkan baris agar layanan statd aktif.
- `service nfs-kernel-server restart` untuk me-restart layanan NFS.

> Pastikan semua konfigurasi sudah sesuai dengan IP server Anda (`192.168.1.10`) dan storage sudah tersedia sebelum melanjutkan ke konfigurasi CloudStack berikutnya.

## Konfigurasi CloudStack