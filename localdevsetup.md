# Local Development Setup

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Docker Desktop | Latest | Enable WSL2 backend on Windows |
| Node.js | 20+ | For running the Vite dev server |
| Git | Any | |

NVIDIA GPU is supported but not required for running the app locally. The Ollama LLM service will run in CPU mode without one.

---

## 1. Clone the repo

```bash
git clone <repo-url>
cd linguist-
```

---

## 2. Create environment files

### Backend (`linguist-backend/app/.env`)
```bash
cp linguist-backend/app/.env.example linguist-backend/app/.env
```

Edit `linguist-backend/app/.env` and set a JWT secret:
```
ENV=Development
JWT_SECRET_KEY=<generate with: python -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())">
JWT_ACCESS_TOKEN_EXPIRES_MINUTES=60
JWT_REFRESH_TOKEN_EXPIRES=1
```

### Frontend (`linguist-frontent/.env`)
```bash
cp linguist-frontent/.env.example linguist-frontent/.env
```

The default value (`/api/v1`) works as-is — no changes needed for local dev.

### Root (optional)
```bash
cp .env.example .env
```

`HF_TOKEN` and `TTS_HOST` are optional and default to empty. Only fill them in if you need Hugging Face authenticated downloads or a TTS service.

---

## 3. Start the infrastructure

```bash
docker compose up -d
```

This starts six services:

| Service | Port | Description |
|---------|------|-------------|
| `postgres` | 5432 | PostgreSQL 16 database |
| `pgadmin` | 5050 | Database GUI |
| `ollama` | 11434 | Local LLM runtime |
| `ollama-init` | — | Pulls `llama3.1` + `phi3:mini` (runs once, then exits) |
| `backend` | 8000 | Flask API (Flask debug mode, auto-reloads on save) |
| `frontend` | 80 | Nginx serving the production React build |

> **First run only:** `ollama-init` pulls two models (~7 GB total). Wait for it to finish before testing chat or quiz features. Check progress with `docker logs -f linguist--ollama-init-1`.

---

## 4. Create an admin user

```bash
docker exec -i linguist--postgres-1 psql -U linguist -d linguist << 'EOF'
INSERT INTO "User" (username, email, password, is_admin)
VALUES ('admin', 'admin@example.com', 'yourpassword', TRUE)
ON CONFLICT (username) DO UPDATE
    SET email = EXCLUDED.email,
        password = EXCLUDED.password,
        is_admin = TRUE;
EOF
```

---

## 5. Verify everything is running

```bash
# All containers should show "healthy" or "running"
docker compose ps

# Backend API
curl http://localhost:8000/api/v1/auth/login -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}'

# Frontend (should return HTTP 200)
curl -o /dev/null -s -w "%{http_code}" http://localhost:80
```

Open [http://localhost](http://localhost) in your browser and log in.

---

## Development Workflow

### Backend — auto-reload on save

The `docker-compose.override.yml` mounts the Python source into the container and runs Flask in debug mode. **Any change to a file in `linguist-backend/app/` restarts the server automatically** — no manual restart needed.

```bash
# Watch live backend logs
docker logs -f linguist--backend-1

# Manual restart if needed
docker compose restart backend
```

### Frontend — Vite dev server (hot reload)

The Docker frontend on port 80 serves the last built bundle. For live hot-reload while editing React components:

```bash
cd linguist-frontent
npm install       # first time only
npm run dev       # starts on http://localhost:5173
```

Vite proxies all `/api/*` requests to `http://localhost:8000`, so the running Docker backend handles all API calls.

> Use **port 5173** for development (hot reload). Use **port 80** to test the built production bundle.

---

## Useful Commands

```bash
# View logs for any service
docker logs -f linguist--backend-1
docker logs -f linguist--frontend-1
docker logs -f linguist--postgres-1

# Restart a single service
docker compose restart backend

# Rebuild and restart backend after changing requirements.txt
docker compose build backend && docker compose up -d backend

# Rebuild frontend Docker image (production bundle)
docker compose build frontend && docker compose up -d frontend

# Open a psql shell
docker exec -it linguist--postgres-1 psql -U linguist -d linguist

# Stop everything
docker compose down

# Stop and wipe all data volumes (destructive)
docker compose down -v
```

---

## Service URLs

| URL | Service |
|-----|---------|
| http://localhost | Frontend (production build) |
| http://localhost:5173 | Frontend (Vite dev server, hot reload) |
| http://localhost:8000/api/v1 | Backend API |
| http://localhost:5050 | pgAdmin (admin@admin.com / admin) |
| http://localhost:11434 | Ollama API |

---

## Project Structure (quick reference)

```
linguist-
├── docker-compose.yml          # Base service definitions
├── docker-compose.override.yml # Dev overrides (auto-applied) — backend hot reload
├── linguist-backend/
│   ├── app/                    # Flask source (mounted into backend container)
│   │   ├── admin/              # Admin routes + views
│   │   ├── auth/               # Login + token refresh
│   │   ├── chat/               # SSE streaming chat
│   │   ├── quiz/               # MCQ generation
│   │   ├── users/              # User profiles + CEFR
│   │   ├── stories/            # Story + paragraph retrieval
│   │   ├── services/           # LangChain, quiz, analytics logic
│   │   └── config/             # DB, LanceDB, logging setup
│   └── init.sql                # PostgreSQL schema
└── linguist-frontent/
    ├── src/
    │   ├── pages/              # Route-level React components
    │   └── services/           # Axios instance + API calls
    └── vite.config.js          # Dev server + proxy config
```
