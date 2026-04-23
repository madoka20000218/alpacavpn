# Alpaca VPN — Технологический стек

> Telegram-бот для продажи VPN-подписок (VLESS) и прокси.
> Целевая нагрузка: **до 5 000 пользователей** (суммарно, не concurrent), ≤ 50 RPS пиков.
> Документ фиксирует **выбранный стек, принятые решения и инфраструктурные контракты**.
> Детали реализации — в `ARCHITECTURE.md`, план работ — в `IMPLEMENTATION_PLAN.md`, UX — в `INTERFACE.md`.
>
> **Статус документа:** стек зафиксирован, открытые вопросы §5 закрыты решениями по умолчанию (см. §5.1). Отклоняться от них можно только через ADR в `docs/adr/`.

---

## 1. Стек одним взглядом

| Слой | Технология | Версия | Назначение |
|---|---|---|---|
| Язык / рантайм | Python | 3.12 | Основной язык сервиса |
| Менеджер зависимостей | **uv** | ≥ 0.4 | Lockfile + быстрый sync, см. `IMPLEMENTATION_PLAN.md` T-01 |
| Bot framework | **aiogram** | 3.x | Роутеры, FSM, middleware, вебхуки |
| HTTP-сервер | aiohttp | ≥ 3.9 | Совместим с aiogram `SimpleRequestHandler`, принимает и TG и платёжные вебхуки |
| HTTP-клиент | httpx | ≥ 0.27 | Для 3x-ui, proxy-provisioner, CryptoBot, Lava/Wata |
| ORM | **SQLAlchemy** (async) | 2.0 | Работа с БД |
| Async DB driver | asyncpg | latest | Через `postgresql+asyncpg://` |
| Миграции | Alembic | latest | Версионирование схемы, async env |
| СУБД | **PostgreSQL** | 16 | Пользователи, подписки, платежи, рефералы |
| Кэш / FSM / брокер | **Redis** | 7 | FSM storage, TTL счетов, дедуп, очередь ARQ |
| Task queue | **ARQ** | latest | Фоновые задачи: рассылки, напоминания, TTL инвойсов, outbox flush |
| Конфиг | pydantic-settings | 2.x | `.env` → типизированные настройки |
| Логи / алерты | structlog + Sentry | — | Структурированные JSON-логи, алерты ошибок |
| Шифрование | cryptography (Fernet) | latest | `proxy_accounts.password`, ключ в env |
| Тесты | pytest + pytest-asyncio + testcontainers | — | Unit, интеграционные (Postgres/Redis в контейнерах), e2e |
| HTTP-моки | respx | latest | Мокаем httpx-клиенты в тестах |
| Качество кода | ruff, mypy (strict), import-linter | — | Линт + статическая типизация + контроль границ слоёв |
| Контейнеризация | Docker + Docker Compose | ≥ 24.x | Единый способ запуска/деплоя |
| Reverse proxy / TLS | Caddy | 2.x | Вебхуки Telegram и платёжных провайдеров, автоматический Let's Encrypt |
| Бэкап-агент | restic **или** pgBackRest | latest | WAL-архив + ежедневный full, см. §4.12 |
| Хостинг | 1 VPS (2 vCPU / 4 GB) | — | Hetzner CX22 / Timeweb Cloud / аналог (Linux + Docker) |

---

## 2. Внешние интеграции

### 2.1. VLESS-панель

**3x-ui** ([MHSanaei/3x-ui](https://github.com/MHSanaei/3x-ui)) — источник VLESS-ключей.

- API: `POST/GET /panel/api/inbounds/*` (`addClient`, `updateClient/:clientId`, `:id/delClient/:clientId`, `list`, `getClientTraffics/:email`).
- **Аутентификация — только сессионная cookie** (login → cookie). Токенов/API-ключей в 3x-ui нет — это архитектурное ограничение, см. §4.7.
- Для тарифа **«Быстрый 1 мес.»** — отдельный inbound / нода с приоритетным QoS.

### 2.2. Прокси-сервис (SOCKS5 / HTTP)

Отдельный компонент `proxy-provisioner` с собственным API:
- Бэкенд: **3proxy** в контейнере + пул аккаунтов в БД.
- Автоочистка просроченных аккаунтов фоновой задачей ARQ.
- На старте допустим wrapper над внешним API провайдера прокси (Proxy6 / ProxyLine) — см. открытый вопрос в §5.

### 2.3. Платёжные провайдеры

| Способ | Провайдер | SDK / API |
|---|---|---|
| Telegram Stars | Telegram Bot API | `sendInvoice` с currency `XTR` (нативно в aiogram 3) |
| Crypto | **CryptoBot** | Официальный HTTP API (`@CryptoBot`) |
| СБП (QR) + карты | **YooKassa** (основной) | `yookassa` — официальный Python SDK |
| СБП (QR) + карты | **Lava** / **Wata** (резерв) | HTTP API — подключаются при блоке на KYC у YooKassa |

Все провайдеры скрыты за единым интерфейсом `PaymentProvider` (см. §4.2).

---

## 3. Инфраструктура

Один VPS, развёртывание через `docker compose`. **Все публичные пути зафиксированы ниже и используются одинаково во всех документах**:

```
┌────────────────────────────────────────────────────────────────┐
│ Caddy (TLS, reverse proxy, автоматический Let's Encrypt)       │
│   ├─ POST /tg/<TG_WEBHOOK_SECRET_PATH>   → bot:8080            │
│   ├─ POST /pay/yookassa                  → bot:8080            │
│   ├─ POST /pay/cryptobot                 → bot:8080            │
│   ├─ POST /pay/lava                      → bot:8080 (резерв)   │
│   ├─ POST /pay/wata                      → bot:8080 (резерв)   │
│   ├─ GET  /healthz                       → 200 inline          │
│   └─ GET  /healthz/worker                → bot:8080            │
│         (всё остальное → 404, публика закрыта)                 │
├────────────────────────────────────────────────────────────────┤
│ bot         — aiogram 3, webhook mode, aiohttp-app             │
│              (также принимает платёжные вебхуки;               │
│               fulfill вынесен в ARQ, см. §4.11)                │
│ worker      — ARQ: outbox flush, invoice TTL, expiry           │
│              reminders, broadcasts, xui compensation,          │
│              proxy cleanup, referral bonus, payment fulfill    │
│ postgres    — 16, volume `postgres_data`                       │
│ redis       — 7, AOF + RDB (persistent)                        │
│ proxy-prov. — 3proxy + mini-API (внутренняя сеть)              │
│ 3x-ui       — та же compose-сеть (по умолчанию)                │
│ uptime-kuma — health-monitoring                                │
└────────────────────────────────────────────────────────────────┘
```

- **Пути вебхуков** — канонический источник правды для `docker/Caddyfile` и `scripts/set_webhook.py`.
- **Бэкапы:** WAL-архив + ежедневный `pg_dump -Fc` → S3-совместимое хранилище (Backblaze B2 по умолчанию, см. §4.12). RPO ≤ 5 минут по платежам за счёт WAL.
- **Мониторинг:** Uptime Kuma (health-check бота, воркера и всех платёжных вебхуков) + Sentry (ошибки + performance-алерты).
- **Секреты:** `.env` на хосте с `chmod 600`, не в репозитории. В коде — через `pydantic-settings`. Fernet-ключ шифрования паролей прокси — отдельная переменная `PROXY_PASSWORD_KEY`.

---

## 4. Ключевые архитектурные решения

### 4.1. Webhook-режим бота, не long polling
Меньше задержка реакции, проще мониторинг, дешевле по трафику. TLS — через Caddy. Секрет живёт в двух местах:
- в URL: `/tg/<TG_WEBHOOK_SECRET_PATH>` (32+ символов nanoid);
- в заголовке: `X-Telegram-Bot-Api-Secret-Token` (свой, тоже 32+ симв.) — проверяется средствами aiogram `SimpleRequestHandler(secret_token=...)`.

Оба секрета устанавливаются скриптом `scripts/set_webhook.py` вместе с `allowed_updates` и `drop_pending_updates=False`.

### 4.2. Абстракция `PaymentProvider`
Бизнес-логика не знает про конкретного провайдера. Интерфейс:

```
create_invoice(amount, currency, metadata) -> InvoiceRef
handle_webhook(payload, headers)           -> PaymentEvent | None
```

Смена YooKassa ⇄ Lava ⇄ Wata не затрагивает сервисы подписок.

### 4.3. Идемпотентность платежей (3 уровня)
Вебхуки всех провайдеров повторяются (YooKassa ретрайнет при любом не-2xx, CryptoBot — по расписанию, Telegram Stars доставляется через обычные апдейты). Двойное начисление ключа недопустимо.
1. **Redis dedup.** Ключ `payment:dedup:<provider>:<provider_payment_id>` с TTL 24 ч (`SET NX EX 86400`). Если уже был — вебхук сразу возвращает 200 OK без работы.
2. **БД unique.** `UNIQUE (provider, provider_payment_id)` на `payments`. `INSERT … ON CONFLICT DO NOTHING RETURNING id` — если строка не вставилась, событие уже обработано.
3. **Outbox dedup.** `UNIQUE(dedup_key)` с ключом `payment:{payment_id}:receipt` — повторный fulfill не создаёт дубля уведомления.

Обработчик вебхука возвращает 200 **после ack (INSERT payments) и enqueue fulfill-таски в ARQ**, но до самой выдачи ключа (§4.11). Это исключает таймауты от провайдеров, если 3x-ui медленный.

### 4.4. Outbox-паттерн для исходящих уведомлений
Все уведомления пользователям (§12 `INTERFACE.md`) и рассылки пишутся в таблицу `outbox` **в той же DB-транзакции, что и бизнес-операция**. Отправляет их ARQ-воркер cron-таской `outbox_flush` раз в секунду. Даёт:
- корректные ретраи при 429/5xx от Telegram (`TelegramRetryAfter` учитывается);
- соблюдение глобального лимита **30 msg/s на бота** (bucket `tx`) и **25 msg/s при рассылках** (bucket `broadcast`) с общим ceiling 30 msg/s, чтобы не превысить лимит Telegram при одновременной tx-нагрузке;
- per-chat лимит 1 msg/s в ЛС (у нас только ЛС, групп нет);
- порядок доставки по `priority ASC, id ASC`;
- защиту от дублей через `UNIQUE(dedup_key)`.

Flusher использует `SELECT … FOR UPDATE SKIP LOCKED` — пригодно для горизонтального масштабирования воркеров без гонок.

### 4.5. Цены и баннеры — в БД, не в коде
Раздел §14.7 `INTERFACE.md` (админка) требует менять цены и изображения на лету. Таблицы `tariffs` и `banners` с `updated_at`. В коде — только дефолты для первоначального сида (`scripts/seed.py`).

**Snapshot цены при создании invoice.** Изменение цены в админке не должно менять стоимость уже открытого пользователем меню оплаты. Поэтому `invoices` фиксируют `amount_rub` / `amount_xtr` на момент создания (см. §5.3 `ARCHITECTURE.md`). Цена в UI инвалидируется при любом обновлении `tariffs.price_rub` — пользователь увидит свежую цену в следующем заходе в меню.

**Баннеры кэшируются как `telegram_file_id`.** Первая отправка — `InputFile(storage_path)`, затем только `file_id` в той же БД-строке. При замене баннера админом `file_id` обнуляется, следующая отправка снова прогреет кэш.

### 4.6. Централизованный `BanCheckMiddleware`
Требование `INTERFACE.md` §1: забанённый пользователь на любое действие получает `«Вы заблокированы.»`. Реализуется одной middleware aiogram на входе всех апдейтов. Без дублирования по хэндлерам.

Статус бана кэшируется в Redis (`user:ban:{tg_id}`, TTL 60 с); админская команда ban/unban инвалидирует ключ явно. Webhook-эндпоинты платежей бана не учитывают (платёжный event от банка обрабатывается даже для забаненного — деньги уже списаны).

### 4.7. 3x-ui клиент с реконнектом сессии
3x-ui не даёт API-токенов — только cookie-сессию. Клиент:
- поднимает сессию лениво, хранит cookie в **Redis (`xui:cookie`, TTL 55 мин)**, чтобы bot и worker не re-login'ились дважды и могли делить сессию;
- при 401/404 делает re-login с distributed-lock через Redis (`xui:login:lock`, TTL 10 с) и повторяет запрос (1 retry);
- все вызовы обёрнуты в custom circuit breaker (5 ошибок подряд → OPEN 60 с) — при падении панели бот не валится, ключ кладётся в `outbox` через saga-компенсацию (§4.8) и выдаётся задним числом воркером;
- **все вызовы имеют `httpx.Timeout(connect=5, read=15, write=10, pool=2)`**, чтобы не удерживать ARQ-джобы дольше минуты.

### 4.8. Транзакционная связка «подписка ↔ клиент 3x-ui»
Локальная запись `vpn_keys` и клиент в 3x-ui должны быть консистентны. Используется паттерн **Saga** с компенсацией:
1. Создаём запись в БД в статусе `pending` (единая транзакция с `payments` INSERT).
2. Создаём клиента в 3x-ui (**после коммита** транзакции шага 1, уже в ARQ-воркере).
3. Переводим `vpn_keys.status='active'`, `subscriptions.status='active'`, пишем outbox-уведомление (одна транзакция).

**Компенсации:**
- Ошибка шага 2 → запись `vpn_keys.status='failed'`, `subscriptions.status='failed'`, `outbox(provision_pending)` пользователю, enqueue `tasks.xui_compensation.retry_provision(sub_id)` с exp-backoff до 24 ч. Через 24 ч → Sentry + алерт админу + `outbox(provision_failed)` с просьбой обратиться в поддержку.
- Ошибка шага 3 (сохранение в БД после успешного 3x-ui) — **редкий кейс**: клиент удаляется из 3x-ui компенсирующей задачей `tasks.xui_compensation.delete_orphan_client(email)` (sweep раз в 10 мин по vpn_keys.status='failed' + xui_client_id NOT NULL).
- **Отзыв** (`revoke`): `XUIClient.delete_client` с компенсацией `tasks.xui_compensation.delete_client(inbound_id, client_id)` при фейле.

### 4.9. i18n-задел с первого коммита
Все пользовательские строки — через `t(key, locale, **args)` + словари локалей. В коде **нет ни одной строки UI** — только ключи. Добавление мультиязычности потом не потребует переписывать UI.

**Fallback-цепочка:** `user.language_code → settings.i18n.default_locale (ru) → ключ как есть (с WARN в Sentry)`. Пользователь с `language_code='kk'` не уронит бота — получит русский текст.

**CI-страж.** Скрипт `tools/check_i18n_coverage.py` парсит AST на вызовы `t("…")` и сверяет со всеми ключами в `locales/ru.yaml`. Недостающий ключ = красный CI.

### 4.10. Единая точка правды по времени
Все даты в БД — `TIMESTAMPTZ` в UTC. Отображение пользователю — через форматтер `alpaca/utils/time.py` с учётом часового пояса (по умолчанию `Europe/Moscow`). Это закрывает §6 `INTERFACE.md` («Действует до», «Осталось дней») без багов с DST.

### 4.11. Развязка webhook → fulfill через ARQ
Обработчик вебхука провайдера (`/pay/yookassa` и др.) **не вызывает 3x-ui / proxy-provisioner синхронно**. Он выполняет:
1. верификация подписи/IP;
2. Redis dedup;
3. `INSERT payments` + обновление `invoices.status='paid'`;
4. `arq.enqueue_job('fulfill_payment', payment_id)`;
5. ответ **200 OK ≤ 200 мс**.

Дальнейший fulfill (создание подписки, вызов 3x-ui, outbox) идёт в ARQ-воркере. Преимущества:
- webhook не таймаутит из-за медленного 3x-ui → провайдеры не ретрайнят;
- можно ретраить fulfill отдельно от приёма платежа;
- backpressure через ARQ-очередь вместо роста длительности HTTP-соединений.

Исключение — Telegram Stars: событие `successful_payment` приходит через основной апдейт-поток бота (webhook `/tg/...`), обрабатывается тем же `PaymentIngestor` и тоже enqueue'ит fulfill-джобу.

### 4.12. Бэкапы: WAL + full
Для финансовых данных недостаточно только `pg_dump` раз в сутки (RPO 24 ч). Используем гибрид:
- **WAL-архив** через `wal-g` или `pgBackRest` → S3-совместимое хранилище каждые 1–5 минут;
- **ежедневный `pg_dump -Fc`** → то же хранилище, retention 14 дней;
- **еженедельная проверка восстановления** — ARQ-джоба: `pg_restore --list` из последнего дампа в ephemeral контейнере + smoke-`SELECT COUNT(*) FROM payments`.

Восстановительный RPO по платежам — **≤ 5 минут**, RTO — **≤ 60 минут** (время на поднятие нового Postgres + применение WAL + перезапуск bot/worker).

### 4.13. FSM lifecycle
- **Storage:** aiogram `RedisStorage`, TTL данных = 1 час, ключи state — автоматически.
- **`/start` всегда очищает FSM.** Это правило прописано в `bot/handlers/common.py` (`await state.clear()` перед роутингом в главное меню). Без него пользователи зависают в «waiting_recipient» после недели отсутствия.
- **Клавиша «Назад»** возвращает на 1 шаг через стек `data["nav"]` (сериализуется в state).
- **Переходы между FSM-группами** явно вызывают `state.clear()` перед установкой нового группового состояния.

### 4.14. Таймауты (сквозные)
Фиксированные значения для всех внешних I/O:

| Компонент | Таймаут | Обоснование |
|---|---|---|
| Caddy `request_timeout` на `/pay/*` | 10 s | webhook-эндпоинты должны отвечать быстро (§4.11) |
| Caddy `request_timeout` на `/tg/*` | 30 s | allowed_updates могут прийти большими batches |
| `httpx.Timeout` 3x-ui | connect 5 / read 15 / write 10 s | панель бывает медленной, но не бесконечной |
| `httpx.Timeout` платёжные API | connect 5 / read 10 / write 5 s | YooKassa SDK имеет свой, не переопределяем |
| ARQ `job_timeout` общий | 120 s | для `fulfill_payment`, `xui_compensation` |
| ARQ `job_timeout` broadcast chunk | 300 s | 500 сообщений × 33 мс/шт + запас |
| Postgres `statement_timeout` | 15 s (сессия), 60 s (миграции) | ловит забытые full-scan'ы |
| Redis command timeout | 2 s | dedup/rate-limit на горячем пути |

---

## 5. Открытые вопросы

### 5.1. Принятые решения по умолчанию

Все вопросы из предыдущей версии этого раздела закрыты решениями по умолчанию. Менять — только через ADR (`docs/adr/*.md`).

| # | Вопрос | Принятое решение | ADR |
|---|---|---|---|
| 1 | Прокси на старте | **Свой `proxy-provisioner` на 3proxy** в отдельном контейнере. Интерфейс `ProxyBackend` сохраняет возможность добавить внешний провайдер без переписывания. | `docs/adr/0001-proxy-backend.md` |
| 2 | Хостинг 3x-ui | **На том же VPS в compose** (в дефолтной конфигурации). Если у заказчика уже есть внешняя панель — переопределяется через `XUI_BASE_URL` в `.env`. | `docs/adr/0002-xui-hosting.md` |
| 3 | YooKassa vs Lava/Wata | **MVP на YooKassa** (основной провайдер для RUB). Lava/Wata — готовые stub'ы в registry, включаются флагом `PAY_LAVA_ENABLED=1` при проблемах с KYC. | `docs/adr/0003-payments-registry.md` |
| 4 | Мультиязычность | **i18n-ключи с первого дня**; MVP поставляет только `locales/ru.yaml`. Добавление `en.yaml` — чистая трудозадача без изменений кода. Fallback на `ru` при отсутствии ключа. | `docs/adr/0004-i18n.md` |
| 5 | TTL инвойса | **15 минут.** Настраивается через `settings.limits.invoice_ttl_minutes`. Для Stars действует политика «если пришёл `successful_payment` после TTL — всё равно fulfill», см. §7 ARCHITECTURE. | `docs/adr/0005-invoice-ttl.md` |
| 6 | Баннеры: формат/хранение | **PNG 1280×720, до 512 КБ.** Хранение: локальный volume `banners/` + кэш `telegram_file_id` в БД. Админ загружает через бот → бот кладёт в volume и сбрасывает `file_id` для перезаливки. | `docs/adr/0006-banners.md` |
| 7 | «Безопасно» в главном меню | **Статический текст в MVP.** В схеме есть поле `health_status` как задел под честный статус (вычисляется из пингов 3x-ui). | — |
| 8 | Возврат при отзыве подписки | **По умолчанию без возврата.** Решение админа — ручное, через поддержку. UI §14.5.3 INTERFACE это явно проговаривает. | — |
| 9 | Бэкапы | **WAL-архив + ежедневный `pg_dump`** (см. §4.12). | `docs/adr/0007-backups.md` |
| 10 | Реферальный антифрод | **Бонус начисляется только за первую ПЛАТНУЮ активацию друга**, триал не считается. Ограничение: max 30 бонусных дней/сутки на одного реферрера. См. §10 `INTERFACE.md` и §6.5 `ARCHITECTURE.md`. | `docs/adr/0008-referral-antifraud.md` |
| 11 | Перезапуск бота из админки | **Кнопка «Перезапустить бота» убрана** из UI (риск для активных рассылок/fulfill). Рестарт — через `ssh`/compose. Вместо неё добавлена кнопка «Сбросить кэш» (§14.7.4 INTERFACE) — точечная инвалидация Redis. | `docs/adr/0009-no-restart-button.md` |

### 5.2. Реально открытые вопросы (блокеров нет)

Требуют подтверждения заказчика, но имеют рабочие дефолты:

1. **Домен и email Let's Encrypt** — ожидаем `alpaca.example.com` + `admin@example.com`. Бот не запустится без этих значений в `.env`.
2. **Реальные URL инструкций** (§9 INTERFACE) — сейчас ссылки на `telegra.ph`. Если заказчик хочет свой домен, правится в `alpaca/domain/links.py` без миграций.
3. **Описание тарифа «Быстрый 1 мес.»** — сейчас placeholder. Текст редактируется через `tariffs.description_key` в i18n без релиза.

---

## 6. Что входит и что не входит в стек

### 6.1. Зафиксировано на уровне стека (source of truth)
- Все технологии из §1 — не обсуждаются без ADR;
- Пути и порты вебхуков из §3;
- Политики идемпотентности (§4.3), outbox (§4.4), саги (§4.8), fulfill-async (§4.11);
- Таймауты (§4.14) и FSM lifecycle (§4.13).

### 6.2. Детализация в `ARCHITECTURE.md`
- Полная схема БД (таблицы, индексы, связи);
- Структура модулей и пакетов;
- Имена хэндлеров и FSM-состояний;
- Sequence-диаграммы сценариев;
- Конфигурация Caddy, Docker Compose, Alembic.

### 6.3. Out of scope
- Горизонтальное масштабирование за пределы 1 VPS (архитектура не препятствует, но деплой-скрипты не предусмотрены);
- Web-кабинет пользователя;
- Мобильное приложение;
- Polling-режим (только webhook).

---

## 7. Обоснование основных выборов

- **Python / aiogram 3** — самое крупное русскоязычное сообщество по Telegram-ботам, aiogram 3 даёт чистую архитектуру (роутеры, DI, типизированные фильтры), нативно поддерживает Telegram Stars (`XTR`).
- **PostgreSQL + Redis** — стандарт де-факто для финансовых данных + кэша/очередей. Для 5 000 пользователей с запасом 100×. Postgres 16 нужен ради логической репликации и `UNIQUE INDEX … NULLS NOT DISTINCT`.
- **ARQ** — нативно async, интегрируется с aiogram без прослоек, не требует отдельного брокера кроме Redis. Для наших объёмов (десятки джоб/сек) — с запасом.
- **3x-ui** — выбор заказчика; ограничения API (только cookie-сессия) учтены в §4.7.
- **YooKassa** — самый быстрый онбординг для СБП + карт в РФ при наличии юрлица/ИП; резервные провайдеры Lava/Wata подключаются через ту же абстракцию §4.2 одним флагом env.
- **1 VPS + Docker Compose** — под 5 000 пользователей избыточна любая оркестрация. Горизонтальное масштабирование — задача на будущее, архитектура ей не препятствует (stateless bot, Postgres/Redis вынесены в отдельные контейнеры, outbox работает с `SKIP LOCKED`).
- **Webhook-mode** — минимальная задержка ответа, проще мониторить через Caddy access-логи, нет postponed updates при рестарте.
- **Caddy вместо nginx** — автоматический Let's Encrypt из коробки, конфигурация в 20 строк вместо 200.

---

## 8. Ссылки на соседние документы

- [`ARCHITECTURE.md`](./ARCHITECTURE.md) — полная архитектура: схема БД, sequence, Saga, outbox, провайдеры, деплой.
- [`INTERFACE.md`](./INTERFACE.md) — пользовательские и админские потоки, тексты, кнопки, уведомления.
- [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md) — пофазный план работ с оценками и DoD.
