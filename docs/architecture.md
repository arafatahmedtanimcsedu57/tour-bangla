# System Architecture — Tour Bangladesh

## Design Decisions Summary

| Decision | Choice | Reason |
|----------|--------|--------|
| Scale target | <1,000 users | Community project, no horizontal scaling needed |
| Hosting | Hetzner CX21 VPS (~€4/mo) | Cost-effective, single server, Docker Compose |
| Auth | Zitadel (self-hosted) | Lightweight (~100MB RAM), OIDC, replaces Keycloak |
| CI/CD | GitHub Actions → VPS SSH deploy | Push to `main` triggers automated deploy |
| Recovery | Daily DB backups + auto-restart | <30min acceptable downtime |
| Image processing | FastAPI resize on upload → R2 | Free, no third-party dependency |
| Caching | Redis (homepage feed + search) | Minimal layer, big query relief |
| Map tiles | OpenStreetMap | Free, no API key, good Bangladesh coverage |
| Monitoring | Grafana + Loki + UptimeRobot | Logs + uptime alerts, zero cost |
| Search | PostgreSQL full-text (`tsvector`) | English only, no external search engine |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Users (Mobile/Desktop)                     │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ HTTPS
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Vercel (CDN Edge)                           │
│                    Next.js 14 — Frontend                            │
│          Static assets cached globally at edge nodes               │
└──────────┬──────────────────────────────────┬───────────────────────┘
           │ OIDC (login/logout)              │ REST API (Bearer JWT)
           │                                  │
           ▼                                  ▼
┌──────────────────────┐          ┌───────────────────────────────────┐
│  Hetzner VPS CX21    │          │       Hetzner VPS CX21            │
│  Zitadel             │          │       Nginx (reverse proxy)       │
│  port 8081           │          │       port 80/443                 │
│  ~100MB RAM          │          └──────────────┬────────────────────┘
└──────────────────────┘                         │
           (same VPS)                            │
                                    ┌────────────┼────────────┐
                                    ▼            ▼            ▼
                             ┌──────────┐ ┌──────────┐ ┌──────────┐
                             │ FastAPI  │ │ Grafana  │ │ Loki     │
                             │ port 8000│ │ port 3001│ │ port 3100│
                             └────┬─────┘ └──────────┘ └──────────┘
                                  │
                     ┌────────────┼────────────┐
                     ▼            ▼            ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │PostgreSQL│ │  Redis   │ │  Loki    │
              │+ PostGIS │ │ (cache)  │ │ (logs)   │
              │ port 5432│ │ port 6379│ │          │
              └──────────┘ └──────────┘ └──────────┘

                                    ┌──────────────────────┐
                                    │   Cloudflare R2      │
                                    │   Object Storage     │
                                    │   + CDN (global)     │
                                    │   media.tourbangladesh│
                                    └──────────────────────┘

                                    ┌──────────────────────┐
                                    │   UptimeRobot        │
                                    │   (external ping)    │
                                    │   alerts via email   │
                                    └──────────────────────┘
```

---

## VPS Resource Allocation (Hetzner CX21: 2 vCPU, 4GB RAM, 40GB SSD)

| Service | RAM Allocation | Notes |
|---------|---------------|-------|
| Nginx | ~20MB | Reverse proxy |
| Zitadel | ~150MB | Auth server |
| FastAPI | ~150MB | 2 workers (uvicorn) |
| PostgreSQL | ~512MB | Shared with Zitadel DB |
| Redis | ~50MB | Volatile cache only |
| Grafana | ~150MB | Monitoring UI |
| Loki | ~150MB | Log aggregation |
| OS + Docker | ~500MB | System overhead |
| **Total** | **~1.7GB** | ~2.3GB headroom |

---

## Component Details

### 1. Frontend — Next.js 14 on Vercel

- App Router with Server Components for public pages (SEO + fast TTFB)
- Client Components only where interactivity is needed (map, forms, social buttons)
- `next-auth` v5 handles Zitadel OIDC token lifecycle
- TanStack Query for API data fetching, caching, and infinite scroll
- Leaflet (dynamic import, `ssr: false`) for map rendering
- Images served from Cloudflare R2 CDN (`media.tourbangladesh.app`)

**Rendering strategy per page:**

| Page | Strategy | Reason |
|------|----------|--------|
| Homepage | SSR | Fresh feed, SEO |
| Experience detail | SSR + ISR (60s) | SEO, OG tags, revalidate on update |
| User profile | SSR | Fresh follower counts |
| Search / Explore | CSR | Interactive filters |
| Create / Edit | CSR | Auth-only, no SEO needed |
| Dashboard / Feed | CSR | Personalised, auth-only |

---

### 2. Auth — Zitadel (Self-hosted)

- Runs in Docker alongside other services
- Uses PostgreSQL as its database (separate schema from app tables)
- Configured with:
  - Email + password login
  - Google as external identity provider
  - PKCE flow for `tour-web` public client
- next-auth `ZitadelProvider` handles frontend token management
- FastAPI validates JWT via Zitadel's JWKS endpoint
- Access token TTL: 15 minutes | Refresh token TTL: 7 days

**JWT Validation Flow (FastAPI):**
```
Request → Extract Bearer token
       → Fetch JWKS from Zitadel (cached, TTL 1hr)
       → Verify RS256 signature
       → Check exp, iss, aud claims
       → Extract sub (user ID) + email
       → Look up / provision user in DB
       → Inject user into request context
```

---

### 3. Backend — FastAPI

- Async Python with `asyncpg` database driver
- 2 Uvicorn workers (sufficient for <1,000 users)
- Layered architecture: routers → services → models
- Pydantic v2 schemas for all inputs/outputs
- Alembic for database migrations

**Request lifecycle:**
```
Nginx → FastAPI → Dependency injection (JWT validation, DB session)
              → Router → Service → SQLAlchemy query → PostgreSQL
                                 → Redis (cache read/write)
                                 → R2 (presigned URL generation)
              → Pydantic response serialization → JSON
```

**Caching strategy (Redis):**

| Cache Key | TTL | Invalidation |
|-----------|-----|-------------|
| `feed:latest:{page}` | 60s | On new experience published |
| `search:{hash}` | 120s | TTL expiry only |
| `user:{username}:profile` | 300s | On profile update |
| `experience:{id}` | 300s | On experience update/delete |
| `explore:bounds:{hash}` | 60s | TTL expiry only |

---

### 4. Database — PostgreSQL 16 + PostGIS

- Single PostgreSQL instance shared between app and Zitadel (separate schemas)
- PostGIS for geo queries (`ST_DWithin`, `ST_MakeEnvelope`, `ST_AsGeoJSON`)
- `pg_trgm` for fuzzy text matching
- `tsvector` + `tsquery` for full-text search (English)
- Alembic handles all schema migrations

**Backup strategy:**
```
Daily: pg_dump → compressed .sql.gz → upload to R2 (separate bucket)
Retention: 7 daily backups
Restore: download from R2 → pg_restore
```

---

### 5. Image Processing Pipeline

```
User selects photo (browser)
       │
       ▼
Frontend validates: MIME type, size ≤ 10MB
       │
       ▼
POST /upload/presigned-url  (FastAPI)
       │
       ▼
FastAPI returns presigned PUT URL (10min expiry)
       │
       ▼
Frontend PUT → R2 (original upload, temp key)
       │
       ▼
POST /experiences (with photo keys)
       │
       ▼
FastAPI background task:
  ├── Download original from R2
  ├── Pillow resize:
  │     full:      1200px wide (WebP, quality 85)
  │     thumbnail: 400px wide  (WebP, quality 80)
  ├── Upload full + thumbnail to R2 (final keys)
  └── Delete original temp file
       │
       ▼
Photo record saved with:
  url:           https://media.tourbangladesh.app/experiences/{id}/full.webp
  thumbnail_url: https://media.tourbangladesh.app/experiences/{id}/thumb.webp
```

---

### 6. Nginx — Reverse Proxy

Single Nginx instance routes all traffic on the VPS:

```nginx
# API
location /api/ {
    proxy_pass http://localhost:8000;
}

# Zitadel auth
location /auth/ {
    proxy_pass http://localhost:8081;
}

# Grafana (restricted)
location /grafana/ {
    proxy_pass http://localhost:3001;
    auth_basic "Monitoring";
}
```

- SSL via Let's Encrypt (Certbot auto-renewal)
- Gzip compression enabled
- Rate limiting: 30 req/s per IP on `/api/`

---

### 7. Monitoring Stack

**Loki** — log aggregation
- FastAPI logs structured as JSON (via `structlog`)
- Docker logging driver ships logs to Loki
- Log levels: INFO for requests, ERROR for exceptions, WARNING for auth failures

**Grafana** — dashboards
- Connected to Loki as data source
- Dashboards:
  - API request rate + error rate
  - Slow query log (>200ms)
  - Auth failures
  - Container health

**UptimeRobot** — external uptime monitoring
- Pings `https://api.tourbangladesh.app/health` every 5 minutes
- Email alert if down for 2 consecutive checks

---

### 8. CI/CD Pipeline (GitHub Actions)

```yaml
# Trigger: push to main
on:
  push:
    branches: [main]

jobs:
  test:
    # Run pytest (backend) + jest (frontend)

  deploy-frontend:
    # Vercel auto-deploys from GitHub (no action needed)

  deploy-backend:
    needs: test
    steps:
      - SSH into Hetzner VPS
      - git pull origin main
      - docker compose pull
      - docker compose up -d --build backend
      - docker compose exec backend alembic upgrade head
      - Health check: curl /health until 200
```

**Deploy flow:**
```
git push origin main
       │
       ├── GitHub Actions: run tests
       │          │ pass
       │          ▼
       │   SSH → VPS → git pull → docker compose up -d --build
       │          │
       │          ▼
       │   alembic upgrade head (zero-downtime migrations)
       │          │
       │          ▼
       │   health check passes → deploy complete
       │
       └── Vercel: auto-detects frontend changes → deploy to CDN
```

---

### 9. Security Architecture

| Layer | Mechanism |
|-------|-----------|
| Transport | HTTPS everywhere (Let's Encrypt) |
| Auth | JWT RS256, validated on every request |
| Secrets | `.env` files, never in git; GitHub Actions secrets for CI |
| File uploads | Presigned URLs only, MIME + size validated server-side |
| Rate limiting | Nginx (IP-level) + FastAPI middleware (user-level) |
| CORS | FastAPI allows only frontend origin |
| R2 bucket | No public write; read-only public access for CDN |
| Database | Not exposed outside Docker network |
| Redis | Not exposed outside Docker network |
| Zitadel | Only Nginx-proxied `/auth/` exposed publicly |
| Grafana | HTTP basic auth + Nginx, not publicly indexed |

---

### 10. Docker Compose Service Graph

```
infra/docker-compose.yml

services:
  nginx          → exposes :80/:443
  zitadel        → depends on: postgres
  backend        → depends on: postgres, redis, zitadel
  postgres       → (base, no deps)
  redis          → (base, no deps)
  grafana        → depends on: loki
  loki           → (base, no deps)

networks:
  internal       → postgres, redis, backend, zitadel, loki, grafana
  public         → nginx only

volumes:
  postgres_data
  redis_data
  loki_data
  grafana_data
  zitadel_data
```

---

### 11. Data Flow Diagrams

#### Create Experience
```
User (mobile)
  │ fill form + select photos
  ▼
Next.js (client)
  │ POST /upload/presigned-url × N photos
  ▼
FastAPI → R2 presigned PUT URLs
  │
  ▼
Next.js → PUT photos directly to R2 (parallel)
  │ all uploads complete
  ▼
Next.js → POST /experiences { ...data, photo_keys[] }
  │
  ▼
FastAPI
  ├── validate JWT
  ├── validate input (Pydantic)
  ├── insert experience + photos to PostgreSQL
  ├── enqueue background task: process images (Pillow → R2)
  └── invalidate Redis cache keys: feed:latest:*
  │
  ▼
Response: { experience_id, slug }
  │
  ▼
Next.js → redirect to /experiences/{slug}
```

#### Browse Homepage Feed
```
User (mobile) → GET /
  │
  ▼
Next.js (SSR, server component)
  │ GET /experiences?page=1&limit=20
  ▼
FastAPI
  ├── check Redis: feed:latest:1
  │   HIT  → return cached JSON (< 5ms)
  │   MISS ↓
  ├── PostgreSQL query (paginated, ordered by created_at DESC)
  ├── write to Redis (TTL 60s)
  └── return JSON
  │
  ▼
Next.js renders HTML with experience cards
  │
  ▼
Photos load from Cloudflare R2 CDN (cached at edge)
```

#### Search
```
User types query → debounce 300ms
  │
  ▼
GET /experiences/search?q=sundarban&division=Khulna&rating=4
  │
  ▼
FastAPI
  ├── check Redis: search:{hash_of_params}
  │   HIT  → return cached (TTL 120s)
  │   MISS ↓
  ├── PostgreSQL:
  │   WHERE search_vector @@ plainto_tsquery('english', 'sundarban')
  │   AND division = 'Khulna'
  │   AND rating >= 4
  │   ORDER BY ts_rank(search_vector, query) DESC
  └── cache + return
```

---

## Infrastructure Costs (Monthly Estimate)

| Service | Cost |
|---------|------|
| Hetzner CX21 VPS | €4.15/mo |
| Cloudflare R2 (10GB storage + 1M ops) | ~$0.15/mo |
| Vercel (Hobby tier) | Free |
| UptimeRobot | Free |
| Let's Encrypt SSL | Free |
| **Total** | **~€5/mo** |

---

## Scalability Path (when needed)

When the app grows beyond <1,000 users, the upgrade path is:

1. **VPS upgrade**: Hetzner CX31 (4 vCPU, 8GB) — just resize, no architecture change
2. **Read replicas**: Add PostgreSQL read replica for heavy read queries
3. **FastAPI workers**: Increase Uvicorn worker count or add second container
4. **Redis**: Already in place — just increase cache TTLs and add more keys
5. **Separate services**: Move Zitadel and monitoring to their own small VPS
6. **CDN**: Already on Cloudflare — no change needed
