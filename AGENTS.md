# AGENTS.md — Halak (Barbershop Booking System)

This file contains general rules for any Agent or developer working in the repository. Each directory has its own `AGENTS.md`, and the one closest to the file you are editing takes precedence when there is a conflict in details. **The golden rules below cannot be overridden by any other file.**

## What the project is
A booking system for a single barbershop in Syria. **No customer accounts**: identity is the phone number, ownership is proven via SMS OTP, and the customer tracks their booking through a secret link. The deposit is paid via Sham Cash manually, and confirmed by the Admin.

```
backend/  NestJS + Prisma + PostgreSQL 16 + Redis/BullMQ   → backend/AGENTS.md
web/      Nuxt 3: customer site (/) + admin dashboard (/admin)   → web/AGENTS.md
mobile/   Flutter: customer app                            → mobile/AGENTS.md
docs/     Documentation, which is the source of truth
deploy/   Compose, Nginx, deployment scripts
```

## Read before you start

| If your task touches… | Read first |
|---|---|
| Anything | [`docs/SRS.md`](docs/SRS.md) §5 (Decisions log D-xx) |
| Booking, states, time slots, deposit, or rescheduling | [`docs/booking-rules.md`](docs/booking-rules.md) |
| OTP, tokens, permissions, logging, or file uploads | [`docs/security.md`](docs/security.md) |
| Tables, constraints, or migrations | [`docs/database.md`](docs/database.md) |
| An endpoint, response shape, or error code | [`docs/api.md`](docs/api.md) |
| Code structure or adding a Module | [`docs/architecture.md`](docs/architecture.md) |
| Tests | [`docs/testing.md`](docs/testing.md) |
| Docker, Nginx, or CI | [`docs/deployment.md`](docs/deployment.md) |

**Order of truth:** the Decisions log in SRS ← the detailed `docs/` files ← the rest of the SRS ← the current code. If the code contradicts the documentation, the documentation is correct, unless the change was intentional, in which case update the documentation in the same change.

## Golden Rules (non-negotiable)

1. **No customer accounts.** No sign-up, no password, no "profile", no Push for customers. Do not add any of this even if it seems useful.
2. **The server is the reference.** Prices, durations, deposit, available slots, and allowed actions (`allowed_actions`) are computed by the Backend. UIs only display, and never re-implement business rules.
3. **Conflict prevention is the database's responsibility.** The Exclusion constraints on `bookings` must not be deleted, weakened, or circumvented. Do not write "check-then-insert" logic that relies on itself alone (D-19).
4. **Every booking state change goes through the state machine** and matches the transition table in booking-rules §2. No direct `UPDATE status` anywhere else.
5. **No secrets in any log, response, or commit.** OTP, Tracking tokens, JWTs, passwords, and full phone numbers in technical logs: forbidden ([security.md §8](docs/security.md#logging)). OTP is stored as HMAC, and tokens as SHA-256.
6. **Time:** storage as `timestamptz` in UTC, and logic and display in `Asia/Damascus` time via a timezone library. No fixed offset, and no `new Date()` inside business logic in the Backend (use `Clock`).
7. **Money** is `numeric(12,2)` in the database, and a string (`"5000.00"`) in the API. Never `float`.
8. **The UI is fully Arabic and RTL.** Every user-facing string is in Arabic and comes from translation files; no strings written directly inside components.
9. **Migrations that have been applied must not be edited.** No `prisma db push`, and no `migrate reset` outside your own machine.
10. **Do not invent a business rule.** If you cannot find the answer in the documentation, add a question to SRS §11 (Open questions), apply the safest option, and state that explicitly in the change description.

## Stop and ask before you…
- Change the `bookings` table, the list of states, or the conflict constraints.
- Change any security value: token lifetimes, rate limits, storage method, or redaction.
- Add a new major dependency, external service, or tracking SDK.
- Delete data, or write a migration that drops a column or table.
- Break an existing API contract in a backward-incompatible way.

## Language and style
- Code, identifiers, code comments, and commit messages: **in English**.
- Documentation in `docs/`, UI strings, error `message` texts, and SMS texts: **in Arabic**.
- Error codes (`code`) in English as `UPPER_SNAKE`, and from the list in api.md.
- JSON over the wire uses `snake_case` (D-25).

## Workflow
- Branches: `feat/<scope>-<short>`, `fix/…`, `chore/…`, `docs/…`. Commits follow Conventional Commits, e.g. `feat(bookings): add reschedule approval`.
- One small, focused change per PR. If it changes behavior, **update the relevant documentation in the same PR**.
- A new or modified endpoint means: update `docs/api.md`, `backend/openapi.json` (generated), and the generated Web types.
- A new decision that resolves a conflict means: a new row `D-xx` in SRS §5, and an old number is never reused.

## Commands

| Area | Command |
|---|---|
| Local environment | `docker compose -f deploy/docker-compose.dev.yml up -d` |
| Backend | `cd backend && npm ci && npm run prisma:migrate && npm run seed:dev && npm run start:dev` |
| Backend (checks) | `npm run lint && npm run typecheck && npm test && npm run test:e2e` |
| Web | `cd web && npm ci && npm run gen:api && npm run dev` |
| Web (checks) | `npm run lint && npm run typecheck && npm run test && npm run test:e2e` |
| Mobile | `cd mobile && flutter pub get && dart run build_runner build -d && flutter run` |
| Mobile (checks) | `flutter analyze && flutter test` |

If a command here does not actually exist in `package.json` or `pubspec.yaml`, add it under this exact name and do not invent an alternative name.

## Definition of Done
- [ ] The code matches the documentation, or the documentation was updated along with it.
- [ ] The required tests from [testing.md](docs/testing.md) are written and passing, and coverage has not dropped below the threshold.
- [ ] lint, typecheck, and analyze pass with no new errors or warnings.
- [ ] No secrets, no leftover `console.log` or `print`, and no `TODO` without a reference.
- [ ] New strings are in Arabic, render correctly in RTL, and have been tested at 320px width for the customer.
- [ ] The PR description states: what changed, why, which D-xx or Q-xx it touches, and how it was tested.

## Subagents

- Before every PR: call `@reviewer` on the diff.
- Before every `prisma migrate dev`: call `@migration-reviewer` on the migration file.
- When adding an endpoint or a state transition: call `@test-writer`.
- Before every Release: call `@security-auditor` (according to Checklist §13).