# Database: Schema, Constraints, and Migrations

PostgreSQL 16 with Prisma. **Database constraints are the final guarantee of correctness, not application code.** The `D-xx` decisions are in [SRS.md §5](SRS.md), and the states are in [booking-rules.md](booking-rules.md).

---

## 1. Conventions

| Item | Rule |
|---|---|
| Table and column names | `snake_case`, tables in plural. Prisma: Models in `PascalCase` and fields in camelCase with `@map`/`@@map` |
| Keys | `uuid` with `@default(uuid())`. No sequential numbers are exposed to customers |
| Times | `timestamptz(3)`, i.e. `@db.Timestamptz(3)`, **without exception** (D-24) |
| Daily times (working hours) | `time(0)`, interpreted in `Asia/Damascus` time |
| Money | `numeric(12,2)` (D-28) |
| Phone | `varchar(16)` in E.164 format `+9639XXXXXXXX` (D-03) |
| Deletion | Soft (`deleted_at`) for barbers, services, and admins only. Bookings are never deleted |
| `created_at` / `updated_at` | In every mutable table (`@default(now())` / `@updatedAt`) |
| Enums | Postgres enums via Prisma |

## 2. Enums

```prisma
enum BookingStatus {
  pending_payment
  awaiting_approval
  confirmed
  reschedule_pending
  rescheduled
  completed
  no_show
  rejected
  expired
  cancelled_by_customer
  cancelled_by_admin
}
enum PaymentStatus   { pending approved rejected refund_due refunded forfeited }
enum PaymentMethod   { sham_cash cash other }
enum RejectKind      { payment_invalid other }
enum AdminRole       { owner staff }
enum OtpPurpose      { booking lookup cancel reschedule review }
enum BlockType       { holiday break maintenance manual }
enum ActorType       { admin customer system }
enum BookingChannel  { web mobile admin }
enum SmsStatus       { queued sent delivered failed }
```

## 3. Tables (20 tables)

### 3.1 `customers`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| phone | varchar(16) **UNIQUE** | E.164 |
| full_name | varchar(100) | Last entered name |
| blocked_until | timestamptz NULL | Blocked if `> now()`, and the value `'infinity'` for a permanent block (D-16) |
| block_reason | text NULL | `auto_no_show` or a manual reason |
| no_show_count | int default 0 | Current counter, reset on automatic blocking |
| no_show_total | int default 0 | Historical counter |
| total_bookings | int default 0 | Number of confirmed bookings |
| last_booking_at | timestamptz NULL | |
| anonymized_at | timestamptz NULL | §11 in security.md |
| created_at, updated_at | | |

No email, no password, no `gender`, and no `is_phone_verified`: the existence of the record means the phone was verified via OTP.

### 3.2 `admins`
`id`, `email` (UNIQUE, citext), `password_hash`, `full_name`, `role AdminRole`, `is_active`, `session_version int default 0`, `last_login_at`, `created_at`, `updated_at`, `deleted_at`.

### 3.3 `admin_refresh_tokens`
`id`, `admin_id FK`, `family_id uuid`, `token_hash char(64) UNIQUE`, `expires_at`, `rotated_at NULL`, `revoked_at NULL`, `ip inet`, `user_agent text`, `created_at`. Index on `(admin_id)`.

### 3.4 `otp_codes`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| phone | varchar(16) | |
| purpose | OtpPurpose | |
| booking_id | uuid NULL FK | For action purposes |
| code_hash | char(64) | HMAC-SHA256 (D-05). **No `code` column** |
| attempts | smallint default 0 | |
| expires_at | timestamptz | |
| used_at / invalidated_at | timestamptz NULL | |
| ip_address | inet | |
| user_agent_hash | char(16) NULL | |
| created_at | | |

Index: `(phone, purpose, booking_id, created_at DESC)`, and a partial index `WHERE used_at IS NULL AND invalidated_at IS NULL`.

### 3.5 `barbers`
`id`, `full_name`, `nickname`, `avatar_object_key`, `bio`, `specialty`, `phone NULL`, `rating_avg numeric(3,2) default 0`, `rating_count int default 0`, `total_bookings int default 0`, `is_active`, `display_order`, `created_at`, `updated_at`, `deleted_at`.

### 3.6 `chairs`
`id`, `name varchar(50) UNIQUE`, `description`, `is_active`, `display_order`, `created_at`, `updated_at`.

### 3.7 `barber_chairs`
`id`, `barber_id FK`, `chair_id FK`, `valid_from timestamptz`, `valid_to timestamptz NULL`, `is_primary bool`, `created_at`.
- Constraint: one active primary chair per barber at any moment:
  `EXCLUDE USING gist (barber_id WITH =, tstzrange(valid_from, COALESCE(valid_to,'infinity'), '[)') WITH &&) WHERE (is_primary)`

### 3.8 `services`
`id`, `name`, `description`, `duration_minutes int` (CHECK `> 0 AND % 5 = 0`), `price numeric(12,2)` (CHECK `>= 0`), `deposit_amount numeric(12,2) NULL` (CHECK `>= 0`), `image_object_key`, `is_active`, `display_order`, `created_at`, `updated_at`, `deleted_at`.

### 3.9 `barber_services` (new)
`barber_id FK`, `service_id FK`, with a composite PK. The absence of any row for a barber means he offers all active services (Q-09).

### 3.10 `schedules`
`id`, `barber_id NULL FK` (NULL = salon default), `day_of_week smallint` (CHECK 0–6, where 0 = Sunday, and 5 = Friday), `start_time time NULL`, `end_time time NULL`, `is_closed bool default false`, `created_at`, `updated_at`.
- CHECK: `is_closed OR (start_time IS NOT NULL AND end_time IS NOT NULL AND start_time < end_time)`.
- More than one row per day is allowed (split windows). Overlapping windows for the same (barber, day) is prevented by the application, and verified by a test.

### 3.11 `blocked_slots`
`id`, `barber_id NULL`, `chair_id NULL`, `start_datetime`, `end_datetime`, `reason`, `block_type BlockType`, `created_by FK admins`, `created_at`.
- CHECK: `end_datetime > start_datetime`, and `NOT (barber_id IS NOT NULL AND chair_id IS NOT NULL)`.
- GiST index on `tstzrange(start_datetime, end_datetime, '[)')`.

### 3.12 `bookings`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| booking_number | varchar(16) **UNIQUE** | `HK-YYMMDD-XXXX` |
| original_booking_id | uuid NULL FK → bookings | Direct parent (reschedule) |
| root_booking_id | uuid NULL FK → bookings | Chain root, and NULL for the root itself |
| customer_id | uuid FK | |
| customer_phone_snapshot | varchar(16) | |
| customer_name_snapshot | varchar(100) | |
| barber_id / chair_id | uuid FK | |
| start_datetime / end_datetime | timestamptz | |
| total_duration | int | Minutes |
| total_price / deposit_amount | numeric(12,2) | |
| currency_code | char(3) | Snapshot at booking time |
| status | BookingStatus | |
| channel | BookingChannel | |
| customer_notes / admin_notes | text NULL | |
| rejection_reason | text NULL | |
| reject_kind | RejectKind NULL | |
| cancel_reason | text NULL | |
| reschedule_count | smallint default 0 | See booking-rules §8 |
| expires_at | timestamptz NULL | For `pending_payment` and `reschedule_pending` |
| tracking_token_hash | char(64) NULL **UNIQUE** | SHA-256 for lookup (D-08) |
| tracking_token_enc | bytea NULL | AES-256-GCM (nonce ‖ ciphertext ‖ tag) for building SMS links. Not returned in any API response |
| tracking_expires_at | timestamptz NULL | |
| tracking_revoked_at | timestamptz NULL | |
| last_admin_reminder_at | timestamptz NULL | |
| confirmed_at, completed_at, cancelled_at, expired_at | timestamptz NULL | |
| created_at, updated_at | | |

**Constraints** (in a migration SQL, because Prisma does not express them):
```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE bookings
  ADD CONSTRAINT bookings_time_order CHECK (end_datetime > start_datetime),
  ADD CONSTRAINT bookings_duration_match
      CHECK (total_duration = EXTRACT(EPOCH FROM (end_datetime - start_datetime)) / 60),
  ADD CONSTRAINT bookings_deposit_le_price CHECK (deposit_amount >= 0 AND deposit_amount <= total_price),
  ADD CONSTRAINT bookings_expires_required
      CHECK (status NOT IN ('pending_payment','reschedule_pending') OR expires_at IS NOT NULL),
  ADD CONSTRAINT bookings_reject_kind_required
      CHECK (status <> 'rejected' OR reject_kind IS NOT NULL OR original_booking_id IS NOT NULL);

-- D-19: the final guarantee against overlap
ALTER TABLE bookings ADD CONSTRAINT bookings_no_barber_overlap
  EXCLUDE USING gist (
    barber_id WITH =,
    tstzrange(start_datetime, end_datetime, '[)') WITH &&
  ) WHERE (status IN ('pending_payment','awaiting_approval','confirmed','reschedule_pending'));

ALTER TABLE bookings ADD CONSTRAINT bookings_no_chair_overlap
  EXCLUDE USING gist (
    chair_id WITH =,
    tstzrange(start_datetime, end_datetime, '[)') WITH &&
  ) WHERE (status IN ('pending_payment','awaiting_approval','confirmed','reschedule_pending'));

-- One pending reschedule request per parent
CREATE UNIQUE INDEX bookings_one_pending_reschedule
  ON bookings (original_booking_id) WHERE status = 'reschedule_pending';
```
The list of states in the `WHERE` predicate **must match** `ACTIVE_STATUSES` in the code. A test compares them from `pg_constraint`.

**Indexes:**
- `(barber_id, start_datetime)`, `(chair_id, start_datetime)`, `(customer_id, start_datetime DESC)`, and `(customer_phone_snapshot, start_datetime DESC)`.
- `(status, expires_at) WHERE status IN ('pending_payment','reschedule_pending')` for the Sweeper.
- `(status, start_datetime) WHERE status = 'awaiting_approval'`.

### 3.13 `booking_services`
`id`, `booking_id FK`, `service_id FK`, `service_name_snapshot`, `duration_snapshot`, `price_snapshot`, `deposit_snapshot NULL`, `sort_order`, `created_at`. UNIQUE on `(booking_id, service_id)`.

### 3.14 `booking_payments`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| booking_id | uuid **UNIQUE** FK | **Always the root** (booking-rules §3.2) |
| amount | numeric(12,2) | |
| method | PaymentMethod | |
| transaction_reference | varchar(100) NULL | |
| receipt_object_key | text NULL | |
| receipt_deleted_at | timestamptz NULL | |
| status | PaymentStatus | |
| reviewed_by / reviewed_at | NULL | |
| rejection_reason | text NULL | |
| refund_reference | varchar(100) NULL | CHECK: mandatory if the status is `refunded` |
| refunded_by / refunded_at | NULL | |
| created_at, updated_at | | |

### 3.15 `booking_status_history`
`id`, `booking_id FK`, `old_status BookingStatus NULL`, `new_status BookingStatus`, `changed_by_type ActorType`, `changed_by_admin_id NULL`, `notes`, `created_at`. Index `(booking_id, created_at)`. **Insert-only.**

### 3.16 `reviews`
`id`, `booking_id UNIQUE FK`, `customer_id FK`, `barber_id FK`, `rating smallint` (CHECK 1–5), `comment varchar(1000) NULL`, `display_name varchar(60)` (FR-REV-05), `is_visible default true`, `admin_reply`, `admin_reply_at`, `created_at`, `updated_at`.

```sql
CREATE OR REPLACE FUNCTION refresh_barber_rating() RETURNS trigger AS $$
DECLARE b uuid;
BEGIN
  FOR b IN SELECT DISTINCT x FROM unnest(ARRAY[
      CASE WHEN TG_OP <> 'INSERT' THEN OLD.barber_id END,
      CASE WHEN TG_OP <> 'DELETE' THEN NEW.barber_id END]) AS x WHERE x IS NOT NULL
  LOOP
    UPDATE barbers SET
      rating_avg   = COALESCE((SELECT ROUND(AVG(rating)::numeric, 2) FROM reviews
                               WHERE barber_id = b AND is_visible), 0),
      rating_count = (SELECT COUNT(*) FROM reviews WHERE barber_id = b AND is_visible)
    WHERE id = b;
  END LOOP;
  RETURN NULL;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_refresh_barber_rating
AFTER INSERT OR UPDATE OF rating, is_visible, barber_id OR DELETE ON reviews
FOR EACH ROW EXECUTE FUNCTION refresh_barber_rating();
```
(D-27: `COALESCE` to 0, and handling `barber_id` changes.)

### 3.17 `notifications` (admin inbox only, D-29)
`id`, `admin_id FK` (one row per admin), `type varchar(50)`, `title`, `body`, `reference_type`, `reference_id`, `read_at NULL`, `created_at`. Index `(admin_id, read_at, created_at DESC)`.

### 3.18 `sms_logs` (D-26)
`id`, `phone_hash char(16)`, `phone_masked varchar(20)`, `template_key varchar(40)`, `params jsonb` (**without `code` and without `link`**), `booking_id NULL`, `provider varchar(30)`, `provider_message_id NULL`, `status SmsStatus`, `attempts`, `segments smallint`, `error text NULL`, `created_at`, `sent_at`.

### 3.19 `settings`
`key varchar(64) PK`, `value jsonb`, `updated_by NULL FK admins`, `updated_at`. Value types and bounds are defined in code with a single Schema ([SRS.md §9](SRS.md)), and not in a `data_type` column.

### 3.20 `audit_logs` (D-23)
`id`, `actor_type ActorType`, `admin_id NULL FK`, `phone_hash char(16) NULL`, `action varchar(64)`, `entity_type varchar(40) NULL`, `entity_id uuid NULL`, `old_values jsonb NULL`, `new_values jsonb NULL`, `ip inet NULL`, `user_agent text NULL`, `request_id varchar(40)`, `created_at`.
- Indexes: `(created_at DESC)`, `(entity_type, entity_id)`, and `(admin_id, created_at DESC)`.
- **Insert-only.** The application's database user does not have `UPDATE` or `DELETE` privileges on it (enforced by `REVOKE` in a migration), and periodic deletion (two years) is handled by a job using a separate user.

## 4. Relationships (summary)

```
customers 1─N bookings N─1 barbers 1─N barber_chairs N─1 chairs
                 │  └─N─1 chairs
                 ├─1─N booking_services N─1 services N─M barbers (barber_services)
                 ├─1─1 booking_payments   (on the root only)
                 ├─1─N booking_status_history
                 ├─1─0..1 reviews
                 └─self: original_booking_id / root_booking_id
barbers 1─N schedules · barbers/chairs 1─N blocked_slots
admins 1─N admin_refresh_tokens · 1─N notifications · 1─N audit_logs
```

## 5. Migrations

**Rules:**
1. `prisma migrate dev --create-only`, then **review the SQL manually** before applying, then `prisma migrate dev`.
2. Everything Prisma does not express (Exclusion, CHECK, Triggers, partial and GiST indexes, REVOKE) is added **manually in the migration.sql file itself**, with a `-- manual:` comment.
3. **Forbidden:** `prisma db push` in any shared environment, editing a migration that has already been applied, and `migrate reset` on any non-local environment.
4. After every migration in CI: run `scripts/check-constraints.sql`, which fails if any of these are missing: `bookings_no_barber_overlap`, `bookings_no_chair_overlap`, `bookings_one_pending_reschedule`, `trg_refresh_barber_rating`, and the CHECK constraints. This protects against Prisma dropping a constraint it does not know about when generating a later migration.
5. Compatible changes (expand → deploy → contract): add a nullable column, then backfill the data, then make it NOT NULL in a later migration.
6. **Adding a value to `BookingStatus`**: `ALTER TYPE ... ADD VALUE` in a separate migration, then update the Exclusion predicate in a following migration (DROP + ADD), then update `ACTIVE_STATUSES` in the code. Any change to the states requires updating [booking-rules.md](booking-rules.md) first.
7. In production: `prisma migrate deploy` as a separate step before running the new version ([deployment.md](deployment.md)).

## 6. Seeds

| File | Environment | Content |
|---|---|---|
| `prisma/seed/base.ts` | All environments, and can be repeated with no effect (idempotent) | Default settings (upsert without overwriting modified values), and default salon working hours |
| `prisma/seed/dev.ts` | local/staging only | Admin `owner@halak.local` / `staff@halak.local` with a dev password, 5 barbers, 4 chairs, 8 services, 10 customers with test phone numbers, 20 bookings in various states, and 10 reviews |
| Production | — | The first Owner is created by a CLI command only: `admin:create --role owner` |

Test phone numbers come from an internally reserved range: `+963999000xxx`, and no real SMS is sent to them. The fake SMS provider rejects real numbers in dev.

## 7. Critical Queries (implementation reference)

```sql
-- Active bookings for a barber on a day (input to the Slots algorithm)
SELECT id, chair_id, start_datetime, end_datetime FROM bookings
WHERE barber_id = $1
  AND status IN ('pending_payment','awaiting_approval','confirmed','reschedule_pending')
  AND tstzrange(start_datetime, end_datetime, '[)') && tstzrange($2, $3, '[)');

-- Sweeper: payment expiry (T-03), batch of 100
WITH due AS (
  SELECT id FROM bookings
  WHERE status = 'pending_payment' AND expires_at <= now()
  ORDER BY expires_at LIMIT 100 FOR UPDATE SKIP LOCKED
)
UPDATE bookings b SET status = 'expired', expired_at = now()
FROM due WHERE b.id = due.id
RETURNING b.id;
-- Then in the same Transaction: INSERT booking_status_history for each id
```

All raw queries go through `prisma.$queryRaw` with tagged templates. **`$queryRawUnsafe` with any user input is forbidden.**