# Deployment Setup (Hetzner + RunPod)

This guide covers deploying Linguist to a Hetzner cloud server. The LLM inference is offloaded to a [RunPod](https://runpod.io) GPU pod running Ollama, since Hetzner servers are CPU-only.

---

## Architecture

```
Internet
   │
   ▼
Nginx (host, port 443/80)   ← SSL termination, domain routing
   │
   ├─► Frontend container  (port 3000, React/Nginx)
   │
   └─► Backend container   (port 8000, Flask/Gunicorn)
          │
          ├─► PostgreSQL container  (port 5432, loopback only)
          └─► Ollama (RunPod)       (remote HTTPS endpoint)
```

---

## Prerequisites

### Hetzner server
- Ubuntu 22.04 LTS (CX21 or higher recommended)
- Docker + Docker Compose installed
- A domain name pointed at the server's IP

### RunPod
- A RunPod account with a GPU pod running the `ollama/ollama` image
- Pod must expose port `11434`
- Models `llama3.1` and `phi3:mini` pulled inside the pod

---

## 1. Provision the Hetzner server

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Install nginx and certbot
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx
```

---

## 2. Start a RunPod Ollama pod

1. Go to [RunPod](https://runpod.io) → **Deploy** → choose a GPU pod (RTX 3090 or A4000 for best price/performance).
2. Use the template `ollama/ollama:latest` or set the Docker image to `ollama/ollama`.
3. Expose port **11434** (HTTP).
4. After the pod starts, open a terminal in the pod and pull the required models:

```bash
ollama pull llama3.1
ollama pull phi3:mini
```

5. From the pod's **Connect** page, copy the public proxy URL — it looks like:
   ```
   https://<pod-id>-11434.proxy.runpod.net
   ```

---

## 3. Clone the repo on the server

```bash
git clone <repo-url>
cd linguist-
```

---

## 4. Create environment files

### Root `.env` (for docker-compose.hetzner.yml)

```bash
cp .env.hetzner.example .env
nano .env
```

Fill in all values:

```bash
# RunPod Ollama URL from Step 2
OLLAMA_HOST=https://REPLACE_WITH_RUNPOD_POD_ID-11434.proxy.runpod.net

QUIZ_LLM_MODEL=phi3:mini

# Strong unique passwords
POSTGRES_DB=linguist
POSTGRES_USER=linguist
POSTGRES_PASSWORD=<strong-random-password>

PGADMIN_EMAIL=admin@yourdomain.com
PGADMIN_PASSWORD=<strong-random-password>

# Generate: python3 -c "import secrets; print(secrets.token_hex(32))"
JWT_SECRET_KEY=<strong-random-secret>

HF_TOKEN=   # optional
```

### Backend `.env`

```bash
cp linguist-backend/app/.env.example linguist-backend/app/.env
nano linguist-backend/app/.env
```

```bash
ENV=Production
JWT_SECRET_KEY=<same value as above>
JWT_ACCESS_TOKEN_EXPIRES_MINUTES=60
JWT_REFRESH_TOKEN_EXPIRES=1
```

---

## 5. Create `Dockerfile.hetzner`

The Hetzner compose file expects `linguist-backend/Dockerfile.hetzner`. Create it as a CPU-only variant (no CUDA base image):

```bash
cat > linguist-backend/Dockerfile.hetzner << 'EOF'
FROM python:3.11-slim

WORKDIR /app
COPY . /app

RUN apt-get update && apt-get install -y git curl && rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir -r requirements.txt
RUN pip install git+https://github.com/openai/whisper.git
RUN python -m textblob.download_corpora

EXPOSE 8000
CMD ["gunicorn", "app.main:linguist_app", "--bind", "0.0.0.0:8000", "--workers", "4", "--worker-class", "gevent", "--timeout", "300", "--keep-alive", "5"]
EOF
```

---

## 6. Build and start the stack

```bash
docker compose -f docker-compose.hetzner.yml --env-file .env up -d --build
```

This starts: `postgres`, `pgadmin`, `backend`, `frontend`. Ollama runs on RunPod — not locally.

Check all services are healthy:

```bash
docker compose -f docker-compose.hetzner.yml ps
```

---

## 7. Configure nginx on the host

The Docker frontend binds to `127.0.0.1:3000` and backend to `127.0.0.1:8000`. The host nginx sits in front and handles SSL + domain routing.

Create `/etc/nginx/sites-available/linguist`:

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    # Redirect HTTP → HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # Proxy all traffic to the frontend container
    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }

    # The frontend container's nginx already proxies /api/v1/* internally,
    # so no separate backend location block is needed here.

    # Increase timeouts for SSE chat streaming
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;
}
```

Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/linguist /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 8. Obtain SSL certificate

```bash
sudo certbot --nginx -d yourdomain.com
```

Certbot auto-renews. Verify with:

```bash
sudo certbot renew --dry-run
```

---

## 9. Create the admin user

```bash
docker exec -i $(docker compose -f docker-compose.hetzner.yml ps -q postgres) \
  psql -U linguist -d linguist << 'EOF'
INSERT INTO "User" (username, email, password, is_admin)
VALUES ('admin', 'admin@yourdomain.com', 'yourpassword', TRUE)
ON CONFLICT (username) DO UPDATE
    SET email = EXCLUDED.email,
        password = EXCLUDED.password,
        is_admin = TRUE;
EOF
```

---

## 10. Access pgAdmin (via SSH tunnel)

pgAdmin binds to `127.0.0.1:5050` (loopback only). Access it from your local machine via SSH tunnel:

```bash
ssh -L 5050:127.0.0.1:5050 user@yourdomain.com
```

Then open [http://localhost:5050](http://localhost:5050) in your browser.

---

## Useful Commands (on the server)

```bash
# View logs
docker compose -f docker-compose.hetzner.yml logs -f backend
docker compose -f docker-compose.hetzner.yml logs -f frontend

# Restart a service
docker compose -f docker-compose.hetzner.yml restart backend

# Pull latest code and redeploy
git pull
docker compose -f docker-compose.hetzner.yml --env-file .env up -d --build

# Open a psql shell
docker exec -it $(docker compose -f docker-compose.hetzner.yml ps -q postgres) \
  psql -U linguist -d linguist

# Stop everything
docker compose -f docker-compose.hetzner.yml down
```

---

## Environment Variable Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `OLLAMA_HOST` | Yes | RunPod Ollama public URL |
| `QUIZ_LLM_MODEL` | Yes | Model name for quiz generation (e.g. `phi3:mini`) |
| `POSTGRES_PASSWORD` | Yes | PostgreSQL password |
| `PGADMIN_PASSWORD` | Yes | pgAdmin login password |
| `JWT_SECRET_KEY` | Yes | JWT signing secret (also set in `app/.env`) |
| `POSTGRES_DB` | No | Database name (default: `linguist`) |
| `POSTGRES_USER` | No | Database user (default: `linguist`) |
| `PGADMIN_EMAIL` | No | pgAdmin login email (default: `admin@admin.com`) |
| `HF_TOKEN` | No | Hugging Face token for faster embedding model downloads |
| `TTS_HOST` | No | Text-to-speech service URL |
