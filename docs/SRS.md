# SRS — Halak Barbershop Booking System

| Item | Value |
|---|---|
| Version | **1.2** (supersedes 1.1 dated 2026-09-23) |
| Date | 2026-09-27 |
| Status | Approved for implementation, with open questions in §11 having default values |
| Country / Timezone | Syria / `Asia/Damascus` |
| Technologies | NestJS + Prisma + PostgreSQL 16 + Redis 7 + BullMQ · Nuxt 3 · Flutter |

---

## 0. Sources of Truth and Order of Priority

When any conflict exists between documents, this order is followed:

1. **The decisions log in this document (§5)**: every decision has a number `D-xx` and is referenced from the other files.
2. The detailed files in `docs/`:
   - [`booking-rules.md`](booking-rules.md): states, transitions, and algorithms
   - [`security.md`](security.md): OTP, tokens, and permissions
   - [`database.md`](database.md): schema and constraints
   - [`api.md`](api.md): contracts and errors
3. The rest of this document.
4. The old SRS (v1.1) and the v1.1 task plan: **historical reference only**.
5. The old UI concept file (`halakdesign.md`): **valid for visual identity only** (colors, fonts, and card style). Its screen flows are based on v1.0 which had login, and they are **canceled**. The approved list of screens is in §7.

> **Rule:** If a developer or Agent finds an unresolved conflict, they must not silently choose a solution. They add it to §11 (Open Questions) and apply the safest default value.

---

## 1. Overview

An electronic booking system **for a single barbershop** in Syria. The customer books an appointment with a specific barber for one or more services, pays a deposit via **Sham Cash** manually and uploads a receipt image, then the admin confirms the booking manually.

**The governing principle:** There are no customer accounts. The Syrian phone number is the identity, and ownership is proven via SMS OTP. The customer tracks their booking through a **tracking link** that arrives by SMS.

### 1.1 In Scope (V1)
- Customer website (Nuxt 3) and customer app (Flutter, Android first)
- Admin dashboard (Nuxt 3, within the same web application, see D-02)
- Managing barbers, chairs, services, working hours, and closures
- Booking with conflict prevention, and the deposit via Sham Cash with manual confirmation
- OTP, the tracking link, and phone lookup
- Cancellation, rescheduling, and reviews (via OTP)
- No-Show and blocking, and customer management
- Notifications: **SMS to the customer** and **a real-time alert inside the admin dashboard** (SSE + sound)
- Basic reports and an audit log

### 1.2 Out of Scope (V1)
- Customer accounts and login, Push for customers, and favorites
- A barber app, and multi-branch
- Automatic electronic payment, and automatic deposit refund (refunds are manual, and only recorded)
- Coupons, loyalty, and product sales
- Languages other than Arabic
- Outgoing email (see D-22)

### 1.3 Glossary

| Term | Definition |
|---|---|
| Slot | A proposed start time that fits the total duration of the selected services |
| Active booking | A booking that actually holds time: `pending_payment`, `awaiting_approval`, `confirmed`, `reschedule_pending` |
| Booking Session | A short token issued after OTP for the `booking` purpose, and consumed when the booking is created (D-06) |
| Lookup Session | A short read-only token issued after OTP for the `lookup` purpose (D-06) |
| Action Token | A one-time token bound to a booking and an action (`cancel`/`reschedule`/`review`) (D-06) |
| Tracking Token | A random secret sent in an SMS link, of which only the hash is stored (D-08) |
| Deposit | The deposit |
| Chain | The chain of bookings produced by rescheduling (`original_booking_id`) |

---

## 2. Roles and Permissions

| Permission | Customer | Staff | Owner |
|---|:-:|:-:|:-:|
| Browse barbers, services, and times | ✅ | ✅ | ✅ |
| Create a booking (after OTP) | ✅ | — | — |
| Track their booking via link or lookup | ✅ | — | — |
| Cancel / reschedule / review (via OTP) | ✅ | — | — |
| View all bookings, confirm, reject, cancel, complete, and No-Show | ❌ | ✅ | ✅ |
| Approve or reject a reschedule request, and reschedule a booking from the admin | ❌ | ✅ | ✅ |
| Mark the deposit as "refunded" | ❌ | ✅ | ✅ |
| Manage customers (view, block, unblock) | ❌ | ✅ | ✅ |
| Resend or revoke the tracking link | ❌ | ✅ | ✅ |
| Close time periods (Blocked slots) | ❌ | ✅ | ✅ |
| Manage barbers, chairs, services, and working hours | ❌ | ❌ | ✅ |
| Settings, Staff account management, and the audit log | ❌ | ❌ | ✅ |
| Reports | ❌ | ✅ | ✅ |
| Manage reviews (hide, reply) | ❌ | ✅ | ✅ |

A barber is a data entity only, and has no platform.

---

## 3. Functional Requirements

Priority: **H** = mandatory for launch, **M** = required in V1, **L** = if time permits.

### 3.1 Phone Verification and Sessions (AUTH)

| # | Requirement | Priority |
|---|---|:-:|
| FR-AUTH-01 | Enter a Syrian number in any accepted format, and normalize to `+9639XXXXXXXX` (D-03) | H |
| FR-AUTH-02 | Validate the format on the client and the server, and the prefix according to `allowed_phone_prefixes` (D-04) | H |
| FR-AUTH-03 | Send a 6-digit OTP valid for 5 minutes, and store only the hash (D-05) | H |
| FR-AUTH-04 | Verify the OTP and issue the appropriate token for the purpose (D-06) | H |
| FR-AUTH-05 | Send limits: per phone, per IP, and a 60-second cooldown before resending ([security.md](security.md#rate-limits)) | H |
| FR-AUTH-06 | A unified response to the OTP request that reveals nothing about the number (D-17) | H |
| FR-AUTH-07 | Admin login with email and password, with a 15-minute Access token and a 7-day Refresh token with rotation | H |
| FR-AUTH-08 | The Owner resets a Staff password, and the Owner's password is reset via a CLI command on the server (D-22) | M |
| FR-AUTH-09 | Temporary IP block after repeated OTP failures | H |

### 3.2 Barbers, Chairs, and Services (BAR/CHR/SVC)

| # | Requirement | Priority |
|---|---|:-:|
| FR-BAR-01 | Add a barber (name, image, bio, specialty, display order) | H |
| FR-BAR-02 | Edit a barber, activate or deactivate him, and soft delete | H |
| FR-BAR-03 | Link a barber to a preferred chair with a validity period (`barber_chairs`), and the link is a **preference**, not a constraint (D-18) | H |
| FR-BAR-04 | Specify the services each barber offers (`barber_services`); the absence of any row means he offers all services | M |
| FR-CHR-01 | Add, edit, and deactivate a chair | H |
| FR-CHR-02 | View chair occupancy for the day | M |
| FR-SVC-01 | Add, edit, deactivate, and soft delete a service (name, description, duration in multiples of 5 minutes, price) | H |
| FR-SVC-02 | A service-specific deposit (optional). The actual booking deposit = the largest (D-13) | M |
| FR-SVC-03 | Order the services | L |

### 3.3 Times (SCH)

| # | Requirement | Priority |
|---|---|:-:|
| FR-SCH-01 | Default salon working hours per weekday, with support for more than one period per day | H |
| FR-SCH-02 | Barber-specific hours for a given day, replacing the default for that entire day | H |
| FR-SCH-03 | A weekly day off (e.g. Friday) via an `is_closed` row | H |
| FR-SCH-04 | Close a period for the whole salon, or a barber, or a chair (Blocked slot) | H |
| FR-SCH-05 | When there are active bookings within the closure period: preview the conflicts, then either cancel or close with forced cancellation (`force`) and cancel the conflicting bookings with a deposit refund | H |
| FR-SCH-06 | Advance booking window: from today until today + `max_advance_booking_days` (D-30) | H |
| FR-SCH-07 | A minimum duration between now and the appointment start: `min_lead_minutes` (D-15) | H |

### 3.4 Booking (BK)

| # | Requirement | Priority |
|---|---|:-:|
| FR-BK-01 | Choose a barber, then one or more services he offers | H |
| FR-BK-02 | The **server** computes the duration, price, and deposit from `service_ids`, and does not trust any value sent by the client | H |
| FR-BK-03 | Display the available days and then the available Slots in 15-minute steps ([booking-rules.md](booking-rules.md#slots)) | H |
| FR-BK-04 | Enter the full name (2–100 characters), the phone, and the notes (500 characters maximum) | H |
| FR-BK-05 | OTP for the `booking` purpose before creating the booking | H |
| FR-BK-06 | Automatic chair assignment (D-18) | H |
| FR-BK-07 | Prevent barber and chair conflicts via a database guarantee (D-19) | H |
| FR-BK-08 | Create the booking with state `pending_payment` with `expires_at = now + pending_payment_minutes` | H |
| FR-BK-09 | Issue a Tracking Token, return it in the response, and send it by SMS | H |
| FR-BK-10 | Prevent booking for a blocked number (403) (D-17) | H |
| FR-BK-11 | A maximum for active future bookings per phone: `max_active_bookings_per_phone` (default 2) | H |
| FR-BK-12 | A mandatory `Idempotency-Key` on booking creation | H |

### 3.5 Payment (PAY)

| # | Requirement | Priority |
|---|---|:-:|
| FR-PAY-01 | Display the Sham Cash number, the account name, the amount, and the deadline timer | H |
| FR-PAY-02 | Upload the receipt image (JPEG/PNG/WebP/HEIC, up to 8MB), and the server re-encodes it and strips EXIF data | H |
| FR-PAY-03 | The transaction number (optional, up to 100 characters) | M |
| FR-PAY-04 | The upload is authorized by the Tracking Token and not by an OTP token (D-07) | H |
| FR-PAY-05 | Transition to `awaiting_approval`, and send a real-time alert to the admin | H |
| FR-PAY-06 | The receipt may be re-uploaded as long as the booking is in `awaiting_approval` and has not yet been reviewed (the previous receipt is replaced) | M |
| FR-PAY-07 | Track the deposit state: `pending` → `approved` \| `rejected`, then `approved` → `refund_due` → `refunded` \| `forfeited` (D-12) | H |

### 3.6 Admin Booking Management (BKM)

| # | Requirement | Priority |
|---|---|:-:|
| FR-BKM-01 | A list with filters (state, barber, date, phone, booking number) and pagination | H |
| FR-BKM-02 | Booking details with the receipt, the status history, and the chain | H |
| FR-BKM-03 | Confirm, reject with a reason and its kind (`payment_invalid` \| `other`), and cancel with a refund option | H |
| FR-BKM-04 | Complete after the appointment start, and No-Show after `start + no_show_grace_minutes` | H |
| FR-BKM-05 | Approve or reject a reschedule request | H |
| FR-BKM-06 | Reschedule the booking from the admin directly without counting it against the customer's limit | M |
| FR-BKM-07 | A daily and weekly calendar per barber | H |
| FR-BKM-08 | A real-time and sound alert on a new payment receipt or a new reschedule request (D-21) | H |
| FR-BKM-09 | Remind the admin of unreviewed requests after `admin_approval_reminder_minutes` | H |
| FR-BKM-10 | Resend, rotate, and revoke the tracking link | M |
| FR-BKM-11 | Mark the deposit as "refunded" with the transfer transaction number | H |

### 3.7 Tracking (TRK)

| # | Requirement | Priority |
|---|---|:-:|
| FR-TRK-01 | The link `https://{APP_DOMAIN}/t/{token}` displays the state, details, and available actions | H |
| FR-TRK-02 | Token validity until `end_datetime + tracking_token_days_after_end` (D-09) | H |
| FR-TRK-03 | Phone lookup + OTP displays this phone's bookings within the last 30 days and upcoming bookings | M |
| FR-TRK-04 | From the lookup results: "Send me the link again" resends the same link by SMS (D-08) | M |
| FR-TRK-05 | The admin resends the link, or rotates it (a new token + SMS), or revokes it | M |
| FR-TRK-06 | The tracking page is `noindex` and with `Referrer-Policy: no-referrer` | H |

### 3.8 Cancellation and Rescheduling (CNL/RSH)

| # | Requirement | Priority |
|---|---|:-:|
| FR-CNL-01 | The customer cancels their booking in `pending_payment` or `awaiting_approval` at any time, and in `confirmed` at least `cancellation_hours_before` beforehand (D-11) | H |
| FR-RSH-01 | The customer requests rescheduling a `confirmed` booking on condition: the number of reschedules in the chain is less than `max_reschedule_count`, the remaining time ≥ `reschedule_hours_before`, and the same barber and the same services | H |
| FR-RSH-02 | The request immediately books the new appointment with state `reschedule_pending`, and the old appointment remains `confirmed` until the decision (D-10) | H |
| FR-RSH-03 | The new appointment must not overlap the old appointment | H |
| FR-RSH-04 | The deposit moves with the chain and is not paid again, and prices remain as in the original Snapshot | H |
| FR-RSH-05 | A request not reviewed within `reschedule_approval_timeout_minutes` expires (`expired`) and the old appointment remains | H |

### 3.9 Reviews (REV)

| # | Requirement | Priority |
|---|---|:-:|
| FR-REV-01 | One review (1–5) with an optional comment (up to 1000 characters) per `completed` booking, via an Action Token for the `review` purpose (Q-06) | H |
| FR-REV-02 | The review window until the Tracking Token expires | H |
| FR-REV-03 | The average rating and the review count are updated automatically from visible reviews only (D-27) | H |
| FR-REV-04 | The admin hides, shows, or replies to the review | M |
| FR-REV-05 | The reviewer's name is displayed as "first name + first letter of the last name" only | H |

### 3.10 Customers and No-Show (CUS)

| # | Requirement | Priority |
|---|---|:-:|
| FR-CUS-01 | Create or update the customer record when the booking is created (upsert by phone) | H |
| FR-CUS-02 | The customer list, their details, and their booking history | M |
| FR-CUS-03 | Manual blocking with or without a duration, and unblocking, with a mandatory reason | M |
| FR-CUS-04 | Automatic blocking upon reaching `no_show_threshold`, then resetting the current counter (D-16) | H |
| FR-CUS-05 | The block ends automatically at `blocked_until` without any Job (D-16) | H |

### 3.11 Notifications (NOT)

| # | Requirement | Priority |
|---|---|:-:|
| FR-NOT-01 | SMS to the customer according to the catalog in §8 only, and no SMS outside it | H |
| FR-NOT-02 | An in-dashboard notification inbox for the admin, with SSE and sound (D-21, D-29) | H |
| FR-NOT-03 | An SMS reminder before the appointment at `booking_reminder_minutes_before` (a value of 0 disables it) | M |
| FR-NOT-04 | An SMS failure does not fail the operation that triggered it. It is retried via the Queue and logged in `sms_logs` | H |

### 3.12 Reports, Settings, and Audit

| # | Requirement | Priority |
|---|---|:-:|
| FR-RPT-01 | Today's dashboard: the number of bookings by state, pending deposits, and awaiting requests | H |
| FR-RPT-02 | Period reports: bookings, expected revenue, deposits collected, refunded and forfeited, No-Show, and barber performance | M |
| FR-RPT-03 | CSV export (the Excel alternative in V1) | L |
| FR-SET-01 | Edit the settings in §9 within the allowed bounds, with every edit logged in the audit log | H |
| FR-AUD-01 | Log every admin write operation and every security event (D-23) | H |

---

## 4. Non-Functional Requirements

| # | Requirement | Target |
|---|---|---|
| NFR-PER-01 | API response time (p95, excluding file uploads) | < 500ms |
| NFR-PER-02 | Slots computation for one barber for one day | < 300ms |
| NFR-PER-03 | Target load | 100 concurrent users, and 500 bookings per day |
| NFR-PER-04 | Mobile app cold start | < 3 seconds on a mid-range device |
| NFR-SEC-* | All security requirements in [security.md](security.md) | — |
| NFR-SEC-17 | No OTP, Tracking Token, or session token appears in any log ([security.md §8](security.md#logging)) | Zero, and verified by an automated test |
| NFR-AVL-01 | Availability | 99% monthly |
| NFR-AVL-02 | Backup | Daily, with 30-day retention and an off-server copy |
| NFR-AVL-03 | RPO / RTO | ≤ 24 hours / ≤ 4 hours, with a monthly restore test |
| NFR-USA-01 | UI | Fully Arabic, RTL, and Mobile-first, from 320px up to 4K |
| NFR-USA-02 | Number of booking steps up to payment | ≤ 6 screens |
| NFR-USA-03 | Numbers and dates | Latin numerals on input (while accepting Arabic-Indic digits), and dates in Damascus time |
| NFR-CMP-01 | Platforms | Android 8+, iOS 13+ (Q-05), and the last two versions of the major browsers |
| NFR-MNT-01 | Quality | Generated OpenAPI, coverage ≥ 70% overall per [testing.md](testing.md), and mandatory CI |
| NFR-OBS-01 | Monitoring | Structured JSON logs with no secrets, error tracking, and uptime checks |

---

## 5. Decisions Log

Every decision here resolves a conflict or a gap in v1.1. The "Supersedes" column shows what changed.

| # | Decision | Reason | Supersedes |
|---|---|---|---|
| **D-01** | A single **Monorepo**: `backend/`, `web/`, `mobile/`, `docs/` | One API contract, and synchronized changes to docs and code | 3 repositories in the task plan |
| **D-02** | A single Nuxt app for the customer (`/`) and the admin (`/admin`). Admin pages are `ssr: false`, and Ant Design Vue is loaded in admin only | Simpler deployment, and shared components and types | Separate "salon-web" + "admin" in Compose |
| **D-03** | The stored phone format is E.164: `+9639XXXXXXXX`. It is displayed to the user as `09XX XXX XXX` | The format SMS providers expect, and a single unambiguous standard | Stored `09XXXXXXXX` |
| **D-04** | The accepted prefixes come from the `allowed_phone_prefixes` setting, and its default value is settled in Q-02. The Regex after normalization: `^\+9639\d{8}$` | The operator list in v1.1 is unreliable, and the 9-digit format without a prefix was wrongly accepted | v1.1 Regex and the operator list |
| **D-05** | OTP is stored in PostgreSQL as HMAC-SHA256 only, and is not stored in Redis. Redis is for counters only | A single source of truth, atomic attempt-count verification, and a database leak does not expose the codes | Plaintext OTP in Redis and in the DB |
| **D-06** | Three token types after OTP: `booking_session` (20 minutes, consumed on successful booking), `lookup_session` (15 minutes, read-only), and `action_token` (10 minutes, one-time, bound to `booking_id` and the purpose). **Action endpoints do not accept a `code`** | OTP was being consumed twice on cancel and reschedule | Guest Token + `code` in the body |
| **D-07** | Payment receipt upload is authorized by the Tracking Token (`X-Tracking-Token` header) | The Guest Token (30 minutes) expired before the payment window (45 minutes) | GuestTokenGuard on `/payment` |
| **D-08** | Tracking Token: 32 random bytes encoded as base64url (43 characters). It is stored twice: **SHA-256** for lookup, and an **AES-256-GCM encrypted** copy with the `TRACKING_TOKEN_KEY` so the Worker can build the link in later messages (confirmation, reminder, and review request). Resending sends the same link, and **rotation** (a new token and invalidating the old) is a separate explicit action when a leak is suspected | v1.1 requested "hash only" and "resend the link" together, and that is impossible. A database leak alone does not expose the tokens. 43 characters instead of 64 saves SMS cost at the same entropy (256 bit) | 64 hex stored as a hash only |
| **D-09** | Token expiry = `end_datetime + 7 days`. On reschedule approval, **the same token moves** to the child booking, and the expiry is recomputed, so the link in the customer's hand (and in their app) remains valid | v1.1 did not specify what happens to the link on reschedule | — |
| **D-10** | Rescheduling creates a child booking with state `reschedule_pending` that reserves the new appointment, and the parent remains `confirmed`. Approval: the child is `confirmed` and the parent is `rescheduled`. Rejection: the child is `rejected` and the parent is unchanged. Timeout: the child is `expired` | v1.1 had two conflicting models, one of which released the old appointment before the decision | §7.4 vs §7.8 in v1.1 |
| **D-11** | Customer cancellation is always allowed in `pending_payment` and `awaiting_approval`, and in `confirmed` subject to the window. Cancelling the parent atomically cancels the `reschedule_pending` child | §7.3 and the state diagram were contradictory | — |
| **D-12** | The deposit is not refunded automatically. Its state is tracked: `refund_due` (due) ← `refunded` (the admin marks it), and `forfeited` (forfeited). The default rules are in [booking-rules.md](booking-rules.md#deposit) and Q-01 | There was no refund policy at all | — |
| **D-13** | The booking deposit = the largest value among (`service.deposit_amount` or the default) for the selected services, capped at `total_price` | It was not specified for multiple services | — |
| **D-14** | `awaiting_approval` does not expire automatically with time. Instead: an admin reminder every `admin_approval_reminder_minutes`, and if less than `approval_cutoff_minutes` remains before the start ← `expired` and the deposit becomes `refund_due` | There was no deadline, so the appointment could stay reserved until it passed | — |
| **D-15** | `min_lead_minutes = 90`, with a lower bound of `pending_payment_minutes + approval_cutoff_minutes + 15`: an appointment is not booked if it starts before the window can accommodate payment and then review | At a value of 60, a customer paying at minute 31 would reach `awaiting_approval` having already exceeded the review cutoff, and their booking would expire immediately | — |
| **D-16** | Blocking is a single field `blocked_until` (the value `infinity` for a permanent block). The customer is blocked if `blocked_until > now()`. On automatic blocking, `no_show_count` is reset and `no_show_total` remains | There was no automatic unblocking, and the counter would re-block immediately after it ended | `is_blocked` + `blocked_until` |
| **D-17** | Blocking is enforced only at `POST /bookings` (403 `CUSTOMER_BLOCKED`). Sending OTP and tracking are unaffected | Checking blocking at send time reveals the number's state and violates the unified message. After OTP, the customer has proven ownership of the number, and may be informed | Blocking check in `send-otp` |
| **D-18** | Chair selection: the barber's valid primary chair if available, otherwise the first available active chair by `display_order`, otherwise the appointment is unavailable | "Automatic assignment" and `is_primary` were contradictory | — |
| **D-19** | Conflict prevention: (1) `pg_advisory_xact_lock` on the barber inside a Transaction, (2) Exclusion Constraints on the barber and the chair. **No Redis lock and no `FOR UPDATE`**. A `23P01` error means 409 `SLOT_UNAVAILABLE` | `FOR UPDATE` does not prevent phantom inserts, and a Redis lock on the start time does not prevent overlap | The three layers in v1.1 |
| **D-20** | Deadline expiry is handled by a **Sweeper** that runs every minute on a timer inside the worker process, with `pg_try_advisory_lock` to prevent double-running. **It does not depend on Redis in any way.** BullMQ is for reminders and SMS only | Losing Redis meant stuck bookings forever | The `expire-booking` Job in BullMQ |
| **D-21** | No Push in V1. The admin receives SSE alerts inside the dashboard with sound, and the customer receives SMS only. No OneSignal and no FCM | The admin dashboard is a web application, and OneSignal on Android goes through FCM anyway | Firebase/OneSignal |
| **D-22** | No email sending in V1: the Owner resets Staff passwords, and the Owner's password is reset via a CLI command | There is no specified email provider, and no need to complicate it | forgot/reset password via email |
| **D-23** | `audit_logs` with an `actor_type` field (`admin`/`customer`/`system`) and a nullable `admin_id`, with `phone_hash` for security events | `admin_id` was a mandatory FK, so OTP events could not be logged | — |
| **D-24** | All times are `timestamptz` at millisecond precision, and the range is `tstzrange(start, end, '[)')` | `tsrange` without a timezone is dangerous, and `[)` allows two consecutive appointments (10:00–10:30 then 10:30) | `tsrange` |
| **D-25** | JSON over the wire uses `snake_case`, and internal code uses camelCase (Prisma `@map`) | Matches the v1.1 examples, and is suitable for Dart | — |
| **D-26** | `sms_logs` does not store the full message text, but rather `template_key` with the variables after masking secret values. Never an OTP or a tracking link in it | NFR-SEC-17 | A text `message` |
| **D-27** | `rating_avg = COALESCE(AVG, 0)` and `rating_count` are updated by a Trigger from visible reviews only | The v1.1 Trigger returned NULL when all reviews were hidden | — |
| **D-28** | Amounts are `DECIMAL(12,2)` in the `currency_code` setting's currency, and are displayed without decimals | The new currency (removing two zeros) is being settled (Q-03) | `DECIMAL(10,2)` |
| **D-29** | The `notifications` table is the admin's internal inbox only. SMS is logged in `sms_logs` | Mixing the two channels in one table is pointless | A multi-channel `notifications` |
| **D-30** | Bookable dates: from today until (today + `max_advance_booking_days`) inclusive, in Damascus time. A value of 2 means 3 dates | "Two days" was ambiguous, and the design showed 4 days | — |
| **D-31** | `GET /availability/slots` accepts `service_ids` and not `duration` | Prevents the customer from manipulating the duration | `duration=Z` in the query |
| **D-32** | The per-IP OTP limit defaults to 20 per hour (adjustable), and the per-phone limit is the primary control | Syrian operators use CGNAT, so many users share one IP | 10/hour |

---

## 6. Main Flows (Summary)

The full details and states are in [booking-rules.md](booking-rules.md).

### 6.1 Booking
```
Choose barber → choose services → choose day → choose Slot
→ enter name, phone, and notes → POST /auth/otp/send {purpose: booking, phone}
→ enter the code → POST /auth/otp/verify → booking_session
→ summary → POST /bookings (Bearer booking_session + Idempotency-Key)
   ← { booking, tracking_token, payment_instructions }   + SMS containing the tracking link
→ payment screen (timer) → POST /bookings/:id/payment (X-Tracking-Token)
→ awaiting_approval → SSE alert to the admin → confirm or reject → SMS to the customer
```

### 6.2 Tracking and Actions
```
/t/{token} → GET /tracking (X-Tracking-Token) ← the state + allowed_actions
An action (cancel/reschedule/review):
  POST /auth/otp/send {purpose, booking_id} + X-Tracking-Token   ← OTP is sent to the booking's phone
  POST /auth/otp/verify {purpose, booking_id, code}              ← action_token
  POST /bookings/:id/{cancel|reschedule|review} (Bearer action_token)
```

### 6.3 Phone Lookup
```
/track → POST /auth/otp/send {purpose: lookup, phone} → verify → lookup_session
→ GET /tracking/lookup/bookings → a list
→ "Send the link" POST /tracking/lookup/bookings/:id/resend-link ← SMS with the same link
→ Actions: OTP for the action purpose, and proof of access via Bearer lookup_session
```

---

## 7. Approved Screens

### 7.1 Customer (Web + Flutter, the same flow)

| # | Screen | Web path | Notes |
|---|---|---|---|
| 1 | Home | `/` | Barbers, services, and the latest reviews, with a prominent "Track your booking" button. **No greeting by name, and no discount offers** |
| 2 | Barber details | `/barbers/:id` | **No favorites button** |
| 3 | Service selection | `/book/:barberId` | The total is displayed for guidance only, and the server is the reference |
| 4 | Day and time selection | `/book/:barberId/time` | Only days per D-30, and closed days are disabled |
| 5 | Customer details | `/book/details` | Name, phone with instant validation, and notes |
| 6 | Verification code | `/book/verify` | **6 boxes**, resend after **60 seconds**, and autofill support |
| 7 | Summary | `/book/summary` | The day is computed from the date in Damascus time and is never written by hand |
| 8 | Payment | `/t/:token/pay` | Timer, number copy, and image upload |
| 9 | Awaiting confirmation | `/t/:token` | It is the tracking page itself in the `awaiting_approval` state |
| 10 | Tracking | `/t/:token` | The state, the timeline, and the available actions |
| 11 | Cancel / reschedule / review | `/t/:token/cancel`, `/reschedule`, `/review` | OTP is embedded in the screen |
| 12 | Phone lookup | `/track` | Phone, then OTP, then the list |
| 13 | About the salon / terms and privacy | `/about`, `/terms`, `/privacy` | Mandatory, because the system collects the name and phone |

**Flutter:** a bottom navigation bar with 3 tabs: Home, "My bookings on this device" (the list of bookings whose links were saved locally, see [`mobile/AGENTS.md`](../mobile/AGENTS.md)), and "More" (phone lookup, about the salon, terms). **No notifications tab and no account, and no mandatory Onboarding.**

### 7.2 Admin (`/admin`)
Login, today's dashboard, bookings, booking details, calendar, barbers, chairs, services, working hours, closures, customers, reviews, reports, settings (Owner), Staff accounts (Owner), the audit log (Owner), and the notification inbox.

---

## 8. SMS Catalog

Rules:
- Arabic, and as short as possible: an Arabic UCS-2 message fits 70 characters for a single segment, and 67 characters per segment in multi-part messages. The target is ≤ 2 segments.
- No sensitive data, and no full barber name. The link comes at the end of the message.
- `{link}` = `https://{APP_DOMAIN}/t/{token}`, and the date in the format `ddd dd/MM` and the time `HH:mm` in Damascus time.

| Key | Trigger | Text (draft) |
|---|---|---|
| `otp` | OTP request | `رمز التحقق: {code}\nصالح {minutes} دقائق. لا تشاركه مع أحد.` |
| `booking_created` | Booking creation | `حجزك {number} {date} {time}. ادفع العربون {deposit} خلال {minutes}د: {link}` |
| `payment_received` | Receipt upload | `استلمنا إشعار الدفع للحجز {number}. سنؤكد قريباً.` |
| `booking_confirmed` | Confirmation | `تم تأكيد حجزك {number} {date} {time}. التفاصيل: {link}` |
| `booking_rejected` | Rejection | `نعتذر، رُفض الحجز {number}: {reason_short}. {link}` |
| `booking_expired` | Payment window expiry (T-03) | `انتهت مهلة دفع الحجز {number} وأُلغي.` |
| `approval_expired` | Review time passing (T-10) | `تعذّر تأكيد حجزك {number} قبل موعده وأُلغي. سيُعاد العربون. {link}` |
| `booking_cancelled` | Cancellation (either party) | `أُلغي الحجز {number}.{refund_note} {link}` |
| `reschedule_requested` | Reschedule request | `طلب تغيير موعد {number} إلى {date} {time} بانتظار الموافقة.` |
| `reschedule_approved` | Approval | `تم تغيير موعدك إلى {date} {time}. التفاصيل: {link}` |
| `reschedule_rejected` | Rejection or expiry | `لم يُعتمد تغيير الموعد. موعدك الأصلي {date} {time} قائم.` |
| `booking_reminder` | Before the appointment | `تذكير: موعدك {time} اليوم. {link}` |
| `review_request` | After completion | `شكراً لزيارتك! قيّم تجربتك: {link}` |
| `tracking_link` | Resending or rotating the link | `رابط متابعة حجزك {number}: {link}` |
| `customer_blocked` | Automatic blocking | `تم إيقاف الحجز من رقمك حتى {date} بسبب تكرار عدم الحضور.` |

`{refund_note}` = " سيُعاد العربون خلال 3 أيام عمل." when the deposit state is `refund_due`.

---

## 9. Settings (Defaults and Bounds)

They are stored in the `settings` table, edited by the Owner only, and every value is validated within its bounds on the server.

| Key | Type | Default | Bounds |
|---|---|---|---|
| `salon_name` / `salon_address` / `salon_phone` | string | — | — |
| `sham_cash_number` / `sham_cash_account_name` | string | — | Mandatory before launch |
| `currency_code` | string | `SYP` | Q-03 |
| `default_deposit_amount` | decimal | Determined after Q-03 | ≥ 0 |
| `pending_payment_minutes` | int | 45 | 10–120 |
| `approval_cutoff_minutes` | int | 30 | 0–120 |
| `admin_approval_reminder_minutes` | int | 10 | 5–60 |
| `reschedule_approval_timeout_minutes` | int | 120 | 15–1440 |
| `min_lead_minutes` | int | 90 | ≥ `pending_payment_minutes + approval_cutoff_minutes + 15` (D-15) |
| `max_advance_booking_days` | int | 2 | 0–30 |
| `slot_step_minutes` | int | 15 | 5, 10, 15, 30 |
| `cancellation_hours_before` | int | 3 | 0–72 |
| `reschedule_hours_before` | int | 3 | 0–72 |
| `max_reschedule_count` | int | 1 | 0–3 |
| `max_active_bookings_per_phone` | int | 2 | 1–10 |
| `no_show_grace_minutes` | int | 15 | 0–60 |
| `no_show_threshold` | int | 3 | 1–10 |
| `no_show_block_days` | int | 7 | 1–365 |
| `otp_expiry_minutes` | int | 5 | 2–10 |
| `otp_max_attempts` | int | 3 | 3–5 |
| `otp_resend_cooldown_seconds` | int | 60 | 30–300 |
| `otp_rate_limit_per_phone_hour` | int | 5 | 3–10 |
| `otp_rate_limit_per_ip_hour` | int | 20 | 10–100 |
| `otp_ip_block_failures` | int | 20 | 10–100 |
| `otp_ip_block_minutes` | int | 60 | 15–1440 |
| `booking_session_minutes` | int | 20 | 10–30 |
| `lookup_session_minutes` | int | 15 | 5–30 |
| `action_token_minutes` | int | 10 | 5–15 |
| `tracking_token_days_after_end` | int | 7 | 1–30 |
| `allowed_phone_prefixes` | string (CSV) | Q-02 | Digits in the form `9X` |
| `review_request_delay_minutes` | int | 30 | 0–1440 |
| `booking_reminder_minutes_before` | int | 60 | 0 (disabled)–1440 |
| `receipt_retention_days` | int | 180 | 30–730 (Q-07) |
| `min_app_version` | string (semver) | `1.0.0` | Returned in `GET /public/salon`, and the app requests an update if it is older |

The OTP length is fixed (6) and is not a setting, because changing it breaks the UIs. The full name is always mandatory.

---

## 10. Acceptance Criteria for Launch

1. There is no path for customer registration or a customer account.
2. All accepted and rejected phone formats in [testing.md](testing.md#phone) pass on the web, mobile, and server.
3. 100 concurrent requests for the same Slot produce exactly one booking, 99 responses of 409, and zero overlap in the database. The same holds for overlapping requests with different starts.
4. An unpaid booking expires within ≤ 60 seconds after `expires_at` even if Redis is restarted and its data is lost.
5. No OTP, Tracking Token, or Bearer token appears in any log (application, Nginx, `sms_logs`, or Sentry), and this is verified by an automated test on the logs.
6. Every state transition in [booking-rules.md](booking-rules.md#transitions) has a test, and every disallowed transition returns 409 `INVALID_STATE`.
7. A full scenario: booking ← payment ← confirmation ← reschedule ← approval ← completion ← review, with all expected SMS arriving and being clicked in a staging environment with a real provider.
8. Automatic blocking on the third No-Show, then it ends on its own after the duration.
9. A backup restore on a clean server is documented and tested.

---

## 11. Open Questions (Settled by the Project Owner)

| # | Question | Default value applied until settled |
|---|---|---|
| **Q-01** | Deposit policy | Admin rejection for reason `payment_invalid`: nothing is refunded. Any other rejection, an admin cancellation, a forced closure, an expiry in `awaiting_approval`, or a customer cancellation within the allowed window: **full refund**. No-Show: **forfeited**. An admin cancellation carries an explicit `refund` option for late phone-call cases |
| **Q-02** | Accepted prefixes | `93,94,95,96,98,99` (i.e. 093, 094, 095, 096, 098, 099). **The current list at Syriatel and MTN must be verified before launch**, because some of what v1.1 stated about prefix distribution across operators was probably inaccurate |
| **Q-03** | Currency | `SYP` with verification of working in the new currency (after removing two zeros) before setting `default_deposit_amount` and the prices. The v1.1 prices (15,000 and 5,000) appear to be in the old currency |
| **Q-04** | Primary and fallback SMS provider | Chosen after a real test on numbers from both operators (delivery time < 10 seconds, delivery rate > 95%, and Arabic Sender ID support) |
| **Q-05** | Availability of services for an account linked to Syria: VPS, GitHub Actions, Sentry, Google Play, and Apple Developer | The alternative plan is documented in [deployment.md](deployment.md#availability): a regional or local VPS, self-hosted CI, self-hosted GlitchTip, and direct APK distribution |
| **Q-06** | Is OTP necessary for reviews, or is the Tracking Token sufficient? | **We keep OTP** as in v1.1 (higher security and an additional SMS cost) |
| **Q-07** | Retention period for receipt images | 180 days after the booking ends |
| **Q-08** | Domain name | `{APP_DOMAIN}` (an environment variable) |
| **Q-09** | Does each barber need an explicit selection of the services he offers? | No: the absence of rows in `barber_services` means he offers all services |
| **Q-10** | Which language is `docs/` written in? `AGENTS.md` §Language and style requires Arabic, while the seven existing `docs/*.md` files are all in English | Arabic, per the Golden Rule, because a Golden Rule cannot be overridden by another file. Applied to [api.md](api.md) from 2026-09-29, so `docs/` is now mixed. **Settle before translating anything further**: either the remaining files are translated in one dedicated change, or the rule is amended to English and [api.md](api.md) is reverted. Note that this change itself added Q-10…Q-23 to this table in English, to match the existing rows |
| **Q-11** | Are the `allowed_actions` and `deadlines` shapes and field names complete and correctly named? The four action names are documented, but the object shape, the deadline field names, and the `null` convention are not | Exactly the names in [api.md §7](api.md), marked `مؤقّت`. **Only `expires_at` and `tracking_expires_at` are documented field names today**; the other five `deadlines` names and the shape of `allowed_actions` were introduced by [api.md](api.md) and need a ruling. Undecided and absent: resending the tracking link from the tracking page. A UI must not display an action absent from that section |
| **Q-12** | What is the pagination shape? It is required by `FR-BKM-01` but never defined | `{ "data": [...], "meta": { "page", "page_size", "total" } }` alongside the `data` envelope, default 20, maximum 100, with no hidden default sort. Provisional: [api.md §4](api.md) |
| **Q-13** | Is the rate-limit backoff field `retry_after` or `retry_after_seconds`? `security.md` §5 writes `retry_after`; `web/AGENTS.md` §API communication and the OTP body field `resend_after_seconds` in `security.md` §2.4 use the `_seconds` form | `details.retry_after_seconds`, matching the name the client already reads and the existing `resend_after_seconds`, and `security.md` §5 is treated as a typo. Provisional: [api.md §3](api.md) |
| **Q-14** | What is the error code for `ValidationPipe` failure? It is not in any document, and the client is told to branch on `code` only | `400 VALIDATION_ERROR` with `details.fields[] = { field, messages[] }`. `400` rather than `422`, because `backend/AGENTS.md` mandates `ValidationPipe` with no custom `exceptionFactory`, whose default status is 400. Provisional: [api.md §3.2](api.md) |
| **Q-15** | What is the error code for a missing or malformed `Idempotency-Key`, and for a replay with the same key but a different body? `FR-BK-12` makes the key mandatory but names no code | `400 IDEMPOTENCY_KEY_REQUIRED` for both missing/malformed and conflicting replay, so one code covers the whole idempotency contract. Provisional: [api.md §3.2](api.md) |
| **Q-16** | What is the error code for a rejected receipt image? `security.md` §7 defines magic-byte validation for JPEG/PNG/WebP/HEIC and the size limit, but no code | `400 RECEIPT_INVALID` with `details.reason ∈ { unsupported_type, too_large, unreadable }`. One code instead of three, because the three cases are not actionable differently by the client. Provisional: [api.md §3.2](api.md) |
| **Q-17** | What is the error code for an authenticated user lacking the required role, and for a failed `Origin` / `X-Requested-With` check on `refresh` and `logout`? Neither is documented | `403 FORBIDDEN` for both. `403` rather than `401`, because the caller is authenticated and the failure is authorization or CSRF, not authentication. The Unified failure response of `security.md` §4 still applies to login. Provisional: [api.md §3.2](api.md) |
| **Q-18** | Is every successful response wrapped in `data`? `security.md` §2.4 shows one enveloped body, while `booking-rules.md` §6, `architecture.md` §3.3 and `SRS.md` §6.1 all write the 201 shape unwrapped | The `data` envelope, because `testing.md` §4.4 tests one body for byte-identity and the envelope leaves room for `meta` on lists. **If confirmed, record it as a D-xx and update the three unwrapped lines in the same change; if not, `openapi.json` and the generated Web types must be written from the unwrapped shape instead.** This is a wire-format decision, so nothing may be generated before it is settled. Provisional: [api.md §2.1](api.md) |
| **Q-19** | How is the `FR-PAY-06` receipt re-upload in `awaiting_approval` exposed to the client? `allowed_actions.pay` is documented only for `pending_payment`, so a client that renders strictly from `allowed_actions` would hide the upload control and make `FR-PAY-06` unreachable | `pay = true` in `awaiting_approval` while `payment.status = pending`, reusing the existing action instead of adding a fifth one outside the Q-11 set. Provisional: [api.md §7.1](api.md) |
| **Q-20** | What are the wire names of `payment_instructions`? The concept is documented in `FR-PAY-01` and the column and setting names are documented, but the API field names are not | The five names in [api.md §7.3](api.md), marked `مؤقّت`, with the source column documenting the value source rather than the name. Provisional: [api.md §7.3](api.md) |
| **Q-21** | Where does the payment screen get `payment_instructions` after a page reload? It is documented only on the `201` of `POST /bookings`, but the screen is reached by navigation and is reloadable | Undecided. Either `GET /tracking` also returns `payment_instructions`, or a separate endpoint does. Provisional: [api.md §7.3](api.md) |
| **Q-22** | What are the paths of the remaining endpoints? [api.md §8](api.md) indexes only the paths written as literal strings in `docs/`, which excludes the admin surface required by `SRS.md` §7.2, the admin writes behind `409 INVALID_STATE`, the blocked-slot creation behind `409 BLOCK_CONFLICTS`, and the customer catalog reads | Undecided. `POST /admin/blocked-slots` is added to the index as a `مؤقّت` name, because `BLOCK_CONFLICTS` is otherwise unreachable. The rest of the admin surface must be designed as its own slice; `SRS.md` §7.2 cannot be built from the current index. Provisional: [api.md §8](api.md) |
| **Q-23** | May Staff manage blocked slots? `SRS.md` §2 grants the Staff column the permission "Close time periods (Blocked slots)", while `security.md` §4 and `backend/AGENTS.md` make the `/admin` default `owner`-only | `owner` only, pending settlement, because `security.md` is a `docs/` file and outranks §2 of the SRS in the order of truth. This **denies Staff a documented permission until it is settled**, so it must not be treated as final. Provisional: [api.md §8.2](api.md) |

---

## 12. Delivery Plan (Revised)

The total number of subtasks in the v1.1 plan was far larger than the announced durations (Backend ≈ 40 days versus 20 announced, and Web ≈ 28 versus 12). The realistic plan for a team of 3 developers (Backend, Web, and Flutter) with partial DevOps and QA support:

| Weeks | Backend | Web | Mobile |
|---|---|---|---|
| 0 | Foundation (Monorepo, Compose, CI, and testing the SMS provider) | ← | ← |
| 1–2 | Schema and constraints, OTP and SMS, sessions, and basic CRUD | Scaffold, design system, and the generated API client | Scaffold, Theme, and networking |
| 3–5 | The Slots engine, booking, payment, the Sweeper, and admin management | The customer booking flow on Mock/staging | Home, barber, and services |
| 6–7 | Tracking, actions, rescheduling, reviews, and SSE | Tracking, then the admin dashboard (bookings and calendar) | The booking and payment flow |
| 8–9 | Reports, settings, audit, and security hardening | The rest of the admin dashboard | Tracking, deep links, and actions |
| 10–11 | Load and concurrency tests, and fixes | E2E, and fixes | Testing on real devices, and Build |
| 12 | Staging with a real SMS provider, then launch | ← | ← |

**Expected duration: 12 weeks (±2)**, and the critical path is the Backend. The Web and Mobile work against the OpenAPI contract from week 2, with a Mock server. The detailed task plan (v1.1) needs to be regenerated from this document.

---

## 13. Changes from v1.1 (Summary)

- Resolved 32 decisions (§5), including: the reschedule model, the tokens, conflict prevention, the deposit, and blocking.
- Added tables: `admin_refresh_tokens`, `barber_services`, `sms_logs`, bringing the table count to 20 ([database.md](database.md)).
- Added deposit states: `refund_due`, `refunded`, `forfeited`.
- Removed: Push, OneSignal, and FCM, password reset via email, the customer notifications tab, and `legacy_bookings` (there is no old data: the project is new).
- Modified the API: `/auth/otp/*`, `/availability/*`, `/bookings/:id/review` instead of `/reviews`, and passing the token in a Header instead of the path ([api.md](api.md)).