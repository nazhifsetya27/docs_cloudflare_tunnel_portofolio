# PostgreSQL Access via Cloudflare Tunnel - Personal VPS Setup

This documentation outlines the steps taken to expose a PostgreSQL service running in Docker on a personal VPS, accessed securely via Cloudflare Tunnel and managed from a remote client (e.g., DBeaver or psql).

---

## ✅ Objectives

* Deploy PostgreSQL in a Docker container on a VPS
* Expose it securely through a Cloudflare Tunnel
* Access it from remote machine using `psql` or GUI (DBeaver)

---

## 1. Run PostgreSQL via Docker

Create a file `docker-compose.postgres.yml`:

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15
    container_name: postgres_service
    restart: unless-stopped
    environment:
      POSTGRES_USER: nazhif
      POSTGRES_PASSWORD: yourpassword
      POSTGRES_DB: argus-nazhif-dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Start it with:

```bash
docker compose -f docker-compose.postgres.yml up -d
```

---

## 2. Set up Cloudflare Tunnel

### a. Create tunnel and config

```bash
cloudflared tunnel login
cloudflared tunnel create postgres-tunnel
```

Then create config:

```bash
/etc/cloudflared/postgres.yml
```

```yaml
tunnel: postgres-tunnel
credentials-file: /etc/cloudflared/<UUID>.json

ingress:
  - hostname: postgres.nazhifsetya.site
    service: tcp://localhost:5432
  - service: http_status:404
```

### b. Add public hostname in Zero Trust dashboard

Navigate to **Access > Tunnels > postgres-tunnel > Public Hostnames** and add:

| Field     | Value                  |
| --------- | ---------------------- |
| Subdomain | postgres               |
| Domain    | nazhifsetya.site       |
| Service   | `tcp://localhost:5432` |

Cloudflare will automatically create a proxied CNAME DNS record.

---

## 3. Create systemd service for tunnel

```ini
# /etc/systemd/system/cloudflared-postgres.service
[Unit]
Description=Cloudflare Tunnel for PostgreSQL
After=network-online.target
Wants=network-online.target

[Service]
User=cloudflared
ExecStart=/usr/bin/cloudflared --config /etc/cloudflared/postgres.yml tunnel run
Restart=on-failure
RestartSec=5s
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### a. Fix common errors

**Error:** `Failed to determine user credentials: No such process`
**Solution:**

```bash
sudo useradd -r -s /usr/sbin/nologin cloudflared
sudo chown -R cloudflared:cloudflared /etc/cloudflared/
sudo chmod 640 /etc/cloudflared/*
```

Then reload and enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cloudflared-postgres.service
```

Check logs:

```bash
journalctl -u cloudflared-postgres -f
```

---

## 4. Accessing the Database Remotely

### a. Using `cloudflared access tcp`

On your local machine:

```bash
cloudflared access tcp --hostname postgres.nazhifsetya.site --url localhost:15432
```

Then in another terminal:

```bash
psql -h localhost -p 15432 -U nazhif -d argus-nazhif-dev sslmode=disable
```

### b. Using DBeaver

In DBeaver, create a PostgreSQL connection:

* Host: `localhost`
* Port: `15432`
* Database: `argus-nazhif-dev`
* Username: `nazhif`
* Password: `yourpassword`
* SSL: disable (unless you configured Postgres TLS manually)

---

## 5. Known Issues and Fixes

### ❌ `cloudflared-postgres.service: Failed to determine user credentials`

**Fix:** Ensure `User=cloudflared` exists and owns config files.

### ❌ `psql` or DBeaver hangs or times out

**Fix:** You must run `cloudflared access tcp` before trying to connect — or set up WARP/Private Network for always-on access.

### ❌ `dig +short CNAME ...` returns nothing

**Explanation:** Cloudflare uses CNAME flattening. Check regular `dig` or use DNS tab in dashboard.

---

## ✅ Final Outcome

* PostgreSQL served on Docker container
* Secure Cloudflare Tunnel exposes it to subdomain
* Successfully connected using CLI (`psql`) and GUI (DBeaver)

