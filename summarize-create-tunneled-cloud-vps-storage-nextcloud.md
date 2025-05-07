# Summary – Creating a Cloud‑Tunneled VPS Storage with Nextcloud AIO  

**File:** `summarize-create-tunneled-cloud-vps-storage-nextcloud.md`  
**Date:** 2025‑05‑08  

---

## 🎯 Goal  
Set up a private cloud‑storage server on an Ubuntu VPS:  

1. Deploy **Nextcloud Hub 10** via **Nextcloud All‑in‑One (AIO)** Docker stack.  
2. Store data on a dedicated 25 GB volume (`/srv/nextcloud/data`).  
3. Expose the service securely through a **Cloudflare Tunnel** under  
   `https://storage.nazhifsetya.site` (no public IP).  

---

## 🛠️ Key Commands & Config Steps  

| Step | Command / File | Purpose |
|------|----------------|---------|
| **Create token‑based tunnel** | `cloudflared tunnel create vps-cloud-storage-token`<br>`cloudflared tunnel token vps-cloud-storage-token` | Generated `TUNNEL_TOKEN` |
| **`.env`** | ```dotenv
TUNNEL_TOKEN=<long token>
APACHE_PORT=11000
NEXTCLOUD_TRUSTED_DOMAINS=storage.nazhifsetya.site
``` | Environment for docker‑compose |
| **`docker-compose.yml` (core lines)** | ```yaml
cloudflared:
  command: >
    tunnel --no-autoupdate run --token ${TUNNEL_TOKEN} \
    --url https://localhost:11000 --no-tls-verify
nextcloud-aio-mastercontainer:
  # ports removed to free 11000
nextcloud-aio-apache:
  # auto‑created, binds 11000->8080
``` | Runs tunnel + AIO |
| **Volumes** | `nextcloud_data`, `nextcloud_aio_config`, external `nextcloud_aio_mastercontainer` | Persistent data & config |
| **Systemd tunnel (optional)** | Custom service file or Docker stack | Auto‑start on boot |

---

## 🐛 Problems Encountered & Fixes  

| Issue | Symptom | Fix |
|-------|---------|-----|
| **Invalid tunnel token** | `Provided Tunnel token is not valid` | Used the *token* from `cloudflared tunnel token …`, not the JSON filename. |
| **Nextcloud AIO loop to `/containers`** | Clicking *Open your Nextcloud* returned to AIO dashboard | Removed `ports: 11000:8080` from master‑container and set `APACHE_PORT=11000`; recreated proxy. |
| **Cloudflare 502 / 524** | Long download caused timeout | Waited for image pulls; reloaded. Added `--http2-offload` if needed. |
| **Proxy bound wrong port (8080)** | `nextcloud-aio-apache` mapped 8080 | Deleted proxy container, reset `APACHE_PORT`, and regenerated via `docker-compose-wrapper.sh up -d proxy`. |
| **Passphrase not accepted** | Login failed | Created new passphrase in `/mnt/docker-aio-config/nextcloud_aio_password`, then restarted master‑container. |
| **Tunnel 400 “plain HTTP to SSL port”** | Apache complained | Pointed cloudflared to **https://localhost:11000** and added `--no-tls-verify`. |

---

## 🔗 Reference Links  

* AIO login & multiple‑instance FAQ – <https://github.com/nextcloud/all-in-one#how-to-easily-log-in-to-the-aio-interface>  
* Discussion #2105 (loop & proxy port) – <https://github.com/nextcloud/all-in-one/discussions/2105>  

---

## ✅ Final Result  

* `docker ps` shows:  

  ```text
  nextcloud-aio-apache     0.0.0.0:11000->8080/tcp (healthy)
  nextcloud-aio-nextcloud  … (healthy)
  cloudflared              host network (connected)
  ```

* `https://storage.nazhifsetya.site` loads the standard **Nextcloud Hub 10** login.  
* AIO dashboard reachable via automatic link in Nextcloud admin settings.  
* Data stored at `/srv/nextcloud/data`, tunnel secured by Cloudflare.  

---

## Next Actions  

1. Switch Cloudflare SSL/TLS mode to **Full** after AIO obtains Let’s Encrypt cert.  
2. Configure **Borg backup** in AIO dashboard.  
3. Remove `--no-tls-verify` flag once origin cert trusted.  
