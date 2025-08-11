**PROMPT ДЛЯ CODEX**

Ты — сеньор full-stack разработчик. Создай продакшн-готовый проект **Donation Hub** — одностраничный сайт пожертвований, визуально минималистичный (в духе gwarp.org/donate), но с уникальным текстом/контентом. Важно: **оплата только 2D (без 3-D Secure)**, валюта **USD**.

### Технологии

* **Next.js 14 (App Router) + TypeScript**
* **Tailwind CSS** + **shadcn/ui**, иконки — **lucide-react**
* Формы/валидация: **react-hook-form + zod**
* БД: **PostgreSQL + Prisma**
* E2E: **Playwright**, unit: **vitest**
* Деплой: **Vercel** (web) + **Neon/Render** (Postgres)
* Почта: **SendGrid** (квитанции)
* Уведомления: **Telegram Bot** (опционально)

### Платежи (Только 2D, USD)

* Провайдер по умолчанию: **Stripe (US)**. Альтернативы (адаптеры): **Authorize.Net**, **Braintree**, **Square** — но сохранить единый интерфейс.
* Глобальный флаг: `REQUIRE_2D_ONLY=true`. Любое требование 3DS/`requires_action` → **авто-отклонение** платежа с понятным сообщением пользователю.
* **Stripe Implementation (server)**:

  * One-time: `PaymentIntent` (`currency: "usd"`, `amount` в центах, `payment_method_types: ["card"]`, `payment_method_options.card.request_three_d_secure = "automatic"`).
  * Если статус `requires_action` → отменить Intent, вернуть ошибку «3-D Secure is not supported (2D only)».
  * Recurring: `Subscription` с ранее сохранённым `payment_method` (через `SetupIntent`). Если в off-session потребуется 3DS — отклонить.
* Два режима UI:

  * **Stripe Elements (hosted fields style)** — на нашем домене, но **не** хранить PAN/CVV (SAQ-A-EP).
  * Возможность переключиться на **HPP** для альтернативных провайдеров.
* **AVS/CVC строгие правила**: при сумме > 200 USD отклонять `AVS: N|U` или `CVC: N|U`.

### Функционал страницы пожертвований

* Пресеты сумм: 5, 10, 25, 50, 100 (USD) + поле «Другая сумма» (мин 1, макс 100 000).
* Табы: **One-time / Monthly** (разовый/ежемесячный).
* Поля: email (обяз.), имя (опц.), чекбокс «Показывать публично» (имя/инициалы в ленте).
* Переключатель тёмной/светлой темы.
* Прогресс-бар к цели (таргет сумма из ENV/БД).
* Счётчики: «Всего собрано» и «Количество доноров» (агрегация по БД, live-обновление).
* Лента последних публичных донатов (имя/инициалы, сумма, время).
* Блок «How funds are used» + FAQ (аккордеоны).
* i18n: **en-US** по умолчанию; добавить **ru** (переключатель локали).
* На кнопке оплаты бейдж: **“2D only — no 3-D Secure”**.

### Админ-панель (/admin)

* Доступ: email magic-link (passwordless) или allowlist по домену из ENV.
* Разделы: донаты, доноры, график по месяцам (recharts), фильтры/поиск, экспорт CSV.
* Кнопка «Refund» (через API провайдера) для разовых платежей.
* Показать AVS/CVC результаты, страну карты/IP.

### API и вебхуки

* `POST /api/checkout` — создать платёж (one-time/subscription) в USD, вернуть `client_secret` (Elements) или `paymentUrl` (HPP).
* `POST /api/webhooks/stripe` — обрабатывать:

  * `payment_intent.succeeded` → `Donation: SUCCEEDED`
  * `payment_intent.payment_failed` → `Donation: FAILED` (логировать причину; если `requires_action` — отметить `declined_3ds_required`)
  * `charge.refunded` → `Donation: REFUNDED`
* `POST /api/refund/:id` — инициировать возврат (только админ).
* Rate limit на `/api/checkout` и вебхуки (например, 5 req/min/IP).

### Модель БД (Prisma)

```prisma
model Donor {
  id         String   @id @default(cuid())
  email      String   @unique
  name       String?
  locale     String?  // "en-US" | "ru-RU"
  createdAt  DateTime @default(now())
  donations  Donation[]
}

model Donation {
  id             String   @id @default(cuid())
  donor          Donor    @relation(fields: [donorId], references: [id])
  donorId        String
  amountMinor    Int      // cents
  currency       String   // "USD"
  type           DonationType
  status         DonationStatus
  provider       String   // "stripe" | "authorizenet" | ...
  externalId     String?  // charge/paymentIntent/subscription id
  isPublic       Boolean  @default(false)
  cardCountry    String?
  ipCountry      String?
  avsResult      String?
  cvcResult      String?
  createdAt      DateTime @default(now())
  @@index([createdAt])
  @@index([status])
}

enum DonationType { ONE_TIME RECURRING }
enum DonationStatus { SUCCEEDED REFUNDED FAILED }
```

### Архитектура платежей (адаптер)

* Интерфейс `PaymentProvider`:

  * `createOneTime(params)`
  * `createSubscription(params)`
  * `handleWebhook(req)`
  * `refund(externalId, amountMinor?)`
* Реализации: `StripeProvider` (реальная), заглушки: `AuthorizeNetProvider`, `BraintreeProvider`, `SquareProvider`.
* Фабрика по ENV: `PAYMENT_PROVIDER=stripe`.

### Страницы/маршруты

* `/` — Donate (главная)
* `/success?payment_id=` — успешная оплата (подтянуть данные)
* `/cancel` — отмена
* `/privacy`, `/terms`, `/faq`
* `/admin` — защищённая админка

### Файловая структура (пример)

```
/app
  /(public)
  /[locale]/page.tsx
  /[locale]/success/page.tsx
  /[locale]/cancel/page.tsx
  /[locale]/admin/page.tsx
  /api/checkout/route.ts
  /api/webhooks/stripe/route.ts
/components
  DonateForm.tsx
  AmountPresets.tsx
  RecurrenceTabs.tsx
  CurrencyBadge.tsx
  ProgressBar.tsx
  PublicFeed.tsx
  AdminTable.tsx
/lib
  payments/
    index.ts
    stripe.ts
    types.ts
  db.ts
  i18n.ts
  validations.ts
/prisma
  schema.prisma
/scripts
  seed.ts
.env.example
README.md
```

### ENV (.env.example)

```
DATABASE_URL=postgres://user:pass@host:5432/db
PAYMENT_PROVIDER=stripe
REQUIRE_2D_ONLY=true
DEFAULT_CURRENCY=USD
SUPPORTED_CURRENCIES=USD
STRIPE_SECRET_KEY=sk_test_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
SENDGRID_API_KEY=...
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
DONATION_TARGET_MINOR=5000000
DONATION_TARGET_CURRENCY=USD
ADMIN_ALLOWED_EMAILS=user@yourdomain.com
```

### Безопасность/комплаенс

* **PCI DSS:** при Elements — **SAQ-A-EP**; при HPP — **SAQ-A**.
* CSP: запретить inline-скрипты, разрешить только домены Stripe/HPP.
* Не логировать PAN/CVV; только токены/идентификаторы.
* Обработка отказа по 3DS: показать баннер «We accept 2D payments only».

### Тесты

* Unit (моки Stripe): `success_2d`, `requires_action` → ожидаем авто-cancel, `refund_ok`.
* E2E (Playwright): разовый платеж в тестовом режиме, подписка (2-й off-session чардж проходит без 3DS).
* Lighthouse на главной: ≥95 по всем метрикам.

### Acceptance Criteria

* Разовые и ежемесячные платежи в **USD** проходят **без 3-D Secure** (если эмитент не требует).
* Любой `requires_action`/3DS → **авто-отклонение** с корректным UI-сообщением и логом причины.
* Счётчики и лента донатов обновляются без перезагрузки (SSE/рефетч).
* Админка: фильтры, экспорт CSV, refund, AVS/CVC поля видны.
* i18n работает (en/ru), тема светлая/тёмная.

### Что нужно сгенерировать

1. Полный код проекта с конфигами (Next, Tailwind, shadcn/ui, Prisma, payments adapter).
2. Миграции Prisma и `seed.ts`.
3. Готовые API-маршруты (`/api/checkout`, `/api/webhooks/stripe`, `/api/refund/:id`).
4. README с шагами: локальный запуск, настройка Stripe (PaymentIntent/Subscription/Webhook), деплой на Vercel, переменные окружения.
5. Минималистичный UI в духе gwarp.org/donate (но без копирования контента).

Сгенерируй код и README сразу.

---

Хочешь — после генерации я проверю платежный флоу и допишу проверку статуса `requires_action`, чтобы точно рубить 3DS на уровне сервера и UI.

