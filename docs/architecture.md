# Architecture and Technical Design

A small system: one barbershop, roughly 500 bookings per day at peak, and a handful of admins. We therefore choose **operational simplicity**: a single VPS, a modular Monolith, and a single database. Complexity is allowed in only two places: booking correctness (concurrency) and security.

---

## 1. Context Overview

```
 ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │ Flutter app  │   │ Customer web │   │ Admin (web)  │
 │ (customer)   │   │ Nuxt  /      │   │ Nuxt /admin  │
 └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
        │ HTTPS             │                  │  + SSE
        └──────────────┬────┴──────────────────┘
                       ▼
               ┌───────────────┐        ┌────────────────┐
               │     Nginx     │───────▶│  web (Nuxt SSR)│
               │ TLS, limits,  │        └────────────────┘
               │ log redaction │
               └───────┬───────┘
                       │ /api/*
                       ▼
               ┌───────────────┐   enqueue   ┌───────────────┐
               │  api (NestJS) │────────────▶│ worker(NestJS)│
               └──┬─────┬───┬──┘   Redis     └──┬─────┬──────┘
                  │     │   │                   │     │
          ┌───────┘     │   └──────┐            │     └────────▶ SMS provider(s)
          ▼             ▼          ▼            ▼
   ┌────────────┐ ┌─────────┐ ┌─────────┐  (same PG/Redis/MinIO)
   │ PostgreSQL │ │  Redis  │ │  MinIO  │
   └────────────┘ └─────────┘ └─────────┘
```

| Container | Responsibility | Does not |
|---|---|---|
| `nginx` | TLS, routing, `limit_req`, request size limit, access log redaction, and Nuxt static files | Business logic |
| `web` | Nuxt 3 SSR for customer, SPA for admin (D-02) | Connect directly to the database or Redis |
| `api` | HTTP API, SSE, business logic, database writes | Send SMS synchronously inside the request |
| `worker` | BullMQ (SMS, reminders, cleanup), and the Sweeper with an internal timer | Receive HTTP |
| `postgres` | The single source of truth for all state | — |
| `redis` | Queues, rate counters, consumed `jti`, Idempotency keys (TTL 24 hours), Slots and settings cache, and Pub/Sub for SSE. **Losing it loses no business state** | Store OTP or any booking state |
| `minio` | Receipts (private and encrypted), and barber and service images (public read) | — |

`api` and `worker` are **a single Docker image** with two different run commands (`node dist/main.js` and `node dist/worker.js`).

## 2. Repository Structure (D-01)

```
halak/
├── AGENTS.md
├── docs/                      ← these documents (source of truth)
├── backend/
│   ├── AGENTS.md
│   ├── prisma/
│   │   ├── schema.prisma
│   │   ├── migrations/
│   │   └── seed/
│   ├── src/
│   │   ├── main.ts            ← HTTP
│   │   ├── worker.ts          ← BullMQ processors + schedulers
│   │   ├── cli.ts             ← admin:create, admin:reset-password, tracking:rotate-all, db:check-constraints
│   │   ├── config/            ← zod env schema
│   │   ├── common/            ← guards, filters, interceptors, pipes, decorators, errors
│   │   ├── infra/             ← prisma, redis, storage (minio), clock, logger
│   │   └── modules/
│   │       ├── otp/  sessions/  tracking/  customers/
│   │       ├── admin-auth/  admins/  audit/
│   │       ├── catalog/       ← barbers, chairs, services, barber-services
│   │       ├── scheduling/    ← schedules, blocked-slots, availability (slots engine)
│   │       ├── bookings/      ← create, state machine, reschedule, admin actions
│   │       ├── payments/  reviews/
│   │       ├── notifications/ ← admin inbox + SSE
│   │       ├── sms/           ← providers, templates, queue
│   │       ├── jobs/          ← sweeper, reminders, cleanup
│   │       ├── reports/  settings/
│   │       └── dev/           ← /__dev (loaded only outside production)
│   ├── test/                  ← e2e + concurrency + fixtures
│   ├── scripts/check-constraints.sql
│   └── openapi.json           ← generated and committed
├── web/
│   ├── AGENTS.md
│   ├── pages/ (index, barbers, book/, t/, track, about, terms, privacy, admin/**)
│   ├── components/{customer,admin,shared}/
│   ├── composables/  stores/  utils/  types/api.d.ts (generated)
│   ├── layouts/{default,booking,admin,admin-auth}.vue
│   └── nuxt.config.ts
├── mobile/
│   ├── AGENTS.md
│   └── lib/{app, core, features/*}
├── deploy/                    ← docker-compose.*.yml, nginx/, scripts/ (see deployment.md)
└── .github/workflows/
```

## 3. The Backend

### 3.1 Layers inside each Module
```
controller  → validates DTO + Guards, then calls the service, then converts the result to a Response DTO
service     → orchestration: transaction, calling policies, writes, and emitting events after COMMIT
policy      → pure functions with no I/O: the rules (is the transition allowed? can it be cancelled now? what is the deposit?)
repository  → optional, only for complex raw queries (slots, sweeper)
```
- **Business logic lives in pure policies** that receive `now` as a parameter. This is what makes booking-rules testable without a database.
- **The state machine** is in `bookings/booking-state-machine.ts`: the transition table from [booking-rules §2](booking-rules.md#transitions) as Data, not scattered `if`s. Every state change in the system goes through `BookingTransitionService.transition(id, to, actor, ctx)`.
- **Transactions**: interactive `prisma.$transaction(async tx => …)`. Everything that is emitted after success (SMS, SSE, cache invalidation) is collected in `afterCommit[]` and executed only after COMMIT.

### 3.2 Cross-cutting elements
| Element | Implementation |
|---|---|
| Settings | `SettingsService`: read from the database, with an in-memory cache (30 seconds) + Redis, invalidated on PATCH, and validated with zod |
| Time | `Clock` (injectable), and `luxon` with `Asia/Damascus`. `new Date()` is forbidden in policies |
| Errors | `DomainError(code, httpStatus, details)`, with a `GlobalExceptionFilter` that converts to the api.md shape, and maps Prisma errors (`P2002` and `23P01`) to known codes |
| Logging | `nestjs-pino` with redact ([security.md §8](security.md#logging)), and a `request_id` per request |
| Audit | An explicit `AuditService.record(...)` inside the Transaction (not a global Interceptor that guesses old values) |
| Internal events | `EventEmitter2` inside the process for post-COMMIT events, with Redis Pub/Sub to distribute SSE across all `api` replicas |
| Queues | BullMQ: `sms`, `reminders`, `maintenance`. Deterministic `jobId` keys to prevent duplication (e.g. `reminder:{booking_id}:{start}`) |

### 3.3 Booking creation flow
```
Client ─POST /bookings─▶ BookingSessionGuard ─▶ IdempotencyInterceptor
   ─▶ BookingsController.create(dto)
   ─▶ CreateBookingService
        ├ policies: validate window, lead time, alignment, services…
        ├ tx: advisory lock → overlap check → chair pick → upsert customer
        │     → insert booking/services/history/audit
        ├ on 23P01 (chair) → retry once
        └ afterCommit: consume jti, invalidate slots cache, enqueue sms(booking_created)
   ─▶ 201 { booking, tracking_token, payment_instructions }
```

### 3.4 SMS
- Links in messages are built in the Worker by decrypting `tracking_token_enc` (D-08), so no tracking token ever passes through the Queue.
- Interface `SmsProvider { send(to, text): Promise<{ id, segments }> }`, with implementations: `fake`, `primary`, `fallback`, and `http-generic` (a template for a local provider).
- `SmsService.enqueue(template_key, booking_id | phone, params)` ← Queue ← Worker builds the text from the template, sends it, and logs it in `sms_logs`.
- On failure: 3 attempts on the primary with exponential backoff, then 2 attempts on the fallback, then `failed` with an admin alert if the message is `booking_created` or `booking_confirmed`.
- Sending happens after COMMIT. If enqueueing fails (Redis down), the error is logged to Sentry and the request continues. The customer already received the tracking link in the response. **A conscious decision: no Outbox table in V1.**

## 4. The Web (D-02)

- Nuxt 3, TypeScript strict, Pinia, `openapi-fetch` with generated types, and VeeValidate + Zod.
- `routeRules`: `'/admin/**'`, `'/book/**'`, `'/t/**'`, and `'/track'` are all `ssr: false`. Tracking pages are not rendered on the server, so the token does not pass through the Nuxt server or any cache. `/t/**` and `/track` also carry `Referrer-Policy: no-referrer` and `X-Robots-Tag: noindex`. Public pages (`/`, `/barbers/**`, `/about`…) use SSR for SEO.
- Customer: Tailwind (with native RTL via `rtl:`/`ltr:` and the logical properties `ms-`/`me-`/`ps-`/`pe-`). **No need for `tailwindcss-rtl`.**
- Admin: Ant Design Vue 4 with `ConfigProvider direction="rtl" locale={arEG}`, loaded only in the admin layout so it does not enter the customer bundle.
- Time: `dayjs` with the `utc` and `timezone` plugins.
- SSE: `EventSource` in an admin-only plugin, and it reconnects with a new ticket.

## 5. Mobile

- Flutter 3 (stable), Riverpod, `go_router`, `dio`, `freezed`/`json_serializable`, and `flutter_secure_storage`.
- **No Push, and no Firebase SDK at all** (D-21).
- Deep links: App Links (Android) and Universal Links (iOS) for `https://{APP_DOMAIN}/t/*`, with the files `assetlinks.json` and `apple-app-site-association` served by Nginx.
- Details in [`mobile/AGENTS.md`](../mobile/AGENTS.md).

## 6. Technical Decisions (A-xx)

These are implementation decisions that do not change functional behavior, unlike the D-xx decisions in the SRS.

| # | Decision | Reason |
|---|---|---|
| A-01 | `api` and `worker` are one image with two entry points | Simpler deployment, and shared code |
| A-02 | `nestjs-pino` instead of Winston | Built-in redact and higher performance |
| A-03 | Luxon in the Backend, dayjs in the web, and the `timezone` package in Flutter | Full timezone support |
| A-04 | `sharp` for image processing, and `file-type` for type detection | security §7 |
| A-05 | GlitchTip or Sentry (compatible with the same SDK) | Q-05 |
| A-06 | The Sweeper is an internal timer (`@nestjs/schedule`) in the worker, with `pg_try_advisory_lock` so only one runs each minute. No BullMQ and no Redis | It stays correct even with more than one worker and with Redis loss (D-20) |
| A-07 | `openapi-typescript` + `openapi-fetch` in the web | Accurate types without a huge generated codebase |
| A-08 | No Outbox table in V1 | SMS is not a source of truth, and the customer already received the link in the response |
| A-09 | Phone test vectors in a single file: `backend/test/fixtures/phone-vectors.json`, read by the web and mobile tests via a relative path | Prevents the three apps from diverging |

## 7. Performance and Capacity

- The expected load is small, and a VPS with 4 vCPU and 8GB is enough. Postgres: `shared_buffers=1GB`, `max_connections=50`, and Prisma pool = 10 per process.
- Slots computation: 3 to 4 indexed queries + a 30-second cache. Target < 300ms.
- Image sizes: receipts ≤ 2000px WebP, and barber images at 3 generated sizes on upload (160, 480, and 960).
- No ORM N+1: explicit `include`/`select`, and query inspection in performance tests for public endpoints.

## 8. Future Expansion (not required in V1, but we do not prevent it)

- **Sham Cash API**: a `PaymentVerifier` interface, with its current implementation `ManualVerifier`.
- **Admin Push**: Web Push (VAPID) can be added without changing the notification model.
- **Multi-branch**: requires `salon_id` on most tables, which is **not supported** in V1, and we do not add columns preemptively.