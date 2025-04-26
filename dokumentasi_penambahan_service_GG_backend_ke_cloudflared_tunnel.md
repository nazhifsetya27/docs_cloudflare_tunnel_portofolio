# Dokumentasi Penambahan Service Gadget Gallery Backend ke Cloudflare Tunnel

## 🎯 Tujuan
Dokumentasi ini menjelaskan langkah-langkah untuk menambahkan service backend aplikasi Gadget Gallery (yang berjalan di port 4000 pada host) ke dalam konfigurasi Cloudflare Tunnel yang sudah ada, menggunakan service systemd terpisah untuk pengelolaan yang independen.

## 🔩 Konteks & Prasyarat
- VPS sudah terkonfigurasi dengan `cloudflared`.
- Sudah ada tunnel lain yang berjalan menggunakan file konfigurasi dan service systemd terpisah (misalnya untuk SSH, Frontend, Portofolio).
- Service backend Gadget Gallery sudah berjalan di dalam container Docker dan diekspos ke port **4000** pada host VPS.
- File konfigurasi tunnel dan credentials (`.json`) disimpan di `/root/.cloudflared/`.
- File service systemd disimpan di `/etc/systemd/system/`.

## ✅ Langkah-langkah Penambahan Service Backend

### 1. Verifikasi/Buat File Konfigurasi Tunnel (`gadget-gallery-api.yml`)
Pastikan file konfigurasi khusus untuk API backend sudah ada dan benar di `/root/.cloudflared/`. Jika belum ada, buat file ini.

**Contoh Isi `/root/.cloudflared/gadget-gallery-api.yml`:**
```yaml
# Nama atau ID Tunnel yang akan digunakan untuk API Backend
# Pastikan ini sesuai dengan tunnel yang terdaftar di Cloudflare Dashboard
# Contoh menggunakan nama tunnel:
tunnel: gadget-gallery-api
# atau menggunakan ID Tunnel:
# tunnel: 8ca7d8b5-7411-46c9-ba8c-72bac9ad2592

# Path ke file credentials JSON yang sesuai dengan Tunnel ID/Nama di atas
credentials-file: /root/.cloudflared/8ca7d8b5-7411-46c9-ba8c-72bac9ad2592.json

# Aturan Ingress untuk merutekan traffic
ingress:
  # Hostname publik yang akan digunakan untuk mengakses backend API
  - hostname: gadget-gallery-api.nazhifsetya.site # Pastikan nama domain ini benar
    # Arahkan traffic ke service backend yang berjalan di localhost port 4000 (port host)
    service: http://localhost:4000
  # Aturan fallback jika tidak ada hostname yang cocok
  - service: http_status:404
Use code with caution.
Markdown
Penting: Sesuaikan tunnel:, credentials-file:, dan hostname: dengan konfigurasi Anda. Pastikan service: menunjuk ke http://localhost:4000.

2. Buat File Service Systemd (cloudflared-be.service)
Buat file service systemd baru untuk mengelola tunnel backend ini secara terpisah. Cara termudah adalah menyalin salah satu file service cloudflared yang sudah ada dan memodifikasinya.

# Salin service yang ada (misal: cloudflared-fe.service) sebagai template
sudo cp /etc/systemd/system/cloudflared-fe.service /etc/systemd/system/cloudflared-be.service

# Edit file service baru
sudo nano /etc/systemd/system/cloudflared-be.service
Use code with caution.
Bash
Ubah isi file /etc/systemd/system/cloudflared-be.service menjadi seperti berikut, perhatikan perubahan pada Description dan path file konfigurasi di ExecStart:

[Unit]
# Berikan deskripsi yang jelas
Description=Cloudflare Tunnel for Gadget Gallery Backend API
After=network-online.target
Wants=network-online.target

[Service]
TimeoutStartSec=0
Type=notify
# Pastikan ExecStart menunjuk ke file konfigurasi YAML yang benar
ExecStart=/usr/bin/cloudflared --config /root/.cloudflared/gadget-gallery-api.yml tunnel run
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
Use code with caution.
Ini
Simpan dan tutup editor.

3. Reload Daemon Systemd
Beri tahu systemd tentang adanya file service baru. Perintah ini tidak me-restart service lain yang sedang berjalan.

sudo systemctl daemon-reload
Use code with caution.
Bash
4. Enable Service Baru
Aktifkan service agar otomatis berjalan saat VPS booting.

sudo systemctl enable cloudflared-be.service
Use code with caution.
Bash
5. Start Service Baru
Jalankan service tunnel backend sekarang juga.

sudo systemctl start cloudflared-be.service
Use code with caution.
Bash
6. Verifikasi
Pastikan tunnel backend berjalan dengan benar:

# Cek status service systemd
sudo systemctl status cloudflared-be.service
# Output harus menunjukkan 'active (running)'

# Cek koneksi tunnel menggunakan nama atau ID tunnel
cloudflared tunnel info gadget-gallery-api
# atau
cloudflared tunnel info 8ca7d8b5-7411-46c9-ba8c-72bac9ad2592
# Output harus menunjukkan koneksi aktif (CONNECTIONS > 0)

# (Opsional) Lihat log service secara real-time
sudo journalctl -u cloudflared-be.service -f

# Coba akses hostname backend dari browser atau curl
# curl https://gadget-gallery-api.nazhifsetya.site/
Use code with caution.
Bash
🎉 Selesai!
Service backend Gadget Gallery sekarang dapat diakses melalui https://gadget-gallery-api.nazhifsetya.site via Cloudflare Tunnel, dan dapat dikelola (start/stop/restart) secara independen menggunakan sudo systemctl [start|stop|restart] cloudflared-be.service.
