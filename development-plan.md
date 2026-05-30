# Fleet Management Platform — Phased Development Plan

> Project: 238-fleet-management-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan turns the research, feature survey, standards alignment, and data-model proposals into a sequenced, buildable specification for an AI-native, self-hostable, open-source fleet management platform. The plan targets the MVP scope from `features.md` first (real-time GPS, geofencing, maintenance, driver behaviour, mobile driver app, REST API, RBAC) then layers on the v1.1 capabilities (predictive maintenance, NL query, fuel/ELD/EV/IFTA, multi-fleet hierarchy) and selected backlog items.

The chosen data foundation follows **Data Model 1 — Entity-Centric Normalised Relational** (PostGIS + range-partitioned telemetry tables) because it maps cleanly to REST resources, is approachable for contributors, supports compliance reporting out of the box, and can later be augmented with the event/CQRS layer from Model 2 if the scale demands it.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary backend language | Python 3.12 | Best ecosystem for the AI/ML differentiators (predictive maintenance, anomaly detection, NL query, embeddings) called out in `research.md` and the AI-native section of `README.md`. Strong libraries for spatial data (Shapely, GeoAlchemy2), telematics protocol parsing, and LLM orchestration. |
| API framework | FastAPI | Async by default (critical for telemetry ingestion throughput), first-class OpenAPI 3.1 generation aligning to `standards.md`, Pydantic v2 validation, and WebSocket support for live tracking. |
| ASGI server | Uvicorn behind Gunicorn workers | Production-grade async server stack used by FastAPI's own deployment guides; Gunicorn supervises Uvicorn workers for resilience. |
| Database | PostgreSQL 16 + PostGIS 3.4 + TimescaleDB 2.x | Postgres is the lingua franca of open-source platforms; PostGIS gives `GEOGRAPHY(POINT/POLYGON, 4326)` queries required for geofencing; TimescaleDB hypertables (from Data Model 2) deliver the high-throughput ingestion path for `telemetry_position`/`telemetry_sensor` without changing the rest of the schema. |
| ORM / migrations | SQLAlchemy 2 (async) + Alembic | SQLAlchemy 2 has native async support; Alembic handles schema and hypertable migrations. |
| Spatial libraries | GeoAlchemy2, Shapely | GeoAlchemy2 exposes PostGIS types through SQLAlchemy; Shapely handles GeoJSON ↔ WKT conversion in app code. |
| Background jobs / queue | Celery 5 + Redis 7 | Celery handles asynchronous workloads: telemetry batch processing, IFTA quarterly aggregation, ML inference, webhook delivery, scheduled maintenance checks. Redis doubles as result backend, broker, and short-lived cache. |
| Realtime layer | FastAPI WebSocket + Redis Pub/Sub | Pushes live vehicle positions and alerts to dashboards; Redis Pub/Sub fans out events from telemetry workers to all connected WebSocket sessions. |
| Telematics ingestion gateway | Standalone asyncio TCP/UDP server (`fleet_ingest`) supporting Traccar protocol family | Reuses the Traccar lineage (2,000+ device models, Apache-licensed protocol docs) so the platform is device-agnostic on day one without re-implementing every protocol. |
| Authentication | OAuth 2.0 + OpenID Connect via Authlib; password + magic link for first-party login; API keys for service accounts | Aligns with `standards.md` (OAuth 2.0 RFC 6749, OIDC). Supports SSO for enterprise buyers and machine-to-machine integrations for fleet hardware. |
| Authorisation | Casbin (RBAC with org + fleet scoping) | Models the role/permission/scope structure described in Data Model 1 declaratively; supports per-tenant policies. |
| Frontend | Next.js 15 (App Router) + React 19 + TypeScript 5 | Modern React stack for the operator dashboard; SSR-capable for SEO of public marketing pages; mature `react-map-gl` + MapLibre integration for live maps. |
| Component library | shadcn/ui + Tailwind CSS | Composable, accessible primitives — common across modern fleet dashboards (Samsara/Motive style KPI cards, drill-down panels). |
| Map rendering | MapLibre GL JS + OpenStreetMap tiles (self-hostable via `openmaptiles` Docker image) | Avoids per-call Google/HERE costs called out in `standards.md` as a structural cost advantage for an open-source platform. |
| Routing engine | OSRM (Open Source Routing Machine) | Self-hostable, OSM-based routing for ETA, route playback, and dynamic route optimisation; matches OSM tile strategy. |
| Mobile driver app | React Native + Expo (managed workflow), TypeScript | Single codebase for iOS/Android; Expo modules cover GPS, camera (DVIR photos), background tasks, push notifications. Shares Pydantic-generated TypeScript types with the web frontend. |
| LLM provider abstraction | LiteLLM | Routes prompts to OpenAI, Anthropic, or self-hosted models (Ollama/vLLM). Self-hosters can run fully offline; SaaS can use commercial models. |
| Vector store (for NL query / RAG) | pgvector extension on the same Postgres instance | Avoids adding another datastore for the v1.1 natural-language reporting feature; embeds vehicle/driver/work-order summaries. |
| ML framework | scikit-learn for v1.1 baselines; PyTorch reserved for v2 dashcam analytics | scikit-learn (gradient-boosted models, isolation forests) is sufficient for predictive maintenance and fuel-anomaly detection MVP. PyTorch deferred until video pipelines are needed. |
| Testing framework | pytest, pytest-asyncio, pytest-postgresql, Playwright (web), Detox (mobile) | Standard Python testing tools plus E2E web/mobile coverage. |
| Test fixtures / factories | factory_boy + Faker | Generate realistic vehicle/driver/trip fixtures. |
| Code quality | Ruff (lint + format), Mypy --strict, ESLint + Prettier + tsc for frontend, ktlint not needed | Single Python tool (Ruff) for lint+format; Mypy strict catches API contract bugs early. |
| Package management | uv (Python), pnpm (JavaScript) | uv is the fastest modern Python resolver; pnpm is efficient for the monorepo's multiple frontends. |
| Monorepo tooling | Turborepo for the JS workspaces; uv workspaces for Python packages | Turborepo caches frontend builds; uv workspaces share code between API, ingest gateway, and workers. |
| Containerisation | Docker + docker-compose for dev; Helm chart for production | Self-host deployers expect docker-compose. A Helm chart is the standard for Kubernetes deployments at fleet scale. |
| CI/CD | GitHub Actions | Standard for open-source projects; matrix builds for Python + Node. |
| Observability | OpenTelemetry SDK → Grafana + Tempo + Loki; Prometheus for metrics | Open standards stack; fleet operators self-host. |
| API documentation | FastAPI's autogenerated OpenAPI 3.1 + Redoc + Stoplight Studio for design-time editing | Satisfies the `standards.md` requirement that "a new fleet management platform should publish an OAS 3.1 spec for all its API endpoints". |
| MCP server | `fleet-mcp` package implementing the Model Context Protocol per `standards.md` | Exposes fleet entities and operations as MCP resources/tools so any MCP-compatible AI client can query and act on fleet data. |
| Licence | Apache 2.0 (TBD per `README.md`) | Matches Traccar/OpenGTS precedent; commercially friendly for self-hosters. |

### Project Structure

```
fleet-management-platform/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── docker-compose.yml                 # dev stack: postgres+postgis+timescale, redis, osrm, maptiles
├── docker-compose.prod.yml
├── Makefile                           # common dev commands (make up, make test, make migrate)
├── pyproject.toml                     # uv workspace root
├── uv.lock
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
├── .github/
│   └── workflows/
│       ├── ci-backend.yml
│       ├── ci-frontend.yml
│       └── release.yml
├── deploy/
│   ├── helm/
│   │   └── fleet-management/          # Helm chart for k8s
│   └── docker/
│       ├── api.Dockerfile
│       ├── ingest.Dockerfile
│       ├── worker.Dockerfile
│       └── web.Dockerfile
├── docs/
│   ├── architecture.md
│   ├── api/
│   │   └── openapi.yaml               # generated; committed for diffability
│   └── runbooks/
├── packages/
│   ├── fleet_core/                    # Python: domain models, shared types, errors
│   │   ├── pyproject.toml
│   │   └── src/fleet_core/
│   │       ├── domain/                # Pydantic v2 models (Vehicle, Driver, Trip, WorkOrder…)
│   │       ├── geo/                   # GeoJSON ↔ WKT, geofence point-in-polygon helpers
│   │       ├── compliance/            # HOS calculators, IFTA distance attribution
│   │       └── events/                # internal event bus contracts
│   ├── fleet_db/                      # Python: SQLAlchemy models, Alembic migrations
│   │   ├── pyproject.toml
│   │   └── src/fleet_db/
│   │       ├── models/
│   │       ├── repositories/
│   │       └── alembic/
│   ├── fleet_api/                     # FastAPI app: REST + WebSocket + webhooks
│   │   ├── pyproject.toml
│   │   └── src/fleet_api/
│   │       ├── main.py
│   │       ├── deps.py
│   │       ├── auth/                  # OAuth2, OIDC, API keys, Casbin policies
│   │       ├── routers/
│   │       │   ├── vehicles.py
│   │       │   ├── drivers.py
│   │       │   ├── trips.py
│   │       │   ├── geofences.py
│   │       │   ├── work_orders.py
│   │       │   ├── inspections.py
│   │       │   ├── fuel.py
│   │       │   ├── hos.py
│   │       │   ├── ifta.py
│   │       │   ├── ev.py
│   │       │   ├── alerts.py
│   │       │   ├── reports.py
│   │       │   ├── ai.py              # NL query, predictive maintenance endpoints
│   │       │   └── webhooks.py
│   │       └── ws/
│   │           ├── positions.py
│   │           └── alerts.py
│   ├── fleet_ingest/                  # Telematics gateway: asyncio TCP/UDP/HTTP listeners
│   │   ├── pyproject.toml
│   │   └── src/fleet_ingest/
│   │       ├── server.py
│   │       ├── protocols/
│   │       │   ├── traccar_osmand.py
│   │       │   ├── traccar_teltonika.py
│   │       │   ├── traccar_concox.py
│   │       │   ├── obd2.py
│   │       │   └── j1939.py
│   │       └── publisher.py           # publishes to Redis Stream / DB
│   ├── fleet_workers/                 # Celery tasks
│   │   ├── pyproject.toml
│   │   └── src/fleet_workers/
│   │       ├── celery_app.py
│   │       ├── tasks/
│   │       │   ├── trip_aggregation.py
│   │       │   ├── geofence_detection.py
│   │       │   ├── safety_event_detection.py
│   │       │   ├── ifta_segmentation.py
│   │       │   ├── hos_violation_checker.py
│   │       │   ├── predictive_maintenance.py
│   │       │   ├── fuel_anomaly.py
│   │       │   ├── webhook_dispatch.py
│   │       │   └── nightly_safety_score.py
│   │       └── schedule.py            # celery beat schedule
│   ├── fleet_ai/                      # ML + LLM glue
│   │   ├── pyproject.toml
│   │   └── src/fleet_ai/
│   │       ├── nl_query/              # SQL generator with guardrails
│   │       ├── predictive_maintenance/
│   │       ├── anomaly/
│   │       └── embeddings/
│   ├── fleet_mcp/                     # Model Context Protocol server
│   │   ├── pyproject.toml
│   │   └── src/fleet_mcp/
│   │       ├── server.py
│   │       ├── resources/
│   │       └── tools/
│   └── fleet_sdk_ts/                  # Generated TypeScript client (from OpenAPI)
│       ├── package.json
│       └── src/
├── apps/
│   ├── web/                           # Next.js operator dashboard
│   │   ├── package.json
│   │   └── src/
│   │       ├── app/
│   │       │   ├── (auth)/login/
│   │       │   ├── (dashboard)/
│   │       │   │   ├── map/
│   │       │   │   ├── vehicles/
│   │       │   │   ├── drivers/
│   │       │   │   ├── work-orders/
│   │       │   │   ├── alerts/
│   │       │   │   ├── reports/
│   │       │   │   └── ai/             # NL query interface
│   │       │   └── api/
│   │       ├── components/
│   │       └── lib/
│   └── mobile-driver/                 # React Native + Expo driver app
│       ├── package.json
│       ├── app.json
│       └── src/
│           ├── screens/
│           │   ├── Login.tsx
│           │   ├── DvirChecklist.tsx
│           │   ├── TripList.tsx
│           │   ├── HosLog.tsx
│           │   └── Messages.tsx
│           └── services/
├── fixtures/                          # sample data for dev and tests
│   ├── seed.sql
│   ├── vehicles.json
│   ├── traccar_replay/                # captured protocol streams for ingest tests
│   └── work_orders.json
└── tests/
    ├── unit/                          # per-package unit tests live next to source
    ├── integration/                   # cross-package: API + DB + Redis
    └── e2e/                           # Playwright (web) and Detox (mobile) flows
```

---

## Phase 1: Foundation — Repository, Tooling, Database Bedrock

### Purpose
Stand up the monorepo with a runnable dev environment, CI, and a baseline database schema that subsequent phases can extend without restructuring. After this phase, a contributor can clone the repo, run `make up && make migrate`, and have a working Postgres+PostGIS+TimescaleDB+Redis stack with the organisation/auth tables in place.

### Tasks

#### 1.1 — Repository Bootstrap and Workspaces

**What**: Initialise the uv + pnpm + Turborepo monorepo with all package skeletons listed in the project structure, plus root tooling configs.

**Design**:
- `pyproject.toml` at the root declares a `[tool.uv.workspace]` block listing every `packages/*` and `apps/mobile-driver` (none — mobile is JS only) Python member.
- Each Python package has its own `pyproject.toml` with `requires-python = ">=3.12"`, declares `fleet_core` as a workspace dep where needed, and pins `ruff>=0.5`, `mypy>=1.10`, `pytest>=8`.
- Root `pnpm-workspace.yaml` lists `apps/*` and `packages/fleet_sdk_ts`.
- `turbo.json` defines `build`, `lint`, `test`, `dev` pipelines with explicit `dependsOn` and cache outputs.
- Root `Makefile` exposes: `make up`, `make down`, `make migrate`, `make seed`, `make api`, `make worker`, `make ingest`, `make web`, `make mobile`, `make test`, `make lint`, `make format`, `make openapi`.
- `.editorconfig`, `.gitignore`, `.gitattributes` with LFS for any large fixtures.
- Ruff config in root `pyproject.toml`: `line-length = 100`, all rules except `D` and `ANN101`, plus `mccabe` complexity threshold 10.
- Mypy config: `strict = true`, `disallow_untyped_decorators = false` (FastAPI decorators), `plugins = ["pydantic.mypy", "sqlalchemy.ext.mypy.plugin"]`.

**Testing**:
- Unit: `make lint` exits 0 on a fresh checkout.
- Unit: `uv sync` succeeds; `pnpm install` succeeds.
- Unit: `turbo run build --dry=json` lists every package once with no dependency cycles.
- CI: `.github/workflows/ci-backend.yml` runs `ruff check`, `mypy`, `pytest` per Python package matrix.
- CI: `.github/workflows/ci-frontend.yml` runs `pnpm lint`, `pnpm test`, `pnpm build` for JS workspaces.

#### 1.2 — Dev Stack via Docker Compose

**What**: Provide a `docker-compose.yml` that brings up every backing service required by the platform.

**Design**:
Services defined:
- `postgres`: `timescale/timescaledb-ha:pg16-all` (includes PostGIS), exposed on `5432`, persistent volume, environment seeded with `POSTGRES_DB=fleet`, `POSTGRES_USER=fleet`, `POSTGRES_PASSWORD=fleet`.
- `redis`: `redis:7-alpine`, exposed on `6379`.
- `osrm`: `osrm/osrm-backend:latest`, mounted volume `./.cache/osrm` with a Makefile target that downloads + extracts a small `monaco-latest.osm.pbf` for dev (so smoke tests don't require continent-sized data).
- `maptiles`: `maptiler/tileserver-gl-light`, serving a small MBTiles file for dev.
- `mailhog`: `mailhog/mailhog` for capturing transactional emails during dev.
- `minio`: S3-compatible object store for dashcam clips, inspection photos, IFTA report PDFs.
- All services live on a `fleet-net` user-defined bridge network with healthchecks (`pg_isready`, `redis-cli ping`).

`docker-compose.prod.yml` overrides image tags, removes the dev-only `mailhog`/`tileserver`, and externalises secrets via env files.

**Testing**:
- Integration: `make up && make health` confirms every service responds (pg_isready, redis ping, OSRM `/route/v1/driving/lat,lng;lat,lng` returns 200, tileserver returns a `.pbf` tile).
- Unit: parse compose file, assert each service has a healthcheck and is connected to `fleet-net`.

#### 1.3 — Database Schema Foundation and Alembic

**What**: Create the initial Alembic migration that installs PostGIS + TimescaleDB extensions and creates the organisation/user/role/permission/jurisdiction tables.

**Design**:
- Alembic env configured for async SQLAlchemy with `compare_type = True` and `include_object` hook that excludes TimescaleDB internal `_timescaledb_*` schemas.
- Migration `0001_extensions_and_org_auth.py` runs:
  ```sql
  CREATE EXTENSION IF NOT EXISTS postgis;
  CREATE EXTENSION IF NOT EXISTS timescaledb;
  CREATE EXTENSION IF NOT EXISTS pgvector;
  CREATE EXTENSION IF NOT EXISTS pg_trgm;
  ```
- Creates the tables from Data Model 1's "Organisations and Tenancy" + "Driver Management" sections (excluding driver-vehicle assignments — those come in Phase 2): `organisation`, `app_user`, `role`, `permission`, `role_permission`, `user_role`, `jurisdiction`.
- Seeds `jurisdiction` from a committed `fixtures/jurisdictions.csv` containing all ISO 3166-2 subdivisions with IFTA membership and 2026 fuel-tax rates (loaded via a separate `0002_seed_jurisdictions.py` data migration).
- Seeds `permission` rows for every `(resource, action)` combination listed in section 3D below (single `0003_seed_permissions.py`).
- Seeds two system roles per new org via an SQLAlchemy `after_insert` hook on `organisation`: `admin` (all permissions) and `viewer` (read-only).

SQLAlchemy models in `packages/fleet_db/src/fleet_db/models/`:
```python
class Organisation(Base):
    __tablename__ = "organisation"
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    name: Mapped[str] = mapped_column(String(255))
    slug: Mapped[str] = mapped_column(String(100), unique=True)
    parent_org_id: Mapped[UUID | None] = mapped_column(ForeignKey("organisation.id"))
    dot_number: Mapped[str | None] = mapped_column(String(20))
    mc_number: Mapped[str | None] = mapped_column(String(20))
    ifta_account: Mapped[str | None] = mapped_column(String(30))
    timezone: Mapped[str] = mapped_column(String(50), default="UTC")
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
```

**Testing**:
- Integration (real Postgres via pytest-postgresql): `alembic upgrade head` succeeds on an empty DB.
- Integration: `alembic downgrade base` then `upgrade head` is idempotent and produces an identical schema (compared via `pg_dump --schema-only` diff).
- Unit: `Organisation(name="Acme", slug="acme")` round-trips through SQLAlchemy and Pydantic schema.
- Unit: every Permission row defined in code is present after migration seeding.

#### 1.4 — Authentication and Multi-Tenant Context

**What**: Implement password + magic-link auth, OIDC SSO, API keys, and a request-scoped tenant context dependency.

**Design**:
- `packages/fleet_api/src/fleet_api/auth/`:
  - `passwords.py`: argon2-cffi for hashing.
  - `tokens.py`: JWT (HS256 in dev, RS256 in prod with a JWK set under `/.well-known/jwks.json`). Claims include `sub` (user UUID), `org` (organisation UUID), `roles` (string list), `scopes` (list of `fleet:<uuid>` constraints).
  - `oidc.py`: Authlib OIDC client; supports Google, Microsoft, Okta, generic.
  - `api_keys.py`: opaque, 32-byte tokens prefixed `flk_`, stored hashed; one-time reveal on creation.
  - `dependencies.py`: FastAPI deps `get_current_user`, `require_permission("vehicle", "read")`, `get_org_id`, `get_fleet_scope`.
- `app_user.password_hash` stores Argon2; magic links are 32-byte tokens with 15-minute TTL in Redis.
- Multi-tenant safety: every repository function takes `org_id` as the first argument and applies `WHERE organisation_id = :org_id`; SQLAlchemy `before_compile` event raises if a query against an org-scoped model omits the filter.

Endpoint surface:
```
POST   /v1/auth/login                 { email, password } → { access_token, refresh_token }
POST   /v1/auth/magic-link            { email } → 202
GET    /v1/auth/magic-link/{token}    → { access_token, refresh_token }
POST   /v1/auth/refresh               { refresh_token } → { access_token }
POST   /v1/auth/logout                → 204
GET    /v1/auth/oidc/{provider}/start → 302
GET    /v1/auth/oidc/{provider}/callback?code=... → { access_token, refresh_token }
POST   /v1/auth/api-keys              { name, scopes } → { id, name, key } (one-time reveal)
GET    /v1/auth/api-keys              → [ { id, name, last_used_at } ]
DELETE /v1/auth/api-keys/{id}         → 204
GET    /v1/me                         → { user, organisation, roles, permissions }
```

**Testing**:
- Unit: password hash and verify; magic link generation, storage in fake Redis, expiry after 15 min.
- Unit: JWT issuance includes all expected claims; tampered token rejected.
- Integration (mocked OIDC provider): full OIDC code-exchange flow returns access token.
- Integration: API key creation reveals key once; subsequent fetches return only metadata.
- Integration: a request authenticated as Org A user cannot read Org B vehicles (returns 404, never 403, to avoid existence disclosure).
- Security: brute-force protection — 5 failed logins in 10 minutes returns 429 (uses Redis sliding-window counter).
- OWASP API Top 10 mapping: explicit test coverage for API1 (BOLA), API2 (broken auth), API3 (broken object property level auth), API5 (broken function level auth).

---

## Phase 2: Fleet Core — Vehicles, Drivers, Fleets, Devices

### Purpose
Add the CRUD surface for the entities every fleet operator manipulates daily. After this phase, an operator can sign up, create a fleet, add vehicles and drivers, register telematics devices, and assign drivers to vehicles. No telemetry data flows yet; that's Phase 3.

### Tasks

#### 2.1 — Vehicle, Fleet, and Lookup Tables

**What**: Create the vehicle, fleet, vehicle_make, vehicle_model tables and a full CRUD REST surface for fleets and vehicles.

**Design**:
- Alembic `0010_fleet_and_vehicles.py` adds the relational tables from Data Model 1's "Fleet and Vehicle Management" section verbatim, scoped by `organisation_id`.
- `vehicle_make` and `vehicle_model` are seeded from `fixtures/vehicle_makes.csv` (covering the top 200 commercial and light-duty makes/models). Operators may add their own custom entries (`is_custom BOOLEAN` flag).
- Pydantic schemas mirror the SQLAlchemy models with separate `Create`, `Update`, `Public` variants.

REST endpoints (every endpoint is org-scoped via the JWT claim):
```
POST   /v1/fleets                              → 201 { fleet }
GET    /v1/fleets                              → { items, next_cursor }
GET    /v1/fleets/{id}                         → { fleet }
PATCH  /v1/fleets/{id}                         → { fleet }
DELETE /v1/fleets/{id}                         → 204
POST   /v1/vehicles                            → 201 { vehicle }
GET    /v1/vehicles?fleet_id=&status=&q=       → cursor-paginated
GET    /v1/vehicles/{id}                       → { vehicle }
PATCH  /v1/vehicles/{id}                       → { vehicle }
DELETE /v1/vehicles/{id}                       → 204 (soft-delete: status=retired)
POST   /v1/vehicle-makes                       → 201 (custom make)
GET    /v1/vehicle-makes                       → list (system + custom merged)
GET    /v1/vehicle-models?make_id=             → list
```

Listing uses keyset pagination on `(created_at DESC, id DESC)`; cursor is a base64-encoded `{created_at, id}` tuple.

**Testing**:
- Unit: Pydantic validation rejects VIN length ≠ 17 with field-name in error.
- Unit: Repository `list_vehicles(org_id, cursor=None, limit=50, filters={...})` returns expected ordering.
- Integration: end-to-end vehicle CRUD round-trip through API + DB.
- Integration: cross-org isolation — Org A's cursor cannot be used to read Org B's vehicles.
- Integration: filter combinations — `fleet_id` + `status=active` + `q="freightliner"` returns expected subset.
- Fixture-based: seed 1,000 vehicles, assert pagination yields all records without duplicates.

#### 2.2 — Driver Management and Vehicle Assignment

**What**: Add `driver` and `driver_vehicle_assignment` tables, plus REST CRUD and the temporal assignment endpoints.

**Design**:
- Schema from Data Model 1's "Driver Management" section, but with a small extension: `driver.preferred_locale CHAR(5) DEFAULT 'en-US'` for mobile app localisation.
- `driver_vehicle_assignment` is append-only: assignments are never updated; reassignment closes the existing row (`unassigned_at = now()`) and creates a new one in a single transaction.
- Helper repository `current_driver_for_vehicle(vehicle_id)` returns the row where `unassigned_at IS NULL`; `current_vehicle_for_driver(driver_id)` symmetric.

REST endpoints:
```
POST   /v1/drivers                                            → 201
GET    /v1/drivers?status=&q=                                 → cursor-paginated
GET    /v1/drivers/{id}                                       → { driver, current_vehicle }
PATCH  /v1/drivers/{id}                                       → { driver }
DELETE /v1/drivers/{id}                                       → 204 (soft-delete via status)
POST   /v1/vehicles/{vehicle_id}/assign                       { driver_id, is_primary }  → 201
POST   /v1/vehicles/{vehicle_id}/unassign                     → 204
GET    /v1/vehicles/{vehicle_id}/assignments?from=&to=        → historical list
GET    /v1/drivers/{driver_id}/assignments?from=&to=          → historical list
```

**Testing**:
- Unit: assignment lifecycle — assign A, then assign B closes A and opens B atomically.
- Unit: `current_driver_for_vehicle` returns None when no active assignment.
- Integration: `GET /vehicles/{id}/assignments?from=2026-01-01&to=2026-06-30` returns chronological list.
- Integration: cannot assign a driver from another org to a vehicle → 422 with `driver_id_not_in_organisation` error code.
- Property-based (Hypothesis): random sequences of assign/unassign always leave at most one active assignment per vehicle.

#### 2.3 — Telematics Device Registry

**What**: Create the `telematics_device` table and CRUD endpoints; device-to-vehicle binding/unbinding maintained as a state machine.

**Design**:
- Schema from Data Model 1 "telematics_device" verbatim; `device_type` enum: `gps_tracker | obd2 | j1939 | eld | dashcam | asset_tracker`.
- `status` state machine: `provisioned → installed → active → maintenance → retired`. Transitions validated in `device_repository.transition_status(device_id, new_status, actor_user_id)`.
- A "registration token" flow lets installers attach hardware in the field: an admin generates a `device_registration_code` (8-digit, 1-hour TTL, single-use, scoped to a vehicle) which the device sends in its first heartbeat to be auto-bound.

REST endpoints:
```
POST   /v1/devices                                       → 201 (status=provisioned)
GET    /v1/devices?status=&type=&unbound=true           → list
GET    /v1/devices/{id}                                  → { device, vehicle }
PATCH  /v1/devices/{id}                                  → 200
POST   /v1/devices/{id}/bind                             { vehicle_id }  → 200 (status=installed)
POST   /v1/devices/{id}/unbind                           → 200
POST   /v1/devices/{id}/transition                       { to_status }   → 200
POST   /v1/vehicles/{vehicle_id}/registration-code       → 201 { code, expires_at }
```

**Testing**:
- Unit: state machine — invalid transition `installed → provisioned` raises with `from_status` and `to_status` in error.
- Unit: registration code expires after 1 hour; single-use enforced.
- Integration: bind, then bind again → 409 `device_already_bound`.
- Integration: unbinding cascades a "last seen" snapshot but does not delete history.

#### 2.4 — RBAC Wiring and Audit Log

**What**: Bring Casbin online with the permissions seeded in Phase 1, then write every mutating API call to the audit log.

**Design**:
- Casbin model file `casbin/rbac_with_scope.conf` defines:
  ```
  [request_definition]
  r = sub, obj, act, org, fleet
  [policy_definition]
  p = sub, obj, act, org, fleet
  [matchers]
  m = g(r.sub, p.sub, r.org) && r.obj == p.obj && r.act == p.act && (p.fleet == "*" || r.fleet == p.fleet)
  ```
- Policies are loaded from the DB on app startup and on `permission_changed` Redis Pub/Sub events.
- FastAPI dependency `RequirePermission(obj, act, fleet_param: str | None = None)` extracts the fleet scope from path/query and calls Casbin.
- Audit log middleware logs every non-`GET` request with `organisation_id`, `user_id`, `action`, `resource_type`, `resource_id`, request body diff (when applicable), IP, user agent. Writes are async via a Celery task so they don't block the request.
- `audit_log` table from Data Model 1 made into a TimescaleDB hypertable partitioned by `occurred_at` (1-month chunks).

**Testing**:
- Unit: Casbin policy evaluation — `viewer` cannot create vehicles; `admin` can.
- Unit: fleet-scoped role — user with `role.scope = fleet:abc` can edit vehicles where `fleet_id = abc` but returns 404 for other fleets.
- Integration: `PATCH /v1/vehicles/{id}` produces an audit log row with `changes = {field: {old, new}}`.
- Integration: audit log respects org boundary — Org A admins cannot query Org B audit rows.
- Performance: audit middleware adds < 5 ms to request latency at p95 (measured with `pytest-benchmark`).

---

## Phase 3: Telematics Ingestion and Live Tracking

### Purpose
Connect real (or simulated) GPS hardware to the platform. After this phase, vehicles report positions over the network, those positions are persisted to time-series tables, derived into trips, and streamed live to operator dashboards over WebSocket. This is the heart of the product.

### Tasks

#### 3.1 — Ingest Gateway Architecture

**What**: Build `fleet_ingest`, an asyncio process that listens on TCP/UDP/HTTP for telematics protocols and publishes parsed observations onto a Redis Stream.

**Design**:
- Process lifecycle: `python -m fleet_ingest` reads `INGEST_LISTEN` env var (e.g., `tcp://0.0.0.0:5001:teltonika,tcp://0.0.0.0:5002:concox,udp://0.0.0.0:5055:osmand,http://0.0.0.0:8080:traccar_http`).
- One asyncio task per protocol per port; each task uses a `Protocol` ABC:
  ```python
  class Protocol(Protocol):
      name: str
      async def handle(self, reader: StreamReader, writer: StreamWriter) -> None: ...
      async def parse_frame(self, raw: bytes) -> Observation | None: ...
  ```
- Initial protocols implemented in Phase 3: OsmAnd (HTTP GET, used by the Traccar mobile app for testing), Teltonika Codec 8 (covers the FMB series — widely deployed), Concox (covers GT06N-style devices), generic OBD-II via Bluetooth bridge (HTTP POST).
- `Observation` dataclass:
  ```python
  @dataclass
  class Observation:
      device_serial: str
      protocol: str
      recorded_at: datetime
      latitude: float
      longitude: float
      altitude_m: float | None
      speed_kmh: float | None
      heading: float | None
      ignition_on: bool | None
      odometer_km: float | None
      engine_hours: float | None
      satellites: int | None
      hdop: float | None
      raw_attributes: dict[str, Any]   # protocol-specific fields preserved
  ```
- Publisher writes each observation to Redis Stream `telemetry:positions` with `MAXLEN ~ 1_000_000`. A separate worker reads the stream and writes to TimescaleDB in batches of 500 or every 250 ms (whichever first).
- Device-to-vehicle resolution: gateway maintains an in-process LRU cache of `device_serial → device_id, vehicle_id, organisation_id` populated from `telematics_device`; cache invalidated by Redis Pub/Sub `device:rebind:{serial}` on bind/unbind.
- Devices reporting an unknown serial are routed to a holding queue (`telemetry:unknown_devices`) and surfaced to admins for assignment.

**Testing**:
- Unit: parse a captured Teltonika Codec 8 frame from `fixtures/traccar_replay/teltonika_codec8.bin` → expected `Observation` fields.
- Unit: parse a Concox GT06N login + GPS data packet sequence → correct ACK responses sent back.
- Unit: parse an OsmAnd HTTP GET `?id=device123&lat=...&lon=...&timestamp=...` → expected `Observation`.
- Integration (real Redis): publish 10,000 observations through the gateway, batch worker drains the stream within 30 s.
- Integration: unknown device serial lands in `telemetry:unknown_devices` and is surfaced via `GET /v1/devices/unknown`.
- Load: 1,000 simulated devices each sending 1 position/sec for 60 s sustains < 200 ms p99 publish latency.

#### 3.2 — TimescaleDB Telemetry Tables

**What**: Add the time-series tables for positions, sensors, and fault codes, and the batch writer that drains the Redis Stream into them.

**Design**:
- Alembic `0020_telemetry_hypertables.py` runs:
  ```sql
  CREATE TABLE telemetry_position (
      time TIMESTAMPTZ NOT NULL,
      vehicle_id UUID NOT NULL,
      device_id UUID NOT NULL,
      organisation_id UUID NOT NULL,
      latitude NUMERIC(10,7) NOT NULL,
      longitude NUMERIC(10,7) NOT NULL,
      altitude_m NUMERIC(8,2),
      speed_kmh NUMERIC(6,2),
      heading NUMERIC(5,2),
      odometer_km NUMERIC(12,2),
      engine_hours NUMERIC(12,2),
      ignition_on BOOLEAN,
      satellites SMALLINT,
      hdop NUMERIC(4,2),
      location GEOGRAPHY(POINT, 4326) GENERATED ALWAYS AS
          (ST_SetSRID(ST_MakePoint(longitude::float, latitude::float), 4326)::geography) STORED
  );
  SELECT create_hypertable('telemetry_position', 'time', chunk_time_interval => INTERVAL '1 day');
  SELECT add_dimension('telemetry_position', 'vehicle_id', number_partitions => 16);
  ALTER TABLE telemetry_position SET (timescaledb.compress, timescaledb.compress_segmentby = 'vehicle_id');
  SELECT add_compression_policy('telemetry_position', INTERVAL '7 days');
  SELECT add_retention_policy('telemetry_position', INTERVAL '2 years');
  ```
- Equivalent migrations for `telemetry_sensor` and `telemetry_fault` from Data Model 2.
- Continuous aggregate `vehicle_hourly_stats` from Data Model 2 created here as well (used in Phase 4 and beyond).
- Batch writer in `fleet_workers/tasks/telemetry_writer.py`:
  - `redis.xreadgroup("ingest", "writer-{n}", "telemetry:positions", ">", count=500, block=250)`.
  - Acknowledges the batch only after `INSERT ... ON CONFLICT DO NOTHING` succeeds.
  - Uses `psycopg.copy()` for max throughput.
- After each successful insert, the writer publishes to Redis Pub/Sub channel `positions:{org_id}` a JSON `{vehicle_id, lat, lng, speed, heading, recorded_at}` for the WebSocket layer (Task 3.4).

**Testing**:
- Integration: write 100,000 positions across 100 vehicles; `SELECT count(*)` returns 100,000 and chunks are split across multiple TimescaleDB chunks.
- Integration: compression policy compresses chunks older than 7 days (simulate with `time_bucket_gapfill` and `decompress_chunk` round-trip).
- Integration: `vehicle_hourly_stats` continuous aggregate produces correct avg/max speed when queried 1 hour after ingestion.
- Performance: ingest 5,000 positions/sec sustained for 5 minutes without backlog growth.

#### 3.3 — Trip Derivation

**What**: A Celery beat task that converts contiguous position streams into `trip` rows.

**Design**:
- Algorithm (runs every 60 s per active vehicle, but coalesced via Redis Stream consumer):
  1. For each vehicle with recent activity, fetch positions since `last_processed_at` from `telemetry_position`.
  2. Segment into trips by gap heuristic: a new trip starts when `ignition_on` transitions false → true OR when the gap between consecutive positions exceeds 5 minutes AND the prior position has `speed_kmh < 5`.
  3. A trip ends symmetrically: ignition false transition, or 5-minute idle-and-stopped gap.
  4. For each completed trip, compute `distance_km` from cumulative haversine, `duration_seconds`, `max/avg speed`, `idle_duration_sec` (cumulative seconds at `speed=0 AND ignition_on=true`), and reverse-geocode start/end addresses via OSRM nearest endpoint + offline Nominatim.
  5. Insert a `trip` row; bookmark `last_processed_at` per vehicle in Redis.
- Trip insertions emit `TripCompleted` events on the Redis Pub/Sub channel `events:{org_id}`; downstream tasks (safety event detection, IFTA segmentation) listen.

REST endpoints:
```
GET /v1/trips?vehicle_id=&driver_id=&from=&to= → cursor-paginated
GET /v1/trips/{id}                              → { trip, positions: [...] }  (positions optional via ?include=positions)
GET /v1/trips/{id}/path                         → GeoJSON LineString
```

**Testing**:
- Unit: synthetic position stream with two distinct trips (gap > 5 min) → trip-segmenter returns two trips.
- Unit: a vehicle stationary at a depot for 10 minutes mid-trip does not get split (because ignition stayed on).
- Unit: reverse geocoding uses an in-process LRU cache so repeated lookups of the same coordinates hit OSRM at most once per hour.
- Integration: replay `fixtures/traccar_replay/route_dallas_to_houston.csv` (10,000 positions) → exactly 1 trip with expected `distance_km ± 1%`.
- Integration: `GET /v1/trips/{id}/path` returns valid GeoJSON validated against RFC 7946.

#### 3.4 — Live Tracking WebSocket

**What**: WebSocket endpoint streaming live vehicle positions to authenticated dashboards.

**Design**:
- Endpoint `WS /v1/ws/positions?fleet_id=`
- On connect: validate JWT from `Authorization` query parameter or `Sec-WebSocket-Protocol` subprotocol; subscribe the connection to Redis Pub/Sub `positions:{org_id}`; filter messages by the optional `fleet_id` and the user's RBAC fleet scope.
- Heartbeat: server sends `{"type":"ping"}` every 30 s; client expected to respond within 10 s or disconnect.
- Messages emitted to client:
  ```json
  {"type":"position","vehicle_id":"...","lat":..., "lng":..., "speed_kmh":..., "heading":..., "recorded_at":"..."}
  {"type":"trip_started","vehicle_id":"...","trip_id":"...","driver_id":"...","at":"..."}
  {"type":"trip_ended","vehicle_id":"...","trip_id":"...","at":"..."}
  ```
- Per-connection in-memory rate limit: max 50 messages/sec coalesced — newer position for the same `vehicle_id` replaces queued older ones.

**Testing**:
- Integration: connect a test client, publish 100 positions via Redis, assert all are received in order.
- Integration: connect without a valid JWT → 1008 close code.
- Integration: connect to org A but Redis publishes to org B → no leakage.
- Integration: connection survives a 60-second test with periodic pings.
- E2E: open the web dashboard map, replay a Traccar fixture, verify vehicle markers move in real time (Playwright).

---

## Phase 4: Geofencing, Alerts, and Driver Safety Events

### Purpose
Turn the raw telemetry stream into operational signal. After this phase, operators define geofences, configure alert rules, and receive notifications when vehicles cross boundaries, exceed speed thresholds, harsh-brake, or sit idle too long.

### Tasks

#### 4.1 — Geofence Authoring and Storage

**What**: CRUD for `geofence` with GeoJSON input/output.

**Design**:
- Schema from Data Model 1's `geofence` table; both `polygon` (PostGIS `GEOGRAPHY(POLYGON, 4326)`) and `circle` (`centre_point` + `radius_m`) shapes.
- API accepts and emits GeoJSON Feature with `geometry` and `properties` per RFC 7946:
  ```
  POST /v1/geofences
  {
    "name": "Dallas Depot",
    "geofence_type": "polygon",
    "geometry": { "type": "Polygon", "coordinates": [[[lng,lat], ...]] },
    "properties": { "speed_limit_kmh": 30, "colour": "#FF0000" }
  }
  ```
- Server validates polygons are valid + simple via `ST_IsValid` and `ST_IsSimple`; rejects with 422 and Shapely error message.
- Bulk operations: `POST /v1/geofences/import` accepts a GeoJSON FeatureCollection (≤ 500 features per request).

REST endpoints:
```
POST   /v1/geofences          → 201
GET    /v1/geofences?bbox=&active=true → FeatureCollection
GET    /v1/geofences/{id}     → Feature
PATCH  /v1/geofences/{id}     → Feature
DELETE /v1/geofences/{id}     → 204
POST   /v1/geofences/import   → { created: N, failed: [...] }
```

**Testing**:
- Unit: invalid (self-intersecting) polygon → 422 with field path.
- Unit: circle with negative radius → 422.
- Integration: `?bbox=lng_min,lat_min,lng_max,lat_max` returns only geofences whose envelope intersects the bbox (uses `ST_Intersects` on the GIST index).
- Integration: GeoJSON output round-trips through the GeoJSON spec validator without warnings.

#### 4.2 — Geofence Crossing Detector

**What**: A Celery task that detects enter/exit events as positions arrive.

**Design**:
- Subscriber to Redis Pub/Sub `positions:{org_id}` (the same stream as the WebSocket fan-out).
- For each position, fetches geofence membership at the prior position from a Redis hash `vehicle:{id}:geofences = {geofence_id: in|out}` and compares with the current point:
  ```sql
  SELECT id FROM geofence
   WHERE organisation_id = :org AND is_active = true
     AND ST_Intersects(boundary, ST_MakePoint(:lng, :lat)::geography);
  ```
- Differences against the previous set produce `geofence_event` rows with `event_type` `enter` or `exit` and emit downstream alerts.
- A coarse pre-filter uses a Redis-Geo index (`GEOADD`) of geofence centroids with radius so the SQL spatial query runs only against candidates within 50 km, keeping per-position cost bounded as the geofence count grows.

**Testing**:
- Unit: synthetic vehicle path crossing a single rectangular geofence in/out → exactly two `geofence_event` rows in correct order.
- Unit: vehicle entering at GPS jitter on the boundary (5 quick in/out flickers) → debounced to a single enter event (uses min-dwell-time 10 s).
- Integration: 50 vehicles × 1000 positions × 100 active geofences processes in < 30 s.
- Integration: deactivating a geofence stops emitting events for that geofence.

#### 4.3 — Alert Rule Engine

**What**: A user-configurable rule engine that produces `alert` rows and dispatches notifications.

**Design**:
- `alert_rule.conditions JSONB` validated against per-`alert_type` JSON Schemas committed under `packages/fleet_core/src/fleet_core/alerts/schemas/`. Schemas:
  - `speed`: `{ "max_speed_kmh": number, "duration_seconds": number, "geofence_id"?: uuid }`
  - `geofence`: `{ "geofence_id": uuid, "event_type": "enter|exit|both" }`
  - `idle`: `{ "max_idle_seconds": number, "ignition_on": boolean }`
  - `maintenance_due`: `{ "lookahead_km": number, "lookahead_days": number }`
  - `dtc`: `{ "codes": [string]?, "severity": "info|warning|critical" }`
  - `fuel_anomaly`: `{ "min_z_score": number }`  (Phase 6 hook)
  - `hos_warning`: `{ "minutes_before_violation": number }`  (Phase 6 hook)
- Rule evaluators are small async callables registered in `packages/fleet_workers/src/fleet_workers/tasks/alerts/` keyed by `alert_type`.
- Notification channels: in-app (always; Redis Pub/Sub `alerts:{org_id}` → WebSocket), email (Mailgun/SMTP), SMS (Twilio), push (Expo for the mobile app), webhook (Task 4.4).
- Throttling: same `(rule_id, vehicle_id, alert_type)` combination is suppressed for `rule.cooldown_seconds` (default 600) to prevent alert storms.

REST endpoints:
```
POST   /v1/alert-rules       → 201
GET    /v1/alert-rules       → list
PATCH  /v1/alert-rules/{id}  → 200
DELETE /v1/alert-rules/{id}  → 204
GET    /v1/alerts?from=&to=&vehicle_id=&acknowledged=  → cursor-paginated
POST   /v1/alerts/{id}/acknowledge → 200
```

WebSocket message: `{ "type":"alert", "id":"...", "severity":"high", "title":"...", "message":"...", "vehicle_id":"...", "occurred_at":"..." }`.

**Testing**:
- Unit: each alert_type's evaluator with a hand-crafted observation → expected `Alert` or `None`.
- Unit: invalid `conditions` payload rejected at rule creation with the JSON Schema error path.
- Integration: vehicle exceeding 100 km/h for 35 s while a rule says `>90 kmh for 30s` → one alert created and dispatched on each channel marked on the rule.
- Integration: alert cooldown — second violation within 5 minutes does not create a second alert; after 10 minutes, second alert created.
- Mocked: email/SMS/push providers receive the expected payloads.

#### 4.4 — Webhook Delivery

**What**: A subscriber model so external systems receive events via signed HTTP callbacks.

**Design**:
- Tables `webhook_endpoint(id, organisation_id, url, secret, event_types[], is_active)` and `webhook_delivery(id, endpoint_id, event_type, payload, attempt_count, last_status, last_attempt_at, succeeded_at, next_retry_at)`.
- Event types use Fleetio's nomenclature where applicable: `vehicle.created`, `vehicle.updated`, `trip.completed`, `geofence.entered`, `geofence.exited`, `alert.created`, `work_order.completed`, `inspection.failed`, `fuel.entry.created`, `dtc.detected`, `hos.violation.detected`.
- Delivery: Celery task with exponential backoff (1m, 5m, 30m, 2h, 12h, 24h; six attempts).
- Signature: `X-Fleet-Signature: sha256=<hmac(secret, body)>` and `X-Fleet-Timestamp` to prevent replay; receivers must reject if `|now - timestamp| > 5 minutes`.

REST endpoints:
```
POST   /v1/webhooks          → 201 { url, secret (one-time), event_types }
GET    /v1/webhooks          → list (secret masked)
PATCH  /v1/webhooks/{id}     → 200
DELETE /v1/webhooks/{id}     → 204
GET    /v1/webhooks/{id}/deliveries?status=success|failed → list
POST   /v1/webhooks/{id}/test → enqueues a `webhook.ping` delivery
```

**Testing**:
- Unit: HMAC signature matches a reference implementation; timestamp mismatch rejected.
- Integration (mock httpx server): event triggers delivery, 5xx response retried with backoff, exhaustion marks delivery as `failed`.
- Integration: `webhook.ping` round-trip succeeds end-to-end.
- Security: signing secret returned only once at creation; subsequent fetches mask it.

#### 4.5 — Driver Safety Event Detector

**What**: A Celery task that derives `safety_event` rows from raw position deltas.

**Design**:
- Triggers off the same Redis position stream as 4.2.
- Maintains a per-vehicle rolling buffer (Redis sorted set, last 60 s of positions).
- Detection rules (all configurable per org in a `safety_event_config` table):
  - **harsh_braking**: `|d(speed)/dt|` over 0.5 s window exceeds 0.4 g (configurable).
  - **harsh_acceleration**: symmetric, +0.35 g.
  - **harsh_cornering**: lateral acceleration estimated from `speed * d(heading)/dt` exceeds 0.4 g.
  - **speeding**: `speed_kmh > posted_limit_kmh + tolerance` for ≥ `min_duration_seconds`; posted limit fetched from OSM via a `road_speed_limit_lookup` service backed by Overpass + a 24-hour cache.
  - **idle**: `ignition_on=true AND speed_kmh=0` continuously ≥ `idle_seconds`.
- Each event inserts a `safety_event` row, publishes to `events:{org_id}`, and may trigger an alert via Phase 4.3.
- Nightly Celery beat task `compute_driver_safety_score` writes `driver_safety_score` rows aggregating last-7-day and last-30-day metrics; scoring formula: weighted (events per 1,000 km) with weights and a 0-100 normalisation curve documented in `packages/fleet_core/src/fleet_core/safety/score.py`.

**Testing**:
- Unit: synthetic position pair `(speed=60kmh, +0.5s, speed=40kmh)` → harsh_braking event with g_force ≈ 1.13.
- Unit: speeding requires sustained duration — 60 km/h in a 50 zone for 3 s does not fire if min_duration is 5 s.
- Integration: replay a fixture with known harsh-event timings → all events detected within tolerance.
- Integration: nightly score job populates `driver_safety_score` for each driver with last-7-day events.
- Property: detection is idempotent — re-running the task over the same position stream produces no duplicates (enforced by `UNIQUE(vehicle_id, occurred_at, event_type)`).

---

## Phase 5: Maintenance, Inspections, and Vehicle Diagnostics

### Purpose
Bring the Fleetio-style maintenance workflow into the platform: work orders, parts, vendors, preventive-maintenance schedules, DVIRs, and diagnostic trouble codes. After this phase, the platform is a complete fleet management product (without compliance/AI), competitive with Fleetio on maintenance UX and with Traccar on tracking.

### Tasks

#### 5.1 — Service Templates and Preventive Maintenance Scheduling

**What**: `service_task_template` table, plus the logic that detects when a vehicle is due for service.

**Design**:
- Schema from Data Model 1.
- A "PM schedule" view computes, per (vehicle, template), the next due date based on:
  - `last_completed_at` of any work_order_line_item referencing the template, AND
  - `vehicle.odometer_km` vs `template.interval_km` since last completion, AND
  - `vehicle.engine_hours` vs `template.interval_hours` since last completion, AND
  - `template.interval_days` since last completion.
- The "due-ness" metric `progress_pct = max(km_progress, hours_progress, days_progress)`.
- Celery beat `pm_check` (hourly) inserts `alert` rows for any vehicle whose `progress_pct ≥ rule.threshold_pct` (default 90%), wired through the Phase 4.3 engine.

REST endpoints:
```
POST   /v1/service-templates      → 201
GET    /v1/service-templates      → list
PATCH  /v1/service-templates/{id} → 200
GET    /v1/vehicles/{id}/pm-schedule → [{template, progress_pct, next_due_at, next_due_km}]
GET    /v1/pm-schedule?status=due → cross-fleet view
```

**Testing**:
- Unit: PM math — template with `interval_km=10000`, vehicle at `odometer_km=15000`, last service at `5000` → `progress_pct=100`, `next_due_km=15000`.
- Unit: when multiple intervals defined, the most-progressed one drives `progress_pct`.
- Integration: completing a work_order line item for a template updates the schedule.

#### 5.2 — Work Order Lifecycle

**What**: Implement the work_order, work_order_line_item, vendor, and part tables with a state-machine-driven REST API.

**Design**:
- State machine for `work_order.status`:
  ```
  open → in_progress → waiting_parts → in_progress → completed
                                          → cancelled
  ```
- `work_order_number` auto-generated per organisation with a configurable prefix and zero-padded sequence (e.g., `WO-000123`).
- Cost rollup: trigger updates `total_parts_cost`, `total_labour_cost`, `total_cost` on `work_order` whenever a line item changes; trigger written in Python (SQLAlchemy event listener) to keep all logic out of the DB.
- Completing a work order: server-side validation that every required line item is marked completed; sets `completed_at`, deducts parts from `part.quantity_on_hand`, emits `work_order.completed` event.

REST endpoints:
```
POST   /v1/work-orders                                  → 201
GET    /v1/work-orders?vehicle_id=&status=&priority=    → list
GET    /v1/work-orders/{id}                             → { wo, line_items, audit }
PATCH  /v1/work-orders/{id}                             → 200
POST   /v1/work-orders/{id}/transition  { to_status }   → 200
POST   /v1/work-orders/{id}/line-items                  → 201
PATCH  /v1/work-orders/{id}/line-items/{lid}            → 200
DELETE /v1/work-orders/{id}/line-items/{lid}            → 204

POST   /v1/vendors    /  GET  /  PATCH  /  DELETE       → standard CRUD
POST   /v1/parts      /  GET  /  PATCH  /  DELETE       → standard CRUD
POST   /v1/parts/{id}/adjust  { delta, reason }         → 200 (manual inventory adjustment)
```

**Testing**:
- Unit: state machine — `cancelled → in_progress` rejected.
- Unit: completing a WO with an uncompleted required line item → 422.
- Integration: completing a WO consumes parts inventory; if inventory insufficient → 422 with shortage detail.
- Integration: cost rollup matches the sum of line items after each change.
- Integration: audit log records every status transition with `actor_user_id` and timestamp.

#### 5.3 — Driver Vehicle Inspection Reports (DVIR)

**What**: Inspection templates and submission workflow, accessible via REST (for web) and a mobile-friendly compact API.

**Design**:
- Tables `inspection_template`, `inspection_template_item`, `inspection`, `inspection_item_result` from Data Model 1.
- Photo uploads to MinIO/S3: pre-signed PUT URLs handed back from `POST /v1/inspections/{id}/items/{item_id}/photo-url`; the driver uploads directly, then PATCHes the inspection item with the resulting object key.
- Failed inspection items with `defect_severity=critical` auto-create a `work_order` of type `corrective` with priority `emergency`, linked back via `work_order.source_inspection_id` (new column).
- DOT-compliant pre-trip + post-trip templates ship preloaded for trucks, vans, and trailers (seeded via fixture).

REST endpoints:
```
POST  /v1/inspection-templates                            → 201
GET   /v1/inspection-templates?vehicle_type=&type=        → list
GET   /v1/inspection-templates/{id}                       → full template with items

POST  /v1/inspections                                     → 201 (driver submission)
GET   /v1/inspections?vehicle_id=&driver_id=&status=      → list
GET   /v1/inspections/{id}                                → full inspection + item results
PATCH /v1/inspections/{id}                                → 200 (review)
POST  /v1/inspections/{id}/items/{item_id}/photo-url      → { upload_url, object_key }
```

**Testing**:
- Unit: critical failure auto-creates emergency WO referencing the inspection.
- Unit: pre-signed URL generated has correct content-type constraint and 15-minute TTL.
- Integration: mobile submission round-trip — create inspection with N items, attach photo, mark inspection complete, fetch via review API.
- Fixture: every preloaded DOT template lints against an inspection-item JSON Schema.

#### 5.4 — Diagnostic Trouble Code Ingestion

**What**: Persist DTCs reported by OBD-II / J1939 devices, classify by severity, and create alerts.

**Design**:
- Schema from Data Model 1 (`diagnostic_trouble_code`); `is_active` derived from `cleared_at IS NULL`.
- DTC catalogue: `dtc_definition(code, protocol, spn, fmi, short_text, severity, manufacturer)` seeded with the publicly available OBD-II generic codes (SAE J2012) and the standard J1939 SPN/FMI table from SAE J1939-71.
- Ingest path: the Phase 3 telematics gateway extracts DTC records from Teltonika / OBD-II frames and writes to Redis Stream `telemetry:faults`. A worker drains into `telemetry_fault` (TimescaleDB hypertable) and `diagnostic_trouble_code` (the operational view).
- Cleared DTCs: when a device reports a DTC that was previously active but is absent from the current snapshot for ≥ 3 consecutive reports, mark `cleared_at = now()`.
- Severity-based alert: any new `critical` DTC creates an alert via the Phase 4.3 engine.

REST endpoints:
```
GET /v1/vehicles/{id}/dtcs?active_only=true → list
GET /v1/dtcs?severity=&from=&to=             → cross-fleet
POST /v1/dtcs/{id}/dismiss { reason }        → 200 (manually clear)
GET /v1/dtc-definitions?q=                   → catalogue search
```

**Testing**:
- Unit: parsing a Teltonika DTC element → correct `code`, `protocol`, `spn`, `fmi`.
- Unit: clearing logic — DTC absent from 3 reports flips `cleared_at`.
- Integration: critical DTC creates an alert.
- Fixture: DTC catalogue loaded with ≥ 1,000 codes; lookup by code returns expected description.

---

## Phase 6: Compliance — ELD / HOS, IFTA, Fuel

### Purpose
Add the regulated workflows that gate adoption in US/Canadian commercial trucking. After this phase, the platform tracks driver duty status compliantly with FMCSA 49 CFR Part 395, generates IFTA quarterly reports, and ingests fuel-card transactions with anomaly detection.

### Tasks

#### 6.1 — HOS Logging and Violation Detection

**What**: Implement the `hos_log` table, FMCSA-compliant duty-status state machine, and rolling HOS calculator.

**Design**:
- Schema from Data Model 1's `hos_log`. Each row is an immutable status interval (`status_start`, `status_end`); edits create a new row pointing back at the original via `replaces_log_id UUID REFERENCES hos_log(id)` (added as a small extension to support 49 CFR Part 395's edit-trail requirement; the original row is never UPDATEd).
- Duty status state machine: `off_duty | sleeper_berth | driving | on_duty_not_driving`. Transitions allowed from any state to any state; `driving` requires an active vehicle assignment.
- Compliance calculator `packages/fleet_core/src/fleet_core/compliance/hos.py`:
  ```python
  @dataclass
  class HosState:
      driving_minutes_today: int
      on_duty_minutes_in_window: int   # rolling 14 hours
      since_last_30_min_break: int     # consecutive driving minutes
      cycle_minutes_in_7_or_8_day: int

  def evaluate(driver_id: UUID, as_of: datetime) -> HosState: ...
  def compute_violations(logs: list[HosLog]) -> list[HosViolation]: ...
  ```
- Rules implemented per 49 CFR Part 395 (Property-Carrying CMV):
  - 11-hour driving limit after 10 consecutive hours off.
  - 14-hour driving window.
  - 30-minute break after 8 cumulative hours of driving.
  - 60/70-hour cycle (7/8-day).
- Celery beat `hos_evaluator` runs every 5 minutes per active driver; inserts `hos_violation` rows; emits alerts via Phase 4.3 (`hos_warning` alert type).
- Ingest: ELD-capable telematics devices stream duty status changes through the Phase 3 gateway; HTTP endpoint `POST /v1/hos/events` accepts driver-app submissions (Phase 7) with an `Idempotency-Key` header.
- Driver certification: at the end of each day, the driver "certifies" the log via `POST /v1/hos/logs/{date}/certify`; this sets `certified=true` and `certified_at`.

REST endpoints:
```
POST  /v1/hos/events                            → 201 { hos_log }
GET   /v1/hos/logs?driver_id=&from=&to=         → list
GET   /v1/hos/logs/{date}?driver_id=            → daily log with totals
POST  /v1/hos/logs/{date}/certify               → 200
PATCH /v1/hos/logs/{id}  { new_status, annotation }  → 200 (creates a replacement, preserves original)
GET   /v1/hos/violations?driver_id=&from=&to=   → list
GET   /v1/hos/violations/{id}/acknowledge       → 200
```

**Testing**:
- Unit: 11-hour driving — synthetic stream of 12 hours of `driving` → exactly one `11_hour_driving` violation at minute 661.
- Unit: 30-minute break rule — 8 hours of driving without a 30-minute non-driving period → violation.
- Unit: 14-hour window from first on-duty triggers correctly.
- Unit: edit preserves original — `PATCH /v1/hos/logs/{id}` creates a new row with `replaces_log_id` set; original `status_end` unchanged.
- Integration: full day of 1,000 status events for 50 drivers processed in < 10 s.
- Compliance: snapshot test against an FMCSA-published sample log set produces matching violation counts.

#### 6.2 — IFTA Trip Segmentation and Quarterly Reporting

**What**: Attribute every trip's mileage and fuel consumption to the correct jurisdiction, then generate quarterly IFTA reports.

**Design**:
- Celery task `ifta_segment_trip` triggered by `trip.completed` events.
- Algorithm:
  1. Load the trip's position stream from `telemetry_position`.
  2. Walk positions in order; for each point, query a pre-built jurisdiction boundary table (`jurisdiction_boundary(country_code, subdivision_code, geom GEOGRAPHY(MULTIPOLYGON, 4326))`, seeded from Census Bureau cartographic boundaries + Statistics Canada) to find the current jurisdiction.
  3. Detect a "jurisdiction crossing" when the lookup result changes; create an `ifta_trip_segment` row capturing the previous jurisdiction with `distance_km` accumulated.
  4. Fuel attribution: distribute `trip.fuel_consumed_l` proportionally to each segment by distance.
- Quarterly report: `POST /v1/ifta/reports { quarter: "2026-Q1", vehicle_ids: [...] }` aggregates segments and fuel purchases per jurisdiction, applies `jurisdiction.fuel_tax_rate`, generates a CSV + PDF stored in MinIO. The CSV format matches the consolidated IFTA report format used by state portals.
- Jurisdiction boundary fixture: `make ingest-jurisdictions` downloads Census + StatCan shapefiles and loads them via `shp2pgsql`; documented runbook.

REST endpoints:
```
POST  /v1/ifta/reports         → 202 (async) { report_id }
GET   /v1/ifta/reports/{id}    → status + download URLs when ready
GET   /v1/ifta/reports?quarter=&vehicle_id=  → list
GET   /v1/ifta/segments?vehicle_id=&from=&to= → raw segments for audit
```

**Testing**:
- Unit: synthetic trip crossing Texas → Oklahoma → Kansas (3 jurisdictions) produces 3 segments with expected distances.
- Unit: fuel attribution sums back to the original trip total within rounding tolerance.
- Integration: end-to-end with `fixtures/traccar_replay/dallas_to_denver.csv` produces a valid IFTA quarterly CSV that passes a referenced state portal's import schema validator.
- Integration: regenerating a report after backdated edits produces a new report version without overwriting the prior one.

#### 6.3 — Fuel Card Integration and Anomaly Detection

**What**: Ingest fuel-card transactions (WEX, Fuelman, Comdata) and flag anomalies with a simple ML model.

**Design**:
- Schema from Data Model 1's `fuel_entry`.
- Ingest paths:
  1. Direct CSV upload: `POST /v1/fuel/entries/upload` (multipart, ≤ 10 MB) maps via a per-format column mapping in `fuel_card_format(name, column_map JSONB)`.
  2. API ingest: `POST /v1/fuel/entries` for partners with REST capability (FleetCor, WEX).
  3. Webhook receipt: dedicated `/v1/fuel/webhooks/{partner}` endpoints with HMAC signing.
- Deduplication: composite natural key `(fuel_card_number, transaction_date, merchant_name, quantity_litres, total_cost)` enforced via `UNIQUE` index.
- Anomaly detector (scikit-learn `IsolationForest`, trained per organisation nightly):
  - Features per transaction: `quantity_litres / tank_capacity_litres`, `cost_per_litre`, `km_since_last_fuel`, `mileage_per_litre`, `time_of_day_bucket`, `merchant_freq`, `distance_to_vehicle_at_time`.
  - "Distance to vehicle at time": the vehicle's nearest `telemetry_position` within ±15 min of `transaction_date`; mismatch > 1 km is a strong fraud signal (the card was used somewhere the vehicle wasn't).
  - Transactions in the top decile of anomaly score: `is_anomalous=true`; emits alert via Phase 4.3 `fuel_anomaly` type.

REST endpoints:
```
POST  /v1/fuel/entries           → 201 (single)
POST  /v1/fuel/entries/upload    → 202 (CSV batch async) { job_id }
GET   /v1/fuel/jobs/{id}         → progress + error report
GET   /v1/fuel/entries?vehicle_id=&from=&to=&anomalous=  → cursor list
PATCH /v1/fuel/entries/{id}      → 200 (review override)
POST  /v1/fuel/webhooks/wex      → 200 (vendor-specific)
```

**Testing**:
- Unit: CSV parser handles WEX/Fuelman/Comdata sample files from `fixtures/fuel_cards/` with no errors.
- Unit: deduplication — re-uploading the same CSV is a no-op.
- Unit: synthetic anomaly (vehicle in Dallas, fuel transaction in Houston, same minute) flagged.
- Integration: fuel anomaly creates an alert.
- Integration: nightly training task produces a model artefact in MinIO with versioned key.

---

## Phase 7: Operator Dashboard (Web) and Mobile Driver App

### Purpose
Ship the user interfaces that turn the platform's API surface into a usable product. The web dashboard is the operator's home; the mobile app is the driver's. After this phase, the platform is fully self-serviceable end-to-end.

### Tasks

#### 7.1 — Web App Shell, Auth Flow, and Theme

**What**: Next.js 15 App Router scaffold with login, dashboard layout, Tailwind theme, and shadcn/ui setup.

**Design**:
- Routes:
  - `(auth)/login` — email/password + magic link + OIDC buttons.
  - `(auth)/oidc/[provider]/callback` — exchanges code, sets session cookie.
  - `(dashboard)/layout.tsx` — sidebar nav (Map · Vehicles · Drivers · Maintenance · Compliance · Reports · AI · Settings), top bar (org switcher, user menu, alert bell).
- Auth: `next-auth` v5 with a custom Credentials provider talking to `/v1/auth/login`; refresh tokens stored httpOnly secure SameSite=Lax.
- API client: generated from OpenAPI via `openapi-typescript` into `packages/fleet_sdk_ts`; consumed via TanStack Query.
- Theme: light + dark via `next-themes`; colour tokens defined as CSS vars on `:root` and `.dark`.
- Accessibility: WCAG AA contrast; keyboard nav for sidebar; aria-live region for alert bell.

**Testing**:
- E2E (Playwright): login with password, log out, log in via magic link, log in via mocked OIDC.
- E2E: dark mode toggle persists across sessions.
- A11y: axe-core scan on every dashboard page returns 0 violations.

#### 7.2 — Live Map View

**What**: The primary operational screen — a MapLibre map showing all vehicles in real time with click-to-detail panels.

**Design**:
- `react-map-gl` + MapLibre rendering OpenStreetMap tiles from the in-cluster `maptiles` service in dev, configurable URL in prod.
- Connects to `WS /v1/ws/positions` on mount; updates a `Map<vehicle_id, Position>` and re-projects markers.
- Marker clustering via `supercluster` when zoom < 11.
- Side panel on vehicle click: name, driver, current speed, last-seen time, HOS remaining (if compliance enabled), open alerts count, link to vehicle detail page.
- Geofence overlay toggle: fetches `GET /v1/geofences?bbox=<viewport>` on map move (debounced 500 ms) and renders polygons.
- Trip playback: pick a vehicle + date range → fetches `GET /v1/trips?...` → click a trip → renders the GeoJSON LineString with a draggable timestamp scrubber.

**Testing**:
- E2E (Playwright): open map, mock WebSocket with replay fixtures, assert markers appear and move.
- E2E: click marker → panel renders with expected data.
- E2E: drawing a polygon geofence with the on-map editor → POSTs valid GeoJSON.
- Visual regression (Playwright): map view stable across runs.

#### 7.3 — Vehicle, Driver, Work Order, and Alert Pages

**What**: List + detail pages for the core entities, with inline editing for fields and full forms for creation.

**Design**:
- Tables built with `@tanstack/react-table`; sortable, filterable, server-paginated via the cursor APIs from Phases 2 and 5.
- Detail pages use a two-column layout: primary info on the left, contextual panels (recent trips, open work orders, recent alerts, compliance status) on the right.
- Forms via `react-hook-form` + Zod schemas auto-generated from the OpenAPI components.
- Work order page includes a line-item editor with running totals and a state-machine action button (Start, Mark Waiting Parts, Complete, Cancel) that calls the transition endpoints.
- Alert page supports bulk acknowledge.

**Testing**:
- E2E: full CRUD for vehicles, drivers, work orders.
- E2E: bulk acknowledge 25 alerts — verify a single API call with `ids: [...]` body.
- Component: forms display backend validation errors at the right field paths.

#### 7.4 — Mobile Driver App

**What**: React Native + Expo app covering DVIR, trip start/end, HOS log, and messaging.

**Design**:
- Screens:
  - **Login** — same JWT flow as web; supports biometric unlock for subsequent sessions.
  - **Dashboard** — today's assigned vehicle, current HOS remaining (large gauge), shift summary.
  - **DVIR** — pick template, walk through items with pass/fail/NA, attach photo (Expo `expo-camera`), sign with finger, submit.
  - **Trip** — start/end with one tap; if linked to an order, surface destination + ETA.
  - **HOS** — change duty status (off duty / sleeper / driving / on duty), see daily totals, certify end of day.
  - **Messages** — two-way text with the dispatcher (uses the same WebSocket gateway).
  - **Inspection History** — list of past DVIRs.
- Offline-first: SQLite cache via `expo-sqlite` for the active shift; mutations queued and replayed when online (idempotency keys ensure no duplicates).
- Background: `expo-task-manager` updates location every 30 s while a trip is active (only when the user has granted "Always" permission); writes to the same Traccar HTTP ingest endpoint as the gateway.
- Push notifications: Expo Push for alert delivery and dispatch messages.

**Testing**:
- Detox E2E: login → start trip → submit DVIR with one critical failure → log off duty → see HOS countdown.
- Unit: offline queue — submit DVIR while offline, restore network → submission posts exactly once.
- Integration: background location updates posted to the ingest gateway show up in the live map.
- Cross-platform: tests run on iOS simulator and Android emulator in CI.

---

## Phase 8: AI-Native Differentiators — Predictive Maintenance, NL Query, MCP Server

### Purpose
Ship the capabilities called out in `README.md` as the AI-native advantage. After this phase, the platform predicts failures before they happen, lets operators ask questions in plain English, and exposes its data and operations to any MCP-compatible AI assistant.

### Tasks

#### 8.1 — Predictive Maintenance Model

**What**: A scikit-learn pipeline that predicts component failure (engine, brakes, transmission) within a 7-day horizon using sensor history and DTC patterns.

**Design**:
- Feature store: a daily Celery beat task `build_pm_features` populates `pm_feature_snapshot(vehicle_id, snapshot_date, features JSONB)` with:
  - 7/30/90-day rolling averages of key sensor parameters (`coolant_temp_c`, `engine_rpm`, `fuel_level_pct`, `oil_pressure_kpa`, `battery_voltage_v`).
  - Standard deviations and trend slopes.
  - Counts of DTCs by severity in the past 30/90 days.
  - Distance and engine-hours since last preventive service per template.
  - Vehicle metadata (make/model/year, fuel type).
- Labels: for supervised training, look back 18 months — for each snapshot, label `failure_7d` = 1 if a `work_order` of type `corrective` with `priority IN ('high','emergency')` was opened within 7 days of the snapshot date.
- Model: gradient-boosted classifier (`HistGradientBoostingClassifier`); trained per (organisation, vehicle_class) when sufficient data exists; otherwise falls back to a global "starter" model bundled with the platform.
- Training Celery task `train_pm_model` runs weekly; persists the model + metadata (AUROC, precision-recall, feature importance, train/test counts) to MinIO under `models/pm/{org}/{vehicle_class}/{trained_at}.joblib`.
- Inference Celery beat task `score_pm_models` runs nightly: scores every active vehicle's most recent feature snapshot; writes `pm_prediction(vehicle_id, snapshot_date, failure_probability, top_contributing_features)`; creates an alert via Phase 4.3 (`predictive_maintenance` alert type) when `probability > threshold` (default 0.6).
- Explainability: top 5 features by SHAP-like contribution stored in `top_contributing_features JSONB`.

REST endpoints:
```
GET /v1/vehicles/{id}/predictions          → list (most recent first)
GET /v1/predictions?from=&to=&min_probability= → cross-fleet
GET /v1/predictions/models                 → model versions in use
POST /v1/predictions/feedback              { prediction_id, was_correct, actual_failure_date? } → 200
```

**Testing**:
- Unit: feature builder produces the documented schema for a synthetic vehicle.
- Unit: model serialisation round-trip via joblib reproduces predictions exactly.
- Integration: training task on a fixture dataset of 500 vehicles × 18 months produces a model with AUROC > 0.7.
- Integration: nightly scoring updates predictions and creates alerts for the top decile.
- Feedback loop: posting `was_correct=false` decreases the model's labelled-positive count in the next retraining cycle.

#### 8.2 — Natural-Language Query Interface

**What**: An assistant that answers fleet questions like "Which 5 vehicles have the worst fuel efficiency this month?" by generating safe, scoped SQL and rendering results.

**Design**:
- Pipeline:
  1. User question + conversation history + the user's `org_id` and RBAC scopes → LiteLLM `complete` call with a system prompt that includes (a) the database schema documentation, (b) the user's allowed tables, (c) instructions to emit only `SELECT` statements with mandatory `WHERE organisation_id = :org_id` clauses.
  2. The model emits a SQL query in fenced code blocks plus a natural-language explanation.
  3. Server-side guardrails:
     - Parse with `sqlglot`; reject anything that isn't a single `SELECT`.
     - Walk the AST to ensure every referenced table has an `organisation_id = :org_id` predicate (or is a public lookup).
     - Cap returned rows at 10,000 and impose a 5-second statement timeout.
     - Run as a low-privileged Postgres role (`fleet_nl_query`) with only `SELECT` on whitelisted views.
  4. Result formatting: tabular for ≤ 100 rows, summary card otherwise; chart suggestion picked heuristically (line vs bar vs map) from column types.
- Conversation memory: stored in `nl_conversation` and `nl_message` tables, scoped per user; embeddings of past questions inserted into a `pgvector` column for retrieval-augmented prompting.
- Schema docs: each table has a markdown description committed in `packages/fleet_db/src/fleet_db/docs/<table>.md`; the system prompt assembler pulls the relevant ones based on the conversation's running context.

REST endpoints:
```
POST /v1/ai/nl-query        { question, conversation_id? }
   → { conversation_id, answer, sql, rows, columns, chart_suggestion }
GET  /v1/ai/conversations   → list (per user)
GET  /v1/ai/conversations/{id} → messages
DELETE /v1/ai/conversations/{id} → 204
```

**Testing**:
- Unit: guardrail rejects `INSERT`, `UPDATE`, `DROP`, `;` sequences.
- Unit: guardrail rejects a `SELECT` against `vehicle` without the required `organisation_id` predicate.
- Integration (mocked LLM): question "list all vehicles in fleet X" produces correct SQL and results.
- Integration: cross-org probe — even if the LLM emits a query against another org's data, the SQL rewriter rejects it; logged to audit.
- Eval suite: 50 hand-curated question/expected-row pairs under `tests/nl_query_eval/` — must pass ≥ 80% in CI before model upgrades.

#### 8.3 — MCP Server

**What**: A Model Context Protocol server exposing fleet entities as resources and high-value operations as tools.

**Design**:
- Package `fleet_mcp` implements the official MCP spec over stdio and SSE transports.
- Resources (read-only):
  - `fleet://vehicles/{id}` — vehicle JSON.
  - `fleet://drivers/{id}` — driver JSON.
  - `fleet://trips/{id}` — trip + summary metrics.
  - `fleet://work-orders/{id}` — work order with line items.
  - `fleet://alerts/recent` — last 100 alerts.
- Tools (mutating, gated by RBAC):
  - `create_work_order(vehicle_id, title, priority, line_items)` → returns the new work order.
  - `acknowledge_alert(alert_id, note?)`.
  - `set_geofence(name, polygon_geojson, speed_limit_kmh?)`.
  - `run_report(name, params)` (wraps the Phase 9 reporting endpoints).
  - `query_fleet(question)` (proxies to 8.2's NL query for clients without their own LLM).
- Auth: clients present an API key; the key's RBAC scopes determine which tools/resources are visible.

**Testing**:
- Unit: MCP protocol conformance against the reference test suite.
- Unit: tool calls round-trip through the platform's normal HTTP path (no separate code path).
- Integration: connect Claude Desktop to the MCP server via SSE, ask "list my five worst-performing drivers this month" → returns expected JSON.
- Security: a read-only API key cannot call mutating tools.

---

## Phase 9: EV Fleet, Reporting, and Multi-Fleet Hierarchy

### Purpose
Round out v1.1 with EV-specific state, organisation-wide reporting, and the multi-fleet hierarchy that enterprise buyers need. After this phase, the platform handles modern fleet composition and answers the executive-level questions.

### Tasks

#### 9.1 — EV Telemetry and Charging State

**What**: Add EV-specific tables and ingestion paths; render EV-aware UI components.

**Design**:
- Tables `ev_status` and `charging_station` from Data Model 1 verbatim.
- Telematics gateway extended with a `tesla_fleet` protocol (HTTP, Tesla Fleet API) and an `ocpp_1_6` listener (WebSocket, for chargers).
- Range estimation: `estimated_range_km = battery_soc_pct * tank_capacity_kwh / avg_kwh_per_km_last_30d`; rolling 30-day energy-per-km computed per vehicle in a Celery task.
- Charging session derivation: a session starts on `is_charging` transition false→true, ends on true→false; persisted to `charging_session(id, vehicle_id, station_id, start_at, end_at, energy_added_kwh, peak_power_kw, cost?)`.
- Web dashboard adds a "Battery" column to the vehicle table and a "Charging" tab to vehicle detail with sessions and a sparkline of SoC over time.

REST endpoints:
```
POST /v1/ev/status                          → 201 (manual or device)
GET  /v1/vehicles/{id}/ev-status            → latest snapshot
GET  /v1/vehicles/{id}/ev-status/history?from=&to= → time series
GET  /v1/vehicles/{id}/charging-sessions    → list
POST /v1/charging-stations                  → 201
GET  /v1/charging-stations?bbox=&public=    → GeoJSON FeatureCollection
```

**Testing**:
- Unit: range estimator with synthetic 30-day history produces expected value.
- Unit: session derivation correctly handles ignition off + charging on transitions.
- Integration: Tesla Fleet API mock returns vehicle data; status persisted.

#### 9.2 — Reports and Exports

**What**: Pre-built reports plus a flexible CSV/Excel export for any table view.

**Design**:
- Pre-built reports (each is a parameterised SQL query rendered to PDF + CSV):
  - **Fleet utilisation** — % of time vehicles are in `driving` state per day; comparison vs prior period.
  - **Driver safety leaderboard** — top/bottom 10 by safety score with event counts.
  - **Maintenance cost per vehicle** — TCO breakdown by category over a period.
  - **Fuel efficiency** — km/L per vehicle with anomaly count.
  - **Idle time** — minutes idling per vehicle, with carbon estimate.
  - **HOS compliance summary** — violations per driver per week.
  - **IFTA quarterly** (already in Phase 6).
- Pre-built report engine in `packages/fleet_api/src/fleet_api/reports/`:
  ```python
  @report("fleet_utilisation")
  async def fleet_utilisation(org_id: UUID, params: FleetUtilisationParams) -> ReportResult: ...
  ```
- Async execution via Celery; result stored in MinIO; `report_run(id, org_id, name, params, status, file_csv_key, file_pdf_key, requested_by, started_at, completed_at)`.

REST endpoints:
```
GET  /v1/reports                          → catalogue of available reports
POST /v1/reports/{name}/run               → 202 { run_id }
GET  /v1/reports/runs/{id}                → status + download URLs
GET  /v1/reports/runs?name=&from=&to=     → list of past runs
```

**Testing**:
- Unit: each report's SQL produces expected rows for a fixture dataset.
- Integration: full pipeline — request → Celery execution → MinIO upload → URL returned.
- Visual: PDF output snapshot tested against a baseline (Pillow image hash).

#### 9.3 — Multi-Fleet Hierarchy

**What**: Support an org tree (parent/child organisations and sub-fleets) with cascading visibility and permissions.

**Design**:
- `organisation.parent_org_id` already exists from Phase 1; extend with a recursive view `organisation_descendants` and a Python helper to compute "all org IDs visible to user X" (the user's home org + all descendants if their role grants `tree_visibility`).
- `fleet` already exists; add `fleet.parent_fleet_id UUID REFERENCES fleet(id)` for sub-fleets.
- Casbin policies extended: `tree:viewer` role visibility uses a wildcard `org` claim resolved at query time to all descendant IDs.
- All listing endpoints accept `?include_descendants=true` (default true for tree-scoped roles, false otherwise).

REST endpoints:
```
POST  /v1/organisations            → 201 (creates a child org under the caller's org)
GET   /v1/organisations            → tree-aware list
GET   /v1/organisations/{id}/tree  → nested response
```

**Testing**:
- Unit: descendants query returns correct set for a 3-level tree.
- Integration: a parent-org admin can list vehicles across all child orgs; a child-org admin cannot see parent or siblings.
- Integration: bulk operations respect the visibility set.

---

## Phase 10: Hardening, Documentation, and Release

### Purpose
Get to a public 1.0: complete API docs, security audit, performance baselines, deployment artefacts, and the contributor onboarding experience.

### Tasks

#### 10.1 — OpenAPI Polish and SDKs

**What**: Ensure the generated OpenAPI 3.1 spec is complete and accurate; publish SDKs for TypeScript, Python, and Go.

**Design**:
- CI step that diffs `docs/api/openapi.yaml` against the FastAPI-generated spec and fails if drift exists.
- Every endpoint must have: `summary`, `description`, `operationId`, tags, examples for request and response, response codes for 200/201/204/400/401/403/404/422/429/500.
- Generators wired:
  - TypeScript: `openapi-typescript` + `openapi-fetch` → `packages/fleet_sdk_ts`.
  - Python: `openapi-python-client` → `packages/fleet_sdk_py`.
  - Go: `oapi-codegen` → `packages/fleet-sdk-go` (new).
- Redoc HTML served at `/docs`; Swagger UI at `/swagger`.

**Testing**:
- CI: spec diff is clean.
- CI: each SDK builds and passes its smoke test (`list vehicles`).
- Linting: `vacuum lint` on the spec yields zero errors.

#### 10.2 — Performance Baselines and Load Tests

**What**: Establish target SLOs and run load tests to confirm them.

**Design**:
- SLO targets:
  - REST API p95 latency: < 200 ms for read endpoints, < 400 ms for writes (excluding async-202).
  - Telemetry ingest sustained throughput: 10,000 positions/sec on a 4-vCPU/16-GB-RAM API node and an 8-vCPU/32-GB Postgres node.
  - WebSocket fan-out latency from ingest to client: < 1 s p95.
- Load test harness: `k6` scripts in `tests/load/`:
  - `rest_read.js`: 200 RPS mixed reads against `/vehicles`, `/trips`, `/alerts`.
  - `rest_write.js`: 50 RPS mixed writes (work orders, fuel entries, inspections).
  - `ingest.js`: 10,000 simulated devices over the Traccar OsmAnd HTTP protocol.
  - `ws.js`: 5,000 concurrent WebSocket connections receiving live positions.
- Grafana dashboards for API latency, queue depths, DB connection saturation, ingest throughput; saved as JSON under `deploy/grafana/`.

**Testing**:
- CI nightly: run the load profile against a docker-compose stack; fail if any SLO regresses by > 20% from the committed baseline `tests/load/baseline.json`.
- Manual: a runbook documenting how to scale each component.

#### 10.3 — Security Review

**What**: Map the platform against OWASP API Top 10 and ISO 39001 / GDPR / CCPA requirements; remediate gaps.

**Design**:
- Document in `docs/security.md`:
  - Threat model with STRIDE per component.
  - OWASP API Top 10 mapping with the specific controls in place.
  - Data-protection workflow: configurable retention policy per data class (positions, trips, HOS logs); data-subject access request (DSAR) export endpoint `POST /v1/privacy/dsar { driver_id }` produces a ZIP of everything we know about a driver.
  - Driver privacy windows: per-driver schedule when telemetry is dropped at ingest (matches GDPR "personal use" carve-outs).
  - Encryption at rest (Postgres TDE via cloud provider or `pgcrypto` for column-level encryption of sensitive fields) and in transit (TLS everywhere; mTLS for telematics ingest).
  - Secrets management via env files in dev and Kubernetes Secrets in prod; documented rotation procedure.
  - Vulnerability scanning in CI: `pip-audit`, `npm audit`, `trivy` for Docker images.
- Pen-test playbook: documented manual tests for each OWASP API Top 10 category.

**Testing**:
- CI: dependency scanners fail the build on `HIGH`/`CRITICAL` CVEs without a documented suppression.
- Integration: DSAR endpoint returns a complete export.
- Integration: configured driver-privacy window drops positions during the window.

#### 10.4 — Deployment Artefacts and Self-Host Documentation

**What**: Production-grade deployment paths (docker-compose, Helm) and a documented self-host quickstart.

**Design**:
- `docker-compose.prod.yml` with sensible defaults (resource limits, logging driver, restart policy).
- Helm chart `deploy/helm/fleet-management/`:
  - Sub-charts (or required dependencies) for Postgres, Redis, MinIO, optional OSRM.
  - Configurable replica counts, resource requests/limits, ingress, TLS via cert-manager.
  - Helm test hooks that run a smoke test (`kubectl exec` into the API pod, hit `/health`).
- Self-host quickstart in `docs/self-host.md`: download compose file, configure env, `docker compose up -d`, create org via `make bootstrap`, browse `https://localhost`.
- Migration guide for users coming from Traccar or Fleetio (CSV import for vehicles, drivers, work orders).

**Testing**:
- CI: `helm lint` and `helm template` succeed.
- Integration: kind cluster deploys the chart; smoke test passes.
- Documentation: a fresh contributor on macOS/Linux/Windows-WSL can follow the quickstart and reach a working install in < 15 minutes (validated by a contributor checklist in CONTRIBUTING.md).

#### 10.5 — Contributor Onboarding

**What**: Make it easy for new contributors to land a first PR.

**Design**:
- `CONTRIBUTING.md` covers: code of conduct, dev environment setup, branch and PR conventions, commit message style, sign-off (DCO), how to run a single test, how to add a migration, how to add a new ingest protocol.
- Issue templates: bug, feature, telematics-device-request, security-disclosure (private).
- "Good first issue" labels seeded with 20 curated tasks (e.g., add a new Traccar protocol, add a new pre-built report, translate the driver app to French).
- `docs/architecture.md` — narrative + Mermaid diagrams showing how data flows from device → ingest → Redis → Timescale → projections → API → UI.

**Testing**:
- Manual: a new contributor (or new-contributor-shaped Playwright test) can sign up, find a good-first-issue, and follow the linked guidance.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                  ─── required by everything
    │
Phase 2: Fleet Core (Vehicles etc.)  ─── requires Phase 1
    │
Phase 3: Telematics Ingestion        ─── requires Phase 2
    ├── Phase 4: Geofencing/Alerts/Safety  ─── requires Phase 3
    │       │
    │       └── Phase 6: Compliance (ELD/IFTA/Fuel) ─── requires Phases 3 + 4
    │
    ├── Phase 5: Maintenance & DTCs        ─── requires Phase 2 (Phase 3 optional)
    │
    └── Phase 7: Web + Mobile UIs          ─── requires Phases 2–6 to be feature-complete
            │
            └── Phase 8: AI Differentiators ─── requires Phases 3–6 data; UI hooks via Phase 7
                    │
                    └── Phase 9: EV + Reports + Hierarchy ─── extends 5/7/8
                            │
                            └── Phase 10: Hardening + Release ─── final gate
```

Parallelism opportunities once Phase 2 is complete:

- **Phase 3 (Ingestion) and Phase 5 (Maintenance) can be developed concurrently** — they touch disjoint parts of the schema; the only shared interface is the `vehicle.odometer_km` field which Phase 3 will start updating from telemetry once both ship.
- **Phase 4 and Phase 5 can also parallelise** — Phase 4 depends on Phase 3 (positions) for live detection but can be developed against fixtures.
- **Phase 7 work** (web shell and mobile shell) can begin as soon as Phase 2 ships its endpoints; pages are added in parallel with backend phases as the corresponding APIs become available.
- **Phase 8 sub-tasks** (predictive maintenance, NL query, MCP) are independent of each other and can be split across three contributors after Phase 6 data is in place.

---

## Definition of Done (per phase)

Every phase is considered complete only when **all** of the following are true:

1. All tasks for the phase are implemented and merged to `main`.
2. Unit tests for every new function/class with at least the named scenarios from the Testing section pass.
3. Integration tests covering the phase's new HTTP endpoints, queues, and DB writes pass against the docker-compose dev stack.
4. `make lint` (Ruff + Mypy --strict + ESLint + tsc) exits 0.
5. `make test` exits 0 (Python + JS suites).
6. Alembic migrations are reversible (`upgrade` then `downgrade` cleanly).
7. The Docker image for every affected service builds and starts.
8. The generated OpenAPI spec at `docs/api/openapi.yaml` matches the running FastAPI app (CI check from 10.1).
9. Any new configuration option is documented in `docs/configuration.md` with default, type, and required/optional flag.
10. Any new database table or column appears in `docs/data-model.md` with column-level descriptions.
11. Any new event type (Phase 3 onward) appears in `docs/events.md` with payload schema.
12. New AI behaviour (Phase 8) includes a prompt template under `packages/fleet_ai/src/fleet_ai/prompts/` with documented inputs and example outputs, plus eval suite coverage.
13. New permissions are added to the permission seed migration and to the Casbin policy file.
14. A short release-notes entry is added to `CHANGELOG.md` under `## Unreleased`.
15. CI is green on the merge commit.
