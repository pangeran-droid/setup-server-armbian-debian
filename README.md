# Setup Server Armbian Debian

Catatan langkah-langkah instalasi & konfigurasi setelah Armbian Debian berhasil masuk (login pertama).

## 1. Update Sistem

```bash
apt-get update
```

## 2. Web Server (Apache)

```bash
apt-get install apache2

systemctl start apache2
systemctl status apache2
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

File web root ada di `/var/www/html`.

## 3. Database (MariaDB) + phpMyAdmin

```bash
apt-get install mariadb-server
mysql_secure_installation

apt-get install phpmyadmin
```

## 4. PHP

```bash
apt-get install libapache2-mod-php8.3
```

## 5. Composer

```bash
apt-get install composer
composer --version
```

## 6. Git

```bash
apt-get install git
git config --global user.name "spasi"
git config --global user.email "spasi@gmail.com"
git config --list
```

SSH key untuk Git:

```bash
ssh-keygen -t ed25519 -C "spasi@gmail.com"
cat ~/.ssh/id_ed25519.pub
```

## 7. Docker

Instalasi via repo resmi Docker:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# atau manual via apt repo Docker
sudo apt-get update && sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo apt-get install docker-ce docker-ce-cli docker-buildx-plugin docker-compose-plugin
```

## 8. Podman (alternatif Docker, rootless)

```bash
sudo apt-get -y install podman
apt-get install cockpit-podman
```

Cek proses/image podman:

```bash
podman ps -a
podman info --format '{{.Store.GraphRoot}}'
podman --version
```

Menjalankan container rootless (contoh untuk user non-root):

```bash
apt-get install uidmap
which newuidmap

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

# install sebagai service (token didapat dari dashboard Cloudflare Zero Trust)
sudo cloudflared service install <TOKEN_TUNNEL>
```

## 10. File Browser

```bash
curl -fsSL https://raw.githubusercontent.com/filebrowser/get/master/get.sh | bash
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
sudo systemctl restart cockpit
```

Cek status instalasi:

```bash
dpkg -l | grep -E 'cockpit|cockpit-bridge'
apt policy cockpit cockpit-system cockpit-bridge
ls -l /usr/lib/cockpit/
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
apt-get install cockpit-sensors

# atau versi manual dari GitHub
wget https://github.com/ocristopfer/cockpit-sensors/releases/latest/download/cockpit-sensors.tar.xz
tar -xf cockpit-sensors.tar.xz cockpit-sensors/dist
mv cockpit-sensors/dist /usr/share/cockpit/sensors
rm -r cockpit-sensors cockpit-sensors.tar.xz
```

## 12. Node.js / npm

```bash
apt-get install npm
```

## 13. Tools Tambahan

```bash
apt-get install fastfetch   # info sistem
apt-get install cron        # scheduler
apt-get install at          # jadwal one-time task
```

Contoh jadwal reboot otomatis pakai `at`:

```bash
echo "/sbin/reboot" | sudo at 23:50 Sun
atq   # cek antrian job
```

## 14. Konfigurasi Jaringan

### Cek status interface

```bash
hostname -I
ip -4 addr
ip route
ip link
ip link set eth0 up
```

### Set IP manual (sementara, hilang setelah reboot)

```bash
ip addr add 192.168.18.96/24 dev eth0
ip route add default via 192.168.18.1
ping -c 3 192.168.18.1
```

### DNS

```bash
cat /etc/resolv.conf
systemctl status systemd-resolved --no-pager
resolvectl status
resolvectl dns eth0 192.168.18.1 8.8.8.8
resolvectl domain eth0 '~.'
resolvectl query ports.ubuntu.com
ping -c 3 ports.ubuntu.com
```

### Netplan (agar konfigurasi permanen & otomatis saat boot)

```bash
apt-get install netplan.io
which netplan

nano /etc/netplan/00-default-use-network-manager.yaml
cat /etc/netplan/00-default-use-network-manager.yaml

netplan generate
netplan try
```

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

Jika base OS sebenarnya Ubuntu (`noble`) tapi baris backports salah menunjuk ke repo Debian, nonaktifkan baris lama lalu tambahkan repo backports yang benar:

```bash
grep -Rni "deb.debian.org.*noble" /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null
sed -i 's|^deb http://deb.debian.org/debian noble-backports|# deb http://deb.debian.org/debian noble-backports|' /etc/apt/sources.list.d/backports.list
apt-get update
apt-get install -t noble-backports cockpit cockpit-system cockpit-bridge
systemctl restart cockpit
```

---

**Catatan umum:** selalu jalankan `reboot` setelah perubahan besar (instalasi kernel module, PAM, jaringan) untuk memastikan service jalan normal.