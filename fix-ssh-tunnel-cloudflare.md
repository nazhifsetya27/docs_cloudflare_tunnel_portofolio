# 🚀 Dokumentasi Perbaikan & Setup Tunnel SSH VPS via Cloudflare

## 🌐 Tujuan
Membuat koneksi SSH ke VPS (tanpa IP publik) dengan aman melalui Cloudflare Tunnel + Zero Trust Access.

---

## 🫠 Masalah yang Ditemukan
- `ssh.nazhifsetya.biz.id` terhubung ke tunnel yang salah (`ssh-vps`) → menyebabkan error `websocket: bad handshake`
- Tunnel seharusnya diarahkan ke `ssh-tunnel`, namun tidak bisa di-overwrite secara CLI
- Solusi: menghapus tunnel lama & mengaitkan ulang domain ke tunnel yang benar

---

## ✅ Langkah Perbaikan

### 1. Verifikasi Tunnel yang Aktif (di VPS)
```bash
cloudflared tunnel list
```
Cari tunnel dengan nama `ssh-tunnel` dan catat ID-nya (misal: `2f6ebd31-...`).

### 2. Update Config File `ssh-config.yml`
```bash
nano /root/.cloudflared/ssh-config.yml
```
Isi:
```yaml
tunnel: ssh-tunnel
credentials-file: /root/.cloudflared/2f6ebd31-9797-4d1c-8439-90a0efb3e51b.json

ingress:
  - hostname: ssh.nazhifsetya.biz.id
    service: ssh://localhost:2222
  - service: http_status:404
```

### 3. Hapus Tunnel yang Salah (Opsional, jika masih nyangkut)
```bash
cloudflared tunnel delete ssh-vps
```

### 4. Assign Hostname ke Tunnel yang Benar
```bash
cloudflared tunnel route dns ssh-tunnel ssh.nazhifsetya.biz.id
```
Jika gagal overwrite, hapus record CNAME `ssh` di dashboard Cloudflare lalu ulangi perintah di atas.

### 5. Jalankan Tunnel Manual untuk Tes
```bash
cloudflared tunnel --config /root/.cloudflared/ssh-config.yml run
```

### 6. Dari Client (MacBook), jalankan:
```bash
cloudflared access login ssh.nazhifsetya.biz.id
ssh ssh-vps
```
Dengan konfigurasi `~/.ssh/config`:
```ssh
Host ssh-vps
  HostName ssh.nazhifsetya.biz.id
  ProxyCommand /opt/homebrew/bin/cloudflared access ssh --hostname %h
  User nazhif
```

---

## ⚙️ Setup Autostart via Systemd (di VPS)

### 1. Buat Service File Baru
```bash
nano /etc/systemd/system/cloudflared-ssh.service
```
Isi:
```ini
[Unit]
Description=Cloudflare SSH Tunnel
After=network.target

[Service]
TimeoutStartSec=0
Restart=always
ExecStart=/usr/bin/cloudflared --config /root/.cloudflared/ssh-config.yml tunnel run
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### 2. Enable dan Start Service
```bash
systemctl daemon-reexec
systemctl daemon-reload
systemctl enable cloudflared-ssh
systemctl start cloudflared-ssh
```

### 3. Cek Status
```bash
systemctl status cloudflared-ssh
```

Jika berhasil, tunnel akan otomatis hidup saat VPS menyala.

---

## 🎉 Selesai!
Kamu sekarang bisa SSH ke VPS dengan:
```bash
ssh ssh-vps
```
Tanpa IP publik, dengan keamanan dari Cloudflare Access ✔️


