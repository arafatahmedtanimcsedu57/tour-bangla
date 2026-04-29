# Tour Bangladesh — Claude Project Configuration

## Project Overview

A mobile-first travel experience sharing platform for Bangladesh.
Users post geo-tagged experiences (photos, maps, routes, ratings, costs).
Social features: follow, like, comment.

**Monorepo layout:**
```
Tour/
├── frontend/    # Next.js 14 + TypeScript + Tailwind CSS
├── backend/     # FastAPI + Python 3.12
├── infra/       # Docker Compose (Zitadel + PostgreSQL + Redis + Nginx)
└── docs/        # Architecture, requirements, API spec
```

**Full stack:** Next.js → Zitadel (OIDC) → FastAPI → PostgreSQL+PostGIS + Redis + Cloudflare R2

---

## Skill Map

Use the correct skill for each type of task. Do not skip these.

### Python / FastAPI (backend/)
| Task | Skill / Agent |
|------|--------------|
| Writing new Python code | `/python-patterns` |
| Writing or fixing tests | `/python-testing` |
| Reviewing Python code | `python-reviewer` agent |
| Designing API endpoints | `/api-design` |
| Backend architecture decisions | `/backend-patterns` |

### TypeScript / Next.js (frontend/)
| Task | Skill / Agent |
|------|--------------|
| Writing frontend code | `/frontend-patterns` |
| UI design and components | `/frontend-design` |
| Reviewing TypeScript/React code | `typescript-reviewer` agent |

### Database (PostgreSQL + PostGIS)
| Task | Skill / Agent |
|------|--------------|
| Writing SQL / migrations | `database-reviewer` agent |
| Query optimization | `database-reviewer` agent |
| Schema design changes | `database-reviewer` agent |

### Testing
| Task | Skill / Agent |
|------|--------------|
| New feature (any layer) | `/tdd-workflow` first, then implement |
| E2E tests (Playwright) | `/e2e-testing` |
| Running E2E tests | `e2e-runner` agent |

### Code Quality & Security
| Task | Skill / Agent |
|------|--------------|
| After writing any code | `code-reviewer` agent |
| Before committing | `security-reviewer` agent |
| Removing dead code | `refactor-cleaner` agent |

### Planning & Architecture
| Task | Skill / Agent |
|------|--------------|
| New feature planning | `planner` agent |
| Architectural decisions | `architect` agent |
| Build errors | `build-error-resolver` agent |

---

## Project-Specific Conventions

### Python (FastAPI)
- Use `SQLModel` for ORM models (combines SQLAlchemy + Pydantic)
- All database operations must be **async** (`asyncpg` driver)
- Use `pydantic-settings` for config/env vars (`app/config.py`)
- Structure: `routers/` → `services/` → `models/` — never put DB queries in routers
- Use `structlog` for structured JSON logging
- Validate all inputs via Pydantic schemas in `schemas/`
- Background tasks (e.g., image processing) use FastAPI `BackgroundTasks`

### TypeScript (Next.js)
- App Router only — no Pages Router
- Server Components for public/SEO pages; Client Components only where needed
- All API calls go through `lib/api-client.ts` (sets Bearer token from next-auth session)
- Map components (`components/map/`) must use `dynamic(() => import(...), { ssr: false })`
- Use TanStack Query for all client-side data fetching
- Forms: `react-hook-form` + `zod` validation schemas
- No `any` types — use generated types from `types/`

### Database
- All migrations via Alembic — never edit tables manually
- New columns must be nullable or have a default (zero-downtime migrations)
- PostGIS columns use `GEOGRAPHY` type (SRID 4326)
- Always add indexes for foreign keys and frequently filtered columns

### Images
- Upload flow: presigned URL from `/upload/presigned-url` → PUT to R2 → submit CDN URL with form
- FastAPI resizes on upload: full (1200px WebP q85) + thumbnail (400px WebP q80) via Pillow
- Never store images in the database or on the VPS disk

### Authentication
- Auth is handled by **Zitadel** (OIDC) via `next-auth` in the frontend
- FastAPI validates JWT on every protected endpoint via JWKS (no session, stateless)
- User provisioning: first login creates a `users` row using `sub` claim as `keycloak_id`
- Never store passwords or tokens in the application database

### Caching (Redis)
- Cache keys must follow pattern: `{resource}:{identifier}:{page}`
- Always invalidate relevant cache keys after writes
- TTLs: feed 60s · search 120s · profiles 300s · experience detail 300s

### Git
- Branch naming: `feat/`, `fix/`, `chore/`, `docs/`
- Commits follow Conventional Commits: `feat:`, `fix:`, `refactor:`, `chore:`
- Never commit `.env` files — use `.env.example` with placeholder values

---

## Environment Setup

```bash
# Start all backend services
cd infra && docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# Run migrations
cd backend && alembic upgrade head

# Start FastAPI (hot reload)
cd backend && uvicorn app.main:app --reload --port 8000

# Start Next.js (hot reload)
cd frontend && npm run dev
```

**Local URLs:**
- Frontend: http://localhost:3000
- FastAPI docs: http://localhost:8000/docs
- Zitadel admin: http://localhost:8081
- Grafana: http://localhost:3001

---

## Project Status & Decisions

> Keep this section updated as the project evolves. This is what any Claude session
> on any device needs to get up to speed instantly.

### What's Been Completed
- [x] `docs/non-technical-requirements.md`
- [x] `docs/technical-requirements.md`
- [x] `docs/architecture.md`
- [x] `CLAUDE.md` (this file)
- [x] Figma design system tokens (colors, spacing, radius, typography)
- [x] `frontend/design/homepage.html` — homepage/feed UI prototype
- [x] `frontend/design/experience-detail.html` — experience detail UI prototype
- [ ] Project scaffolding (Next.js + FastAPI)
- [ ] Docker Compose + Zitadel setup
- [ ] Database migrations (Alembic)
- [ ] FastAPI backend implementation
- [ ] Next.js frontend pages

### Key Architecture Decisions
| Decision | Choice | Why |
|----------|--------|-----|
| Auth | **Zitadel** (not Keycloak) | ~100MB RAM vs 512MB — fits Hetzner CX21 |
| Hosting | **Hetzner CX21** (~€4/mo) | <1,000 users, single VPS, Docker Compose sufficient |
| Image processing | **FastAPI + Pillow** | Free; resize to WebP on upload, no Cloudflare Images cost |
| Maps | **Leaflet + OpenStreetMap** | Free, no API key, good Bangladesh coverage |
| Caching | **Redis** (minimal) | Only homepage feed + search results |
| Search | **PostgreSQL tsvector** | English only; no external search engine needed |
| Design workflow | **HTML prototypes first** | Figma Starter plan limits (3 pages, limited MCP calls) |
| Display font | **Fraunces** | Warm serif, editorial travel feel |
| UI font | **Plus Jakarta Sans** | Clean, readable on mobile |

### Design System (Figma)
- File: https://www.figma.com/design/DYe9nqWdKyWidBcKMWfoPP
- Tokens created: 36 primitive colors, 35 semantic colors, 12 spacing, 7 radius, 17 text styles
- Primary: `#2D6A4F` (Forest Green) · Accent: `#E07A5F` (Terracotta) · Bg: `#FAFAF8`

### UI Pages Status
| Page | File | Status |
|------|------|--------|
| Homepage / Feed | `frontend/design/homepage.html` | ✅ Done |
| Experience Detail | `frontend/design/experience-detail.html` | ✅ Done |
| Create Experience | `frontend/design/create-experience.html` | ⏳ Pending |
| Explore / Search | `frontend/design/explore.html` | ⏳ Pending |
| User Profile | `frontend/design/profile.html` | ⏳ Pending |

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `backend/app/dependencies.py` | `get_current_user`, `get_optional_user`, `get_db` |
| `backend/app/utils/jwt.py` | Zitadel JWT validation via JWKS |
| `backend/app/services/storage_service.py` | R2 presigned URL generation |
| `backend/app/services/geo_service.py` | PostGIS spatial queries |
| `frontend/src/lib/auth.ts` | next-auth Zitadel provider config |
| `frontend/src/lib/api-client.ts` | Axios wrapper with Bearer token injection |
| `infra/docker-compose.yml` | All backend services |
| `infra/keycloak/realm-export.json` | Zitadel realm config (rename to zitadel) |
| `docs/architecture.md` | Full system architecture |
| `docs/technical-requirements.md` | Technical requirements |
| `docs/non-technical-requirements.md` | Product requirements |
