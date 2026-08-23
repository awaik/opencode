# Порядок интеграции и выпуска

## 1. Стратегия

Проект развивается сквозными вертикальными срезами. Нельзя сначала построить
полный router или desktop UI, а затем пытаться добавить биллинг. Каждый этап
должен завершаться работающим end-to-end сценарием и проверяемым денежным
инвариантом.

## 2. Этап A: контракты

Результат:

- утверждённые state machines;
- JSON schemas подписанных сообщений;
- canonical JSON algorithm;
- Ed25519 test vectors;
- error contract;
- OpenAI-compatible public contract;
- pricing/rounding rules;
- перечень запрещённых логируемых данных.

Gate:

- одинаковые fixtures проходят в `direct-mcp` и `aura-proxy-hetzner`;
- unknown schema version и invalid signature fail closed;
- денежные поля не используют binary floating point.

## 3. Этап B: billing без inference

Результат:

- migrations;
- reservation/increase;
- signed spend grant;
- settlement inbox/worker;
- cancellation;
- AI transaction;
- reconciliation queries.

Тест использует подписанные fixtures, но не вызывает Hetzner/OpenRouter.

Gate:

- конкурентные holds не превышают баланс;
- duplicate settlement не списывает повторно;
- settlement выше hold атомарно отклоняется;
- failed transaction не оставляет частично завершённый run.

## 4. Этап C: gateway с fake dependencies

Результат:

- отдельный inference gateway;
- local operational PostgreSQL;
- fake Selectel billing endpoint;
- fake OpenRouter SSE;
- run/request/model-call ledger;
- durable billing outbox;
- cancel и reconnect policy.

Gate:

- рестарт gateway/worker не теряет settlement;
- повтор provider request не вызывает второй upstream call;
- public stream не раскрывает internal model/provider;
- client-controlled routing fields отклоняются или удаляются по контракту.

## 5. Этап D: локальный server-to-server E2E

Поднять оба server contours локально с тестовыми ключами:

```text
fake desktop
  -> inference gateway
  -> fake OpenRouter

inference gateway
  <-> real direct-mcp billing code + test PostgreSQL
```

Обязательные сценарии:

1. Успешный run и частичное освобождение hold.
2. Duplicate settlement.
3. Недостаточный баланс до model work.
4. Cancel до model work.
5. Cancel после первых tokens.
6. Gateway crash после OpenRouter response, до billing delivery.
7. Selectel outage во время settlement.
8. Generation ID есть, usage отсутствует.
9. Settlement выше grant.
10. Две параллельные задачи одного пользователя.

Gate:

- после recovery обе базы приходят к одному финансовому результату;
- ни один сценарий не создаёт повторного списания;
- неизвестное внешнее состояние не повторяется автоматически.

## 6. Этап E: OpenCode against fake gateway

Результат:

- managed LidFly provider;
- one virtual model;
- provider-turn idempotency;
- run lifecycle;
- tool calls;
- cancel;
- public billing errors.

Gate:

- задача с локальным read tool и edit tool проходит полностью;
- OpenCode делает несколько provider turns в одном billing run;
- UI нигде не показывает внутренние model/provider fields;
- project config не перенаправляет provider на другой endpoint.

## 7. Этап F: staging infrastructure

### Hetzner

- новый domain/router;
- inference gateway/worker;
- PostgreSQL/Redis volumes;
- test OpenRouter key;
- test signing keys;
- logs/metrics/alerts;
- backup and restore drill.

### Selectel

- billing routes;
- public access-grant route;
- billing worker;
- key mounts;
- IP allowlist;
- metrics/alerts;
- migrations.

Gate:

- production traffic существующего aura proxy не изменился;
- health/readiness корректно различают process health и provider outage;
- start-first deploy не обрывает тестовый long stream;
- billing inbox/outbox lag виден в monitoring.

## 8. Этап G: платный canary

Ограничения:

- отдельный OpenRouter key;
- внешний key spending limit;
- один тестовый пользователь;
- одна дешёвая tool-capable coding-модель;
- один active run;
- небольшой server max charge;
- routing/fallback отключены;
- ручная ежедневная сверка.

Сверка:

```text
OpenRouter activity/cost
  = Hetzner inference_model_calls
  = delivered settlement model calls
  = Selectel inference_billing_model_calls
  = inference_billing_runs aggregate
  = transactions debit
  = users.balance delta
```

Допустимы только объяснимые rounding differences в пределах утверждённого
денежного правила.

Gate:

- несколько десятков runs без потери/двойного списания;
- нулевой unresolved reconciliation backlog;
- полная сверка себестоимости и пользовательских списаний;
- model/provider не раскрыты клиенту.

## 9. Этап H: routing tiers

Добавлять по одному изменению:

1. Provider fallback той же модели.
2. Strong model tier.
3. Deterministic promotion.
4. Long-context tier.
5. Optional classifier.

Каждое изменение требует:

- новой routing policy version;
- price/reservation regression;
- compatibility test tools/reasoning/context;
- cost/margin comparison;
- rollback на предыдущую policy version.

## 10. Этап I: внутренняя beta

Перед расширением аудитории:

- завершить permissions;
- ограничить чтение/запись project root;
- закрыть credentials paths;
- проверить package installation и network confirmations;
- проверить updater/signing;
- подготовить privacy disclosure;
- подготовить support runbook по run/request IDs;
- добавить user-facing balance/reservation history;
- выполнить threat model desktop token extraction и gateway abuse.

## 11. Этап J: внешняя beta

Постепенное расширение:

```text
internal users
  -> invited technical users
  -> limited paid beta
  -> general availability
```

Для каждого уровня отдельно задаются:

- active users;
- requests/minute;
- concurrent runs;
- max charge/run;
- daily/user spend;
- OpenRouter key limit;
- support SLA;
- rollback conditions.

## 12. Метрики допуска к следующему этапу

- billing reconciliation backlog = 0;
- duplicate financial effects = 0;
- unresolved model-work state ниже утверждённого порога;
- public model/provider leakage = 0;
- gateway 5xx и disconnect rate в пределах SLO;
- p50/p95 TTFT и duration измеряются;
- routing fallback cost наблюдаем;
- margin рассчитывается по authoritative usage;
- backup/restore проверен;
- rollback routing policy и application deploy протестирован.

## 13. Rollback

Независимые rollback levers:

- отключить выдачу новых access grants на Selectel;
- запретить новые reservations, сохранив settlement для активных runs;
- переключить routing policy на предыдущую версию;
- отключить конкретный model/provider endpoint;
- ограничить OpenRouter key;
- drain inference gateway;
- откатить application image start-first/rollback механизмом Swarm.

Нельзя при rollback:

- удалять holds или inbox/outbox rows;
- откатывать DB migration, теряя новые финансовые записи;
- выключать settlement endpoint раньше завершения active runs;
- повторно запускать ambiguous OpenRouter calls;
- использовать старую версию, не понимающую уже выданные schema versions.

## 14. Первые рабочие задачи

1. Создать private remote для этого OpenCode fork.
2. Утвердить `InferenceAccessGrantV1` и `InferenceSpendGrantV1`.
3. Утвердить reservation/settlement/cancellation schemas.
4. Реализовать signing fixtures на обеих серверных сторонах.
5. Реализовать `inference_billing_holds` и reservation transaction.
6. Создать отдельный inference gateway entrypoint.
7. Добавить fake OpenRouter SSE server.
8. Пройти первый сквозной тест `10 ₽ -> hold 1 ₽ -> debit 0,12 ₽`.
9. Только затем подключать OpenCode adapter.

## 15. Общий definition of done для GA

- Деньги списываются только Selectel и только после signed settlement.
- Inference выполняется только Hetzner и только в пределах spend grant.
- OpenRouter credentials отсутствуют в desktop и Selectel.
- Prompt/code отсутствуют в Selectel billing plane и стандартных логах.
- Все финансовые эффекты идемпотентны.
- Gateway и outbox восстанавливаются после restart.
- Model routing меняется без обновления desktop.
- Пользователь не видит реальный model/provider/routing configuration.
- Desktop local permissions прошли security regression.
- Production deploy, backup, monitoring и rollback документированы и проверены.
