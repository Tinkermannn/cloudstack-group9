# Cloud 9: Apache CloudStack Setup on VirtualBox Ubuntu Server 24.04


---

## Spesifikasi

| Komponen | Detail |
|---|---|
| OS | Ubuntu Server 24.04.4 LTS |
| CloudStack | 4.18.2.5 |
| Hypervisor | KVM |
| Virtualisasi | VirtualBox (NAT Network) |
| Akses Remote | Tailscale |

---

## Dependencies

Sebelum memulai, penting untuk memahami komponen utama yang digunakan dalam setup ini dan peran masing-masing.

- **Apache CloudStack**  
Platform IaaS (Infrastructure as a Service) open source yang menjadi inti dari setup ini. CloudStack bertanggung jawab untuk mengelola seluruh siklus hidup VM, mulai dari provisioning hingga penghapusan.
- **MySQL**
Digunakan sebagai database utama CloudStack untuk menyimpan seluruh konfigurasi, metadata operasional, status sistem, akun pengguna, detail VM, konfigurasi jaringan, dan job history.
- **KVM (Kernel-based Virtual Machine)**
Hypervisor yang digunakan CloudStack untuk membuat dan mengelola VM di atas host fisik. Dipilih karena stabilitasnya, status open-source, dan integrasi native dengan Linux.
- **NFS (Network File System)**
Digunakan sebagai primary storage (menyimpan disk volume VM) dan secondary storage (menyimpan ISO, template, dan snapshot).
- **libvirtd**
Daemon manajemen virtualisasi yang menjadi jembatan antara CloudStack Agent dengan KVM. CloudStack berkomunikasi dengan KVM melalui libvirtd via TCP port 16509.
- **Tailscale**
VPN mesh berbasis WireGuard yang digunakan untuk mengakses dashboard CloudStack dari luar jaringan VirtualBox.

---

## Prasyarat

Setup ini menggunakan VirtualBox dengan beberapa konfigurasi khusus. Perhatikan hal-hal berikut sebelum memulai.

- VirtualBox sudah terinstall di komputer host
- Ubuntu Server 24.04 sudah terinstall di VirtualBox
- Network adapter diset ke **NAT Network** bukan Bridged Adapter, karena sebagian besar driver WiFi di Windows tidak mendukung promiscuous mode yang dibutuhkan untuk bridge networking di VirtualBox
- Promiscuous Mode diset ke **Allow All** di pengaturan adapter
- Nested Virtualization diaktifkan melalui Settings → System → Processor → Enable Nested VT-x/AMD-V, agar KVM bisa berjalan di dalam VirtualBox
- RAM VM minimal **4 GB**: CloudStack, KVM, MySQL, dan System VMs membutuhkan memori yang cukup besar
- Disk VM minimal **25 GB**: CloudStack akan otomatis mendownload System VM Template sekitar 1.65 GB saat pertama kali dijalankan

---

## Arsitektur
![image](https://hackmd.io/_uploads/rygIDYVAlMe.png)

---

### Penjelasan Arsitektur

**Host PC** satu-satunya mesin fisik yang ada. Di atasnya berjalan VirtualBox yang membuat jaringan virtual (NAT Network) terisolasi dari jaringan WiFi rumah.

Di dalam VirtualBox, berjalan satu **Ubuntu Server VM** yang menjalankan semua peran sekaligus management server, database, hypervisor, dan storage.

Di dalam VM tersebut, **CloudStack + KVM** bekerja bersama. CloudStack berperan sebagai otak yang mengatur infrastruktur, sementara KVM adalah mesin yang benar-benar menjalankan VM-VM guest. KVM kemudian membuat jaringan terisolasi tersendiri (10.1.1.0/24) khusus untuk VM guest, terpisah dari jaringan host.

Untuk akses dari luar, ada dua jalur masuk:
- **Jalur normal**: lewat router LAN rumah → masuk ke VM via NAT VirtualBox
- **Jalur remote**: lewat tools seperti Tailscale atau Ngrok yang membuat tunnel langsung ke dalam VM, memungkinkan akses dari mana saja tanpa perlu berada di jaringan yang sama

---

### Keterbatasan Single Node

Semua berjalan di satu mesin, setup ini hanya cocok untuk **keperluan lab dan pembelajaran**. Pada deployment nyata, setiap komponen (management server, compute node, storage) biasanya dipisah ke mesin fisik yang berbeda untuk keandalan dan skalabilitas.

## Step-by-Step Installation

### Step 1: Cek IP dan Network Interface

Langkah pertama adalah mengidentifikasi nama interface jaringan dan IP address yang didapat dari DHCP. Nama interface ini (misalnya `enp0s3`) akan digunakan pada konfigurasi bridge di langkah berikutnya.

```bash
ip addr show
```

Catat nama interface dan IP address yang muncul di bawahnya.

---

### Step 2: Konfigurasi Network Bridge (Netplan)

CloudStack membutuhkan network bridge bernama `cloudbr0` sebagai jembatan antara VM-VM yang berjalan di atas KVM dengan jaringan fisik host. Bridge ini memungkinkan VM mendapatkan konektivitas jaringan seolah-olah terhubung langsung ke switch fisik.

Di Ubuntu 24.04, konfigurasi jaringan dikelola oleh Netplan. Kita perlu menonaktifkan DHCP pada interface fisik dan mengalihkan IP ke bridge `cloudbr0`.

Masuk sebagai root terlebih dahulu:
```bash
sudo -i
cd /etc/netplan
nano ./01-dhcp.yaml
```

Ganti seluruh isi file dengan konfigurasi berikut:
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    cloudbr0:
      addresses: [192.168.1.4/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1,8.8.8.8]
      interfaces: [enp0s3]
      dhcp4: false
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```

Sesuaikan `addresses` dengan IP statis yang diinginkan dan `via` dengan IP gateway/router. STP dinonaktifkan dan `forward-delay` diset ke 0 agar bridge langsung aktif tanpa delay.

```bash
netplan apply
ping -c 3 8.8.8.8
```

---

### Step 3: Install Monitoring Tools

Sebelum melanjutkan ke instalasi komponen utama, install beberapa tools yang berguna untuk memantau kondisi sistem selama proses setup berlangsung.

```bash
apt update -y && apt upgrade -y
apt install htop lynx duf bridge-utils -y
```

- `htop` digunakan untuk memantau penggunaan CPU dan proses yang berjalan
- `duf` digunakan untuk memantau penggunaan disk
- `lynx` adalah browser berbasis teks untuk menguji konektivitas
- `bridge-utils` menyediakan perintah-perintah untuk mengelola network bridge

---

### Step 4: Konfigurasi SSH

SSH diperlukan agar kita bisa mengakses server dari komputer lain, termasuk untuk keperluan remote management dan debugging. Root login diaktifkan karena CloudStack Agent membutuhkan akses root saat melakukan koneksi ke host.

```bash
apt install openssh-server -y
systemctl start ssh
systemctl enable ssh

sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
systemctl restart ssh
```

---

### Step 5: Install Apache CloudStack dan MySQL

#### 5a. Tambah Repository CloudStack

Package CloudStack tidak tersedia di repository Ubuntu default, sehingga kita perlu menambahkan repository resmi dari ShapeBlue, yang merupakan maintainer utama CloudStack. GPG key digunakan untuk memverifikasi keaslian package yang didownload.

```bash
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list
apt-get update -y
```

#### 5b. Install CloudStack dan MySQL

```bash
apt-get install cloudstack-management mysql-server -y
```

#### 5c. Konfigurasi MySQL

MySQL perlu dikonfigurasi secara khusus agar kompatibel dengan kebutuhan CloudStack. Beberapa parameter penting yang disetel antara lain: `innodb_lock_wait_timeout` untuk mencegah timeout pada operasi database yang lama, `max_connections` untuk mengakomodasi banyak koneksi dari CloudStack, dan binary logging yang diaktifkan untuk mendukung replikasi dan recovery.

```bash
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Tambahkan di bawah section `[mysqld]`:
```
server-id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'
```

Setelah konfigurasi disimpan, restart MySQL dan inisialisasi database CloudStack. Perintah ini akan membuat database `cloud` beserta user `cloud` dengan password `cloud`, serta menginisialisasi seluruh skema tabel yang dibutuhkan.

```bash
systemctl restart mysql
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:<password> -i <IP_server>
```

Ganti `<password>` dengan password root Ubuntu dan `<IP_server>` dengan IP address server.

#### 5d. Konfigurasi NFS Storage

NFS digunakan sebagai backend storage CloudStack. Primary storage menyimpan disk volume VM yang sedang berjalan, sementara secondary storage menyimpan ISO, template VM, dan snapshot. Port-port spesifik ditetapkan untuk menghindari konflik dan memudahkan konfigurasi firewall.

```bash
apt-get install nfs-kernel-server quota -y

echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a

sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

---

### Step 6: Setup KVM Hypervisor

#### 6a. Install KVM dan CloudStack Agent

`qemu-kvm` adalah implementasi KVM yang digunakan untuk menjalankan VM. `cloudstack-agent` adalah daemon yang berjalan di host dan bertindak sebagai jembatan antara CloudStack Management Server dengan KVM, menerima instruksi untuk membuat, menghapus, atau memodifikasi VM. `ifupdown` diinstall karena dibutuhkan oleh script setup agent CloudStack.

```bash
apt-get install qemu-kvm cloudstack-agent ifupdown -y
```

#### 6b. Konfigurasi libvirt

VNC dikonfigurasi untuk listen di semua interface agar console VM bisa diakses dari jaringan. libvirtd dikonfigurasi untuk menerima koneksi TCP tanpa autentikasi pada port 16509, karena CloudStack Management Server perlu berkomunikasi dengan libvirtd secara langsung. mDNS dinonaktifkan untuk menghindari konflik di jaringan.

```bash
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf
```

#### 6c. Aktifkan TCP Listening via systemd Override

Di Ubuntu 24.04, libvirtd dikelola sepenuhnya oleh systemd dengan mekanisme socket activation. Pendekatan lama menggunakan `LIBVIRTD_ARGS="--listen"` di `/etc/default/libvirtd` tidak bekerja di Ubuntu 24.04 karena systemd mengabaikan file tersebut. Solusinya adalah membuat systemd service override yang secara eksplisit menambahkan flag `--listen` ke perintah ExecStart libvirtd.

Socket-socket bawaan libvirtd juga di-mask agar tidak berkonflik dengan mode TCP listening yang kita aktifkan.

```bash
mkdir -p /etc/systemd/system/libvirtd.service.d/
cat > /etc/systemd/system/libvirtd.service.d/override.conf << 'EOF'
[Service]
ExecStart=
ExecStart=/usr/sbin/libvirtd --listen
EOF

systemctl daemon-reload
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd

# Verifikasi port 16509 sudah terbuka
ss -tlnp | grep 16509
```

#### 6d. Konfigurasi sysctl untuk Bridge Networking

Secara default, Linux akan memfilter paket yang melewati bridge melalui iptables dan arptables. Untuk CloudStack, filtering ini perlu dinonaktifkan agar traffic antar VM dan ke luar jaringan dapat mengalir tanpa hambatan yang tidak perlu.

```bash
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

#### 6e. Buat UUID Unik untuk Host

Setiap host dalam CloudStack harus memiliki identitas unik. UUID ini digunakan oleh CloudStack untuk mengidentifikasi host secara konsisten, terutama penting jika ada beberapa host dalam satu cluster.

```bash
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

#### 6f. Setup Iptables Firewall

Aturan firewall ini membuka port-port yang diperlukan untuk komunikasi antara CloudStack Management Server, KVM host, dan layanan NFS. Semua rule dibatasi hanya untuk subnet lokal (`192.168.1.0/24`) sebagai langkah keamanan. `iptables-persistent` diinstall agar rules tetap tersimpan setelah reboot.

```bash
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
apt-get install iptables-persistent -y
```

Pilih **Yes** saat ditanya untuk menyimpan rules yang ada.

#### 6g. Nonaktifkan AppArmor untuk libvirt

AppArmor adalah sistem keamanan Linux yang membatasi akses program ke resource sistem. Profile AppArmor bawaan untuk libvirtd terkadang terlalu restriktif dan dapat mengganggu operasi virtualisasi CloudStack, seperti akses ke storage dan network. Oleh karena itu profile ini perlu dinonaktifkan.

```bash
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

#### 6h. Buat File `/etc/network/interfaces`

Ubuntu 24.04 menggunakan Netplan sebagai sistem konfigurasi jaringan, sehingga file `/etc/network/interfaces` tidak ada secara default. Namun script `cloudstack-setup-agent` masih mencari file ini untuk membaca konfigurasi bridge. Kita perlu membuatnya secara manual agar proses setup agent dapat berjalan.

```bash
cat > /etc/network/interfaces << 'EOF'
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet manual

auto cloudbr0
iface cloudbr0 inet static
    address 192.168.1.4
    netmask 255.255.255.0
    gateway 192.168.1.1
    bridge_ports enp0s3
    bridge_stp off
    bridge_fd 0
    bridge_maxwait 0
EOF
```

#### 6i. Konfigurasi CloudStack Agent

File `agent.properties` adalah file konfigurasi utama CloudStack Agent yang menentukan bagaimana agent terhubung ke Management Server dan mengidentifikasi resource jaringan yang akan digunakan. GUID yang unik wajib diisi agar agent dapat dikenali oleh Management Server.

```bash
UUID=$(uuidgen)
sed -i "s/^guid=$/guid=$UUID/" /etc/cloudstack/agent/agent.properties
```

Buka file dan pastikan baris-baris berikut sudah terisi dengan benar:

```bash
nano /etc/cloudstack/agent/agent.properties
```

```
guid=<UUID yang digenerate>
host=192.168.1.4
cluster=1
pod=1
zone=1
public.network.device=cloudbr0
private.network.device=cloudbr0
guest.network.device=cloudbr0
```

#### 6j. Start CloudStack Management Server

Setelah semua konfigurasi selesai, jalankan perintah setup management untuk menyelesaikan inisialisasi, lalu pantau log untuk memastikan server berhasil start. Proses ini bisa memakan waktu 5-15 menit.

```bash
cloudstack-setup-management
systemctl restart cloudstack-agent
systemctl status cloudstack-management

tail -f /var/log/cloudstack/management/management-server.log
```

Tekan `Ctrl+C` setelah muncul pesan yang menandakan server sudah berjalan.

---

### Step 7: Akses Dashboard CloudStack

Setelah Management Server berjalan, dashboard dapat diakses melalui browser. Karena setup ini menggunakan VirtualBox NAT Network, IP server tidak bisa diakses langsung dari luar. Solusinya adalah menggunakan Tailscale yang menyediakan IP yang dapat dijangkau dari mana saja.

Buka browser dan akses:
```
http://<TAILSCALE_IP>:8080/client
```

Login dengan kredensial default:
```
Username : admin
Password : password
```

Jika berhasil, dashboard CloudStack akan tampil dan setup infrastruktur cloud dapat dilanjutkan.

---

## Catatan Penting

**VirtualBox dan WiFi Bridging**  
Sebagian besar driver WiFi di Windows tidak mendukung promiscuous mode yang dibutuhkan untuk bridge networking di VirtualBox. Karena itu setup ini menggunakan NAT Network sebagai alternatif yang lebih reliable.

**Power State: Disabled pada Host**  
Status ini normal untuk setup lab di VirtualBox dan bukan merupakan error. Power State mengacu pada fitur Out-of-band management (IPMI/iDRAC) yang hanya relevan untuk server fisik enterprise. Yang penting adalah status **Up** dan **Resource State: Enabled**.

**Memory**  
CloudStack, KVM, MySQL, dan System VMs (SSVM + Console Proxy) berjalan bersamaan dan mengonsumsi RAM yang cukup besar. Alokasikan minimal 4 GB RAM untuk VM agar sistem dapat berjalan stabil.

**IP Statis**  
Seluruh konfigurasi CloudStack (database, zone, host, storage) mengacu ke IP spesifik. Pastikan IP server selalu statis agar tidak berubah setelah reboot, dengan menggunakan `dhcp4: false` di konfigurasi Netplan.

---

## Referensi

- [Apache CloudStack Documentation](https://docs.cloudstack.apache.org)
- [ShapeBlue CloudStack Packages](http://packages.shapeblue.com)
- [Cloud Computing Group 20 2025](https://github.com/PhoebeIvana/cloud20-uas)
