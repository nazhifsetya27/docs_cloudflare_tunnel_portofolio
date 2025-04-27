# Dokumentasi Deploy Frontend Lunar via Docker & Cloudflare Tunnel

## 🎯 Tujuan
Dokumentasi ini menjelaskan langkah-langkah lengkap untuk mendeploy aplikasi frontend statis "Lunar" dari build lokal ke VPS, menjalankannya di dalam container Docker, dan membuatnya dapat diakses publik melalui domain `kak_biro_kesra_jabar.nazhifsetya.site` menggunakan Cloudflare Tunnel terpisah.

## 🔩 Prasyarat
-   Akses SSH ke VPS (menggunakan alias `Samsung-VPS-cloudflared` dari `~/.ssh/config`).
-   Docker sudah terinstall di VPS.
-   `cloudflared` sudah terinstall dan terkonfigurasi di VPS (di `/root/.cloudflared/`).
-   Folder `build` hasil build aplikasi Lunar sudah siap di direktori proyek lokal di MacBook.
-   Domain `nazhifsetya.site` dikelola oleh Cloudflare.

## ✅ Langkah-langkah Deployment

### 1. Persiapan di VPS & Transfer File

1.1. **Buat Direktori Proyek di VPS:**
    Login ke VPS dan buat direktori untuk Lunar.
    ```bash
    # Login via alias SSH
    ssh Samsung-VPS-cloudflared

    # Buat direktori dan masuk ke dalamnya
    mkdir -p /home/nazhif/lunar
    cd /home/nazhif/lunar

    # Keluar sementara
    exit
    ```

1.2. **Salin Folder `build` dari Lokal ke VPS:**
    Dari terminal **MacBook lokal**, gunakan `scp` dengan alias SSH untuk mentransfer folder `build`. Ganti `~/path/to/lunar/build` dengan path sebenarnya.
    ```bash
    # Pastikan Anda berada di direktori yang berisi folder 'build' atau gunakan path absolut
    # Contoh menggunakan path relatif dari direktori proyek lunar lokal:
    scp -r build Samsung-VPS-cloudflared:/home/nazhif/lunar/
    ```
    Masukkan password jika diminta oleh `scp`.

### 2. Konfigurasi & Build Docker Image

2.1. **Buat Dockerfile di VPS:**
    Kembali login ke VPS dan masuk ke direktori Lunar. Buat file `Dockerfile`.
    ```bash
    ssh Samsung-VPS-cloudflared
    cd /home/nazhif/lunar
    nano Dockerfile
    ```

2.2. **Isi `Dockerfile`:**
    Tempelkan konfigurasi berikut untuk menyajikan file statis menggunakan `serve` pada port **3000** internal.
    ```Dockerfile
    # Gunakan Node.js sebagai base image
    FROM node:18-alpine

    # Set direktori kerja di dalam container
    WORKDIR /app

    # Salin HANYA isi folder build ke direktori kerja container
    COPY build/ .

    # Install 'serve' secara global untuk menyajikan file statis
    RUN npm install -g serve

    # Ekspos port 3000 (port internal container)
    EXPOSE 3000

    # Jalankan serve pada port 3000, sajikan isi direktori saat ini
    CMD ["serve", "-s", ".", "-l", "3000", "--no-clipboard"]
    ```
    Simpan dan tutup editor.

2.3. **Build Docker Image:**
    Dari direktori `/home/nazhif/lunar` di VPS, build image.
    ```bash
    docker build -t lunar-frontend .
    ```

### 3. Jalankan Docker Container

3.1. **Jalankan Container:**
    Jalankan container dari image `lunar-frontend`, memetakan port host **2998** ke port container **3000**.
    ```bash
    docker run -d -p 2998:3000 --name lunar-container --restart unless-stopped lunar-frontend
    ```

3.2. **Verifikasi Lokal di VPS:**
    Pastikan container berjalan dan merespons di port host `2998`.
    ```bash
    # Cek container aktif
    docker ps

    # Coba akses service dari dalam VPS
    curl http://localhost:2998
    # Anda seharusnya melihat output HTML dari Lunar
    ```

### 4. Setup Cloudflare Tunnel

4.1. **Buat Tunnel Baru:**
    ```bash
    sudo cloudflared tunnel create lunar-tunnel
    ```
    Catat ID Tunnel yang dihasilkan (misal: `fb1ef99f-...`) dan path file JSON credentials-nya.

4.2. **Buat File Konfigurasi Tunnel (`lunar-config.yml`):**
    ```bash
    sudo nano /root/.cloudflared/lunar-config.yml
    ```
    Isi dengan konfigurasi berikut, sesuaikan ID Tunnel dan path file JSON:
    ```yaml
    # Gunakan ID tunnel yang baru dibuat
    tunnel: fb1ef99f-aa11-4563-8f1f-f51508b59e36
    # atau nama tunnel:
    # tunnel: lunar-tunnel

    # Path ke file credentials JSON yang sesuai
    credentials-file: /root/.cloudflared/fb1ef99f-aa11-4563-8f1f-f51508b59e36.json

    # Aturan Ingress
    ingress:
      # Hostname publik yang diinginkan
      - hostname: kak_biro_kesra_jabar.nazhifsetya.site
        # Arahkan ke service Lunar yang berjalan di localhost port host 2998
        service: http://localhost:2998
      # Aturan fallback
      - service: http_status:404
    ```
    Simpan dan tutup editor.

4.3. **Routing DNS (Manual):**
    Perintah `cloudflared tunnel route dns lunar-tunnel kak_biro_kesra_jabar.nazhifsetya.site` mungkin mengarahkan ke domain yang salah (`.biz.id`). **Solusi:** Tambahkan record CNAME secara **manual** melalui Dashboard Cloudflare untuk domain `nazhifsetya.site`:
    -   **Type:** CNAME
    -   **Name:** `kak_biro_kesra_jabar`
    -   **Target:** `<LUNAR_TUNNEL_ID>.cfargotunnel.com` (contoh: `fb1ef99f-aa11-4563-8f1f-f51508b59e36.cfargotunnel.com`)
    -   **Proxy status:** Proxied (awan oranye)

### 5. Setup Systemd Service untuk Tunnel

5.1. **Buat Service File:**
    Salin service file yang ada dan edit untuk Lunar.
    ```bash
    sudo cp /etc/systemd/system/cloudflared-web.service /etc/systemd/system/cloudflared-lunar.service
    sudo nano /etc/systemd/system/cloudflared-lunar.service
    ```

5.2. **Edit Service File:**
    Ubah `Description` dan `ExecStart` agar menunjuk ke `lunar-config.yml`.
    ```ini
    [Unit]
    Description=Cloudflare Tunnel for Lunar Frontend # <- Ubah
    After=network-online.target
    Wants=network-online.target

    [Service]
    TimeoutStartSec=0
    Type=notify
    ExecStart=/usr/bin/cloudflared --config /root/.cloudflared/lunar-config.yml tunnel run # <- Ubah
    Restart=on-failure
    RestartSec=5s

    [Install]
    WantedBy=multi-user.target
    ```
    Simpan dan tutup editor.

5.3. **Reload, Enable, dan Start Service:**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable cloudflared-lunar.service
    sudo systemctl start cloudflared-lunar.service
    ```

### 6. Verifikasi Akhir

6.1. **Cek Status Service Tunnel:**
    ```bash
    sudo systemctl status cloudflared-lunar.service
    # Pastikan 'active (running)'
    ```

6.2. **Cek Koneksi Tunnel:**
    ```bash
    sudo cloudflared tunnel info lunar-tunnel
    # Pastikan ada CONNECTOR ID yang aktif
    ```

6.3. **Verifikasi DNS Propagation:**
    Gunakan tool seperti [dnschecker.org](https://dnschecker.org/) untuk memastikan record CNAME `kak_biro_kesra_jabar.nazhifsetya.site` sudah terpropagasi dan mengarah ke target tunnel yang benar.

6.4. **Akses Domain Publik:**
    Buka `https://kak_biro_kesra_jabar.nazhifsetya.site` di browser (mungkin perlu membersihkan cache atau menggunakan mode Incognito jika baru saja mengubah DNS).

## ⚠️ Catatan Troubleshooting
-   **Error 503 (Service Unavailable):** Seringkali disebabkan oleh masalah koneksi antara `cloudflared` di VPS dan service Docker (`http://localhost:2998`). Periksa log `cloudflared` (`sudo journalctl -u cloudflared-lunar.service -f`) untuk pesan seperti `No ingress rules defined` (indikasi masalah syntax/parsing `lunar-config.yml`) atau `failed to connect to origin`. Pastikan `lunar-config.yml` benar dan restart service (`sudo systemctl restart cloudflared-lunar.service`) setelah perbaikan.
-   **Website Tidak Muncul Setelah DNS Dibuat:** Biasanya karena penundaan propagasi DNS. Tunggu beberapa menit, bersihkan cache browser/DNS, dan gunakan [dnschecker.org](https://dnschecker.org/) untuk memverifikasi.

## 🎉 Selesai!
Aplikasi frontend Lunar sekarang berhasil dideploy menggunakan Docker dan dapat diakses secara publik melalui Cloudflare Tunnel.
