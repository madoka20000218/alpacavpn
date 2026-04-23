# Alpaca VPN — Архитектура проекта

> Полная архитектура Telegram-бота Alpaca VPN на основе `STACK.md` и `INTERFACE.md`.
> Рантайм: Python 3.12 + aiogram 3 + PostgreSQL 16 + Redis 7 + ARQ, webhook-режим, Docker Compose на 1 VPS.
> Цель документа — снять с разработчика все архитектурные решения: от структуры пакетов и схемы БД до последовательностей обработки платежей, провижининга VPN/прокси, outbox-доставки уведомлений, админки, i18n и деплоя.

**Документ читается сверху вниз.** Разделы §1–§3 дают общую картину, §4–§12 — детали подсистем, §13–§17 — эксплуатация и нефункциональные аспекты, §18 — открытые вопросы, которые остаются на заказчике, и §19 — план миграции с нулевого коммита до MVP.

---

## Оглавление

1. [Контекст и границы системы](#1-контекст-и-границы-системы)
2. [Контейнерная диаграмма (C4 level 2)](#2-контейнерная-диаграмма-c4-level-2)
3. [Структура репозитория и пакетов](#3-структура-репозитория-и-пакетов)
4. [Слоистая архитектура и зависимости](#4-слоистая-архитектура-и-зависимости)
5. [Доменная модель и схема БД](#5-доменная-модель-и-схема-бд)
6. [Ключевые сценарии (sequence)](#6-ключевые-сценарии-sequence)
7. [Payments: абстракция и провайдеры](#7-payments-абстракция-и-провайдеры)
8. [VPN-провижининг через 3x-ui](#8-vpn-провижининг-через-3x-ui)
9. [Proxy-provisioner (SOCKS5/HTTP)](#9-proxy-provisioner-socks5http)
10. [Outbox и фоновый воркер (ARQ)](#10-outbox-и-фоновый-воркер-arq)
11. [FSM, роутеры и middleware](#11-fsm-роутеры-и-middleware)
12. [Админ-панель](#12-админ-панель)
13. [Конфигурация, секреты, i18n](#13-конфигурация-секреты-i18n)
14. [Деплой: Docker Compose + Caddy](#14-деплой-docker-compose--caddy)
15. [Наблюдаемость и алертинг](#15-наблюдаемость-и-алертинг)
16. [Безопасность](#16-безопасность)
17. [Тестирование, миграции, CI/CD](#17-тестирование-миграции-cicd)
18. [Открытые вопросы (для заказчика)](#18-открытые-вопросы-для-заказчика)
19. [Дорожная карта реализации](#19-дорожная-карта-реализации)

---

## 1. Контекст и границы системы

### 1.1. Акторы и внешние системы

```
                    ┌──────────────────┐
                    │   Telegram user  │
                    └────────┬─────────┘
                             │ Telegram messages / callbacks
                             ▼
 ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐
 │  Admin user  │───▶│   Alpaca VPN     │───▶│  Telegram Bot API│
 └──────────────┘    │   (bot + worker) │    └──────────────────┘
                     │                  │
                     │                  │───▶ 3x-ui panel   (VLESS clients)
                     │                  │───▶ 3proxy / ext. │ (SOCKS5/HTTP accounts)
                     │                  │───▶ YooKassa / CryptoBot / Lava / Wata
                     │                  │───▶ Telegram Stars (через Bot API)
                     │                  │───▶ Sentry        (ошибки)
                     │                  │───▶ S3-совместимое хранилище (бэкапы, опц. баннеры)
                     └──────────────────┘
```

**Alpaca VPN — единый сервис**, который:
- принимает апдейты от Telegram через webhook;
- принимает вебхуки от YooKassa / CryptoBot / Lava / Wata;
- вызывает API 3x-ui и proxy-бэкенда;
- пишет в PostgreSQL, кэширует в Redis, запускает фоновые задачи в ARQ.

### 1.2. Нефункциональные требования (из STACK §1, §7)

| # | Требование | Целевой показатель |
|---|---|---|
| NFR-1 | Пропускная способность | до 5 000 пользователей суммарно, ≤ 50 RPS пиков |
| NFR-2 | Доступность бота | ≥ 99.5% месяц (SLO), алерт при downtime > 5 мин |
| NFR-3 | Время ответа на callback | p95 ≤ 300 мс (без внешних API), p95 ≤ 2 с (с 3x-ui/платежами) |
| NFR-4 | Глобальный лимит Telegram | 30 msg/s на бота, 25 msg/s при массовой рассылке |
| NFR-5 | Идемпотентность платежей | 0 двойных начислений при повторных вебхуках |
| NFR-6 | Восстановление после сбоя панели | бот не валится; ключ выдаётся задним числом через outbox |
| NFR-7 | Бэкапы БД | ежедневно, хранение 14 дней, RPO ≤ 24 ч |
| NFR-8 | Горизонтальная масштабируемость | stateless bot + worker, Postgres/Redis вынесены отдельно |

### 1.3. Что в scope / out of scope MVP

**В scope MVP:**
- UI бота (§2–§11 INTERFACE.md), админ-панель (§14 INTERFACE.md);
- покупка/продление/подарок подписок, триал 3 дня, реферальная программа (2 активации = 1 день);
- 4 способа оплаты: Telegram Stars, CryptoBot, YooKassa (СБП + карта);
- VLESS-провижининг через 3x-ui (основной inbound + «быстрый» inbound);
- proxy-provisioner через 3proxy в контейнере (опция А, см. §18);
- уведомления пользователям (§12 INTERFACE.md) и рассылки (§14.3);
- i18n-задел (ru по умолчанию, структура готова под en);
- мониторинг Sentry + Uptime Kuma + structlog.

**Out of scope MVP** (делаем задел, но не имплементируем):
- кабинет в веб-браузере;
- мобильное приложение;
- мультиязычные переводы (только ключи + русский словарь);
- интеграция с Lava/Wata (оставлены интерфейсы);
- горизонтальный масштаб (один инстанс bot + один worker).

---

## 2. Контейнерная диаграмма (C4 level 2)

```
 ┌───────────────────────────────────────────────────────────────────────────┐
 │                                 VPS (Hetzner/Timeweb)                     │
 │                                                                           │
 │  ┌─────────────────────────┐                                              │
 │  │        Caddy 2.x        │  TLS, HTTP/2, reverse proxy                  │
 │  │  /tg/<secret>           │─────────────┐                                │
 │  │  /pay/yookassa          │─────────────┤                                │
 │  │  /pay/cryptobot         │─────────────┤                                │
 │  │  /pay/lava │ /pay/wata  │─────────────┤                                │
 │  │  /healthz               │─────────────┤                                │
 │  └─────────────────────────┘             │                                │
 │                                          ▼                                │
 │  ┌─────────────────────────────────────────────────────────────────────┐  │
 │  │                    bot (aiohttp + aiogram 3)                        │  │
 │  │  - webhook endpoint (SimpleRequestHandler + secret_token)           │  │
 │  │  - payments webhook endpoints (provider-specific)                   │  │
 │  │  - aiogram FSM на Redis                                             │  │
 │  │  - domain services, repositories (SQLAlchemy async)                 │  │
 │  │  - пишет в outbox, 3x-ui, proxy API                                 │  │
 │  └─────────────────────────────────────────────────────────────────────┘  │
 │           │ async SQL            │ Redis              │ HTTP/S            │
 │           ▼                      ▼                    ▼                   │
 │  ┌────────────────┐  ┌─────────────────────┐  ┌─────────────────────────┐ │
 │  │ PostgreSQL 16  │  │ Redis 7 (AOF)       │  │ 3x-ui (пан. HTTP API)   │ │
 │  │                │  │ - FSM storage       │  │ (на том же хосте или    │ │
 │  │                │  │ - dedup кэш         │  │  удалённо)              │ │
 │  │                │  │ - ARQ queue         │  └─────────────────────────┘ │
 │  │                │  │ - rate-limit meter  │  ┌─────────────────────────┐ │
 │  └────────────────┘  └─────────────────────┘  │ proxy-provisioner       │ │
 │           ▲                      ▲            │ (3proxy + внутр. API)   │ │
 │           │                      │            └─────────────────────────┘ │
 │  ┌─────────────────────────────────────────────────────────────────────┐  │
 │  │                    worker (ARQ)                                      │  │
 │  │  - outbox flusher (Telegram send + retry + rate limit)              │  │
 │  │  - invoice TTL cancel                                                │  │
 │  │  - expiry reminders (T-3, T-0)                                       │  │
 │  │  - broadcasts (чанки, прогресс)                                      │  │
 │  │  - 3x-ui compensations (saga)                                        │  │
 │  │  - proxy pool cleanup                                                │  │
 │  │  - referral bonus grant                                              │  │
 │  └─────────────────────────────────────────────────────────────────────┘  │
 │                                                                           │
 │  ┌──────────────────┐  ┌──────────────────┐                               │
 │  │ Uptime Kuma      │  │ node_exporter    │  (опционально)                │
 │  │ (health checks)  │  │ + Prometheus     │                               │
 │  └──────────────────┘  └──────────────────┘                               │
 │                                                                           │
 └───────────────────────────────────────────────────────────────────────────┘
         │                                     │
         ▼                                     ▼
   Sentry (cloud)                        B2 / S3 (pg_dump бэкапы, опц. баннеры)
```

**Ключевые потоки:**
- **Telegram → Caddy → bot** (webhook `/tg/<secret>`): все апдейты пользователя.
- **Платёжный провайдер → Caddy → bot** (`/pay/<provider>`): подтверждение оплаты.
- **bot → Postgres**: транзакционные операции (пользователи, подписки, платежи).
- **bot → Redis**: FSM, dedup кэш, очередь ARQ.
- **worker → Telegram Bot API**: исходящая доставка (outbox).
- **bot/worker → 3x-ui / proxy-provisioner**: провижининг.

---

## 3. Структура репозитория и пакетов

### 3.1. Верхнеуровневая структура

```
alpaca/
├── alembic/                     # миграции
│   ├── versions/
│   ├── env.py
│   └── script.py.mako
├── alpaca/                      # python package
│   ├── __init__.py
│   ├── __main__.py              # точка входа: `python -m alpaca`
│   ├── bot.py                   # aiohttp-приложение + aiogram Dispatcher
│   ├── worker.py                # ARQ WorkerSettings
│   ├── config.py                # pydantic-settings: Settings
│   ├── logging.py               # structlog config + Sentry init
│   ├── di.py                    # провайдер зависимостей (aiogram DI + worker ctx)
│   │
│   ├── domain/                  # чистая бизнес-логика (без I/O)
│   │   ├── entities.py          # dataclass/enum: SubscriptionPlan, ProxyPlan, ...
│   │   ├── errors.py            # DomainError, PaymentError, ProvisionError
│   │   ├── pricing.py           # расчёт цен, скидок, периодов
│   │   ├── referral.py          # правила начисления бонусов
│   │   └── policies.py          # «можно ли подарить»,  «можно ли продлить» и т.п.
│   │
│   ├── db/
│   │   ├── base.py              # DeclarativeBase, naming convention
│   │   ├── session.py           # async_sessionmaker, engine
│   │   ├── models/              # SQLAlchemy-модели
│   │   │   ├── user.py
│   │   │   ├── subscription.py
│   │   │   ├── vpn_key.py
│   │   │   ├── proxy_account.py
│   │   │   ├── payment.py
│   │   │   ├── invoice.py
│   │   │   ├── tariff.py
│   │   │   ├── banner.py
│   │   │   ├── referral.py
│   │   │   ├── outbox.py
│   │   │   └── audit_log.py
│   │   └── repositories/        # доступ к данным (тонкий слой)
│   │       ├── users.py
│   │       ├── subscriptions.py
│   │       ├── payments.py
│   │       ├── invoices.py
│   │       ├── outbox.py
│   │       └── tariffs.py
│   │
│   ├── services/                # прикладные сервисы (оркестрация)
│   │   ├── subscription_service.py   # покупка, продление, выдача, отзыв
│   │   ├── gift_service.py
│   │   ├── trial_service.py
│   │   ├── referral_service.py
│   │   ├── proxy_service.py
│   │   ├── broadcast_service.py
│   │   ├── stats_service.py
│   │   └── admin_service.py
│   │
│   ├── payments/                # абстракция платежей
│   │   ├── base.py              # PaymentProvider Protocol, dataclasses
│   │   ├── registry.py          # маппинг provider_code → PaymentProvider
│   │   ├── stars.py             # Telegram Stars (XTR)
│   │   ├── cryptobot.py
│   │   ├── yookassa.py
│   │   ├── lava.py              # stub для резерва
│   │   └── wata.py              # stub для резерва
│   │
│   ├── integrations/
│   │   ├── xui/                 # клиент 3x-ui
│   │   │   ├── client.py        # httpx.AsyncClient + session cookie + retry
│   │   │   ├── models.py        # pydantic DTO inbound/client
│   │   │   └── errors.py
│   │   └── proxy_provisioner/
│   │       ├── client.py
│   │       └── models.py
│   │
│   ├── bot/                     # всё, что про Telegram-интерфейс
│   │   ├── dispatcher.py        # сборка Dispatcher + Router tree
│   │   ├── middlewares/
│   │   │   ├── db_session.py    # открыть AsyncSession на апдейт
│   │   │   ├── user.py          # upsert User, инъекция в data
│   │   │   ├── ban_check.py     # §4.6 STACK
│   │   │   ├── i18n.py          # выбор локали
│   │   │   ├── throttle.py      # per-user антиспам
│   │   │   └── logging.py       # request_id, structlog bind
│   │   ├── filters/
│   │   │   ├── admin.py         # IsAdmin
│   │   │   └── callback_data.py # aiogram CallbackData factory
│   │   ├── keyboards/
│   │   │   ├── main_menu.py
│   │   │   ├── tariffs.py
│   │   │   ├── proxy.py
│   │   │   ├── payment.py
│   │   │   ├── subscriptions.py
│   │   │   ├── extend.py
│   │   │   ├── gift.py
│   │   │   ├── instruction.py
│   │   │   ├── partner.py
│   │   │   ├── help.py
│   │   │   └── admin/
│   │   ├── handlers/
│   │   │   ├── common.py        # /start, /help, Назад
│   │   │   ├── main_menu.py
│   │   │   ├── buy.py
│   │   │   ├── proxy.py
│   │   │   ├── subscriptions.py
│   │   │   ├── extend.py
│   │   │   ├── gift.py
│   │   │   ├── instruction.py
│   │   │   ├── partner.py
│   │   │   ├── help.py
│   │   │   ├── payment.py       # выбор способа + прожатие кнопки
│   │   │   ├── trial.py
│   │   │   └── admin/           # см. §12
│   │   ├── states.py            # все StatesGroup в одном месте
│   │   ├── texts.py              # i18n-ключи, загрузка словарей
│   │   ├── renderer.py          # сборка (text, kb, photo) из доменных данных
│   │   └── webhook.py           # SimpleRequestHandler + payment webhooks
│   │
│   ├── outbox/
│   │   ├── dispatcher.py        # выбор pending записей, rate-limit, отправка
│   │   ├── telegram_sender.py
│   │   └── errors.py
│   │
│   ├── tasks/                   # ARQ-задачи
│   │   ├── broadcast.py
│   │   ├── expiry_reminders.py
│   │   ├── invoice_ttl.py
│   │   ├── outbox_flush.py
│   │   ├── xui_compensation.py
│   │   ├── proxy_cleanup.py
│   │   └── referral_bonus.py
│   │
│   └── utils/
│       ├── time.py              # TZ-форматтер (Europe/Moscow)
│       ├── ids.py               # ULID/UUID7 генератор
│       ├── retry.py             # backoff helpers
│       └── security.py          # constant-time compare, token utils
│
├── banners/                     # дефолтные PNG (сидим в БД при первом запуске)
│   ├── glavmenu.png
│   ├── tarif.png
│   ├── proxy.png
│   ├── sub.png
│   ├── extend.png
│   ├── gift.png
│   ├── inst.png
│   ├── partner.png
│   └── pomosh.png
│
├── locales/
│   └── ru.yaml                  # плоский словарь ключ → текст
│
├── tests/
│   ├── conftest.py              # фикстуры: контейнерная pg/redis, httpx mock
│   ├── unit/
│   ├── integration/
│   └── e2e/                     # полные сценарии webhook → outbox
│
├── scripts/
│   ├── seed.py                  # первичный сид тарифов, баннеров, админов
│   ├── set_webhook.py           # setWebhook + secret_token
│   └── backup.sh                # pg_dump → S3
│
├── docker/
│   ├── Dockerfile
│   ├── entrypoint.sh
│   └── Caddyfile
├── docker-compose.yml
├── docker-compose.override.yml.example
├── pyproject.toml               # uv / poetry
├── alembic.ini
├── .env.example
├── .editorconfig
├── .pre-commit-config.yaml
├── README.md
└── LICENSE
```

### 3.2. Правила именования и модульности

- **Python-пакет один — `alpaca`.** Никаких `src/`, чтобы не плодить вложенность под один сервис.
- **Один контейнер с образом**, два процесса: `bot` и `worker` (разные `command` в compose).
- **Границы слоёв**: `bot/*` → `services/*` → `db/repositories/*` + `integrations/*` + `domain/*`. Импорт «наверх» запрещён (проверяется `ruff` + `import-linter`).
- **Handlers не работают с БД напрямую** — только через сервисы. Сервисы не зависят от aiogram.

---

## 4. Слоистая архитектура и зависимости

```
┌──────────────────────────────────────────────────────────────────┐
│  Presentation (aiogram)                                          │
│  alpaca.bot.handlers / keyboards / middlewares / renderer        │
└──────────────────────────────────────────────────────────────────┘
                 │ вызывает только application services
                 ▼
┌──────────────────────────────────────────────────────────────────┐
│  Application services (use-case orchestration)                   │
│  alpaca.services.* + alpaca.outbox + alpaca.tasks                │
└──────────────────────────────────────────────────────────────────┘
     │                 │                  │                   │
     ▼                 ▼                  ▼                   ▼
┌────────────┐  ┌───────────────┐  ┌────────────────┐  ┌────────────┐
│ Domain     │  │ DB (repos,    │  │ Integrations   │  │ Payments   │
│ (pure)     │  │ models, uow)  │  │ (3x-ui, proxy) │  │ providers  │
└────────────┘  └───────────────┘  └────────────────┘  └────────────┘
```

### 4.1. Unit of Work и транзакции

- В `bot/middlewares/db_session.py` создаётся один `AsyncSession` на апдейт и кладётся в `data["session"]`.
- Сервисы принимают `session: AsyncSession` и работают через репозитории.
- Коммит/роллбэк — на уровне middleware (after handler). При необходимости вложенные транзакции через `session.begin_nested()`.
- Worker-таски открывают свою сессию через ctx.

### 4.2. Dependency Injection

- В `alpaca.di.build_di(settings)` собираются singleton'ы: `engine`, `redis`, `XUIClient`, `ProxyClient`, `PaymentRegistry`, `Outbox`.
- Передаются в `Dispatcher(**deps)` и в ARQ `ctx` через `on_startup` worker'а.
- Сессия БД и текущий `User` инжектятся middleware, а не через DI (зависят от апдейта).

### 4.3. Ошибки

- Все доменные ошибки — подклассы `DomainError` (`NotFoundError`, `ValidationError`, `PaymentError`, `ProvisionError`, `BannedUserError`, …).
- Центральный `error_handler` в aiogram отвечает пользователю по таблице §13 INTERFACE.md и отправляет в Sentry.
- Вызовы внешних API — через `tenacity` retry + circuit breaker (см. §8 и §10).

---

## 5. Доменная модель и схема БД

### 5.1. ER-диаграмма (текстом)

```
users ──< subscriptions ──< vpn_keys
  │           │                │
  │           └── outbox       └── (audit_log)
  │
  ├──< payments ──── invoices
  │       │
  │       └── subscriptions (fulfilled by payment)
  │
  ├──< proxy_accounts
  │
  ├──< referrals (referrer_id, referee_id)
  │
  └──< user_bans (история бан/анбан)

tariffs (ref) ──< subscriptions
banners (kv store)
admin_users (ref)
```

### 5.2. Конвенции БД

- Все первичные ключи — `BIGINT GENERATED BY DEFAULT AS IDENTITY` для таблиц с высокой скоростью, `UUID` для платёжных сущностей (удобнее для идемпотентности, т.к. провайдеры возвращают свой ID).
- Все временные метки — `TIMESTAMPTZ DEFAULT now()`. Таймзоны — только UTC, форматирование — на уровне renderer (§4.10 STACK).
- Каждая таблица содержит `created_at`, изменяемые — `updated_at`.
- Денежные поля — `NUMERIC(12,2)` в рублях (и `NUMERIC(12,0)` для XTR/Stars, целые).
- Naming convention Alembic — в `db/base.py` через `MetaData(naming_convention=...)`.
- Soft delete не используем (усложняет аналитику); вместо него — статусные поля.

### 5.3. Таблицы

#### `users`
Единая запись о человеке в системе (пользователь или админ).

| Поле | Тип | Описание |
|---|---|---|
| id | BIGSERIAL PK | внутренний ID |
| tg_id | BIGINT UNIQUE NOT NULL | Telegram user id |
| username | TEXT NULL | без `@`, хранится в нижнем регистре |
| first_name | TEXT NULL | |
| last_name | TEXT NULL | |
| language_code | TEXT DEFAULT 'ru' | для i18n |
| is_banned | BOOLEAN NOT NULL DEFAULT false | кэшируется в Redis (§11) |
| is_admin | BOOLEAN NOT NULL DEFAULT false | дублирует `ADMIN_IDS` из env (§13) |
| referrer_id | BIGINT NULL FK users.id | кто пригласил (set once) |
| referral_code | TEXT UNIQUE NOT NULL | короткий код (nanoid, 8 симв.) |
| trial_used | BOOLEAN NOT NULL DEFAULT false | для §2.1 INTERFACE |
| referral_bonus_days_granted | INT NOT NULL DEFAULT 0 | сколько дней уже начислено |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |
| last_seen_at | TIMESTAMPTZ | обновляется middleware |

Индексы: `UNIQUE(tg_id)`, `UNIQUE(referral_code)`, `UNIQUE(LOWER(username))` (частичный, `WHERE username IS NOT NULL`), `BTREE(referrer_id)`, `BTREE(created_at)`.

#### `user_bans`
История банов (для аудита).

| Поле | Тип | Описание |
|---|---|---|
| id | BIGSERIAL PK | |
| user_id | BIGINT FK users.id | |
| banned_at | TIMESTAMPTZ | |
| unbanned_at | TIMESTAMPTZ NULL | |
| banned_by | BIGINT NULL FK users.id | админ |
| reason | TEXT NULL | |

#### `tariffs`
Справочник тарифов, редактируется в админке (§4.5 STACK, §14.7.1 INTERFACE).

| Поле | Тип | Описание |
|---|---|---|
| id | BIGSERIAL PK | |
| code | TEXT UNIQUE NOT NULL | `WEEK`, `MONTH`, `FAST_MONTH`, `Q`, `H`, `YEAR`, `PROXY_SOCKS_WEEK`, ... |
| kind | TEXT NOT NULL | `vpn` | `vpn_fast` | `proxy_socks5` | `proxy_http` |
| title_key | TEXT NOT NULL | i18n-ключ для UI |
| duration_days | INT NOT NULL | |
| price_rub | NUMERIC(12,2) NOT NULL | базовая цена |
| price_rub_strikethrough | NUMERIC(12,2) NULL | «зачёркнутая» цена (§0.2 INTERFACE) |
| price_rub_extend | NUMERIC(12,2) NULL | цена продления (§7 INTERFACE) |
| price_xtr | NUMERIC(12,0) NULL | в звёздах |
| giftable | BOOLEAN NOT NULL DEFAULT true | для «Подарить другу» |
| xui_inbound_id | INT NULL | какой inbound использовать (быстрый vs обычный) |
| is_active | BOOLEAN NOT NULL DEFAULT true | |
| sort_order | INT NOT NULL DEFAULT 0 | |
| created_at / updated_at | TIMESTAMPTZ | |

Сид по умолчанию — в `scripts/seed.py` ровно по таблицам §3 и §4 INTERFACE.md.

#### `banners`
Картинки меню (§14.7.2 INTERFACE).

| Поле | Тип | Описание |
|---|---|---|
| id | BIGSERIAL PK | |
| code | TEXT UNIQUE NOT NULL | `glavmenu`, `tarif`, `proxy`, `sub`, `extend`, `gift`, `inst`, `partner`, `pomosh` |
| telegram_file_id | TEXT NULL | file_id после первой загрузки (быстро) |
| storage_path | TEXT NULL | путь в volume, если file_id ещё не получен |
| content_hash | TEXT NOT NULL | sha256 для инвалидации |
| updated_at | TIMESTAMPTZ | |

**Паттерн**: при первом использовании баннер шлётся как `InputFile(storage_path)`, `file_id` из ответа сохраняется. Далее — только `file_id` (0 I/O).

#### `subscriptions`
Экземпляр подписки конкретного пользователя.

| Поле | Тип | Описание |
|---|---|---|
| id | BIGSERIAL PK | |
| user_id | BIGINT NOT NULL FK users.id | |
| tariff_id | BIGINT NOT NULL FK tariffs.id | |
| kind | TEXT NOT NULL | `vpn` | `vpn_fast` | `proxy_socks5` | `proxy_http` (дубль из tariff для индексов) |
| status | TEXT NOT NULL | `pending` | `active` | `expired` | `revoked` | `failed` |
| source | TEXT NOT NULL | `purchase` | `gift_received` | `trial` | `admin_grant` | `referral_bonus` |
| started_at | TIMESTAMPTZ | фактическое начало |
| expires_at | TIMESTAMPTZ NOT NULL | срок действия |
| gifted_by_user_id | BIGINT NULL FK users.id | |
| revoked_at | TIMESTAMPTZ NULL | |
| revoked_by | BIGINT NULL FK users.id | |
| payment_id | UUID NULL FK payments.id | если оплачено |
| created_at / updated_at | TIMESTAMPTZ | |

Индексы: `BTREE(user_id, status)`, `BTREE(expires_at) WHERE status='active'` (для напоминаний и cron expiry).

**Правила (§7 INTERFACE — продление):**
- если есть `active` подписка того же `kind` — период прибавляется к `expires_at` текущей, новой записи не создаётся;
- если `expired`/нет — создаётся новая с `started_at = now()`, `expires_at = now() + duration`.

#### `vpn_keys`
Связь подписки с клиентом в 3x-ui (saga-паттерн, §4.8 STACK).

| Поле | Тип | Описание |
|---|---|---|
| id | BIGSERIAL PK | |
| subscription_id | BIGINT NOT NULL FK subscriptions.id | |
| xui_inbound_id | INT NOT NULL | |
| xui_client_id | TEXT NOT NULL | UUID в 3x-ui |
| xui_client_email | TEXT NOT NULL | идентификатор клиента в 3x-ui, формат `u{user_id}-s{sub_id}` |
| vless_url | TEXT NOT NULL | `vless://...` для выдачи пользователю |
| status | TEXT NOT NULL | `pending` | `active` | `revoked` | `failed` |
| last_synced_at | TIMESTAMPTZ NULL | последняя сверка с панелью |
| created_at / updated_at | TIMESTAMPTZ | |

Индексы: `UNIQUE(xui_client_id)`, `BTREE(subscription_id)`, `BTREE(status)`.

#### `proxy_accounts`
Аккаунт SOCKS5/HTTP-прокси.

| Поле | Тип | Описание |
|---|---|---|
| id | BIGSERIAL PK | |
| subscription_id | BIGINT NOT NULL FK subscriptions.id | |
| kind | TEXT NOT NULL | `socks5` | `http` |
| host | TEXT NOT NULL | |
| port | INT NOT NULL | |
| login | TEXT NOT NULL | |
| password | TEXT NOT NULL | хранится в зашифрованном виде (Fernet), ключ в env |
| provider | TEXT NOT NULL | `local_3proxy` | `proxy6` | `proxyline` |
| external_id | TEXT NULL | id во внешнем API провайдера |
| status | TEXT NOT NULL | `pending` | `active` | `expired` | `failed` |
| expires_at | TIMESTAMPTZ NOT NULL | |
| created_at / updated_at | TIMESTAMPTZ | |

Индексы: `BTREE(subscription_id)`, `BTREE(expires_at) WHERE status='active'`.

#### `invoices`
Исходящие счета на оплату (один invoice — один способ оплаты — одна подписка).

| Поле | Тип | Описание |
|---|---|---|
| id | UUID PK | наш внутренний id, идёт в `metadata` провайдера |
| user_id | BIGINT NOT NULL FK users.id | |
| intent | TEXT NOT NULL | `buy` | `extend` | `gift` | `proxy` |
| tariff_id | BIGINT NOT NULL FK tariffs.id | |
| target_user_id | BIGINT NULL FK users.id | для gift |
| amount_rub | NUMERIC(12,2) NULL | |
| amount_xtr | NUMERIC(12,0) NULL | |
| currency | TEXT NOT NULL | `RUB` | `XTR` | `USDT` | … |
| provider | TEXT NOT NULL | `stars` | `cryptobot` | `yookassa` | `lava` | `wata` |
| provider_invoice_id | TEXT NULL | id счёта в провайдере (может прийти позже) |
| provider_payload | JSONB NULL | ответ создания счёта, payment_url, QR и т.п. |
| status | TEXT NOT NULL | `created` | `pending` | `paid` | `expired` | `canceled` | `failed` |
| expires_at | TIMESTAMPTZ NOT NULL | TTL 15 минут (§5 INTERFACE) |
| paid_at | TIMESTAMPTZ NULL | |
| created_at / updated_at | TIMESTAMPTZ | |

Индексы: `BTREE(user_id, status, created_at DESC)`, `UNIQUE(provider, provider_invoice_id) WHERE provider_invoice_id IS NOT NULL`, `BTREE(expires_at) WHERE status IN ('created','pending')` — для TTL-таски.

#### `payments`
Подтверждённые платежи (1:1 с `invoices` в успешном кейсе).

| Поле | Тип | Описание |
|---|---|---|
| id | UUID PK | |
| invoice_id | UUID NOT NULL UNIQUE FK invoices.id | |
| user_id | BIGINT NOT NULL FK users.id | |
| provider | TEXT NOT NULL | |
| provider_payment_id | TEXT NOT NULL | id транзакции |
| amount_rub | NUMERIC(12,2) NULL | |
| amount_xtr | NUMERIC(12,0) NULL | |
| currency | TEXT NOT NULL | |
| raw_payload | JSONB NOT NULL | сырое тело вебхука (для разбирательств) |
| status | TEXT NOT NULL | `succeeded` | `refunded` | `chargeback` |
| received_at | TIMESTAMPTZ NOT NULL | |

Индексы: `UNIQUE(provider, provider_payment_id)` — **реализует §4.3 STACK (идемпотентность)**.

#### `referrals`
Связь реферреров и приглашённых.

| Поле | Тип | Описание |
|---|---|---|
| id | BIGSERIAL PK | |
| referrer_id | BIGINT NOT NULL FK users.id | |
| referee_id | BIGINT NOT NULL UNIQUE FK users.id | один человек — один реферрер |
| joined_at | TIMESTAMPTZ NOT NULL | момент `/start` с ?ref=... |
| first_activation_at | TIMESTAMPTZ NULL | первая активированная подписка, в т.ч. триал |
| counted_for_bonus | BOOLEAN NOT NULL DEFAULT false | зачтено ли при начислении бонусов |

Индексы: `BTREE(referrer_id, first_activation_at)`.

**Правило §10 INTERFACE:** раз в `activation` проверяем `COUNT(*) WHERE referrer=me AND first_activation_at IS NOT NULL AND counted_for_bonus=false`; если ≥ 2 — транзакционно помечаем 2 записи `counted_for_bonus=true` и начисляем реферреру `subscriptions(source='referral_bonus', duration_days=1)`.

#### `outbox`
Отложенные сообщения пользователям (§4.4 STACK, §12 INTERFACE).

| Поле | Тип | Описание |
|---|---|---|
| id | BIGSERIAL PK | |
| user_id | BIGINT NOT NULL FK users.id | кому отправить |
| kind | TEXT NOT NULL | `text` | `photo` | `invoice` (Stars) | `document` |
| payload | JSONB NOT NULL | `{ "text": "...", "photo_file_id": "...", "reply_markup": {...}, "parse_mode": "HTML" }` |
| dedup_key | TEXT NULL | идемпотентность (напр. `payment:{payment_id}:receipt`) |
| priority | SMALLINT NOT NULL DEFAULT 100 | меньше = раньше |
| available_at | TIMESTAMPTZ NOT NULL DEFAULT now() | отложенные |
| attempts | INT NOT NULL DEFAULT 0 | |
| max_attempts | INT NOT NULL DEFAULT 10 | |
| last_error | TEXT NULL | |
| status | TEXT NOT NULL | `pending` | `sent` | `failed` |
| sent_at | TIMESTAMPTZ NULL | |
| created_at | TIMESTAMPTZ | |

Индексы: `UNIQUE(dedup_key) WHERE dedup_key IS NOT NULL`, `BTREE(status, available_at) WHERE status='pending'`.

#### `audit_log`
Административные действия (бан/разбан, отзыв подписки, изменение цен).

| Поле | Тип | Описание |
|---|---|---|
| id | BIGSERIAL PK | |
| actor_id | BIGINT NOT NULL FK users.id | админ |
| action | TEXT NOT NULL | `ban_user`, `revoke_sub`, `change_price`, `upload_banner`, `broadcast`, `grant_sub`, `add_admin`, … |
| target_type | TEXT NULL | `user`, `subscription`, `tariff`, ... |
| target_id | TEXT NULL | string для полиморфного хранения |
| metadata | JSONB | |
| created_at | TIMESTAMPTZ | |

#### `broadcasts`
Рассылки (§14.3 INTERFACE).

| Поле | Тип | Описание |
|---|---|---|
| id | BIGSERIAL PK | |
| created_by | BIGINT FK users.id | |
| filter_json | JSONB | критерий отбора |
| text | TEXT NOT NULL | |
| banner_id | BIGINT NULL FK banners.id | |
| total | INT NOT NULL DEFAULT 0 | |
| sent_ok | INT NOT NULL DEFAULT 0 | |
| sent_fail | INT NOT NULL DEFAULT 0 | |
| status | TEXT NOT NULL | `draft` | `running` | `done` | `canceled` |
| created_at / finished_at | TIMESTAMPTZ | |

### 5.4. Миграции

- Первая миграция — `0001_init.py` с полным набором таблиц выше.
- Последующие — атомарные, с оператором `CONCURRENTLY` для индексов на крупных таблицах.
- Запуск в `entrypoint.sh`: `alembic upgrade head` перед стартом `bot`/`worker`.
- Downgrade — писать всегда, проверять на стейдже (см. §17).

---

## 6. Ключевые сценарии (sequence)

### 6.1. Покупка подписки (§3 → §5 → §12 INTERFACE)

```
User ──▶ bot: tap "Месячный"
bot: upsert user, проверить бан, создать invoice(status=created, ttl=15m)
bot ──▶ User: меню оплаты (§5) + callback'и для каждого способа

User ──▶ bot: tap "СБП (QR)"
bot ──▶ PaymentRegistry.get("yookassa").create_invoice(invoice)
YooKassa ──▶ bot: {id, confirmation_url, ...}
bot: invoice.provider_invoice_id=..., status=pending
bot ──▶ User: сообщение со ссылкой на оплату + кнопкой "Проверить"

...
YooKassa ──▶ Caddy ──▶ bot: POST /pay/yookassa (webhook)
bot:
  1) verify signature/IP
  2) Redis SET payment:dedup:yookassa:<id> NX EX 86400 — если не NX, возвращаем 200 и выходим
  3) INSERT payments(...) ON CONFLICT (provider, provider_payment_id) DO NOTHING
  4) если реально вставили — SubscriptionService.fulfill(invoice)
       - создаём subscription(status=pending)
       - XUIClient.add_client() → vpn_keys(status=pending)
       - subscription.status=active, vpn_keys.status=active
       - outbox.enqueue("payment_success", {sub_id, vless_url})
       - referral_bonus: если первая активация — referrals.first_activation_at=now()
  5) bot ──▶ YooKassa: 200 OK

worker (outbox_flush):
  ──▶ Telegram: sendPhoto(key banner=None) + sendMessage(text, kb)
  ──▶ Telegram: sendMessage(<code>{vless_url}</code>)
```

Идемпотентность обеспечивается на 3 уровнях: Redis dedup, UNIQUE `(provider, provider_payment_id)`, UNIQUE `outbox.dedup_key`.

### 6.2. Триал 3 дня (§2.1 INTERFACE)

```
User: /start
bot: upsert user(trial_used=false), показать главное меню + доп. сообщение "3 дня VPN бесплатно"
User: tap "Активировать 3 дня"
bot: SubscriptionService.grant_trial(user)
     - проверяем trial_used — если true, сообщаем "триал уже активирован"
     - создаём tariff=TRIAL (duration=3, price=0, kind='vpn')
     - выдаём vpn-ключ (тот же саго-поток §4.8)
     - user.trial_used=true
     - outbox.enqueue("trial_activated")
```

### 6.3. Продление (§7 INTERFACE)

```
User: tap "Продлить" → выбор +30
bot: SubscriptionService.quote_extend(user, +30) → invoice(intent='extend')
User: оплачивает → webhook → payments → SubscriptionService.apply_extension(sub, +30)
     if sub.status == 'active':
         sub.expires_at += 30d
     else:
         sub.started_at = now()
         sub.expires_at = now() + 30d
         sub.status = 'active'
         XUIClient.update_client(expiryTime=new_ts)  # или addClient, если был удалён
     outbox.enqueue("extend_success")
```

### 6.4. Подарок (§8 INTERFACE)

```
User A: tap "Подарить другу" → "Месячная"
bot: FSM: WaitingRecipient
User A: пересылает сообщение B или @username_b
bot: UserService.resolve(@b) → User(B)
     - если не найден или trial_used=false и никогда не делал /start — ошибка §13
bot: подтверждение → invoice(intent='gift', target_user_id=B.id)
User A: оплачивает → webhook → SubscriptionService.fulfill_gift(invoice)
       - subscription создаётся на user_id=B, gifted_by_user_id=A
       - outbox.enqueue("gift_received", to=B)
       - outbox.enqueue("gift_sent_receipt", to=A)
```

### 6.5. Реферальная программа (§10 INTERFACE)

```
Новый User /start?ref=<referral_code>
bot: upsert, если referrer пуст — user.referrer_id = lookup(referral_code)
     INSERT referrals(referrer_id, referee_id)
     outbox.enqueue("referral_joined", to=referrer)

Позже User активирует любую подписку (покупка или триал) — первое активное состояние:
SubscriptionService.on_first_activation(user):
     if user.referrer_id:
         referrals[user].first_activation_at = now()
         outbox.enqueue("referral_activated", to=referrer)
         task enqueue referral_bonus.try_grant(referrer_id)

task referral_bonus.try_grant(referrer_id):
     # выполняется в одной транзакции + advisory lock на user.id
     uncounted = SELECT COUNT(*) FROM referrals
                 WHERE referrer_id=? AND first_activation_at IS NOT NULL
                   AND counted_for_bonus=false
     bonuses_to_grant = uncounted // 2
     for _ in range(bonuses_to_grant):
         - помечаем 2 строки counted_for_bonus=true
         - создаём/продлеваем subscription(source='referral_bonus', +1d)
         - user.referral_bonus_days_granted += 1
         - outbox.enqueue("referral_bonus", to=referrer)
```

### 6.6. Покупка прокси (§4 INTERFACE)

Отличие от VPN: провижининг идёт через `proxy-provisioner` (§9), результат — `host:port:login:password`, который кладётся в `proxy_accounts` и отправляется отдельным сообщением в `<code>…</code>`.

### 6.7. Рассылка (§14.3 INTERFACE)

```
Admin: выбирает "По фильтру" → фильтры → вводит текст → подтверждает
bot: INSERT broadcasts(status='draft')  → status='running'
bot: chunk user_ids по 500 и enqueue tasks.broadcast.send_chunk(broadcast_id, user_ids)
worker: на каждый chunk — кладёт сообщения в outbox с priority=200 (ниже обычного)
worker(outbox_flush): соблюдает 25 msg/s при рассылках (bucket `broadcast`)
worker: по завершении agg → broadcasts.sent_ok/sent_fail, status='done'
bot: пушит прогресс админу каждые N сообщений (SELECT COUNT)
```

### 6.8. Отзыв подписки админом (§14.5.3 INTERFACE)

```
Admin: tap "Отозвать подписку" → @username → подтверждение
bot: SubscriptionService.revoke(sub, by=admin)
     - sub.status='revoked', revoked_at, revoked_by
     - XUIClient.delete_client(xui_client_id) — если падает, enqueue xui_compensation.delete_client
     - vpn_keys.status='revoked'
     - audit_log
     - outbox.enqueue("subscription_revoked", to=user)
```

---

## 7. Payments: абстракция и провайдеры

### 7.1. Протокол

```python
# alpaca/payments/base.py

@dataclass(frozen=True)
class InvoiceRequest:
    invoice_id: UUID              # наш id, пойдёт в metadata
    amount_rub: Decimal | None
    amount_xtr: int | None
    currency: str                 # RUB | XTR | USDT
    description: str              # "Alpaca VPN — Месячная подписка"
    user_tg_id: int
    return_url: str | None        # для провайдеров с редиректом
    metadata: dict[str, str]

@dataclass(frozen=True)
class InvoiceRef:
    provider_invoice_id: str
    pay_url: str | None
    raw: dict                     # для БД

@dataclass(frozen=True)
class PaymentEvent:
    provider: str
    provider_payment_id: str
    invoice_id: UUID              # восстановлен из metadata
    amount_rub: Decimal | None
    amount_xtr: int | None
    currency: str
    status: Literal["succeeded","refunded","chargeback"]
    raw: dict

class PaymentProvider(Protocol):
    code: str                     # 'yookassa', 'cryptobot', 'stars', ...
    async def create_invoice(self, req: InvoiceRequest) -> InvoiceRef: ...
    async def handle_webhook(self, headers: Mapping[str,str], body: bytes) -> PaymentEvent | None: ...
    async def verify_secret(self, headers: Mapping[str,str], body: bytes) -> bool: ...
```

### 7.2. Особенности каждого провайдера

| Провайдер | Создание счёта | Верификация вебхука | currency |
|---|---|---|---|
| `stars` | `bot.send_invoice(chat_id, currency='XTR', prices=[LabeledPrice(...)], payload=invoice_id)` — счёт шлётся отдельным сообщением напрямую в чат | обрабатываем `pre_checkout_query` → answer OK; затем `message.successful_payment` — это и есть событие | `XTR` |
| `cryptobot` | HTTP `POST /api/createInvoice` с `payload=invoice_id` | вебхук с HMAC-SHA256 header `crypto-pay-api-signature`, проверяем своим токеном | `USDT` / `TON` / ... |
| `yookassa` | SDK `yookassa.Payment.create({..., metadata: {invoice_id}})` | вебхук без HMAC — проверяем IP allowlist YooKassa + сверку `payment.status=='succeeded'` через SDK | `RUB` |
| `lava` / `wata` | HTTP с HMAC или фирменным signing | HMAC в заголовке | `RUB` |

**Важный кейс — Telegram Stars:**
- нет отдельного HTTP webhook, это обычный апдейт через тот же `/tg/<secret>`;
- в `bot/handlers/payment.py` ловим `pre_checkout_query` и `message.successful_payment`;
- перекладываем в единый `PaymentEvent` и дальше — через ту же `SubscriptionService.fulfill`.

### 7.3. Регистрация провайдеров

```python
# alpaca/payments/registry.py
class PaymentRegistry:
    def __init__(self, providers: dict[str, PaymentProvider]): ...
    def get(self, code: str) -> PaymentProvider: ...
    def active(self) -> list[PaymentProvider]: ...

# в di.py
providers = {
    'stars': StarsProvider(bot=bot),
    'cryptobot': CryptoBotProvider(token=settings.cryptobot_token),
    'yookassa': YooKassaProvider(shop_id=..., secret_key=..., allowed_ips=[...]),
}
```

Показ кнопок в меню оплаты (§5 INTERFACE) фильтруется по `settings.enabled_providers: list[str]`.

### 7.4. Идемпотентность (реализация §4.3 STACK)

1. **Redis dedup**: `SET payment:dedup:{provider}:{provider_payment_id} NX EX 86400` перед парсингом события. Если ключ уже был — сразу 200 OK.
2. **DB unique**: `INSERT INTO payments (...) ON CONFLICT (provider, provider_payment_id) DO NOTHING RETURNING id`. Если не вернули строку — выходим.
3. **Outbox dedup_key**: `payment:{payment_id}:receipt` — повторная обработка не создаёт дубликатов уведомлений.

### 7.5. TTL инвойсов

Таска `tasks.invoice_ttl.sweep` раз в 30 с:
```sql
UPDATE invoices SET status='expired', updated_at=now()
 WHERE status IN ('created','pending') AND expires_at < now()
 RETURNING id, user_id;
```
По возвращённым id — `outbox.enqueue("invoice_expired", dedup_key="invoice:{id}:expired")`.

Провайдеры, которые допускают явный cancel (YooKassa), также получают `cancel_invoice` вызов в task (best-effort).

---

## 8. VPN-провижининг через 3x-ui

### 8.1. Ограничения панели

- Только cookie-сессия (подтверждено через deepwiki). Клиент входит `POST /login`, хранит cookie в памяти процесса.
- 401/404 → panel недоступна или сессия протухла. Клиент делает **одну** перелогин-попытку и повторяет запрос.
- API не транзакционное — отсюда saga-паттерн (§4.8 STACK).

### 8.2. Endpoints, которые используем

| Операция | HTTP | Назначение |
|---|---|---|
| login | `POST /login` (form-urlencoded) | получить session cookie |
| list inbounds | `GET /panel/api/inbounds/list` | валидация, сбор статистики |
| addClient | `POST /panel/api/inbounds/addClient` | создать клиента в нужном inbound |
| updateClient | `POST /panel/api/inbounds/updateClient/:clientId` | продление, смена limit |
| delClient | `POST /panel/api/inbounds/:id/delClient/:clientId` | отзыв / очистка |
| getClientTraffics | `GET /panel/api/inbounds/getClientTraffics/:email` | опционально, для будущей отчётности |

### 8.3. XUIClient

```python
class XUIClient:
    def __init__(self, base_url, username, password, http: httpx.AsyncClient, breaker: CircuitBreaker):
        self._cookie = None
        self._lock = asyncio.Lock()           # one in-flight login at a time

    async def _login(self): ...
    async def _request(self, method, url, **kw):
        async with self._breaker:
            if not self._cookie: await self._login()
            r = await self._http.request(method, url, cookies=self._cookie, **kw)
            if r.status_code in (401, 404) and <retried=False>:
                await self._login()
                r = await self._http.request(...)
            r.raise_for_status()
            return r.json()

    async def add_client(self, inbound_id: int, email: str, ttl_days: int) -> XUIClientInfo: ...
    async def update_client(self, client_id: str, *, expiry_ts: int): ...
    async def delete_client(self, inbound_id: int, client_id: str): ...
    async def build_vless_url(self, inbound: XUIInbound, client: XUIClientInfo) -> str: ...
```

**Circuit breaker** (pybreaker или свой): после 5 подряд ошибок — OPEN на 60с, в это время `add_client` сразу кидает `ProvisionError`, а SubscriptionService кладёт пользовательский retry в очередь `xui_compensation`.

### 8.4. Формат email клиента

`u{user_id}-s{subscription_id}` — позволяет быстро найти клиента в панели и по трафикам (`getClientTraffics/:email`).

### 8.5. Сага «оплата → ключ»

```
T0: subscription(status=pending) ← внутри одной DB-транзакции с payments INSERT
T1: try XUIClient.add_client
    success → vpn_keys(status=active), subscription.status=active, outbox.enqueue
    fail    → vpn_keys(status=failed, last_error)
              subscription.status=failed (оплата уже принята, отрабатывает компенсация)
              enqueue task xui_compensation.retry_provision(sub_id) с exp. backoff до 24ч
              outbox.enqueue("provision_pending", "ваш ключ скоро будет выдан")
T2: retry_provision(sub_id):
    если subscription.status != 'failed' — выходим
    try add_client → переводим в active, шлём ключ
    после 24ч fail → alert в Sentry, уведомление админу, пользователь получает уведомление
```

### 8.6. Быстрый инбаунд

`tariffs.xui_inbound_id` ссылается на конкретный inbound. Для «Быстрый 1 мес.» — отдельный inbound на ноде с QoS (настраивается вручную в 3x-ui). В коде ничего не ветвится — просто другой `inbound_id`.

---

## 9. Proxy-provisioner (SOCKS5/HTTP)

### 9.1. Варианты

| Вариант A (MVP, опция по умолчанию) | Вариант B (если нужна гео-география) |
|---|---|
| Локальный `3proxy` в отдельном контейнере + мини-API | Wrapper над Proxy6 / ProxyLine API |
| Полный контроль, нет комиссии провайдера | Быстрее выкатить гео; зависимость от внешнего API |

Архитектурно оба варианта — за одним интерфейсом `ProxyBackend`.

### 9.2. Интерфейс

```python
class ProxyBackend(Protocol):
    async def issue(self, *, kind: Literal['socks5','http'], ttl_days: int) -> IssuedProxy: ...
    async def revoke(self, external_id: str) -> None: ...
    async def list_expired(self) -> list[IssuedProxy]: ...
```

### 9.3. Локальный 3proxy-бэкенд

- `3proxy.cfg` генерируется из БД на каждый пул-рестарт (ни одного секрета в файле — только `$include /run/secrets/users.cfg`).
- Мини-FastAPI `proxy-provisioner` в том же контейнере:
  - `POST /accounts` → создаёт login/password (nanoid), пишет в `proxy_accounts`, перегенерирует `users.cfg`, `SIGHUP` 3proxy;
  - `DELETE /accounts/{id}` → удаляет, SIGHUP;
  - `GET /accounts?expired=true` для ARQ-cleanup.
- Бот ходит в provisioner по внутренней сети compose (`http://proxy-provisioner:8080`). Секрет — Bearer-токен.
- Паспорта / идентификация провайдера не нужны.

### 9.4. ARQ-таска `proxy_cleanup`

Раз в 10 минут: `SELECT * FROM proxy_accounts WHERE status='active' AND expires_at<now()` → revoke в бэкенде + `status='expired'` + уведомление.

---

## 10. Outbox и фоновый воркер (ARQ)

### 10.1. Зачем outbox

- Гарантии доставки при 429/5xx от Telegram.
- Глобальный rate-limit (30 msg/s), отдельный bucket для рассылок (25 msg/s).
- Транзакционная запись «в БД» вместе с бизнес-операцией (никаких «платёж прошёл, но уведомление потерялось»).
- Дедупликация (`dedup_key`).

### 10.2. Запись в outbox

Из сервисов — через репозиторий в той же DB-транзакции:

```python
await outbox_repo.enqueue(
    user_id=user.id,
    kind="text",
    payload={
        "text_key": "notify.payment_success",
        "text_args": {"key": key.vless_url},
        "reply_markup_key": "kb.after_purchase",
        "parse_mode": "HTML",
    },
    dedup_key=f"payment:{payment.id}:receipt",
    priority=10,
)
```

Рендеринг текста/клавиатуры происходит на стороне воркера (с учётом актуальной локали), чтобы изменение переводов не портило старые очереди.

### 10.3. Flusher

ARQ-cron `outbox_flush` каждую секунду:

```
1. SELECT ... FROM outbox
   WHERE status='pending' AND available_at<=now()
   ORDER BY priority ASC, id ASC
   LIMIT :batch (25–30)
   FOR UPDATE SKIP LOCKED;
2. Для каждой записи:
     - rate-limit: token bucket per bot (30/s) и per chat (1 msg/s, 20/min для обычных)
     - render payload (из i18n + renderer)
     - bot.send_* через aiogram
     - 200 → status='sent', sent_at=now()
     - TelegramRetryAfter(sec) → available_at=now()+sec, attempts+=1
     - TelegramForbidden (bot blocked) → status='failed', last_error
     - прочее → attempts+=1, backoff = min(60*2**attempts, 3600)
     - attempts>=max_attempts → 'failed'
3. COMMIT (lock освобождается).
```

Rate-limit — в Redis через Lua-скрипт `INCR + EXPIRE`, ключи `rl:bot:sec:{t}`, `rl:chat:{id}:sec:{t}`.

### 10.4. ARQ воркер — расписание

| Таска | Расписание / триггер | Назначение |
|---|---|---|
| `outbox_flush` | `cron *:*:* ` (каждую секунду) | доставка |
| `invoice_ttl_sweep` | каждые 30 с | инвалидация инвойсов |
| `expiry_reminders_t3` | каждые 5 мин | напоминания «за 3 дня» |
| `expiry_reminders_t0` | каждые 5 мин | «подписка истекла» + статус sub → expired |
| `proxy_cleanup` | каждые 10 мин | |
| `xui_compensation_retry` | on-demand + каждые 2 мин sweep | |
| `referral_bonus_try_grant` | on-demand | |
| `broadcast_send_chunk` | on-demand | массовая рассылка |
| `pg_backup_trigger` | `cron 03:00` | запуск `scripts/backup.sh` |

### 10.5. Защита от дублей в ARQ

Все задачи принимают `job_id` = детерминированный хеш аргументов. `job_try` → backoff. `max_tries=5`.

---

## 11. FSM, роутеры и middleware

### 11.1. Структура Dispatcher

```
Dispatcher
├── StorageRedis (FSM)
├── update_middleware
│    ├── LoggingMiddleware (bind request_id)
│    ├── DBSessionMiddleware  (open AsyncSession)
│    ├── UserMiddleware        (upsert User, last_seen_at, inject `user`)
│    ├── BanCheckMiddleware    (§4.6 STACK)
│    ├── I18nMiddleware        (locale из user.language_code)
│    └── ThrottleMiddleware    (Redis: 1 msg/user/sec, soft)
│
├── Router("common")         — /start (с обработкой ref-кода), /help, "Назад"
├── Router("main_menu")      — все корневые кнопки
├── Router("trial")
├── Router("buy_vpn")
├── Router("buy_proxy")
├── Router("payment")        — + pre_checkout_query + successful_payment
├── Router("subscriptions")
├── Router("extend")
├── Router("gift")           — FSM WaitingRecipient
├── Router("instruction")
├── Router("partner")
├── Router("help")
│
└── Router("admin")          — include только если IsAdmin
     ├── Router("admin_stats")
     ├── Router("admin_broadcast") — FSM WaitingText, WaitingBanner
     ├── Router("admin_users")     — FSM SearchQuery
     ├── Router("admin_subs")
     ├── Router("admin_finance")
     └── Router("admin_settings")  — FSM WaitingPrice, WaitingBanner, WaitingAdmin
```

### 11.2. BanCheckMiddleware

```python
async def __call__(self, handler, event, data):
    user = data.get("user")       # уже после UserMiddleware
    if user and user.is_banned:
        await answer(event, texts["common.banned"])
        return                    # не вызываем handler
    return await handler(event, data)
```

`is_banned` кэшируется в Redis (`user:ban:{tg_id}` TTL 60с), инвалидируется админской командой бан/разбан.

### 11.3. CallbackData

Все callback-кнопки — через `aiogram.filters.callback_data.CallbackData`:

```python
class BuyCB(CallbackData, prefix="buy"): tariff_code: str
class PayCB(CallbackData, prefix="pay"): invoice_id: UUID; provider: str
class ExtendCB(CallbackData, prefix="ext"): days: int
class GiftCB(CallbackData, prefix="gift"): tariff_code: str
class AdminCB(CallbackData, prefix="adm"): action: str; target: str | None = None
```

### 11.4. FSM-состояния

Все `StatesGroup` в одном файле `bot/states.py` для обзора. Примеры:

```python
class GiftFlow(StatesGroup):
    choosing_tariff = State()
    waiting_recipient = State()
    confirming = State()

class AdminBroadcast(StatesGroup):
    choosing_audience = State()
    choosing_filters = State()
    waiting_banner = State()
    waiting_text = State()
    preview = State()

class AdminChangePrice(StatesGroup):
    choosing_tariff = State()
    waiting_new_price = State()
    confirming = State()

class AdminSearchUser(StatesGroup):
    waiting_query = State()
```

### 11.5. Renderer

Единый слой «доменная модель → `(text, InlineKeyboardMarkup, photo_file_id)`»:

```python
class MenuRenderer:
    def __init__(self, i18n: I18n, banners: BannerCache, tz): ...
    def main_menu(self, user) -> RenderedScreen: ...
    def tariffs(self, tariffs: list[Tariff]) -> RenderedScreen: ...
    def subscriptions(self, subs: list[Subscription]) -> RenderedScreen: ...
    ...
```

Это удобно для:
- переиспользования из outbox-воркера;
- unit-тестов (renderer тестируется без aiogram).

---

## 12. Админ-панель

Маппинг INTERFACE.md §14 → модули:

| Раздел UI | Handler | Service | Доп. сущности |
|---|---|---|---|
| 14.2 Статистика | `bot/handlers/admin/stats.py` | `services/stats_service.py` (materialized-view или просто SELECT'ы) | экспорт CSV/JSON — stream в `InputFile` |
| 14.3 Рассылка | `admin/broadcast.py` | `broadcast_service` | таблица `broadcasts`, ARQ `broadcast_send_chunk` |
| 14.4 Пользователи | `admin/users.py` | `admin_service` + `users_repo` | `user_bans` |
| 14.5 Подписки | `admin/subs.py` | `subscription_service` | audit_log |
| 14.6 Финансы | `admin/finance.py` | `stats_service.financials()` | |
| 14.7.1 Цены | `admin/settings.py` | `admin_service.change_price` | audit_log |
| 14.7.2 Баннеры | `admin/settings.py` | `admin_service.upload_banner` | `banners` + сброс `BannerCache` |
| 14.7.3 Админы | `admin/settings.py` | `admin_service.add/remove_admin` | `users.is_admin` |
| 14.7.4 Перезапуск | `admin/settings.py` | — | `kill -HUP` или SIGTERM от supervisor'а контейнера |
| 14.7.5 Логи | `admin/settings.py` | — | `tail -n 100 /var/log/alpaca/app.jsonl`, отдаём как `InputFile` |

**Фильтр `IsAdmin`** использует `users.is_admin` (источник правды — БД; список из env читается один раз при старте и применяется к БД). Так можно добавлять админов через UI без рестарта.

**Статистика** — для 5 000 пользователей достаточно обычных SELECT с индексами; materialized view добавляется позже, если потребуется.

**Экспорт CSV/JSON** — стримом через `asyncpg.Cursor` и `aiogram.types.BufferedInputFile`.

---

## 13. Конфигурация, секреты, i18n

### 13.1. pydantic-settings

```python
# alpaca/config.py
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_nested_delimiter="__")

    env: Literal["dev","staging","prod"] = "prod"
    timezone: str = "Europe/Moscow"

    telegram: TelegramCfg
    postgres: PostgresCfg
    redis: RedisCfg
    arq: ArqCfg
    xui: XUICfg
    proxy_provisioner: ProxyCfg
    payments: PaymentsCfg           # enabled_providers, ключи YooKassa/CryptoBot/...
    sentry: SentryCfg
    admins: AdminsCfg               # initial_admin_ids, для сида
    limits: LimitsCfg               # rate limits, invoice ttl, trial days
    i18n: I18nCfg                   # default_locale, available
```

`.env.example` полностью покрывает список переменных, группы разделены по префиксам.

### 13.2. Секреты

- Ни одного секрета в репозитории.
- На VPS — `/home/alpaca/.env` с `chmod 600`.
- Внутри compose — `env_file: .env` на сервисы `bot` и `worker`.
- Fernet-ключ для шифрования `proxy_accounts.password` — отдельная env `PROXY_PASSWORD_KEY`. Ротация — в `scripts/rotate_password_key.py` (не в MVP, но задел заложен).
- Webhook-секрет Telegram (`telegram.webhook_secret`) проверяется middleware через `X-Telegram-Bot-Api-Secret-Token` (см. §2 и §4.1 STACK).

### 13.3. i18n

- Файлы: `locales/<lang>.yaml`, плоские ключи `notify.payment_success`, `btn.buy`, ...
- Во время рендера — `i18n.t(key, locale=user.locale, **args)`.
- В коде нет ни одной строки UI — только ключи.
- Сид MVP: только `ru.yaml`. Добавление `en.yaml` — чистая трудозадача без изменений кода.
- Баннерные изображения — отдельно от i18n (не переводим).

---

## 14. Деплой: Docker Compose + Caddy

### 14.1. docker-compose.yml (упрощённо)

```yaml
services:
  caddy:
    image: caddy:2
    restart: unless-stopped
    ports: ["80:80","443:443"]
    volumes:
      - ./docker/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on: [bot]

  bot:
    build: .
    command: ["python","-m","alpaca","bot"]
    restart: unless-stopped
    env_file: [.env]
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
    volumes:
      - banners:/app/banners
      - logs:/var/log/alpaca

  worker:
    build: .
    command: ["python","-m","alpaca","worker"]
    restart: unless-stopped
    env_file: [.env]
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
    volumes:
      - logs:/var/log/alpaca

  postgres:
    image: postgres:16
    restart: unless-stopped
    env_file: [.env.postgres]
    volumes: ["postgres_data:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD","pg_isready","-U","$${POSTGRES_USER}"]
      interval: 5s

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server","--appendonly","yes","--save","60","1"]
    volumes: ["redis_data:/data"]
    healthcheck:
      test: ["CMD","redis-cli","ping"]
      interval: 5s

  proxy-provisioner:
    build: ./docker/proxy_provisioner
    restart: unless-stopped
    env_file: [.env.proxy]
    ports: []            # только внутренняя сеть
    cap_add: [NET_ADMIN] # для 3proxy
    volumes: ["proxy_data:/var/lib/3proxy"]

  uptime-kuma:
    image: louislam/uptime-kuma:1
    restart: unless-stopped
    volumes: ["uptime_data:/app/data"]

volumes: { postgres_data: {}, redis_data: {}, caddy_data: {}, caddy_config: {},
           banners: {}, logs: {}, proxy_data: {}, uptime_data: {} }
```

### 14.2. Caddyfile

```
{$DOMAIN} {
    encode zstd gzip

    handle /healthz {
        respond "ok" 200
    }

    # Telegram webhook: путь включает секрет
    handle /tg/{$TG_WEBHOOK_SECRET_PATH} {
        reverse_proxy bot:8080
    }

    # Платёжные вебхуки
    handle /pay/yookassa  { reverse_proxy bot:8080 }
    handle /pay/cryptobot { reverse_proxy bot:8080 }
    handle /pay/lava      { reverse_proxy bot:8080 }
    handle /pay/wata      { reverse_proxy bot:8080 }

    # Всё остальное — 404 (закрыто для публики)
    handle { respond 404 }
}
```

### 14.3. Процесс релиза

1. `git pull` → `docker compose build`;
2. `docker compose run --rm bot alembic upgrade head`;
3. `docker compose up -d bot worker`;
4. `scripts/set_webhook.py` (один раз после смены `TG_BOT_TOKEN` или URL);
5. smoke-тест `/start` от тестового аккаунта.

### 14.4. Бэкапы

- Cron на хосте (или ARQ cron): `pg_dump -Fc` → upload в B2/S3 по `rclone`;
- Retention 14 дней;
- Раз в неделю — проверка `pg_restore --list` на пригодность;
- Проверяется одним job'ом в ARQ, алерт в Sentry при фэйле.

---

## 15. Наблюдаемость и алертинг

### 15.1. Логи

- **structlog**, JSON-строки в stdout, volume `logs` для `admin/settings` → «Логи».
- Контекст: `request_id`, `user_id`, `tg_id`, `handler`, `route`, `provider` — привязываются middleware.
- Уровни: `INFO` — все бизнес-события, `WARNING` — ретраи, `ERROR` — валится в Sentry.
- PII: `username` и `first_name` пишутся, тело сообщений пользователя — **не пишем** в лог.

### 15.2. Sentry

- `sentry-sdk[aiohttp]` + `integrations.sqlalchemy, redis, arq (custom)`.
- release = git sha, environment = `env`;
- before_send хук вычищает `payload.text`, `vless_url`, `proxy.password`, номера карт.

### 15.3. Health

- `bot`: `/healthz` → проверяет DB ping + Redis ping + 3x-ui (кэшированный результат, не чаще 1 раз/мин);
- `worker`: endpoint не нужен, но кладёт в Redis `worker:heartbeat` каждые 30 с — Uptime Kuma проверяет возраст ключа через мини-HTTP в bot (`/healthz/worker`).

### 15.4. Метрики (опционально, но задел)

- `prometheus_client` в bot/worker на порту `9100`/`9101`;
- метрики: `outbox_pending`, `outbox_send_latency_seconds`, `payment_events_total{provider,status}`, `xui_errors_total`, `ban_checks_total`, `broadcasts_in_progress`;
- скрейпим позже, если понадобится Grafana.

### 15.5. Алерты

| Триггер | Канал |
|---|---|
| `outbox_pending > 1000` | Sentry issue + Uptime Kuma |
| 3x-ui недоступна > 10 мин | Sentry + Telegram-уведомление админу |
| Падение бэкапа | Sentry |
| CircuitBreaker OPEN > 5 мин | Sentry |
| Платёжный вебхук не принят 5xx | Sentry (каждый случай) |

---

## 16. Безопасность

1. **Telegram webhook**: путь с секретом + `X-Telegram-Bot-Api-Secret-Token` (подтверждено deepwiki'ем для aiogram 3).
2. **YooKassa webhook**: IP allowlist + сверка через SDK (`Payment.find_one`) перед выдачей ключа.
3. **CryptoBot webhook**: HMAC-SHA256 по токену.
4. **Админ-доступ**: `users.is_admin`, первый список — из env. Невозможно повысить себя UI.
5. **PII**: пароли прокси — Fernet (симметричный AES-128), ключ — env-переменная, не в коде.
6. **SQL-инъекции**: исключены — только SQLAlchemy ORM/Core с параметризацией.
7. **XSS в HTML-парсинге**: экранирование через `html.escape` в renderer для любых user-generated строк (никогда не кладём сырые `first_name`/`username` в `<code>` без escape).
8. **Rate-limit на входе**: `ThrottleMiddleware` (Redis) — 1 апдейт/user/sec soft, hard-блокировка при флуде (15 апдейтов за 10 сек → 429 + лог).
9. **Секреты в контейнере**: `env_file`, а не `environment:` (не светятся в `docker inspect` без `-f`).
10. **Upgrade path**: автоматический `alembic upgrade head` при старте. Не добавляем breaking-миграции без двухфазного релиза.

---

## 17. Тестирование, миграции, CI/CD

### 17.1. Пирамида тестов

| Уровень | Тулинг | Что покрываем |
|---|---|---|
| Unit | `pytest`, `pytest-asyncio` | `domain/*`, `pricing.py`, `referral.py`, renderer, провайдеры платежей (моки) |
| Integration | `testcontainers-python` (pg, redis) | репозитории, сервисы, outbox flusher, xui_client с `respx` моком, yookassa с `respx` |
| E2E | `aiogram` `Dispatcher.feed_update` + реальный Postgres + mock Telegram | сценарии §6 — от апдейта до outbox |
| Контрактные | `respx` + snapshot | формы запросов к 3x-ui и YooKassa |

Отдельно — `alembic upgrade head && alembic downgrade base && upgrade head` в CI на каждом PR.

### 17.2. Pre-commit

`.pre-commit-config.yaml`:
```
ruff (lint + format)
mypy (strict, кроме миграций)
codespell
yamllint
prettier (для Caddyfile через плагин или кастомный hook)
```

### 17.3. CI/CD

GitHub Actions (если у заказчика GitHub):

- `ci.yml`:
  1. `uv sync`
  2. `pre-commit run --all`
  3. `pytest -q --cov`
  4. `alembic up/down smoke` в ephemeral pg
  5. build Docker image, push в GHCR (tag = sha + branch)
- `deploy.yml` (по тегу `v*`):
  - SSH на VPS → `docker compose pull && alembic upgrade head && up -d`;
  - smoke `/healthz`;
  - rollback-сценарий — держим предыдущий image tag под `alpaca:previous`.

Если заказчик на GitLab — те же шаги в `.gitlab-ci.yml` (оставлено за рамками MVP, отмечено как out of scope в STACK §6).

---

## 18. Открытые вопросы (для заказчика)

Повторяют §5 STACK и §0 INTERFACE, с архитектурными рекомендациями:

| # | Вопрос | Рекомендация архитектора |
|---|---|---|
| 1 | Прокси: свой 3proxy vs API Proxy6/ProxyLine | **MVP — свой `proxy-provisioner` (3proxy)**. Интерфейс `ProxyBackend` сохранит возможность добавить внешний провайдер. |
| 2 | Где живёт 3x-ui | **Внутри `docker-compose` на том же VPS**, если заказчик не против; так проще бэкапить и сетевые политики. |
| 3 | YooKassa vs Lava/Wata | **Начать с YooKassa**, резерв Lava/Wata через абстракцию §7. |
| 4 | Мультиязычность | **Закладываем i18n-ключи всегда.** Фактический `en.yaml` — не в MVP, добавляется без изменений кода. |
| 5 | TTL инвойса | **15 мин подтверждён**, настройка в `settings.limits.invoice_ttl_minutes`. |
| 6 | Баннеры (формат/хранение) | **PNG 1280×720, максимум 512 КБ**. Хранение — volume `banners/`, кэш `telegram_file_id` в `banners`. Админ загружает через бот — он сам кладёт в volume и обновляет запись. |
| 7 | «Безопасно» в главном меню | **Статично** в MVP — рассчитывать текущий статус без реальной трафик-метрики нечестно. Задел — поле `health_status` вычисляем из 3x-ui пингов. |
| 8 | Быстрый тариф — описание | **Подставляется из `tariffs.description_key`** — текст редактируется в i18n без релиза. |
| 9 | Возврат при отзыве подписки | **По умолчанию без возврата**, решение админа — ручное. В UI §14.5.3 уже проговорено. |

---

## 19. Дорожная карта реализации

Порядок спринтов для команды разработки (1 исполнитель, 2-недельные спринты):

### Спринт 1 — каркас (5–7 дней)
- структура пакетов §3, `pyproject.toml`, `pre-commit`, Dockerfile, compose;
- `config.py`, `logging.py`, `di.py`, `db/session.py`, `alembic` init;
- миграция `0001_init` со всеми таблицами из §5;
- seed тарифов и баннеров из дефолтов;
- базовый bot с `/start` и ответом «hello» через webhook;
- Caddy + secret webhook; health-check; Sentry подключён.

### Спринт 2 — домен + провижининг (7–10 дней)
- `domain/*` с юнит-тестами;
- `XUIClient` + e2e против реальной тестовой панели;
- `SubscriptionService` (purchase happy path);
- outbox + `outbox_flush`;
- renderer + главное меню, «Мои подписки», «Инструкция», «Помощь», «Партнёрам» (статика + инфо).

### Спринт 3 — платежи (7–10 дней)
- `PaymentProvider` + регистрация;
- Stars (pre_checkout + successful_payment), CryptoBot, YooKassa;
- идемпотентность (3 уровня);
- TTL инвойсов, expired-уведомления;
- e2e тест «оплата → ключ → сообщение».

### Спринт 4 — продление / подарок / триал / рефералка (7 дней)
- `ExtendService`, `GiftService`, `TrialService`, `ReferralService` + их FSM-flow;
- уведомления §12 из INTERFACE.md полностью;
- таски `expiry_reminders`, `referral_bonus_try_grant`.

### Спринт 5 — прокси (5 дней)
- `proxy-provisioner` контейнер, `ProxyBackend` с 3proxy;
- `ProxyService` и UI §4;
- `proxy_cleanup`.

### Спринт 6 — админка (10 дней)
- Статистика, Финансы (селекты и рендер);
- Рассылки (FSM + ARQ chunks + прогресс);
- Пользователи (бан/разбан, поиск);
- Подписки (выдать/продлить/отозвать, списки);
- Настройки (цены, баннеры, админы, логи, restart).

### Спринт 7 — стабилизация (5 дней)
- нагрузочный тест (locust или k6) — 50 RPS на вебхук;
- «хаос-режим» (3x-ui down, Redis down) — проверка саги и outbox;
- завершение документации (README, runbook в `docs/`);
- переход на prod-VPS, бэкапы, Uptime Kuma.

**Итого MVP — ~7 недель** работы одного разработчика, исходя из 5 продуктивных часов/день.

---

## Резюме для разработчика

Вся сложность проекта сосредоточена в **4 местах**:
1. **Идемпотентность платежей** — §7.4 (Redis dedup + DB unique + outbox dedup_key).
2. **Сага «оплата → 3x-ui → пользователь»** — §8.5 с компенсацией.
3. **Outbox-доставка** — §10 (rate-limit + ретраи + dedup).
4. **Админка с горячим редактированием** — §5 (tariffs, banners в БД) + §12.

Всё остальное — механика поверх устойчивого каркаса, изложенного в §3–§4. Если эти четыре места разобраны и покрыты тестами — остаётся только UI-работа по §11/§12 INTERFACE.md.
