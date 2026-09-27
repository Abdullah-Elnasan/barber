# Testing Strategy

**The principle:** The most expensive mistakes in this system are three: two bookings at the same time, a leaked token or code, and a wrong booking state. So we test these three deeply on a real database, and accept lighter coverage for CRUD interfaces.

---

## 1. Layers and Tools

| Layer | Backend | Web | Mobile |
|---|---|---|---|
| Unit (pure functions) | Jest | Vitest | `flutter_test` |
| Integration (with real PostgreSQL and Redis) | Jest + Docker Compose `test` | — | — |
| API e2e | Jest + Supertest on a full Nest app | — | — |
| Component / Widget | — | `@vue/test-utils` + Vitest | Widget tests |
| UI E2E | — | Playwright (Chromium + WebKit mobile viewport) | `integration_test` on Emulator |
| Load and concurrency | k6 + a custom concurrency script | — | — |

**Forbidden:** using a Mock for Prisma or PostgreSQL in the booking, OTP, and tracking tests. Database constraints are part of the logic under test. Mocks are allowed only for external boundaries: the SMS provider (`FakeSmsProvider`), time (`FakeClock`), and MinIO in unit tests.

## 2. Environment

- `deploy/docker-compose.test.yml`: Postgres 16, Redis 7, and MinIO on different ports, with `tmpfs` for speed.
- Every integration test file runs on a **separate schema** (`test_<worker_id>`) and applies migrations once. Cleanup is via `TRUNCATE ... CASCADE` between tests.
- `FakeClock`: pins `now` to `2026-10-01T07:00:00Z` (10:00 Damascus time, a Thursday), with `advance(minutes)`.
- `FakeSmsProvider`: stores messages in memory, and extracts `lastOtpFor(phone)` and `lastTrackingLinkFor(phone)` via Regex. **This is the only way to retrieve the OTP in tests** (no reading from the database).
- Test numbers: `+963999000001` … `+963999000999`.

<a id="phone"></a>
## 3. Phone Number Vectors (shared)

Source: `backend/test/fixtures/phone-vectors.json` (A-09). The web and mobile tests read the same file. The default prefix list in the test: `93,94,95,96,98,99`.

| Input | Expected |
|---|---|
| `0933123456` | `+963933123456` |
| `+963933123456` | `+963933123456` |
| `00963933123456` | `+963933123456` |
| `963933123456` | `+963933123456` |
| `0933 123 456` | `+963933123456` |
| `0933-123-456` | `+963933123456` |
| `٠٩٣٣١٢٣٤٥٦` (Arabic-Indic digits) | `+963933123456` |
| `‎0933123456‎` (with directional marks) | `+963933123456` |
| `0999000001` | `+963999000001` |
| `933123456` (9 digits without a zero) | ❌ `PHONE_INVALID` |
| `09331234567` (11 digits) | ❌ |
| `093312345` (9 digits) | ❌ |
| `0833123456` | ❌ |
| `0113123456` (a landline number) | ❌ |
| `+964933123456` (another country) | ❌ |
| `0913123456` (a prefix outside the list) | ❌ |
| `""` · `abc` · `+963` | ❌ |

## 4. Critical Tests (mandatory, no merge without them)

### 4.1 Concurrency and Conflict Prevention
| # | Scenario | Expected |
|---|---|---|
| C-1 | 100 concurrent `POST /bookings` requests for the same barber and time, each with a different session and key | 1 × 201, and 99 × 409 `SLOT_UNAVAILABLE`, and one active row in the database |
| C-2 | 50 concurrent requests with different overlapping starts (10:00, 10:15, 10:30, for 45 minutes) | No overlap: a `tstzrange &&` query on active bookings returns 0 pairs |
| C-3 | Two different barbers, the same time, and one active chair | Only one booking succeeds |
| C-4 | Two exactly consecutive bookings (10:00–10:30, then 10:30) | Both succeed |
| C-5 | The admin confirming and the customer cancelling concurrently on `awaiting_approval` | Only one succeeds, and the other gets 409 `INVALID_STATE` |
| C-6 | Direct insertion into the database (without the application) of two overlapping bookings | The constraint rejects with `23P01` |
| C-7 | Rescheduling to a time that overlaps the parent | 409 `RESCHEDULE_NOT_ALLOWED` (`overlaps_current`) |

C-1 and C-2 run in CI with `Promise.all`, and also run with k6 on staging before launch.

### 4.2 The State Machine
- **A comprehensive table-driven test**: for every pair (from, to) of the 11 × 11 states and for every actor, it verifies that `canTransition` matches the table in [booking-rules §2](booking-rules.md#transitions) literally. The table in the test is copied by hand from the document, and is not imported from the code.
- An API test for every transition T-01 … T-22: success, side effects (the payment, the counters, the log, and the expected SMS), and rejection in the wrong state.
- **`ACTIVE_STATUSES` = the constraint predicate**: a test reads `pg_get_constraintdef` for `bookings_no_barber_overlap` and compares the list against the constant in the code.

### 4.3 Time and Slots
- Boundaries: `min_lead_minutes`, the last day in the window (D-30), a closed day, split periods, a service that does not fit before the end of the period, a barber closure, a closure of the only chair, and a salon closure.
- Midnight in Damascus time versus UTC (21:00Z = 00:00 local).
- Alignment: a `start_datetime` not aligned to the step is rejected.
- Property-based (`fast-check`): for any set of random bookings and closures, every returned Slot does not overlap any of them, and falls within a working period.

### 4.4 OTP and Sessions
- A correct code, a wrong code three times then the correct one (rejected), an expired code, a used code, and a new send invalidating the old.
- The unified response: for a new number, a blocked number, and a provider failure, the response is **byte-for-byte identical** (except for `request_id`).
- Rate limits: the phone, the IP, the resend cooldown, and an IP block after 20 failures.
- A `booking_session` token is not accepted on `/bookings/:id/cancel`, and vice versa. An `action_token` for booking A does not work on booking B. An `action_token` is used only once.
- An OTP with purpose `cancel` without `X-Tracking-Token` or `lookup_session` gives 401.
- `OTP_FIXED_CODE` with `NODE_ENV=production` means boot failure.

### 4.5 Tracking and the Token
- Access with a valid token succeeds, and with an expired, revoked, or rotated token gives 404.
- Resending (from lookup or from the admin) sends the same link, and it stays valid.
- Rotation (from the admin): the old token fails immediately, and the new one arrives by SMS.
- `tracking_token_enc` decrypts to the same token that was returned in `POST /bookings`, and the Worker builds the link from it in `booking_confirmed` and `review_request`.
- Reschedule approval: the same link now displays the child booking (`confirmed`), and is no longer tied to the parent.
- Uploading the receipt after `expires_at` and before the Sweeper runs gives 409 `PAYMENT_WINDOW_CLOSED`.

### 4.6 No Secret Leakage
An e2e test runs the full flow (OTP ← booking ← payment ← confirmation ← reschedule ← review) while capturing **all** pino output, Queue payloads, and the `sms_logs` and `audit_logs` tables, then fails if any of the following appear in them:
- The OTP codes used, any raw Tracking Token, any JWT (`eyJ`), the admin password, or the full phone number in technical logs.

### 4.7 The Database
- `scripts/check-constraints.sql` after migrations.
- The rating Trigger: insertion, hiding, hiding all (the average = 0 and not NULL), and moving a review to another barber.
- `audit_logs`: an `UPDATE` with the application user is rejected.

### 4.8 The Jobs
- The Sweeper processes an expired booking and does not process it twice (two concurrent runs).
- After a Redis `FLUSHALL`: the next Sweeper still expires due bookings.
- A reminder for a booking that was cancelled or whose appointment changed is not sent.

## 5. The Web

- Unit: phone normalization (the shared vectors), money and date formatting in Damascus time, and computing the remaining timer from `expires_at` with clock drift.
- Component: `OtpInput` (paste, delete, auto-submit, and RTL), `SlotPicker`, and `ReceiptUploader`.
- Playwright (against a real Backend with `SMS_PROVIDER=fake`, and the OTP fetched from `/__dev/sms`):
  1. A full booking up to `awaiting_approval`, then admin confirmation, then `confirmed` appearing on the tracking page.
  2. Cancellation from the tracking page with OTP.
  3. Phone lookup, then "Send the link" arrives by SMS with the same link. Then "Rotate the link" from the admin, and the old link displays "invalid".
  4. Admin: login, an SSE alert appears when a receipt is uploaded in another tab, and confirmation.
  5. A Staff admin does not see the settings page (redirect + 403 from the API).
- Automated checks: `axe-core` on the main customer pages (no critical violations), Lighthouse CI on `/` and `/barbers/:id` (Performance ≥ 85 on Mobile, and Accessibility ≥ 90), and RTL snapshots at 320px, 768px, and 1440px widths.

## 6. Mobile

- Unit: normalization (the shared vectors), deep link parsing, and converting API errors to messages.
- Widget: the details screen (instant validation), OTP (6 boxes, the timer, and resend after 60 seconds), the payment screen (the timer, and copy), and tracking (the buttons according to `allowed_actions`).
- Golden tests in RTL for the main booking screens.
- `integration_test`: a full booking against staging, and opening `https://{APP_DOMAIN}/t/{token}` as a deep link.
- Manual on real devices before every release: a low-spec Android 8, a recent Android, and iOS if available (Q-05), with a slow 3G network (Network throttling), and SMS autofill on Android.

## 7. Coverage Criteria and CI Gates

| Scope | Minimum |
|---|---|
| Backend overall (lines) | 70% |
| `bookings/**`, `scheduling/availability/**`, `otp/**`, `sessions/**`, and `tracking/**` (branches) | 90% |
| Web: `utils/`, `composables/`, and `stores/` | 70% |
| Mobile: `lib/core/`, and `lib/features/*/domain/` | 70% |

CI fails on: any test failure, coverage dropping below the threshold, lint/type errors, an `openapi.json` mismatch, a `check-constraints.sql` failure, or a `gitleaks` finding.

## 8. Before Launch (Staging with a real SMS provider)

- [ ] OTP arrives at numbers from both operators within < 10 seconds (20 attempts per operator)
- [ ] The links in the SMS are clickable, and open the app if installed, otherwise the browser
- [ ] The Arabic message renders correctly (UCS-2 encoding) and with the expected number of segments
- [ ] k6: 100 virtual users for 10 minutes on a mix (browsing 80%, slots 15%, booking 5%), with p95 < 500ms and zero 5xx errors
- [ ] C-1 and C-2 on staging
- [ ] Restoring a backup on a clean server
- [ ] The security checklist in [security.md §13](security.md)