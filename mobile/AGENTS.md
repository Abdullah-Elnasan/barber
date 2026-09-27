# mobile/AGENTS.md — Flutter (Customer App)

The rules here are added to [`../AGENTS.md`](../AGENTS.md). The app is for the customer only, and its flow is **identical** to the customer website ([SRS §7](../docs/SRS.md)). The old design file is a reference for colors and fonts only, and **its Splash/Onboarding/Login/My Account/Notifications screens are canceled**.

## Stack
- Flutter stable (latest version), and Dart 3 with null-safety.
- `flutter_riverpod` (+ `riverpod_annotation`/`riverpod_generator`), `go_router`, `dio`, `freezed` + `json_serializable`, `flutter_secure_storage`, `intl` + `flutter_localizations`, `timezone`, `image_picker` + `flutter_image_compress`, `cached_network_image`, `url_launcher`, `share_plus`, `smart_auth` (SMS User Consent on Android), `shimmer`.
- **Forbidden:** Firebase in any form, OneSignal, and any Push, analytics, or ads SDK (D-21), and Hive (not needed).
- Static analysis: `very_good_analysis` with `prefer_relative_imports`. **Zero warnings.**
- Android `minSdk 26` (Android 8.0), and iOS 13 (Q-05).

## Structure (Feature-first)
```
lib/
├── main.dart                 # bootstrap: tz init, env, ProviderScope
├── app/                      # App widget, router, theme
├── core/
│   ├── api/                  # dio client, interceptors, ApiError, idempotency
│   ├── phone/                # normalizePhone (shared vectors)
│   ├── time/                 # Damascus tz, server offset, formatters
│   ├── money/                # formatMoney
│   ├── storage/              # SecureStore (tracking tokens only)
│   ├── l10n/                 # ARB (ar)
│   └── widgets/              # PrimaryButton, OtpField, PhoneField, Countdown, states…
└── features/
    ├── home/  barbers/  booking/  tracking/  lookup/  my_bookings/  info/
    │   └── data/ (api + dto) · domain/ (models) · presentation/ (screens, providers)
```

## Routes (`go_router`)
Routes match the web, so that deep links behave the same way:
```
/                           Home
/barbers/:id
/book/:barberId             Services
/book/:barberId/time
/book/details  /book/verify  /book/summary
/t/:token                   Tracking (and awaiting confirmation)
/t/:token/pay | cancel | reschedule | review
/track                      Phone lookup
/my-bookings                My bookings on this device
/more                       About the salon, terms, privacy, contact
```
Bottom navigation has 3 tabs: Home, My bookings on this device, and More. **No mandatory Onboarding.** The Splash is the native splash only, with no artificial wait.

## Deep links
- App Links (Android, with `autoVerify="true"`) and Universal Links (iOS, Associated Domains) for the domain `https://{APP_DOMAIN}/t/*`. The files `assetlinks.json` and `apple-app-site-association` are served by Nginx ([deployment.md §4](../docs/deployment.md#nginx)).
- **No custom scheme** (`salon://`): only https links, because that is what arrives in the SMS.
- When `/t/:token` is opened: fetch `GET /tracking`. If it succeeds, store `{ token, booking_number, start, status, saved_at }` in "My bookings on this device". If it returns 404, remove it from the list and show "The link is invalid or expired" with a "Search with your phone number" button.

## Tokens and storage
| Data | Where |
|---|---|
| Tracking tokens (for this device's bookings) | `flutter_secure_storage` (Keychain / Keystore), with a limit of 20 bookings, and automatic deletion of expired bookings |
| `booking_session`, `lookup_session`, and `action_token` | Memory only (Riverpod state). **Never written to disk** |
| Booking draft | Memory (lost when the app is killed, and this is acceptable) |
| Non-sensitive settings | `shared_preferences` |

- After creating a booking: save `tracking_token` in secure storage **before** navigating to the payment screen.
- "My bookings on this device" is a convenience feature only, not an account. The UI text makes that clear.

## Networking
- A single `dio` with `baseUrl` from `--dart-define=API_BASE_URL`. Timeouts: 10 seconds for connect, 20 seconds for receive, and 60 seconds for receipt upload.
- Interceptors: `Accept-Language: ar`, `X-Client: halak-mobile/<version>`, and error conversion to `ApiError { code, message, details, status }`.
- `X-Tracking-Token` and `Authorization` are passed **explicitly per call**, not through a global interceptor, so that a token does not leak into another endpoint.
- Automatic retry only for `GET` requests (twice with backoff). `POST` requests are retried manually via a button, with the **same** `Idempotency-Key`.
- **Forbidden:** `LogInterceptor` with request or response content or Headers in any build. In debug: method, path, and status only.
- Models: `freezed` with `@JsonSerializable(fieldRename: FieldRename.snake)`. Money is `String` (displayed via `formatMoney`), and times are `DateTime` in UTC from ISO.
- Parsing tests use fixtures from the examples in `../backend/openapi.json` to ensure the contract matches.

## Time
- `tz.initializeTimeZones()` at bootstrap, then `damascus = tz.getLocation('Asia/Damascus')`. Every time display goes through `formatDamascus(DateTime utc)`, regardless of the device's timezone.
- Timers come from `expires_at` with `serverOffset` (from the `Date` header). At zero: re-fetch the state. **Do not rely on `DateTime.now()` alone for any timeout.**

## UI
- `MaterialApp(locale: Locale('ar'), supportedLocales: [Locale('ar')], localizationsDelegates: [...])`, and all strings from ARB (`AppLocalizations`). No strings written directly in the widgets.
- **RTL:** `EdgeInsetsDirectional`, `AlignmentDirectional`, `PositionedDirectional`, `BorderRadiusDirectional`. No `EdgeInsets.only(left/right)` and no `Alignment.centerLeft` (checked by grep in CI). Directional icons flip automatically (`Icons.arrow_back` with `matchTextDirection`).
- Theme from the visual identity: Primary `#1E3A8A`, Secondary `#D4AF37`, Cairo font (embedded in assets), and Latin numerals.
- Actions are shown according to `allowed_actions` and `deadlines` from the server only, and business rules are **not recalculated locally**.
- Phone field: accepts Arabic-Indic digits, validates instantly with `core/phone` (vectors in [testing.md](../docs/testing.md#phone)), with `TextDirection.ltr` inside the field and `keyboardType: TextInputType.phone`.
- OTP field: 6 boxes in LTR order, with `autofillHints: [AutofillHints.oneTimeCode]` (iOS). On Android: SMS User Consent API via `smart_auth`, which does not require changing the message text, with a resend button after `resend_after_seconds`.
- Receipt: `image_picker` (camera or gallery), then compression to ≤ 2000px at quality 85, then preview, then upload with a progress bar. iOS permission texts (`NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription`) in Arabic.
- Mandatory states on every data screen: loading (Shimmer), empty, error with retry, and offline.
- `Semantics` for buttons and images, and font size follows system settings up to 1.3× without breaking the layout.

## Build and release
- Flavors via `--dart-define-from-file=env/<dev|staging|prod>.json`, containing only `API_BASE_URL` and `APP_DOMAIN`. **No secrets inside the app.**
- Android: `appbundle` for Play, and `apk --split-per-abi` for direct distribution (Q-05), with R8/ProGuard enabled. The signing key is **not in the repository**, but from CI secrets.
- Version: `version: X.Y.Z+build` in `pubspec.yaml`, and tag `mobile-vX.Y.Z` triggers `mobile-release.yml`.
- Direct distribution (if Play is unavailable): the app checks `GET /public/salon` → `min_app_version` and shows "Update required" with a download link.

## Tests
- Unit: `core/phone` (with the shared vectors from `../backend/test/fixtures/phone-vectors.json`), `core/time`, `core/money`, deep link parsing, and error conversion.
- Widget: details, OTP, payment, tracking (buttons according to `allowed_actions`), and my bookings on this device.
- Golden: the main booking screens in RTL.
- `integration_test`: a full booking against staging, and opening a `/t/` link as a deep link.
- Manual before every release: the list in [testing.md §6](../docs/testing.md).

## Commands
```
flutter pub get
dart run build_runner build -d
flutter analyze
flutter test
flutter test integration_test --dart-define-from-file=env/staging.json
flutter build appbundle --release --dart-define-from-file=env/prod.json
```

## Quick prohibitions
- Storing any session or action token on disk, or printing any API response in the log.
- Login, account, notifications, favorites, or discount screens.
- `DateTime.now()` as the sole source for a timeout, or displaying time in the device's timezone.
- Computing the final price or deposit or cancellation eligibility locally.
- Adding any external SDK without approval (the rule in `../AGENTS.md`).