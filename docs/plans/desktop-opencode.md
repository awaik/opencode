# План desktop и OpenCode

## 1. Scope

Этот репозиторий является private product fork OpenCode 2. Он отвечает за
desktop UI, локальный agent runtime, managed LidFly provider и безопасное
выполнение локальных tools.

Upstream:

```text
https://github.com/awaik/opencode
branch: dev
```

Текущий git remote после первоначального клонирования называется `origin` и
указывает на upstream. Перед публикацией proprietary fork нужно:

1. Создать private repository.
2. Переименовать текущий remote в `upstream`.
3. Добавить private repository как `origin`.
4. Не push-ить proprietary изменения в публичный upstream.

## 2. Принципы работы с upstream

- Сохранять минимальный patch surface относительно `upstream/dev`.
- Размещать product-specific код в явно названных LidFly modules.
- Не переписывать общие provider/session subsystems без необходимости.
- Держать branding, billing UI и auth за product boundary.
- Для каждого upstream update прогонять provider, session, permission и desktop
  regression suites.
- Не смешивать upstream sync и product feature в одном PR.

## 3. Предлагаемые модули

Точные package names нужно согласовать с существующими package boundaries, но
логически требуются:

```text
lidfly-auth
  browser login, PKCE, refresh, OS credential storage

lidfly-provider
  managed provider/model, access grant, run credential

lidfly-run-client
  create/status/cancel run, idempotency, public errors

lidfly-billing-ui
  balance, reserved amount, top-up handoff, run cost

lidfly-permissions
  project-root policy and high-risk confirmations

lidfly-branding
  product name, links, legal notices, updates
```

## 4. Managed provider

OpenCode должен видеть одну configuration:

```text
provider_id = lidfly
provider_name = LidFly
model_id = lidfly-code
base_url = https://inference.lidfly.ru/v1
```

Требования:

- provider доступен из managed product configuration;
- произвольный `/connect` не добавляет сторонние providers в product mode;
- model picker либо скрыт, либо содержит единственный `LidFly`;
- project config не может переопределить `baseURL`, headers или model;
- environment variables пользователя не могут подменить product provider;
- OpenRouter-compatible fields не показываются в UI;
- public model limit соответствует гарантированному server contract.

Product restriction не является sandbox для произвольного plugin-кода. Plugin
governance и local tool permissions проектируются отдельно.

## 5. Auth

### Login

1. Desktop создаёт PKCE verifier/challenge.
2. Открывает системный браузер на LidFly authorization endpoint.
3. Принимает callback через зарегистрированный app link или loopback redirect.
4. Обменивает code на access/refresh credentials.
5. Сохраняет refresh credential в Keychain/Windows Credential Manager.
6. Держит access credential только в памяти и обновляет его до истечения.

### Передача credential в OpenCode

Предпочтительный вариант — narrow IPC между desktop host и provider adapter.
OpenCode получает только короткий inference access/run credential, а не основной
refresh token.

Fallback — loopback bridge:

- bind только `127.0.0.1`;
- случайный port;
- per-launch random local secret;
- запрет внешнего интерфейса;
- отсутствие credentials в project config и логах.

## 6. Run lifecycle

Одна пользовательская команда верхнего уровня создаёт один billable run.

```text
user_submit
  -> create inference run
  -> OpenCode provider turns
  -> local tools
  -> provider continuation
  -> complete/cancel/fail
```

Adapter должен передавать:

- opaque run ID/credential;
- уникальный idempotency key каждого provider turn;
- client version;
- protocol version;
- product-controlled phase hint;
- cancel signal.

Любые hints остаются недоверенными на сервере.

Если приложение перезапустилось:

1. Проверить server run status.
2. Не повторять автоматически неоднозначный provider turn.
3. Восстановить локальную session projection.
4. Предложить продолжить только через безопасную новую границу provider turn.

## 7. OpenAI-compatible adapter

Первый контракт использует Chat Completions SSE и должен поддерживать:

- system/user/assistant/tool history;
- text deltas;
- reasoning content в согласованном virtual format;
- fragmented function arguments;
- tool result continuation;
- finish reasons;
- token usage;
- abort/cancel;
- normalized public errors.

Обработать продуктовые ошибки:

```text
401 invalid_or_expired_grant
402 insufficient_balance
409 idempotency_conflict
409 run_already_active
413 context_or_request_too_large
429 rate_limited
503 inference_temporarily_unavailable
```

UI не показывает raw server/OpenRouter error body.

## 8. Billing UX

Пользователь видит:

- общий баланс;
- временно зарезервированную сумму активной задачи;
- фактическую стоимость завершённого run;
- понятное предложение пополнить баланс;
- собственный понижающий max-spend limit, если эта функция будет включена.

Пользователь не видит:

- model/provider;
- routing tier;
- reasoning effort;
- internal token prices;
- OpenRouter;
- generation IDs.

Client-provided max spend может только уменьшать server limit, но не увеличивать
его.

## 9. Permissions и local security

Inference proxy не изолирует shell. Desktop обязан иметь отдельную permission
policy.

### Разрешено по умолчанию

- чтение выбранного project root;
- изменение файлов внутри project root;
- безопасные project-local dev commands;
- read-only git operations.

### С подтверждением

- package installation;
- network commands;
- запуск неизвестного executable;
- команды вне project root;
- `git commit`;
- изменение hooks/config с внешним эффектом;
- длительные или ресурсоёмкие команды.

### Запрещено по умолчанию

- чтение `~/.ssh` и credential stores;
- запись вне project root;
- `git push`;
- destructive filesystem commands;
- системные package managers;
- управление сервисами ОС;
- чтение browser/profile secrets.

Permissions являются approval policy, а не настоящей OS isolation. Для режима
повышенной безопасности потребуется sandbox/container/VM profile.

## 10. Обновления и конфигурация

Desktop binary содержит только:

- public inference URL;
- public key/issuer metadata для grants;
- virtual model capabilities;
- protocol compatibility range;
- локальные security defaults.

Desktop binary не содержит:

- реальные model IDs;
- model prices;
- provider order;
- OpenRouter key;
- routing fallback graph.

Server может менять routing без обновления desktop, пока сохраняется virtual
model protocol/capability contract.

## 11. Тестовый план

### Provider contract

- только LidFly provider/model;
- project config override rejected;
- correct SSE parsing;
- tool calls и continuation;
- reasoning transition;
- cancel и disconnect;
- normalized errors;
- no internal model leakage.

### Auth

- login/callback/PKCE;
- refresh rotation;
- expired/revoked grant;
- credential store failure;
- logout/revoke;
- отсутствие secrets в logs/crash report/project files.

### Run lifecycle

- один user submit -> один run;
- несколько provider turns -> один billing aggregate;
- app restart;
- duplicate idempotency key;
- cancellation before/after model work;
- insufficient balance;
- server outage.

### Permissions

- project-root traversal и symlinks;
- external absolute paths;
- shell escaping;
- destructive commands;
- package install;
- git push;
- credential paths;
- Windows/macOS path variants.

## 12. Этапы PR

1. Product mode boundary и managed provider skeleton.
2. Auth broker и secure credential storage.
3. Run lifecycle/idempotency/cancel.
4. OpenAI-compatible provider integration с fake gateway.
5. Billing UI и public errors.
6. Permission defaults и regression suite.
7. Branding, license notices, updater и packaging.

## 13. Definition of done

- Пользователь не вводит и не получает OpenRouter API key.
- В product mode нельзя выбрать внешний provider/model.
- Secrets не хранятся в project config или plaintext logs.
- Одна пользовательская задача имеет один run ID.
- Cancel проходит от UI до gateway.
- Внешние model/provider names отсутствуют во всех UI/error/history surfaces.
- Permission defaults защищают внешние директории и credentials.
- macOS и Windows packages проходят signing/update проверки.
- Upstream notices и third-party licenses поставляются с продуктом.
