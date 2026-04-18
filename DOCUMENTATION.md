# Linguist v1 — Technical Documentation

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Directory Structure](#3-directory-structure)
4. [Tech Stack](#4-tech-stack)
5. [Database Schema](#5-database-schema)
6. [API Reference](#6-api-reference)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [Core Features](#8-core-features)
9. [AI / LLM Integration](#9-ai--llm-integration)
10. [Analytics & Metrics](#10-analytics--metrics)
11. [Frontend Architecture](#11-frontend-architecture)
12. [Deployment Guide](#12-deployment-guide)
13. [Environment Variables](#13-environment-variables)

---

## 1. Project Overview

**Linguist** is an AI-powered English language learning platform designed for students (ages ~10+). It uses a conversational AI character ("Linguist") to guide students through reading comprehension, speaking practice, and adaptive quizzing — all tied to the CEFR proficiency framework.

### Key Goals

- Make English learning engaging through story-driven AI conversation
- Provide real-time, multi-dimensional feedback on student responses
- Adapt difficulty automatically as students improve (CEFR levels A1 → C2)
- Give administrators full control over content, users, and progress tracking

---

## 2. Architecture

Linguist follows a **monolithic two-tier architecture** — a React SPA frontend served by Nginx, and a Flask REST API backend, both orchestrated via Docker Compose.

```
┌─────────────────────────────────────────────────────────┐
│                        Browser                          │
│                  React 19 (Vite/MUI)                    │
└─────────────────────────┬───────────────────────────────┘
                          │ HTTP / SSE
┌─────────────────────────▼───────────────────────────────┐
│               Nginx (Port 80)                           │
│   ┌───────────────────────────────────────────────┐    │
│   │  Serves React SPA  │  /api/v1/* → backend:8000│    │
│   └───────────────────────────────────────────────┘    │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│          Flask + Gunicorn (Port 8000)                   │
│   ┌────────────┐  ┌──────────────┐  ┌───────────────┐  │
│   │   auth/    │  │    chat/     │  │    quiz/      │  │
│   │   users/   │  │   stories/   │  │    admin/     │  │
│   └────────────┘  └──────────────┘  └───────────────┘  │
│   ┌───────────────────────────────────────────────────┐ │
│   │               services/                           │ │
│   │  chat_service  │  quiz_service  │  analytics      │ │
│   └───────────────────────────────────────────────────┘ │
└──────┬────────────────────┬───────────────┬─────────────┘
       │                    │               │
┌──────▼──────┐   ┌─────────▼──────┐  ┌────▼────────┐
│ PostgreSQL  │   │    LanceDB     │  │   Ollama    │
│ (Port 5432) │   │ (Vector Cache) │  │ (LLM Runtime│
└─────────────┘   └────────────────┘  └─────────────┘
```

### Request Flow (Chat)

1. Student types a message in `ChatPage`
2. React sends `POST /api/v1/chat/stream` via Axios
3. Nginx proxies to Flask backend
4. Flask calls Ollama LLM via LangChain/LangGraph
5. Response streams back as Server-Sent Events (SSE)
6. After streaming completes, analytics metrics are computed and saved to PostgreSQL

---

## 3. Directory Structure

```
linguist-v1/
├── linguist-backend/
│   ├── app/
│   │   ├── __init__.py           # Blueprint registration
│   │   ├── main.py               # App factory, JWT & CORS setup
│   │   ├── auth/
│   │   │   ├── routes.py         # POST /auth/login, /auth/refresh
│   │   │   └── views.py          # Login logic, token creation
│   │   ├── users/
│   │   │   ├── routes.py         # GET /users/{id}/cefr-level, /performance, /progress
│   │   │   └── views.py          # User data retrieval
│   │   ├── chat/
│   │   │   ├── routes.py         # POST /chat, /chat/stream
│   │   │   └── views.py          # Chat handler, SSE streaming
│   │   ├── quiz/
│   │   │   ├── routes.py         # POST /quiz/generate-mcq, /generate-blanks, /submit-mcq
│   │   │   └── views.py          # Quiz generation, CEFR level evaluation
│   │   ├── stories/
│   │   │   ├── routes.py         # GET /stories, /stories/{id}/paragraphs
│   │   │   └── views.py          # Story and paragraph retrieval
│   │   ├── admin/
│   │   │   ├── routes.py         # All /admin/* endpoints
│   │   │   └── views.py          # Admin logic, @admin_required decorator
│   │   ├── services/
│   │   │   ├── chat_service.py   # LangGraph + Ollama conversation
│   │   │   ├── quiz_service.py   # LLM quiz generation + LanceDB caching
│   │   │   └── analytics_service.py # TextAnalyzer (grammar, complexity, etc.)
│   │   ├── config/
│   │   │   ├── config.py         # Flask app config
│   │   │   ├── database.py       # PostgreSQL & LanceDB connections
│   │   │   └── log_config.py     # Logging setup
│   │   ├── constants/
│   │   │   ├── constants.py      # CEFR levels, topic maps, metric weights
│   │   │   └── prompts.py        # Linguist character system prompts
│   │   └── utils/                # Helper functions
│   ├── init.sql                  # PostgreSQL schema (14 tables)
│   ├── create_admin.sql          # Admin user seed
│   ├── requirements.txt          # Python dependencies
│   ├── Dockerfile                # CUDA-based backend image
│   └── run.py                    # Local dev entry point
│
├── linguist-frontent/
│   ├── src/
│   │   ├── main.jsx              # React entry
│   │   ├── App.jsx               # Root component
│   │   ├── routes/Routes.jsx     # React Router + PrivateRoute/AdminRoute
│   │   ├── pages/
│   │   │   ├── loginPage/        # Student login
│   │   │   ├── homePage/         # Progress dashboard
│   │   │   ├── chatPage/         # Chat with Linguist
│   │   │   ├── quizPage/         # MCQ quiz
│   │   │   ├── introToChatPage/  # Chat onboarding
│   │   │   └── admin/            # Admin login + dashboard
│   │   └── services/
│   │       ├── moduleApi.js      # All API call functions
│   │       └── axiosInstance.js  # JWT interceptors, auto-refresh
│   ├── nginx.conf                # SPA routing + /api/v1/* proxy
│   ├── Dockerfile                # Multi-stage: Node build → Nginx serve
│   └── package.json
│
├── docker-compose.yml            # Local development
├── docker-compose.hetzner.yml    # Cloud (Hetzner) production
├── .env                          # Local environment variables
└── .env.hetzner.example          # Cloud environment template
```

---

## 4. Tech Stack

### Backend

| Layer | Technology |
|---|---|
| Language | Python 3 |
| Web Framework | Flask |
| App Server | Gunicorn (4 workers, gevent) |
| Database | PostgreSQL 16 |
| Vector DB | LanceDB (question caching) |
| LLM Runtime | Ollama (local) / RunPod (cloud) |
| LLM Library | LangChain + LangGraph |
| Auth | Flask-JWT-Extended |
| NLP | TextBlob, scikit-learn |
| Container | Docker (CUDA 12.2 runtime base) |

### Frontend

| Layer | Technology |
|---|---|
| Framework | React 19 |
| Build Tool | Vite 6.1 |
| UI Library | Material-UI (MUI) v6.4 |
| Routing | React Router v7 |
| HTTP Client | Axios v1.8 |
| Charts | react-chartjs-2 |
| Animations | react-type-animation, react-spinners |
| Server | Nginx 1.27 (Alpine) |

---

## 5. Database Schema

PostgreSQL schema with 14 tables (defined in `linguist-backend/init.sql`).

### Core Tables

#### `User`
| Column | Type | Notes |
|---|---|---|
| id | SERIAL PRIMARY KEY | |
| username | VARCHAR UNIQUE | Login identifier |
| email | VARCHAR | |
| password | VARCHAR | Hashed |
| is_admin | BOOLEAN | Admin flag |

#### `UserProfile`
| Column | Type | Notes |
|---|---|---|
| user_id | FK → User | |
| english_level | VARCHAR | CEFR level (A1–C2) |
| age | INTEGER | |
| grade | INTEGER | |
| board | VARCHAR | Education board |
| native_language | VARCHAR | |
| quiz_unlocked | BOOLEAN | Unlocked after story completion |

#### `Story`
| Column | Type | Notes |
|---|---|---|
| id | SERIAL PRIMARY KEY | |
| title | VARCHAR | |
| difficulty_level | VARCHAR | Matched to CEFR level |

#### `Paragraph`
| Column | Type | Notes |
|---|---|---|
| id | SERIAL PRIMARY KEY | |
| story_id | FK → Story | |
| content | TEXT | |
| paragraph_index | INTEGER | Order within story |

#### `Attempt`
| Column | Type | Notes |
|---|---|---|
| id | SERIAL PRIMARY KEY | |
| user_id | FK → User | |
| paragraph_id | FK → Paragraph | |
| metrics | JSONB | Aggregated metrics |
| status | VARCHAR | in_progress / completed |

#### `ConversationSet`
| Column | Type | Notes |
|---|---|---|
| id | SERIAL PRIMARY KEY | |
| attempt_id | FK → Attempt | |
| sequence metrics | NUMERIC | Per-turn tracking |
| improvement_tracking | JSONB | Delta tracking |

#### `Message`
| Column | Type | Notes |
|---|---|---|
| id | SERIAL PRIMARY KEY | |
| conversation_set_id | FK → ConversationSet | |
| content | TEXT | |
| sender_type | VARCHAR | `student` or `ai` |

#### `MetricDelta`
Tracks per-message metric changes for improvement analysis.

#### `QuizBank`
| Column | Type | Notes |
|---|---|---|
| id | SERIAL PRIMARY KEY | |
| english_level | VARCHAR | CEFR level |
| grade | INTEGER | |
| question | TEXT | |
| options | JSONB | MCQ options |
| answer | VARCHAR | Correct answer |

#### `UserCEFR`
Tracks CEFR level history per user with timestamps.

---

## 6. API Reference

All endpoints are prefixed with `/api/v1`.

### Authentication

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/auth/login` | None | Login with username/password. Returns access & refresh tokens. |
| POST | `/auth/refresh` | JWT (refresh) | Get a new access token. |

**Login request:**
```json
{
  "username": "student1",
  "password": "password123"
}
```

**Login response:**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "user_details": {
    "user_id": 1,
    "username": "student1",
    "is_admin": false,
    "completed_paragraphs": [1, 2]
  }
}
```

---

### Users

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/users/{user_id}/cefr-level` | JWT | Get current CEFR level |
| GET | `/users/{user_id}/performance` | JWT | Get performance metrics |
| GET | `/users/{user_id}/progress` | JWT | Get story progress data |

---

### Chat

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/chat` | JWT | Send a message, receive full response + metrics |
| POST | `/chat/stream` | JWT | Send a message, receive streaming SSE response |

**Chat request:**
```json
{
  "user_id": 1,
  "paragraph_id": 5,
  "message": "The fox was very clever and sneaky.",
  "conversation_set_id": 12
}
```

**Stream response** (`text/event-stream`):
```
data: {"token": "That"}
data: {"token": " is"}
data: {"token": " a great observation!"}
data: {"done": true, "metrics": {...}}
```

---

### Quiz

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/quiz/generate-mcq` | JWT | Generate a multiple choice question |
| POST | `/quiz/generate-blanks` | JWT | Generate a fill-in-the-blanks question |
| POST | `/quiz/submit-mcq` | JWT | Submit answer and update CEFR level |

**Generate MCQ request:**
```json
{
  "user_id": 1,
  "cefr_level": "A2",
  "grade": 6,
  "topic": "Animals"
}
```

**Submit MCQ request:**
```json
{
  "user_id": 1,
  "score": 80,
  "current_level": "A2"
}
```

---

### Stories

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/stories?user_id={id}` | JWT | List all stories with user progress |
| GET | `/stories/{story_id}/paragraphs` | JWT | Get all paragraphs for a story |

---

### Admin (all require admin JWT)

| Method | Path | Description |
|---|---|---|
| GET | `/admin/overview` | Dashboard stats |
| GET/POST | `/admin/users` | List / create users |
| POST | `/admin/users/bulk-upload` | Bulk upload users via CSV |
| GET | `/admin/users/{user_id}/performance` | Student performance data |
| GET | `/admin/story-progress` | Progress across all students |
| GET/POST | `/admin/stories` | List / create stories |
| POST | `/admin/stories/bulk-upload` | Bulk upload stories |
| GET/PUT/DELETE | `/admin/stories/{story_id}/paragraphs` | Manage paragraphs |
| GET/POST/DELETE | `/admin/quiz-bank` | Manage quiz bank |

---

## 7. Authentication & Authorization

### Flow

```
Student → POST /auth/login
         ← access_token (60 min) + refresh_token (1 day)

(Request) → Authorization: Bearer <access_token>

(Token expired) → Axios interceptor catches 401
               → POST /auth/refresh with refresh_token
               ← new access_token
               → retry original request
```

### Token Storage (Frontend)

| Key | Storage | Value |
|---|---|---|
| `token` | localStorage | JWT access token |
| `refresh_token` | localStorage | JWT refresh token |
| `username` | localStorage | Logged-in username |
| `user_id` | sessionStorage | Numeric user ID |
| `completed_paragraphs` | sessionStorage | JSON array of paragraph IDs |

### Admin Authorization

A custom `@admin_required` decorator on all `/admin/*` routes checks the `is_admin` flag in the PostgreSQL `User` table. Non-admin JWTs receive a `403 Forbidden`.

---

## 8. Core Features

### Student Journey

```
Login → Home Dashboard → Select Story → Read Paragraph
     → Chat with Linguist AI → Get Metrics Feedback
     → Complete Paragraph → Unlock Quiz
     → Take MCQ Quiz → CEFR Level Assessed
     → Progress to Next Level / Next Paragraph
```

### CEFR Progression

| Level | Name | Quiz Pass Threshold |
|---|---|---|
| A1 | Beginner | ≥ 75% → advances to A2 |
| A2 | Elementary | ≥ 75% → advances to B1 |
| B1 | Intermediate | ≥ 75% → advances to B2 |
| B2 | Upper-Intermediate | ≥ 75% → advances to C1 |
| C1 | Advanced | ≥ 75% → advances to C2 |
| C2 | Proficient | Mastery |

### Chat Behavior (Linguist Character)

- Linguist is a **warm, playful story companion** — not a grammar teacher
- Maintains strict topic focus (redirects off-topic questions)
- Keeps responses under 3 sentences
- Uses a **3-level correction system** based on mistake frequency history:
  - Level 1: Gentle encouragement
  - Level 2: Subtle correction embedded in response
  - Level 3: Explicit correction with explanation
- Context-aware: knows the current story paragraph, user CEFR level, and past mistake summary

### Admin Capabilities

- Create/edit/delete students and their profiles
- Bulk upload students via CSV (`/admin/users/bulk-upload`)
- Create/edit/delete stories and paragraphs
- Bulk upload story content
- View per-student performance analytics
- Manage quiz bank (add questions, fill from paragraphs)

---

## 9. AI / LLM Integration

### Conversation (LangGraph + Ollama)

The chat service uses **LangGraph** to maintain a persistent conversation graph with message history per `conversation_set_id`. Each node in the graph:

1. Loads the conversation history from memory
2. Injects the system prompt (Linguist character + story context)
3. Calls **OllamaLLM** (`llama3.1` or configured model)
4. Returns the response token-by-token (streaming) or as a full string

**Prompt context includes:**
- Current paragraph text
- Student CEFR level
- Summary of past mistakes
- Full conversation history

### Quiz Generation (LLM + LanceDB Cache)

1. LLM is asked to generate a JSON-formatted MCQ question
2. Response is validated; if invalid JSON or wrong structure, retried (up to 3 times)
3. Generated question is checked against **LanceDB vector cache** to avoid repeating similar questions in the same session
4. Accepted questions are stored in the cache

**LLM Models (configurable via env):**
- Chat: `llama3.1` (default)
- Quiz: `phi3:mini` (faster, smaller)

### Ollama Hosting

| Environment | Ollama Host |
|---|---|
| Local dev | `http://ollama:11434` (Docker service) |
| Cloud (Hetzner) | `https://<runpod-id>-11434.proxy.runpod.net` |

---

## 10. Analytics & Metrics

Every student message is analyzed by the `TextAnalyzer` class (`analytics_service.py`) across 6 dimensions:

| Metric | Calculation Method | Weight |
|---|---|---|
| Grammar | TextBlob correction comparison | 18% |
| Complexity | Word length + syllable count (Flesch-Kincaid style) | 18% |
| Discourse | Word count vs. target (200 words max) | 18% |
| Politeness | Frequency of predefined polite phrases | 18% |
| Idiomatic | Detection of predefined English idioms | 18% |
| Cosine Similarity | TF-IDF vs. current paragraph | 10% |

**Overall Efficiency Formula:**
```
efficiency = 0.18×grammar + 0.18×complexity + 0.18×discourse
           + 0.18×politeness + 0.18×idiomatic + 0.10×cosine_similarity
```

Metrics are stored in:
- `MetricDelta` table (per message)
- `Attempt.metrics` (aggregated per paragraph attempt)
- Surfaced on the student `HomePage` progress chart

---

## 11. Frontend Architecture

### Routing & Guards

```
/                    → LoginPage          (public)
/HomePage            → HomePage           (PrivateRoute)
/ChatPageIntro       → ChatPageIntro      (PrivateRoute)
/chatpage            → ChatPage           (PrivateRoute)
/quizPage            → QuizPage           (PrivateRoute)
/admin/login         → AdminLoginPage     (public)
/admin/dashboard     → AdminDashboard     (AdminRoute)
```

`PrivateRoute` checks for `token` in localStorage.
`AdminRoute` checks for `token` and `is_admin` flag.

### Axios Instance (`axiosInstance.js`)

- Attaches `Authorization: Bearer <token>` to every request
- On `401` response:
  1. Calls `POST /auth/refresh`
  2. Saves new token to localStorage
  3. Retries the original request
  4. On refresh failure → clears localStorage → redirects to `/`

### Key Pages

| Page | Key Functionality |
|---|---|
| `LoginPage` | Username/password login, stores tokens |
| `HomePage` | Progress bar per story, CEFR badge, performance chart |
| `ChatPage` | Real-time SSE chat, message history, metric display |
| `QuizPage` | MCQ UI, score tracking, CEFR level result |
| `AdminDashboard` | Tabbed interface for users, stories, paragraphs, quiz bank |

---

## 12. Deployment Guide

### Local Development

**Prerequisites:** Docker Desktop, Docker Compose

```bash
# Clone and navigate
cd linguist-v1

# Copy and configure environment
cp .env.example .env   # Edit as needed

# Start all services
docker compose up --build

# Services:
#   Frontend:  http://localhost:80
#   Backend:   http://localhost:8000
#   pgAdmin:   http://localhost:5050
#   Ollama:    http://localhost:11434
```

On first start, `ollama-init` automatically pulls `llama3.1` and `phi3:mini` models.

**Seed admin user:**
```bash
docker exec -it <postgres-container> psql -U linguist -d linguist_db -f /create_admin.sql
```

---

### Cloud (Hetzner) Deployment

**Prerequisites:** Hetzner VPS, RunPod account (for Ollama GPU inference)

```bash
# On the Hetzner server
cd linguist-v1

# Configure production environment
cp .env.hetzner.example .env
# Edit: OLLAMA_HOST, POSTGRES_PASSWORD, JWT_SECRET_KEY, PGADMIN credentials

# Start production services
docker compose -f docker-compose.hetzner.yml up -d --build
```

**Cloud architecture differences:**
- PostgreSQL and pgAdmin bound to `localhost` only (SSH tunnel for access)
- Backend bound to `localhost:8000` (proxied by host Nginx)
- Frontend bound to `localhost:3000`
- Ollama hosted externally on RunPod (GPU inference)
- Host-level Nginx handles TLS termination and routing

---

### Docker Images Summary

| Service | Base Image | Exposed Port |
|---|---|---|
| Backend | `nvidia/cuda:12.2.0-runtime-ubuntu22.04` | 8000 |
| Frontend | `node:20-alpine` → `nginx:1.27-alpine` | 80 |
| PostgreSQL | `postgres:16-alpine` | 5432 |
| Ollama | `ollama/ollama` | 11434 |

---

## 13. Environment Variables

### Required Variables

| Variable | Description | Example |
|---|---|---|
| `OLLAMA_HOST` | Ollama server URL | `http://ollama:11434` |
| `CHAT_LLM_MODEL` | LLM model for chat | `llama3.1` |
| `QUIZ_LLM_MODEL` | LLM model for quiz generation | `phi3:mini` |
| `POSTGRES_HOST` | PostgreSQL host | `postgres` |
| `POSTGRES_PORT` | PostgreSQL port | `5432` |
| `POSTGRES_DB` | Database name | `linguist_db` |
| `POSTGRES_USER` | Database user | `linguist` |
| `POSTGRES_PASSWORD` | Database password | *(strong password)* |
| `JWT_SECRET_KEY` | JWT signing key | *(random hex string)* |
| `JWT_ACCESS_TOKEN_EXPIRES_MINUTES` | Access token TTL | `60` |
| `JWT_REFRESH_TOKEN_EXPIRES` | Refresh token TTL (seconds) | `86400` |

### Optional Variables

| Variable | Description |
|---|---|
| `HF_TOKEN` | Hugging Face token (optional models) |
| `PGADMIN_EMAIL` | pgAdmin login email |
| `PGADMIN_PASSWORD` | pgAdmin login password |

---

*Documentation generated for Linguist v1, branch `feature/pradeep`*
