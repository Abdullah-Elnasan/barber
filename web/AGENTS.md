# web/AGENTS.md — Nuxt 3 (Customer Site + Admin Dashboard)

The rules here are added to [`../AGENTS.md`](../AGENTS.md). A single Nuxt application (D-02): the customer on `/`, and the admin on `/admin`. The approved screens are in [SRS §7](../docs/SRS.md). The old design file `halakdesign.md` is a reference for colors and fonts only, and **its flows are canceled**.

## Stack
- Nuxt 3, TypeScript strict, Pinia (with no persistence for tokens), VeeValidate + Zod, `openapi-fetch` + `openapi-typescript`, `@nuxtjs/i18n` (Arabic only), `dayjs` (utc + timezone).
- Customer: Tailwind CSS with logical properties for direction. Admin: Ant Design Vue 4 (`ConfigProvider` with `direction="rtl"` and Arabic locale) with Chart.js.
- Fonts: Cairo (self-hosted via `@nuxt/fonts`, not from Google at runtime).

## Structure
```
web/
├── pages/
│   ├── index.vue · barbers/ · about.vue · terms.vue · privacy.vue     ← SSR
│   ├── book/[barberId]/index.vue · book/[barberId]/time.vue
│   ├── book/details.vue · book/verify.vue · book/summary.vue           ← ssr:false
│   ├── t/[token]/index.vue · pay.vue · cancel.vue · reschedule.vue · review.vue   ← ssr:false
│   ├── track.vue                                                       ← ssr:false
│   └── admin/**                                                        ← ssr:false
├── layouts/  default.vue · booking.vue · admin.vue · admin-auth.vue
├── components/  customer/ · admin/ · shared/ (OtpInput, PhoneInput, Money, DateTime…)
├── composables/ useApi.ts · useAdminAuth.ts · useAdminEvents.ts · useCountdown.ts
├── stores/      bookingDraft.ts · customerSession.ts · adminAuth.ts · adminNotifications.ts
├── utils/       phone.ts · money.ts · time.ts · errors.ts
├── types/api.d.ts    ← generated, do not edit by hand
├── locales/ar.json
└── tests/  unit/ · e2e/ (Playwright)
```

## Separation between customer and admin
- **Importing `ant-design-vue` is forbidden** outside `components/admin/**`, `pages/admin/**`, and `layouts/admin*.vue` (rule `no-restricted-imports`). The customer bundle must not contain antd. Verify with `npx nuxi analyze` whenever in doubt.
- Tailwind preflight does not apply to admin (the admin layout wraps its content in a class that resets the rules, or preflight is enabled only inside `.customer-root`).
- Shared components in `components/shared/` do not depend on either library.

## API communication
- All calls go through `useApi()` (a wrapper around `openapi-fetch`, with base `/api/v1`). **No direct `$fetch` or `fetch` to the API.**
- Types: `npm run gen:api` generates `types/api.d.ts` from `../backend/openapi.json`. Do not write response types by hand.
- Errors are converted to `ApiError { code, message, details, status }`. Display `message` as-is, and build logic on `code` only:
  - `SLOT_UNAVAILABLE`: return to time selection and refresh the slots.
  - `UNAUTHENTICATED`/`TOKEN_USED` in the customer flow: restart the OTP step.
  - `RATE_LIMITED`: show the countdown from `retry_after_seconds`.
- `Idempotency-Key`: generated once when "Confirm booking", "Upload receipt", or "Confirm new appointment" (reschedule) is pressed, stored in the store, and reused on retry. The submit button is disabled during the request.

## Tokens and state
| Data | Where it is stored |
|---|---|
| `booking_session`, `lookup_session`, and `action_token` | Pinia in memory only. **No localStorage, no sessionStorage, no Cookies** |
| Tracking token | From the URL path (`/t/[token]`) into memory, then the `X-Tracking-Token` header. Not stored in any storage |
| Booking draft (barber, services, slot, name, phone, notes) | `sessionStorage` so it survives a page refresh, **with no token** |
| Admin access token | Memory (`adminAuth` store). Refresh via an HttpOnly Cookie managed by the server |

- After a successful `POST /bookings`: `navigateTo('/t/' + tracking_token + '/pay', { replace: true })`, then clear the booking draft and the session.
- On 401 in admin: **one** `refresh` attempt (with a lock to prevent parallel refresh requests), then retry the request, otherwise redirect to `/admin/login`.

## Rendering rules
- `<html lang="ar" dir="rtl">`. Use `ms-*`/`me-*`/`ps-*`/`pe-*`/`start-*`/`end-*`, and **not `ml-`/`mr-`/`left-`/`right-`** (lint rule). Directional icons (arrows) are flipped with `rtl:-scale-x-100`.
- Every string comes from `locales/ar.json` via `$t()`. No Arabic strings written directly in templates.
- Numbers: Latin numerals (`ar-SY-u-nu-latn`). Money: `formatMoney("5000.00", "SYP")` → `5,000 ل.س` (without decimals).
- Time: `formatDateTime(iso)` in `Asia/Damascus` time always, whether the device is in another timezone or not. The day name is computed from the date and never written by hand.
- **Timers** (payment, OTP resend): computed from `expires_at` with clock-drift correction (`serverOffset` from the `Date` header of the last response). At zero: re-fetch the state from the server, and do not assume the outcome.
- **Available actions** (pay, cancel, reschedule, and review buttons) are shown according to `allowed_actions` and `deadlines` from the server only. **Do not recalculate booking-rules in the UI.**
- Totals on the services screen are indicative. The numbers on the summary and payment screens come from the server's response.
- **`v-html` is forbidden** with any user or API content (reviews, notes, names).

## Components with mandatory behavior
- `PhoneInput`: accepts Arabic-Indic digits and converts them, validates instantly with `utils/phone.ts` (the same vectors as [testing.md](../docs/testing.md#phone)), displays `09XX XXX XXX`, and submits the normalized value, with `inputmode="tel"`, `autocomplete="tel"`, and `dir="ltr"` inside the field.
- `OtpInput`: 6 boxes, with `autocomplete="one-time-code"` and `inputmode="numeric"`, paste support, auto-submit on completion, resend after `resend_after_seconds`, and box order from left to right (`dir="ltr"`).
- `ReceiptUploader`: camera or gallery, preview, and in-browser compression to ≤ 2000px before upload, with a progress bar and retry.

## Admin dashboard
- The navigation menu is built according to role, but **the real protection is on the server**. The `admin` middleware redirects when there is no session, and Owner-only pages verify the role.
- `useAdminEvents()`: requests a ticket via `POST /admin/events/ticket`, then opens an `EventSource`, and reconnects with a new ticket using backoff. If SSE fails 3 times in a row, it switches to polling `/admin/dashboard` every 30 seconds.
- Audio: browsers block autoplay, so an "Enable sound alert" button is visible until the user presses it once (unlocking `AudioContext`). On a `booking.awaiting_approval` or `booking.reschedule_requested` event: sound + system notification (Notification API if permitted) + badge update.
- Tables: pagination and filtering from the server. Filters are kept in the query string so they are shareable.
- Receipt display: `<img>` from the admin endpoint via `blob` with the Authorization header, not a direct link.
- Dangerous actions (reject, cancel, no-show, force-close) require a confirmation Modal that shows the effect on the deposit.

## Performance and accessibility
- Customer pages: LCP < 2.5s on fast 3G (Moto G Power profile), and initial customer JS < 150KB gzip.
- Images: `<NuxtImg>` with defined sizes and `loading="lazy"` outside the first screen.
- AA contrast, every field has a `<label>`, focus is visible, and keyboard navigation works through the entire booking flow.

## Tests
- Vitest: `utils/*` (phone with the shared vectors from `../backend/test/fixtures/phone-vectors.json`, money, time, and timer), the stores, and the mandatory components above.
- Playwright: the scenarios in [testing.md §5](../docs/testing.md), against a real Backend with `SMS_PROVIDER=fake`, and the OTP from `/__dev/sms`.
- Snapshots: RTL at 320px, 768px, and 1440px widths for the booking and tracking pages.

## Scripts
```
dev · build · preview · lint · typecheck · test · test:e2e · gen:api · analyze
```

## Quick prohibitions
- Storing any token in `localStorage`, `sessionStorage`, or a Cookie set from JS.
- SSR for `/t/**`, `/book/**`, or `/admin/**` pages.
- `console.log` for any API response (it may contain `tracking_token`).
- Using the device clock as the source of truth for timeout expiry.
- Adding customer login, an account-based "My bookings", a favorites button, or a promotions banner.