# План Hetzner: inference data plane

## 1. Scope

Немецкий сервер `loomknot` (`88.99.138.18`) принимает inference-трафик desktop,
выбирает модель/provider, обращается к OpenRouter и доставляет usage в Selectel.

Owning-репозиторий:

```text
/Users/mavbook/projects/servers-infra/aura-proxy-hetzner
```

Текущий `proxy` остаётся reverse proxy для существующих OpenRouter, Resend и
Telegram consumers. Product inference строится отдельным сервисом и не меняет
семантику существующего `/api/*`.

## 2. Почему нельзя открыть текущий `/api/*` desktop-клиенту

Текущий raw proxy:

- использует shared mobile/server keys, а не user identity;
- пропускает client-controlled OpenRouter body;
- позволяет передать реальный `model` и `provider`;
- ожидает клиентский OpenRouter Bearer;
- не создаёт reservation и usage ledger;
- не фильтрует model/provider из upstream response;
- имеет permissive CORS;
- проксирует body без продуктового size/schema validation.

Поэтому новый gateway разбирает и пересобирает OpenAI-compatible запросы на
application level. `http-proxy-middleware` можно оставить для старого egress,
но не использовать как security boundary публичного inference.

## 3. Целевые сервисы

```text
proxy
  существующий reverse proxy

inference-gateway
  public HTTPS/SSE API, минимум 2 replicas

inference-worker
  billing outbox, reconciliation, cleanup

postgres
  operational inference ledger

redis
  distributed user/device/IP rate limits и concurrency
```

На одном физическом сервере replicas защищают от process/container failure и
позволяют start-first deploy, но не обеспечивают host-level HA.

## 4. Предлагаемая структура кода

```text
src/inference/server.js
src/inference/config.js
src/inference/auth.js
src/inference/contracts.js
src/inference/request-policy.js
src/inference/run-service.js
src/inference/router.js
src/inference/routing-policy.js
src/inference/openrouter-client.js
src/inference/openrouter-stream.js
src/inference/public-stream.js
src/inference/usage.js
src/inference/billing-client.js
src/inference/billing-outbox.js
src/inference/reconciliation.js
src/inference/logger.js
src/inference/db/*
```

Транспортные keep-alive, readiness и correlation primitives текущего proxy можно
выделить и переиспользовать, если это не меняет старые маршруты.

## 5. Public API

### `GET /v1/models`

Возвращает ровно одну виртуальную модель:

```json
{
  "object": "list",
  "data": [
    {
      "id": "lidfly-code",
      "object": "model",
      "owned_by": "lidfly"
    }
  ]
}
```

### `POST /v1/chat/completions`

Первый релиз поддерживает только streaming OpenAI Chat Completions, необходимый
текущему OpenCode adapter:

- `messages`;
- function `tools`;
- `tool_choice`;
- `stream=true`;
- bounded `max_tokens` hint;
- ограниченный набор sampling fields;
- text и при отдельном решении image input.

Gateway отклоняет:

- model, отличный от `lidfly-code`;
- client `provider`, `models`, `route`, `plugins`;
- client price/fallback configuration;
- неизвестные privileged fields;
- oversized body/context/tool schema;
- unsupported media/data URLs;
- malformed tool history;
- запрос без активного run/spend grant.

### Run API

```text
POST /v1/runs
GET  /v1/runs/:run_id
POST /v1/runs/:run_id/cancel
```

Создание run принимает user access grant, но до OpenRouter work обязательно
создаёт reservation на Selectel.

## 6. Operational schema

### `inference_runs`

```text
id uuid primary key
user_subject_hash text
device_id_hash text
status text
grant_id uuid unique
grant_jti uuid unique
max_charge_rub numeric(12,2)
accrued_charge_rub numeric(12,2)
pricing_policy_version integer
routing_policy_version integer
routing_epoch integer
selected_tier text
created_at timestamptz
last_activity_at timestamptz
completed_at timestamptz
```

Сырой email или лишние profile data не сохраняются. `sub` хранится только если
он необходим для подписанного billing payload; observability использует HMAC/hash.

### `inference_requests`

Один OpenCode provider turn.

```text
id uuid primary key
run_id -> inference_runs.id
idempotency_key text
request_digest text
status text
model_work_started_at timestamptz
first_byte_at timestamptz
completed_at timestamptz
finish_reason text
error_class text
```

Unique constraint: `(run_id, idempotency_key)`. Повтор с другим digest является
конфликтом, а не новым inference.

### `inference_model_calls`

```text
id uuid primary key
request_id -> inference_requests.id
purpose text
attempt integer
model text
provider_policy jsonb
generation_id text
status text
usage jsonb
cost_usd numeric(18,8)
cost_source text
started_at timestamptz
completed_at timestamptz
```

Prompt, messages, output и reasoning в таблицу не записываются.

### `inference_billing_outbox`

```text
event_id uuid primary key
run_id uuid
event_type text
payload jsonb
signature text
status text
attempts integer
deliver_after timestamptz
lease_expires_at timestamptz
last_error text
created_at timestamptz
delivered_at timestamptz
```

## 7. Routing policy

Политика хранится версионированно и активируется атомарно:

```text
draft -> validated -> active -> retired
```

Политика содержит только структурированные данные:

- tiers и реальные model IDs;
- required capabilities;
- context/output limits;
- reasoning default/max;
- provider order/only;
- `allow_fallbacks`;
- `require_parameters`;
- `max_price`;
- data-retention/ZDR requirements;
- fallback graph;
- per-request и per-run safety caps.

Секреты и произвольный исполняемый код в routing policy не хранятся.

Первый release содержит одну model config и не выполняет LLM-классификацию.

## 8. Reservation flow

До любого платного model call:

1. Проверить access grant локально.
2. Валидировать и оценить request без внешней модели.
3. Выбрать разрешённый initial tier детерминированно.
4. Рассчитать conservative upper bound.
5. Создать подписанный reservation request.
6. Получить spend grant от Selectel.
7. Проверить подпись, audience, run/user и срок.
8. Сохранить grant в local run transaction.
9. Только теперь вызвать OpenRouter.

Если следующему provider turn не хватает остатка grant, run приостанавливается и
запрашивает reservation increase. Нельзя сначала потратить, а потом пытаться
получить разрешение.

## 9. OpenRouter client

Используется отдельный secret:

```text
OPENROUTER_INFERENCE_API_KEY
```

Ключ:

- никогда не принимается от desktop;
- никогда не возвращается в response/error;
- изолирован от встроенного чата и image generation;
- имеет OpenRouter spending limit;
- поддерживает rotation без остановки активных requests.

Gateway самостоятельно добавляет internal app attribution и pseudonymous
session/user tracking. Email и прямой LidFly user ID в OpenRouter не передаются.

## 10. Streaming

Gateway должен корректно проксировать:

- text deltas;
- reasoning deltas в поддерживаемой виртуальной форме;
- incremental tool calls;
- finish reason;
- финальный usage;
- `[DONE]`.

Gateway фильтрует:

- реальный response `model`;
- provider/openrouter metadata;
- upstream infrastructure headers;
- raw provider errors;
- internal generation IDs из public payload, если они не нужны adapter.

При client disconnect:

1. Abort upstream request.
2. Зафиксировать `model_work_started` и известный generation ID.
3. Не повторять request автоматически.
4. Собрать доступный usage или перевести call в reconciliation.
5. Доставить settlement/cancellation event.

## 11. Billing outbox

Settlement формируется только после durable записи local usage. Outbox worker:

1. Claim через lease/`SKIP LOCKED`.
2. Подписывает canonical payload отдельным Hetzner private key.
3. POST в Selectel billing inbox.
4. Считает `2xx` идемпотентным подтверждением.
5. Повторяет retryable failures с backoff.
6. Не повторяет invalid signature/schema failures бесконечно.
7. Поднимает alert при lag/exhaustion.

Временная недоступность Selectel после уже выданного spend grant не обрывает
активный OpenRouter stream. Settlement сохраняется локально и доставляется позже.

## 12. Rate limits и abuse protection

Лимиты накладываются одновременно по:

- user subject;
- device;
- run;
- source IP;
- числу параллельных streams;
- requests/minute;
- calls/run;
- run duration;
- reserved и accrued RUB;
- request/body/tool schema size.

Shared API key не является tenant boundary. Redis limiter использует проверенный
grant subject, а не client-provided header.

## 13. Таймауты и deploy

Разделить:

- connect timeout;
- time-to-first-byte limit;
- upstream inactivity watchdog;
- documented absolute request cap;
- run inactivity timeout;
- billing delivery timeout.

`stop_grace_period` и application drain должны быть длиннее максимального
inference request. Новый gateway нельзя наследовать текущий 30-second forced
shutdown для десятиминутных streams.

Traefik:

- отдельный router `inference.lidfly.ru`;
- frequent SSE flush;
- no response buffering/transformation;
- request body limit;
- start-first update;
- readiness отдельно от liveness.

## 14. Observability

Метрики:

- request rate и active streams;
- TTFT, duration, tokens/sec;
- client disconnect rate;
- provider HTTP/transport errors;
- routing tier/model/provider internally;
- fallback rate;
- usage pending;
- OpenRouter cost;
- reserved/accrued customer RUB;
- billing outbox lag;
- reconciliation queue;
- OpenRouter key remaining limit.

Логи не содержат prompts, tools или response content.

## 15. Тестовый план

### Fake OpenRouter

- normal SSE;
- reasoning;
- fragmented tool arguments;
- final usage;
- 400/401/403/408/429/5xx;
- timeout before headers;
- disconnect after headers;
- partial output disconnect;
- generation ID without usage;
- malformed SSE;
- unsupported parameters.

### Billing contract

- valid/invalid/expired grants;
- wrong audience/key ID;
- duplicate reservation;
- reservation increase;
- duplicate settlement;
- delayed Selectel availability;
- outbox restart recovery.

### Security

- client model/provider override;
- leaked internal error/model fields;
- oversized compressed/uncompressed body;
- tool schema bombs;
- forged subject/run headers;
- shared-key replay;
- CORS/browser abuse;
- concurrent use of one spend grant.

## 16. Этапы PR

1. Contracts, config validation и fake upstream.
2. Operational schema и run/request services.
3. Billing client, spend grants и durable outbox.
4. OpenAI-compatible streaming gateway для одной модели.
5. Redis limits, cancellation и reconciliation.
6. Routing policy versions и дополнительные tiers.
7. Production stack, secrets, monitoring и backup.

## 17. Definition of done

- Desktop не может управлять реальными моделями/providers.
- OpenRouter key не покидает Hetzner.
- Нельзя начать model work без spend grant.
- Local usage и settlement переживают restart.
- Duplicate requests/events не создают двойной inference или debit.
- Public response не раскрывает OpenRouter/model/provider.
- Активные streams корректно drain при deploy.
- Operational PostgreSQL имеет проверенный backup/restore.
- Paid canary ограничен отдельным OpenRouter key и жёстким spend cap.
