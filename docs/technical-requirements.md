# Technical Requirements — Tour Bangladesh

## 1. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client (Browser)                         │
│                   Next.js 14 — Vercel                           │
└───────────┬─────────────────────┬───────────────────────────────┘
            │ OIDC / OAuth2        │ REST API (Bearer JWT)
            ▼                     ▼
┌───────────────────┐   ┌──────────────────────┐
│     Keycloak      │   │   FastAPI (Python)    │
│  (Docker, VPS)    │   │   (Docker, VPS)       │
│  realm: tour-bd   │   │   port 8000           │
└───────────────────┘   └──────────┬───────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    ▼                             ▼
          ┌──────────────────┐       ┌────────────────────┐
          │   PostgreSQL 16  │       │  Cloudflare R2     │
          │   + PostGIS      │       │  (Object Storage)  │
          │   (Docker, VPS)  │       │  CDN delivery      │
          └──────────────────┘       └────────────────────┘
```

---

## 2. Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend framework | Next.js (App Router) | 14.x |
| Frontend language | TypeScript | 5.x |
| Frontend styling | Tailwind CSS | 3.x |
| Auth provider | Keycloak | 24.x |
| Frontend auth client | next-auth | v5 (beta) |
| Backend framework | FastAPI | 0.111.x |
| Backend language | Python | 3.12 |
| ORM | SQLModel (SQLAlchemy 2.0 core) | latest |
| Database | PostgreSQL | 16 |
| Geo extension | PostGIS | 3.4 |
| DB migrations | Alembic | latest |
| Object storage | Cloudflare R2 | — |
| R2 SDK | boto3 (S3-compatible) | latest |
| Maps | Leaflet + react-leaflet | 1.9.x / 4.x |
| Frontend data fetching | TanStack Query (react-query) | v5 |
| Forms | react-hook-form + zod | latest |
| Container runtime | Docker + Docker Compose | 24.x |
| Frontend deployment | Vercel | — |
| Backend deployment | VPS (Docker Compose) | — |

---

## 3. Repository Structure

```
Tour/                                   # Monorepo root
├── frontend/                           # Next.js application
│   ├── src/
│   │   ├── app/
│   │   │   ├── (public)/               # Guest-accessible pages
│   │   │   │   ├── page.tsx            # Homepage / feed
│   │   │   │   ├── explore/page.tsx    # Map explore
│   │   │   │   ├── experiences/[id]/page.tsx
│   │   │   │   ├── users/[username]/page.tsx
│   │   │   │   └── search/page.tsx
│   │   │   ├── (auth)/                 # Auth-required pages
│   │   │   │   ├── experiences/new/page.tsx
│   │   │   │   ├── experiences/[id]/edit/page.tsx
│   │   │   │   ├── profile/page.tsx
│   │   │   │   └── settings/page.tsx
│   │   │   ├── api/auth/[...nextauth]/route.ts
│   │   │   └── layout.tsx
│   │   ├── components/
│   │   │   ├── ui/                     # Button, Input, Card, Badge, Skeleton
│   │   │   ├── experience/             # ExperienceCard, ExperienceForm, PhotoGallery
│   │   │   ├── map/                    # MapView, RouteEditor, ExploreMap
│   │   │   ├── social/                 # LikeButton, FollowButton, CommentSection
│   │   │   ├── layout/                 # BottomNav, Header, MobileShell
│   │   │   └── search/                 # SearchBar, FilterPanel
│   │   ├── hooks/
│   │   │   ├── use-auth.ts
│   │   │   ├── use-api.ts
│   │   │   ├── use-infinite-scroll.ts
│   │   │   └── use-debounce.ts
│   │   ├── lib/
│   │   │   ├── auth.ts                 # next-auth config (Keycloak provider)
│   │   │   ├── api-client.ts           # Axios/fetch wrapper with auth headers
│   │   │   ├── utils.ts
│   │   │   └── constants.ts
│   │   └── types/
│   │       ├── experience.ts
│   │       └── user.ts
│   ├── next.config.js
│   ├── tailwind.config.ts
│   ├── tsconfig.json
│   └── package.json
│
├── backend/                            # FastAPI application
│   ├── app/
│   │   ├── main.py                     # App factory, router registration, CORS
│   │   ├── config.py                   # pydantic-settings: env vars
│   │   ├── database.py                 # Async SQLAlchemy engine + session factory
│   │   ├── dependencies.py             # get_db, get_current_user, get_optional_user
│   │   ├── routers/
│   │   │   ├── auth.py                 # GET /auth/me
│   │   │   ├── experiences.py          # CRUD + list + search
│   │   │   ├── users.py                # Profile, follow/unfollow
│   │   │   ├── social.py               # Likes, comments
│   │   │   ├── upload.py               # Presigned URL generation
│   │   │   ├── explore.py              # Geo/map queries
│   │   │   └── health.py
│   │   ├── models/                     # SQLModel ORM models
│   │   │   ├── user.py
│   │   │   ├── experience.py
│   │   │   ├── photo.py
│   │   │   ├── social.py               # Like, Comment, Follow
│   │   │   └── tag.py
│   │   ├── schemas/                    # Pydantic request/response schemas
│   │   │   ├── experience.py
│   │   │   ├── user.py
│   │   │   ├── social.py
│   │   │   └── upload.py
│   │   ├── services/                   # Business logic
│   │   │   ├── experience_service.py
│   │   │   ├── user_service.py
│   │   │   ├── social_service.py
│   │   │   ├── search_service.py
│   │   │   ├── geo_service.py
│   │   │   └── storage_service.py
│   │   └── utils/
│   │       ├── jwt.py                  # JWKS fetch + token validation
│   │       └── pagination.py
│   ├── migrations/                     # Alembic
│   │   ├── env.py
│   │   └── versions/
│   ├── tests/
│   │   ├── conftest.py
│   │   ├── test_experiences.py
│   │   ├── test_users.py
│   │   └── test_social.py
│   ├── alembic.ini
│   ├── pyproject.toml
│   ├── Dockerfile
│   └── .env.example
│
├── infra/
│   ├── docker-compose.yml              # Production
│   ├── docker-compose.dev.yml          # Dev overrides (hot reload)
│   ├── keycloak/
│   │   └── realm-export.json           # Tour Bangladesh realm config
│   ├── postgres/
│   │   └── init.sql                    # CREATE EXTENSION postgis;
│   └── nginx/
│       └── nginx.conf                  # Reverse proxy (production)
│
└── docs/
    ├── non-technical-requirements.md
    ├── technical-requirements.md
    ├── architecture.md
    └── api-spec.md
```

---

## 4. Database Schema

### Extensions
```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

### Tables

#### `users`
```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    keycloak_id     VARCHAR(255) UNIQUE NOT NULL,
    username        VARCHAR(50) UNIQUE NOT NULL,
    display_name    VARCHAR(100) NOT NULL,
    bio             TEXT,
    avatar_url      VARCHAR(500),
    division        VARCHAR(50),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `experiences`
```sql
CREATE TABLE experiences (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    author_id           UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title               VARCHAR(200) NOT NULL,
    description         TEXT NOT NULL,
    category            VARCHAR(50) NOT NULL DEFAULT 'general',
    rating              SMALLINT CHECK (rating BETWEEN 1 AND 5),
    estimated_cost_bdt  INTEGER,
    travel_date_start   DATE,
    travel_date_end     DATE,
    travel_tips         TEXT,
    division            VARCHAR(50) NOT NULL,
    district            VARCHAR(50) NOT NULL,
    location            GEOGRAPHY(POINT, 4326),
    route               GEOGRAPHY(LINESTRING, 4326),
    search_vector       TSVECTOR,
    is_published        BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);
```

#### `photos`
```sql
CREATE TABLE photos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experience_id   UUID NOT NULL REFERENCES experiences(id) ON DELETE CASCADE,
    url             VARCHAR(500) NOT NULL,
    thumbnail_url   VARCHAR(500),
    caption         VARCHAR(200),
    sort_order      SMALLINT DEFAULT 0,
    width           INTEGER,
    height          INTEGER,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `map_pins`
```sql
CREATE TABLE map_pins (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experience_id   UUID NOT NULL REFERENCES experiences(id) ON DELETE CASCADE,
    label           VARCHAR(100),
    description     VARCHAR(300),
    location        GEOGRAPHY(POINT, 4326) NOT NULL,
    sort_order      SMALLINT DEFAULT 0
);
```

#### `tags` + `experience_tags`
```sql
CREATE TABLE tags (
    id      SERIAL PRIMARY KEY,
    name    VARCHAR(50) UNIQUE NOT NULL,
    slug    VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE experience_tags (
    experience_id   UUID REFERENCES experiences(id) ON DELETE CASCADE,
    tag_id          INTEGER REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (experience_id, tag_id)
);
```

#### `follows`
```sql
CREATE TABLE follows (
    follower_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    following_id    UUID REFERENCES users(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (follower_id, following_id),
    CHECK (follower_id != following_id)
);
```

#### `likes`
```sql
CREATE TABLE likes (
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    experience_id   UUID REFERENCES experiences(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, experience_id)
);
```

#### `comments`
```sql
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experience_id   UUID REFERENCES experiences(id) ON DELETE CASCADE,
    author_id       UUID REFERENCES users(id) ON DELETE CASCADE,
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### Indexes
```sql
CREATE INDEX idx_experiences_author    ON experiences(author_id);
CREATE INDEX idx_experiences_division  ON experiences(division);
CREATE INDEX idx_experiences_district  ON experiences(district);
CREATE INDEX idx_experiences_location  ON experiences USING GIST(location);
CREATE INDEX idx_experiences_route     ON experiences USING GIST(route);
CREATE INDEX idx_experiences_search    ON experiences USING GIN(search_vector);
CREATE INDEX idx_experiences_created   ON experiences(created_at DESC);
CREATE INDEX idx_photos_experience     ON photos(experience_id, sort_order);
CREATE INDEX idx_comments_experience   ON comments(experience_id, created_at DESC);
CREATE INDEX idx_follows_following     ON follows(following_id);
CREATE INDEX idx_likes_experience      ON likes(experience_id);
```

### Trigger: Auto-update `search_vector`
```sql
CREATE OR REPLACE FUNCTION update_experience_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := to_tsvector('english',
        COALESCE(NEW.title, '') || ' ' ||
        COALESCE(NEW.description, '') || ' ' ||
        COALESCE(NEW.division, '') || ' ' ||
        COALESCE(NEW.district, '') || ' ' ||
        COALESCE(NEW.travel_tips, '')
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_experience_search_vector
    BEFORE INSERT OR UPDATE ON experiences
    FOR EACH ROW EXECUTE FUNCTION update_experience_search_vector();
```

---

## 5. API Design

### Base URL
- Development: `http://localhost:8000/api/v1`
- Production: `https://api.tourbangladesh.app/api/v1`

### Authentication
All protected endpoints require:
```
Authorization: Bearer <access_token>
```
Token is a JWT issued by Keycloak, validated via JWKS.

### Endpoints

#### Experiences
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/experiences` | No | List experiences (paginated, filterable) |
| `POST` | `/experiences` | Yes | Create experience |
| `GET` | `/experiences/{id}` | No | Get single experience |
| `PUT` | `/experiences/{id}` | Yes (owner) | Update experience |
| `DELETE` | `/experiences/{id}` | Yes (owner) | Delete experience |
| `GET` | `/experiences/search` | No | Full-text search |
| `GET` | `/experiences/feed` | Yes | Feed from followed users |

#### Users
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/users/{username}` | No | Get user profile |
| `GET` | `/users/{username}/experiences` | No | User's experiences |
| `PUT` | `/users/me` | Yes | Update own profile |
| `POST` | `/users/{id}/follow` | Yes | Follow user |
| `DELETE` | `/users/{id}/follow` | Yes | Unfollow user |

#### Social
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/experiences/{id}/like` | Yes | Like experience |
| `DELETE` | `/experiences/{id}/like` | Yes | Unlike experience |
| `GET` | `/experiences/{id}/comments` | No | List comments |
| `POST` | `/experiences/{id}/comments` | Yes | Add comment |
| `DELETE` | `/comments/{id}` | Yes (owner) | Delete comment |

#### Upload
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/upload/presigned-url` | Yes | Get R2 presigned PUT URL |

#### Explore (Geo)
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/explore/nearby` | No | Experiences within radius (`lat`, `lng`, `radius_km`) |
| `GET` | `/explore/bounds` | No | Experiences within map bounds (`ne_lat`, `ne_lng`, `sw_lat`, `sw_lng`) |

#### Auth
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/auth/me` | Yes | Get current user info |

### Pagination
All list endpoints support:
```
GET /experiences?page=1&limit=20
```
Response envelope:
```json
{
  "data": [...],
  "total": 120,
  "page": 1,
  "limit": 20,
  "has_next": true
}
```

---

## 6. Authentication Flow (Keycloak + next-auth + FastAPI)

```
1.  User clicks "Login" in Next.js
2.  next-auth redirects → Keycloak login page
3.  User authenticates (email/password or Google)
4.  Keycloak → next-auth callback with auth code
5.  next-auth exchanges code for tokens (PKCE)
6.  Tokens stored in encrypted HTTP-only session cookie
7.  Frontend API calls: reads access_token from session → sets Authorization header
8.  FastAPI: extracts Bearer token → fetches Keycloak JWKS → validates RS256 signature
9.  FastAPI: extracts `sub` claim → looks up or creates user in DB
10. On expiry: next-auth jwt callback detects expiry → calls Keycloak token endpoint → refreshes
```

### Keycloak Configuration
- Realm: `tour-bangladesh`
- Client ID: `tour-web` (public client, PKCE enabled)
- Valid redirect URIs: `http://localhost:3000/*`, `https://tourbangladesh.app/*`
- Identity provider: Google (clientId + secret from Google Cloud Console)
- Token lifespan: access token 15min, refresh token 7 days

### next-auth Config
```typescript
// src/lib/auth.ts
providers: [KeycloakProvider({ issuer, clientId, clientSecret })]
callbacks: {
  jwt: async ({ token, account }) => {
    // store access_token, refresh_token, expires_at
    // refresh access_token when expired
  },
  session: async ({ session, token }) => {
    // expose access_token on session for API calls
  }
}
```

---

## 7. Cloudflare R2 Upload Flow

```
1.  User selects photo(s) in ExperienceForm
2.  Frontend: POST /upload/presigned-url
    Body: { filename: "beach.jpg", content_type: "image/jpeg", size_bytes: 2048000 }
3.  Backend validates: authenticated, file type allowed, size ≤ 10MB
4.  Backend generates R2 key: experiences/{user_id}/{uuid}.jpg
5.  Backend: boto3 s3 client (endpoint_url=R2_ENDPOINT)
    → generate_presigned_url("put_object", Params={...}, ExpiresIn=600)
6.  Backend returns: { upload_url, cdn_url, key }
7.  Frontend: PUT <upload_url> with file binary + Content-Type header
8.  Frontend: on 200 → stores cdn_url in form state
9.  On experience submit → sends cdn_urls[] to POST /experiences
```

### R2 Configuration
| Setting | Value |
|---------|-------|
| Bucket name | `tour-bangladesh-media` |
| Public access | Enabled |
| Custom domain | `media.tourbangladesh.app` |
| Allowed MIME types | `image/jpeg`, `image/png`, `image/webp` |
| Max file size | 10 MB |
| Presigned URL expiry | 10 minutes |
| Key pattern | `experiences/{user_id}/{uuid}.{ext}` |

---

## 8. Infrastructure

### Docker Compose Services

| Service | Image | Port | Notes |
|---------|-------|------|-------|
| `postgres` | `postgis/postgis:16-3.4` | 5432 | PostGIS enabled via init.sql |
| `keycloak` | `quay.io/keycloak/keycloak:24.0` | 8080 | Imports realm-export.json on start |
| `backend` | Custom (FastAPI Dockerfile) | 8000 | Depends on postgres + keycloak |
| `nginx` | `nginx:alpine` | 80/443 | Reverse proxy (production only) |

### Environment Variables

#### Backend (`.env`)
```
DATABASE_URL=postgresql+asyncpg://user:pass@postgres:5432/tour_bangladesh
KEYCLOAK_ISSUER=http://keycloak:8080/realms/tour-bangladesh
KEYCLOAK_JWKS_URL=http://keycloak:8080/realms/tour-bangladesh/protocol/openid-connect/certs
R2_ENDPOINT=https://<account_id>.r2.cloudflarestorage.com
R2_ACCESS_KEY=<r2_access_key>
R2_SECRET_KEY=<r2_secret_key>
R2_BUCKET=tour-bangladesh-media
R2_PUBLIC_URL=https://media.tourbangladesh.app
CORS_ORIGINS=http://localhost:3000,https://tourbangladesh.app
```

#### Frontend (`.env.local`)
```
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=<random_secret>
KEYCLOAK_ID=tour-web
KEYCLOAK_SECRET=<client_secret_if_confidential>
KEYCLOAK_ISSUER=http://localhost:8080/realms/tour-bangladesh
NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
NEXT_PUBLIC_R2_PUBLIC_URL=https://media.tourbangladesh.app
```

---

## 9. Security Requirements

- [ ] All API endpoints validate JWT signature (RS256) against Keycloak JWKS
- [ ] HTTPS enforced in production for all services
- [ ] Passwords never handled by the application (Keycloak only)
- [ ] R2 presigned URLs scoped to authenticated users only
- [ ] File upload validation: MIME type check, 10MB size limit
- [ ] CORS configured to allow only known frontend origins
- [ ] Rate limiting on all write endpoints (FastAPI middleware)
- [ ] SQL injection prevention via ORM parameterized queries
- [ ] Input validation via Pydantic schemas on all API inputs
- [ ] No secrets in source code (`.env` files gitignored)
- [ ] R2 bucket: no public write access (only via presigned URLs)

---

## 10. Non-Functional Technical Requirements

| Requirement | Target |
|-------------|--------|
| API response time (p95) | < 300ms |
| Homepage load time | < 3s on 3G |
| Image delivery | Via Cloudflare CDN (cached globally) |
| Database query time | < 100ms for paginated lists |
| Geo query time | < 200ms with GIST index |
| Full-text search | < 500ms |
| Uptime | 99.5% |
| Max photo size | 10 MB per photo |
| Max photos per experience | 10 |
| API pagination | Default 20 items, max 100 |
| Token refresh | Automatic via next-auth |
| Database backups | Daily (production) |

---

## 11. Testing Requirements

### Backend (pytest)
- Unit tests for all service layer functions
- Integration tests for all API endpoints (using `httpx` + test database)
- JWT validation tests (valid, expired, tampered tokens)
- Geo query tests with sample PostGIS data
- Minimum coverage: **80%**

### Frontend (Jest + React Testing Library)
- Unit tests for utility functions and hooks
- Component tests for forms, cards, and auth flows
- Minimum coverage: **70%**

### End-to-End (Playwright)
- User registration and login flow
- Create experience with photo upload
- Search and filter experiences
- Like, comment, follow flows

---

## 12. Development Setup (Quick Start)

```bash
# 1. Start backend infrastructure
cd infra
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# 2. Run database migrations
cd backend
alembic upgrade head

# 3. Start FastAPI dev server
uvicorn app.main:app --reload --port 8000

# 4. Start Next.js dev server
cd frontend
npm install
npm run dev
```

API docs available at: `http://localhost:8000/docs`
Keycloak admin at: `http://localhost:8080/admin` (admin / admin)
