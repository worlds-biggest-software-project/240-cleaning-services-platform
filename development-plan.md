# Cleaning Services Platform — Phased Development Plan

> Project: 240-cleaning-services-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises the project's `research.md`, `features.md`, `standards.md`, and `data-model-suggestion-1.md` (Entity-Centric Normalized Relational) into an implementation roadmap. It targets a self-hostable, API-first, AI-native cleaning operations platform that spans residential and commercial use cases — closing the gap between residential-only tools (ZenMaid), commercial-only platforms (Swept, Janitorial Manager), and inspection-only apps (OrangeQC).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | The platform's differentiating features (route optimisation, computer-vision quality inspection, dynamic pricing, retention analytics, AI phone agent) are ML/LLM-heavy. Python has best-in-class libraries (OR-Tools, OpenCV, scikit-learn, pandas, the Anthropic/OpenAI SDKs, LangChain), and FastAPI gives modern async I/O for the API surface. |
| API framework | FastAPI 0.110+ | Native async support for streaming AI calls and concurrent third-party integrations; automatic OpenAPI 3.1 generation (aligns with `standards.md`); built-in Pydantic validation; excellent typing story. |
| ORM | SQLAlchemy 2.0 (async) + Alembic | The chosen data model (suggestion 1) has ~46 tables with rich relationships; SQLAlchemy is the de-facto Python ORM with mature async support and Alembic for migrations. |
| Database | PostgreSQL 16 + PostGIS 3.4 | Required by `data-model-suggestion-1.md`: UUIDs, JSONB (settings, tags), GIN indexes, RLS for multi-tenant isolation, partitioning for GPS breadcrumbs. PostGIS enables geofencing, proximity queries, and route stop optimisation. |
| Cache & broker | Redis 7 | Cache for session/rate-limiting, broker for Celery, pub/sub for live cleaner location streaming to the dashboard. |
| Task queue | Celery 5 with Redis broker | Required for: outbound SMS/email, recurring job materialisation from `schedule_rule`, route optimisation jobs, image-analysis pipelines, webhook delivery to third parties, QuickBooks sync, invoice generation. Celery beat handles cron-style scheduled tasks. |
| Object storage | S3-compatible (MinIO self-hosted, AWS S3 cloud) | Inspection photos, before/after job documentation, SDS PDFs, generated inspection report PDFs — all referenced by `storage_key` in the schema. MinIO keeps the self-host story zero-ops. |
| Real-time | WebSockets via FastAPI + Redis pub/sub | Cleaner location streaming, live job status updates on the dispatcher dashboard, push notifications for newly assigned jobs. |
| Web frontend | Next.js 14 (App Router) + TypeScript + Tailwind + shadcn/ui | Admin dashboard, client portal, embeddable booking widget. App Router supports server actions for forms; shadcn gives a consistent design system. |
| Mobile (cleaner app) | React Native 0.74 + Expo SDK 51 | iOS/Android parity is a stated goal (addresses ServiceM8 iOS-bias gap). Expo provides location, camera, offline storage (SQLite via expo-sqlite), push notifications, and background tasks in one stack. |
| Auth | OAuth 2.0 + OIDC via Authlib (server) + PKCE for mobile | Required by `standards.md` (RFC 6749, RFC 7636, OpenID Connect Core 1.0). Supports social login (Google, Apple) and SSO for enterprise. |
| Payments | Stripe (Stripe Connect for multi-tenant payouts) | Listed in MVP and identified as standard for category. Stripe Connect supports the franchise/multi-vendor model called out in features.md. |
| Accounting | QuickBooks Online API | Listed in MVP integrations; ranked top by buyers in research.md. |
| SMS / voice | Twilio | MVP for SMS reminders; foundation for the v1.1 AI phone agent (post-MVP). |
| Email | SendGrid (transactional) + DKIM/SPF/DMARC | Booking confirmations, invoices, inspection reports, review requests. |
| Mapping & routing | Google Maps Routes API + Route Optimization API | Cited in `standards.md`; provides driving times and the constraint-aware multi-stop optimiser the platform needs. Fallback: OpenRouteService for self-hosted deployments. |
| ML route optimisation | Google OR-Tools (CP-SAT / VRP solver) | Self-hostable, vendor-neutral route optimisation when Google Maps Route Optimization is not used; supports skill matching, time windows, multi-cleaner VRP. |
| AI / LLM | Anthropic Claude (default), with provider abstraction supporting OpenAI | Powers communication drafting, AI phone agent, photo quality analysis (via vision models), retention narrative generation. Abstracted behind an `LLMClient` interface so deployments can swap providers. |
| Computer vision | OpenCV + a vision-capable LLM (Claude Sonnet with image input) | Photo-based quality inspection (missed-area detection). OpenCV for preprocessing/cropping; vision LLM for semantic understanding ("is this surface clean?"). |
| i18n | Babel + gettext (.po files) | English + Spanish for MVP cleaner app per features.md; pluggable for additional locales. |
| PDF generation | WeasyPrint | Server-side rendering of inspection reports and invoice PDFs from HTML/CSS templates. |
| Background image processing | Pillow + boto3 streaming | Resize, EXIF strip, generate thumbnails on upload. |
| Containerisation | Docker + docker-compose (dev/self-host) + Helm chart (k8s, cloud) | Self-hostable is explicit in README.md; one-command bootstrap with docker-compose; Helm for the managed cloud offering. |
| CI/CD | GitHub Actions | Test, lint, build images, push to registry, deploy to staging on merge to `main`. |
| Testing | pytest + pytest-asyncio + httpx (API), Vitest + Playwright (web), Jest + Detox (mobile) | Standard for each stack; Playwright covers booking-widget e2e and admin dashboard. |
| Code quality | Ruff (lint + format), Mypy (type check), Bandit (security) | Single-tool linter/formatter via Ruff is fast and reproducible; Mypy strict on the API; Bandit catches OWASP-relevant issues per `standards.md`. |
| Package manager | uv (Python), pnpm (web + mobile) | Fast, lockfile-first, reproducible. |
| Observability | OpenTelemetry + Prometheus + Grafana + Sentry | Tracing for the async task pipeline; Sentry for application errors. |
| Secrets | Pydantic Settings + .env (dev), AWS Secrets Manager / HashiCorp Vault (prod) | Single source of truth via `Settings` class. |

### Project Structure

```
cleaning-platform/
├── pyproject.toml
├── uv.lock
├── README.md
├── Dockerfile
├── docker-compose.yml
├── docker-compose.override.yml          # dev-only overrides
├── .env.example
├── alembic.ini
├── deploy/
│   ├── helm/                            # Helm chart for k8s
│   └── terraform/                       # Optional cloud bootstrap
├── docs/
│   ├── api-reference.md                 # Generated from OpenAPI
│   ├── data-model.md
│   └── developer-guide.md
├── migrations/                          # Alembic
│   └── versions/
├── src/
│   └── cleaning_platform/
│       ├── __init__.py
│       ├── main.py                      # FastAPI app factory
│       ├── settings.py                  # Pydantic Settings
│       ├── db.py                        # Async engine, session factory
│       ├── cache.py                     # Redis client
│       ├── storage.py                   # S3/MinIO client
│       ├── auth/
│       │   ├── oauth.py
│       │   ├── jwt.py
│       │   ├── rbac.py
│       │   └── tenant.py                # Tenant resolution, RLS context
│       ├── models/                      # SQLAlchemy ORM (one file per category)
│       │   ├── base.py                  # TenantMixin, TimestampMixin
│       │   ├── identity.py              # Tenant, User, AuthProvider
│       │   ├── client.py                # Client, Property
│       │   ├── workforce.py             # Cleaner, Skill, Team, ...
│       │   ├── service.py               # ServiceType, ServiceAddon
│       │   ├── scheduling.py            # ScheduleRule, Job, JobAssignment
│       │   ├── tracking.py              # ClockEvent, GpsBreadcrumb
│       │   ├── inspection.py            # InspectionTemplate, Inspection, ...
│       │   ├── billing.py               # Invoice, Payment, Quote
│       │   ├── compliance.py            # ChemicalProduct, CimsElement, ...
│       │   ├── equipment.py
│       │   ├── communication.py         # CommunicationLog, ReviewRequest
│       │   ├── routing.py               # Route, RouteStop
│       │   ├── contract.py
│       │   └── audit.py                 # AuditLog (hybrid audit-trail)
│       ├── schemas/                     # Pydantic request/response models
│       │   └── ...                      # One file per domain
│       ├── api/                         # FastAPI routers
│       │   ├── v1/
│       │   │   ├── auth.py
│       │   │   ├── clients.py
│       │   │   ├── properties.py
│       │   │   ├── cleaners.py
│       │   │   ├── jobs.py
│       │   │   ├── schedule.py
│       │   │   ├── inspections.py
│       │   │   ├── invoices.py
│       │   │   ├── payments.py
│       │   │   ├── quotes.py
│       │   │   ├── chemicals.py
│       │   │   ├── routes.py
│       │   │   ├── webhooks.py
│       │   │   └── booking_widget.py    # Public, no-auth booking endpoints
│       │   └── public_api.py            # External API gateway
│       ├── services/                    # Business logic
│       │   ├── scheduling/
│       │   │   ├── rrule.py             # RFC 5545 expansion
│       │   │   └── materialiser.py
│       │   ├── routing/
│       │   │   ├── optimiser.py         # OR-Tools VRP
│       │   │   └── google_routes.py
│       │   ├── pricing/
│       │   │   └── engine.py
│       │   ├── billing/
│       │   │   ├── invoice_generator.py
│       │   │   └── stripe_gateway.py
│       │   ├── inspection/
│       │   │   ├── scoring.py
│       │   │   └── report.py
│       │   ├── compliance/
│       │   │   ├── cims.py
│       │   │   └── sds.py
│       │   ├── ai/
│       │   │   ├── client.py            # LLMClient abstraction
│       │   │   ├── prompts/             # Jinja templates
│       │   │   ├── vision.py            # Photo quality analysis
│       │   │   ├── communication.py     # Drafting reminders, summaries
│       │   │   ├── phone_agent.py       # Twilio + LLM
│       │   │   └── retention.py
│       │   ├── communication/
│       │   │   ├── sms.py               # Twilio
│       │   │   ├── email.py             # SendGrid
│       │   │   └── templates/           # Jinja message templates
│       │   ├── integrations/
│       │   │   ├── quickbooks.py
│       │   │   ├── stripe_connect.py
│       │   │   └── google_maps.py
│       │   └── reporting/
│       ├── tasks/                       # Celery tasks
│       │   ├── celery_app.py
│       │   ├── scheduling.py
│       │   ├── communication.py
│       │   ├── routing.py
│       │   ├── invoicing.py
│       │   ├── vision.py
│       │   └── webhooks.py
│       ├── webhooks/
│       │   ├── delivery.py              # Outbound to subscribers
│       │   ├── stripe.py                # Inbound from Stripe
│       │   └── quickbooks.py
│       ├── utils/
│       │   ├── pdf.py                   # WeasyPrint helpers
│       │   ├── geo.py                   # Haversine, geofencing
│       │   ├── pagination.py
│       │   └── rate_limit.py
│       ├── i18n/
│       │   └── locale/
│       │       ├── en/LC_MESSAGES/
│       │       └── es/LC_MESSAGES/
│       └── cli.py                       # Typer admin CLI
├── tests/
│   ├── conftest.py
│   ├── fixtures/
│   │   ├── photos/                      # Sample inspection photos
│   │   ├── sds/                         # Sample SDS PDFs
│   │   └── webhooks/                    # Sample inbound payloads
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── web/                                  # Next.js admin dashboard + client portal
│   ├── app/
│   │   ├── (admin)/
│   │   ├── (client-portal)/
│   │   └── (booking)/                   # Embeddable widget
│   ├── components/
│   ├── lib/                              # API client (generated from OpenAPI)
│   └── package.json
├── mobile/                               # Expo / React Native cleaner app
│   ├── app/
│   ├── src/
│   ├── locales/                          # en, es
│   └── package.json
└── .github/
    └── workflows/
        ├── api-ci.yml
        ├── web-ci.yml
        └── mobile-ci.yml
```

---

## Phase 1: Foundation & Multi-Tenant Skeleton

### Purpose
Establish the runtime — Docker stack, FastAPI app, async PostgreSQL, Alembic migrations, settings, logging, the core tenant/user/auth models, and a deployable health endpoint. After this phase, the platform boots locally, a developer can run `docker compose up`, and a tenant can be created via CLI.

### Tasks

#### 1.1 — Repository scaffolding and tooling

**What**: Initialise the monorepo with Python project, Ruff, Mypy, pytest, pre-commit hooks, and Dockerfile.

**Design**:
- `pyproject.toml` uses `uv` with build backend `hatchling`; declares Python 3.12; pins FastAPI, SQLAlchemy[asyncio], asyncpg, alembic, pydantic-settings, redis, celery, anthropic, openai, twilio, stripe, sendgrid-python, weasyprint, pillow, ortools, structlog.
- `Dockerfile` is a two-stage build: stage 1 installs deps with `uv pip install --system`; stage 2 copies source, runs as non-root `app` user, exposes port 8000.
- `docker-compose.yml` services: `api`, `worker` (celery), `beat` (celery beat), `web`, `postgres` (with PostGIS extension), `redis`, `minio`, `mailhog`.
- `pre-commit` runs `ruff check --fix`, `ruff format`, `mypy src/`.
- `.env.example` lists all required env vars with sane defaults.

**Testing**:
- `Unit: tests/test_smoke.py asserts import cleaning_platform succeeds`
- `Integration: docker compose up exits 0 within 60s for all services`
- `CI: ruff and mypy run clean on the empty scaffold`

#### 1.2 — Settings, logging, database engine

**What**: Centralised `Settings` (Pydantic), structured logging (structlog), and an async SQLAlchemy engine + session factory.

**Design**:
```python
# settings.py
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="CP_")
    environment: Literal["dev", "staging", "prod"] = "dev"
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: HttpUrl
    s3_bucket: str
    s3_access_key: SecretStr
    s3_secret_key: SecretStr
    secret_key: SecretStr                  # JWT signing
    stripe_secret_key: SecretStr | None = None
    twilio_sid: str | None = None
    twilio_token: SecretStr | None = None
    sendgrid_api_key: SecretStr | None = None
    anthropic_api_key: SecretStr | None = None
    google_maps_api_key: SecretStr | None = None
    quickbooks_client_id: str | None = None
    quickbooks_client_secret: SecretStr | None = None
    log_level: str = "INFO"
    cors_origins: list[str] = ["http://localhost:3000"]
```
- `db.py` exposes `create_engine() -> AsyncEngine` and `get_session() -> AsyncIterator[AsyncSession]` (FastAPI dependency).
- `logging.py` configures structlog with JSON output in prod, key=value in dev; injects `request_id`, `tenant_id`, `user_id` from contextvars.

**Testing**:
- `Unit: Settings loads from environment with no missing required vars given .env.example values`
- `Unit: Settings raises ValidationError when DATABASE_URL is missing`
- `Integration: get_session yields a working AsyncSession against the docker-compose postgres`
- `Unit: log record JSON includes request_id when contextvar set`

#### 1.3 — Alembic migrations + base mixins

**What**: Configure Alembic with async support; introduce `TenantMixin`, `TimestampMixin`, and `Base` declarative class.

**Design**:
- `alembic.ini` points to `migrations/`; `env.py` uses `asyncio.run(do_run_migrations())`.
- `models/base.py`:
  ```python
  class Base(DeclarativeBase): pass
  class TimestampMixin:
      created_at: Mapped[datetime] = mapped_column(server_default=func.now())
      updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
  class TenantMixin:
      tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenant.id"), index=True)
  ```
- Initial migration `0001_init.py` creates `pgcrypto` and `postgis` extensions plus the `tenant`, `user`, `auth_provider` tables (from `data-model-suggestion-1.md`).
- Postgres RLS policy template (applied later per-tenant table): `USING (tenant_id = current_setting('app.tenant_id')::uuid)`.

**Testing**:
- `Integration: alembic upgrade head succeeds on empty db, creates tenant/user/auth_provider tables`
- `Integration: alembic downgrade base then upgrade head is idempotent`
- `Unit: A model inheriting TenantMixin + TimestampMixin produces correct DDL`

#### 1.4 — Tenant, User, AuthProvider models + bootstrap CLI

**What**: SQLAlchemy models for `tenant`, `user`, `auth_provider`; Typer-based `cleaning-platform` CLI with `tenant create` and `user create-owner` commands.

**Design**:
- Models mirror SQL in `data-model-suggestion-1.md` lines 46-86.
- `User.role` enum: `owner | admin | manager | member | cleaner | client`.
- Password hashed with `argon2-cffi` (OWASP-recommended; `standards.md` cites NIST SP 800-63B).
- `cli.py`:
  ```bash
  cleaning-platform tenant create --name "Acme Cleaning" --slug acme --timezone "America/New_York"
  cleaning-platform user create-owner --tenant-slug acme --email owner@acme.com --password ...
  cleaning-platform db seed-demo --tenant-slug acme
  ```

**Testing**:
- `Unit: User.set_password() then verify_password() returns True; wrong password returns False`
- `Unit: Tenant slug uniqueness enforced (IntegrityError on duplicate)`
- `Integration: CLI tenant create persists row; user create-owner persists row with role=owner`
- `Fixture: tests/fixtures/seed_demo.py creates a tenant + owner + sample data for downstream tests`

#### 1.5 — FastAPI app factory + health endpoint + OpenAPI

**What**: `create_app()` factory wiring middleware (CORS, request-id, tenant resolution, error handlers); `/healthz` and `/readyz` endpoints; OpenAPI 3.1 spec exposed at `/openapi.json`.

**Design**:
- Middleware order: RequestId → CORS → TenantResolver → ExceptionHandler.
- `TenantResolver` reads `X-Tenant-Slug` header OR resolves from JWT `tenant_id` claim; sets `app.tenant_id` via `SET LOCAL` on the DB session for RLS.
- `/healthz` returns `{"status":"ok"}` unconditionally (liveness).
- `/readyz` pings DB and Redis; returns 503 with details if either fails.
- App title, version, contact info wired into OpenAPI metadata.

**Testing**:
- `Integration: GET /healthz returns 200 {"status":"ok"}`
- `Integration: GET /readyz returns 200 when DB+Redis up; returns 503 when DB down (stop container)`
- `Integration: GET /openapi.json returns valid OAS 3.1 (validate with openapi-spec-validator)`
- `Integration: missing X-Tenant-Slug on a tenant-scoped endpoint returns 400`

### Definition of Done
All five tasks complete; `docker compose up` brings up a working stack; `cleaning-platform tenant create` works; `pytest tests/` passes; CI green.

---

## Phase 2: Identity, Authentication & Authorisation

### Purpose
Add OAuth 2.0 / OIDC compliant authentication, JWT-based session tokens, role-based access control (RBAC), and a multi-tenant authorisation layer that wires PostgreSQL row-level security to every request. After this phase, users can log in, the API enforces "you can only see your tenant's data", and external OAuth flows (Google) work.

### Tasks

#### 2.1 — Password + email login with JWT

**What**: `/api/v1/auth/login` (email+password → access+refresh JWT pair), `/api/v1/auth/refresh`, `/api/v1/auth/logout` (revokes refresh token).

**Design**:
- Access token: 15 min TTL, signed HS256, claims `{sub:user_id, tenant_id, role, scopes, exp, iat}`.
- Refresh token: 30 day TTL, opaque random 256-bit; stored hashed in `refresh_token` table with `revoked_at`.
- Bearer token dependency `get_current_user(session, token: str = Depends(oauth2_scheme)) -> User`.
- Rate limit: 5 failed attempts per IP per 15 min via Redis sliding window.

```python
# schemas/auth.py
class LoginRequest(BaseModel):
    email: EmailStr
    password: SecretStr
    tenant_slug: str

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: Literal["bearer"] = "bearer"
    expires_in: int
```

**Testing**:
- `Unit: JWT encode/decode round-trip with claims preserved; expired token raises`
- `Integration: POST /auth/login with valid creds → 200 + tokens; invalid → 401`
- `Integration: 6th failed login from same IP → 429`
- `Integration: refresh endpoint exchanges refresh for new access; revoked refresh → 401`

#### 2.2 — OAuth 2.0 + OIDC social login (Google, Apple)

**What**: Authlib-backed OAuth flow for Google and Apple sign-in; auto-create `user` + `auth_provider` rows on first login.

**Design**:
- Endpoints: `GET /auth/oauth/{provider}/start`, `GET /auth/oauth/{provider}/callback`.
- PKCE required for mobile flows (RFC 7636).
- On callback: verify ID token signature (Apple JWKS, Google JWKS), upsert `user` by `(tenant_id, email)`, insert `auth_provider` row.
- Returns the same `TokenResponse` shape as 2.1.

**Testing**:
- `Unit: ID token verification rejects bad signature, expired, wrong audience`
- `Integration (mocked provider): callback creates user + auth_provider, returns tokens`
- `Integration (mocked provider): repeat callback reuses existing user`

#### 2.3 — RBAC and scope-based dependencies

**What**: A declarative permission system; `require(scope=...)` dependency for routes.

**Design**:
- Role → scopes mapping in `auth/rbac.py`:
  ```python
  ROLE_SCOPES: dict[str, set[str]] = {
      "owner":   {"*"},
      "admin":   {"client:*", "job:*", "invoice:*", "cleaner:*", "inspection:*", "settings:*"},
      "manager": {"client:read", "job:*", "cleaner:read", "inspection:*"},
      "member":  {"job:read", "job:update_own", "inspection:create"},
      "cleaner": {"job:read_assigned", "clock_event:*", "inspection:create"},
      "client":  {"job:read_own", "invoice:read_own", "payment:create_own"},
  }
  ```
- `require("client:read")` resolves to a FastAPI dependency that 403s if missing.
- "Own" scopes use a `resource_owner_check` callable injected per route.

**Testing**:
- `Unit: ROLE_SCOPES covers all roles; no typos in scope names`
- `Integration: GET /clients as cleaner → 403; as admin → 200`
- `Integration: GET /jobs/{id} as client when not owner → 403`

#### 2.4 — Row-level security (RLS) integration

**What**: Apply Postgres RLS policies to all tenant-scoped tables; set session variable `app.tenant_id` on every DB session.

**Design**:
- Migration `0002_rls.py` enables RLS on each tenant-scoped table and creates the policy:
  ```sql
  ALTER TABLE client ENABLE ROW LEVEL SECURITY;
  CREATE POLICY tenant_isolation ON client
      USING (tenant_id = current_setting('app.tenant_id', true)::uuid);
  ```
- Session-level hook in `db.py`: after `get_session`, executes `SET LOCAL app.tenant_id = '<uuid>'` derived from the request context.
- Admin/service roles bypass via `BYPASSRLS` only for the migration user.

**Testing**:
- `Integration: Two tenants, tenant A's client query returns 0 rows when session is set to tenant B`
- `Integration: RLS holds for direct SQL queries (not just ORM)`
- `Unit: setting an invalid uuid raises before query executes`

#### 2.5 — Audit log

**What**: A hybrid audit-trail table (`audit_log`) capturing every mutation across the API for forensic and compliance purposes (HIPAA, ISO 27001, CIMS).

**Design**:
```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    actor_id        UUID,                          -- user, or NULL for system
    actor_type      VARCHAR(20),                   -- user, system, integration
    action          VARCHAR(100) NOT NULL,         -- "client.create", "job.assign"
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID,
    before_state    JSONB,
    after_state     JSONB,
    request_id      VARCHAR(64),
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
```
- A SQLAlchemy event listener on `after_flush` writes audit rows for `Insert | Update | Delete` of any model inheriting `Auditable`.
- Sensitive fields (password_hash, secret_key) redacted to `"***"`.

**Testing**:
- `Integration: creating a Client writes one audit_log row with action="client.create" and after_state matching`
- `Integration: updating a field writes before_state/after_state diff`
- `Unit: redact() removes password_hash from after_state`

### Definition of Done
Auth flow works end-to-end; multi-tenant isolation enforced by both RLS and code; audit log populates; mypy strict passes on `auth/`.

---

## Phase 3: Core Domain — Clients, Properties, Cleaners, Services

### Purpose
Implement the foundational business entities: clients, their properties, the cleaner workforce, and the catalogue of services and add-ons. After this phase, an operator can use the admin dashboard (built in Phase 4) to manage their entire customer base, property portfolio, and crew.

### Tasks

#### 3.1 — Client and Property CRUD + search

**What**: SQLAlchemy models, Pydantic schemas, and FastAPI router for `client` and `property` resources.

**Design**:
- Models follow `data-model-suggestion-1.md` lines 91-148 exactly.
- Endpoints:
  - `GET /api/v1/clients?q=&tag=&source=&page=&per_page=` — cursor-paginated search (Postgres full-text on `last_name||first_name||company_name||email`).
  - `POST /api/v1/clients` — create with optional initial property.
  - `GET /api/v1/clients/{id}`, `PATCH /api/v1/clients/{id}`, `DELETE` (soft-delete via `is_archived=true`).
  - `GET /api/v1/clients/{id}/properties`, `POST /api/v1/clients/{id}/properties`.
  - `GET /api/v1/properties/{id}`, `PATCH`, `DELETE`.
- Geocoding hook: when a property address is created/updated, enqueue a Celery task to populate `latitude`/`longitude` via Google Geocoding.

```python
class ClientCreate(BaseModel):
    is_company: bool = False
    company_name: str | None = None
    first_name: str | None = None
    last_name: str | None = None
    email: EmailStr | None = None
    phone: str | None = None
    billing_address_line1: str | None = None
    billing_city: str | None = None
    billing_country: CountryAlpha2 = "US"
    source: ClientSource | None = None
    tags: list[str] = []
    initial_property: PropertyCreate | None = None

    @model_validator(mode="after")
    def name_required(self):
        if not self.is_company and not (self.first_name or self.last_name):
            raise ValueError("first_name or last_name required for individuals")
        if self.is_company and not self.company_name:
            raise ValueError("company_name required for companies")
        return self
```

**Testing**:
- `Unit: ClientCreate validator rejects company with no company_name`
- `Integration: POST /clients with initial_property creates both rows atomically`
- `Integration: GET /clients?q=acme returns matches by company name`
- `Integration (mocked geocoder): creating property enqueues geocode task; task populates lat/lng`
- `Integration: DELETE /clients/{id} sets is_archived=true, does not hard-delete; archived clients excluded by default`

#### 3.2 — Cleaner, Skill, Availability, Team

**What**: Workforce models and APIs.

**Design**:
- Models from `data-model-suggestion-1.md` lines 154-220.
- Endpoints under `/api/v1/cleaners`, `/api/v1/skills`, `/api/v1/teams`.
- Cleaner availability normalised into `cleaner_availability` rows (one per `day_of_week`); helper schema:
  ```python
  class CleanerAvailability(BaseModel):
      day_of_week: int = Field(ge=0, le=6)
      start_time: time
      end_time: time
      is_available: bool = True
  ```
- Skill assignment with proficiency level (`trainee | standard | expert`).
- Inviting a cleaner: `POST /cleaners` with `create_user_account: true` + `email` sends an invite email (Phase 6) and creates a `user` row with role `cleaner` and a one-time setup token.

**Testing**:
- `Unit: end_time > start_time validated`
- `Integration: POST /cleaners with skills attaches cleaner_skill rows`
- `Integration: PUT /cleaners/{id}/availability replaces all availability rows atomically`
- `Integration: archive cleaner sets is_active=false; cleaner excluded from default lists`

#### 3.3 — Service catalogue (service types, add-ons, skill requirements)

**What**: CRUD for `service_type`, `service_addon`, `service_type_skill_requirement`.

**Design**:
- Models from `data-model-suggestion-1.md` lines 226-254.
- Endpoints under `/api/v1/services` and `/api/v1/services/{id}/addons`.
- `service_type.default_duration_minutes` and `base_price` + `price_per_sqft` feed Phase 9 pricing engine.
- Validation: cannot delete a service type referenced by any active schedule rule or job; archive instead.

**Testing**:
- `Unit: ServiceTypeCreate accepts price_per_sqft as Decimal; rejects negative`
- `Integration: DELETE /services/{id} when in use → 409 with reason`
- `Integration: POST /services with skill_requirements creates junction rows`

#### 3.4 — Seed data and demo fixtures

**What**: A Typer CLI command and pytest fixture that seeds a realistic tenant with clients, properties, cleaners, services, and inspection templates.

**Design**:
- `cleaning-platform db seed-demo --tenant-slug acme [--clients 50] [--cleaners 10]`.
- Uses Faker to produce realistic addresses (validated against US ZIP ranges), names, phone numbers.
- Seed file `tests/fixtures/seed.json` is a deterministic snapshot used by e2e tests.

**Testing**:
- `Integration: seed-demo produces N clients, M properties; rerunning is idempotent (idempotency_key column)`
- `Integration: pytest fixture seeded_tenant yields a tenant with at least 50 clients, 10 cleaners`

### Definition of Done
All CRUD endpoints documented in OpenAPI; mypy strict; >=85% line coverage on `api/v1/clients`, `properties`, `cleaners`, `services`; seed CLI works.

---

## Phase 4: Scheduling — Recurring Rules, Job Materialisation, Calendar

### Purpose
The scheduling engine is the heart of the platform. This phase implements RFC 5545 RRULE-compatible recurring rules, the materialiser that turns rules into individual job rows, the dispatcher's drag-and-drop calendar API, and assignment of cleaners to jobs.

### Tasks

#### 4.1 — Schedule rule + RRULE expansion

**What**: `schedule_rule` model + an `RRuleExpander` service that materialises rules into concrete date-times for a given window.

**Design**:
- Model from `data-model-suggestion-1.md` lines 260-279.
- `services/scheduling/rrule.py`:
  ```python
  class RRuleExpander:
      def expand(self, rule: ScheduleRule, window_start: date, window_end: date) -> list[datetime]:
          """Returns scheduled_start datetimes for all occurrences within the window."""
  ```
- Wraps `dateutil.rrule.rrule` with the rule's `frequency` mapped to RRULE constants (`DAILY|WEEKLY|MONTHLY`), honouring `interval_value`, `day_of_week`, `preferred_start_time`, `starts_on`, `ends_on`.
- Timezone-aware: rule's tenant.timezone applies; converts to UTC for storage.

**Testing**:
- `Unit: weekly rule starting Mon 2026-05-04 with interval=2 over 4 weeks yields [2026-05-04, 2026-05-18]`
- `Unit: rule with ends_on=2026-06-01 truncates correctly`
- `Unit: DST transition (US Pacific 2026-03-08) preserves local 09:00 wall time`
- `Unit: invalid combinations (e.g., monthly with day_of_week=10) raise`

#### 4.2 — Job materialisation worker

**What**: A Celery beat task that runs nightly to materialise jobs for the next N days for all active schedule rules.

**Design**:
- `tasks/scheduling.py::materialise_jobs(tenant_id: UUID, horizon_days: int = 30)`.
- Beat schedule: `crontab(hour=2, minute=0)` (per-tenant timezone offset applied via wrapper).
- For each active rule:
  1. Expand the rule for the window `(now, now + horizon_days)`.
  2. For each occurrence not already represented in `job` (uniqueness: `(schedule_rule_id, scheduled_start)`), insert a new `job` row in `scheduled` status.
  3. Skip if a job already exists.
- Emits Celery events for monitoring; structured log per rule.

**Testing**:
- `Integration: rule with daily frequency materialises 30 jobs; rerunning is idempotent (no duplicates)`
- `Integration: ends_on respected — no jobs after end date`
- `Integration: pausing a rule (is_active=false) stops materialisation but does not delete existing jobs`

#### 4.3 — Job CRUD + assignment + status state machine

**What**: REST endpoints for jobs and assignments; state machine enforces valid transitions.

**Design**:
- Models from `data-model-suggestion-1.md` lines 281-345.
- State machine in `services/scheduling/state.py`:
  ```python
  TRANSITIONS = {
      "scheduled":   {"dispatched", "cancelled"},
      "dispatched":  {"in_progress", "cancelled", "no_show"},
      "in_progress": {"completed", "cancelled"},
      "completed":   set(),  # terminal
      "cancelled":   set(),  # terminal
      "no_show":     {"scheduled"},  # reschedule
  }
  ```
- Endpoints:
  - `GET /jobs?from=&to=&cleaner_id=&status=&property_id=`
  - `POST /jobs` (one-off ad-hoc job)
  - `GET /jobs/{id}`, `PATCH /jobs/{id}`
  - `POST /jobs/{id}/assign` body `{cleaner_id, role}` → inserts `job_assignment`
  - `DELETE /jobs/{id}/assignments/{assignment_id}`
  - `POST /jobs/{id}/transition` body `{to_status, reason?}` validates against `TRANSITIONS`

**Testing**:
- `Unit: transition("scheduled", "completed") → ValueError`
- `Unit: transition("dispatched", "in_progress") → "in_progress"`
- `Integration: POST /jobs/{id}/assign twice with same cleaner → 409 (unique constraint)`
- `Integration: GET /jobs?from=2026-06-01&to=2026-06-07 returns jobs in window only`

#### 4.4 — Calendar view endpoint + iCalendar feed

**What**: A `GET /api/v1/calendar` endpoint optimised for the dispatcher's drag-and-drop UI; `GET /api/v1/calendar.ics` for external calendar subscription (RFC 5545).

**Design**:
- `/calendar` returns `{date_range, jobs: [...], cleaners: [...], conflicts: [...]}` in one payload to avoid N+1 queries from the UI.
- Conflicts detected: same cleaner overlapping scheduled_start..scheduled_end windows.
- `.ics` endpoint generates a per-tenant or per-cleaner iCalendar file using `icalendar` library — emits `VEVENT` per job with `DTSTART`, `DTEND`, `LOCATION`, `SUMMARY`, `UID = job:<uuid>`, `RRULE` if from a schedule rule.

**Testing**:
- `Integration: GET /calendar?week=2026-W22 returns all jobs in week including assignments`
- `Integration: overlapping assignments surfaced in conflicts[]`
- `Unit: ics output validates against RFC 5545 (icalendar.Calendar.from_ical round-trip)`
- `Integration: GET /calendar.ics?cleaner_id=... returns only that cleaner's jobs`

#### 4.5 — Reschedule and bulk-edit

**What**: APIs for moving a job to a new time, swapping cleaners, and bulk-edit (e.g., move all of Maria's Tuesday jobs to Wednesday).

**Design**:
- `POST /jobs/{id}/reschedule` body `{new_start, new_end, reason}` → updates `scheduled_start`, `scheduled_end`, writes audit entry, optionally notifies client (Phase 6).
- `POST /jobs/bulk-reschedule` body `{filter: {cleaner_id, date}, shift_hours: int}` → background task.

**Testing**:
- `Integration: reschedule from completed status → 409`
- `Integration: bulk reschedule moves N jobs, returns task_id; task completes`

### Definition of Done
Scheduling engine produces correct recurring jobs; calendar API returns within 200ms for a tenant with 1,000 active jobs; state machine prevents invalid transitions; ics feed validates.

---

## Phase 5: Mobile Cleaner App + GPS + Clock-In/Out

### Purpose
Build the React Native cleaner app and the supporting backend APIs for clocking in/out with GPS, geofencing validation, breadcrumb tracking, and viewing assigned jobs. After this phase, a cleaner can install the app, see today's route, clock in on-site, complete the job, and clock out.

### Tasks

#### 5.1 — Clock-in / clock-out API + geofencing

**What**: Endpoints for cleaners to record clock events; server-side geofencing validation.

**Design**:
- Model from `data-model-suggestion-1.md` lines 350-365.
- Endpoint: `POST /api/v1/cleaner/jobs/{job_id}/clock` body:
  ```python
  class ClockEventCreate(BaseModel):
      event_type: Literal["clock_in", "clock_out", "break_start", "break_end"]
      latitude: float = Field(ge=-90, le=90)
      longitude: float = Field(ge=-180, le=180)
      accuracy_meters: float | None = None
      recorded_at: datetime                # client clock (server records authoritative)
      device_info: str | None = None
  ```
- Server computes `is_within_geofence` by haversine distance between event and `property.latitude/longitude` against a configurable radius (default 150m; per-property override).
- Returns `{accepted: true, within_geofence: bool, server_recorded_at: ...}`.
- Optional strict mode: if tenant.settings.strict_geofence and not within_geofence → 422 with reason.

**Testing**:
- `Unit: haversine(40.7,-74.0, 40.7001,-74.0) ≈ 11m within tolerance`
- `Integration: clock_in with coords within 150m of property → within_geofence=true`
- `Integration: clock_in with coords 1km away → within_geofence=false; in strict mode → 422`
- `Integration: clock_out without prior clock_in → 409`

#### 5.2 — GPS breadcrumb ingestion

**What**: Endpoint for the mobile app to batch-upload location samples during a shift.

**Design**:
- Model from `data-model-suggestion-1.md` lines 367-377.
- Endpoint: `POST /api/v1/cleaner/breadcrumbs` body `{samples: [{latitude, longitude, accuracy_meters, recorded_at}, ...]}`.
- Stores at most 1 sample per cleaner per 30s (server-side dedupe by `recorded_at` bucket).
- `gps_breadcrumb` table partitioned by month (Alembic migration creates partitioning template).

**Testing**:
- `Integration: POST 100 breadcrumbs in one batch → 100 rows inserted (or fewer with dedupe)`
- `Integration: breadcrumbs older than 30 days excluded by retention policy task`

#### 5.3 — Cleaner-facing job list and detail endpoints

**What**: Endpoints scoped to the authenticated cleaner; returns only their assigned jobs.

**Design**:
- `GET /api/v1/cleaner/jobs?date=2026-06-01` — jobs assigned to current user.
- `GET /api/v1/cleaner/jobs/{id}` — full detail including property address, access instructions, checklist, attached photos, previous-visit notes.
- `POST /api/v1/cleaner/jobs/{id}/checklist/{item_id}/complete`.
- All endpoints require `role=cleaner` and check `job_assignment.cleaner_id == current_user.cleaner.id`.

**Testing**:
- `Integration: cleaner A cannot fetch a job assigned only to cleaner B → 404 (not 403, to avoid info leak)`
- `Integration: GET /cleaner/jobs?date=2026-06-01 returns only that day's jobs`

#### 5.4 — Photo upload endpoints + processing

**What**: Pre-signed S3/MinIO uploads + post-upload metadata recording for `inspection_photo` and a new `job_photo` table.

**Design**:
- `POST /api/v1/uploads/signed-url` body `{purpose: "job_photo"|"inspection_photo", filename, content_type}` → returns `{upload_url, storage_key, expires_at}`.
- Pre-signed PUT expires in 5 min; max size 20MB enforced by S3 policy.
- After upload, client calls `POST /jobs/{id}/photos` body `{storage_key, photo_type: "before"|"after"|"issue", caption, latitude, longitude, taken_at}`.
- Celery task on photo registration: strip EXIF (privacy), generate 256px and 1024px thumbnails, write back as `{key}_thumb.jpg` and `{key}_lg.jpg`.

```sql
CREATE TABLE job_photo (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    job_id UUID NOT NULL REFERENCES job(id) ON DELETE CASCADE,
    cleaner_id UUID REFERENCES cleaner(id),
    storage_key VARCHAR(500) NOT NULL,
    thumb_storage_key VARCHAR(500),
    photo_type VARCHAR(20) NOT NULL,
    caption VARCHAR(255),
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    taken_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing**:
- `Integration: signed URL successfully uploads a 1MB jpeg; subsequent POST registers photo; thumbnails generated within 30s`
- `Integration: 25MB file rejected at S3 with 400`
- `Unit: EXIF stripped (Pillow image.info empty after process)`

#### 5.5 — React Native app — login, today's jobs, clock-in, photo capture, offline

**What**: Expo-based cross-platform app covering the cleaner's daily workflow.

**Design**:
- Screens: Login → Today (job list with map) → Job Detail (instructions, checklist, photos) → Camera → Clock-in/out buttons → Sync indicator.
- Offline-first: SQLite cache mirrors `/cleaner/jobs?date=today`; mutations queued in `outbox` table and replayed when online.
- Photo capture uses `expo-camera`; photos written to device storage first, uploaded via pre-signed URL when online.
- Background location: `expo-location` with `Location.startLocationUpdatesAsync` posting breadcrumbs in 60s batches.
- Localised strings via `i18n-js` with `en` and `es` bundled.
- Push notifications via Expo Notifications for new job assignments and dispatcher messages.

**Testing**:
- `E2E (Detox): login → today screen lists jobs → tap job → clock-in → take photo → clock-out → completes job`
- `E2E (offline): airplane mode → clock-in queued → online → clock event posted`
- `Unit (Jest): outbox replay handles 409 by marking conflict; UI shows resolution prompt`
- `Manual: i18n switch to es shows Spanish strings; pluralisation rules correct`

### Definition of Done
A cleaner can use the app on a real device for an entire shift; breadcrumbs visible on dispatcher map; offline workflow proven; cross-platform parity verified on iOS 17 and Android 14.

---

## Phase 6: Communications — SMS, Email, Templates, Reminders

### Purpose
Implement the platform's outbound communication layer: SMS via Twilio, email via SendGrid, templated messages for the entire client and cleaner lifecycle, and the automation engine that fires reminders, post-service summaries, and review requests on schedule. This is required for ZenMaid/Housecall-Pro parity.

### Tasks

#### 6.1 — Channel adapters + communication log

**What**: Pluggable `SmsClient` and `EmailClient` interfaces with Twilio and SendGrid implementations; every send writes to `communication_log`.

**Design**:
- `communication_log` from `data-model-suggestion-1.md` lines 668-686.
- Abstract base classes:
  ```python
  class SmsClient(Protocol):
      async def send(self, to: str, body: str, from_: str | None = None) -> SendResult: ...
  class EmailClient(Protocol):
      async def send(self, to: EmailAddress, subject: str, html: str, text: str | None = None) -> SendResult: ...
  class SendResult(BaseModel):
      external_id: str
      status: Literal["queued", "sent", "failed"]
      error: str | None = None
  ```
- All sends go through a `Messenger` service that writes a `communication_log` row before and updates after.
- Inbound webhooks (`/webhooks/twilio/status`, `/webhooks/sendgrid/events`) update `communication_log.status` (delivered, bounced, failed).
- HMAC-validated webhook signatures.

**Testing**:
- `Unit (mocked Twilio): send returns external_id, log row inserted with status=sent`
- `Integration: Twilio status webhook with valid signature updates log to delivered`
- `Integration: webhook with bad signature → 401, no update`

#### 6.2 — Template engine

**What**: Jinja2-based templates for SMS and email, with content for each lifecycle event.

**Design**:
- Templates in `services/communication/templates/`, organised:
  ```
  templates/
    sms/
      client/booking_confirmation.en.txt
      client/booking_confirmation.es.txt
      client/reminder_24h.en.txt
      client/cleaner_on_way.en.txt
      client/post_service_summary.en.txt
      client/review_request.en.txt
      cleaner/new_assignment.en.txt
      cleaner/shift_reminder.en.txt
    email/
      client/booking_confirmation.en.html
      client/invoice.en.html
      client/inspection_report.en.html
      ...
  ```
- Resolution: locale fallback `es` → `en` for missing templates.
- Variables documented per template; rendering validated against a schema in `templates/manifest.yaml`.

**Testing**:
- `Unit: render booking_confirmation.en.txt with valid context produces expected output`
- `Unit: missing required variable raises before send`
- `Unit: es locale falls back to en when es template missing`

#### 6.3 — Automation engine (reminders, follow-ups)

**What**: Celery beat tasks that scan upcoming/recent jobs and trigger templated messages.

**Design**:
- Tasks:
  - `send_24h_reminders` (hourly): finds jobs starting in the next 23–25 hours that haven't received `reminder_24h`; sends SMS and/or email per client preference.
  - `send_cleaner_on_way` (every 5 min): on `job.transition → dispatched`, fires.
  - `send_post_service_summary` (every 10 min): on `job.transition → completed`, fires after a 1-hour delay.
  - `send_review_request` (daily): finds jobs completed 24h ago with `client.email` present and no existing review_request row; creates and sends.
  - `send_new_assignment` (event-driven): on `job_assignment` insert, sends to cleaner.
- Client communication preferences stored in `client.settings.communication: {sms: bool, email: bool, channels_per_event: {...}}`.

**Testing**:
- `Integration: a job 24h out generates exactly one reminder; rerunning task does not duplicate (idempotency check via communication_log)`
- `Integration: client opted out of SMS → only email sent`
- `Integration: completed job triggers post-service summary after 1h delay`

#### 6.4 — Review request workflow (Google, Facebook)

**What**: Generate trackable review-request links; record outcome.

**Design**:
- `review_request` model from `data-model-suggestion-1.md` lines 688-699.
- For Google: link constructed from `https://search.google.com/local/writereview?placeid=<gbp_place_id>` (tenant's Google Business Profile ID stored in `tenant.settings`).
- Link includes signed short ID `https://app.example.com/r/{token}` that redirects after recording the click → `status=sent → clicked`.
- Optional internal review form `/r/{token}/feedback` for low-NPS branches (gated by client rating ≤ 3).

**Testing**:
- `Integration: GET /r/{token} → 302 to Google review URL; review_request.status=clicked`
- `Integration: low-rating internal form submission stores rating in review_request`

#### 6.5 — Inbound SMS replies + opt-out (STOP)

**What**: Inbound SMS webhook handler; processes STOP/HELP/SUBSCRIBE keywords per TCPA.

**Design**:
- `/webhooks/twilio/inbound` parses body, finds client by `phone`, updates `client.settings.communication.sms=false` on STOP.
- Replies with confirmation per Twilio carrier requirements ("You have been unsubscribed...").
- All inbound messages logged as `communication_log` direction=`inbound`.

**Testing**:
- `Integration: STOP inbound → client.settings.communication.sms=false`
- `Integration: subsequent send_24h_reminders skips this client`

### Definition of Done
End-to-end SMS and email send/receive working in a staging environment; reminders fire at the right time on a real tenant; unsubscribe respected within one minute.

---

## Phase 7: Client Acquisition — Booking Widget, Client Portal, Quoting

### Purpose
Build the customer-facing surface: an embeddable booking form for ingestion of new bookings from a cleaning business's marketing site; a client portal for viewing appointments and paying invoices; and a quote-to-proposal pipeline with online client approval.

### Tasks

#### 7.1 — Public booking endpoints (no auth)

**What**: Anonymous endpoints powering the embeddable widget. They produce a new `client` (if not existing), a `property`, and one or more `job`s (or a `schedule_rule` for recurring).

**Design**:
- `GET /api/v1/booking/{tenant_slug}/services` — public list of `service_type` and `service_addon` (filtered to `is_active`).
- `GET /api/v1/booking/{tenant_slug}/availability?service_type_id=&date=` — returns 30-min slots open for booking; computed from cleaner availability and existing jobs.
- `POST /api/v1/booking/{tenant_slug}` — body:
  ```python
  class PublicBookingRequest(BaseModel):
      first_name: str
      last_name: str
      email: EmailStr
      phone: str
      property: PropertyCreate
      service_type_id: UUID
      addon_ids: list[UUID] = []
      requested_start: datetime
      frequency: Literal["once", "weekly", "biweekly", "monthly"] = "once"
      special_instructions: str | None = None
      payment_intent_token: str | None = None   # from Stripe Elements
      recaptcha_token: str
  ```
- Rate-limited: 5 bookings per IP per hour; reCAPTCHA v3 required (skip in dev).
- On success: returns `{booking_id, job_id, confirmation_token}`. Triggers booking confirmation email + SMS.

**Testing**:
- `Integration: POST /booking/{slug} with valid payload creates client + property + job; status=scheduled`
- `Integration: same email rebooking attaches to existing client`
- `Integration: 6 bookings from same IP → 429`
- `Integration: invalid recaptcha → 422`

#### 7.2 — Embeddable booking widget (Next.js)

**What**: Standalone JS bundle (`<script src="https://.../widget.js" data-tenant="acme">`) that injects a booking form into any host page.

**Design**:
- Built with Next.js but exported as a static `widget.js` + `widget.css` (using Next.js standalone output + a wrapper).
- Iframe sandbox option for stricter isolation.
- Steps: service selection → property details → date/time picker → contact → Stripe Elements card capture → confirm.
- All API calls to `/api/v1/booking/{tenant_slug}/...`.
- Theming via CSS variables exposed to the host page.

**Testing**:
- `E2E (Playwright): visit a demo host page → complete booking → confirmation shown → confirmation email arrives`
- `Manual: widget renders correctly in iframe and inline modes; mobile-responsive`

#### 7.3 — Client portal (Next.js)

**What**: Logged-in portal at `/portal` for clients to view appointments, approve quotes, pay invoices.

**Design**:
- Authentication: magic link login (`POST /api/v1/auth/magic-link` sends signed link valid 15 min) — clients are typically not used to password apps.
- Pages: Dashboard (next appointment, outstanding invoices, quick actions), Appointments, Invoices, Quotes, Payment Methods, Profile.
- All data from API endpoints scoped to `client:read_own` scope.
- Stripe Elements for adding payment methods (`POST /clients/{id}/payment-methods`).

**Testing**:
- `E2E: magic link login → dashboard loads → pay outstanding invoice → status updates to paid`
- `E2E: approve a sent quote → quote.status=accepted`

#### 7.4 — Quote-to-proposal workflow

**What**: APIs to create, send, and track quotes; client-facing quote view in the portal.

**Design**:
- Models from `data-model-suggestion-1.md` lines 526-554.
- Endpoints:
  - `POST /api/v1/quotes` body `{client_id, property_id, service_type_id, line_items, valid_until, message}`.
  - `POST /api/v1/quotes/{id}/send` — sets `status=sent`, sends email with signed link.
  - `GET /api/v1/portal/quotes/{id}?token=...` — client-facing view (no auth required when token valid).
  - `POST /api/v1/portal/quotes/{id}/accept|decline?token=...`.
- On accept: optionally auto-convert to a job or schedule rule (configurable).

**Testing**:
- `Integration: send quote with valid email → quote.sent_at populated`
- `Integration: accept quote via token → status=accepted, responded_at set, audit row written`
- `Integration: expired quote (valid_until past) cannot be accepted → 422`

### Definition of Done
A complete booking-to-paid flow proven end-to-end on staging with a real Stripe test account; embeddable widget loads on three third-party sites without conflict; magic link auth works.

---

## Phase 8: Billing — Invoicing, Stripe Payments, Recurring Charges

### Purpose
Implement the financial layer: invoice generation from completed jobs and from contracts, Stripe payment processing, saved payment methods, recurring auto-charging, and outstanding-balance tracking.

### Tasks

#### 8.1 — Invoice model + auto-generation on job completion

**What**: `invoice`, `invoice_line_item`, `payment` models and the worker that creates draft invoices when a job is completed.

**Design**:
- Models from `data-model-suggestion-1.md` lines 461-521.
- Celery task `on_job_completed(job_id)`: creates a draft invoice with one `invoice_line_item` per `service_type` + add-ons, applies tax rate from tenant.settings, calculates totals.
- Endpoints: `GET /invoices`, `GET /invoices/{id}`, `POST /invoices`, `PATCH /invoices/{id}`, `POST /invoices/{id}/send`, `POST /invoices/{id}/void`.
- Invoice number format: `INV-YYYY-NNNNNN` per tenant, generated via Postgres sequence.

**Testing**:
- `Integration: completing a job with service base_price=$120 + 1 addon $30 → invoice total=$150 + tax`
- `Integration: invoice number unique per tenant; concurrent generation produces no duplicates`
- `Integration: sending invoice fires email; invoice.sent_at and status=sent`

#### 8.2 — Stripe integration — payment intents, payment methods

**What**: Stripe payment processing for one-off invoice payments and saved card management.

**Design**:
- `services/billing/stripe_gateway.py` wraps the Stripe SDK.
- Endpoints:
  - `POST /api/v1/invoices/{id}/pay` — creates a Stripe PaymentIntent for the invoice amount, returns `client_secret` for Stripe Elements.
  - `POST /api/v1/clients/{id}/payment-methods` — body `{stripe_pm_id}`; attaches to Stripe customer, stores in `client_payment_method`.
- Webhook `/webhooks/stripe`:
  - `payment_intent.succeeded` → mark invoice paid, insert `payment` row.
  - `payment_intent.payment_failed` → notify client.
  - HMAC-SHA256 verified per Stripe docs.

**Testing**:
- `Integration (Stripe test mode): create PaymentIntent for invoice → succeed → invoice.status=paid`
- `Integration: webhook with invalid signature → 400`
- `Integration: 3DS-required test card → action_required handled by Elements`

#### 8.3 — Recurring auto-charge worker

**What**: Daily Celery beat task that finds invoices for clients with a default `client_payment_method` and auto-charges if the invoice is past `auto_charge_after_days` (tenant setting).

**Design**:
- Task: `auto_charge_invoices`.
- Logic: for each unpaid invoice with `auto_charge_eligible=true`, attempt a PaymentIntent with `confirm=true, off_session=true, payment_method=<saved>`.
- Records attempt in `payment` row (status=pending → succeeded|failed).
- On failure, sends client an email with a manual-pay link; retries with exponential backoff up to 3 times.

**Testing**:
- `Integration: invoice past due with saved card → auto-charge succeeds, status=paid`
- `Integration: auto-charge declined → email sent, 3 retries scheduled at 1d, 3d, 7d`

#### 8.4 — Invoice PDF rendering

**What**: WeasyPrint-based PDF rendering for invoices using an HTML template.

**Design**:
- `services/billing/pdf.py::render_invoice(invoice) -> bytes`.
- Template `templates/pdf/invoice.html` with tenant branding (logo from `tenant.settings.branding.logo_url`).
- PDF stored in S3 (`invoices/{tenant_id}/{invoice_id}.pdf`) and linked from email.

**Testing**:
- `Unit: render_invoice returns valid PDF (signature %PDF-)`
- `Manual: visually inspect a rendered invoice`

#### 8.5 — QuickBooks Online sync (basic)

**What**: One-way push of customers and invoices to QuickBooks Online.

**Design**:
- OAuth 2.0 connection flow at `/integrations/quickbooks/connect` → stores refresh token in `integration_credential` table (encrypted via Fernet using `Settings.secret_key`).
- Push on events: `client.created` → upsert Customer; `invoice.sent` → upsert Invoice with mapped line items; `payment.succeeded` → record Payment.
- Mapping table `quickbooks_id_map (entity_type, internal_id, qb_id)` tracks correlations.
- Background reconciliation task hourly to detect drift.

**Testing**:
- `Integration (mocked QBO): client.created event → API call observed with correct fields`
- `Integration (mocked QBO): existing client (qb_id known) → UPDATE not CREATE`
- `Integration: OAuth refresh token automatically refreshes on 401`

### Definition of Done
End-to-end paid invoice via Stripe; recurring auto-charge produces no duplicates; QBO sync mirrors at least clients + invoices + payments without manual intervention.

---

## Phase 9: Quality Inspection — Templates, Mobile Inspections, Reports

### Purpose
Implement the differentiating inspection workflow: customisable templates per property type, mobile inspection completion with photos and GPS, automatic scoring, and scheduled client-facing PDF report delivery (OrangeQC-class capability inside an FSM platform).

### Tasks

#### 9.1 — Inspection template models + builder API

**What**: Hierarchical templates (template → sections → items); endpoints to create, version, and manage them.

**Design**:
- Models from `data-model-suggestion-1.md` lines 383-410.
- Endpoints under `/api/v1/inspection-templates`.
- Versioning: `PATCH /templates/{id}` increments `version` only when items change; existing inspections always reference the version they were created with via `inspection.template_id` snapshot.
- Pre-loaded templates seeded for residential, commercial, healthcare.

**Testing**:
- `Integration: POST template + sections + items in one call, transactional`
- `Integration: editing items increments version; existing inspections unchanged`

#### 9.2 — Inspection execution API (in-progress, complete, submit)

**What**: APIs for inspectors to perform an inspection in the field.

**Design**:
- Models from `data-model-suggestion-1.md` lines 412-455.
- Endpoints:
  - `POST /inspections` body `{property_id, template_id, job_id?, latitude, longitude}` → returns `inspection_id` in `in_progress`.
  - `POST /inspections/{id}/results` body `{template_item_id, rating, numeric_score?, notes}` (upsert per item).
  - `POST /inspections/{id}/photos` body `{result_id?, storage_key, caption, photo_type, latitude, longitude}`.
  - `POST /inspections/{id}/complete` → calculates `overall_score` from weighted item ratings, transitions to `completed`.
  - `POST /inspections/{id}/submit` → transitions to `submitted`, schedules report generation (Phase 9.4).
- Scoring: per-section weighted average of item scores (normalised 0-100); overall is weighted average across sections.

**Testing**:
- `Unit: scoring with 3 items pass/fail/pass with weights → expected aggregate`
- `Integration: submit when not all required items rated → 422 with list of missing`
- `Integration: completing then submitting fires report task`

#### 9.3 — Inspection in the cleaner/inspector mobile app

**What**: Mobile screens for performing inspections offline.

**Design**:
- Inspection list (assigned inspections), template loaded into local SQLite, items rendered as forms (pass/fail toggles, 1-5 sliders, exceeds/meets/doesn't dropdowns).
- Photo capture per item with annotation (drawing tool via `react-native-skia`).
- Offline sync queue mirrors API endpoints from 9.2.

**Testing**:
- `E2E: inspector completes a 20-item inspection offline → goes online → all results sync, scoring correct`
- `Manual: annotation tool produces a flattened image at upload time`

#### 9.4 — PDF inspection report + scheduled delivery

**What**: Templated PDF report; cron task to deliver per-property report schedules to client stakeholders.

**Design**:
- `services/inspection/report.py::render_report(inspection) -> bytes`.
- Template `templates/pdf/inspection_report.html` shows: header (tenant logo, property), overall score, per-section breakdown with item details, embedded photo thumbnails (linked to full-size in PDF appendix), inspector signature.
- A new table:
  ```sql
  CREATE TABLE report_subscription (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      tenant_id UUID NOT NULL REFERENCES tenant(id),
      property_id UUID NOT NULL REFERENCES property(id),
      email VARCHAR(255) NOT NULL,
      cadence VARCHAR(20) NOT NULL,  -- per_inspection, daily, weekly, monthly
      next_send_at TIMESTAMPTZ,
      is_active BOOLEAN DEFAULT true
  );
  ```
- Celery beat task `send_report_subscriptions` (hourly): gathers inspections in the period, generates a digest PDF if multiple, emails each subscriber.

**Testing**:
- `Unit: render_report returns valid PDF; contains overall_score text`
- `Integration: weekly subscription accumulates 3 inspections → one email with merged report`
- `Integration: per_inspection subscription sends one email per submit`

#### 9.5 — Inspection-driven remediation jobs

**What**: When an item rates "fail" or below threshold, auto-create a remediation job assigned to the next available cleaner.

**Design**:
- Trigger on `inspection.submitted`: collect failed items, group by location, create one `job` with status=`scheduled`, `priority=high`, with checklist items copied from the failed inspection items.
- Configurable per template: `auto_remediation_enabled: bool`.

**Testing**:
- `Integration: submit inspection with 3 failed items → 1 remediation job created with 3 checklist items`
- `Integration: auto_remediation_enabled=false → no job created`

### Definition of Done
Inspectors can perform inspections offline on iPad/Android tablet; clients receive professional PDF reports; remediation loop closes the feedback loop end-to-end.

---

## Phase 10: Public Webhooks & External REST API

### Purpose
Open the platform via a stable, versioned public REST API and outbound webhooks — addressing the standards.md insight that incumbents lock APIs behind enterprise plans or don't publish them at all. After this phase, third-party developers can integrate against documented endpoints, subscribe to event webhooks, and build a best-of-breed cleaning stack.

### Tasks

#### 10.1 — API key management for external integrations

**What**: `api_key` model and management endpoints for tenants to issue/revoke programmatic access keys.

**Design**:
```sql
CREATE TABLE api_key (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(100) NOT NULL,
    key_prefix CHAR(8) NOT NULL,           -- displayed in UI
    key_hash VARCHAR(128) NOT NULL,        -- argon2 hash
    scopes TEXT[] NOT NULL DEFAULT '{}',
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    created_by UUID REFERENCES "user"(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, key_prefix)
);
```
- Key format: `cp_live_<32 base62 chars>`; `key_prefix` is first 8 chars.
- Endpoint `POST /api/v1/api-keys` returns the full key once; only the hash is stored.
- Auth dependency `get_api_key()` checks Bearer prefix, hashes, looks up, verifies scopes.

**Testing**:
- `Integration: created key works for one read; revoked key → 401`
- `Integration: scoped key (read-only) cannot POST → 403`
- `Unit: hashing is deterministic for same input; collision-free for random keys`

#### 10.2 — Webhook subscription + delivery

**What**: Subscriptions to platform events with HMAC-signed deliveries.

**Design**:
```sql
CREATE TABLE webhook_subscription (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    url VARCHAR(500) NOT NULL,
    event_types TEXT[] NOT NULL,
    signing_secret VARCHAR(128) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_delivery (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES webhook_subscription(id),
    event_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, succeeded, failed, abandoned
    attempt_count INTEGER DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    next_attempt_at TIMESTAMPTZ,
    response_status_code INTEGER,
    response_body TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- Event types catalogue (subset of event store concepts from data-model-suggestion-2): `job.created`, `job.completed`, `job.cancelled`, `invoice.paid`, `inspection.submitted`, `client.created`, `cleaner.clock_in`, `quote.accepted`.
- A domain-event publisher (`services/events.py`) emits typed events on key transactions; Celery task fans out to active subscriptions.
- Delivery: HMAC-SHA256 (`Cleaning-Platform-Signature: t=<unix>,v1=<hex>`); 5s timeout; retry with exponential backoff (1m, 5m, 30m, 2h, 6h, 24h) up to 6 attempts.

**Testing**:
- `Integration: create subscription; trigger job.completed → delivery queued; mock receiver gets POST with valid signature`
- `Unit: signature verification logic mirrors Stripe pattern (timestamp + payload)`
- `Integration: failing receiver (500) → 6 retries observed; then status=abandoned`

#### 10.3 — Public API surface + OpenAPI 3.1 spec polish

**What**: Curate the public-stable endpoints under `/api/v1/public/...` with strict semver, documented changelog, and a polished OpenAPI 3.1 spec.

**Design**:
- Promote stable read endpoints (`/clients`, `/jobs`, `/inspections`, `/invoices`) under a `/public` namespace; write endpoints follow.
- Each route tagged with stability level (`stable`, `beta`, `experimental`) in OpenAPI extensions.
- `docs/api-reference.md` is generated from the OpenAPI spec via Redoc.
- Version negotiation via `Cleaning-Platform-Version: 2026-05-01` header.

**Testing**:
- `Integration: GET /public/v1/jobs with API key returns paginated list`
- `Validation: OpenAPI 3.1 lint passes (spectral)`
- `Integration: omitted version header defaults to latest`

#### 10.4 — Rate limiting + usage metering

**What**: Per-API-key sliding-window rate limit and metering for billing/quotas.

**Design**:
- Redis-backed sliding window: 60 req/min default, configurable per `api_key.scopes`.
- Headers: `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`.
- Daily metering aggregated into `api_usage_daily` table for billing/reporting.

**Testing**:
- `Integration: 60 successful requests, 61st → 429 with Retry-After`
- `Integration: daily aggregate shows correct request count`

### Definition of Done
External developer can register an API key, subscribe to webhooks, and integrate against the documented spec in under an hour; webhook deliveries achieve >99% reliability in staging soak test.

---

## Phase 11: Routing & Dispatch — Google Maps + OR-Tools VRP

### Purpose
Implement intelligent route optimisation — the AI-native advantage over manual dispatch in incumbent platforms. Operators get one-click optimised routes accounting for cleaner skills, proximity, traffic, and time windows.

### Tasks

#### 11.1 — Route + RouteStop persistence

**What**: Models for storing optimisation outputs.

**Design**: Models from `data-model-suggestion-1.md` lines 705-732.

**Testing**:
- `Integration: create route + 5 stops; query returns ordered`
- `Integration: deleting a job cascade-deletes route_stop (or sets job_id null per FK policy)`

#### 11.2 — Google Routes API adapter

**What**: Wrapper over Google's Route Optimization API for online use.

**Design**:
- `services/routing/google_routes.py::optimise(request: OptimiseRequest) -> Route`.
- Input: list of `Stop` (job_id, address, lat/lng, time_window, service_duration_minutes), list of `Vehicle` (cleaner_id, start/end location, available time windows, skills).
- Output: ordered stops per vehicle with ETA and travel times.
- Cached per `(tenant_id, route_date)` for 1 hour to avoid API spend.

**Testing**:
- `Integration (mocked Google): 10-stop request → returns 2 vehicle routes with all stops covered`
- `Integration (live, optional): real Google API call against test project key, marked @slow`

#### 11.3 — OR-Tools VRP solver (self-hosted alternative)

**What**: Constraint-aware VRP solver that runs in-process for self-hosted deployments.

**Design**:
- `services/routing/ortools_solver.py::optimise(...) -> Route`.
- Uses OR-Tools CP-SAT + Routing Library: vehicles=cleaners, locations=properties, demand=service_duration, time windows from `job.scheduled_start..scheduled_end`, soft constraint for cleaner-skill match.
- Distance matrix from OSRM (self-hosted) or precomputed haversine.
- Solver timeout configurable (default 30s).

**Testing**:
- `Unit: 5 stops, 2 vehicles, 1 hour each → solver returns feasible solution`
- `Unit: 50 stops with infeasible time windows → returns best-effort with unassigned list`
- `Bench: 100-stop problem solves in <30s on a dev machine`

#### 11.4 — Optimise endpoint + dispatcher UI hooks

**What**: `POST /api/v1/routes/optimise` body `{date, cleaner_ids[], strategy: "google"|"ortools"}` → returns proposed route + persists.

**Design**:
- Returns `{routes: [{cleaner_id, stops: [...]}], unassigned: [...]}`.
- Operator can preview then `POST /routes/{id}/commit` to update `job.scheduled_start` per route ordering.

**Testing**:
- `Integration: optimise returns route within 60s for 50 jobs`
- `Integration: commit updates job rows transactionally`

#### 11.5 — Travel time forecasts in calendar view

**What**: Display estimated travel time between consecutive jobs in the calendar.

**Design**:
- Calendar API (from Phase 4.4) augmented with `travel_minutes_to_next` on each scheduled job; computed from `route_stop` if route exists, else from cached distance matrix.

**Testing**:
- `Integration: GET /calendar?week=... includes travel_minutes_to_next on adjacent same-cleaner jobs`

### Definition of Done
Dispatcher can optimise an entire day's schedule in <60s; cleaner sees route order in mobile app; both Google and OR-Tools backends produce comparable solutions for benchmark cases.

---

## Phase 12: AI Layer — Communications, Vision, Pricing, Retention

### Purpose
Layer in the AI-native differentiators that justify the platform's positioning. Each AI feature is gated behind a feature flag so deployments can disable individual capabilities or swap LLM providers.

### Tasks

#### 12.1 — LLM client abstraction

**What**: A provider-neutral LLM client with prompt caching, retries, and observability.

**Design**:
```python
class LLMClient(Protocol):
    async def complete(self, system: str, messages: list[Message],
                       tools: list[Tool] | None = None,
                       max_tokens: int = 1024) -> Completion: ...
    async def complete_vision(self, system: str, prompt: str,
                              images: list[Image]) -> Completion: ...

# Implementations
class AnthropicClient(LLMClient): ...
class OpenAIClient(LLMClient): ...
```
- Anthropic implementation uses prompt caching for system prompts and reusable context.
- Retries with jitter on 429/503 (max 3).
- Token usage logged to `llm_usage` table for cost attribution per tenant.

**Testing**:
- `Unit (mocked SDK): complete returns Completion with text and usage`
- `Unit: 429 retry succeeds after 2 attempts`
- `Integration: usage row written per call`

#### 12.2 — AI communication drafting

**What**: Generate personalised, on-brand reminders, post-service summaries, and review requests from templates with LLM-augmented variation.

**Design**:
- `services/ai/communication.py::draft(template_key: str, context: dict, tone: str) -> str`.
- Loads a Jinja template containing placeholders + instructs the LLM to generate variations: "Generate a friendly post-service summary for {client.name} after their {service_type} on {date}. Highlight: {notes}. Keep to 2 sentences."
- Cached per `(template_key, hash(context))` for 24h (Redis) to reduce cost.
- Output passes a profanity/PII filter before sending.

**Testing**:
- `Unit (mocked LLM): draft returns string ≤ N words`
- `Integration: variant cache hit avoids second LLM call`
- `Unit: PII filter strips credit card numbers`

#### 12.3 — AI photo quality inspection

**What**: Vision-based analysis of submitted job photos to flag potential issues before client sees them.

**Design**:
- Celery task `analyse_photo(photo_id)` triggered after thumbnail generation.
- Prompt template:
  ```
  System: You are a quality inspector for a cleaning service. Analyse before/after photos and identify any visible issues: dust, streaks, missed areas, debris, water marks. Be precise and reference visible regions.
  User: {photo_type} photo of {property_type} {room_type if known}. Compare to standard expectations.
  ```
- Vision LLM returns structured JSON via tool use: `{"overall": "clean|acceptable|issues", "issues": [{"region": "top-left", "type": "dust", "severity": "minor"}]}`.
- Issues with severity ≥ moderate create an `inspection_alert` row visible to the operations dashboard.
- Configurable confidence threshold; can be disabled per tenant.

**Testing**:
- `Unit (mocked vision): structured output parses correctly`
- `Integration: photo with known issue (fixture) flagged with severity≥moderate`
- `Integration: photo flagged as clean produces no alert`

#### 12.4 — Dynamic pricing engine

**What**: Suggest quotes based on property attributes, historical job durations, and market rates.

**Design**:
- `services/pricing/engine.py::suggest_price(property: Property, service_type_id: UUID, frequency: str) -> PriceSuggestion`.
- Inputs:
  - Property: `square_footage`, `bedroom_count`, `bathroom_count`, `property_type`.
  - Historical: avg `actual_duration` for this `service_type` at similar properties (Postgres percentile query).
  - Market: per-region baseline from `tenant.settings.market_rates` (operator-configured).
  - Discount: frequency factor (weekly 0.9, biweekly 0.95, monthly 1.0, once 1.05).
- Returns `{low, suggested, high, breakdown}`.
- Optional LLM-narrative: "Suggested $145 because similar 3-bed 2-bath homes in your area take 2.5h on average."

**Testing**:
- `Unit: 1200 sqft, 3-bed, biweekly standard clean → suggested in expected range`
- `Unit: no historical data falls back to base_price + price_per_sqft`
- `Snapshot: pricing for fixture properties matches recorded baselines`

#### 12.5 — Cleaner retention analytics

**What**: Dashboard surfacing cleaners with patterns predictive of turnover.

**Design**:
- Weekly Celery task `compute_retention_signals(tenant_id)`:
  - Hours worked vs `max_hours_per_week` ratio.
  - Late clock-ins per week trend.
  - Inspection score trend.
  - Inter-job idle time.
  - Sentiment from cleaner in-app messages (LLM sentiment classification).
- Outputs a `cleaner_retention_score` row per cleaner: `{risk: low|moderate|high, signals: [...], recommended_actions: [...]}`.
- LLM-generated narrative for high-risk cleaners with suggested interventions.

**Testing**:
- `Unit: scoring function produces deterministic risk band given known inputs`
- `Integration: end-to-end task produces rows for every active cleaner`
- `Snapshot: high-risk narrative includes at least one recommendation`

#### 12.6 — MCP server for AI agents (post-MVP nicety)

**What**: Expose key operations via Model Context Protocol so external AI agents (Claude Desktop, Cursor) can query and act on the platform.

**Design**:
- `/mcp` endpoint serving the MCP protocol per https://modelcontextprotocol.io/.
- Tools exposed: `list_jobs(date)`, `get_job(id)`, `assign_cleaner(job_id, cleaner_id)`, `create_quote(...)`, `get_inspection_summary(property_id)`.
- Auth via API key; scopes mapped to MCP tool whitelist.

**Testing**:
- `Integration: MCP handshake completes; tools list returned`
- `Integration: list_jobs tool returns same data as REST equivalent`

### Definition of Done
Each AI feature can be toggled off without breaking the rest of the system; LLM cost per tenant tracked; vision pipeline flags 90%+ of known-bad photos in fixture set; pricing suggestions land within ±15% of operator-set baselines.

---

## Phase 13: Compliance — SDS, EPA Safer Choice, LEED, CIMS

### Purpose
Implement the chemical, product, and certification compliance features that no incumbent reviewed in research.md covers — opening doors to institutional contracts that mandate LEED, CIMS, or OSHA compliance documentation.

### Tasks

#### 13.1 — Chemical product registry + SDS management

**What**: Models and APIs for managing the chemical inventory and Safety Data Sheets.

**Design**:
- Models from `data-model-suggestion-1.md` lines 560-598.
- Endpoints: `GET/POST /api/v1/chemicals`, `POST /chemicals/{id}/sds` (multipart PDF upload).
- SDS stored in S3 with virus scan via ClamAV (sidecar in docker-compose).
- Revision date validation: warn if SDS older than 3 years (OSHA expectation).

**Testing**:
- `Integration: upload SDS PDF → sds_storage_key populated`
- `Integration: SDS revision >3 years old → returned with `is_stale=true` in API response`

#### 13.2 — Product certification tracking

**What**: Per-product certification records (EPA Safer Choice, LEED, Green Seal, EcoLogo).

**Design**:
- Model from `data-model-suggestion-1.md` lines 578-586.
- Endpoint: `POST /chemicals/{id}/certifications`.
- Auto-flag products expiring within 90 days.

**Testing**:
- `Integration: expiring-soon products listed by /chemicals?expiring_in=90`

#### 13.3 — Job product usage tracking

**What**: Capture which chemicals were used on which job, for compliance audit.

**Design**:
- Model from `data-model-suggestion-1.md` lines 588-598.
- Endpoint `POST /jobs/{id}/product-usage` recorded by cleaner.
- Mobile app: optional barcode/QR scan of product → preselects.

**Testing**:
- `Integration: recording usage on LEED-certified property with non-Safer-Choice product → warning in response`

#### 13.4 — CIMS compliance dashboard

**What**: Pre-loaded CIMS sections and elements; per-tenant compliance records; dashboard.

**Design**:
- Models from `data-model-suggestion-1.md` lines 628-662.
- Seed migration loads the full CIMS framework (5 sections, ~150 elements with mandatory flags).
- Endpoint `GET /api/v1/compliance/cims` returns scorecard per section + per-element status.
- Operator-facing UI: checklist, attach evidence (file upload to S3), reviewer assignment.

**Testing**:
- `Integration: seed migration creates 5 cims_section rows and 150+ cims_element rows`
- `Integration: marking elements compliant updates dashboard percentages`
- `Snapshot: dashboard JSON matches expected shape for a half-complete tenant`

#### 13.5 — Compliance reports (LEED, CIMS, OSHA)

**What**: One-click PDF generation for compliance reviews.

**Design**:
- `GET /api/v1/compliance/reports/leed?property_id=&period=` → PDF listing product usage with certification status.
- `GET /api/v1/compliance/reports/cims` → PDF audit-ready CIMS scorecard.
- `GET /api/v1/compliance/reports/osha-sds-inventory` → CSV of all chemicals with SDS storage links.

**Testing**:
- `Integration: LEED report for property with mixed product usage flags non-compliant items`
- `Manual: review generated PDFs against ISSA CIMS sample format`

### Definition of Done
A commercial operator can complete a CIMS self-audit in the platform; LEED report generation produces auditor-ready output; SDS expiry alerts fire on schedule.

---

## Phase 14: Hardening — Security, Performance, i18n, Deployment

### Purpose
Production-readiness pass: security audit against OWASP, performance optimisation, comprehensive i18n, observability, Helm chart, and the public/cloud deployment story.

### Tasks

#### 14.1 — OWASP ASVS Level 2 audit

**What**: Systematic verification against OWASP Application Security Verification Standard.

**Design**:
- Checklist coverage for: V2 Authentication, V3 Session Management, V4 Access Control, V5 Validation, V7 Error Handling/Logging, V8 Data Protection, V9 Communication, V13 API.
- Automated tools: Bandit (Python), Semgrep, OWASP ZAP baseline scan in CI.
- Remediate findings; document deviations.

**Testing**:
- `CI: bandit and semgrep run on every PR; high-severity blocks merge`
- `Integration: OWASP ZAP baseline scan against staging passes (no high/medium)`

#### 14.2 — Performance budget + load tests

**What**: Set and verify performance SLOs.

**Design**:
- SLOs: p95 API latency <250ms for read endpoints, <500ms for writes; calendar endpoint <300ms for 1k-job tenant.
- Locust scripts simulating: 50 concurrent cleaner clock-ins; 100 simultaneous booking submissions; 10 concurrent route optimisations.
- Identify and fix hot paths (N+1, missing indexes, oversized payloads).

**Testing**:
- `Bench: locust scenario 'cleaner_clock_in' achieves SLO with 50 concurrent`
- `Bench: profile of /calendar endpoint shows <300ms p95`

#### 14.3 — i18n completion

**What**: All user-facing strings extracted; Spanish translations for cleaner app and client portal complete; reporting templates localised.

**Design**:
- Backend uses `gettext` with `.po` files per language; CI extracts new strings via `pybabel extract`.
- Frontend uses `next-intl` and `i18n-js`.
- Translation review workflow via `.po` file PRs.

**Testing**:
- `Unit: every user-facing exception message has translation entry`
- `Manual: cleaner app fully functional in es-MX locale`

#### 14.4 — Observability + alerting

**What**: OpenTelemetry tracing + Prometheus metrics + Sentry error tracking + a base Grafana dashboard.

**Design**:
- FastAPI OTel instrumentation; Celery task tracing.
- Metrics: `http_request_duration_seconds`, `celery_task_duration_seconds`, `db_pool_size`, `webhook_delivery_attempts_total`, `llm_tokens_total`.
- Sentry SDK with `release` tagging; PII scrubbing for emails/phones.
- Alert rules: 5xx rate >1% over 5m; webhook delivery success <95% over 1h; DB pool exhaustion.

**Testing**:
- `Integration: triggering a 500 → Sentry event captured (sandboxed)`
- `Integration: Prometheus /metrics endpoint serves all expected series`

#### 14.5 — Helm chart + cloud bootstrap docs

**What**: Production-grade Helm chart and an opinionated cloud deployment guide.

**Design**:
- Helm chart sub-charts: api, worker, beat, web, plus dependencies (Bitnami postgres, redis, minio).
- Values for: replica counts, ingress, TLS via cert-manager, autoscaling (HPA on CPU/RPS).
- Documentation in `docs/deployment.md` for: docker-compose self-host, single-VM, k8s cloud, plus rotation procedures (DB backups, secret rotation).

**Testing**:
- `Integration (kind cluster): helm install renders and pods come ready in <5min`
- `Documentation review: smoke-test the docker-compose path on a fresh machine`

#### 14.6 — Backup, restore, disaster recovery

**What**: Documented and tested backup procedure.

**Design**:
- Postgres logical backups via `pg_dump` to S3 daily; PITR via WAL archiving optional.
- MinIO/S3 object lifecycle: photos retained indefinitely, breadcrumbs after 90 days.
- Restore drill documented; CLI command `cleaning-platform backup verify` checks last backup integrity.

**Testing**:
- `Integration: backup → restore to clean DB → smoke test API`
- `Manual: documented restore procedure executed by a second developer in <30min`

### Definition of Done
Platform passes a third-party security review (or internal equivalent); load tests meet SLOs; deployment can be replicated by a new operator following docs alone; observability dashboard surfaces every key metric.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Multi-Tenant Skeleton
    │
Phase 2: Identity, Auth & RBAC ─── requires 1
    │
Phase 3: Core Domain (Clients, Cleaners, Services) ─── requires 2
    │
Phase 4: Scheduling Engine ─── requires 3
    ├── Phase 5: Mobile + GPS + Photos ───────── requires 4 (can parallel with 6, 7, 8)
    ├── Phase 6: Communications ──────────────── requires 3 (can parallel with 5, 7, 8)
    ├── Phase 7: Booking Widget + Client Portal ─ requires 4, 6
    └── Phase 8: Billing + Stripe + QBO ──────── requires 3, 6 (can parallel with 5, 7)
         │
Phase 9: Quality Inspection ──── requires 5 (mobile + photos)
    │
Phase 10: Public API + Webhooks ─ requires 4, 5, 6, 8, 9 (any subset can ship)
    │
Phase 11: Route Optimisation ─── requires 4 (can parallel with 9, 10)
    │
Phase 12: AI Layer ───────────── requires 6 (comms), 9 (vision), 4 (pricing); incremental
    │
Phase 13: Compliance ─────────── requires 3 (chemicals), 9 (inspections); independent of 11/12
    │
Phase 14: Hardening ──────────── requires all prior phases for full coverage
```

Parallelism opportunities:
- After Phase 4: Phases 5, 6, and 8 can be developed by separate developers.
- Phase 11 (routing) and Phase 13 (compliance) are largely independent and can ship in either order after Phase 9.
- AI sub-features in Phase 12 (12.1 → 12.2/12.3/12.4/12.5) can be parallelised once the LLM client (12.1) is in place.

---

## Definition of Done (per phase)

Every phase must satisfy this checklist before being marked complete:

1. All tasks in the phase are implemented and merged to `main`.
2. All new unit, integration, and e2e tests pass; coverage on new code ≥ 85%.
3. `ruff check` and `ruff format --check` are clean.
4. `mypy --strict` passes on all `src/cleaning_platform/` code.
5. Bandit and Semgrep checks are clean (no new high/medium findings).
6. `docker compose up` builds and runs successfully end-to-end with the new functionality demonstrable.
7. New API endpoints appear in the auto-generated OpenAPI 3.1 spec with examples, descriptions, and tags.
8. New database changes ship as Alembic migrations; `alembic upgrade head` then `downgrade base` then `upgrade head` is clean.
9. New configuration options are documented in `.env.example` and `docs/developer-guide.md`.
10. New webhooks / events are documented in `docs/events.md` (AsyncAPI spec, per `standards.md`).
11. New user-facing strings are extracted to `.po` files with English source strings.
12. A short manual smoke-test script for the phase's primary user journey is committed under `docs/smoke-tests/phase-N.md`.
13. The CHANGELOG and (if user-facing) the project README are updated.
