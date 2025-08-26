# Git Workflow - Windows 10 ke Ubuntu 24 Server

Panduan lengkap untuk setup workflow git dari komputer Windows 10 ke server Ubuntu 24. Dengan setup ini, kamu bisa coding dengan nyaman di Windows sambil otomatis deploy ke server Ubuntu.

## Yang Akan Kamu Dapatkan

1. **Development yang nyaman** - Coding di Windows 10 dengan tools favorit
2. **Version control yang rapi** - Semua kode tersimpan aman di GitHub
3. **Auto deployment** - Push code langsung deploy ke server Ubuntu
4. **Workflow sederhana** - Cuma butuh 2 branch: `main` dan `production`

## Yang Kamu Butuhkan

- Komputer Windows 10 (tempat coding)
- Server Ubuntu 24 (tempat website live)
- Akun GitHub
- Akses SSH ke server Ubuntu

---

## Setup di Windows (Komputer Development)

### 1. Install Git di Windows

Download file installer Git dari [git-scm.com](https://git-scm.com/download/win), pilih versi 64-bit untuk Windows. Jalankan file .exe dan ikuti wizard instalasi dengan setting default.

### 2. Konfigurasi Git

Buka Git Bash dan setup identitas kamu:

```bash
git config --global user.name "Nama Kamu"
git config --global user.email "email@domain.com"
git config --global init.defaultBranch main
git config --global core.autocrlf true
```

Cek konfigurasi:
```bash
git config --list
```

### 3. Buat SSH Key untuk GitHub

```bash
# Di Git Bash
ssh-keygen -t rsa -b 4096 -C "email@domain.com"
# Tekan Enter untuk lokasi default: C:\Users\NamaKamu\.ssh\id_rsa

# Start SSH agent
eval $(ssh-agent -s)

# Tambahkan SSH key
ssh-add ~/.ssh/id_rsa

# Tampilkan public key untuk di-copy
cat ~/.ssh/id_rsa.pub
```

Copy public key yang muncul, lalu tambahkan ke GitHub:
- Buka GitHub → Settings → SSH and GPG keys → New SSH key
- Paste public key kamu

### 4. Buat SSH Key untuk Server

```bash
# Buat SSH key khusus untuk deploy ke server
ssh-keygen -t rsa -b 4096 -C "deploy@server" -f ~/.ssh/deploy_key

# Tampilkan public key untuk server
cat ~/.ssh/deploy_key.pub
```

### 5. Buat Repository di GitHub

1. Buka GitHub → New Repository
2. Nama repository: `test-deploy`
3. Set sebagai **Private** (untuk keamanan)
4. Centang "Add a README file"
5. Pilih .gitignore sesuai project (Node, PHP, dsb)
6. Klik "Create repository"

### 6. Clone dan Setup Branch

```bash
# Clone repository ke komputer kamu
git clone git@github.com:username/test-deploy.git
cd test-deploy

# Buat branch production
git checkout -b production
git push -u origin production

# Kembali ke main branch
git checkout main
```

---

## Setup di Ubuntu Server

### 7. Test Koneksi SSH dari Windows

```bash
# Test koneksi SSH dari Git Bash
ssh username@192.168.1.100

# Copy SSH key ke server
ssh-copy-id -i ~/.ssh/deploy_key username@192.168.1.100

# Test koneksi dengan deploy key
ssh -i ~/.ssh/deploy_key username@192.168.1.100
```

### 8. Install Git di Ubuntu Server

Login ke server via SSH, lalu install git:

```bash
# Update system
sudo apt update

# Install git
sudo apt install git

# Konfigurasi git (opsional)
git config --global user.name "Server Deploy"
git config --global user.email "deploy@server.com"
```

### 9. Setup Repository di Server

```bash
# Buat folder untuk bare repository
cd /var/www/
sudo mkdir test-deploy.git
sudo chown $USER:$USER test-deploy.git
cd test-deploy.git
git init --bare

# Buat folder untuk file website live
cd /var/www/
sudo mkdir test-deploy-live
sudo chown $USER:$USER test-deploy-live

# Set permission untuk web server
sudo chown -R www-data:www-data test-deploy-live
sudo chmod -R 755 test-deploy-live
```

### 10. Install Docker dan Docker Compose

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Tambahkan user ke docker group
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Logout dan login lagi untuk apply group changes
exit
# Login SSH lagi
```

### 11. Buat Auto Deploy Hook dengan Docker

```bash
# Buat file post-receive hook
nano /var/www/test-deploy.git/hooks/post-receive
```

Isi file dengan script berikut:

```bash
#!/bin/bash
echo "🚀 Deploying dari Windows ke Ubuntu server..."

# Checkout files dari production branch
cd /var/www/test-deploy-live
git --git-dir=/var/www/test-deploy.git --work-tree=/var/www/test-deploy-live checkout -f production

# Build dan jalankan dengan Docker Compose
cd /var/www/test-deploy-live
docker-compose down
docker-compose up -d --build

echo "✅ Deploy berhasil! Website running di Docker container"
```

Buat file executable:
```bash
chmod +x /var/www/test-deploy.git/hooks/post-receive
```

---

## 🔄 Kembali ke Windows - Setup Remote

### 11. Tambahkan Server sebagai Remote

Di folder project kamu (Git Bash):

```bash
# Tambahkan server sebagai remote "production"
git remote add production username@192.168.1.100:/var/www/test-deploy.git

# Cek remote yang tersedia
git remote -v
```

Hasilnya akan seperti ini:
```
origin      git@github.com:username/test-deploy.git (fetch)
origin      git@github.com:username/test-deploy.git (push)
production  username@192.168.1.100:/var/www/test-deploy.git (fetch)
production  username@192.168.1.100:/var/www/test-deploy.git (push)
```

### 12. Setup SSH Config (Opsional)

Buat file `C:\Users\Username\.ssh\config` untuk memudahkan koneksi:

```
# GitHub
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa

# Ubuntu Server
Host myserver
    HostName 192.168.1.100
    User username
    IdentityFile ~/.ssh/deploy_key
    Port 22
```

Sekarang bisa SSH dengan perintah singkat: `ssh myserver`

---

---

## 🐳 Setup Docker di Windows (Project Files)

### 12. Buat File Docker Configuration

Buat file-file ini di root project kamu (Windows):

**docker-compose.yml**
```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "80:80"
    volumes:
      - ./src:/usr/share/nginx/html
    container_name: test-deploy-web
    restart: unless-stopped
```

**Dockerfile**
```dockerfile
FROM nginx:alpine

# Copy nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

# Copy website files
COPY src/ /usr/share/nginx/html/

# Expose port 80
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**nginx.conf**
```nginx
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    server {
        listen 80;
        server_name localhost;
        
        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
            try_files $uri $uri/ /index.html;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
}
```

### 13. Buat Struktur Project

```bash
# Di Git Bash Windows, dalam folder project
mkdir src
echo '<h1>Hello from Windows with Docker!</h1>' > src/index.html
```

---

## Cara Menggunakan Workflow Ini

### Development Sehari-hari

```bash
# 1. Pastikan di branch main
git checkout main
git pull origin main

# 2. Edit file di folder src/
# Gunakan code editor (VS Code, Sublime, dsb)
# Semua file HTML, CSS, JS taruh di folder src/

# 3. Commit perubahan
git add .
git commit -m "feat: tambah fitur baru"

# 4. Push ke GitHub
git push origin main
```

### Deploy ke Production

```bash
# 1. Pindah ke branch production
git checkout production

# 2. Merge perubahan dari main
git merge main

# 3. Push ke GitHub (backup)
git push origin production

# 4. Deploy langsung ke server Ubuntu
git push production production
```

Setelah perintah terakhir, file kamu akan otomatis ter-deploy ke server Ubuntu!

### Cek Hasil di Server

```bash
# SSH ke server untuk cek hasil
ssh username@192.168.1.100

# Cek container yang berjalan
docker ps

# Cek logs deployment
docker-compose logs web

# Test website
curl localhost
# atau buka browser: http://192.168.1.100

# Keluar dari server
exit
```

---

## Tips dan Trik

### Rekomendasi Code Editor di Windows
- **VS Code** - Paling populer dengan Git integration
- **Sublime Text** - Ringan dan cepat
- **PhpStorm** - Bagus untuk project PHP

### Terminal yang Direkomendasikan
- **SSH operations**: Git Bash
- **Git commands**: Git Bash atau PowerShell
- **File operations**: PowerShell atau File Explorer

### Struktur Project yang Disarankan
```
test-deploy/
├── README.md
├── .gitignore
├── docker-compose.yml
├── Dockerfile
├── nginx.conf
└── src/
    ├── index.html
    ├── css/
    │   └── style.css
    ├── js/
    │   └── script.js
    └── assets/
        └── images/
```

---

## Troubleshooting

### SSH Permission Denied
```bash
# Pastikan SSH agent berjalan
eval $(ssh-agent -s)
ssh-add ~/.ssh/deploy_key

# Test koneksi
ssh -i ~/.ssh/deploy_key username@192.168.1.100
```

### Git Push Rejected
```bash
# Pull perubahan terbaru dulu
git pull origin production

# Lalu push lagi
git push production production
```

### Docker Container Error
```bash
# SSH ke server dan cek logs
docker-compose logs web

# Restart container
docker-compose restart web

# Rebuild container
docker-compose down
docker-compose up -d --build
```

### Port Already in Use
```bash
# Cek port yang dipakai
sudo netstat -tulpn | grep :80

# Stop service yang pakai port 80
sudo systemctl stop apache2  # jika ada
sudo systemctl stop nginx    # jika ada

# Atau ganti port di docker-compose.yml
# ports: "8080:80"
```

### File Permission Error di Server
```bash
# SSH ke server dan fix permission
sudo chown -R $USER:$USER /var/www/test-deploy-live
```

---

## Workflow Summary

1. **Setup Docker** - Buat docker-compose.yml, Dockerfile, dan nginx.conf
2. **Coding** - Edit file di folder `src/` dengan nyaman di Windows
3. **Commit** - Simpan perubahan dengan `git commit`
4. **Push to main** - Upload ke GitHub dengan `git push origin main`
5. **Merge to production** - Gabungkan ke branch production
6. **Deploy** - Push ke server dengan `git push production production`
7. **Auto Build** - Docker otomatis build dan jalankan Nginx container
8. **Live!** - Website langsung live di http://192.168.1.100

Dengan Docker, website akan:
- Berjalan di Nginx Alpine (ringan dan cepat)
- Konsisten di semua environment
- Mudah di-scale dan maintain
- Auto restart jika ada masalah

Sekarang bisa fokus coding tanpa ribet urusan server setup!
