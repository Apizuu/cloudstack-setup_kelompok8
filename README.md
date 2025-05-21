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
> Untuk memeriksa apakah konfigurasi jaringan sudah diterapkan, gunakan perintah ifconfig dan cari interface "br0", pastikan alamat IP-nya sesuai dengan yang sudah diatur.

### Uji Jaringan, pastikan konfigurasi sudah diterapkan

```
ifconfig     #cek alamat IP dan interface yang ada
ping -c 20 google.com  #pastikan dapat terhubung ke internet
```

> Jika tidak bisa ping ke google.com, coba ping ke gateway dan 8.8.8.8.
> Langkah ini akan membantu mengetahui masalah koneksi antara komputer dan internet, pastikan tidak ada masalah karena akan mengunduh paket dari internet.

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
tailscale up                                       #aktifkan Tailscale dan login dengan akun 
```

Setelah menjalankan perintah di atas, ikuti instruksi pada terminal untuk login ke akun Tailscale melalui browser. Setelah berhasil login, server akan terhubung ke jaringan privat Tailscale dan dapat diakses dari perangkat lain yang juga terhubung ke jaringan yang sama.

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

> Pastikan semua konfigurasi sudah sesuai dengan IP server (`192.168.1.10`) dan storage sudah tersedia sebelum melanjutkan ke konfigurasi CloudStack berikutnya.

### Konfigurasi CloudStack
-

### Konfigurasi untuk Docker Support dan Layanan Lainnya
```
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```
**Penjelasan:**
- Mengatur agar paket ARP dan IP yang melalui bridge tidak diproses oleh arptables dan iptables, menghindari konflik dengan container seperti Docker yang memakai bridge networking.
- ```sysctl -p``` untuk menerapkan konfigurasi tersebut.

## Generate Unique Host ID
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```
> UUID digunakan oleh libvirtd untuk mengidentifikasi host secara unik dalam lingkungan virtualisasi. Konfigurasi ditambahkan ke libvirtd.conf, layanan perlu di-restart.

## Configure Iptables Firewall and Make it Persistent
```
NETWORK=192.168.1.0/24
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 32803 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 32769 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 875 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 662 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8250 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8080 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8443 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 9090 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 16514 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 3128 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 3128 -j ACCEPT

apt-get install iptables-persistent
```
> input y/n isi dengan yes dan yes
**Penjelasan:**
- Membuka port yang digunakan CloudStack dan layanan terkait (seperti NFS, libvirt, KVM) untuk jaringan lokal 192.168.101.0/24.
- -m state --state NEW: hanya berlaku untuk koneksi yang baru masuk.
- apt-get install iptables-persistent: menyimpan aturan iptables agar tetap aktif setelah reboot.


## Disable AppArmor on libvirtd
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```
**Penjelasan:**
- AppArmor adalah sistem keamanan berbasis profil. Untuk mempermudah pengelolaan libvirtd, profilnya dinonaktifkan.
- Menggunakan symbolic link ke folder disable dan menghapus profil dari kernel dengan apparmor_parser -R.

## Launch Management Server
```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
```
> cloudstack-setup-management digunakan untuk menginisialisasi server manajemen CloudStack (konfigurasi DB, IP, dan service). Command di bawahnya akan mengecek status layanan CLoudStack dan memantau log secara langsung untuk troubleshooting.
>

## Open Web Browser to Access Dashboard
```
http://<IP-DINAMIS>:8080
```
> IP dinamis adalah IP Address dari host.


## Multi-host Cloud Infrastructure: Importing CloudStack Repositories Key
```
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg
echo deb [signed-by=...] ... > /etc/apt/sources.list.d/cloudstack.list
```
**Penjelasan**
Untuk menambahkan repositori resmi CloudStack dari ShapeBlue ke sistem. Kunci GPG diimpor untuk memverifikasi paket dari repositori tersebut.

### Installing KVM Host and CloudStack-Agent
```
apt-get install qemu-kvm cloudstack-agent
```
> Menginstal hypervisor KVM (qemu-kvm) dan agen CloudStack yang diperlukan agar host bisa dikontrol oleh manajemen server.

### Configure Qemu KVM Virtualisation Management (libvirtd)
```
# Mengatur VNC agar bisa menerima koneksi dari semua IP (remote akses ke VM via VNC)
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# Mengaktifkan mode listening di libvirtd agar menerima koneksi TCP dari remote (CloudStack management server)
sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd

# Menonaktifkan TLS karena tidak digunakan
echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf

# Mengaktifkan koneksi TCP biasa
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf

# Menentukan port TCP yang digunakan untuk koneksi (default libvirt TCP port)
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf

# Mematikan fitur Multicast DNS (mDNS) karena tidak diperlukan dalam konfigurasi ini
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf

# Menonaktifkan autentikasi pada koneksi TCP (gunakan ini hanya jika jaringan internal aman)
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

# Menonaktifkan semua socket libvirtd default yang tidak diperlukan untuk menghindari konflik
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket

# Me-restart layanan libvirtd agar semua perubahan konfigurasi diterapkan
systemctl restart libvirtd
```

## Setup on GUI : Creating Pod, Zone, Cluster, Host, ISO, and Instance. (VIERIN)

## Network Architecture Setup: Setup VM, Expose subnet route on Tailscale, Creating SNAT rules, and Final Topology. (HAFIZ)

## Setup VM
```bash
vm1@vm1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> ...
    inet 127.0.0.1/8 scope host lo
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.1.114/24 metric 100 brd 192.168.1.255 scope global ens3
```

## Expose subnet route

Jalankan perintah berikut pada laptop host di jaringan `192.168.1.0/24` dan yang telah login Tailscale:

```bash
sudo tailscale up --advertise-routes=192.168.1.0/24
````

### Approve route dari admin console

Masuk ke [Tailscale Admin Console](https://login.tailscale.com/admin) lalu:

* Buka tab **Machines**
* Cari perangkat host untuk advertise route
* Klik **Review route** lalu **Enable** untuk subnet `192.168.1.0/24`



### Cek konektivitas

Setelah route disetujui, perangkat lain di jaringan Tailscale akan dapat mengakses alamat IP host dalam subnet `192.168.1.0/24`.

Contoh:

```bash
ping 192.168.1.114
```

* aktifkan IP forwarding untuk akses dua arah:


  ```bash
  sudo sysctl -w net.ipv4.ip_forward=1
  ```

## Creating SNAT rules

### Situasi

- Laptop Host:
  - IP Lokal: `192.168.1.10`
  - IP Tailscale: `100.66.37.47`
- VM berjalan di dalam laptop host:
  - IP VM: `192.168.1.114`
- Perangkat lain di jaringan Tailscale:
  - IP Tailscale: `100.124.10.100` (laptop lain)

Perangkat dengan IP Tailscale `100.124.10.100` ingin melakukan ping ke `192.168.1.114` (VM).

---

## Problem

- Ping dari `100.124.10.100` berhasil **masuk ke VM**, namun gagal mendapatkan **reply**.
- Setelah ditelusuri, reply dari VM tidak kembali ke `100.124.10.100`, tapi malah dilempar ke **default gateway rumah** (`192.168.1.1`).
- Ini terjadi karena VM tidak mengenali IP Tailscale `100.124.10.100` sebagai bagian dari jaringan yang valid.

---

## Penyebab

- Saat ping masuk, source IP-nya adalah `100.124.10.100`.
- VM tidak tahu harus balas ke mana, karena tidak ada routing ke IP tersebut.
- Akibatnya, VM mengirim balasan ke default gateway (`192.168.1.1`) dan bukannya kembali melalui laptop host.

---

## Solusi

Agar VM bisa mengenali sumber ping dan membalas melalui jalur yang benar, kita perlu mengubah source IP saat paket diteruskan dari Tailscale ke VM. Caranya adalah dengan menggunakan iptables SNAT (dalam bentuk MASQUERADE).

Tambahkan rule berikut di laptop host (bukan di VM):

```bash
sudo iptables -t nat -A POSTROUTING -o cloudbr0 -j MASQUERADE
````

Penjelasan:

* `-t nat`: mengatur tabel NAT
* `-A POSTROUTING`: menambahkan rule di tahap setelah routing diproses
* `-o cloudbr0`: interface virtual bridge (jaringan antara host dan VM)
* `-j MASQUERADE`: mengganti source IP menjadi IP dari interface tersebut (dalam hal ini `192.168.1.10`)

---

## Hasil

Setelah rule diterapkan:

* Saat laptop lain (100.124.10.100) ping ke `192.168.1.114`, paket diteruskan dari Tailscale ke VM melalui laptop host.
* Source IP diubah dari `100.124.10.100` menjadi `192.168.1.10`.
* VM mengenali dan merespon ke IP `192.168.1.10`.
* Laptop host kemudian meneruskan kembali balasan ke `100.124.10.100` melalui Tailscale.

---

### Final Topology
![image](https://hackmd.io/_uploads/BkNaPVoWge.png)

## Service Setup: SSH and Http via apache (HAFIZ)


### Instalasi OpenSSH Server

```bash
sudo apt install openssh-server -y
```

### Cek Status SSH

```bash
sudo systemctl status ssh
```

### Jalankan SSH

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

### Cek IP Address untuk Akses

```bash
ip a
```

Gunakan IP tersebut untuk mengakses SSH dari perangkat lain:

```bash
ssh vm1@192.168.1.114
```

---

## Setup HTTP Service (Apache)

### Instalasi Apache2

```bash
sudo apt install apache2 -y
```

### Cek Status Apache

```bash
sudo systemctl status apache2
```

### Jalankan Apache (jika belum aktif)

```bash
sudo systemctl enable apache2
sudo systemctl start apache2
```

### Tes di Browser

Buka browser dan akses:

```
http://localhost
```

atau dari perangkat lain:

```
http://192.168.1.114
```
![image](https://hackmd.io/_uploads/Bk-PiVoWlg.png)


---


