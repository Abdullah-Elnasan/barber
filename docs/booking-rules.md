# Booking Rules

This file is the **single reference** for booking logic: states, transitions, times, deposit, and conflict prevention. Any code that changes a booking state must match the transitions table here literally. The `D-xx` decisions are defined in [SRS.md §5](SRS.md).

---

## 1. States

| State | Active? (holds time) | Final? | Meaning |
|---|:-:|:-:|---|
| `pending_payment` | ✅ | | Booking created, waiting for the deposit receipt upload until `expires_at` |
| `awaiting_approval` | ✅ | | Receipt uploaded, waiting for the admin's decision |
| `confirmed` | ✅ | | Confirmed |
| `reschedule_pending` | ✅ | | **Child booking** representing a new appointment request, while its parent is still `confirmed` (D-10) |
| `rescheduled` | | ✅ | Parent replaced by its child after approval |
| `completed` | | ✅ | Service was performed |
| `no_show` | | ✅ | The customer did not show up |
| `rejected` | | ✅ | Rejected by the admin (regular booking or reschedule request) |
| `expired` | | ✅ | Payment window ended, or the review time passed, or the reschedule request window ended |
| `cancelled_by_customer` | | ✅ | |
| `cancelled_by_admin` | | ✅ | Includes cancellation due to a forced closure |

```
ACTIVE_STATUSES = [pending_payment, awaiting_approval, confirmed, reschedule_pending]
```
This list **must be identical** in: the Exclusion Constraint predicate, the Slots algorithm, and the active-bookings-per-phone limit check. A test verifies the match (see [testing.md](testing.md)).

<a id="transitions"></a>
## 2. Transitions Table

Any transition not in this table is **forbidden**, and returns `409 INVALID_STATE`. Every transition writes a row in `booking_status_history`, inside the same Transaction, with `changed_by_type`.

| # | From | To | Actor | Condition | Side effects |
|---|---|---|---|---|---|
| T-01 | — | `pending_payment` | customer | [§6](#create) | `expires_at`, the token, and SMS `booking_created` |
| T-02 | `pending_payment` | `awaiting_approval` | customer | `now < expires_at`, and a valid receipt | Payment `pending`, SSE alert, and SMS `payment_received` |
| T-03 | `pending_payment` | `expired` | system | `now ≥ expires_at` (Sweeper) | SMS `booking_expired` |
| T-04 | `pending_payment` | `cancelled_by_customer` | customer | action token `cancel` | SMS `booking_cancelled` |
| T-05 | `pending_payment` | `cancelled_by_admin` | admin | — | SMS |
| T-06 | `awaiting_approval` | `confirmed` | admin | `start > now` | Payment `approved`, `customers.total_bookings += 1`, SMS `booking_confirmed`, and reminder scheduling |
| T-07 | `awaiting_approval` | `rejected` | admin | `reason` and `reject_kind` are mandatory | Payment `rejected` if `payment_invalid`, otherwise `refund_due` (Q-01), and SMS `booking_rejected` |
| T-08 | `awaiting_approval` | `cancelled_by_customer` | customer | action token `cancel` | Payment `refund_due` |
| T-09 | `awaiting_approval` | `cancelled_by_admin` | admin | option `refund` (default true) | `refund_due` or `forfeited` |
| T-10 | `awaiting_approval` | `expired` | system | `start − now < approval_cutoff_minutes` | `refund_due`, admin alert, and SMS `approval_expired` |
| T-11 | `confirmed` | `completed` | admin | `now ≥ start` | `barbers.total_bookings += 1`, and `review_request` scheduling |
| T-12 | `confirmed` | `no_show` | admin | `now ≥ start + no_show_grace_minutes` | Payment `forfeited`, and [§9](#no-show) |
| T-13 | `confirmed` | `cancelled_by_customer` | customer | `start − now ≥ cancellation_hours_before`, and action token | `refund_due` (Q-01), and cancelling the active child (T-19) |
| T-14 | `confirmed` | `cancelled_by_admin` | admin | option `refund` | `refund_due` or `forfeited`, and cancelling the active child |
| T-15 | `confirmed` | `rescheduled` | system | Only as an effect of T-17 or T-21 | — |
| T-16 | — | `reschedule_pending` (child) | customer | [§8](#reschedule) | SMS `reschedule_requested`, and SSE alert |
| T-17 | `reschedule_pending` | `confirmed` | admin | `start > now`, and the parent is `confirmed` | Parent ← `rescheduled` (T-15), transferring the **same** token to the child (D-09), and SMS `reschedule_approved` |
| T-18 | `reschedule_pending` | `rejected` | admin | `reason` | The parent does not change, and SMS `reschedule_rejected` |
| T-19 | `reschedule_pending` | `cancelled_by_customer` / `cancelled_by_admin` | system | Only as an effect of T-13 or T-14 on the parent (for forced cancellation see T-22) | — |
| T-20 | `reschedule_pending` | `expired` | system | Passing `reschedule_approval_timeout_minutes`, or `start_child − now < approval_cutoff_minutes`, or `start_parent ≤ now` | SMS `reschedule_rejected` |
| T-21 | — | `confirmed` (child) + parent ← `rescheduled` | admin | Reschedule by the admin (FR-BKM-06), and the parent is `confirmed` | Not counted against the customer's limit, transferring the same token to the child, and SMS `reschedule_approved` |
| T-22 | Any active state (including a child `reschedule_pending`) | `cancelled_by_admin` | admin | Forced closure (`force: true`) of a period overlapping the booking ([§10](#blocks)) | If a payment exists: `refund_due`. Cancelling the parent here also cancels its active child, and cancelling only the child does not touch the parent. SMS `booking_cancelled` or `reschedule_rejected` for the child |

**General rules:**
- Verifying the current state and updating it happen in a single conditional statement, then the number of affected rows is checked:
  `UPDATE bookings SET status = $new ... WHERE id = $id AND status = $expected`. If the count is 0, the result is `409 INVALID_STATE`. No read-then-write.
- Times (`now`) come from an injectable `Clock` service, and `new Date()` is not called directly in booking logic.
- Timestamps: `confirmed_at`, `completed_at`, `cancelled_at`, and `expired_at` are set in the same transition.

<a id="deposit"></a>
## 3. Deposit (D-12, D-13, Q-01)

### 3.1 Calculation
```
per_service_deposit(s) = s.deposit_amount ?? settings.default_deposit_amount
deposit = min( max(per_service_deposit(s) for s in services), total_price )
```

### 3.2 Payment states (`booking_payments.status`)
```
pending ──▶ approved ──▶ refund_due ──▶ refunded
   │            └──────▶ forfeited
   ├──▶ refund_due        (receipt uploaded but not reviewed, then the booking was cancelled or expired: T-07 other, T-08, T-09, T-10, T-22)
   ├──▶ forfeited         (T-09 with refund = false)
   └──▶ rejected          (T-07 payment_invalid)
```
Moving from `pending` to `refund_due` means the admin manually verifies the amount actually arrived before refunding it, because the receipt was not reviewed.
- `refund_due` ← `refunded`: manual only by the admin, with a mandatory `refund_reference`.
- Each root booking has at most one **active** payment. Children (reschedules) inherit the root's payment and do not create payments.
- When re-uploading the receipt (FR-PAY-06) before review: the same `pending` payment is updated, and the old file is deleted.

### 3.3 Effects matrix (default values until Q-01 is resolved)

| Event | Payment state |
|---|---|
| Rejection for reason `payment_invalid` | `rejected` (nothing was collected) |
| Rejection for reason `other` | `refund_due` |
| `awaiting_approval` expiry (T-10) | `refund_due` |
| Customer cancellation in `awaiting_approval` or `confirmed` within the window | `refund_due` |
| Admin cancellation | According to `refund` in the request (default `true` ← `refund_due`) |
| Forced closure of a period (T-22) | `refund_due` if a payment exists. A `pending_payment` booking has no payment |
| No-Show | `forfeited` |
| Completion | `approved` (stays) |

## 4. Time Rules

All calculations use `Asia/Damascus` time with a timezone library, **not a fixed +03:00 offset**, even though Syria has abolished daylight saving since 2022. Storage is in UTC (`timestamptz`).

| Rule | Formula |
|---|---|
| Bookable dates (D-30) | `today_dm ≤ date ≤ today_dm + max_advance_booking_days` |
| Earliest allowed start (D-15) | `start ≥ now + min_lead_minutes` |
| Payment window expiry | `expires_at = created_at + pending_payment_minutes` |
| Customer cancellation window | `start − now ≥ cancellation_hours_before` |
| Reschedule window | `start_parent − now ≥ reschedule_hours_before` |
| Tracking validity expiry | `tracking_expires_at = end + tracking_token_days_after_end` |
| No-Show allowed | `now ≥ start + no_show_grace_minutes` |
| Duration | `total_duration = Σ duration_minutes`, and `end = start + total_duration` |

- Slots start at multiples of `slot_step_minutes` from the beginning of the working window.
- A booking must fall **entirely** within one working window, and must not cross a break or midnight.

<a id="slots"></a>
## 5. Available Slots Algorithm

**Inputs:** `barber_id`, `date` (local date), and `service_ids`.
**Precondition:** the barber is active and not deleted, the services are active and offered by the barber, and the date is within the window (D-30).

```
duration = Σ services.duration_minutes

1. windows ← working hours for this day:
     rows = schedules(barber_id = X, day_of_week = dow(date))
     if rows is empty: rows = schedules(barber_id IS NULL, day_of_week = dow(date))
     if any(row.is_closed) or rows is empty: return []
     windows = [ (date + row.start_time, date + row.end_time) in Asia/Damascus → UTC ]

2. barber_busy ← the span of every active booking for the barber overlapping the day
              ∪ blocked_slots where (barber_id = X) or (barber_id IS NULL and chair_id IS NULL)

3. free = windows − barber_busy           -- set difference of intervals

4. candidates = for each range in free:
                  t = align_up(range.start, step, relative to window start)
                  while t + duration ≤ range.end: yield t; t += step

5. filter: t ≥ now + min_lead_minutes

6. chair check: for each t, there must exist an active chair c such that:
     no active booking on c overlaps [t, t+duration)
     and no blocked_slot on c (chair_id = c) overlaps
   (all chair bookings and closures for the day are loaded once, then checked in memory)

OUTPUT: [{ start: ISO-UTC, end: ISO-UTC, label: "HH:mm" }]
```

- The query: 3 to 4 queries for the whole day, **not one query per Slot**.
- Cache: Redis with key `slots:{barber}:{date}:{sorted service_ids hash}` for 30 seconds, invalidated on any change to the barber's bookings or closures on that date. **The cache is never used in the booking creation path.**
- The list is indicative: booking creation re-validates fully, and the database constraint is the judge.

<a id="create"></a>
## 6. Booking Creation (T-01)

```
POST /bookings   Authorization: Bearer <booking_session>   Idempotency-Key: <uuid>

0. Idempotency: if a saved response exists for the same (phone, key) within 24 hours ← return it as is
1. Validate DTO: barber_id, service_ids (1..5, no duplicates), start_datetime (ISO UTC), customer_name, customer_notes?
2. phone ← from the token (never from the body)
3. Customer: if blocked_until > now ← 403 CUSTOMER_BLOCKED
4. Number of active future bookings for this phone ≥ max_active_bookings_per_phone ← 409 ACTIVE_BOOKING_LIMIT
5. Validate the rules: barber, services, window, lead time, alignment to the step, falling within a working window, and no overlap with a closure
6. Compute duration, total_price, and deposit from the database
7. BEGIN (READ COMMITTED)
   a. SELECT pg_advisory_xact_lock(hashtextextended('barber:' || barber_id, 0))
   b. Verify no active booking for the barber overlaps    ← otherwise 409 SLOT_UNAVAILABLE
   c. Pick the chair (§7)                              ← otherwise 409 SLOT_UNAVAILABLE
   d. upsert customer (phone, full_name = customer_name, last_booking_at)
   e. INSERT booking (status = pending_payment, snapshots, tracking_token_hash, tracking_token_enc, expires_at, ...)
   f. INSERT booking_services (snapshots)
   g. INSERT booking_status_history (null → pending_payment, customer)
   h. INSERT audit (actor = customer)
   COMMIT
   — If the INSERT fails with 23P01 (exclusion):
       If it is the chair constraint and we have not retried yet ← repeat steps c–h once, excluding the chair
       otherwise ← 409 SLOT_UNAVAILABLE
8. After COMMIT: consume the booking_session's jti, invalidate the slots cache, and enqueue SMS `booking_created`
9. Response 201: { booking, tracking_token (raw, only once), payment_instructions }
```

- The advisory lock on the barber serializes bookings **for the same barber** only, so it does not affect overall performance.
- Chair conflicts between different barbers are caught by the chair constraint, and we retry once.
- Booking number: `HK-{YYMMDD}-{4 chars}` in Crockford base32 alphabet (no I, L, O, U), and the date is the local appointment date. A `UNIQUE` constraint exists, and on collision it is regenerated.
- `customer_name` in `customers.full_name` is updated to the last entered name, while the Snapshots in the booking remain as they were.

## 7. Chair Selection (D-18)

```
primary = barber_chairs where barber_id = X and is_primary
          and valid_from ≤ start and (valid_to IS NULL or valid_to > end)
candidates = [primary] + active chairs ordered by display_order, id   (without duplicates)
return first c in candidates where:
    c.is_active
    and no active booking on c overlapping [start, end)
    and no blocked_slot with chair_id = c overlapping
```

<a id="reschedule"></a>
## 8. Rescheduling (D-10)

### 8.1 Customer request (T-16)
Conditions, all on the **parent**:
- State is `confirmed`, and there is no child with state `reschedule_pending`.
- `reschedule_count < max_reschedule_count`. The `reschedule_count` column in the current booking = the number of **T-17** approvals in its chain. It is copied to the child at creation, and increments by 1 only on T-17. An admin reschedule (T-21) copies it without incrementing.
- `start_parent − now ≥ reschedule_hours_before`.
- The new appointment: same barber and same services, satisfies all rules in §4 and §6, **and does not overlap the parent**. The overlap is checked before insertion and returns `409 RESCHEDULE_NOT_ALLOWED` with `reason: overlaps_current`. The database constraint would have rejected it anyway, but the message here is clearer to the customer.

Creation follows the same steps as §6, with:
- `status = reschedule_pending`, `original_booking_id = parent`, and `root_booking_id = chain root`.
- Prices and deposit are copied from the parent's Snapshot, **not from current prices**.
- `expires_at = min(now + reschedule_approval_timeout_minutes, start_child − approval_cutoff_minutes)`.
- No Tracking Token for the child until approval, and the parent's link keeps working and shows "Appointment change request awaiting approval".

### 8.2 Approval (T-17)
In a single Transaction:
1. Parent ← `rescheduled`.
2. Token transfer: `tracking_token_hash` and `tracking_token_enc` move from the parent to the child (the parent becomes NULL first due to the UNIQUE constraint), and `tracking_expires_at = end_child + tracking_token_days_after_end`.
3. Child ← `confirmed`, and `reschedule_count = parent.reschedule_count + 1`.
4. SMS `reschedule_approved` with the same link, which now shows the new booking.

### 8.3 By the admin (T-21)
The admin picks a new appointment for a `confirmed` booking: the child is created directly with state `confirmed`, the parent becomes `rescheduled`, and the same token is transferred to the child as in §8.2. Overlap with the parent's appointment is allowed here, because the parent leaves the active states in the same Transaction. **The order is mandatory**: update the parent first, then insert the child.

<a id="no-show"></a>
## 9. No-Show and Blocking (D-16)

On T-12, in the same Transaction:
```
UPDATE customers
SET no_show_count = no_show_count + 1, no_show_total = no_show_total + 1
WHERE id = $c
RETURNING no_show_count;

if no_show_count >= no_show_threshold:
    blocked_until = now + no_show_block_days
    no_show_count = 0
    block_reason = 'auto_no_show'
    → SMS customer_blocked (after COMMIT)
```
- Blocked ⇔ `blocked_until IS NOT NULL AND blocked_until > now()`. The block ends without any Job.
- Manual block: `blocked_until` = a specific date or `'infinity'`, with a mandatory `block_reason`. Unblock: `blocked_until = NULL`.
- Blocking does not cancel existing bookings, and the admin decides on a case-by-case basis.

<a id="blocks"></a>
## 10. Blocked Slots

- **Scope**: the whole salon (`barber_id` and `chair_id` both NULL), or a barber, or a chair. Filling in both is not allowed.
- **Preview**: `POST /admin/blocked-slots/preview` returns the overlapping active bookings.
- **Creation without `force`** when there is an overlap: `409 BLOCK_CONFLICTS` with the list.
- **With `force: true`**: in a single Transaction, the closure, then applying T-22 to every active overlapping booking (with `refund_due` if a payment exists), then SMS to each customer after COMMIT.
- The closure is not verified by a database constraint, but by the application alone. Therefore, creating a closure for a barber takes the same advisory lock for that barber, and closing the salon or a chair takes the locks of **all** active barbers in a fixed order (by `id`) to avoid deadlock.

## 11. Scheduled Operations (D-20)

| Operation | Mechanism | Frequency | Notes |
|---|---|---|---|
| `pending_payment` expiry (T-03) | Sweeper: a timer inside the worker process (`@nestjs/schedule`) with `pg_try_advisory_lock` to prevent double-running. **No Redis** | Every minute | `UPDATE ... WHERE status = 'pending_payment' AND expires_at <= now() RETURNING` in batches of 100 |
| `awaiting_approval` expiry (T-10) | Sweeper | Every minute | |
| `reschedule_pending` expiry (T-20) | Sweeper | Every minute | |
| Admin reminder for pending requests | Sweeper | Every minute | Once per `admin_approval_reminder_minutes` per booking (`last_admin_reminder_at`) |
| Appointment reminder SMS | Delayed job on confirmation, with state check at execution | — | Verifies `status = confirmed` and that the appointment has not changed |
| Review request SMS | Delayed job on completion | — | |
| SMS sending | Queue `sms`: 3 attempts on the primary provider with exponential backoff, then 2 attempts on the fallback | — | |
| Cleanup of `otp_codes` (> 30 days) and `sms_logs` (> 90 days) | Cron | Daily at 03:00 | Idempotency keys in Redis expire by TTL on their own |
| Deletion of old receipts | Cron | Daily at 03:30 | After `receipt_retention_days` from the end of the booking |

**General principle:** every Job re-reads the state and verifies it before any effect, because Jobs may repeat or be delayed (idempotent). If Redis is lost, the Sweeper stays correct, because all its state is in PostgreSQL.

## 12. Edge Cases That Must Be Tested

1. Two overlapping bookings with different starts (10:00 for 45 minutes, and 10:30) for the same barber.
2. Two exactly consecutive bookings (10:00–10:30, then 10:30–11:00) are allowed (`[)`).
3. Two different barbers at the same time, with only one chair available.
4. Uploading the receipt one second after `expires_at` and before the Sweeper runs: rejected (`409 PAYMENT_WINDOW_CLOSED`), because the check is on time, not just on state.
5. The admin approving `awaiting_approval` in the same second the customer cancels: only one succeeds (conditional update).
6. Rescheduling to an appointment that overlaps the parent: `409 RESCHEDULE_NOT_ALLOWED` (`overlaps_current`).
7. Cancelling the parent while a `reschedule_pending` child exists: both are cancelled.
8. A forced closure that includes a `reschedule_pending` child (T-22): the child is cancelled, and the parent stays `confirmed` if it does not overlap the closure.
12. Restarting Redis or losing its data: the Sweeper continues on the next minute without restarting the worker.
9. Changing a service price after the booking: does not affect the booking or its child.
10. Deactivating a barber with upcoming bookings: allowed with a warning, and existing bookings are not cancelled automatically.
11. Booking on the last day of the window after midnight in Damascus time, not in UTC time.