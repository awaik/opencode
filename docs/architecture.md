# Архитектура LidFly Desktop inference

## 1. Назначение

LidFly Desktop — закрытое desktop-приложение с локальным OpenCode 2. OpenCode
читает и изменяет выбранный пользователем проект локально, а inference выполняет
через принадлежащий LidFly gateway в Германии. Пользователь не получает
OpenRouter API key и не управляет реальными моделями.

## 2. Границы системы

```text
┌─────────────────────────────────────────────────────────────┐
│ Компьютер пользователя                                      │
│                                                             │
│ proprietary UI -> OpenCode -> LidFly provider adapter       │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTPS / SSE
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Hetzner, Германия: inference data plane                     │
│                                                             │
│ auth grant verification                                     │
│ request validation -> reservation -> routing -> OpenRouter   │
│ operational ledger -> signed billing outbox                 │
└──────────────────────┬───────────────────────┬──────────────┘
                       │ signed control plane  │ HTTPS
                       ▼                       ▼
┌────────────────────────────────────┐  ┌─────────────────────┐
│ Selectel: billing control plane     │  │ OpenRouter          │
│                                    │  │ real model/provider │
│ identity, balance, holds, debit     │  └─────────────────────┘
│ transactions, payment history      │
└────────────────────────────────────┘
```

### Selectel хранит

- пользователя и его статус;
- баланс и платформенные платежи;
- billing holds;
- итоговые inference runs для истории списаний;
- per-call usage, необходимый для финансового аудита;
- `transactions` и пользовательскую стоимость;
- pricing policy, FX snapshot и markup;
- idempotent inbox подписанных событий от Hetzner.

### Selectel не хранит

- промпты;
- исходный код и содержимое файлов;
- tool arguments и tool results;
- reasoning content;
- model output;
- OpenRouter API key;
- runtime routing state.

### Hetzner хранит

- активные inference runs и provider turns;
- server-side routing policies;
- выбранные реальные модели/providers;
- OpenRouter generation IDs и usage;
- техническую себестоимость;
- durable outbox финансовых событий;
- correlation IDs, тайминги и транспортные статусы.

### Desktop хранит

- локальные OpenCode sessions;
- файлы и изменения проекта;
- refresh credential в защищённом хранилище ОС;
- локальные permission decisions;
- пользовательскую историю и состояние UI.

## 3. Публичный inference-контракт

Gateway предоставляет минимальный OpenAI-compatible surface:

```text
GET  /v1/models
POST /v1/chat/completions
POST /v1/runs
GET  /v1/runs/:run_id
POST /v1/runs/:run_id/cancel
```

OpenCode видит только:

```text
provider_id = lidfly
model_id    = lidfly-code
```

`GET /v1/models` никогда не возвращает внутренние модели. Gateway принимает
только `lidfly-code`, полностью пересобирает upstream body по allowlist и
самостоятельно устанавливает:

- реальный `model`;
- `provider.order` и provider allowlist;
- `allow_fallbacks`;
- `require_parameters`;
- `max_price`;
- reasoning configuration;
- output и context limits;
- privacy routing.

В ответах реальные model/provider names заменяются виртуальным ID. Upstream
headers и error bodies не передаются пользователю без фильтрации.

## 4. Идентификация и доступ

### User access grant

Desktop после пользовательского входа получает от Selectel короткоживущий
EdDSA-signed grant:

```json
{
  "schema_version": 1,
  "sub": "user_id",
  "device_id": "device_id",
  "aud": "lidfly-inference",
  "iat": 0,
  "exp": 0,
  "jti": "uuid"
}
```

Hetzner проверяет подпись локально. Публичный ключ Selectel не является
секретом. Access grant не содержит баланс и не даёт безусловного права начать
платный inference.

### Spend grant

До первого OpenRouter call Hetzner запрашивает reservation. Selectel блокирует
деньги и возвращает подписанный spend grant:

```json
{
  "schema_version": 1,
  "grant_id": "uuid",
  "run_id": "uuid",
  "sub": "user_id",
  "max_charge_rub": "30.00",
  "fx_rate": "92.50000000",
  "pricing_multiplier": "2.00000000",
  "pricing_policy_version": 1,
  "aud": "lidfly-inference-spend",
  "iat": 0,
  "exp": 0,
  "jti": "uuid"
}
```

Денежные значения в wire-контрактах передаются decimal strings, а не binary
floating point.

## 5. Billing state machine

```text
reservation_requested
  -> reserved
     -> increased
     -> settlement_pending
        -> settled
     -> cancellation_pending
        -> cancelled
     -> reconciliation_required
```

Основные инварианты:

- один `run_id` соответствует одному пользовательскому списанию;
- один `event_id` применяется не более одного раза;
- один `idempotency_key` не может описывать разные payload;
- сумма settlement не превышает активный hold;
- OpenRouter work не начинается до `reserved`;
- повторная доставка возвращает сохранённый результат;
- остаток hold освобождается только при settlement, cancellation без model work
  или подтверждённом reconciliation;
- неизвестный внешний результат нельзя автоматически повторять.

## 6. Inference state machine

```text
created
  -> awaiting_reservation
  -> ready
  -> running
     -> usage_pending
     -> settlement_pending
     -> completed

terminal alternatives:
  rejected
  cancelled_without_work
  cancelled_after_work
  failed_without_work
  reconciliation_required
```

Каждый provider turn имеет отдельный идемпотентный request ID. Каждый
фактический OpenRouter вызов имеет отдельную model-call запись. Router,
classifier, main, fallback и recovery являются разными model-call purposes.

## 7. Routing

Routing находится только на Hetzner и управляется версионированной политикой.
Client fields являются недоверенными hints и не могут увеличивать стоимость или
лимиты.

Первый релиз использует одну tool-capable coding-модель. Дальнейший порядок:

1. Детерминированный default tier.
2. Strong tier для сложных задач.
3. Long-context tier.
4. Provider fallback внутри одной модели.
5. Model fallback при доказанном отсутствии model work.
6. Optional classifier, учитываемый как отдельный платный call.

Реальная модель фиксируется на routing epoch. При смене model family gateway
удаляет несовместимые provider-native reasoning/continuation metadata.

## 8. Расчёт стоимости

Финансовая формула определяется Selectel и фиксируется в spend grant:

```text
customer_cost_rub =
  round_up(openrouter_cost_usd * fx_rate * pricing_multiplier, 0.01 ₽)
```

OpenRouter-reported cost является предпочтительным источником себестоимости.
Если финальный usage отсутствует, gateway переводит call в `usage_pending` и
пытается получить generation stats. Неоднозначный результат не запускается
повторно автоматически.

Статические model prices используются только для предварительного reservation и
аварийной маркированной оценки, но не должны молча заменять authoritative usage.

## 9. Межсерверное доверие

Используются две независимые пары Ed25519-ключей:

```text
Selectel private -> подписывает access/spend grants
Hetzner public   -> проверяет grants

Hetzner private -> подписывает billing events
Selectel public -> проверяет billing events
```

Ключи других продуктов, включая Glas, повторно не используются. Source IP
allowlist является defense in depth, а не основной аутентификацией.

Каждый подписываемый payload содержит:

- `schema_version`;
- `event_id` или `jti`;
- `sub`;
- `aud`;
- время создания и истечения;
- idempotency key;
- canonical JSON representation.

## 10. Данные и логи

Запрещено логировать:

- Authorization и refresh credentials;
- spend grants;
- prompt/message bodies;
- file contents;
- tool inputs/results;
- reasoning;
- OpenRouter response text.

Разрешены:

- user/run/request/call IDs;
- generation ID;
- размеры тела и контекста;
- token counts;
- cost;
- routing policy version;
- latency и transport outcome;
- нормализованные error classes.

Немецкий egress IP не означает, что upstream provider обрабатывает данные только
в ЕС. Юрисдикция и data-retention class выбранного provider должны быть частью
routing policy и пользовательской документации.

## 11. Нефункциональные требования

- Нельзя держать PostgreSQL-транзакцию открытой во время внешнего HTTP/SSE.
- Deployment gateway не должен обрывать активные streams раньше documented
  maximum request duration.
- Billing outbox переживает restart и доставляет события at-least-once.
- Billing inbox применяет события exactly-once на уровне финансового эффекта.
- OpenRouter key для desktop изолирован от ключей чата и изображений и имеет
  внешний spending limit.
- Все production write/deploy операции выполняются только из owning infra repo
  и после отдельного подтверждения.

## 12. Решения, которые нужно принять до внешней беты

- стандартный server-side max cost одного run;
- разрешённое пользователю уменьшение собственного лимита;
- политика оплаты отменённой после model work задачи;
- срок хранения технического usage ledger;
- правила write-off при необратимо неизвестной upstream-стоимости;
- поддержка изображений в первом релизе;
- допустимые provider data-retention классы;
- необходимость второго немецкого хоста для host-level HA.
