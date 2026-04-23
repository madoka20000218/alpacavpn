# Alpaca VPN — План реализации

> Поэтапный план разработки. Основан на `ARCHITECTURE.md`, `STACK.md` и `INTERFACE.md`.
> Исходная точка: пустой репозиторий (`LICENSE` + `README.md`).
> Цель: MVP, полностью покрывающий §2–§14 INTERFACE, с работой в проде на 1 VPS под 5 000 пользователей.
>
> **Формат плана:** `Фаза → Задача (T-xx) → Подзадачи → Критерии приёмки → Артефакты → Команды`. Оценка времени — в «идеальных часах» одного разработчика (×1.5 для календарных). Итого MVP ≈ **245 ч идеальных ≈ 7 календарных недель** при 5 продуктивных часах в день.

---

## Оглавление

- [Легенда](#легенда)
- [Фаза 0 — подготовка окружения и инфраструктуры](#фаза-0--подготовка-окружения-и-инфраструктуры-12-ч)
- [Фаза 1 — каркас и БД](#фаза-1--каркас-и-бд-26-ч)
- [Фаза 2 — домен, 3x-ui, outbox, базовый UI](#фаза-2--домен-3x-ui-outbox-базовый-ui-42-ч)
- [Фаза 3 — платежи и покупка подписки](#фаза-3--платежи-и-покупка-подписки-40-ч)
- [Фаза 4 — продление, подарок, триал, рефералка](#фаза-4--продление-подарок-триал-рефералка-36-ч)
- [Фаза 5 — прокси](#фаза-5--прокси-22-ч)
- [Фаза 6 — админка](#фаза-6--админка-44-ч)
- [Фаза 7 — стабилизация и прод](#фаза-7--стабилизация-и-прод-23-ч)
- [Правила работы в репозитории](#правила-работы-в-репозитории)
- [Риски и их митигация](#риски-и-их-митигация)
- [Чек-лист готовности к релизу MVP](#чек-лист-готовности-к-релизу-mvp)

---

## Легенда

- **T-XX** — сквозная нумерация задач для ссылок в PR/issue.
- **DoD** (Definition of Done) — критерии приёмки задачи.
- **Art** — артефакты (файлы, команды, таблицы, ручки).
- **Dep** — зависимости от других задач.
- **Est** — оценка в идеальных часах.
- **PR** — каждый T-XX = один PR. Merge только после прохождения CI и ревью (self-review на MVP).

Общие правила качества для всех задач:
- `ruff check .`, `ruff format --check .`, `mypy alpaca` — green;
- тесты, затронутые задачей, — green;
- PR-шаблон: «что/зачем/как тестировал/риски»;
- никаких секретов в коде и коммитах;
- каждый merge обновляет `CHANGELOG.md` (keep-a-changelog).

---

## Фаза 0 — подготовка окружения и инфраструктуры (12 ч)

### T-01. Выбор менеджера зависимостей и инициализация (1 ч)
- **Решение:** `uv` (быстро, детерминированно, умеет lockfile), таргет Python 3.12.
- **Шаги:**
  1. `uv init alpaca --package` (в пустом репо);
  2. `pyproject.toml` — минимальный скелет с `project`, `tool.ruff`, `tool.mypy`, `tool.pytest.ini_options`.
- **DoD:** `uv sync && uv run python -c "print('ok')"` работает.
- **Art:** `pyproject.toml`, `uv.lock`, `.python-version`.

### T-02. Lint / type / format инструменты (1 ч)
- `ruff` (lint + format), `mypy --strict` для `alpaca/*`, `codespell`, `yamllint`.
- **pre-commit:** устанавливаем сразу, чтобы хуки работали с первого коммита.
- **DoD:** `pre-commit run --all-files` на пустом проекте проходит; в CI-скрипте есть шаг `pre-commit run --all-files --show-diff-on-failure`.
- **Art:** `.pre-commit-config.yaml`, `.editorconfig`, `.gitattributes`.

### T-03. Репозиторный README и AGENTS.md (1 ч)
- Минимальный README: описание проекта, как запустить `docker compose up`, ссылка на `ARCHITECTURE.md`.
- `AGENTS.md` (для будущих Devin-сессий и разработчиков): команды lint/test/run, конвенции, структура.
- **DoD:** README пошагово воспроизводим; AGENTS.md перечисляет все ключевые команды.

### T-04. Базовый Dockerfile и docker-compose каркас (3 ч)
- Multi-stage `Dockerfile` (`python:3.12-slim` → builder → runtime), non-root user, `uv`-stage для lockfile.
- `docker-compose.yml` — сервисы `postgres`, `redis`, `bot`, `worker`, `caddy`, `proxy-provisioner` (пока stub), `uptime-kuma`.
- **DoD:** `docker compose build` проходит; `docker compose up postgres redis` поднимает работающие контейнеры с healthcheck.
- **Art:** `docker/Dockerfile`, `docker/entrypoint.sh`, `docker-compose.yml`, `docker/Caddyfile`.

### T-05. `.env.example` + конфиг pydantic-settings (2 ч)
- Все переменные из §13 архитектуры, сгруппированные префиксами (`TG_`, `PG_`, `REDIS_`, `XUI_`, `PAY_YOOKASSA_`, …).
- `alpaca/config.py` с вложенными dataclass'ами `TelegramCfg`, `PostgresCfg`, …, валидаторами.
- **DoD:** `uv run python -c "from alpaca.config import Settings; Settings()"` грузит `.env.example`.
- **Art:** `.env.example`, `alpaca/config.py`, `tests/unit/test_config.py` (10 кейсов: дефолты, required, type coercion).

### T-06. Logging + Sentry init (1.5 ч)
- `alpaca/logging.py`: structlog JSON-логи, processors (request_id, user_id), уровень из env.
- Sentry: `sentry_sdk.init(dsn, environment, release, before_send=scrub_pii)`.
- **DoD:** в stdout приходит JSON-строка с `event`, `level`, `request_id`; тестовое `1/0` → в Sentry (если DSN задан) попадает без PII.
- **Art:** `alpaca/logging.py`, unit-тест фильтра PII.

### T-07. CI (GitHub Actions) — минимум (2.5 ч)
- `.github/workflows/ci.yml`:
  - `lint` job: `uv sync --frozen` + `pre-commit run --all-files`;
  - `test` job: `uv run pytest -q --cov=alpaca --cov-fail-under=70`;
  - `migrations` job: поднимаем временный Postgres в services, выполняем `alembic upgrade head && downgrade base && upgrade head`;
  - `build` job: `docker build` без push (push добавим в T-62).
- **DoD:** CI green на пустом проекте (до первой миграции `migrations` job помечаем `if: hashFiles('alembic/versions/*.py') != ''`).
- **Art:** `.github/workflows/ci.yml`, `.github/PULL_REQUEST_TEMPLATE.md`.

**Фаза 0 DoD:** репо собирается, линты/тесты/ci зелёные, docker-compose поднимает `postgres` + `redis`, в проекте нет кода сервиса (это нормально — готов каркас).

---

## Фаза 1 — каркас и БД (26 ч)

### T-10. Структура пакетов `alpaca/*` (1 ч)
- Создаём пустые `__init__.py` для всех подпакетов из §3 архитектуры;
- `alpaca/__main__.py`: парсит аргумент `bot`|`worker` и вызывает соответствующую точку входа (пока stub'ы).
- **DoD:** `python -m alpaca bot --help` не падает.

### T-11. DB engine / session / UoW (2 ч)
- `alpaca/db/session.py`: `create_async_engine(dsn, pool_size=10, max_overflow=20)`, `async_sessionmaker(expire_on_commit=False)`.
- `alpaca/db/base.py`: `DeclarativeBase` с `MetaData(naming_convention={...})`.
- **DoD:** unit-тест подключается к Postgres из testcontainers и выполняет `SELECT 1`.
- **Art:** `tests/integration/test_db_session.py`.

### T-12. Alembic init + базовые миграции (1.5 ч)
- `alembic init alembic`, правим `env.py` под async + `target_metadata = Base.metadata`, берём DSN из `Settings`.
- **DoD:** `alembic upgrade head` / `downgrade base` работают на пустой схеме.

### T-13. Модели и миграция `0001_init` — справочники (3 ч)
- Таблицы: `users`, `user_bans`, `tariffs`, `banners`, `admin_users` (если выберем эту модель), `audit_log`.
- Индексы по §5.3 архитектуры.
- Seed: `scripts/seed.py` с тарифами из §3/§7 INTERFACE и баннерными записями (без файлов — заполним в T-20).
- **DoD:** `alembic upgrade head` создаёт таблицы; `python scripts/seed.py` кладёт ровно 6 VPN-тарифов + 1 «быстрый» + 4 прокси.
- **Art:** `alembic/versions/0001_init.py`, `alpaca/db/models/*.py`, `scripts/seed.py`, `tests/integration/test_seed.py`.

### T-14. Модели и миграция `0002_domain` — бизнес-сущности (3 ч)
- Таблицы: `subscriptions`, `vpn_keys`, `proxy_accounts`, `invoices`, `payments`, `referrals`, `outbox`, `broadcasts`.
- Все constraints, UNIQUE, PARTIAL INDEX'ы из §5.
- **DoD:** `alembic up/down` чистый, в тесте вставляем по 1 записи в каждую таблицу и проверяем FK.
- **Art:** миграция + `tests/integration/test_schema.py`.

### T-15. Репозитории — тонкий слой доступа (4 ч)
- `alpaca/db/repositories/*.py` — по репозиторию на агрегат: `users.py`, `subscriptions.py`, `invoices.py`, `payments.py`, `outbox.py`, `tariffs.py`, `banners.py`, `referrals.py`, `proxy_accounts.py`, `vpn_keys.py`.
- Методы минимальны: `get_by_id`, `upsert`, `list_by(...)`, специфичные селектора для задач (напр. `list_active_expiring_in(days)`).
- **DoD:** unit-тесты на ключевые методы (`upsert_user`, `enqueue_outbox` с `dedup_key`, `find_invoice_by_provider_id`).
- **Art:** `tests/integration/test_repositories.py` — 30+ кейсов.

### T-16. Утилиты времени и ID (1 ч)
- `alpaca/utils/time.py`: функции `utcnow()`, `to_moscow(dt)`, `format_day_remaining(dt)`.
- `alpaca/utils/ids.py`: `ulid_new()`, `uuid7_new()` (для UUID-PK в `invoices`/`payments`), `nanoid_referral()` (8 симв.).
- **DoD:** unit-тесты, включая DST-кейс (2024-03-31 02:30 Europe/Moscow).

### T-17. Aiohttp-приложение и aiogram Dispatcher — skeleton (3 ч)
- `alpaca/bot.py`: `build_app(settings)` — создаёт `Bot`, `Dispatcher`, `RedisStorage`, регистрирует роуты `SimpleRequestHandler(path=/tg/{secret})` + заглушки платёжных вебхуков (`/pay/<provider>`).
- `alpaca/__main__.py`: `asyncio.run(web._run_app(build_app(settings)))`.
- **DoD:** `docker compose up bot` поднимает webhook endpoint; `curl /healthz` → 200.
- **Art:** `alpaca/bot.py`, `alpaca/bot/webhook.py`, smoke-тест.

### T-18. Middlewares: логирование + DB-session (3 ч)
- `LoggingMiddleware` (bind request_id, handler), `DBSessionMiddleware` (session per update, commit/rollback).
- Unit-тесты на middleware через `Dispatcher.feed_update`.
- **DoD:** в логе при апдейте есть `request_id`, `user_id` = None; сессия закрывается всегда (тест на raise в handler).

### T-19. ARQ worker — skeleton (2 ч)
- `alpaca/worker.py`: `WorkerSettings` с пустым `functions=[]` и `cron_jobs=[]`, `on_startup`/`on_shutdown` собирают DI-ресурсы.
- **DoD:** `docker compose up worker` не падает и принимает shutdown по SIGTERM.

### T-20. Webhook-скрипты (2.5 ч)
- `scripts/set_webhook.py`: ставит webhook с `secret_token` и allowed_updates, печатает ответ Bot API.
- `scripts/delete_webhook.py`.
- **DoD:** локально (при реальном `TG_BOT_TOKEN`) скрипт ставит webhook и возвращает `ok: true`. На CI — dry-run (mock).

**Фаза 1 DoD:** поднятый compose даёт бот, который принимает `/start` (пустой ответ), записывает апдейт в лог, сохраняет сессию в БД через middleware; миграции актуальны; репозитории покрыты тестами.

---

## Фаза 2 — домен, 3x-ui, outbox, базовый UI (42 ч)

### T-21. Доменные сущности и политики (3 ч)
- `alpaca/domain/entities.py`: `SubscriptionPlan`, `ProxyPlan`, `Money`, `Period`, enum'ы статусов.
- `alpaca/domain/pricing.py`: функции, считающие цену покупки/продления/подарка с учётом зачёркнутой цены.
- `alpaca/domain/policies.py`: `can_gift(plan)`, `is_trial_allowed(user)`, `can_extend(user, plan)` + правила §7 INTERFACE о продлении active vs expired.
- `alpaca/domain/referral.py`: функция `calculate_bonus_days(uncounted_activations) -> (bonus_days, to_mark)`.
- **DoD:** 100% unit-покрытие, 30+ кейсов. Без импорта SQLAlchemy.

### T-22. i18n-инфраструктура (2 ч)
- `locales/ru.yaml` — черновик со всеми ключами из §2–§14 INTERFACE (текст/кнопки/уведомления/ошибки).
- `alpaca/bot/texts.py`: `I18n.load(dir)`, `t(key, locale, **kwargs)`.
- `I18nMiddleware`: читает `user.language_code`, кладёт в `data["t"]`.
- **DoD:** unit-тест: все ключи, используемые в коде, существуют в `ru.yaml` (скрипт `tests/unit/test_i18n_keys.py` парсит AST и сверяет).

### T-23. Клавиатуры и CallbackData (3 ч)
- `alpaca/bot/filters/callback_data.py`: все `CallbackData` из §11.3 архитектуры.
- `alpaca/bot/keyboards/*.py`: функции, возвращающие `InlineKeyboardMarkup` для всех меню §2–§11 INTERFACE.
- **DoD:** snapshot-тесты на JSON-репрезентацию каждой клавиатуры.

### T-24. Banner cache (1.5 ч)
- `alpaca/bot/banner_cache.py`: при первой отправке — `InputFile(storage_path)`, сохраняем `telegram_file_id` в БД, дальше — `file_id`.
- **DoD:** интеграционный тест с моком Bot API: первый вызов отправляет `InputFile`, второй — `file_id`.

### T-25. Renderer (2.5 ч)
- `alpaca/bot/renderer.py`: класс `MenuRenderer` по §11.5, возвращает `RenderedScreen(text, reply_markup, banner_code)`.
- Методы для всех меню + уведомлений из §12.
- **DoD:** unit-тесты на каждый метод, snapshot текста.

### T-26. `UserMiddleware` + `BanCheckMiddleware` + `ThrottleMiddleware` (3 ч)
- `UserMiddleware`: upsert `users` (tg_id, username, names) + `last_seen_at`, инжектит `data["user"]`.
- `BanCheckMiddleware`: читает Redis `user:ban:{tg_id}` (TTL 60с), при miss — из БД; при true — отвечает `t("common.banned")` и прерывает.
- `ThrottleMiddleware`: Redis TOKEN bucket (1 msg/s user, 15/10s hard-bound).
- **DoD:** integration тест: забаненный пользователь не может пройти хэндлер; флудер (20 апдейтов/сек) получает 429-подобный ответ в логе.

### T-27. Handler `/start` + главное меню + триал-инфо (2 ч)
- `bot/handlers/common.py`: `/start [?ref=<code>]` — upsert пользователя, если есть `ref` — заполняет `referrer_id` + создаёт `referrals` + outbox-уведомление реферреру.
- `bot/handlers/main_menu.py`: отрисовка главного меню по §2 INTERFACE; если `user.trial_used is False` — дополнительное сообщение §2.1.
- **DoD:** e2e-тест: `/start?ref=XXX` создаёт user+referrer link; повторный `/start` не дублирует.

### T-28. XUIClient (4 ч)
- `alpaca/integrations/xui/client.py`: `httpx.AsyncClient` + session cookie + asyncio.Lock на логин + retry one-time на 401/404 + circuit breaker (custom lightweight, без зависимостей).
- Методы: `login`, `list_inbounds`, `add_client`, `update_client`, `delete_client`, `get_client_traffics`, `build_vless_url`.
- Формирование `vless://` — по inbound settings (protocol, port, tls, реальность, sni, pbk, sid, fp).
- **DoD:**
  - unit-тесты с `respx` на все методы (happy + 401→re-login → retry + 5xx → breaker);
  - integration-тест c реальной dev-панелью **в отдельном marker `@pytest.mark.xui_live`** (skip в CI по умолчанию).
- **Риск:** нестабильность API 3x-ui между версиями. **Митигация:** фиксируем версию 3x-ui в compose, регулярно сверяемся с [MHSanaei/3x-ui issues](https://github.com/MHSanaei/3x-ui/issues).

### T-29. Outbox repository + enqueue helper (1.5 ч)
- `enqueue(user_id, kind, payload, *, dedup_key=None, priority=100, available_at=None)`; возвращает запись или `None` (если был дубль).
- Внутренний helper `outbox.enqueue_rendered(t_key, user_id, **text_args, reply_markup_key=None, banner_code=None)`.
- **DoD:** integration: два вызова с одним `dedup_key` — ровно одна запись в БД.

### T-30. Outbox flusher + rate-limit (4 ч)
- `alpaca/outbox/dispatcher.py`: цикл SELECT/UPDATE `FOR UPDATE SKIP LOCKED`, рендер полезной нагрузки на лету (locales, клавиатуры), вызов `bot.send_*`.
- Rate-limit: Lua-скрипт Redis INCR + EXPIRE, ключи `rl:bot:sec:{t}`, `rl:chat:{id}:sec:{t}`. Бакет «broadcast» — 25/s, «tx» — 30/s.
- Обработка `TelegramRetryAfter`, `TelegramForbiddenError`, generic 5xx → backoff.
- **DoD:**
  - unit: корректные переходы статусов (`pending`→`sent`/`failed`/`pending+backoff`);
  - e2e: 50 сообщений, bot замокан: выдержан глобальный лимит 30/s.

### T-31. ARQ cron для outbox + invoice TTL (2 ч)
- `tasks.outbox_flush` — cron `{second: {*/1}}` с короткой блокировкой;
- `tasks.invoice_ttl_sweep` — cron каждые 30 с: expired + outbox уведомление + попытка cancel у провайдера (stub на этом этапе).
- **DoD:** worker включённый `outbox_flush` пропускает накопленные записи; счёт с `expires_at < now()` переходит в `expired`.

### T-32. Handlers «Мои подписки» / «Инструкция» / «Помощь» / «Партнёрам» (статика) (3 ч)
- Простые хэндлеры, рендерящие через `MenuRenderer`.
- Партнёрка: расчёт статистики через `referral_service.get_stats(user)` (join'ы по `referrals`, `subscriptions`).
- Генерация `deeplink`: `t.me/<bot_username>?start=<ref_code>`.
- Кнопка «Поделиться» — через `switch_inline_query` (текст: «Подключайся к Alpaca VPN по моей ссылке: {deeplink}»).
- **DoD:** e2e: после `/start`, нажатие «Партнёрам» возвращает сообщение со статистикой и ссылкой в `<code>...</code>`.

### T-33. Backoff / retry / circuit breaker утилиты (1.5 ч)
- `alpaca/utils/retry.py`: `@retry(exceptions, max_attempts, base, cap, jitter)` поверх `tenacity` либо свой (чтобы не плодить тяжёлых зависимостей).
- `CircuitBreaker(name, fail_threshold=5, reset_timeout=60)` на asyncio.
- **DoD:** unit-тесты на открытие/закрытие, jitter, jittered backoff.

### T-34. SubscriptionService — базовые операции (без оплаты) (5 ч)
- Методы:
  - `grant_trial(user)` → создаёт trial subscription + vpn_key + outbox `trial_activated`;
  - `provision_vpn(subscription)` → saga (§8.5 архитектуры): pending → XUIClient.add_client → active → outbox.
  - `revoke(subscription, by_user_id)` → xui.delete_client (с компенсацией при ошибке).
  - `extend(subscription, days)` → active → `expires_at += days`, expired → новый период + xui.update_client или add_client.
- **DoD:**
  - integration (с моком XUIClient): успешный и неуспешный provision, корректные статусы, outbox dedup_keys правильные;
  - e2e: `/start` → «Активировать 3 дня» → в outbox два сообщения (welcome + ключ).

### T-35. Compensation taska для XUIClient (2 ч)
- `tasks.xui_compensation.retry_provision(subscription_id)` — ARQ-задача с exponential backoff, max 24 ч.
- Триггерится из `SubscriptionService` при `ProvisionError`.
- Еще есть «sweep»: раз в 2 мин находим `subscriptions.status='failed' OR vpn_keys.status='pending' AND updated_at < now()-5min` и пере-enqueue'им.
- **DoD:** интеграционный тест: XUI фэйлит 3 раза, на 4-й — success, пользователь получает ключ через outbox с дедупом.

**Фаза 2 DoD:** пользователь может нажать `/start`, получить триал, увидеть свою подписку в «Мои подписки», работает outbox с rate-limit, выдача ключа устойчива к падению панели.

---

## Фаза 3 — платежи и покупка подписки (40 ч)

### T-40. Интерфейс `PaymentProvider` + dataclass'ы (1 ч)
- `alpaca/payments/base.py` по §7.1 архитектуры (`InvoiceRequest`, `InvoiceRef`, `PaymentEvent`, `PaymentProvider` Protocol).
- **DoD:** mypy-чистые сигнатуры, unit-тест сериализации `PaymentEvent.raw` → JSON.

### T-41. PaymentRegistry + DI (1 ч)
- Registry по коду, `enabled_providers` из `settings.payments.enabled_providers`.
- **DoD:** unit-тест: `registry.active()` возвращает только включённые.

### T-42. Handler «Купить подписку» + меню тарифов + меню оплаты (3 ч)
- `handlers/buy.py`: рендер §3 INTERFACE, callback → создание `invoice(status='created')` в БД, переход в меню оплаты §5.
- В меню оплаты — кнопки только из `registry.active()`; текст в §5 формата.
- **DoD:** e2e: при клике «Месячный» → «Картой» создаётся invoice в статусе `pending` после обращения к провайдеру (mock).

### T-43. Stars provider (3 ч)
- `alpaca/payments/stars.py`:
  - `create_invoice`: вызывает `bot.send_invoice(chat_id, title, description, payload=invoice_id, currency='XTR', prices=[LabeledPrice(label, amount)])`;
  - т.к. вебхука у Stars нет, завязываемся на апдейты — отдельный handler в `handlers/payment.py`:
    - `pre_checkout_query` → `bot.answer_pre_checkout_query(ok=True)` (после проверок);
    - `message.successful_payment` → формируем `PaymentEvent` и отдаём в `PaymentIngestor` (см. T-47).
- **DoD:** e2e c моком Bot API — `PaymentEvent` дошёл до `PaymentIngestor.process` с корректным `invoice_id`.

### T-44. YooKassa provider (4 ч)
- SDK `yookassa`:
  - `create_invoice`: `Payment.create({ amount, confirmation: {type: 'redirect', return_url}, metadata: {invoice_id}, description })`;
  - `handle_webhook`: парсинг тела, IP allowlist (из `settings.payments.yookassa.allowed_ips`), верификация через `Payment.find_one(id)` перед выдачей (защита от спуфа);
- Регистрация вебхука — вручную в админке YooKassa (документируем в README).
- **DoD:**
  - unit: подпись/парсинг;
  - integration: `respx` мок сервера YooKassa — create+webhook → `PaymentEvent`;
  - e2e: webhook на `/pay/yookassa` → в БД payment, в outbox receipt.

### T-45. CryptoBot provider (2 ч)
- HTTP `createInvoice` с `payload=invoice_id`;
- Вебхук с HMAC: проверка `crypto-pay-api-signature` (см. [docs](https://help.crypt.bot/crypto-pay-api)).
- **DoD:** аналогично T-44 (`respx` + e2e).

### T-46. Lava / Wata stub (1 ч)
- Класс с методами, которые кидают `NotImplementedError`. Достаточно для будущего подключения.
- **DoD:** в registry провайдеры появляются только если в env `PAY_LAVA_ENABLED=1`, по умолчанию выключены.

### T-47. PaymentIngestor — единая точка обработки событий (3 ч)
- `alpaca/payments/ingestor.py`:
  1. Redis dedup `SET NX EX 86400`;
  2. `INSERT payments ... ON CONFLICT DO NOTHING RETURNING id`;
  3. если строка вставлена — `SubscriptionService.fulfill(invoice, payment)`;
  4. enqueue outbox с `dedup_key=f"payment:{payment.id}:receipt"`.
- **DoD:** unit-тест: повторный вызов с тем же provider_payment_id → одна запись в БД, одно сообщение в outbox.

### T-48. SubscriptionService.fulfill — покупка (3 ч)
- Ветвление по `invoice.intent`:
  - `buy` → `provision_vpn(new subscription)` или `provision_proxy`;
  - `extend` → `extend(existing subscription)`;
  - `gift` → создаёт подписку на `target_user_id`, шлёт ключ туда + receipt покупателю;
  - `proxy` → `ProxyService.issue(...)`.
- При первой активации — триггер `referral_service.on_first_activation`.
- **DoD:**
  - integration для каждой ветки;
  - e2e «Stars pay → ключ пришёл за 5 сек»;
  - e2e «YooKassa webhook → ключ пришёл».

### T-49. Invoice TTL — cancel у провайдера (2 ч)
- В `tasks.invoice_ttl_sweep` после перевода в `expired` — best-effort `provider.cancel(invoice)`:
  - YooKassa — `Payment.cancel(id)` (если в статусе `pending`);
  - CryptoBot — `deleteInvoice`;
  - Stars — нет cancel API, оставляем как есть;
- Логируем warning без падения.
- **DoD:** integration: invoice в состоянии `pending` старше TTL → `expired`, у YooKassa вызван cancel.

### T-50. Handlers: проверка статуса инвойса, «Повторить», «Назад» (2 ч)
- Кнопка «Проверить оплату» — пассивно, ручной триггер: читает `invoices.status`, если `paid` — шлёт ключ (на самом деле уже в outbox), если `pending` — сообщает.
- «Назад» из любого платёжного меню корректно сбрасывает FSM.
- **DoD:** e2e сценарий «нажал оплатить, закрыл и вернулся — получил статус pending».

### T-51. Антифрод — минимум (2 ч)
- Минимальные проверки:
  - суммы в `PaymentEvent` сверяем с `invoices.amount_*`; если расходятся — `status='failed'`, Sentry alert, пользователю — "оплата не прошла" (§13 INTERFACE);
  - поле `user_id` в webhook (если есть) должно совпадать с `invoices.user_id`.
- **DoD:** unit-тест: webhook с амаунтом, отличным от invoice, → failed + outbox error.

### T-52. Error handler + UX ошибок (2 ч)
- Глобальный `dp.errors_router.register(on_error)` — таблица §13 INTERFACE: ошибка → сообщение пользователю + Sentry.
- Отдельный обработчик `TelegramRetryAfter` / `TelegramNetworkError` — логируем, не показываем пользователю.
- **DoD:** все строки §13 INTERFACE реально отправляются при соответствующих `DomainError`-подклассах.

### T-53. Тесты на идемпотентность и гонки (3 ч)
- Параллельный тест: 10 одновременных webhook-ов с одним `provider_payment_id` → ровно 1 запись `payments`, 1 `subscription`, 1 outbox.
- Тест на crash после INSERT payments, до provision: saga дорабатывает через компенсацию (T-35).
- **DoD:** тесты зелёные, race не воспроизводится.

### T-54. Документация платежей и runbook (2 ч)
- `docs/payments.md`: карта вебхуков, как зарегистрировать в YooKassa/CryptoBot, curl-примеры тестовых событий, раннеровское «что делать если …».
- **DoD:** по документу dev’ов со стороны заказчика может настроить интеграцию с нуля.

### T-55. Handler «Купить подписку» полная версия + e2e (3 ч)
- Покрываем все тарифы §3 INTERFACE (включая «Быстрый 1 мес.»), выбор inbound по `tariff.xui_inbound_id`.
- **DoD:** e2e: каждый из 6 тарифов можно купить через Stars и YooKassa (mock).

### T-56. Уведомления по §12 (3 ч)
- Реализуем шаблоны:
  - `payment_success`, `extend_success`, `gift_received`, `trial_activated`, `expiry_t3`, `expiry_t0`, `referral_joined`, `referral_activated`, `referral_bonus`, `ban`, `unban`, `revoke`.
- Кнопки «Инструкция» / «Мои подписки» / «Продлить» / «Купить подписку» / «Написать в поддержку» согласно таблице.
- **DoD:** unit-тесты на рендер каждого уведомления; integration: все 12 уведомлений отправляются через outbox.

### T-57. Cron: expiry_reminders (2 ч)
- Задача каждые 5 мин:
  - `expires_at BETWEEN now()+'3 days' AND now()+'3 days 10 min'` → outbox `expiry_t3` (dedup_key: `reminder:t3:{sub_id}`);
  - `expires_at <= now()` AND `status='active'` → сначала `status='expired'`, затем outbox `expiry_t0`.
- **DoD:** integration: прогоняем время через `freezegun` в тесте — корректные уведомления в нужные моменты.

**Фаза 3 DoD:** полный цикл покупки через Stars / CryptoBot / YooKassa работает в dev (моки) и на стейдже (реальный YooKassa-test). Все уведомления §12 реализованы.

---

## Фаза 4 — продление, подарок, триал, рефералка (36 ч)

### T-60. Handler «Продлить подписку» (§7) (3 ч)
- FSM: выбор периода → invoice(intent='extend') → меню оплаты.
- После оплаты — `SubscriptionService.extend` (уже реализован в T-48).
- Отдельное предусмотрено сообщение «Подписка продлена! Новый срок: {дата}».
- **DoD:** e2e active-extend и expired-extend.

### T-61. Handler «Подарить другу» (§8) (5 ч)
- FSM: выбор тарифа (без FAST_MONTH) → ввод получателя (@username или forwarded message) → резолв через `users_repo` → подтверждение → invoice(intent='gift').
- Ошибка «получатель не запускал бота» (§13).
- После оплаты — два уведомления (A: «gift_sent», B: «gift_received»).
- **DoD:** e2e happy path + кейс «получатель не запускал /start» + кейс «дарю себе» (запрещено, понятное сообщение).

### T-62. Handler «Триал 3 дня» (§2.1) (2 ч)
- Callback «Активировать 3 дня» → `TrialService.activate`.
- Защита от двойного клика — `trial_used` + Redis lock на user.
- **DoD:** повторный клик → сообщение «Триал уже активирован».

### T-63. Referral — дeeplink + регистрация (2 ч)
- В `/start` обрабатываем `start_param`: ищем `users.referral_code`, создаём запись в `referrals` одноразово.
- Валидация: referrer ≠ referee, referee без реферера (set-once), referrer не забанен.
- **DoD:** unit + e2e.

### T-64. Referral — триггер активации + уведомления (2 ч)
- `ReferralService.on_first_activation(user)`:
  - помечает `referrals.first_activation_at = now()`;
  - отправляет реферреру `referral_activated` (с `dedup_key`);
  - enqueue `tasks.referral_bonus.try_grant(referrer_id)`.
- **DoD:** integration: триал у приглашённого → реферреру пришло «активировал подписку».

### T-65. Referral — начисление бонуса (3 ч)
- ARQ-таска `referral_bonus.try_grant(referrer_id)`:
  - advisory lock по user.id;
  - выбираем `COUNT(*) WHERE counted_for_bonus=false AND first_activation_at IS NOT NULL`;
  - `bonus_days = count // 2`;
  - транзакционно помечаем `2*bonus_days` записей + создаём `subscription(source='referral_bonus', duration_days=bonus_days)` (или продлеваем активную);
  - outbox `referral_bonus` × `bonus_days`.
- **DoD:** интеграция: 1→0 бонусов, 2→1, 3→1, 4→2. Гонка двух параллельных триггеров не создаёт дубля (advisory lock).

### T-66. Handler «Партнёрам» полная реализация (§10) (2 ч)
- Статистика, deeplink в `<code>...</code>`, `switch_inline_query`.
- «До следующего бонуса» = `2 - uncounted_active % 2`.
- **DoD:** e2e, snapshot текста.

### T-67. Инструкция / Помощь / URL-ки (1 ч)
- Handlers возвращают меню с кнопками URL (из §9, §11 INTERFACE).
- URL'ы — в `alpaca/domain/links.py` (редактируемо без БД, можно сделать в БД — но в MVP просто константы).
- **DoD:** e2e; ссылки — реальные из INTERFACE.

### T-68. Навигация «Назад» и стек меню (2 ч)
- Единый механизм через `data["nav"]` — стек FSM; кнопка «Назад» всегда возвращает ровно на 1 уровень.
- **DoD:** мультиуровневая навигация в тестах.

### T-69. Ban / Unban пользовательская часть (1 ч)
- `BanCheckMiddleware` уже работает. Здесь убеждаемся, что:
  - при бане приходит текст «Вы заблокированы» и любые inline-нажатия игнорируются с тем же текстом;
  - при разбане — текст «Вы разблокированы» через outbox + кнопка «Купить подписку».
- **DoD:** e2e сценарий.

### T-70. Gift / Extend: покрытие uвеomлений и ошибок (2 ч)
- Все уведомления §12 связаны с правильными сценариями, отсутствуют дублирующие.
- **DoD:** чек-лист сверки с таблицей §12 INTERFACE.

### T-71. Полный E2E-тест «новый пользователь путь до реферального бонуса» (2 ч)
- Сценарий:
  1. user A запускает бот, получает referral_code;
  2. user B `/start?ref=<code>` → referrer записан;
  3. user B активирует триал → A получает referral_activated;
  4. user C `/start?ref=<code>` → триал → A получает 1 день бонуса и referral_bonus.
- **DoD:** один pytest-файл прогоняет весь путь, все outbox-записи правильные.

### T-72. Документация пользователя (1 ч)
- `docs/user-guide.md` — короткий гайд «от регистрации до покупки». Пригодится для поддержки.
- **DoD:** read-only PR.

### T-73. Отладка/мониторинг референтных сценариев (1 ч)
- Добавляем structlog-поля `scenario` (purchase/extend/gift/trial/referral).
- **DoD:** в логе видно сквозной `request_id` и тип сценария для каждой операции.

### T-74. Проверка локализации всех уведомлений (1 ч)
- Скрипт `tools/check_i18n_coverage.py` — сверяет список ключей в коде с `ru.yaml`, падает в CI.
- **DoD:** job `i18n-coverage` в CI зелёный.

### T-75. Review и рефакторинг фаз 2–4 (3 ч)
- Code review самому себе: дубли, длинные функции, магические числа (вынести в `settings.limits`).
- **DoD:** PR-ревью чек-лист заполнен.

### T-76. Performance smoke (1 ч)
- Локально: k6 или locust, 100 RPS на webhook в течение 2 мин — p95 < 500 мс.
- **DoD:** отчёт в `docs/perf/smoke.md`.

### T-77. Chaos-smoke: отключаем 3x-ui (1 ч)
- Стопаем dev-панель, делаем 10 покупок → все попадают в saga-retry, по подъёму панели раздаются ключи.
- **DoD:** тест воспроизводим через `make chaos-xui`.

### T-78. Чистка FSM в Redis — periodic (1 ч)
- TTL на ключи FSM — 1 час (настройка RedisStorage). Периодический `FSM expiry` не требуется, но добавить cron `cleanup orphan jobs` в ARQ.
- **DoD:** конфиг RedisStorage валиден; таска удаляет застарелые джобы ARQ.

**Фаза 4 DoD:** пользовательские функциональные сценарии §2–§13 INTERFACE закрыты; триал, подарок, продление, рефералка работают end-to-end; 100% уведомлений §12 покрыты.

---

## Фаза 5 — прокси (22 ч)

### T-80. Решение по бэкенду прокси (0.5 ч)
- Если заказчик ответил на §18.1 архитектуры — следуем. Если нет — MVP на 3proxy (§9 архитектуры).
- **DoD:** зафиксирован выбор в `docs/adr/0001-proxy-backend.md` (ADR).

### T-81. Proxy-provisioner: контейнер + мини-API (5 ч)
- `docker/proxy_provisioner/Dockerfile`: Alpine + 3proxy + mini-FastAPI (`uvicorn`) на порту 8080.
- API:
  - `POST /accounts {kind, ttl_days}` → создать login/password, записать в `users.cfg`, SIGHUP, вернуть `{id, host, port, login, password, expires_at}`;
  - `DELETE /accounts/{id}` → удалить из конфига, SIGHUP;
  - `GET /healthz`.
- Auth: Bearer-токен из env.
- **DoD:** `curl` создаёт и удаляет аккаунты; `docker exec` видит 3proxy перечитавшим конфиг.

### T-82. Shared state между provisioner и bot (1 ч)
- Реальным источником правды остаётся БД `alpaca.proxy_accounts` в Postgres (одна БД на весь проект). Provisioner ходит в тот же Postgres через свой минимальный репозиторий (только write в `proxy_accounts` при issue/revoke).
- **Альтернатива:** provisioner отдаёт только факт, а запись в БД — на стороне bot. В MVP выбираем альтернативу — provisioner stateless, БД пишет bot. Это проще и соответствует §9 архитектуры.
- **DoD:** ADR дополнен.

### T-83. `ProxyBackend` в bot (2 ч)
- Интерфейс по §9.2 архитектуры, реализация `LocalProxyBackend` — httpx-клиент на provisioner.
- Fernet-шифрование `password` при записи в БД.
- **DoD:** unit + integration с поднятым provisioner.

### T-84. ProxyService (2 ч)
- `issue(user, tariff)` → `ProxyBackend.issue` → `proxy_accounts` → outbox `proxy_issued` (текст с `<code>host:port:login:password</code>`).
- `revoke(account)` → `ProxyBackend.revoke`.
- **DoD:** e2e: покупка прокси → пользователь получает строку формата.

### T-85. Handler «Купить прокси» (§4 INTERFACE) (2 ч)
- 4 кнопки (2×2), invoice intent='proxy'.
- **DoD:** e2e: каждая комбинация работает.

### T-86. SubscriptionService.fulfill proxy (1 ч)
- Ветка `proxy` в T-48 уже заготовлена, здесь доведение до работающего кода.
- **DoD:** e2e оплаты прокси через YooKassa (mock).

### T-87. ARQ `proxy_cleanup` (1 ч)
- Раз в 10 минут: `SELECT * WHERE status='active' AND expires_at<now()` → revoke в backend + `status='expired'`.
- **DoD:** integration: 2 аккаунта, `freezegun` → один истёк → cleanup удалил.

### T-88. Управление прокси в «Мои подписки» (§6) (1.5 ч)
- В разделе «Мои подписки» показываем не только VPN-ключи, но и активные прокси.
- **DoD:** e2e отрисовки с mixed-типами.

### T-89. Безопасность 3proxy (2 ч)
- `3proxy.cfg`: отключены все protocols, кроме нужных; bind на конкретный публичный IP; логи в volume с ротацией.
- Dry-run для конфига: `3proxy -t /path/cfg` в entrypoint.
- **DoD:** при синтаксической ошибке конфига — контейнер не стартует.

### T-90. E2E: полный путь (2 ч)
- Покупка SOCKS5 1 нед → активация → истёк через 7 дней (freezegun в тесте) → пользователь получил уведомление и аккаунт в `expired`.
- **DoD:** зелёный.

### T-91. Документация прокси (1 ч)
- `docs/proxy.md`: инструкция для админа по добавлению IP, настройке 3proxy, смене Fernet-ключа.
- **DoD:** read-only PR.

### T-92. Буфер (2 ч)
- Резерв на отладку.

**Фаза 5 DoD:** прокси-продажа работает end-to-end, истёкшие аккаунты закрываются автоматически.

---

## Фаза 6 — админка (44 ч)

### T-100. Router + `IsAdmin` filter + навигация (2 ч)
- `handlers/admin/__init__.py`: `Router("admin")` включается всегда, но внутри handler'ов — filter `IsAdmin` (по `users.is_admin`).
- Команда `/admin` → главное меню §14.1.
- **DoD:** не-админ получает «У вас нет доступа к этой функции».

### T-101. Начальный сид админов (1 ч)
- При старте `bot` — читаем `ADMIN_INITIAL_IDS` из env и `UPDATE users SET is_admin=true WHERE tg_id=ANY(...)`. Идемпотентно.
- **DoD:** integration: контейнер стартует → первый `/start` от админа уже в `is_admin=true`.

### T-102. Админ: Статистика §14.2 (4 ч)
- `stats_service.py` с методами: `users_counters`, `subscriptions_counters`, `tariff_distribution`, `proxy_counters`.
- Handler рендерит текст ровно по §14.2 + кнопки `Обновить`, `Экспорт CSV`, `Экспорт JSON`.
- Экспорт: async cursor → `BufferedInputFile`.
- **DoD:** e2e: админ получает текст + файлы.

### T-103. Админ: Финансы §14.6 (3 ч)
- `stats_service.financials()` — `SUM(amount_rub) WHERE status='succeeded' AND received_at BETWEEN ...`.
- Распределение по провайдерам + средний чек.
- История — пагинация 50/стр.
- Фильтр (по провайдеру/периоду) — FSM.
- **DoD:** e2e, валидные числа.

### T-104. Админ: Рассылка — подготовка (3 ч)
- FSM `AdminBroadcast` (см. §11.4 архитектуры): choosing_audience → filters → waiting_banner → waiting_text → preview.
- Сохранение в `broadcasts(status='draft')`.
- **DoD:** e2e до preview.

### T-105. Админ: Рассылка — запуск и прогресс (4 ч)
- При «Отправить» → `broadcasts.status='running'` → chunks по 500 user_id → enqueue `tasks.broadcast.send_chunk(broadcast_id, ids)` → записывает в outbox с priority=200 (low).
- Каждую минуту выдаём админу прогресс — через outbox (`dedup_key=broadcast:{id}:progress:{ts}`).
- По завершении → `status='done'`, итог.
- **DoD:** 1000 пользователей — прогресс виден, итог корректный, нет нарушения rate-limit.

### T-106. Админ: Пользователи — поиск и карточка §14.4.1 (3 ч)
- Ввод `@username`|id → `users_repo.search`.
- Карточка с реальными данными: все поля §14.4.1.
- Кнопки: «Выдать подписку» (T-110), «Забанить» (T-107), «История покупок», «Отправить сообщение».
- **DoD:** e2e.

### T-107. Админ: Бан / Разбан / Список (4 ч)
- `admin_service.ban(user_id, by_admin)`:
  - `users.is_banned=true`, запись в `user_bans`;
  - деактивируем активные подписки — `status='revoked'`, XUIClient.delete_client, proxy revoke (без возврата средств, §14.4.2);
  - инвалидируем Redis-кэш бана;
  - пользователю — `ban` уведомление.
- `admin_service.unban` — обратно (без восстановления ключей).
- Список забаненных — пагинация 10/стр.
- **DoD:** e2e, проверка, что подписки реально отозваны.

### T-108. Админ: «Отправить сообщение» (1 ч)
- FSM: ввод текста → enqueue outbox с priority=50.
- **DoD:** e2e.

### T-109. Админ: История покупок пользователя (1 ч)
- Пагинация 20/стр, JOIN `invoices + payments + subscriptions`.
- **DoD:** e2e.

### T-110. Админ: Выдать подписку §14.5.1 (3 ч)
- FSM: user → тариф (включая «Бесплатно» = $0 promo) → подтверждение → `SubscriptionService.provision_vpn(new subscription, source='admin_grant')`.
- При «Бесплатно» — не создаём invoice/payment.
- Audit log.
- **DoD:** e2e, пользователь получает ключ.

### T-111. Админ: Продлить / Отозвать §14.5.2–14.5.3 (3 ч)
- Используем `SubscriptionService.extend` / `revoke` (уже реализованы).
- Audit log.
- **DoD:** e2e.

### T-112. Админ: Списки подписок §14.5.4–14.5.5 (3 ч)
- Пагинация 20/стр, фильтры по тарифу, поиск.
- Массовое продление истёкших: чекбоксы (через callback data с batch_id в Redis) → выбор периода → enqueue `tasks.admin.bulk_extend`.
- **DoD:** e2e, 50 истёкших — массовое продление работает.

### T-113. Админ: Настройки — цены §14.7.1 (2 ч)
- FSM: выбор тарифа → ввод новой цены → подтверждение → `tariffs.price_rub` (+ `price_rub_extend` при необходимости) → audit log → инвалидация кэша тарифов (если будет).
- **DoD:** e2e.

### T-114. Админ: Настройки — баннеры §14.7.2 (3 ч)
- FSM: выбор баннера → ожидание photo → валидация (size, dimensions) → сохраняем в volume `banners/` + обновляем `banners.telegram_file_id=null` (чтобы при следующей отправке файл перезагрузился и получил новый `file_id`).
- **DoD:** e2e, новая картинка видна пользователю.

### T-115. Админ: Настройки — админы §14.7.3 (2 ч)
- Добавить/удалить админа по @/id, обновление `users.is_admin`, audit log.
- Защита: нельзя удалить последнего админа.
- **DoD:** e2e, корректная ошибка.

### T-116. Админ: Настройки — перезапуск §14.7.4 (1.5 ч)
- Подтверждение → `os.kill(1, signal.SIGTERM)` → компоновщик рестартует контейнер.
- Полный lifecycle: shutdown_hooks корректно, outbox завершает текущий batch.
- **DoD:** e2e в compose: админ жмёт кнопку → контейнер рестартует, бот возвращается в рабочее состояние.

### T-117. Админ: Настройки — логи §14.7.5 (1.5 ч)
- Чтение `/var/log/alpaca/app.jsonl` — `tail -n 100` с фильтрами (ошибки/платежи/входы — по structlog-полям).
- Выдаём файлом (BufferedInputFile) + preview в сообщении.
- **DoD:** e2e.

### T-118. Audit log UI (1 ч)
- Опционально для MVP: в `/admin` добавить «Журнал действий» — последние 50 записей `audit_log`.
- **DoD:** если не успеваем — переносим в пост-MVP.

### T-119. Права: все admin-действия логируем (1 ч)
- Декоратор `@audited("action_name")` на admin-сервисных методах — пишет `audit_log` в той же транзакции.
- **DoD:** в audit_log есть записи на каждое действие из фазы 6.

### T-120. Админ: документация (1 ч)
- `docs/admin.md` — чек-лист для оператора, FAQ.
- **DoD:** read-only PR.

### T-121. Буфер (0.5 ч)
- Запас на фиксы.

**Фаза 6 DoD:** админ может выполнить все операции §14, действия аудируются, интерфейс устойчив к невалидному вводу.

---

## Фаза 7 — стабилизация и прод (23 ч)

### T-130. Нагрузочное тестирование (3 ч)
- k6/locust скрипт: 50 RPS на /tg/<secret> с смесью апдейтов (80% сообщений, 20% callback).
- Цель: p95 < 500 мс, p99 < 1500 мс; CPU < 70%.
- **DoD:** отчёт в `docs/perf/load.md`, тюнинг pool_size Postgres по факту.

### T-131. Chaos-тесты в compose (2 ч)
- Выключаем `postgres` / `redis` / `3x-ui` по очереди — проверяем: health-чек, graceful ответ пользователю, outbox не теряется.
- **DoD:** `docs/chaos.md` с логами и результатами.

### T-132. Финальные метрики и алерты (2 ч)
- `/metrics` endpoint для `prometheus_client`;
- Uptime Kuma monitors: webhook, /healthz, /healthz/worker, платёжные вебхуки (fake ping через cron).
- Sentry: настроены алерты по частоте ошибок.
- **DoD:** в Uptime Kuma зелёные, ping падает — SMS/TG.

### T-133. Backup-pipeline (2 ч)
- `scripts/backup.sh`: `pg_dump -Fc` → `rclone copy` в B2, retention 14 дней.
- Ежедневный ARQ cron `pg_backup_trigger` + проверка pg_restore.
- **DoD:** backup восстанавливается в ephemeral контейнере, smoke-select проходит.

### T-134. Деплой на prod VPS (3 ч)
- Инструкция в `docs/deploy.md`:
  1. создать пользователя `alpaca`, установить docker;
  2. клонировать репо, создать `.env`, DNS на VPS;
  3. `docker compose up -d postgres redis`;
  4. `docker compose run --rm bot alembic upgrade head`;
  5. `docker compose up -d`;
  6. `python scripts/set_webhook.py`;
  7. smoke `/start` от тестового аккаунта.
- **DoD:** на prod VPS бот отвечает.

### T-135. YooKassa / CryptoBot регистрация вебхуков (1 ч)
- В кабинетах провайдеров регистрируем URL-ы `/pay/yookassa`, `/pay/cryptobot`.
- Приватные ключи — в `.env` на VPS.
- **DoD:** тестовый платёж на 10₽ прошёл, ключ выдан.

### T-136. Финальный прогон сценариев (2 ч)
- Все сценарии §15 INTERFACE руками прогоняем на prod c реальным telegram-аккаунтом.
- **DoD:** чек-лист `docs/release/mvp-smoke.md` полностью пройден.

### T-137. Runbook'и (2 ч)
- `docs/runbooks/`: `3xui-down.md`, `redis-down.md`, `payment-stuck.md`, `user-stuck.md`, `mass-refund.md`, `rollback.md`.
- **DoD:** по каждому — шаги, команды, контактные каналы.

### T-138. Secrets rotation policy (1 ч)
- Документ + скрипт: как ротировать `TG_BOT_TOKEN`, Fernet-ключ (с миграцией `proxy_accounts`), YooKassa secret.
- **DoD:** `docs/security/rotation.md`.

### T-139. SLO / SLA definitions (1 ч)
- `docs/slo.md`: формализуем NFR-1…NFR-8 из архитектуры, error budget.
- **DoD:** документ.

### T-140. Финальный CHANGELOG + релизная веха (1 ч)
- Теги `v0.9.0` (staging) → `v1.0.0` (prod).
- GH Release со ссылкой на документы.
- **DoD:** релиз опубликован.

### T-141. Передача проекта (2 ч)
- Встреча с заказчиком: demo, объяснение админки, передача доступов (.env оффлайн), ответы на вопросы.
- **DoD:** заказчик подтвердил приёмку.

### T-142. Пост-релиз мониторинг 72 часа (1 ч актива)
- Смотрим Sentry/Uptime Kuma; фикс-форварды — в отдельной ветке `hotfix/*`.
- **DoD:** первая неделя без P0-инцидентов.

**Фаза 7 DoD:** бот в проде, мониторинг активен, бэкапы идут, runbook'и готовы, заказчик принял MVP.

---

## Правила работы в репозитории

### Бранчинг
- `main` — protected, merge только через PR.
- Ветки: `feat/T-XX-slug`, `fix/T-XX-slug`, `chore/...`, `docs/...`.
- PR — малый (≤ 500 строк diff без учёта тестов). Большие задачи дробятся.

### Коммиты
- Conventional Commits: `feat(payments): add yookassa provider (T-44)`;
- `fix:`, `chore:`, `docs:`, `test:`, `refactor:`.

### PR
- Шаблон: «что / зачем / как тестировал / риски / связанные задачи».
- CI green (lint + tests + migrations + build).
- Self-review + оптически review от заказчика для фаз 3/6 (платежи, админка).

### Тесты
- Каждый сервис-метод ≥ 1 happy + ≥ 1 error кейс.
- Интеграционные тесты запускают Postgres/Redis через `testcontainers`.
- Маркеры: `@pytest.mark.slow`, `@pytest.mark.xui_live`, `@pytest.mark.pay_live`. В обычном CI только fast + integration.

### Документация
- `docs/architecture.md` = это `ARCHITECTURE.md`, копируется в репо.
- `docs/adr/` — архитектурные решения по ходу.
- `docs/runbooks/` — операционные.
- Любое UX-изменение — обновление `docs/user-guide.md`.

---

## Риски и их митигация

| # | Риск | Вероятность | Удар | Митигация |
|---|---|---|---|---|
| R-1 | 3x-ui API изменится между версиями | Средняя | Высокий | Фиксируем версию в docker-compose; респондер-тесты; при обновлении — прогон `tests/integration/xui/`. |
| R-2 | YooKassa отклонит KYC | Средняя | Средний | Готовый интерфейс под Lava/Wata; один день на переключение (T-46 → работающая реализация). |
| R-3 | Двойное начисление ключа | Низкая | Очень высокий | 3-уровневая идемпотентность (T-47, T-53). |
| R-4 | Потеря outbox при крэше | Низкая | Высокий | Outbox в Postgres, не в Redis; `SKIP LOCKED`; тесты крэша. |
| R-5 | CryptoBot / Lava API изменится | Низкая | Средний | Строгий DTO + schema-test на ответ; Sentry-alert при отклонении. |
| R-6 | DDoS на webhook | Низкая | Средний | Caddy rate-limit middleware + fail2ban по 429; белый список IP для платёжных вебхуков. |
| R-7 | Рассылка упирается в rate-limit | Высокая | Низкий | Bucket «broadcast» 25/s, отдельно от транзакционных уведомлений. |
| R-8 | Админ делает `restart bot` при активной рассылке | Низкая | Средний | Graceful shutdown: воркер завершает текущий chunk; остаток перезапускается по флагу `broadcasts.status='running'`. |
| R-9 | Потеря Fernet-ключа | Низкая | Очень высокий | Хранение в 2 местах (env + password manager), процедура rotate в T-138. |
| R-10 | `banners` volume потеряется при миграции сервера | Средняя | Низкий | Бэкап volume в B2 еженедельно + `telegram_file_id` обычно уже «прогрет» после первой отправки. |
| R-11 | Локальный 3proxy — узкое место по IP | Средняя | Средний | Архитектура предусматривает внешний провайдер через `ProxyBackend`; замена = 1 модуль. |
| R-12 | Ошибка в миграции на prod | Низкая | Высокий | `alembic up/down` smoke в CI; ручная проверка миграции на staging; backup перед apply. |
| R-13 | Выгорание разработчика на фазе 6 | Средняя | Средний | Разбили на мелкие T-XX; чёткие DoD; можно перенести T-117/T-118 в пост-MVP. |

---

## Чек-лист готовности к релизу MVP

- [ ] CI зелёный на `main`, тэг `v1.0.0` создан.
- [ ] Все T-01 … T-140 закрыты или осознанно отложены с issue-комментом.
- [ ] `.env` на prod заполнен, секреты не в git.
- [ ] Postgres backup выполнился и восстановлен (test restore).
- [ ] Uptime Kuma мониторит 4 эндпоинта, все зелёные.
- [ ] Sentry получает тестовое событие из prod (канарейка).
- [ ] Telegram webhook установлен с `secret_token`, `getWebhookInfo` возвращает корректный URL.
- [ ] YooKassa / CryptoBot webhook-ы зарегистрированы, тестовый платёж на 10 ₽ прошёл.
- [ ] Триал 3 дня выдаётся новому аккаунту.
- [ ] Реферальный бонус начислен при 2 активациях.
- [ ] Покупка SOCKS5 / HTTP прокси — строка выдаётся, конфиг 3proxy обновляется.
- [ ] Админка: ban/unban, выдача подписки, цена, баннер, рассылка на 100 пользователей — прошли.
- [ ] Runbook'и лежат в `docs/runbooks/`, описаны шаги для всех P0-сценариев.
- [ ] Заказчик получил доступ к prod VPS (SSH), Uptime Kuma, YooKassa/CryptoBot/Sentry кабинетам.
- [ ] Демонстрация проведена, MVP принят.

---

## Финальное замечание

План построен так, чтобы **после каждой фазы проект оставался рабочим и продемонстрируемым**:
- после фазы 1 — есть бот, который отвечает и пишет в БД;
- после фазы 2 — работает триал и выдача VPN-ключей;
- после фазы 3 — работает платный цикл;
- после фазы 4 — все пользовательские функции INTERFACE закрыты;
- после фазы 5 — прокси;
- после фазы 6 — админка;
- после фазы 7 — прод.

Это важно для психологии разработчика и для возможности передать проект в любой момент с минимальными потерями.
