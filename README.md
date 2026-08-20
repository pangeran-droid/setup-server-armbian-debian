# Setup Server Armbian Debian

Catatan langkah-langkah instalasi & konfigurasi setelah Armbian Debian berhasil masuk (login pertama). Setiap perintah dilengkapi contoh output yang biasa muncul, supaya lebih mudah tahu apakah langkahnya berhasil.

## 1. Update Sistem

```bash
apt-get update
```

Contoh output:
```
Hit:1 http://deb.debian.org/debian bookworm InRelease
Get:2 http://deb.debian.org/debian bookworm-updates InRelease [52.1 kB]
Reading package lists... Done
```

## 2. Web Server (Apache)

```bash
apt-get install apache2

systemctl start apache2
systemctl status apache2
```

Contoh output `systemctl status apache2` (jika berhasil):
```
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled)
     Active: active (running) since Thu 2026-08-20 09:12:01 WIB; 5s ago
    Process: 1423 ExecStart=/usr/sbin/apachectl start (code=exited, status=0/SUCCESS)
```

Cek log jika gagal:
```bash
journalctl -u apache2 -n 50
```

Jika Apache gagal start karena masalah PAM:

```bash
apt-get install --reinstall libpam-modules
apt-get install libpam-systemd
reboot
```

Jika folder log error (`/var/log/apache2`) bermasalah:

```bash
cd /var/log
mkdir apache2
chmod -R 777 apache2
systemctl start apache2
```

Konfigurasi virtual host:

```bash
nano /etc/apache2/sites-available/000-default.conf
```

Contoh isi file sederhana:
```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Tes akses via browser atau curl:
```bash
curl -I http://localhost
```
Output yang diharapkan:
```
HTTP/1.1 200 OK
Server: Apache/2.4.62 (Debian)
```

File web root ada di `/var/www/html`.

## 3. Database (MariaDB) + phpMyAdmin

```bash
apt-get install mariadb-server
mysql_secure_installation
```

Contoh interaksi `mysql_secure_installation`:
```
Enter current password for root (enter for none): [kosongkan lalu Enter]
Set root password? [Y/n] Y
New password: ********
Re-enter new password: ********
Remove anonymous users? [Y/n] Y
Disallow root login remotely? [Y/n] Y
Remove test database and access to it? [Y/n] Y
Reload privilege tables now? [Y/n] Y
```

```bash
apt-get install phpmyadmin
```
Saat instalasi akan muncul dialog: pilih web server `apache2`, lalu pilih `Yes` untuk `dbconfig-common` mengatur database phpMyAdmin otomatis.

Cek phpMyAdmin bisa diakses:
```bash
curl -I http://localhost/phpmyadmin
```

## 4. PHP

```bash
apt-get install libapache2-mod-php8.3
```

Cek versi PHP aktif di Apache:
```bash
php -v
```
Contoh output:
```
PHP 8.3.6 (cli) (built: Apr 10 2026 10:22:31) (NTS)
```

## 5. Composer

```bash
apt-get install composer
composer --version
```
Contoh output:
```
Composer version 2.7.1 2026-03-14 15:03:16
```

## 6. Git

```bash
apt-get install git
git config --global user.name "spasi"
git config --global user.email "spasi@gmail.com"
git config --list
```
Contoh output `git config --list`:
```
user.name=spasi
user.email=spasi@gmail.com
```

SSH key untuk Git:

```bash
ssh-keygen -t ed25519 -C "spasi@gmail.com"
```
Contoh output:
```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/root/.ssh/id_ed25519): [Enter]
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
```

```bash
cat ~/.ssh/id_ed25519.pub
```
Contoh output (tambahkan ke GitHub/GitLab > SSH Keys):
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... spasi@gmail.com
```

## 7. Docker

Instalasi via repo resmi Docker:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Atau manual via apt repo Docker:
```bash
sudo apt-get update && sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo apt-get install docker-ce docker-ce-cli docker-buildx-plugin docker-compose-plugin
```

Verifikasi instalasi:
```bash
docker --version
docker run hello-world
```
Contoh output:
```
Docker version 27.3.1, build ce12230
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

## 8. Podman (alternatif Docker, rootless)

```bash
sudo apt-get -y install podman
apt-get install cockpit-podman
```

Cek proses/image podman:

```bash
podman ps -a
```
Contoh output (belum ada container):
```
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
```

```bash
podman info --format '{{.Store.GraphRoot}}'
```
Contoh output:
```
/var/lib/containers/storage
```

```bash
podman --version
```
Contoh output:
```
podman version 4.9.3
```

Menjalankan container rootless (contoh untuk user non-root):

```bash
apt-get install uidmap
which newuidmap
```
Contoh output:
```
/usr/bin/newuidmap
```

```bash
su - <NAMA_USER> -c 'podman ps -a'
su - <NAMA_USER> -c 'podman images'
```

Contoh menjalankan container (9router):

```bash
mkdir -p ~/9router-data
podman run -d --name 9router --restart unless-stopped \
  -p 20128:20128 \
  -v ~/9router-data:/root/.9router \
  docker.io/library/node:20-alpine \
  sh -c "npm install -g 9router && 9router"
```
Contoh output (ID container hasil `podman run -d`):
```
a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef1234
```

Membersihkan container/image:

```bash
podman rm -f 9router
podman rmi docker.io/library/node:20-alpine
rm -rf ~/9router-data
```

## 9. Cloudflare Tunnel (cloudflared)

```bash
# tambahkan GPG key Cloudflare
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

# tambahkan repo Cloudflare
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | \
  sudo tee /etc/apt/sources.list.d/cloudflared.list

# install
sudo apt-get update && sudo apt-get install cloudflared
```

```bash
cloudflared --version
```
Contoh output:
```
cloudflared version 2026.7.0 (built 2026-07-15)
```

```bash
# install sebagai service (token didapat dari dashboard Cloudflare Zero Trust)
sudo cloudflared service install <TOKEN_TUNNEL>
```
Contoh output jika berhasil:
```
Installing an Argo Tunnel Service
Argo Tunnel service installed successfully
```

## 10. File Browser

```bash
curl -fsSL https://raw.githubusercontent.com/filebrowser/get/master/get.sh | bash
```
Contoh output akhir instalasi:
```
File Browser installed to /usr/local/bin/filebrowser
Run 'filebrowser -h' to see the help
```

## 11. Cockpit (Web Admin Panel)

Aktifkan repo backports sesuai versi OS:

```bash
. /etc/os-release
echo "deb http://deb.debian.org/debian ${VERSION_CODENAME}-backports main" | \
  sudo tee /etc/apt/sources.list.d/backports.list

sudo apt update
sudo apt install -t ${VERSION_CODENAME}-backports cockpit
```

Kelola user yang dilarang login ke Cockpit:

```bash
sudo nano /etc/cockpit/disallowed-users
```
Contoh isi file (satu username per baris):
```
root
```

```bash
sudo systemctl restart cockpit
```

Cek status instalasi:

```bash
dpkg -l | grep -E 'cockpit|cockpit-bridge'
```
Contoh output:
```
ii  cockpit               309-1~bpo12+1   arm64  Web Console for Linux servers
ii  cockpit-bridge        309-1~bpo12+1   arm64  Cockpit privileged bridge
```

Akses Cockpit via browser:
```
https://<IP-SERVER>:9090
```

### Plugin Cockpit

**Navigator (file manager):**

```bash
wget https://github.com/45Drives/cockpit-navigator/releases/download/v0.5.10/cockpit-navigator_0.5.10-1focal_all.deb
apt install ./cockpit-navigator_0.5.10-1focal_all.deb
systemctl restart cockpit
```

**Podman:**

```bash
apt-get install -t ${VERSION_CODENAME}-backports cockpit-podman
systemctl restart cockpit
```

**Sensors (suhu/hardware):**

```bash
apt-get install lm-sensors -y
sensors-detect
```
Contoh pertanyaan interaktif `sensors-detect` (biasanya jawab `YES`/Enter untuk semua):
```
Do you want to generate /etc/sysconfig/lm_sensors? YES
```

```bash
apt-get install cockpit-sensors
```

Setelah restart Cockpit, buka menu **Sensors** di sidebar untuk melihat suhu CPU/board.

## 12. Node.js / npm

```bash
apt-get install npm
```
Cek versi:
```bash
node -v
npm -v
```
Contoh output:
```
v18.19.1
9.2.0
```

## 13. Tools Tambahan

```bash
apt-get install fastfetch   # info sistem
apt-get install cron        # scheduler
apt-get install at          # jadwal one-time task
```

Contoh output `fastfetch`:
```
root@armbian
------------
OS: Debian GNU/Linux 12 (bookworm) aarch64
Kernel: 6.1.75-current-rockchip64
Uptime: 2 days, 3 hours
CPU: Rockchip RK3588
Memory: 1.2GiB / 8.0GiB
```

Contoh jadwal reboot otomatis pakai `at`:

```bash
echo "/sbin/reboot" | sudo at 23:50 Sun
```
Contoh output:
```
warning: commands will be executed using /bin/sh
job 3 at Sun Aug 23 23:50:00 2026
```

```bash
atq   # cek antrian job
```
Contoh output:
```
3    Sun Aug 23 23:50:00 2026 a root
```

## 14. Konfigurasi Jaringan

### Cek status interface

```bash
hostname -I
```
Contoh output:
```
192.168.18.96
```

```bash
ip -4 addr
```
Contoh output:
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 192.168.18.96/24 brd 192.168.18.255 scope global eth0
```

```bash
ip route
```
Contoh output:
```
default via 192.168.18.1 dev eth0
192.168.18.0/24 dev eth0 proto kernel scope link src 192.168.18.96
```

### Set IP manual (sementara, hilang setelah reboot)

```bash
ip addr add 192.168.18.96/24 dev eth0
ip route add default via 192.168.18.1
ping -c 3 192.168.18.1
```
Contoh output ping berhasil:
```
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
```

### DNS

```bash
cat /etc/resolv.conf
```
Contoh output:
```
nameserver 192.168.18.1
nameserver 8.8.8.8
```

```bash
resolvectl status
```
Contoh output (ringkas):
```
Global
       Protocols: +LLMNR +mDNS -DNSOverTLS DNSSEC=no/unsupported

Link 2 (eth0)
    Current Scopes: DNS
DefaultRoute setting: yes
     Current DNS Server: 192.168.18.1
        DNS Servers: 192.168.18.1 8.8.8.8
```

```bash
resolvectl dns eth0 192.168.18.1 8.8.8.8
resolvectl domain eth0 '~.'
resolvectl query ports.ubuntu.com
```
Contoh output query berhasil:
```
ports.ubuntu.com: 91.189.91.83
```

### Netplan (agar konfigurasi permanen & otomatis saat boot)

```bash
apt-get install netplan.io
which netplan
```
Contoh output:
```
/usr/sbin/netplan
```

```bash
nano /etc/netplan/00-default-use-network-manager.yaml
```
Contoh isi file untuk IP statis:
```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.18.96/24]
      routes:
        - to: default
          via: 192.168.18.1
      nameservers:
        addresses: [192.168.18.1, 8.8.8.8]
```

```bash
netplan generate
netplan try
```
Contoh output `netplan try`:
```
Do you want to keep these settings?

Press ENTER before the timeout to accept the new configuration


Changes will revert in 120 seconds
```
Tekan Enter jika koneksi tetap jalan normal.

Verifikasi akhir:

```bash
ip -4 addr show eth0
ip route
resolvectl status
resolvectl query ports.ubuntu.com

ping -c 2 192.168.18.1
ping -c 2 8.8.8.8
ping -c 2 ports.ubuntu.com

reboot
```

> Catatan: jika sebelumnya sempat instal Samba lalu dihapus, bersihkan sisa konfigurasi PAM:
> ```bash
> sudo systemctl stop smbd nmbd
> sudo apt-get purge samba samba-common samba-common-bin smbclient -y
> sudo apt-get autoremove -y
> sudo pam-auth-update --force
> sudo rm -f /etc/pam.d/samba
> ```

## 15. Troubleshooting Repo `noble-backports` (Debian vs Ubuntu codename)

Jika base OS sebenarnya Ubuntu (`noble`) tapi baris backports salah menunjuk ke repo Debian, `apt-get update` akan gagal seperti ini:

```
Err:3 http://deb.debian.org/debian noble-backports InRelease
  404  Not Found
```

Perbaikan — nonaktifkan baris lama lalu tambahkan repo backports yang benar:

```bash
grep -Rni "deb.debian.org.*noble" /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null
sed -i 's|^deb http://deb.debian.org/debian noble-backports|# deb http://deb.debian.org/debian noble-backports|' /etc/apt/sources.list.d/backports.list
apt-get update
apt-get install -t noble-backports cockpit cockpit-system cockpit-bridge
systemctl restart cockpit
```

Setelah diperbaiki, `apt-get update` seharusnya menunjukkan:
```
Hit:3 http://archive.ubuntu.com/ubuntu noble-backports InRelease
Reading package lists... Done
```

---

**Catatan umum:** selalu jalankan `reboot` setelah perubahan besar (instalasi kernel module, PAM, jaringan) untuk memastikan service jalan normal.
