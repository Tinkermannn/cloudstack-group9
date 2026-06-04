# Troubleshooting: Apache CloudStack Setup on VirtualBox (Ubuntu Server 24.04)

Dokumen ini mencatat seluruh error yang ditemui selama proses setup Apache CloudStack dan solusi yang berhasil menyelesaikannya.

---

## 1. Koneksi Internet Putus Setelah netplan apply

Masalah: ping ke 8.8.8.8 menghasilkan "Network is unreachable" atau "Destination Host Unreachable".

Penyebab: Setelah bridge cloudbr0 dikonfigurasi, interface enp0s3 dilepas dari IP-nya dan diserahkan ke bridge. Koneksi terputus karena driver WiFi di VirtualBox tidak mendukung promiscuous mode yang dibutuhkan untuk Bridged Adapter.

Solusi: Ganti mode network adapter di VirtualBox dari Bridged Adapter ke NAT Network.

1. Matikan VM
2. VirtualBox → Settings → Network → Adapter 1
3. Ubah "Attached to" dari Bridged Adapter ke NAT Network
4. Buat NAT Network baru jika belum ada: File → Tools → Network Manager → NAT Networks → Create
5. Nyalakan VM kembali

---

## 2. apt update Menampilkan IGN dan 0%

Masalah: Semua repository menampilkan IGN dan persentase tidak bergerak dari 0%.

Penyebab: IGN (Ignored) berarti apt tidak bisa menjangkau repository karena tidak ada koneksi internet. Ini adalah efek lanjutan dari masalah bridge networking di nomor 1.

Solusi: Selesaikan masalah koneksi internet terlebih dahulu (lihat nomor 1), lalu jalankan ulang `apt update`.

---

## 3. libvirtd Tidak Listen di Port 16509

Masalah: `ss -tlnp | grep 16509` tidak menghasilkan output apapun.

Penyebab: Di Ubuntu 24.04, libvirtd dikelola sepenuhnya oleh systemd dengan mekanisme socket activation. Pendekatan lama menggunakan LIBVIRTD_ARGS="--listen" di /etc/default/libvirtd tidak bekerja karena systemd mengabaikan file tersebut. Selain itu, ditemukan duplikasi konfigurasi yang konflik di libvirtd.conf.

Solusi: Bersihkan duplikasi dan buat systemd override.

```bash
sed -i '/^listen_tls/d' /etc/libvirt/libvirtd.conf
sed -i '/^listen_tcp/d' /etc/libvirt/libvirtd.conf
sed -i '/^tcp_port/d' /etc/libvirt/libvirtd.conf
sed -i '/^auth_tcp/d' /etc/libvirt/libvirtd.conf
sed -i '/^mdns_adv/d' /etc/libvirt/libvirtd.conf

echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

mkdir -p /etc/systemd/system/libvirtd.service.d/
printf '[Service]\nExecStart=\nExecStart=/usr/sbin/libvirtd --listen\n' > /etc/systemd/system/libvirtd.service.d/override.conf

systemctl daemon-reload
systemctl restart libvirtd
ss -tlnp | grep 16509
```

---

## 4. cloudstack-setup-agent Gagal: /etc/network/interfaces Tidak Ditemukan

Masalah:
```
Configure Network ... [Failed]
Missing device configuration, Need to add your network configuration
into /etc/network/interfaces at first
```

Penyebab: Ubuntu 24.04 menggunakan Netplan sebagai sistem konfigurasi jaringan sehingga file /etc/network/interfaces tidak ada secara default. Script cloudstack-setup-agent masih mengasumsikan file ini ada.

Solusi: Buat file /etc/network/interfaces secara manual. Sesuaikan address, gateway, dan nama interface dengan kondisi server.

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

---

## 5. cloudstack-setup-agent Gagal: ifup not found

Masalah:
```
Configure Network ... [Failed]
Can't start network:cloudbr0 /bin/sh: 1: ifup: not found
```

Penyebab: Package ifupdown yang menyediakan perintah ifup dan ifdown tidak terinstall secara default di Ubuntu 24.04.

Solusi:
```bash
apt-get install ifupdown -y
```

---

## 6. cloudstack-setup-agent Gagal: Address Already Assigned

Masalah:
```
Configure Network ... [Failed]
Can't start network:cloudbr0 Error: ipv4: Address already assigned.
ifup: failed to bring up cloudbr0
```

Penyebab: cloudbr0 sudah memiliki IP yang diassign oleh Netplan. Ketika setup agent mencoba menjalankan ifup cloudbr0, gagal karena IP sudah ada.

Solusi: Lewati setup agent dan konfigurasi agent.properties secara manual.

```bash
UUID=$(uuidgen)
sed -i "s/^guid=$/guid=$UUID/" /etc/cloudstack/agent/agent.properties
nano /etc/cloudstack/agent/agent.properties
```

Pastikan baris berikut terisi dengan benar:
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

---

## 7. CloudStack Agent Terus Restart (crash loop)

Masalah: `systemctl status cloudstack-agent` menampilkan "activating (auto-restart)" dan agent terus restart setiap ~11 detik.

Penyebab: Field guid di agent.properties masih kosong. CloudStack Agent tidak dapat start tanpa GUID yang valid.

Solusi:
```bash
UUID=$(uuidgen)
sed -i "s/^guid=$/guid=$UUID/" /etc/cloudstack/agent/agent.properties
systemctl restart cloudstack-agent
```

---

## 8. Disk Penuh Setelah Instalasi

Masalah di log management server:
```
Image storage [1] has not enough capacity. Capacity: total=[12 GB], used=[12 GB]
Zone [1] is not ready to launch secondary storage VM.
```

Penyebab: CloudStack secara otomatis mendownload System VM Template (~1.65 GB) ke secondary storage saat pertama kali dijalankan. Dikombinasikan dengan instalasi Ubuntu, CloudStack, MySQL, dan KVM, total penggunaan disk bisa mencapai batas LVM default Ubuntu.

Solusi: Cek free space di LVM terlebih dahulu sebelum resize disk di VirtualBox.

```bash
vgdisplay | grep "Free"
```

Jika ada free space, extend langsung:
```bash
lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
df -h /
```

---

## 9. Add Host Gagal saat Setup Zone

Masalah:
```
Could not add host at [http://192.168.1.4] with zone [1], pod [1] and cluster [1]
due to: CloudStack Agent setup through command [cloudstack-setup-agent] failed.
```

Penyebab: cloudstack-setup-agent gagal karena beberapa dependency tidak terpenuhi atau konfigurasi agent belum lengkap.

Solusi: Pastikan semua kondisi berikut terpenuhi sebelum mencoba add host:

1. libvirtd listen di port 16509 — verifikasi dengan `ss -tlnp | grep 16509`
2. File /etc/network/interfaces sudah dibuat (lihat nomor 4)
3. Package ifupdown sudah terinstall (lihat nomor 5)
4. agent.properties sudah dikonfigurasi lengkap termasuk GUID (lihat nomor 7)
5. cloudstack-agent berstatus running — verifikasi dengan `systemctl status cloudstack-agent`

---

## 10. ISO Status Not Ready / Request Failed 530

Masalah:
```
Request failed. (530)
Unable to find image store to download template null
```

Penyebab: Secondary Storage VM (SSVM) belum berjalan atau belum siap. SSVM adalah system VM yang bertanggung jawab untuk mendownload ISO dan template ke secondary storage.

Solusi: Cek status SSVM di Infrastructure → System VMs. Tunggu hingga statusnya Running. SSVM membutuhkan beberapa menit untuk start setelah zone diaktifkan. Jika disk masih penuh, selesaikan masalah disk terlebih dahulu (lihat nomor 8) karena SSVM tidak akan start jika kapasitas storage tidak mencukupi.