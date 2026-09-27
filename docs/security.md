# Security: OTP, Tokens, Authentication, and Secrets

The customer's only identity is their phone number, and a booking reserves real time against real money. The most important risks, in order:

1. **SMS abuse**: draining the balance, or flooding someone else's number with messages.
2. **Fake booking that blocks the schedule**: booking all Slots without paying.
3. **Hijacking someone else's booking**: cancelling it or changing its appointment.
4. **Leaking secrets in logs**: OTP, tracking links, and tokens.
5. **Compromising an admin account.**

Every rule below serves one or more of these risks. The `D-xx` decisions are in [SRS.md §5](SRS.md).

---

## 1. Phone Number Normalization (D-03, D-04)

```
normalizePhone(input):
  s = input
      → convert Arabic-Indic digits (٠-٩) and Persian digits (۰-۹) to 0-9
      → strip spaces, hyphens, dots, parentheses, and directional marks (U+200E/U+200F/U+061C)
  match s:
    ^00963(9\d{8})$  → "+963" + g1
    ^\+963(9\d{8})$  → "+963" + g1
    ^963(9\d{8})$    → "+963" + g1
    ^0(9\d{8})$      → "+963" + g1
    otherwise        → null   (400 PHONE_INVALID)
  prefix = digits[4..5]   // "93" in +96393...
  if prefix ∉ settings.allowed_phone_prefixes → null  (400 PHONE_INVALID)
  return "+9639XXXXXXXX"
```
- The format `9XXXXXXXX` (9 digits without a leading zero) is **rejected**, and was wrongly accepted in v1.1.
- The same function is implemented in `backend` (the source), `web`, and `mobile`, with shared test vectors in [testing.md](testing.md#phone).
- Number display: `09XX XXX XXX`. The masked number in public UIs: `09•• ••• •78` (last two digits only).

## 2. OTP (D-05)

### 2.1 Purposes
| Purpose | Input | Additional proof required to send | Produces |
|---|---|---|---|
| `booking` | `phone` | — | `booking_session` |
| `lookup` | `phone` | — | `lookup_session` |
| `cancel` / `reschedule` / `review` | `booking_id` | `X-Tracking-Token` for this booking, **or** Bearer `lookup_session` for the same phone | `action_token` |

For action purposes, the code is sent to the booking's `customer_phone_snapshot`, and no number from the customer is accepted.

### 2.2 Generation and storage
```
code      = crypto.randomInt(0, 1_000_000).toString().padStart(6, '0')
code_hash = HMAC_SHA256(OTP_PEPPER, `${phone}|${purpose}|${booking_id ?? ''}|${code}`)
```
- A row in `otp_codes` contains: `phone`, `purpose`, `booking_id`, `code_hash`, `expires_at`, `attempts = 0`, `ip_address`, `user_agent_hash`.
- A new send for the same triple (`phone`, `purpose`, `booking_id`) invalidates all previous unused rows (`invalidated_at = now()`).
- **The code is never stored in plaintext anywhere**: not in the database, not in Redis, not in logs, not in `sms_logs`. It exists only in memory until it is handed to the provider.

### 2.3 Verification
```
BEGIN
  row = SELECT ... FROM otp_codes
        WHERE phone=$p AND purpose=$u AND booking_id IS NOT DISTINCT FROM $b
          AND used_at IS NULL AND invalidated_at IS NULL AND expires_at > now()
        ORDER BY created_at DESC LIMIT 1
        FOR UPDATE
  if !row → COMMIT; return fail
  match = timingSafeEqual(hmac(code), row.code_hash)
  if match:
      UPDATE otp_codes SET used_at = now() WHERE id = row.id
  else:
      UPDATE otp_codes SET attempts = attempts + 1,
             invalidated_at = CASE WHEN attempts + 1 >= $max THEN now() END
      WHERE id = row.id
COMMIT                      ← executed in both cases, and no rollback on failure
if !match → return fail
issue token(purpose)
```
- **The counter increment must be persisted before returning the error.** If an exception is thrown inside the Transaction and a rollback occurs, the counter is lost and the attempt limit is not enforced. Write the function so it returns a result (`{ ok: false }`), and throw `DomainError` after the Transaction ends.
- **All failures return the same response**: `400 OTP_INVALID` ("The code is incorrect or expired"), whether the code is wrong, expired, used, or nonexistent.
- Every failure increments the IP block counter (§6).

### 2.4 Unified response to the send request
`POST /auth/otp/send` always returns `202` in the same shape, whether or not a message was sent, i.e. even if the number is blocked (D-17) or the provider failed:
```json
{ "data": { "message": "إذا كان الرقم صحيحاً ستصلك رسالة خلال لحظات", "resend_after_seconds": 60, "masked_phone": "09•• ••• •78" } }
```
- `masked_phone` is returned only for action purposes, because the requester already holds proof of access to the booking.
- The only exceptions: `400 PHONE_INVALID` (a format error, and it reveals nothing about whether the number exists), and `429 RATE_LIMITED`. And for action purposes only: `401 UNAUTHENTICATED` when proof of access is missing, and `404 NOT_FOUND` if the booking does not belong to the requester.

### 2.5 Development mode
- `SMS_PROVIDER=fake` writes messages to the service's memory, and exposes them via a development-only endpoint: `GET /__dev/sms` (available only if `NODE_ENV !== 'production'`).
- `OTP_FIXED_CODE` is allowed only if `NODE_ENV ∈ {development, test}`. **Boot fails** if it is present in production.

## 3. Tokens (D-06, D-07, D-08)

| Token | Format | Lifetime | Server-side storage | Transport | On the client | Revocation |
|---|---|---|---|---|---|---|
| Admin access | JWT HS256, `typ=admin_access`, `sub=admin_id`, `role`, `sv` | 15 minutes | — | `Authorization: Bearer` | Memory only | Change `admins.session_version` |
| Admin refresh | 32 random bytes base64url | 7 days | SHA-256 in `admin_refresh_tokens` with `family_id` | Cookie `__Host-hk_rt` | Cookie (HttpOnly) | Rotation on every use, and reusing a rotated token = revoking the whole family |
| `booking_session` | JWT HS256, `typ=booking_session`, `sub=phone`, `jti` | 20 minutes | `jti` in Redis after consumption | Bearer | Memory only (not localStorage) | Consumed on a successful `POST /bookings` |
| `lookup_session` | JWT, `typ=lookup_session`, `sub=phone` | 15 minutes | — | Bearer | Memory only | Expiry |
| `action_token` | JWT, `typ=action`, `sub=phone`, `bid`, `act`, `jti` | 10 minutes | `jti` in Redis after consumption | Bearer | Memory only | One-time use |
| Tracking | 32 random bytes base64url (43 chars) | Until `end + 7 days` | SHA-256 in `bookings.tracking_token_hash` (UNIQUE) for lookup, and AES-256-GCM in `tracking_token_enc` for building SMS links | `X-Tracking-Token` for the API, and the path `/t/{token}` for pages | Mobile: secure storage. Web: the URL only | Rotate or revoke (D-08) |
| SSE ticket | 32 random bytes | 60 seconds | Redis `sse:{hash}` ← `admin_id` | Query `?ticket=` | — | One-time use |

**Shared rules:**
- Two separate secrets: `ADMIN_JWT_SECRET` and `CUSTOMER_TOKEN_SECRET`, each ≥ 32 random bytes. Every Guard verifies `typ`, `iss=halak`, and `aud` (`admin` or `customer`). A token of one type is never accepted in a place expecting another type.
- Compare hashes with `timingSafeEqual`, and look up by hash via a UNIQUE index.
- One-time consumption: `SET used:{jti} 1 NX EX <ttl>`, and an NX failure means `401 TOKEN_USED`. If Redis is lost, a token could theoretically be reused within its short lifetime. This is acceptable, because every action is in turn protected by the conditional state update.
- **Ownership check** on every customer endpoint:
  - `action_token`: `token.bid == :id` and `token.act == the action`, and `token.sub == booking.customer_phone_snapshot`.
  - `X-Tracking-Token`: `sha256(token) == booking.tracking_token_hash`, and `now < tracking_expires_at`, and `tracking_revoked_at IS NULL`.
  - `lookup_session`: every returned booking satisfies `customer_phone_snapshot == token.sub`.
- A booking that does not exist and a booking the requester does not own return **the same error**: `404 NOT_FOUND`.

## 4. Admin Authentication

- Passwords: bcrypt with cost 12, length ≥ 10 characters, and rejection of common passwords (top 10,000 list).
- Login: 5 failed attempts per (email + IP) within 15 minutes lead to a 15-minute lock. The message is unified: "The email or password is incorrect".
- The refresh Cookie: `__Host-hk_rt; HttpOnly; Secure; SameSite=Strict; Path=/`, and it is accepted only at `/api/v1/admin/auth/refresh` and `/logout`.
- **CSRF**: the Access token is in a Header from memory, so there is no CSRF risk on the rest of the API. As for `refresh` and `logout` (via Cookie), they require `SameSite=Strict`, an `Origin` check against `WEB_ORIGIN`, and the Header `X-Requested-With: halak`.
- Changing the password, disabling the account, or changing the role: `session_version += 1` and revoking all refresh tokens.
- No outgoing email (D-22). Resetting the Owner's password is done via `node dist/cli.js admin:reset-password --email ...` on the server, with it recorded in `audit_logs`.
- Roles: `owner` and `staff` per the matrix in [SRS.md §2](SRS.md). Enforced with `@Roles()` + `RolesGuard` at the Controller level. **The default for any route under `/admin` is `owner` only** unless explicitly stated otherwise.

<a id="rate-limits"></a>
## 5. Rate Limits

Implemented with Redis (sliding window or fixed window with TTL keys). Keys carry the phone hash and not the number itself: `rl:otp:phone:{sha256(phone)}`.

| Scope | Limit (default) | On exceeding |
|---|---|---|
| `POST /auth/otp/send` per phone (all purposes) | `otp_rate_limit_per_phone_hour` = 5 per hour | 429 |
| `POST /auth/otp/send` per phone | One resend every `otp_resend_cooldown_seconds` = 60 seconds | 429 with `retry_after` |
| `POST /auth/otp/send` per IP | `otp_rate_limit_per_ip_hour` = 20 per hour (D-32) | 429 |
| `POST /auth/otp/send` per booking (action purposes) | 5 per hour | 429 |
| `POST /auth/otp/verify` per phone | 10 per hour | 429 |
| `POST /bookings` per phone | 10 per hour, with an active-bookings limit | 429 or 409 |
| `POST /bookings/:id/payment` per booking | 5 per hour | 429 |
| `/tracking*` per IP | 60 per minute | 429 |
| `POST /admin/auth/login` | See §4 | 429 |
| Global per IP | 100 per minute (Nginx `limit_req` + Throttler) | 429 |
| **System daily SMS budget** | `SMS_DAILY_BUDGET` (environment variable) | Stop OTP messages with an admin alert, and continue messages for existing bookings |

- The client IP is read from `X-Forwarded-For` **only** from the trusted Nginx (`trust proxy` = one address). It is not accepted from the client directly.
- The daily budget protects against balance drain even if an attacker bypasses all the above limits by distributing requests.

## 6. Temporary IP Block
- `otp_ip_block_failures` (20) verification failures from a single IP within an hour lead to a block of `otp_ip_block_minutes` (60 minutes) on `/auth/otp/*` only.
- Every block is logged as a security event in `audit_logs` (`actor_type=system`, `action=security.ip_blocked`).
- The admin can lift an IP block from the settings (by deleting the key from Redis).

## 7. File Uploads (Payment Receipts and Barber Images)

1. Size ≤ 8MB (enforced in Nginx `client_max_body_size 10m` and in Multer).
2. The type is determined from **magic bytes** (`file-type`), not from the extension or the Content-Type. Allowed: JPEG, PNG, WebP, HEIC.
3. Mandatory re-encoding with `sharp` to WebP at quality 82 and a max dimension of 2000px, with all metadata (EXIF and GPS) stripped. The original file is not stored.
4. The storage key is random: `receipts/{yyyy}/{mm}/{uuid}.webp`, and the original filename is never used.
5. MinIO: a private Bucket (with no public policy), with at-rest encryption via `MINIO_KMS_SECRET_KEY` (SSE-S3). Barber images go in a separate `public-media` bucket for public read only.
6. The admin displays the receipt via `GET /admin/bookings/:id/payment/receipt`, which streams the file after verifying authorization. **No Presigned URLs for receipts.**

<a id="logging"></a>
## 8. Logging and Redaction (NFR-SEC-17, D-26)

**Forbidden from appearing in any log** (the application, Nginx, Sentry/GlitchTip, `sms_logs`, `audit_logs`, or Queue payloads):
- OTP (the code), any Tracking Token, any JWT or refresh token, passwords, the `Authorization` header, the `X-Tracking-Token` header, the `__Host-hk_rt` Cookie, and `ticket`.
- Full phone numbers in technical logs. Use `phone_hash` (the first 16 characters of HMAC-SHA256 with the `OTP_PEPPER` key) or the masked number.

**Implementation:**
- Backend: `nestjs-pino` with `redact` for these paths: `req.headers.authorization`, `req.headers["x-tracking-token"]`, `req.headers.cookie`, `*.code`, `*.password`, `*.token`, `*.tracking_token`, `*.refresh_token`, `*.phone`, `*.ticket`.
- The page routes `/t/{token}` do not pass through the API, but Nginx rewrites the path in the log (see [deployment.md](deployment.md#nginx)).
- Sentry: `beforeSend` applies the same redaction, and removes `request.cookies`, sensitive `request.headers`, and all query strings.
- The Queue: the SMS message payload carries `booking_id`, `template_key`, and non-secret params only. The Worker builds the link by decrypting `tracking_token_enc` at send time. **One exception**: the `otp` message carries the code in the payload inside Redis for a short time (`removeOnComplete: true`, and `removeOnFail: true`), and it is never logged.
- **An automated test** captures all log output during a full booking flow, and fails if any code or token used in the test appears in it ([testing.md](testing.md)).

## 9. HTTP Headers and CORS

- `Strict-Transport-Security: max-age=31536000; includeSubDomains`, and TLS 1.2 minimum with 1.3 preferred.
- `X-Content-Type-Options: nosniff`, `frame-ancestors 'none'` (via CSP), `Referrer-Policy: strict-origin-when-cross-origin`, and **`no-referrer` on `/t/*` and `/track`**.
- CSP for the web: `default-src 'self'; img-src 'self' data: blob: https://{APP_DOMAIN}; connect-src 'self'; script-src 'self'` (with a nonce for Nuxt scripts if needed), and `style-src 'self' 'unsafe-inline'` (required for Ant Design Vue).
- `X-Robots-Tag: noindex, nofollow` on `/t/*`, `/track`, `/admin/*`, and on all APIs.
- CORS: the web and the API are on the same origin (`/api` behind Nginx), so no CORS is needed for the web. A Flutter app is not subject to CORS. **No `Access-Control-Allow-Origin: *`.**

## 10. Secrets

| Variable | Use | Generation |
|---|---|---|
| `ADMIN_JWT_SECRET` | Signing admin tokens | `openssl rand -base64 48` |
| `CUSTOMER_TOKEN_SECRET` | Signing `booking_session`, `lookup_session`, and `action_token` | Same |
| `OTP_PEPPER` | HMAC for OTP and `phone_hash` | Same. Changing it invalidates current codes and changes new `phone_hash` values |
| `TRACKING_TOKEN_KEY` | AES-256-GCM for `tracking_token_enc` (32 bytes, base64) | `openssl rand -base64 32`. Rotating it requires re-encrypting the field (a CLI command), and losing it only means being unable to build links in future messages |
| `DATABASE_URL`, `REDIS_URL` | Connection | Passwords ≥ 32 characters |
| `MINIO_ROOT_USER/PASSWORD`, `MINIO_KMS_SECRET_KEY` | Storage and encryption | `MINIO_KMS_SECRET_KEY=halak-key:$(openssl rand -base64 32)` |
| `SMS_PRIMARY_*`, `SMS_FALLBACK_*` | Providers | From the provider's dashboard |
| `SENTRY_DSN` | Error tracking | — |

- No secrets in Git ever. The `.env.example` file contains variable names with dummy values only. `gitleaks` runs in CI.
- In production: the `/opt/halak/.env` file with `600` permissions and `root` ownership, with an encrypted backup off the server.
- Boot-time validation (`zod` schema): any missing secret or one shorter than 32 bytes means boot failure.
- Secret rotation: `ADMIN_JWT_SECRET` is rotated with support for two keys during the transition (`kid`), or by logging everyone out. This is acceptable for a small number of admins.

## 11. Privacy and Data Retention

| Data | Retention |
|---|---|
| `otp_codes` | 30 days |
| `sms_logs` | 90 days |
| Receipts (files) | `receipt_retention_days` after the booking ends (Q-07). The payment row remains, and `receipt_deleted_at` is set |
| `audit_logs` | Two years |
| Bookings and customers | As long as the system exists. When a customer requests deletion: anonymize the name and phone in `customers` and the snapshots, while keeping the financial figures |

## 12. Logged Security Events (D-23)

`otp.sent`, `otp.verify_failed`, `otp.locked`, `security.ip_blocked`, `security.sms_budget_exceeded`, `admin.login`, `admin.login_failed`, `admin.refresh_reuse_detected`, `admin.password_changed`, `tracking.resent`, `tracking.rotated`, `tracking.revoked`, `customer.blocked`, `customer.unblocked`, in addition to every admin write.

## 13. Checklist Before Every Release

- [ ] `OTP_FIXED_CODE` is absent, `SMS_PROVIDER` is not `fake`, and `/__dev/*` returns 404
- [ ] The "no secrets in logs" test passes
- [ ] `gitleaks`, `npm audit --omit=dev --audit-level=high`, and `flutter pub outdated` show no critical vulnerabilities
- [ ] Exclusion constraints exist in the production database (`scripts/check-constraints.sql`)
- [ ] Swagger UI is disabled or protected in production
- [ ] The daily SMS budget is set
- [ ] The last backup was successfully restored within the last 30 days