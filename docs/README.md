# LidFly Desktop: план разработки

Этот каталог фиксирует исходную архитектуру закрытого desktop-продукта на базе
OpenCode 2. Документы описывают границы между клиентом, немецким inference data
plane и российским billing control plane. Это планирование, а не описание уже
реализованных возможностей.

## Целевая схема

```text
Пользователь
  -> proprietary desktop + локальный OpenCode
  -> inference.lidfly.ru, Hetzner Германия
  -> server-side routing
  -> OpenRouter

Hetzner
  <-> подписанные reservation/settlement события
Selectel LidFly
  -> пользователи, баланс, holds, transactions, платежи
```

Основные правила:

- OpenRouter API key находится только на немецком сервере.
- Пользователь и OpenCode видят один виртуальный provider/model: `lidfly/lidfly-code`.
- Клиент не выбирает реальную модель, provider, reasoning policy или максимальную цену.
- Selectel является единственным источником истины по деньгам.
- Hetzner является источником истины по выполнению inference и доставляет usage в billing через durable outbox.
- Промпты, исходный код, tool arguments и model output не передаются на Selectel и не попадают в обычные серверные логи.
- Ни один вызов OpenRouter не начинается без предварительно зарезервированного пользовательского бюджета.
- Повторная доставка одного события никогда не создаёт повторное списание.

## Репозитории и ответственность

| Контур | Репозиторий | Ответственность |
|---|---|---|
| Desktop/OpenCode | `/Users/mavbook/projects/lidfly-app/opencode-besed-UI` | UI, локальный агент, provider adapter, permissions, run lifecycle |
| Billing application | `/Users/mavbook/projects/direct-mcp/direct-mcp` | identity, balance, holds, settlements, transactions, billing API |
| Selectel infrastructure | `/Users/mavbook/projects/servers-infra/server-selectel-lidfly` | deploy billing routes/workers, secrets, allowlist, monitoring |
| German inference | `/Users/mavbook/projects/servers-infra/aura-proxy-hetzner` | public gateway, routing, OpenRouter, operational ledger, billing outbox |

## Документы

- [Архитектура и контракты](architecture.md)
- [План Selectel: billing control plane](plans/selectel-billing.md)
- [План Hetzner: inference data plane](plans/hetzner-inference.md)
- [План desktop и OpenCode](plans/desktop-opencode.md)
- [Порядок интеграции и выпуска](plans/integration-rollout.md)

## Рекомендуемый порядок реализации

1. Зафиксировать wire-контракты, state machines и правила списания.
2. Реализовать billing holds и идемпотентный settlement в `direct-mcp`.
3. Реализовать немецкий gateway против fake billing и fake OpenRouter.
4. Соединить два серверных контура локально тестовыми Ed25519-ключами.
5. Добавить managed LidFly provider в OpenCode.
6. Пройти end-to-end тест с fake OpenRouter.
7. Выполнить дешёвый платный canary с отдельным OpenRouter key и тестовым пользователем.
8. Добавить routing tiers и fallback только после финансовой сверки одиночной модели.
9. Завершить desktop permissions и security review до внешней беты.

## Первый вертикальный срез

Первая версия должна доказывать только один сценарий:

```text
Баланс пользователя: 10,00 ₽
  -> reservation: 1,00 ₽
  -> один streaming request к fake OpenRouter
  -> фактическая цена: 0,12 ₽
  -> подписанный settlement
  -> balance: 9,88 ₽
  -> hold закрыт
  -> остаток 0,88 ₽ освобождён
  -> создана ровно одна AI-транзакция
```

Повтор того же settlement обязан вернуть прежний результат без изменения
баланса. До прохождения этого теста нельзя добавлять несколько реальных моделей,
автоматический fallback или повышать production-лимиты.

## Статус

Все этапы в этих документах имеют начальный статус `planned`. Реализация,
production-конфигурация и deploy требуют отдельных задач и проверок в каждом
целевом репозитории.
