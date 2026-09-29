# api.md — عقد الواجهة البرمجية (API Contract)

| البند | القيمة |
|---|---|
| الإصدار | **1.0** (الأساس) |
| التاريخ | 2026-09-29 |
| الحالة | **الأسس مكتملة، وأجسام الاستجابة تُكتب وحدةً بوحدة** |
| آخر تحديث | 2026-09-29 |

هذا الملف هو **مصدر الحقيقة الوحيد** لكل ما يلي: مسار الـ API، وأظرف الاستجابة، وقائمة رموز الخطأ، وأطوار المصادقة، وشكل الحقول التي يحسبها الخادم. تحتاجه قبل أي قراءة لـ `backend/AGENTS.md` §أخطاء ومدخلات ومخرجات، وقبل أي مراجعة من `@reviewer` أو `@security-auditor`.

**ترتيب الأولوية** (`AGENTS.md` §Read before you start): قرارات `D-xx` في [SRS §5](SRS.md) ← ملفات `docs/` المفصّلة ← باقي SRS ← الشيفرة الحالية.

## اصطلاحات القراءة في هذا الملف

| الوسم | المعنى |
|---|---|
| **موثّق** | مستند حرفياً في ملف آخر ضمن `docs/`، ويُذكر مرجعه |
| **مؤقّت** | غير موثّق؛ طُبِّقت عليه أأمن قيمة ممكنة وفق [القاعدة الذهبية 10](../AGENTS.md)، وسُجّل سؤالٌ في [SRS §11](SRS.md). **يجب تسويته قبل الإطلاق** |

> **قاعدة صارمة:** أي رمز خطأ أو حقل أو نقطة نهاية **غير** مذكور في هذا الملف يُعدّ **غير موجود**. لا يُخمَّن، ولا يُستنتَج من DTO أو من تحكّم NestJS، ولا يُعاد بناؤه من كود. الرمز الذي لا تجده هنا لا تضيفه إلا بإضافة سطر في §3 مع مرجعه.

---

## 1. المسار الأساسي وسياسة الإصدارات

```
المسار الأساسي:  /api/v1
```

- كل النقاط العامة تحت `/api/v1/…` ([`web/AGENTS.md`](../web/AGENTS.md) §API communication، و[`deployment.md`](deployment.md) §4).
- Nginx يوجّه `/api/` إلى الحاوية `api` على المنفذ 3000 ([`deployment.md`](deployment.md) §4).

| سياسة | القاعدة |
|---|---|
| الإصدار | جزء من المسار: `/api/v1` |
| التوافق | **بلا كسر**: لا يُحذف حقل ولا يُغيَّر نوعه ولا يتغيّر رمز خطأ في `v1` ([`AGENTS.md`](../AGENTS.md) §Stop and ask) |
| الشكل | كل JSON **بلا استثناء** باستخدام `snake_case` (D-25) |
| الترميز | `application/json` بترميز UTF-8 (**مؤقّت** — تفصيل غير موثّق) |

**خارج** `/api/v1` يوجد مسار واحد موثّق: `GET /__dev/sms`، وهو متاح فقط حين `NODE_ENV !== 'production'` ([`security.md`](security.md) §2.5).

---

## 2. أظرف الاستجابة

### 2.1 النجاح — **مؤقّت — [Q-18](SRS.md)**

**توجد تعارضات موثّقة لم تُحسم بعد، ولم يُتخذ قرار بشأنها هنا.**

| المصدر | الشكل المكتوب حرفياً |
|---|---|
| [`security.md` §2.4](security.md) | `{ "data": { "message", "resend_after_seconds", "masked_phone" } }` — **ملفوف** |
| [`booking-rules.md` §6](booking-rules.md) | `201: { booking, tracking_token, payment_instructions }` — **غير ملفوف** |
| [`architecture.md` §3.3](architecture.md) | `201 { booking, tracking_token, payment_instructions }` — **غير ملفوف** |
| [SRS §6.1](SRS.md) | `{ booking, tracking_token, payment_instructions }` — **غير ملفوف** |

**القيمة المطبَّقة حتى التسوية:** الغلاف `data` لكل الاستجابات الناجحة، لأسباب [`testing.md` §4.4](testing.md) الذي يختبر التطابق **بايت-ببايت** لجسم واحد، ولأن الغلاف يفسح مكاناً لبيانات `meta` في القوائم دون كسر:

```json
{ "data": { } }
```

- الغلاف يحتوي `data` فقط — بلا `meta` ولا `request_id` في الجسم، ليبقى [`security.md` §2.4](security.md) مطابقاً بايت-ببايت.
- عند التسوية، إمّا يُعمَّم الغلاف وتُحدَّث الملفات الثلاثة الأخرى في **نفس التغيير** مع تسجيله `D-xx`، وإمّا يُعتمد الشكل غير الملفوف. **لا يجوز كتابة `openapi.json` والأنواع المولّدة قبل التسوية.**

### 2.2 الخطأ

الخطأ **دائماً** HTTP status غير `2xx`، وجسمه:

```json
{
  "error": {
    "code": "SLOT_UNAVAILABLE",
    "message": "لم تعد الموعد متاحة، يرجى اختيار وقت آخر.",
    "details": {}
  }
}
```

| الحقل | النوع | الإلزام | الوصف |
|---|---|:-:|---|
| `code` | string | ✅ | رمز `UPPER_SNAKE` من §3 فقط. **الواجهة تبني منطقها على `code` حصراً** ([`web/AGENTS.md`](../web/AGENTS.md) §API communication) |
| `message` | string | ✅ | نص **عربي** للعرض كما هو. مصدره `common/errors/messages.ar.ts` ([القاعدة الذهبية](../AGENTS.md) 8) |
| `details` | object | — | تفاصيل اختيارية موثّقة لكل رمز في §3. **لا يحتوي أي سر** ([`security.md`](security.md) §8) |

**النوع على العميل** ([`web/AGENTS.md`](../web/AGENTS.md) §API communication):
`ApiError { code, message, details, status }`، حيث `status` هو **رمز حالة HTTP** المأخوذ من ترويسة الاستجابة، وهو ليس جزءاً من الجسم.

### 2.3 `request_id`

- يُعاد في الترويسة **`X-Request-Id`** فقط، ولا يدخل جسم الاستجابة (يبقى §2.1 مطابقاً بايت-ببايت لـ [`security.md`](security.md) §2.4).
- يُولّده Nginx ويمرّره إلى الـ API ([`deployment.md`](deployment.md) §4).
- يُستخدم في `audit_logs.request_id` و`audit_logs.ip` لكل عملية ([`database.md`](database.md) §3.20).

### 2.4 ترويسة `X-Request-Id` من جهة العميل

`X-Request-Id` ترويسة **صادرة من الخادم فقط** ولا تُقبل أي ترويسة تعريف من العميل (**مؤقّت** — قاعدة خادم غير موثّقة).

---

## 3. قائمة رموز الخطأ

**هذه القائمة هي المعجم الوحيد.** `DomainError` في الـ Backend لا يُرمز له إلا رمز من هنا ([`backend/AGENTS.md`](../backend/AGENTS.md) §Errors).

### 3.1 الرموز الموثّقة

| الرمز | HTTP | المعنى | مرجع |
|---|:-:|---|---|
| `PHONE_INVALID` | 400 | تنسيق الرقم غير صالح، أو بادئة مشغّل خارج `allowed_phone_prefixes` | [`security.md`](security.md) §1، §2.4 |
| `OTP_INVALID` | 400 | الرمز غير صحيح أو منتهي أو مستخدم أو غير موجود — **ردّ واحد لكل الحالات** | [`security.md`](security.md) §2.3 |
| `UNAUTHENTICATED` | 401 | لا رمز صالح، أو انتهت صلاحيته، أو فشل التحقق من `iss`/`aud`/`typ` | [`security.md`](security.md) §2.4، §3 |
| `TOKEN_USED` | 401 | رمز أحادي الاستخدام (`booking_session` أو `action_token`) استُهلك مسبقاً | [`security.md`](security.md) §3 |
| `CUSTOMER_BLOCKED` | 403 | الرقم محظور (`blocked_until > now()`) | D-17، [`booking-rules.md`](booking-rules.md) §6 |
| `NOT_FOUND` | 404 | غير موجود **أو** لا يملكه مقدم الطلب — **الخطآن غير قابلين للتمييز** | [`security.md`](security.md) §3، [`testing.md`](testing.md) §4.5 |
| `INVALID_STATE` | 409 | انتقال حالة غير مسموح به في [جدول الانتقالات](booking-rules.md#transitions) | [`booking-rules.md`](booking-rules.md) §2 |
| `SLOT_UNAVAILABLE` | 409 | لم تعد الموعد متاحة. يُسجَّل أيضاً عند خطأ القيد `23P01` | D-19، [`booking-rules.md`](booking-rules.md) §6 |
| `ACTIVE_BOOKING_LIMIT` | 409 | عدد الحجوزات النشطة للرقم بلغ `max_active_bookings_per_phone` | [`booking-rules.md`](booking-rules.md) §6 |
| `RESCHEDULE_NOT_ALLOWED` | 409 | طلب تغيير الموعد غير مسموح. `details.reason = "overlaps_current"` في حالة التداخل مع الموعد الأصلي | [`booking-rules.md`](booking-rules.md) §8.1 |
| `PAYMENT_WINDOW_CLOSED` | 409 | `now ≥ expires_at`. **الفحص على الوقت لا على الحالة** | [`booking-rules.md`](booking-rules.md) §12 بند 4 |
| `BLOCK_CONFLICTS` | 409 | الإغلاق المطلوب يتعارض مع حجوزات نشطة، ولم يُمرَّر `force: true` | [`booking-rules.md`](booking-rules.md) §10 |
| `RATE_LIMITED` | 429 | تجاوز حدّ المعدّل. `details.retry_after_seconds` (**مؤقّت** — [Q-13](SRS.md)) | [`security.md`](security.md) §5 |
| `INTERNAL` | 500 | خطأ داخلي. **لا يُكشف أي تفصيل في الرد** | [`backend/AGENTS.md`](../backend/AGENTS.md) §Errors |

**خارج جدول الرموز، موثّق صراحة:**
- تجاوز ميزانية SMS اليومية ([`security.md`](security.md) §5) **لا يولّد رمزاً جديداً**: تبقى `POST /auth/otp/send` عند `202` بالردّ الموحّد، ولا تُرسل الرسالة، ويُسجَّل `security.sms_budget_exceeded` في `audit_logs` ويُنبَّه المدير.
- خطأ المزامنة `P2002` من Prisma يُحوَّل `409` بحسب القيد ([`backend/AGENTS.md`](../backend/AGENTS.md) §Errors)؛ و`23P01` ← `SLOT_UNAVAILABLE`، و`P2025` ← `NOT_FOUND`. **كل ما عدا ذلك ← `500 INTERNAL`**.

### 3.2 الرموز المؤقّتة

طُبِّقت وفق [القاعدة الذهبية 10](../AGENTS.md) إلى أن تُسوّى أسئلتها في [SRS §11](SRS.md). **لا تُعتمد في واجهة عميل قبل التسوية.**

| الرمز | HTTP | المعنى | السؤال |
|---|:-:|---|---|
| `VALIDATION_ERROR` | 400 | فشل `ValidationPipe` على حقل واحد أو أكثر. `details.fields[] = { field, messages[] }` | [Q-14](SRS.md) |
| `IDEMPOTENCY_KEY_REQUIRED` | 400 | `Idempotency-Key` مفقود أو ليس UUID على `POST /bookings` | [Q-15](SRS.md) |
| `RECEIPT_INVALID` | 400 | الإيصال مرفوض. `details.reason ∈ { unsupported_type, too_large, unreadable }` | [Q-16](SRS.md) |
| `FORBIDDEN` | 403 | دور المستخدم لا يسمح بالمسار، أو فشل فحص `Origin`/`X-Requested-With` على `refresh`/`logout` | [Q-17](SRS.md) |

---

## 4. الترقيم

**مؤقّت — [Q-12](SRS.md).** يُستخدم في كل نقاط القوائم (`FR-BKM-01` وأمثالها):

```json
{ "data": [], "meta": { "page": 1, "page_size": 20, "total": 137 } }
```

| الحقل | القاعدة |
|---|---|
| `page` | ‎≥ 1 |
| `page_size` | الافتراضي 20، الحدّ الأقصى 100 |
| `total` | العدد الكلي قبل الترقيم |
| الفرز | **لا يوجد فرز افتراضي مخفي**: إمّا `sort` صريح من العميل، وإمّا ترتيب ثابت موثّق لوحدة بعينها |

مرشّحات القائمة تُحفظ في سلسلة الاستعلام لتكون قابلة للمشاركة ([`web/AGENTS.md`](../web/AGENTS.md) §Admin dashboard).

---

## 5. المصادقة والتفويض

### 5.1 الأظر

| الترويسة | القيمة | يُستخدم في |
|---|---|---|
| `Authorization` | `Bearer <access_token>` | رموز الإدارة، و`booking_session`، و`lookup_session`، و`action_token` |
| `X-Tracking-Token` | الرمز الخام (43 حرفاً، base64url) | كل نقاط التتبّع والدفع؛ **ليس** `Bearer` |
| `Idempotency-Key` | UUID | `POST /bookings` إلزامي ([FR-BK-12](SRS.md))، و`POST /bookings/:id/payment`، و`POST /bookings/:id/reschedule` |
| `X-Requested-With` | `halak` | **إلزامي** على `refresh` و`logout` فقط ([`security.md`](security.md) §4) |
| `X-Client` | `halak-mobile/<version>` | من تطبيق Flutter ([`mobile/AGENTS.md`](../mobile/AGENTS.md) §Networking) |
| `Accept-Language` | `ar` | اختياري |

**ممنوع منعاً باتاً** ([`security.md`](security.md) §8):
- `X-Tracking-Token` أو `Authorization` أو `__Host-hk_rt` في **سجل** أو **استجابة** أو **تفاصيل** خطأ.
- تمرير `X-Tracking-Token` عبر `Authorization` أو العكس. الحارس يتوقع نوعاً واحداً في مكان واحد.

### 5.2 أنواع الرموز

كل رمز JWT يحمل: `typ`، `iss = halak`، `aud ∈ { admin, customer }`، `exp`. **الرمز من نوع لا يُقبل في موضع يتوقع نوعاً آخر، أبداً** ([`security.md`](security.md) §3).

| الرمز | `typ` | السر | `aud` | العمر | الحفظ على الخادم |
|---|---|---|---|---|---|
| Admin access | `admin_access` | `ADMIN_JWT_SECRET` | `admin` | 15 دقيقة | — |
| Admin refresh | (عشوائي 32 بايت) | — | — | 7 أيام | `SHA-256` في `admin_refresh_tokens` مع `family_id` |
| `booking_session` | `booking_session` | `CUSTOMER_TOKEN_SECRET` | `customer` | `booking_session_minutes` (20) | `jti` في Redis بعد الاستهلاك |
| `lookup_session` | `lookup_session` | `CUSTOMER_TOKEN_SECRET` | `customer` | `lookup_session_minutes` (15) | — |
| `action_token` | `action` | `CUSTOMER_TOKEN_SECRET` | `customer` | `action_token_minutes` (10) | `jti` في Redis بعد الاستهلاك |
| Tracking | (عشوائي 32 بايت) | — | — | `end + tracking_token_days_after_end` | `SHA-256` + `AES-256-GCM` في `bookings` (D-08) |
| SSE ticket | (عشوائي 32 بايت) | — | — | 60 ثانية | Redis `sse:{hash}`، أحادي الاستخدام |

- سرّان منفصلان، كل واحد ‎≥ 32 بايت عشوائية: `ADMIN_JWT_SECRET` و`CUSTOMER_TOKEN_SECRET` ([`security.md`](security.md) §3، §10).
- الاستهلاك الأحادي: `SET used:{jti} 1 NX EX <ttl>`، وفشل `NX` ← `401 TOKEN_USED`.
- كوكي التحديث: `__Host-hk_rt; HttpOnly; Secure; SameSite=Strict; Path=/`، ومقبول **فقط** على `/api/v1/admin/auth/refresh` و`/logout` ([`security.md`](security.md) §4).

### 5.3 فحص الملكية

يُطبَّق على **كل** نقطة عميل، وداخل الخدمة لا في المتحمّم فقط ([`backend/AGENTS.md`](../backend/AGENTS.md) §Authentication and authorization):

| الحارس | الشرط |
|---|---|
| `action_token` | `token.bid == :id` **و** `token.act == الفعل` **و** `token.sub == booking.customer_phone_snapshot` |
| `X-Tracking-Token` | `sha256(token) == booking.tracking_token_hash` **و** `now < tracking_expires_at` **و** `tracking_revoked_at IS NULL` |
| `lookup_session` | كل حجز مُرجَع يحقق `customer_phone_snapshot == token.sub` |

مورد غير موجود ومورد لا يملكه مقدم الطلب يعيدان **نفس** `404 NOT_FOUND` ([`security.md`](security.md) §3).

**حجب الرقم** يُفرض **على `POST /bookings` فقط** (`403 CUSTOMER_BLOCKED`)، ولا يُفرض على إرسال OTP ولا على التتبّع (D-17).

---

## 6. `Idempotency-Key`

**إلزامي** على `POST /bookings` ([FR-BK-12](SRS.md))، ومستعمل كذلك على الرفع وتغيير الموعد ([`web/AGENTS.md`](../web/AGENTS.md) §API communication).

| القاعدة | التفصيل |
|---|---|
| الشكل | UUID |
| النطاق | زوج `(رقم الهاتف، المفتاح)` — الرقم يؤخذ من الرمز **لا من الجسم** ([`booking-rules.md`](booking-rules.md) §6) |
| نافذة التشغيل | 24 ساعة |
| عند التطابق | **تُعاد الاستجابة المحفوظة كما هي**، بلا تنفيذ جديد ([`booking-rules.md`](booking-rules.md) §6 خطوة 0) |
| عند التعارض | المفتاح نفسه مع محتوى مختلف ← يُرفض، بالرمز **المؤقّت** [Q-15](SRS.md) |
| التخزين | Redis، انتهاء تلقائي بـ TTL ([`architecture.md`](architecture.md) §1، [`booking-rules.md`](booking-rules.md) §11) |

- يُولَّد **مرة واحدة** عند الضغط على الزر، ويُحفظ، **ويُعاد استخدامه** عند إعادة المحاولة اليدوية. زر الإرسال معطّل أثناء الطلب ([`web/AGENTS.md`](../web/AGENTS.md) §API communication؛ والمثل في [`mobile/AGENTS.md`](../mobile/AGENTS.md) §Networking).
- المعرّف **لا** يُرسل في جسم الطلب ولا في سلسلة الاستعلام، بل في الترويسة حصراً.
- **العميل لا يعيد التوليد عند الفشل.** إعادة التوليد تكسر الضمانة.

---

## 7. الحقول المحسوبة من الخادم

هذه الحقول **يثبّتها الخادم**، والواجهة **تعرض فقط** ولا تعيد حساب أي منها ([القاعدة الذهبية](../AGENTS.md) 2). تُرجَع ضمن جسم التتبّع `GET /tracking`.

### 7.1 `allowed_actions` — **شكل مؤقّت — [Q-11](SRS.md)**

**شكل الكائن (كائن منطقي بأربعة حقول) غير موثّق في أي ملف؛ الأسماء الأربعة وحدها هي الموثّقة.** مُطبَّق مبدئياً، ويجب تسوية [Q-11](SRS.md) قبل توليد الأنواع.

كائن بأربعة أزرار منطقية، تُعرض كما هي، **بلا أي شرط إضافي على العميل** ([`web/AGENTS.md`](../web/AGENTS.md) §Rendering rules، [`mobile/AGENTS.md`](../mobile/AGENTS.md) §UI).

```json
"allowed_actions": { "pay": false, "cancel": true, "reschedule": true, "review": false }
```

| الفعل | الشرط الذي يحسبه الخادم | المصدر |
|---|:-:|---|
| `pay` | **حالتان:** (أ) `status = pending_payment` **و** `now < expires_at`؛ (ب) `status = awaiting_approval` **و** `payment.status = pending` | (أ) T-02، [`booking-rules.md`](booking-rules.md) §12 بند 4 — (ب) [FR-PAY-06](SRS.md)، [`booking-rules.md` §3.2](booking-rules.md) |
| `cancel` | `status ∈ { pending_payment, awaiting_approval }`، أو `status = confirmed` **و** `start − now ≥ cancellation_hours_before` | T-04، T-08، T-13 |
| `reschedule` | `status = confirmed` **و** لا يوجد ابن `reschedule_pending` **و** `reschedule_count < max_reschedule_count` **و** `start − now ≥ reschedule_hours_before` | T-16، [`booking-rules.md`](booking-rules.md) §8.1 |
| `review` | `status = completed` **و** لا يوجد تقييم لهذا الحجز **و** `now < tracking_expires_at` | [FR-REV-01](SRS.md)، [FR-REV-02](SRS.md) |

**حالة (ب) في `pay` ضرورية:** [FR-PAY-06](SRS.md) تُلزم بإعادة رفع الإيصال ما دام الحجز في `awaiting_approval` وقبل مراجعته، و[`booking-rules.md` §3.2](booking-rules.md) ينص على تحديث نفس الدفعة `pending` وحذف الملف القديم. لولا (ب) لأخفى عميل يتبع [القاعدة الذهبية 2](../AGENTS.md) زر الرفع في تلك الحالة، وأصبحت FR-PAY-06 غير قابلة للتنفيذ. **انظر [Q-19](SRS.md)**.

**ليس فعلاً:** لا يوجد `cancel` على ابن `reschedule_pending`؛ فإلغاؤه في هذه الحالة أثرٌ للجهة الموقّعة على الأب (T-19)، وابن تغيير الموعد لا يحمل رمز تتبّع قبل الموافقة.

> **أسماء الأفعال ومجموعتها غير محسومة** ([Q-11](SRS.md)): الأزرار الأربعة أعلاه هي **كل ما وثّقته** المستندات. إن وُجد فعل إضافي (مثل «إعادة إرسال الرابط») فلا يجوز إضافته دون تحديث هذا القسم أولاً.

### 7.2 `deadlines` — **أسماء الحقول مؤقّتة — [Q-11](SRS.md)**

**اسما حقلين فقط موثّقان في شجرة المستندات:** `expires_at` و`tracking_expires_at` ([`database.md`](database.md)). **الأسماء الخمسة الأخرى أدناه جديدة في هذا الملف**؛ مصادر *قيمها* موثّقة، لكن **أسماءها ليست**، ويجب تسوية [Q-11](SRS.md) قبل توليد الأنواع.

كل المواعيد المتعلقة بالإجراء، مخزَّنة بـ `timestamptz` ومرساة إلى `Asia/Damascus`، وتُرسَل بصيغة ISO-8601 UTC. الحقل يساوي `null` حين لا ينطبق على الحالة الحالية — **والعميل لا يستنتج شيئاً من `null`**.

| الحقل | القيمة | المصدر |
|---|---|---|
| `payment_expires_at` **(اسم جديد)** | `expires_at` — في `pending_payment` فقط | القيمة من T-02، T-03 |
| `approval_cutoff_at` **(اسم جديد)** | `start_datetime − approval_cutoff_minutes` — في `awaiting_approval` | القيمة من T-10، D-14 |
| `reschedule_request_expires_at` **(اسم جديد)** | `expires_at` — في `reschedule_pending` فقط | القيمة من T-20، [`booking-rules.md`](booking-rules.md) §8.1 |
| `cancellation_allowed_until` **(اسم جديد)** | `start_datetime − cancellation_hours_before` — في `confirmed` | القيمة من T-13، [`booking-rules.md`](booking-rules.md) §4 |
| `reschedule_allowed_until` **(اسم جديد)** | `start_datetime − reschedule_hours_before` — في `confirmed` | القيمة من T-16، [`booking-rules.md`](booking-rules.md) §4 |
| `review_allowed_until` **(اسم جديد)** | `tracking_expires_at` — في `completed` | القيمة من [FR-REV-02](SRS.md) |
| `tracking_expires_at` | `end_datetime + tracking_token_days_after_end` | D-09، [`database.md`](database.md)، [`booking-rules.md`](booking-rules.md) §4 |

- **المؤقّتات تُحسب من `expires_at` مع تصحيح انزياح الساعة** من ترويسة `Date` ([`web/AGENTS.md`](../web/AGENTS.md) §Rendering rules؛ والمثل في [`mobile/AGENTS.md`](../mobile/AGENTS.md) §Time). عند بلوغ الصفر **يُعاد جلب الحالة من الخادم** ولا يُفترض النتيجة.
- **لا يظهر `no_show`** هنا: قاعدة يطبّقها المدير في [`booking-rules.md`](booking-rules.md) §9.

### 7.3 `payment_instructions` — **أسماء الحقول مؤقّتة — [Q-20](SRS.md)**

**المفهوم موثّق ([FR-PAY-01](SRS.md): رقم شام كاش، واسم الحساب، والمبلغ، ومؤقّت التنبيه) وأسماء أعمدة قاعدة البيانات والإعدادات موثّقة، لكن أسماء حقول الـ API أدناه جديدة في هذا الملف.** العمود الأخير يوثّق **مصدر القيمة**، لا أصل الاسم.

تُرجَع مع `201` من `POST /bookings` ([`booking-rules.md`](booking-rules.md) §6) وتُستخدم في شاشة الدفع ([FR-PAY-01](SRS.md)).

```json
"payment_instructions": {
  "sham_cash_number": "09XXXXXXXX",
  "account_name": "اسم صاحب الحساب",
  "amount": "0.00",
  "currency_code": "SYP",
  "expires_at": "2026-09-29T09:45:00.000Z"
}
```

> **`amount` في المثال `0.00` وليس مبلغاً حقيقياً:** قيم Q-03 للعملة لم تُحسم بعد، وأسعار v1.1 «تبدو بالعملة القديمة». القيمة تُقرأ من `bookings.deposit_amount`، وهو **نص وليس عدداً** (D-28، [القاعدة الذهبية](../AGENTS.md) 7).

| الحقل | **مصدر القيمة** |
|---|---|
| `sham_cash_number` | الإعداد `sham_cash_number` ([SRS §9](SRS.md)) |
| `account_name` | الإعداد `sham_cash_account_name` |
| `amount` | `bookings.deposit_amount` — **نص وليس عدداً** (D-28، [القاعدة الذهبية](../AGENTS.md) 7) |
| `currency_code` | `bookings.currency_code` — لقطة وقت الحجز |
| `expires_at` | `bookings.expires_at` |

القيم المحسوبة (`duration`، `total_price`، `deposit`) تأتي من **الخادم فقط** ([FR-BK-02](SRS.md))، ومجموع شاشة الخدمات **إرشادي فقط** ([`web/AGENTS.md`](../web/AGENTS.md) §Rendering rules).

> **مصادر شاشة الدفع بعد إعادة تحميل الصفحة غير محسومة** ([Q-21](SRS.md)): شاشة الدفع هي `/t/:token/pay` ([SRS §7.1](SRS.md)) وتُفتح تنقّلاً بعد الإنشاء ([`web/AGENTS.md`](../web/AGENTS.md) §Booking flow)، ومع إعادة التحميل لا مصدر موثّقاً لبيانات شام كاش في `GET /tracking`. الواجب: أم `GET /tracking` يُعيد `payment_instructions`، أم نقطة مستقلة.

---

## 8. فهرس النقاط

**هذه كل النقاط التي كُتب مسارها حرفياً في ملفات `docs/`، وليست كامل واجهة الـ API.** أي نقطة أخرى تُضاف إلى هذا الجدول **في نفس التغيير** الذي يضيفها ([`backend/AGENTS.md`](../backend/AGENTS.md) §When adding an endpoint).

> **نطاق الفهرس غير مكتمل** ([Q-22](SRS.md)): الشاشات الإدارية المطلوبة في [SRS §7.2](SRS.md) (قائمة الحجوزات وتفاصيلها، التقويم، الحلاقون، الكراسي، الخدمات، أوقات العمل، الإغلاقات، الزبائن، التقييمات، التقارير، الإعدادات، الطاقم، سجل التدقيق) **لا نقاط لها في هذا الجدول بعد**، لأن المستندات لم تكتب مساراتها. كذلك نقاط الكتابة الإدارية التي تُرجع `409 INVALID_STATE` (T-05…T-09، T-11، T-12، T-14، T-17، T-18، T-21، T-22). **لا يجوز بناء واجهة SRS §7 من هذا الفهرس وحده.**

### 8.1 العميل

| الطريقة | المسار | المصادقة | الحالات الموثّقة | الوحدة |
|---|---|---|---|---|
| POST | `/auth/otp/send` | عام؛ `X-Tracking-Token` أو `lookup_session` لأغراض الأفعال | `202`، `400 PHONE_INVALID`، `429 RATE_LIMITED`، `401 UNAUTHENTICATED` (أغراض الأفعال)، `404 NOT_FOUND` (أغراض الأفعال) | `otp` |
| POST | `/auth/otp/verify` | عام | `400 OTP_INVALID`، `429 RATE_LIMITED` | `sessions` |
| GET | `/availability/slots` | عام | — | `scheduling` |
| GET | `/barbers` | عام | `200` | `catalog` |
| GET | `/public/salon` | عام | — | `settings` |
| POST | `/bookings` | `Bearer booking_session` + `Idempotency-Key` | `201`، `403 CUSTOMER_BLOCKED`، `409 ACTIVE_BOOKING_LIMIT`، `409 SLOT_UNAVAILABLE`، `429 RATE_LIMITED` | `bookings` |
| POST | `/bookings/:id/payment` | `X-Tracking-Token` | `409 PAYMENT_WINDOW_CLOSED`، `409 INVALID_STATE`، `429 RATE_LIMITED`، `404 NOT_FOUND` | `payments` |
| GET | `/tracking` | `X-Tracking-Token` | `200`، `404 NOT_FOUND` | `tracking` |
| GET | `/tracking/lookup/bookings` | `Bearer lookup_session` | — | `tracking` |
| POST | `/tracking/lookup/bookings/:id/resend-link` | `Bearer lookup_session` | — | `tracking` |
| POST | `/bookings/:id/cancel` | `Bearer action_token` (`act=cancel`) | `409 INVALID_STATE`، `404 NOT_FOUND`، `401 TOKEN_USED` | `bookings` |
| POST | `/bookings/:id/reschedule` | `Bearer action_token` (`act=reschedule`) | `409 RESCHEDULE_NOT_ALLOWED`، `409 SLOT_UNAVAILABLE`، `409 INVALID_STATE` | `bookings` |
| POST | `/bookings/:id/review` | `Bearer action_token` (`act=review`) | `409 INVALID_STATE` | `reviews` |

**الاستجابة الموحّدة الموثّقة لـ `POST /auth/otp/send`** ([`security.md` §2.4](security.md))، والمختبَرة على التطابق **بايت-ببايت** ([`testing.md` §4.4](testing.md)):

```json
{
  "data": {
    "message": "تم إرسال رمز التحقق.",
    "resend_after_seconds": 60,
    "masked_phone": "09XX XXX XXX"
  }
}
```

`masked_phone` يظهر **لأغراض الأفعال فقط**، لأن طالب الرمز يحمل أصلاً إثبات وصوله للحجز ([`security.md`](security.md) §2.4). `resend_after_seconds` هو الاسم الموحّد لمهلة إعادة الإرسال، وهو الاسم الذي يقرأه العميل ([`web/AGENTS.md`](../web/AGENTS.md) §API communication؛ والمثل في [`mobile/AGENTS.md`](../mobile/AGENTS.md) §Networking).

### 8.2 الإدارة

> **قاعدة الدور:** كل متحمّم تحت `/admin` يحمل `@UseGuards(AdminAuthGuard, RolesGuard)`، **والافتراضي `owner` وحده**. حقّ الطاقم يُمنح **صراحةً** بـ `@Roles('owner','staff')` ([`backend/AGENTS.md`](../backend/AGENTS.md)، [`security.md`](security.md) §4). الحماية الحقيقية على الخادم، لا على قائمة الواجهة.
>
> **تعارض غير محسوم** ([Q-23](SRS.md)): [SRS §2](SRS.md) يمنح الطاقم صلاحية **«إغلاق الفترات الزمنية (المواعيد المحظورة)»**، لكن [`security.md`](security.md) §4 و[`backend/AGENTS.md`](../backend/AGENTS.md) يفرضان `owner` وحده افتراضياً تحت `/admin`. **القيمة المطبَّقة حتى التسوية: `owner` وحده**، لأن `security.md` ملف ضمن `docs/` وهو أعلى في ترتيب الأولوية من §2 في SRS. صفّا `blocked-slots` أدناه يعكسان هذه القيمة المؤقّتة.

| الطريقة | المسار | المصادقة | الأدوار | الوحدة |
|---|---|---|---|---|
| POST | `/admin/auth/login` | عام | — | `admin-auth` |
| POST | `/admin/auth/refresh` | كوكي `__Host-hk_rt` + `X-Requested-With: halak` + فحص `Origin` | — | `admin-auth` |
| POST | `/admin/auth/logout` | كوكي `__Host-hk_rt` + `X-Requested-With: halak` + فحص `Origin` | — | `admin-auth` |
| GET | `/admin/settings` | `Bearer admin_access` | `owner` ([SRS §2](SRS.md)، [`backend/AGENTS.md`](../backend/AGENTS.md) §Settings) | `settings` |
| GET | `/admin/dashboard` | `Bearer admin_access` | `owner`, `staff` ([SRS §2](SRS.md) §Reports) | `reports` |
| POST | `/admin/events/ticket` | `Bearer admin_access` | `owner`, `staff` | `notifications` |
| GET | `/admin/events` | `?ticket=` (أحادي الاستخدام، 60 ثانية) | `owner`, `staff` | `notifications` |
| GET | `/admin/bookings/:id/payment/receipt` | `Bearer admin_access` | `owner`, `staff` ([SRS §2](SRS.md) §deposit refunded) | `payments` |
| POST | `/admin/blocked-slots/preview` | `Bearer admin_access` | `owner` — **مؤقّت**، [Q-23](SRS.md) | `scheduling` |
| POST | `/admin/blocked-slots` | `Bearer admin_access` | `owner` — **مؤقّت**، [Q-23](SRS.md) | `scheduling` |

### 8.3 التشغيل والتطوير

| الطريقة | المسار | الشرط | الوحدة |
|---|---|---|---|
| GET | `/health/ready` | — | infra |
| GET | `/__dev/sms` | `NODE_ENV !== 'production'` | `dev` |

> `POST /admin/blocked-slots` (إنشاء إغلاق) **مُدرج هنا بـ [Q-22](SRS.md)** لأنّه المسار الوحيد القادر على إرجاع `409 BLOCK_CONFLICTS` الموثّق في [`booking-rules.md`](booking-rules.md) §10. لم يوثّقه أي ملف كسلسلة نصية، فاسمه ومساره **مؤقّتان**.

`/__dev/sms` يعيد `404` في الإنتاج، وهو بند في قائمة ما قبل الإطلاق ([`security.md`](security.md) §13).

---

## 9. إضافة نقطة نهاية: القائمة الإلزامية

مطابقة لـ [`backend/AGENTS.md`](../backend/AGENTS.md) §When adding an endpoint:

1. DTO طلب واستجابة مع Swagger؛ **ولا يُعاد كائن Prisma من المتحمّم أبداً** (يمنع تسرّب `tracking_token_hash` و`password_hash`).
2. الحارس الصحيح + `@Roles` للإدارة + فحص الملكية للعميل.
3. منطق القاعدة في **policy خالصة** لها اختبار وحدة.
4. سجل `audit_logs` لكل كتابة إدارية، ولكل حدث أمني في [`security.md`](security.md) §12.
5. حدّ معدّل إن كانت النقطة عامة أو تُرسل رسالة.
6. اختبار e2e: النجاح، و`401`/`403`/`404`، وأخطاء `409` الأساسية، و`400 VALIDATION_ERROR` عند التحقق.
7. **`npm run openapi:export`**، وتحديث §8 في هذا الملف، **و** `npm run gen:api` في `web`.
8. §3 إن لزم رمز خطأ جديد.

---

## 10. الأسئلة المفتوحة التي كشفها هذا الملف

كل ما لم يمكن توثيقه بثقة وُثّق كسؤال في [SRS §11](SRS.md) بدل اختراعه ([القاعدة الذهبية](../AGENTS.md) 10). **لا تُعتمد أي قيمة مؤقّتة في واجهة عميل قبل تسوية سؤالها**، ولا يجوز توليد `openapi.json` والأنواع المولّدة قبل تسوية [Q-18](SRS.md).

| السؤال | الموضوع | القيمة المؤقّتة المطبَّقة |
|---|---|---|
| [Q-10](SRS.md) | لغة التوثيق في `docs/` | العربية، وفق [القاعدة الذهبية](../AGENTS.md) §Language and style، ريثما تُحسم بقية الملفات على القاعدة نفسها |
| [Q-11](SRS.md) | أسماء حقول وشكل `allowed_actions` و`deadlines` | الأسماء المذكورة في §7.1 و§7.2 كما هي، مع التصريح أن خمسة منها جديدة |
| [Q-12](SRS.md) | شكل الترقيم | `{ page, page_size, total }` بجوار `data`، افتراضي 20 وحدّ 100 |
| [Q-13](SRS.md) | اسم حقل مهلة إعادة المحاولة | `details.retry_after_seconds`، موحّداً مع `resend_after_seconds` في §8.1 |
| [Q-14](SRS.md) | رمز فشل التحقق من الحقول | `400 VALIDATION_ERROR` مع `details.fields[]` |
| [Q-15](SRS.md) | مفتاح `Idempotency-Key` المفقود أو المتعارض | `400 IDEMPOTENCY_KEY_REQUIRED`، والتعارض يُرفض بنفس الرمز |
| [Q-16](SRS.md) | رمز رفض الإيصال | `400 RECEIPT_INVALID` مع `details.reason` |
| [Q-17](SRS.md) | رمز رفض الدور وفشل CSRF | `403 FORBIDDEN` |
| [Q-18](SRS.md) | غلاف `data` أم جسم غير ملفوف | غلاف `data` لكل الاستجابات الناجحة، **وتسويته إجبارية قبل أي توليد** لـ `openapi.json` والأنواع المولّدة |
| [Q-19](SRS.md) | إعادة رفع الإيصال في `awaiting_approval` | `pay = true` في تلك الحالة ما دام `payment.status = pending` |
| [Q-20](SRS.md) | أسماء حقول `payment_instructions` | الأسماء الخمسة في §7.3 |
| [Q-21](SRS.md) | مصدر بيانات الدفع بعد إعادة التحميل | غير محسوم؛ لم تُطبَّق قيمة |
| [Q-22](SRS.md) | مسارات النقاط المتبقية | `POST /admin/blocked-slots` مُدرج كمؤقّت لبلوغ `BLOCK_CONFLICTS`، وباقي واجهة SRS §7.2 لم يُصمَّم بعد |
| [Q-23](SRS.md) | دور الطاقم في إغلاق المواعيد | `owner` وحده، مع نفي صريح مؤقّت لحقّ الطاقم الموثّق في [SRS §2](SRS.md) |
