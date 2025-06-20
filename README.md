# Huly Self-Hosted on Windows (WSL2)

This guide explains how to deploy **Huly** on a Windows machine using **WSL2 (Windows Subsystem for Linux)**. It follows the official Linux setup with adaptations for Windows compatibility.

> **Minimum Recommended Requirements**
>
> * Windows 10/11
> * WSL2 with Ubuntu 20.04 or later
> * Docker Desktop (WSL2 integration enabled)
> * At least 2 vCPUs and 4GB RAM assigned to Docker

---

## 1. Setup WSL2 and Docker

### Enable WSL2 and Install Ubuntu:

```powershell
wsl --install
```

Restart your machine when prompted.

### Install Docker Desktop for Windows:

* Download: [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
* Enable **WSL2 Integration** in Docker Desktop settings (Settings > Resources > WSL Integration)
* Ensure your Ubuntu distro is checked

---

## 2. Access Your Project Directory from WSL

Navigate to your Windows directory from Ubuntu:

```bash
cd /mnt/c/Users/<YourUsername>/Desktop/huly/huly-selfhost
```

---

## 3. Fix Line Endings (CRLF → LF)

Ensure all `.sh` and `.conf` files use Unix line endings:

```bash
dos2unix *.sh *.conf
```

---

## 4. Run Setup Script

```bash
./setup.sh
```

This script will:

* Ask for `HOST_ADDRESS`, `HTTP_PORT`, etc.
* Generate `.env` and `huly.conf`
* Create `.huly.nginx` from the template

---

## 5. Validate Environment File

Make sure `.env` exists and contains:

```env
HULY_VERSION=v0.6.501
HOST_ADDRESS=localhost
HTTP_PORT=80
HTTP_BIND=0.0.0.0
SECRET=<your-secret>
... # Additional values from setup.sh
```

---

## 6. Start Huly via Docker Compose

```bash
docker compose up -d
```

To verify containers:

```bash
docker compose ps
```

---

## 7. Access the Web UI

Visit:

```
http://localhost
```

If you see the default Nginx welcome page, verify:

* `.huly.nginx` has LF endings (`file .huly.nginx` should say "ASCII text")
* It's correctly mounted in `docker-compose.yml`
* Port 80 isn’t hijacked by native WSL Nginx (`sudo apt purge nginx`)

---

## 8. Optional: Configure SMTP for Email Notifications

Add to `docker-compose.yml`:

```yaml
  mail:
    image: hardcoreeng/mail:v0.6.501
    ports:
      - 8097:8097
    environment:
      - PORT=8097
      - SOURCE=your@email.com
      - SMTP_HOST=smtp.server.com
      - SMTP_PORT=587
      - SMTP_USERNAME=your_smtp_user
      - SMTP_PASSWORD=your_smtp_pass
```

Add this to `account` and `transactor`:

```yaml
- MAIL_URL=http://mail:8097
```

---

## 9. Optional: AI Service Setup

```yaml
  aibot:
    image: hardcoreeng/ai-bot:v0.6.501
    ports:
      - 4010:4010
    environment:
      - OPENAI_API_KEY=your_openai_key
      - STORAGE_CONFIG=minio|minio?accessKey=minioadmin&secretKey=minioadmin
      - SERVER_SECRET=${SECRET}
      - ACCOUNTS_URL=http://account:3000
      - DB_URL=mongodb://mongodb:27017
```

Also add to:

* `front`: `AI_URL=http://aibot:4010`
* `transactor`: `AI_BOT_URL=http://aibot:4010`

---

## 10. VAPID Keys for Push Notifications

Install Node.js:

```bash
sudo apt install npm -y
sudo npm install -g web-push
web-push generate-vapid-keys
```

Add keys to `front` container:

```yaml
- PUSH_PUBLIC_KEY=...
- PUSH_PRIVATE_KEY=...
```

---

## 11. Common Fixes

* `ENOTFOUND your.host.name.or.ip`: means `.env` or `huly.conf` contains a placeholder. Replace with `localhost`.
* Nginx welcome page? Run:

  ```bash
  docker exec -it huly-nginx-1 cat /etc/nginx/conf.d/default.conf
  ```

  It must show proxy rules for `/` → `http://front:8080`

---

## 12. Post-install Notes

* To disable signup:

```yaml
- DISABLE_SIGNUP=true
```

Add to `front` and `account`

* To reset everything:

```bash
docker compose down -v
./setup.sh --secret
docker compose up -d
```

---

You now have Huly running self-hosted on Windows with WSL2. Welcome to your new workspace.
