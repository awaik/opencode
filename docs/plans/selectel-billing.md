# План Selectel: billing control plane

## 1. Scope

Selectel отвечает только за identity и финансовый control plane этого продукта.
Inference, prompts, routing и OpenRouter credentials на Selectel не размещаются.

Затрагиваемые репозитории:

- application: `/Users/mavbook/projects/direct-mcp/direct-mcp`;
- infrastructure: `/Users/mavbook/projects/servers-infra/server-selectel-lidfly`.

Production-контур:

- приложение и PostgreSQL LidFly на сервере `directmcp`;
- `users.balance`, платформенные платежи и `transactions` остаются текущим
  источником истины;
- новый inference billing использует общий кошелёк, но отдельные runs/holds.

## 2. Что не переиспользовать

Нельзя использовать `chat_runs` как desktop inference ledger, потому что он:

- связан с `chat_conversations` и `chat_messages`;
- предназначен для встроенного чата;
- имеет собственные concurrency constraints;
- использует отдельный onboarding promo ledger;
- имеет chat-specific lifecycle и worker semantics.

Переиспользуются только общие финансовые сущности и проверенные primitives:

- `users`;
- `payments` и пополнения;
- `transactions`;
- FX rate service;
- PostgreSQL transactions и row locks;
- Ed25519 canonical JSON/signature pattern;
- durable inbox/worker pattern Glas control plane.

## 3. Новая схема данных

### `inference_billing_runs`

Один пользовательский OpenCode run и одно агрегированное списание.

Минимальные поля:

```text
id uuid primary key
user_id -> users.id
device_id text
status
pricing_policy_version integer
fx_rate numeric(18,8)
pricing_multiplier numeric(18,8)
reserved_rub numeric(12,2)
cost_usd numeric(18,8)
cost_rub numeric(12,2)
wallet_debit_rub numeric(12,2)
unbilled_overage_rub numeric(12,2)
balance_before numeric(12,2)
balance_after numeric(12,2)
started_at timestamptz
completed_at timestamptz
created_at timestamptz
updated_at timestamptz
```

### `inference_billing_holds`

Authoritative reservation пользовательского баланса.

```text
id uuid primary key
run_id unique -> inference_billing_runs.id
user_id -> users.id
status
amount_rub numeric(12,2)
settled_rub numeric(12,2)
idempotency_key unique
request_fingerprint text
expires_at timestamptz
reconcile_after timestamptz
created_at timestamptz
settled_at timestamptz
cancelled_at timestamptz
```

### `inference_billing_model_calls`

Per-call финансовый audit ledger без prompt/output.

```text
id uuid primary key
run_id -> inference_billing_runs.id
request_id text
call_id text unique
purpose text
generation_id text
model text
provider text
routing_policy_version integer
input_tokens integer
output_tokens integer
reasoning_tokens integer
cached_tokens integer
total_tokens integer
cost_usd numeric(18,8)
finish_reason text
status text
cost_source text
created_at timestamptz
```

Реальные model/provider поля являются internal-only и исключаются из public
serialization.

### `inference_billing_event_inbox`

Идемпотентный inbox подписанных событий Hetzner.

```text
event_id uuid primary key
run_id uuid
user_id -> users.id
event_type text
idempotency_key text unique
payload jsonb
signature text
payload_digest text
status text
attempts integer
next_attempt_at timestamptz
received_at timestamptz
processed_at timestamptz
last_error text
```

## 4. Денежные инварианты

В базе должны быть CHECK/UNIQUE constraints, подтверждающие:

- суммы неотрицательны;
- `settled_rub <= amount_rub`;
- один run имеет не более одного активного hold;
- один run создаёт не более одной `transactions.type='ai'`;
- terminal run не может иметь active hold;
- один event/idempotency key не меняет смысл после первого приёма;
- status принадлежит закрытому перечислению;
- итог run равен сумме принятых model calls с учётом billing disposition.

Доступный пользователю остаток для новых reservations:

```text
available_balance = users.balance - sum(active inference holds)
```

Расчёт и создание hold выполняются под row lock пользователя. Никаких внешних
HTTP-вызовов внутри этой транзакции быть не должно.

## 5. Криптография

Создать отдельные secrets:

```text
LIDFLY_INFERENCE_GRANT_PRIVATE_KEY_FILE
LIDFLY_INFERENCE_BILLING_PUBLIC_KEY_FILE
```

Требования:

- Ed25519;
- отдельная key pair от Glas, skills и updater;
- private key Selectel монтируется только в billing application/worker;
- public key Hetzner billing-event signer монтируется read-only;
- startup validation проверяет наличие файла и тип ключа;
- canonical JSON и подписи покрываются shared test vectors;
- rotation проектируется через `key_id`, current + previous public key.

## 6. Public identity endpoint

```text
GET /api/inference/access-grant
```

Endpoint использует существующую пользовательскую сессию/API auth, проверяет:

- пользователь существует и verified;
- аккаунт не suspended/deleted;
- устройство разрешено;
- продукт не отключён администратором.

Ответ содержит короткоживущий signed access grant и срок действия. Баланс,
OpenRouter, routing tier и model names в grant не включаются.

## 7. Internal billing API

### Reservation

```text
POST /internal/inference/v1/reservations
```

Порядок:

1. Проверить source IP как defense in depth.
2. Проверить Ed25519 signature и `aud`.
3. Проверить schema version, время и payload digest.
4. Найти существующий idempotency key.
5. При повторе с тем же digest вернуть прежний spend grant.
6. При конфликтующем payload вернуть `409 idempotency_conflict`.
7. Заблокировать пользователя и active holds.
8. Проверить available balance.
9. Создать run + hold.
10. Сохранить pricing/FX snapshot.
11. Подписать и вернуть spend grant.

Недостаток средств возвращается до начала OpenRouter work.

### Reservation increase

```text
POST /internal/inference/v1/reservations/increase
```

Увеличение использует новый idempotency key, блокирует тот же run/user и не
может уменьшать уже зарезервированную сумму. Клиент никогда не задаёт model price
или markup.

### Settlement

```text
POST /internal/inference/v1/settlements
```

HTTP handler только валидирует envelope, проверяет подпись и идемпотентно пишет
inbox. Финансовый эффект выполняет worker.

Worker в одной транзакции:

1. Claim события через `FOR UPDATE SKIP LOCKED`.
2. Заблокировать run, hold и пользователя в фиксированном порядке.
3. Проверить grant/run/user/policy и лимит reservation.
4. Вставить model-call ledger через conflict-safe уникальные IDs.
5. Рассчитать или проверить агрегированную стоимость.
6. Списать `users.balance`.
7. Создать одну `transactions.type='ai'` с `inference_run_id`.
8. Закрыть hold и освободить остаток.
9. Завершить run и inbox event.

### Cancellation

```text
POST /internal/inference/v1/cancellations
```

- `model_work_started=false`: закрыть run без списания и освободить hold;
- `model_work_started=true`: ждать usage/settlement;
- неизвестный результат: `reconciliation_required`, hold не освобождать молча.

## 8. Пользовательский billing UI

История показывает агрегированную строку:

```text
Работа и ответ ИИ
```

Публичные данные:

- дата;
- сумма;
- статус;
- короткий run ID для поддержки;
- возвращённый/освобождённый hold при необходимости.

Скрытые данные:

- model/provider;
- reasoning effort;
- OpenRouter и generation IDs;
- internal routing policy.

## 9. Reconciliation и алерты

Worker регулярно ищет:

- holds без settlement после `reconcile_after`;
- inbox events с исчерпанными retry;
- terminal runs с active holds;
- settled runs без AI-транзакции;
- AI-транзакции без settled run;
- расхождение run aggregate и model calls;
- отрицательный available balance;
- повторяющиеся подписи с разными payload digests.

Автоматически повторять финансовый эффект безопасно только через idempotent
event processing. Неоднозначный upstream work требует сверки с Hetzner.

## 10. Тестовый план

### Unit

- canonical JSON;
- signature verify/rotation;
- decimal rounding;
- reservation calculation;
- state transitions;
- duplicate/conflicting idempotency keys;
- public serialization redaction.

### Database integration

- две параллельные reservations одного баланса;
- duplicate settlement;
- model calls с повторяющимися IDs;
- rollback при ошибке transaction insert;
- lock ordering без deadlock;
- stale hold reconciliation.

### Contract

- известные payload/signature fixtures для обеих сторон;
- unsupported schema version;
- expired/future-dated event;
- invalid audience;
- malformed decimal strings;
- settlement выше reservation.

## 11. Инфраструктурный план Selectel

В `server-selectel-lidfly`:

1. Добавить key-file mounts для новых inference keys.
2. Добавить env только для secret file paths/credentials; pricing и limits не
   хранить в `.env`.
3. Опубликовать internal billing routes через отдельный Traefik router.
4. Разрешить source IP Hetzner, сохранив обязательную application signature.
5. Добавить health/readiness billing worker.
6. Добавить Loki/Grafana alerts для inbox lag и stale holds.
7. Включить новые таблицы в существующий PostgreSQL backup автоматически через
   полный `pg_dump`.
8. Добавить smoke без финансовой записи и отдельный явный paid canary.

## 12. Этапы PR

1. Contracts, crypto и test vectors.
2. Schema/migration и query layer.
3. Reservation и access grants.
4. Settlement inbox/worker.
5. Billing history/admin analytics.
6. Infra routes, secrets и monitoring.

## 13. Definition of done

- Все финансовые операции идемпотентны на уровне БД.
- Ни один prompt/output не сохраняется и не логируется.
- Параллельные reservations не превышают баланс.
- Settlement выше hold отклоняется без частичного эффекта.
- Повтор settlement не меняет баланс.
- Пользователь видит одну агрегированную транзакцию.
- Stale/unknown состояния наблюдаемы и не освобождаются молча.
- Production deploy выполнен только после отдельного подтверждения и exact-SHA
  release checks owning-репозитория.
