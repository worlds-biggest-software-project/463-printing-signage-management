# Printing & Signage Management — Phased Development Plan

> Project: 463-printing-signage-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

An open-source Management Information System (MIS) for commercial print shops and wide-format signage studios. It unifies estimation, quote-to-order, production scheduling, shop-floor tracking, proofing, inventory, delivery/installation, and invoicing in a single multi-tenant platform, with AI-native estimation, anomaly detection, and scheduling optimisation as headline differentiators against the all-proprietary incumbent field (PrintPLANR, EFI PrintSmith Vision, ePS PACE, Clarity, Ordant, ShopVOX, Printavo).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | TypeScript on Node.js 22 LTS | The product is API + real-time + frontend heavy, not ML-runtime heavy. A single language across the API, WebSocket layer, customer portal, and shop-floor PWA minimises context-switching and lets the shared pricing engine (the cross-cutting "W2P ↔ MIS price parity" gap identified in research) be authored once and consumed by both server and client. |
| API framework | NestJS 11 (Fastify adapter) | Modular DI architecture maps cleanly onto the bounded contexts of an MIS (estimation, production, inventory, delivery, invoicing). First-class OpenAPI 3.1 generation via `@nestjs/swagger` satisfies the standards requirement. Fastify adapter gives the throughput needed for high-frequency shop-floor scan endpoints. |
| Database | PostgreSQL 16 with `btree_gist`, `pg_trgm`, JSONB | Data-model-suggestion-3 (hybrid relational + JSONB) is adopted: normalised tables for high-integrity data (customers, invoices, payments, machines, schedule entries) and JSONB for job-type-variable data (job specs, estimation inputs, preflight results, AI outputs). `btree_gist` enables a machine double-booking exclusion constraint (from suggestion-1); `pg_trgm` powers job/customer search. |
| ORM / migrations | Prisma 6 | Type-safe schema, generated client, declarative migrations. JSONB columns map to typed `Json` fields validated at the app layer with Zod, giving suggestion-3's "JSON Schema validation to prevent data sprawl". |
| JSONB validation | Zod (shared package) | Zod schemas validate JSONB payloads (job specs, preflight reports, AI outputs) on write and are reused to derive OpenAPI component schemas and frontend form types. One source of truth. |
| Task queue | BullMQ on Redis 7 | Async workloads: proof-chase emails, accounting sync, preflight processing, AI estimation calls, scheduling rebalance, demand forecasting. BullMQ gives delayed jobs (proof chasing), repeatable jobs (forecasting), and retries with backoff (accounting sync resilience). |
| Cache / pub-sub | Redis 7 | Gantt read-model cache, session store, and the pub/sub backbone for WebSocket fan-out of schedule and job-status changes. |
| Real-time transport | Socket.IO (Redis adapter) | Gantt schedule updates and shop-floor status must push to dozens of machines and hundreds of concurrent jobs. Redis adapter allows horizontal scaling of WebSocket nodes. |
| Frontend (back-office) | Next.js 15 (App Router) + React 19 + TypeScript | Server components for data-heavy dashboards; client components for the interactive Gantt. Shares the Zod/pricing packages with the backend. |
| Gantt rendering | `@dnd-kit` + a virtualised canvas timeline (custom, on `react-window`) | Research flags that Gantt views across dozens of machines and hundreds of jobs need virtualised rendering; off-the-shelf Gantt libraries do not virtualise machine rows and time columns well. |
| Shop-floor client | PWA (same Next.js app, installable; Service Worker + Web App Manifest) | Research underserved area: "mobile-native shop floor". A PWA delivers camera-based QR/barcode scanning (`BarcodeDetector` API / `@zxing/browser` fallback) and offline-tolerant scan queueing without app-store distribution. |
| UI components | shadcn/ui + Tailwind CSS 4 | Fast, accessible, themeable component layer for back-office and portal. |
| Auth | Better-stack pattern: OAuth 2.0 + JWT (access) + rotating refresh tokens; bcrypt/argon2 password hashing | Implements RFC 6749 / RFC 7519. Customer portal supports passwordless secure-link login (InfoFlo pattern) in addition to password auth. Aligns to NIST SP 800-63B. |
| File storage | S3-compatible (AWS S3 in cloud; MinIO for self-host) via presigned URLs | Large artwork files (PDF/AI/EPS) never transit the API server; uploads/downloads use presigned URLs. Per-customer prefix isolation supports GDPR retention/erasure. |
| Artwork / prepress | `pdf-lib` + Ghostscript + `mutool` (MuPDF) in a worker container | Extract page geometry, DPI, colour space, bleed, font embedding, and check PDF/X-4 (ISO 15930-7) conformance for the automated preflight feature. |
| LLM provider | Provider-abstracted (OpenAI + Anthropic) behind an internal `LlmClient` interface; prompt-cached | AI-native estimation, anomaly explanation, and scheduling-rationale generation. Abstraction avoids vendor lock-in and allows self-hosters to point at a local model. |
| Accounting sync | QuickBooks Online API + Xero API (OAuth 2.0) | The MVP accounting requirement from features.md. Implemented as queue consumers reacting to invoice/payment domain events. |
| Equipment integration | XJDF (CIP4), JDF fallback; JMF status ingest | Standards.md mandates CIP4 for press/finishing integration. XJDF preferred for new integrations; JDF for legacy. Implemented as an adapter, deferred to a later phase. |
| Email / SMS | SMTP (RFC 5321) via Nodemailer + provider (SES/SendGrid); Twilio for SMS | Proof requests, milestone notifications, invoice delivery. |
| Error format | RFC 7807 Problem Details | Consistent machine-readable API errors via a global NestJS exception filter. |
| Testing | Vitest (unit), Supertest (HTTP integration), Testcontainers (real Postgres/Redis), Playwright (E2E) | Fast unit runner; Testcontainers gives real DB integration tests without mocks where integrity matters (exclusion constraints, transactions). |
| Quality tools | ESLint (typescript-eslint) + Prettier + `tsc --noEmit` + `prisma validate` | Lint, format, typecheck, schema-check gates. |
| Package manager / monorepo | pnpm workspaces + Turborepo | Monorepo holds API, web, worker, and shared packages (pricing engine, Zod schemas, OpenAPI types). Turborepo caches builds/tests. |
| Containerisation | Docker + docker-compose (dev), Helm chart (prod) | Self-hosted and cloud deployment parity. |
| Observability | OpenTelemetry traces + Pino structured logs + Prometheus metrics | Production debugging and the schedule-lag / queue-depth dashboards. |

### Project Structure

```
printing-signage-management/
├── package.json                 # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml           # postgres, redis, minio, mailhog, api, web, worker
├── .env.example
├── packages/
│   ├── domain/                  # Zod schemas, domain types, enums (shared)
│   │   └── src/
│   │       ├── schemas/         # job-spec, estimate-input, preflight, ai-output Zod schemas
│   │       ├── enums.ts         # job status, step types, machine types, roles
│   │       └── index.ts
│   ├── pricing-engine/          # SINGLE shared pricing engine (W2P↔MIS parity)
│   │   └── src/
│   │       ├── methods/         # fixed, per_unit, per_sqm, per_linear_m, per_panel
│   │       ├── engine.ts
│   │       └── index.ts
│   ├── api-client/              # generated typed client from OpenAPI (used by web)
│   └── config/                  # shared eslint/tsconfig/tailwind presets
├── apps/
│   ├── api/                     # NestJS application
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   ├── migrations/
│   │   │   └── seed.ts
│   │   └── src/
│   │       ├── main.ts
│   │       ├── app.module.ts
│   │       ├── common/          # RFC7807 filter, auth guards, tenancy interceptor
│   │       ├── modules/
│   │       │   ├── auth/
│   │       │   ├── tenancy/     # organisation + RLS context
│   │       │   ├── crm/         # customers, contacts, suppliers
│   │       │   ├── catalogue/   # materials, machines, labour rates
│   │       │   ├── estimation/  # estimates, items, templates, pricing
│   │       │   ├── jobs/        # jobs, job items, production steps, scans, usage
│   │       │   ├── scheduling/  # schedule entries, downtime, rebalance
│   │       │   ├── proofing/    # artwork, proofs, annotations, preflight
│   │       │   ├── inventory/   # stock, purchase orders
│   │       │   ├── delivery/    # deliveries, vehicles, installation, POD
│   │       │   ├── invoicing/   # invoices, payments, accounting sync
│   │       │   ├── reporting/   # margin, utilisation, accuracy
│   │       │   ├── ai/          # estimation assistant, anomaly, optimiser, forecast
│   │       │   ├── events/      # domain event bus + outbox
│   │       │   ├── realtime/    # Socket.IO gateway
│   │       │   └── integrations/# quickbooks, xero, xjdf, twilio, smtp
│   │       └── jobs/            # BullMQ processors (separate worker entrypoint)
│   ├── worker/                  # BullMQ worker entrypoint (imports api modules)
│   └── web/                     # Next.js back-office + portal + shop-floor PWA
│       └── src/
│           ├── app/
│           │   ├── (back-office)/
│           │   ├── (portal)/    # customer self-service
│           │   └── (floor)/     # PWA shop-floor scanning
│           ├── components/
│           │   └── gantt/       # virtualised scheduler
│           └── lib/
├── deploy/
│   └── helm/
└── tests/
    ├── e2e/                     # Playwright specs
    └── fixtures/                # sample PDFs, XJDF tickets, seed data
```

---

## Phase 1: Foundation, Tenancy & Auth

### Purpose
Stand up the monorepo, database, container stack, and the cross-cutting concerns every later phase depends on: multi-tenant isolation, authentication/authorisation, the RFC 7807 error contract, OpenAPI generation, and the domain/Zod shared package. After this phase the platform boots, a user can register an organisation and log in, and every subsequent module inherits tenancy and auth for free.

### Tasks

#### 1.1 — Monorepo, tooling, and container stack

**What**: Initialise the pnpm/Turborepo workspace, the `apps/api` NestJS app, the `packages/domain` package, and a `docker-compose.yml` bringing up Postgres, Redis, MinIO, and Mailhog.

**Design**:
- `pnpm-workspace.yaml` includes `apps/*` and `packages/*`.
- `turbo.json` pipelines: `build`, `lint`, `test`, `typecheck` with `dependsOn: ["^build"]`.
- `docker-compose.yml` services: `postgres:16` (with `POSTGRES_DB=printmis`), `redis:7`, `minio` (+ `createbuckets` init for `artwork`), `mailhog`, `api`, `worker`, `web`.
- `.env.example` keys: `DATABASE_URL`, `REDIS_URL`, `S3_ENDPOINT`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_BUCKET`, `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `SMTP_URL`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `APP_BASE_URL`.
- NestJS `main.ts` bootstraps Fastify adapter, global `ValidationPipe`, the RFC 7807 filter (task 1.3), and Swagger at `/api/docs`.
- Health endpoints: `GET /healthz` (liveness), `GET /readyz` (DB + Redis ping).

**Testing**:
- `Unit: GET /healthz → 200 {status:"ok"}`.
- `Integration (Testcontainers): GET /readyz with Postgres+Redis up → 200; with Redis down → 503`.
- `CI: pnpm turbo lint typecheck build passes on clean checkout`.
- `Smoke: docker compose up → api reachable, /api/docs renders OpenAPI 3.1 document`.

#### 1.2 — Database schema bootstrap (Prisma) and tenancy

**What**: Define the `organisations` and `users` models, enable Postgres extensions, and implement row-level tenant scoping.

**Design**:
- Prisma models (initial migration):

```prisma
model Organisation {
  id            String   @id @default(uuid()) @db.Uuid
  name          String
  slug          String   @unique
  countryCode   String   @default("AU") @db.Char(2)
  currencyCode  String   @default("AUD") @db.Char(3)
  timezone      String   @default("Australia/Sydney")
  settings      Json     @default("{}")
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  users         User[]
}

enum UserRole {
  admin estimator production_manager operator delivery_driver sales accounts customer
}

model User {
  id             String   @id @default(uuid()) @db.Uuid
  organisationId String   @db.Uuid
  organisation   Organisation @relation(fields: [organisationId], references: [id])
  email          String
  passwordHash   String?
  firstName      String
  lastName       String
  role           UserRole
  isActive       Boolean  @default(true)
  lastLoginAt    DateTime?
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt
  @@unique([organisationId, email])
  @@index([organisationId, role])
}
```

- Raw SQL migration enables `CREATE EXTENSION IF NOT EXISTS btree_gist; CREATE EXTENSION IF NOT EXISTS pg_trgm;`.
- Tenancy: a NestJS `TenancyInterceptor` reads `organisationId` from the validated JWT and stores it in an `AsyncLocalStorage` context. A Prisma client extension (`$extends`) injects `where: { organisationId }` on `findMany`/`findFirst` and sets it on `create` for tenant-scoped models. Postgres RLS policies (`USING (organisation_id = current_setting('app.org_id')::uuid)`) are added as defence-in-depth; the connection sets `app.org_id` per request via `SET LOCAL`.

**Testing**:
- `Unit: TenancyInterceptor extracts org id from JWT claims → stored in ALS`.
- `Integration (Testcontainers): two orgs seeded; user A queries customers → only org A rows returned`.
- `Integration: RLS — direct query without app.org_id set → zero rows (policy denies)`.
- `Unit: Prisma extension sets organisationId on create automatically`.

#### 1.3 — RFC 7807 error contract & validation

**What**: A global exception filter that renders all errors as `application/problem+json` and a validation pipeline that maps Zod/class-validator failures into structured `errors`.

**Design**:
- Response shape: `{ type, title, status, detail, instance, errors?: [{ field, message }] }`.
- Map: `NotFoundException → 404`, `UnauthorizedException → 401`, validation → `422` with `errors[]`, unexpected → `500` (detail hidden in prod, logged with trace id).
- A `ZodValidationPipe` validates request bodies against domain Zod schemas and throws a `422` problem.

**Testing**:
- `Unit: ValidationError on POST body → 422, problem+json, errors[] lists field + message`.
- `Unit: thrown NotFoundException → 404 problem with type URI`.
- `Unit: unexpected Error in prod mode → 500, detail omitted, log contains stack + traceId`.

#### 1.4 — Authentication & RBAC

**What**: Registration of an organisation + admin user, password login, JWT access/refresh issuance and rotation, passwordless secure-link login for customers, and a role guard.

**Design**:
- Endpoints:
  - `POST /auth/register-org` `{ orgName, adminEmail, password, firstName, lastName }` → creates org + admin user, returns tokens.
  - `POST /auth/login` `{ email, password }` → `{ accessToken (15m), refreshToken (30d, rotated) }`.
  - `POST /auth/refresh` `{ refreshToken }` → new pair; old refresh revoked (stored hashed in `refresh_tokens` table, single-use).
  - `POST /auth/portal-link` `{ email }` → emails a signed magic-link (JWT, 30m, audience `portal`) to a customer-role user.
  - `GET /auth/portal/callback?token=...` → exchanges link for portal session.
- Password hashing: argon2id. JWT claims: `sub`, `organisationId`, `role`, `aud`.
- `@Roles('admin','production_manager')` decorator + `RolesGuard` reads `role` claim.

**Testing**:
- `Integration: register-org → org + admin persisted, tokens returned, password not echoed`.
- `Integration: login wrong password → 401 problem; no token`.
- `Integration: refresh with rotated (already-used) token → 401, token family revoked`.
- `Integration: portal-link issues single-use magic link; second callback use → 401`.
- `Unit: RolesGuard denies operator on admin-only route → 403`.

---

## Phase 2: Reference Data — CRM & Catalogue

### Purpose
Implement the reference entities that estimation, production, and scheduling all read from: customers and contacts, suppliers, materials/substrates, machines, and labour rates. These are mostly CRUD but encode the dimensional and cost attributes (square-metre pricing inputs, machine speeds, JDF device ids) that the pricing and scheduling engines depend on. After this phase a shop can be fully configured.

### Tasks

#### 2.1 — Customers, contacts & suppliers (CRM)

**What**: CRUD for `customers`, `customer_contacts`, and `suppliers` with search.

**Design**:
- Prisma models follow data-model-suggestion-1 Module 2 & 3 (`customers`, `customer_contacts`, `suppliers`) with `organisationId` scoping. `suppliers.isTradePrinter` flags trade/outsourcing partners.
- Endpoints (standard REST, all tenant-scoped, OpenAPI-documented): `GET/POST /customers`, `GET/PATCH/DELETE /customers/:id`, nested `POST /customers/:id/contacts`, `GET/POST /suppliers`.
- Search: `GET /customers?q=` uses `pg_trgm` similarity on `companyName`.
- `creditLimit`, `paymentTermsDays`, `taxExempt` carried for downstream invoicing.

**Testing**:
- `Integration: create customer + primary contact → both persisted, contact.isPrimary enforced unique per customer`.
- `Integration: GET /customers?q=acm → trigram match returns "Acme Signs"`.
- `Integration: DELETE customer with jobs → 409 problem (referential guard)`.

#### 2.2 — Materials & substrate catalogue

**What**: CRUD for `materials` with dimensional and cost attributes, categories, and supplier links.

**Design**:
- Model from suggestion-1 Module 3 `materials` + `material_categories`: `materialType` enum, `widthMm`, `heightMm`, `weightGsm`, `costPerUnit`, `unitOfMeasure`, `iccProfileName`, `colourSpace`, `currentStock`, `reorderPoint`, `reorderQuantity`, `preferredSupplierId`.
- Per-job-type variable attributes go in a JSONB `attributes` field validated by a `MaterialAttributesSchema` (Zod) — e.g. vinyl: `{ adhesive, finish, removability }` (suggestion-3 principle).
- Endpoints: `GET/POST /materials`, `GET/PATCH /materials/:id`, `GET /materials?type=substrate`.

**Testing**:
- `Unit: MaterialAttributesSchema rejects unknown adhesive enum → 422`.
- `Integration: create vinyl material with attributes JSONB → persisted, GIN-indexed query by attribute works`.
- `Integration: cost_per_unit defaults to 0; negative cost → 422`.

#### 2.3 — Machines, capabilities & labour rates

**What**: CRUD for `machines` (speeds, cost rates, JDF ids), the `machine_material_capabilities` join, and `labour_rates`.

**Design**:
- Models from suggestion-1 Module 4. Machine carries `speedSheetsHr`, `speedSqmHr`, `setupCost`, `hourlyRate`, `clickCost`, `setupTimeMinutes`, `changeoverTimeMinutes`, `availableShifts`, `jdfDeviceId`, `status`.
- `machine_material_capabilities (machineId, materialId)` records which substrates each machine runs — consumed by AI press selection and scheduling validity later.
- Endpoints: `GET/POST /machines`, `PATCH /machines/:id`, `POST /machines/:id/capabilities`, `GET/POST /labour-rates`.

**Testing**:
- `Integration: add capability links machine↔material; duplicate → 409`.
- `Integration: PATCH machine status to maintenance → reflected in GET`.
- `Unit: labour_rate overtimeMultiplier default 1.5`.

---

## Phase 3: Pricing Engine & Estimation (Core Value, Part 1)

### Purpose
Build the shared pricing engine — the single source of truth that resolves the recurring "Web2Print ↔ MIS price divergence" gap — and the estimation/quoting workflow on top of it. This is the heart of the product. After this phase an estimator can build a multi-line quote with substrate/labour/machine/overhead costs, square-metre and panel pricing for wide format, and a computed margin.

### Tasks

#### 3.1 — Shared pricing engine (`packages/pricing-engine`)

**What**: A pure, deterministic, dependency-free function library computing line and job totals from structured inputs, usable identically on server and (future) Web2Print client.

**Design**:
- Input/output types (Zod-validated, exported):

```ts
type PricingMethod = 'fixed' | 'per_unit' | 'per_sqm' | 'per_linear_m' | 'per_panel';

interface CostComponentInput {
  type: 'substrate'|'ink'|'laminate'|'machine_time'|'setup'|'labour'|'outsource'|'overhead'|'other';
  quantity: number;       // units, sqm, hours, etc.
  unitCost: number;       // from material/machine/labour reference
  wasteFactor?: number;   // default 0
}
interface LineInput {
  quantity: number;
  widthMm?: number; heightMm?: number; panels?: number; linearMm?: number;
  pricingMethod: PricingMethod;
  components: CostComponentInput[];
  marginPercent?: number;     // target margin; mutually exclusive with markupPercent
  markupPercent?: number;
}
interface LineResult {
  areaSqm?: number;
  totalCost: number;          // sum of component costs incl. waste
  unitPrice: number;
  totalPrice: number;
  marginPercent: number;
  costBreakdown: { type: string; cost: number }[];
}
function priceLine(input: LineInput): LineResult;
function priceJob(lines: LineInput[], taxRate: number): { subtotal; taxAmount; total; lines: LineResult[] };
```

- Algorithm: compute `areaSqm = (widthMm*heightMm)/1e6` when relevant; component cost = `quantity * unitCost * (1+wasteFactor)`; `totalCost = Σ components`; price by method (e.g. `per_sqm` ⇒ `unitPrice = costPerSqm/(1-margin)`); `marginPercent = (totalPrice-totalCost)/totalPrice*100`.
- Pure functions, no I/O, no DB. Currency handled as integer minor units internally to avoid float drift; expose decimals at the boundary.

**Testing**:
- `Unit: per_sqm 3000×2000mm @ $25/sqm, 30% margin → areaSqm=6, totalCost=150, totalPrice≈214.29`.
- `Unit: per_panel 4 panels with setup component → setup counted once, panel cost ×4`.
- `Unit: waste factor 0.1 inflates substrate cost by 10%`.
- `Unit: margin and markup both supplied → throws (mutually exclusive)`.
- `Unit: integer-minor-unit arithmetic — 0.1+0.2 inputs sum exactly to 0.30 output`.

#### 3.2 — Estimates, items, cost lines

**What**: Persist estimates and their line items / cost breakdown, computing prices via the engine.

**Design**:
- Models from suggestion-1 Module 5 (`estimates`, `estimate_items`, `estimate_cost_lines`) plus a JSONB `spec` on `estimate_items` for job-type-variable detail (suggestion-3).
- Estimate status state machine: `draft → sent → approved | rejected | expired`, and `approved → converted`. Enforced in service; illegal transition → 409 problem.
- Endpoints:
  - `POST /estimates` → creates draft `{ customerId, title }`.
  - `POST /estimates/:id/items` `{ description, jobType, quantity, dimensions, pricingMethod, components[] }` → calls `priceLine`, persists item + cost lines.
  - `POST /estimates/:id/price` → recomputes via `priceJob`, sets subtotal/tax/total/margin.
  - `GET /estimates/:id` → full estimate with items, costs, totals.
- `estimate_number` generated `EST-{org-seq}` via a per-org sequence row in a `number_sequences` table (transactional `UPDATE ... RETURNING`).

**Testing**:
- `Integration: add two items then /price → estimate.total = Σ line totals + tax`.
- `Integration: estimate_number is sequential and unique per org under concurrent creates`.
- `Unit: transition sent→draft → 409 invalid transition`.
- `Integration: changing a material cost then re-/price updates totals (dynamic pricing)`.

#### 3.3 — Estimate templates

**What**: Reusable templates for common job types (brochure, banner, vehicle wrap, signage) that pre-fill items and components.

**Design**:
- `estimate_templates` (suggestion-1) with a JSONB `definition` holding default lines/components. `POST /estimates/from-template/:templateId` instantiates a draft estimate.
- Seed a starter template set in `seed.ts`.

**Testing**:
- `Integration: instantiate banner template → estimate created with default per_sqm line and substrate component`.
- `Unit: template definition validated by Zod TemplateSchema`.

---

## Phase 4: Quote-to-Order & Job Lifecycle (Core Value, Part 2)

### Purpose
Convert approved estimates into production jobs and model the full job lifecycle, the job number/barcode, job items, and the domain-event bus + transactional outbox that every async feature (notifications, accounting sync, AI, equipment) consumes. After this phase the estimate→order spine exists end-to-end.

### Tasks

#### 4.1 — Domain event bus & transactional outbox

**What**: An in-process event emitter backed by an `outbox` table written in the same transaction as state changes, drained to BullMQ by a relay.

**Design**:
- `outbox` table: `{ id bigserial, organisationId, aggregateType, aggregateId, eventType, payload jsonb, createdAt, dispatchedAt }`. (This adopts the event-stream benefits of suggestion-2 without full event sourcing — pragmatic for MVP.)
- `EventBus.publish(event)` inside a Prisma `$transaction` inserts the outbox row alongside the entity mutation. A relay worker polls undispatched rows (or uses `LISTEN/NOTIFY`) and enqueues BullMQ jobs per subscriber, then marks `dispatchedAt`. Idempotency key = `event.id`.
- Canonical event names mirror suggestion-2: `EstimateApprovedByCustomer`, `JobCreated`, `JobStatusChanged`, `ProofSentToCustomer`, `InvoiceCreated`, `PaymentReceived`, `MaterialConsumed`, `ReorderPointReached`, `DeliveryDispatched`, etc.

**Testing**:
- `Integration: publishing inside a failing transaction → no outbox row (atomicity)`.
- `Integration: relay enqueues each undispatched row once; re-run does not double-dispatch`.
- `Unit: subscriber receiving duplicate event id → handled idempotently`.

#### 4.2 — Quote approval & conversion to job

**What**: Customer approval of a sent estimate and automatic job creation.

**Design**:
- `POST /estimates/:id/send` → status `sent`, sets `validUntil`, emits `EstimateSentToCustomer` (→ email in Phase 8).
- `POST /portal/estimates/:id/approve` (portal auth) `{ signature }` → status `approved`, emits `EstimateApprovedByCustomer`; a handler runs `CreateJob`, copying items into `job_items`, linking `estimate.convertedToJobId`, status `converted`.
- `POST /portal/estimates/:id/reject` `{ reason }` → `rejected`.

**Testing**:
- `Integration: approve sent estimate → job created with matching items, estimate.status=converted`.
- `Integration: approve expired estimate → 409`.
- `Integration: approval emits exactly one EstimateApprovedByCustomer in outbox`.

#### 4.3 — Jobs, job items, status machine & barcode

**What**: Job CRUD, the lifecycle state machine, job-item management, and barcode/QR generation.

**Design**:
- Models from suggestion-1 Module 6 (`jobs`, `job_items`) with JSONB `spec` per item.
- Status enum and legal transitions (subset enforced): `pending → artwork_pending → proof_pending → proof_approved → scheduled → in_prepress → in_production → in_finishing → quality_check → ready_for_delivery → out_for_delivery → delivered → invoiced → complete`; plus `on_hold` (reversible) and `cancelled` (terminal) from most states.
- `PATCH /jobs/:id/status` validates transition, emits `JobStatusChanged`.
- `barcode` = job number; `qrCodeUrl` points to `/floor/jobs/:id`. QR PNG generated on demand via `qrcode`.
- `job_number` from the per-org sequence (`JOB-{seq}`).

**Testing**:
- `Unit: transition complete→pending → 409`.
- `Integration: status change emits JobStatusChanged with old+new`.
- `Integration: GET /jobs?status=in_production filters correctly and tenant-scoped`.
- `Integration: scanning barcode resolves to the correct job`.

---

## Phase 5: Shop-Floor Tracking & Production Steps

### Purpose
Make production visible in real time. Implement production steps per job item, the barcode/QR scan event endpoint optimised for the PWA, material-usage and time capture, and the WebSocket push that drives both the floor and back-office views. This delivers the "mobile-native shop floor" underserved area.

### Tasks

#### 5.1 — Production steps & routing

**What**: Define ordered production steps per job item with machine assignment.

**Design**:
- `production_steps` (suggestion-1 Module 6): `stepOrder`, `stepType` enum, `machineId`, `status` (`pending → scheduled → in_progress → paused → complete | failed`), `estimatedDurationMinutes`, `actual*`.
- `POST /jobs/:id/items/:itemId/steps` (or auto-generated from template route). Steps may run in parallel (no strict order dependency unless `dependsOnStepId` set — supports the parallel routing case from suggestion-4 without a graph DB).

**Testing**:
- `Integration: create 3 steps; complete out of order allowed unless dependency set`.
- `Unit: step transition pending→complete without start → 409 (must start first)`.

#### 5.2 — Scan endpoint & real-time push

**What**: A low-latency endpoint recording shop-floor scans and broadcasting status.

**Design**:
- `POST /floor/scan` `{ jobId|barcode, stepId, scanType: 'start'|'pause'|'resume'|'complete'|'reject', deviceInfo }` (operator role).
- Writes `production_scans` row, transitions the step, may transition the job (all steps complete → `JobReadyForDelivery` emitted), records `actual_start/end`, all in one transaction with an outbox event.
- Socket.IO gateway broadcasts to rooms `org:{id}:job:{jobId}` and `org:{id}:floor`. PWA queues scans offline in IndexedDB and replays with the original `scanned_at` timestamp + idempotency key.

**Testing**:
- `Integration: start then complete scans → step.actualDurationMinutes computed`.
- `Integration: completing final step emits JobReadyForDelivery`.
- `Integration (Socket.IO): scan → connected back-office client receives job:update event`.
- `Integration: replayed offline scan with duplicate idempotency key → no double record`.

#### 5.3 — Material usage & time capture

**What**: Record actual consumed materials and labour time against a job for actual-cost tracking.

**Design**:
- `job_material_usage` and `job_time_entries` (suggestion-1). `POST /jobs/:id/material-usage` decrements `materials.currentStock` and emits `MaterialConsumed` (→ inventory + reorder check in Phase 7). `POST /jobs/:id/time-entries` computes `totalCost` from the labour rate.
- `jobs.actualCost` recomputed on each usage/time event.

**Testing**:
- `Integration: record material usage → stock decremented, actualCost increased, MaterialConsumed emitted`.
- `Integration: time entry with labour rate → totalCost = hours × rate`.
- `Integration: usage exceeding stock → allowed but emits ReorderPointReached / negative-stock warning`.

---

## Phase 6: Proofing, Artwork & Preflight

### Purpose
Handle commercially sensitive artwork securely and manage the proof approval loop with versioning and automated chasing. Add automated PDF/X preflight — an AI-augmentation candidate and a differentiator. After this phase a job can collect artwork, run preflight, and obtain customer proof approval.

### Tasks

#### 6.1 — Artwork upload via presigned URLs

**What**: Secure large-file upload/download without proxying through the API.

**Design**:
- `POST /jobs/:id/artwork/presign` → returns S3 presigned PUT URL + the `artwork_files` row (status `pending`, version assigned). Client uploads directly to S3. `POST /artwork/:id/complete` confirms and triggers preflight.
- `artwork_files` model from suggestion-1 Module 8: `fileType`, `widthMm`, `resolutionDpi`, `colourSpace`, `hasBleed`, `pdfXCompliant`, `iccProfile`, `version`, `isCurrent`.
- Access control: download presign only for users in the owning org or the customer who owns the job (OWASP A01 broken access control). Per-customer S3 prefix.

**Testing**:
- `Integration: presign returns time-limited URL; expired URL → S3 403`.
- `Integration: customer A cannot presign-download customer B's artwork → 403 problem`.
- `Integration: second upload to same item → version=2, prior isCurrent=false`.

#### 6.2 — Automated preflight (PDF/X-4)

**What**: A worker that inspects uploaded PDFs and produces a preflight report.

**Design**:
- BullMQ `preflight` job (triggered by `ArtworkUploaded`): runs Ghostscript/MuPDF to extract page size, DPI of placed images, colour space, bleed box vs trim box, font embedding, and checks against ISO 15930-7 (PDF/X-4) markers.
- Report (JSONB, Zod `PreflightReportSchema`): `{ status: 'passed'|'warnings'|'failed', checks: [{ id, severity, message, page? }] }`. Checks: `MIN_DPI` (< 150 warn / < 100 fail for the substrate), `BLEED_MISSING`, `RGB_IN_CMYK_JOB`, `FONT_NOT_EMBEDDED`, `NOT_PDFX4`.
- Updates `artwork_files.preflightStatus/Report`, emits `ArtworkPreflightCompleted`.

**Testing**:
- `Fixture: tests/fixtures/good-pdfx4.pdf → status passed`.
- `Fixture: low-res-72dpi.pdf → MIN_DPI fail`.
- `Fixture: rgb-artwork.pdf on CMYK job → RGB_IN_CMYK_JOB warning`.
- `Fixture: missing-bleed.pdf → BLEED_MISSING warning`.

#### 6.3 — Proof loop with versioning & chasing

**What**: Send proofs, capture approve/reject/revision with annotations, and auto-chase non-responders.

**Design**:
- Models `proofs`, `proof_annotations` (suggestion-1). Proof status: `pending → sent → viewed → approved | rejected | revision_requested`.
- `POST /jobs/:id/proofs` (from current artwork) → emits `ProofCreated`; `POST /proofs/:id/send` → emits `ProofSentToCustomer` (email in Phase 8).
- Portal: `POST /portal/proofs/:id/approve|reject` `{ reason?, annotations? }`. Approval can advance job to `proof_approved`.
- Chasing: a repeatable BullMQ job scans `sent`/`viewed` proofs older than N days, emits `ProofChased` (email), increments `chaseCount`, capped.

**Testing**:
- `Integration: approve proof → job advances to proof_approved`.
- `Integration: revision_requested with annotations → annotations persisted, new proof version expected`.
- `Integration: chase scheduler picks proof older than threshold once per interval, respects cap`.

---

## Phase 7: Inventory & Purchase Orders

### Purpose
Track stock accurately from the consumption events already emitted, raise low-stock alerts, and turn them into purchase orders. After this phase inventory is self-maintaining and procurement is triggered automatically.

### Tasks

#### 7.1 — Stock ledger & reorder alerts

**What**: Maintain stock from `MaterialConsumed`/receipts and raise reorder alerts.

**Design**:
- A `stock_movements` ledger table (`materialId, delta, reason: 'consume'|'receive'|'adjust'|'waste', referenceId, createdAt`) is the source of truth; `materials.currentStock` is a cached running total updated transactionally (reconcilable by summing the ledger).
- Subscriber on `MaterialConsumed` writes a `consume` movement; when `currentStock <= reorderPoint` emits `ReorderPointReached`.
- `POST /materials/:id/adjust` and `/receive` write movements.

**Testing**:
- `Integration: consume below reorder point → ReorderPointReached emitted once (debounced until restock)`.
- `Integration: sum(stock_movements) == materials.currentStock invariant after random ops`.
- `Integration: adjust with reason logged in ledger`.

#### 7.2 — Purchase orders

**What**: Create POs (manually or auto from reorder events), receive against them.

**Design**:
- `purchase_orders` + `purchase_order_lines` (suggestion-1 Module 11). A handler on `ReorderPointReached` creates a `draft` PO to the material's preferred supplier with `reorderQuantity`.
- `POST /purchase-orders/:id/receive` `{ lines: [{ lineId, quantityReceived }] }` writes `receive` stock movements, updates PO status `partially_received|received`.

**Testing**:
- `Integration: reorder event → draft PO created to preferred supplier with reorder quantity`.
- `Integration: receive full PO → stock incremented, status received`.
- `Integration: partial receipt → status partially_received, remaining tracked`.

---

## Phase 8: Invoicing, Payments & Accounting Sync

### Purpose
Close the financial loop: generate invoices on job completion, record payments, and sync bidirectionally with QuickBooks Online and Xero. Also stand up the email/SMS notification consumers that earlier phases emit events for. After this phase the estimate→cash cycle is complete.

### Tasks

#### 8.1 — Invoices & payments

**What**: Generate invoices from completed jobs and record payments.

**Design**:
- `invoices`, `invoice_lines`, `payments` (suggestion-1 Module 10). `POST /jobs/:id/invoice` builds lines from job items / quoted totals, status `draft`; `POST /invoices/:id/send` → `sent`, emits `InvoiceCreated`. Payment recording updates `amountPaid`/`balanceDue`, transitions to `partially_paid|paid`, emits `PaymentReceived`.
- `invoice_number` from per-org sequence. Tax computed from customer/org settings.

**Testing**:
- `Integration: invoice from job → lines mirror job items, total matches`.
- `Integration: partial payment → status partially_paid, balanceDue correct`.
- `Integration: full payment → status paid, PaymentReceived emitted`.

#### 8.2 — QuickBooks & Xero sync

**What**: OAuth-connected, queue-driven, idempotent push of invoices/payments to accounting platforms.

**Design**:
- OAuth 2.0 connect flow per provider; tokens stored encrypted per org. `integrations/quickbooks` and `integrations/xero` adapters implement a common `AccountingProvider` interface (`upsertCustomer`, `upsertInvoice`, `recordPayment`).
- BullMQ consumers on `InvoiceCreated`/`PaymentReceived` call the connected provider; store `quickbooksId`/`xeroId` back on the row (idempotent upsert keyed on the external id). Retries with exponential backoff; failures land in a dead-letter set surfaced in an admin view (suggestion-2 resilient-integration principle).

**Testing**:
- `Integration (mocked QBO API): InvoiceCreated → upsertInvoice called, quickbooksId stored`.
- `Integration (mocked): API 500 → retried per backoff; permanent failure → dead-letter`.
- `Integration: replaying same InvoiceCreated → idempotent (no duplicate remote invoice)`.

#### 8.3 — Notifications (email/SMS)

**What**: Event-driven transactional email and SMS.

**Design**:
- Consumers on `EstimateSentToCustomer`, `ProofSentToCustomer`, `ProofChased`, `InvoiceCreated`, `DeliveryDispatched`, `DeliveryCompleted` render templates and send via Nodemailer (SMTP, RFC 5321) / Twilio. Customer/contact `receivesProofs`/`receivesInvoices` flags gate recipients. Dev uses Mailhog.

**Testing**:
- `Integration (Mailhog): ProofSentToCustomer → email delivered to contact with receivesProofs=true only`.
- `Integration (mocked Twilio): DeliveryDispatched → SMS send invoked`.
- `Unit: template renders job number, links, totals correctly`.

---

## Phase 9: Production Scheduling (Gantt)

### Purpose
Add the drag-and-drop, multi-machine, multi-shift Gantt scheduler with database-enforced no-double-booking, setup/changeover/curing handling, downtime blocks, and live WebSocket updates — a top differentiator. After this phase production is planned, not just tracked.

### Tasks

#### 9.1 — Schedule entries with double-booking prevention

**What**: Create/move schedule entries against machines with conflict prevention.

**Design**:
- `schedule_entries` (suggestion-1 Module 7) including the Postgres exclusion constraint:
  `EXCLUDE USING gist (machine_id WITH =, tstzrange(scheduled_start, scheduled_end) WITH &&) WHERE (status <> 'cancelled')`.
- `setupMinutes`, `changeoverMinutes`, `curingDelayMinutes` extend the occupied range. `machine_downtime` blocks are also enforced (a downtime range conflicts with entries).
- Endpoints: `POST /schedule`, `PATCH /schedule/:id/move` `{ machineId, start, end }` → emits `ScheduleEntryMoved`; conflict → 409 problem identifying the clashing entry.

**Testing**:
- `Integration (Testcontainers): overlapping entry on same machine → 409, DB exclusion fires`.
- `Integration: move entry into downtime window → 409`.
- `Integration: setup/changeover minutes extend effective range and trigger conflict accordingly`.

#### 9.2 — Virtualised Gantt UI & read-model cache

**What**: The interactive front-end Gantt and its Redis-backed read model.

**Design**:
- A denormalised `rm_schedule` read model (suggestion-2 Projection 2) maintained by subscribers on schedule events, cached in Redis per org/date-window. The Gantt fetches a window (`GET /schedule?from&to&machineIds`) and subscribes to `org:{id}:schedule` Socket.IO room for deltas.
- Frontend: virtualised timeline (rows=machines, columns=time) with `@dnd-kit` drag; optimistic move with rollback on 409.

**Testing**:
- `Integration: creating an entry updates rm_schedule and pushes delta over Socket.IO`.
- `E2E (Playwright): drag a job bar to another machine → persists, conflicting drop shows error and reverts`.
- `Perf: render 30 machines × 200 entries window stays interactive (virtualisation smoke)`.

---

## Phase 10: Delivery & Installation

### Purpose
Schedule deliveries and multi-day installations with crew/vehicle assignment, capture digital proof-of-delivery, and link it to invoicing — covering the integrated-installation underserved area and Clarity-style POD.

### Tasks

#### 10.1 — Delivery & installation scheduling

**What**: Plan deliveries/installations, assign vehicles/drivers/sub-contractors, group jobs.

**Design**:
- `deliveries`, `delivery_items`, `vehicles` (suggestion-1 Module 9). `deliveryType` includes `installation`; `installationDays`, `subcontractorId` for multi-day jobs. iCalendar (RFC 5545) export endpoint `GET /deliveries.ics`.
- `POST /deliveries`, `POST /deliveries/:id/items`, `POST /deliveries/:id/dispatch` → emits `DeliveryDispatched`.

**Testing**:
- `Integration: dispatch delivery → status dispatched, DeliveryDispatched emitted`.
- `Integration: .ics export contains VEVENT per scheduled delivery`.

#### 10.2 — Proof of delivery & installation sign-off

**What**: Capture signature + photos on the PWA and link to the job/invoice.

**Design**:
- `POST /floor/deliveries/:id/complete` `{ signature (base64), photoKeys[], receivedBy }` (driver role) → stores POD, status `delivered`, advances linked jobs toward `delivered`, emits `DeliveryCompleted` (→ notification + invoice trigger). Photos uploaded via presigned URLs. Installation: per-day `InstallationDayCompleted`, final `InstallationCompleted`.

**Testing**:
- `Integration: complete delivery with POD → job(s) → delivered, DeliveryCompleted emitted`.
- `Integration: photos stored via presigned keys, retrievable only by owning org/customer`.

---

## Phase 11: Reporting & Analytics

### Purpose
Deliver the profitability and utilisation reporting that justifies the platform: estimate-vs-actual margin, press utilisation, estimator accuracy, revenue by period. These feed the AI features in Phase 12.

### Tasks

#### 11.1 — Profitability & utilisation read models

**What**: Maintain `rm_job_profitability` and machine-utilisation projections.

**Design**:
- `rm_job_profitability` and `rm_estimator_accuracy` projections (suggestion-2 Projection 5) updated on `JobCompleted`, comparing estimate cost lines to `job_material_usage` + `job_time_entries`. Utilisation = scheduled/actual machine hours ÷ available shift hours over a period.
- Endpoints: `GET /reports/profitability?from&to&jobType`, `GET /reports/utilisation`, `GET /reports/estimator-accuracy`, `GET /reports/revenue?granularity=month`. CSV export for each.

**Testing**:
- `Integration: completed job → profitability row with correct estimation_accuracy = actual/estimated`.
- `Integration: utilisation excludes downtime hours from denominator`.
- `Unit: revenue rollup groups by month boundary in org timezone`.

---

## Phase 12: AI-Native Features

### Purpose
Ship the headline differentiators no incumbent offers in production: conversational/artwork-driven estimation, pre-acceptance anomaly detection, schedule-rebalancing optimisation, demand forecasting, and an MCP server exposing MIS data to agents.

### Tasks

#### 12.1 — AI estimation assistant

**What**: Generate a draft estimate from a natural-language job description (and optional artwork).

**Design**:
- `POST /ai/estimate` `{ description, attachmentKey? }` → `LlmClient` (tool-calling) is given tools backed by the catalogue (`search_materials`, `list_machines`, `get_template`) and the pricing engine. The model proposes job type, substrate, quantity, dimensions, and production route; the server runs the deterministic `priceLine`/`priceJob` (never lets the LLM compute prices) and returns a draft estimate for human review. System prompt template lives in `ai/prompts/estimation.md`; requests use prompt caching for the catalogue context.

**Testing**:
- `Integration (mocked LLM): "100 A1 posters, gloss, 5 working days" → draft with jobType=poster, per_unit pricing, qty=100, priced by engine`.
- `Unit: LLM-suggested price is ignored; engine price authoritative`.
- `Integration: ambiguous description → returns clarifying questions, no estimate`.

#### 12.2 — Estimate anomaly detection

**What**: Flag quotes that deviate from historical actuals for similar jobs before they are sent.

**Design**:
- On `EstimatePriced`, a consumer computes z-score of margin and unit price vs historical completed jobs of the same `jobType` and similar quantity band (from `rm_job_profitability`). If |z| > threshold, emit `EstimateAnomalyFlagged` with `confidenceScore`, `similarJobs`, and an LLM-generated plain-English rationale. Surfaced as a warning banner before send.

**Testing**:
- `Integration: estimate 50% below historical median margin for job type → flagged with reason`.
- `Integration: estimate within normal band → not flagged`.
- `Unit: cold-start (no history) → not flagged, no error`.

#### 12.3 — Schedule rebalancing optimiser

**What**: Recompute the schedule when a machine goes down or a job is delayed, with downstream impact alerts.

**Design**:
- On `MachineDowntimeRecorded` (or a manual trigger), a BullMQ job loads affected `schedule_entries`, runs a greedy/heuristic re-pack honouring machine capabilities, setup/changeover, shifts, and job due dates, and proposes moves. Applying emits `ScheduleRebalanced` (all moves) and pushes deltas + per-job impact alerts over Socket.IO. Proposal is reviewable before commit.

**Testing**:
- `Integration: downtime block over 3 entries → proposal moves them to capable available machines, respects due dates`.
- `Integration: applying proposal emits ScheduleRebalanced and updates rm_schedule`.
- `Integration: no feasible slot before due date → entry flagged at-risk, not silently dropped`.

#### 12.4 — Demand forecasting & MCP server

**What**: Forecast consumable consumption from the forward order book, and expose an MCP server.

**Design**:
- Repeatable BullMQ job computes per-material avg daily usage and projected days-remaining (`rm_inventory` rolling stats, suggestion-2 Projection 4), factoring scheduled jobs' bill-of-materials; raises early reorder suggestions.
- MCP server (`integrations/mcp`) exposes the standards.md candidate tools: `get_job_status`, `create_estimate`, `get_inventory_levels`, `schedule_job`, `approve_proof` — each delegating to existing services with the caller's auth/tenancy.

**Testing**:
- `Integration: forward order book consuming material X → projected days-remaining drops, early reorder suggested`.
- `Integration (MCP): get_job_status(jobId) returns live status; unauthorised org → denied`.
- `Integration (MCP): create_estimate tool produces same draft as REST AI estimate`.

---

## Phase 13: Equipment Integration (XJDF/JMF) & Hardening

### Purpose
Connect to presses and finishing equipment via CIP4 standards and complete production hardening: security, GDPR, observability, and the Helm deployment. After this phase the platform integrates with the shop's machines and is production-ready.

### Tasks

#### 13.1 — XJDF/JDF ticket export & JMF status ingest

**What**: Emit XJDF job tickets and ingest JMF machine status.

**Design**:
- On `JobCreated`/`ProductionStepDefined`, generate XJDF (CIP4) tickets (JDF fallback for legacy `machine.jdfDeviceId`s) to the machine's hot folder / endpoint. A JMF ingest endpoint `POST /integrations/jmf` parses status messages (start/stop/counter/error) and maps them to scan/step events, closing the loop with shop-floor tracking.

**Testing**:
- `Fixture: JobCreated → XJDF validates against CIP4 schema`.
- `Integration: JMF "process complete" message → corresponding step marked complete`.
- `Unit: legacy device → JDF (not XJDF) emitted`.

#### 13.2 — Security, GDPR & observability hardening

**What**: Address OWASP Top 10, GDPR data lifecycle, and production telemetry.

**Design**:
- Rate limiting on auth and public portal endpoints; HMAC-SHA256 signed + idempotent webhooks for outbound events; CSP/security headers; input validation already via Zod (A03).
- GDPR: per-customer data export endpoint and right-to-erasure that cascades through jobs, artwork (S3 prefix delete), proofs, invoices (subject to legal retention), recorded in `activity_log` (suggestion-1 Module 12).
- OpenTelemetry tracing across HTTP→queue→DB; Prometheus metrics for queue depth, projection/event lag, scan latency; Pino logs with trace correlation.

**Testing**:
- `Integration: erasure request → customer PII removed/anonymised, artwork objects deleted, action audit-logged`.
- `Integration: outbound webhook carries valid HMAC signature; tampered body rejected by verifier`.
- `Integration: brute-force login attempts rate-limited → 429`.
- `Smoke: /metrics exposes queue depth and event-lag gauges`.

#### 13.3 — Deployment (Helm) & docs

**What**: Production Helm chart and operator/API documentation.

**Design**:
- Helm chart for api/worker/web, Postgres (or external), Redis, MinIO/S3 config, secrets, HPA on the api/web. Published OpenAPI 3.1 spec + Redoc docs site; self-host quickstart in README.

**Testing**:
- `CI: helm lint + helm template renders without error`.
- `Smoke: chart deploys to kind cluster; /readyz green`.
- `Docs: generated OpenAPI covers every controller (lint check fails on undocumented route)`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Auth      ── required by everything
    │
Phase 2: Reference Data (CRM & Catalogue) ── requires 1
    │
Phase 3: Pricing Engine & Estimation      ── requires 2
    │
Phase 4: Quote-to-Order & Job Lifecycle   ── requires 3 (introduces event bus/outbox)
    │
    ├── Phase 5: Shop-Floor Tracking       ── requires 4
    ├── Phase 6: Proofing & Preflight       ── requires 4   ┐ can parallel
    ├── Phase 7: Inventory & POs            ── requires 4,5 ┘ (5 feeds 7)
    │
    ├── Phase 8: Invoicing & Accounting     ── requires 4 (notifications used by 6,10)
    ├── Phase 9: Scheduling (Gantt)         ── requires 4,5 ─ can parallel with 6/7/8
    └── Phase 10: Delivery & Installation   ── requires 4,8
         │
Phase 11: Reporting & Analytics            ── requires 5,7,8 (consumes events/projections)
    │
Phase 12: AI-Native Features               ── requires 3,9,11 (uses pricing, schedule, profitability)
    │
Phase 13: Equipment Integration & Hardening ── requires 5,9 (final hardening; ships last)
```

**Parallelism opportunities (after Phase 4 establishes the event bus):**
- Phases 6, 7, and 8 can be built concurrently by separate developers — they share only the event bus and reference data.
- Phase 9 (Gantt) can proceed in parallel with 6/7/8 once Phase 5 exists.
- Within Phase 12, tasks 12.1–12.4 are independent of one another and can be parallelised.

**MVP cut line:** Phases 1–8 deliver the features.md "Must-have (MVP)" set (template estimation, quote-to-order, scan-based tracking, proofing, basic inventory, QuickBooks/Xero invoicing). Phases 9–11 deliver the "Should-have (v1.1)" set (Gantt, wide-format already in pricing engine, delivery/installation, portal, reporting). Phases 12–13 deliver the "Nice-to-have" / differentiator backlog (AI, preflight depth, outsourcing already modelled, equipment integration).

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented and merged behind passing CI.
2. All unit and mocked-integration tests pass; Testcontainers integration tests pass locally and in CI.
3. ESLint and Prettier pass with zero errors; `tsc --noEmit` passes across all touched packages.
4. `prisma validate` passes and a forward migration is committed (no schema drift); `prisma migrate deploy` runs clean on an empty DB.
5. `docker compose up` brings the stack up healthy; new env vars are added to `.env.example`.
6. The phase's headline capability works end-to-end (demonstrated by at least one E2E or full integration test exercising the user-facing flow).
7. New config options and env vars are documented in the README.
8. Every new HTTP route appears in the generated OpenAPI 3.1 document (the docs lint check fails otherwise) and returns RFC 7807 problems on error.
9. New domain events are registered in the event catalogue and have at least one idempotent subscriber test.
10. Tenant isolation is verified for any new tenant-scoped table (a cross-org access test exists and passes).
