# backend/AGENTS.md — NestJS + Prisma

The rules here are added to [`../AGENTS.md`](../AGENTS.md) and do not override them. The architectural reference is in [`docs/architecture.md`](../docs/architecture.md) §3.

## Stack
- Node.js 22 LTS, and TypeScript in `strict: true` mode with `noUncheckedIndexedAccess`.
- NestJS (latest stable major version), Prisma, PostgreSQL 16, Redis 7, BullMQ, `nestjs-pino`, Luxon, `class-validator`/`class-transformer`, `zod` (for config and settings only), `sharp`, and `file-type`.
- Versions are **pinned exactly** in `package.json` (no `^`), and updates happen in a separate PR.
- Jest + Supertest for tests, and `fast-check` for property tests.

## Module structure
```
src/modules/<name>/
├── <name>.module.ts
├── <name>.controller.ts          # admin controllers in a separate file: <name>.admin.controller.ts
├── <name>.service.ts
├── dto/
│   ├── create-<x>.dto.ts         # Request DTOs (class-validator)
│   └── <x>.response.ts           # Response DTOs (explicit, with @ApiProperty)
├── policies/<x>.policy.ts        # Pure functions: (input, now, settings) => result | DomainError
└── __tests__/*.spec.ts
```
Modules and the boundaries between them are in architecture §2. A Module calls another Module through an **exported service**, and never touches another Module's tables directly, except for reads in `reports`.

## Code rules

### Inputs and outputs
- `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })` globally.
- DTO properties use `snake_case` matching the JSON (D-25). Conversion to camelCase happens only at the Prisma call site.
- **Returning a Prisma object from a controller is forbidden.** Always convert to a Response DTO through a `toXResponse()` function, because this prevents leaking `tracking_token_hash`, `password_hash`, and the like.
- `Decimal` is converted to a string via `toFixed(2)`, and dates to `toISOString()`.
- The phone always enters through `normalizePhone()` from `common/phone`, which is the only allowed function for that.

### Errors
- Throw `DomainError` with a code from [api.md §3](../docs/api.md) only, e.g. `throw new DomainError('SLOT_UNAVAILABLE')`. The Arabic message comes from `common/errors/messages.ar.ts`.
- No raw `HttpException` in the services. `GlobalExceptionFilter` maps:
  `P2002` to 409 (depending on the constraint), `23P01` to `SLOT_UNAVAILABLE`, `P2025` to `NOT_FOUND`, and everything else to 500 `INTERNAL`, while logging the error without details in the response.

### Authentication and authorization
- Guards: `AdminAuthGuard`, `RolesGuard`, `BookingSessionGuard`, `LookupSessionGuard`, `ActionTokenGuard(action)`, `TrackingTokenGuard`, and `BookingAccessGuard` (Tracking **or** Lookup for `otp/send`).
- Every controller under `/admin` has `@UseGuards(AdminAuthGuard, RolesGuard)`, and **defaults to `@Roles('owner')`**. Any route allowed for staff is explicitly marked with `@Roles('owner','staff')`.
- Ownership checks happen inside the service and not only in the controller: `assertBookingAccess(booking, principal)`, with 404 for other people's resources.
- decorators: `@CurrentAdmin()`, and `@CurrentCustomer()` (returns `{ phone, tokenType, bookingId?, action? }`).

### Transactions and states
- Every booking state change: `BookingTransitionService.transition(tx, bookingId, to, actor, ctx)`, which applies the table, writes `booking_status_history`, uses `UPDATE … WHERE status = $expected`, and verifies the row count.
- `prisma.$transaction(async (tx) => …, { timeout: 10_000, isolationLevel: 'ReadCommitted' })`.
- Locks: `await tx.$executeRaw\`SELECT pg_advisory_xact_lock(hashtextextended(${'barber:' + id}, 0))\``. For multi-barber locks: locks ordered by ascending `id`.
- External side effects (SMS, SSE, cache invalidation, `jti` consumption) are registered in `ctx.afterCommit.push(fn)`, and executed **only after** the Transaction succeeds. No `await sms.send()` inside a transaction.
- `ACTIVE_STATUSES` is a single constant in `bookings/booking-status.ts`. Do not duplicate the list anywhere.

### Database
- A single Prisma client via `PrismaService`, with `$queryRaw` using tagged templates only. **`$queryRawUnsafe` and `$executeRawUnsafe` are forbidden** (ESLint rule).
- Any migration: `npx prisma migrate dev --create-only --name <snake_name>`, then review the SQL, then add manual constraints with a `-- manual:` comment, then apply. Details in [database.md §5](../docs/database.md).
- After any migration: `npm run db:check-constraints`.
- No `findMany` without `take` in list endpoints, and no deep `include` without `select`.

### Time and settings
- `Clock.now()` only. Policies receive `now: DateTime` (Luxon) as a parameter.
- Local conversion: `DateTime.fromJSDate(d, { zone: 'utc' }).setZone('Asia/Damascus')`.
- `SettingsService.get('pending_payment_minutes')` with types generated from the settings schema. No hardcoded values in code for anything that exists in the settings table ([SRS §9](../docs/SRS.md)).

### Queues and jobs
- BullMQ with a separate connection (`maxRetriesPerRequest: null`). Queue names: `sms`, `reminders`, `maintenance`.
- Deterministic `jobId` to prevent duplication. Every processor is **idempotent**: it re-reads the booking, verifies the state, then executes.
- The Sweeper in `jobs/sweeper.service.ts`: `@Interval(60_000)` inside the worker, takes `pg_try_advisory_lock` (not Redis), then releases it. The last success is stored in the `settings` table under an internal key `_sweeper_last_ok`, and exposed at `/health/ready`. Keys starting with `_` are internal, and `/admin/settings` neither returns nor accepts them. **It does not depend on Redis** (D-20).
- SMS payload: `{ template_key, booking_id?, phone?, params }`. The worker builds the link by decrypting `tracking_token_enc` at send time. The only secret allowed in the payload is the OTP code for the `otp` template, with `removeOnComplete` and `removeOnFail` ([security.md §8](../docs/security.md#logging)).

### Logging
- `Logger` from `nestjs-pino` only. No `console.*` (ESLint rule).
- Do not log any request body. Log identifiers only: `{ booking_id, request_id, phone_hash }`.
- Add any new sensitive field to the `redact` list in `infra/logger` in the same PR.

## Config
`src/config/env.schema.ts` with zod. Boot fails when any variable is missing or invalid. **No `process.env` outside `config/`.**

```
NODE_ENV, PORT, APP_DOMAIN, WEB_ORIGIN
DATABASE_URL, MIGRATION_DATABASE_URL, REDIS_URL
ADMIN_JWT_SECRET, CUSTOMER_TOKEN_SECRET, OTP_PEPPER        # ≥ 32 bytes
TRACKING_TOKEN_KEY                                          # 32 bytes base64 (AES-256-GCM)
MINIO_ENDPOINT, MINIO_ACCESS_KEY, MINIO_SECRET_KEY, MINIO_BUCKET_PRIVATE, MINIO_BUCKET_PUBLIC
SMS_PRIMARY, SMS_PRIMARY_URL, SMS_PRIMARY_API_KEY, SMS_PRIMARY_SENDER
SMS_FALLBACK, SMS_FALLBACK_URL, SMS_FALLBACK_API_KEY, SMS_FALLBACK_SENDER
SMS_DAILY_BUDGET, SMS_ALLOWLIST (staging)
SENTRY_DSN (optional), LOG_LEVEL
OTP_FIXED_CODE (development/test only, otherwise boot fails)
```

## Tests
- Unit tests next to the code (`__tests__/*.spec.ts`), and e2e and integration in `test/` (`*.e2e-spec.ts`, `*.int-spec.ts`).
- Shared utilities in `test/utils/`: `createTestApp()`, `FakeClock`, `FakeSmsProvider` (with `lastOtpFor(phone)`), factories (`makeBarber`, `makeBooking({ status })`…), `asAdmin(role)`, and `withBookingSession(phone)`.
- **No Prisma mock in booking, OTP, or tracking tests.** Use the real database from `docker-compose.test.yml`.
- Every new or modified state transition means updating the table-driven test in `bookings/__tests__/state-machine.spec.ts`.
- The full list of mandatory tests is in [testing.md §4](../docs/testing.md).

## Scripts (`package.json`)
```
start:dev · start:worker:dev · build · lint · typecheck · test · test:int · test:e2e · test:cov
prisma:migrate · prisma:generate · db:check-constraints · seed:base · seed:dev
openapi:export · cli (admin:create | admin:reset-password | tracking:rotate-all | db:check-constraints)
```

## When adding an endpoint (Checklist)
1. Request and response DTOs with Swagger decorators.
2. The correct guard + `@Roles` (for admin) + ownership check (for customer).
3. Business-rule logic in pure policies with unit tests.
4. Audit log for every admin write, and for security events.
5. Rate limit if it is public or sends SMS.
6. e2e: success, 401/403/404, the main 409 errors, and `400 VALIDATION_ERROR` (see [Q-14](../docs/SRS.md), provisional).
7. `npm run openapi:export` and update [api.md](../docs/api.md). Nothing may be generated before [Q-18](../docs/SRS.md) (the `data` envelope) is settled.

## Common mistakes to avoid
- Using `@db.Timestamp` instead of `@db.Timestamptz(3)`.
- Computing "today" in UTC. The first hours of the day in Damascus (00:00–03:00) fall on the previous day in UTC.
- Relying on the slots cache in the booking creation path.
- Sending SMS or SSE before COMMIT.
- Forgetting the chair constraint when retrying after `23P01`, or retrying without a limit. The limit is once.
- Letting `prisma migrate dev` generate a migration that drops a manual constraint without you noticing. Review every SQL.
- Returning a different message from `otp/send` when the number is blocked or does not exist.