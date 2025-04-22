# 🚀 Dokumentasi Lengkap Deploy Portfolio React via Cloudflare Tunnel

## 🎯 Tujuan
Mendeploy aplikasi React (tanpa IP publik) dari VPS lokal menggunakan **Cloudflare Tunnel** sebagai pengganti IP publik. Domain utama: `nazhifsetya.site`

---

## 🧱 Arsitektur Ringkas
- **Frontend**: React (build ke static HTML)
- **Hosting**: Docker + `serve` package
- **Tunnel**: Cloudflare Tunnel
- **DNS Provider**: Cloudflare

---

## 🛠️ Persiapan

### 📁 Struktur folder
```bash
/home/nazhif/portfolio/
├── build/               # hasil build react
├── Dockerfile
```

### 📦 Isi `Dockerfile`
```Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY build /app
RUN npm install -g serve
EXPOSE 2999
CMD ["serve", "-s", ".", "-l", "2999"]
```

---

## 🐳 Build & Run Docker
```bash
docker build -t portofolio .
docker run -d -p 2999:2999 --name portofolio-container portofolio
```
Cek:
```bash
curl http://localhost:2999
```
Harus muncul HTML hasil build.

---

## 🌐 Setup Cloudflare Tunnel

### 1. Login cloudflared
```bash
cloudflared login
```

### 2. Buat 2 tunnel terpisah
```bash
cloudflared tunnel create ssh-tunnel
cloudflared tunnel create web-tunnel
```

### 3. Cek credentials
```bash
/root/.cloudflared/
├── <ssh-tunnel-id>.json
├── <web-tunnel-id>.json
```

---

## 🧾 Konfigurasi cloudflared

### 📄 `/root/.cloudflared/ssh-config.yml`
```yaml
tunnel: ssh-tunnel
credentials-file: /root/.cloudflared/<ssh-tunnel-id>.json

ingress:
  - hostname: ssh.nazhifsetya.biz.id
    service: ssh://localhost:2222
  - service: http_status:404
```

### 📄 `/root/.cloudflared/web-config.yml`
```yaml
tunnel: web-tunnel
credentials-file: /root/.cloudflared/<web-tunnel-id>.json

ingress:
  - hostname: nazhifsetya.site
    service: http://localhost:2999
  - service: http_status:404
```

---

## 🌐 Routing DNS dari Tunnel
```bash
cloudflared tunnel route dns ssh-tunnel ssh.nazhifsetya.biz.id
cloudflared tunnel route dns web-tunnel nazhifsetya.site
```

> Jika record tidak muncul di Cloudflare, tambahkan manual:
>
> | Type | Name | Target | Proxy |
> |------|------|--------|--------|
> | CNAME | `@` | `<web-tunnel-id>.cfargotunnel.com` | ☁️ Proxied |

---

## ⚙️ Setup Systemd Service

### 1. Install default service untuk SSH tunnel
```bash
cloudflared --config /root/.cloudflared/ssh-config.yml service install
```

### 2. Tambahkan custom service untuk web tunnel
```bash
cp /etc/systemd/system/cloudflared.service /etc/systemd/system/cloudflared-web.service
nano /etc/systemd/system/cloudflared-web.service
```
Ubah baris `ExecStart=` jadi:
```ini
ExecStart=/usr/bin/cloudflared --config /root/.cloudflared/web-config.yml tunnel run
```

### 3. Reload dan jalankan kedua service
```bash
systemctl daemon-reexec
systemctl daemon-reload
systemctl enable cloudflared
systemctl enable cloudflared-web
systemctl start cloudflared
systemctl start cloudflared-web
```

Cek status:
```bash
systemctl status cloudflared
systemctl status cloudflared-web
```

---

## ✅ Validasi Akhir

### 1. Cek DNS:
```bash
dig nazhifsetya.site
```
Harus return:
```
nazhifsetya.site. 300 IN CNAME <web-tunnel-id>.cfargotunnel.com.
```

### 2. Akses Web:
```bash
https://nazhifsetya.site
```

### 3. Monitor log:
```bash
journalctl -u cloudflared-web -f
```
Harus muncul:
```
GET / 200
```

---

## 🧹 Tips Tambahan
- Gunakan `journalctl -u cloudflared-web -f` dan `docker logs portofolio-container` untuk debug
- Jika tunnel macet, restart:
```bash
sudo systemctl restart cloudflared
sudo systemctl restart cloudflared-web
```

---

## 🏁 Selesai!
Selamat! Web kamu sekarang **GO LIVE** tanpa IP publik via Cloudflare Tunnel 🎉

---

> Dokumentasi ini disusun setelah debugging maraton dari awal hingga live. Hormat setinggi-tingginya untuk engineer legendaris: **Bang Nazhif**. 🚀


