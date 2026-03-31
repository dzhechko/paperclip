# HITL через Telegram для Paperclip: анализ Elite Problem Solver

## MODULE 1: First Principles -- Что фундаментально требуется?

"Человек управляет AI-агентом через Telegram" распадается на пять неснижаемых требований:

### 1.1 Двунаправленная связь Telegram <-> Paperclip

- **Telegram -> Paperclip:** Telegram Bot API доставляет сообщения пользователя через webhook или long polling. Paperclip должен принять эти сообщения и маршрутизировать их к нужному агенту/компании.
- **Paperclip -> Telegram:** При возникновении событий (approval.created, agent.run.failed, agent.status_changed) Paperclip вызывает Telegram Bot API `sendMessage` для уведомления конкретного пользователя.

### 1.2 Привязка "человек -> агенты"

- Каждый Telegram-пользователь (chat_id) привязан к одной или нескольким парам (companyId, agentId).
- Один агент может быть назначен нескольким операторам (escalation chain).
- Один оператор может управлять несколькими агентами в разных компаниях.

### 1.3 Уведомления (Paperclip -> Human)

- Агент создал approval -- человек получает сообщение в Telegram с кнопками "Approve" / "Reject" / "Comment".
- Агент упал (agent.run.failed) -- уведомление.
- Агент завершил задачу -- сводка.
- Бюджет исчерпан (auto-pause) -- алерт.

### 1.4 Команды (Human -> Paperclip)

- `/approve <id>` -- одобрить approval
- `/reject <id> [причина]` -- отклонить
- `/pause <agent>` -- поставить агента на паузу
- `/resume <agent>` -- возобновить
- `/status` -- текущий статус всех подконтрольных агентов
- `/invoke <agent> <prompt>` -- дать агенту задание
- Inline-кнопки в уведомлениях -- тот же эффект без ввода команд

### 1.5 Контекст (Human видит работу агента)

- При запросе `/status` -- список агентов со статусами, текущими задачами, последним activity.
- При approval -- payload с описанием, что агент хочет сделать.
- По запросу `/issues <agent>` -- список задач агента.

---

## MODULE 2: Root Cause Analysis -- Что есть vs что отсутствует

### Что уже есть в Paperclip (подтверждено кодом):

| Возможность | Где в коде | Готовность |
|---|---|---|
| **REST API для approvals** | `server/src/routes/approvals.ts` -- CRUD, approve, reject, revision | Полная |
| **REST API для агентов** | `server/src/routes/agents.ts` -- list, get, pause, resume, invoke, sessions | Полная |
| **REST API для issues** | `server/src/routes/issues.ts` -- list, get, create, update, comments | Полная |
| **Event system (plugin)** | `packages/shared/src/constants.ts` -- 24 типа событий включая `approval.created`, `approval.decided`, `agent.status_changed`, `agent.run.started/finished/failed` | Полная |
| **WebSocket live events** | `server/src/realtime/live-events-ws.ts` + `server/src/services/live-events.ts` -- real-time push по companyId | Полная |
| **Plugin SDK** | `packages/plugins/sdk/` -- definePlugin, events.on, webhooks, jobs, tools, http.fetch, state, entities, agents.* | Полная |
| **Webhook ingestion** | `server/src/routes/plugins.ts` строка ~1895 -- `POST /api/plugins/:pluginId/webhooks/:endpointKey` | Полная |
| **Agent sessions** | SDK: `agents.sessions.create/sendMessage/close` | Полная |
| **Аутентификация** | Bearer token (agent API key), session-based (board), local trusted | Полная |

### Что отсутствует:

| Отсутствует | Критичность |
|---|---|
| **Telegram Bot** -- бот не создан, нет интеграции | Ядро задачи |
| **Маппинг chat_id -> (companyId, agentId[])** | Ядро задачи |
| **Преобразование Telegram-команд в API-вызовы** | Ядро задачи |
| **Push-уведомления при событиях** | Ядро задачи |
| **Inline keyboard для approvals** | UX |

### Ключевой вывод:

Вся серверная инфраструктура Paperclip **уже готова**. REST API покрывает 100% необходимых операций. Plugin SDK предоставляет подписку на события, webhook ingestion, state storage, HTTP outbound, и прямые вызовы agents.pause/resume/invoke. **Задача сводится к созданию плагина-моста между Telegram Bot API и существующим Plugin SDK.**

---

## MODULE 3: SCQA Framework

### Situation (Ситуация)
Paperclip -- control plane для AI-агентных компаний. Board members (человеческие операторы) управляют агентами через web UI: одобряют действия, ставят на паузу, назначают задачи. Система событий и approvals уже зрелая.

### Complication (Осложнение)
Board members привязаны к браузеру. Approval может ждать часами, пока человек откроет UI. Агент может упасть ночью -- никто не узнает до утра. На мобильном web UI неоптимален. Множественные компании/агенты умножают проблему -- оператор должен мониторить несколько дашбордов.

### Question (Вопрос)
Как обеспечить мгновенный mobile-first канал HITL-взаимодействия, позволяющий нескольким операторам управлять разными агентами, не дублируя бизнес-логику Paperclip?

### Answer (Ответ)
Paperclip-плагин `plugin-telegram-hitl`, использующий Plugin SDK. Плагин подписывается на события, отправляет уведомления в Telegram, принимает webhook от Telegram Bot API, транслирует команды в вызовы SDK (agents.pause, agents.resume, agents.invoke, approvals API через http.fetch).

---

## MODULE 6: TRIZ -- Разрешение противоречий

### Техническое противоречие #1

> "Мы хотим **богатый governance UI** (approvals с payload, деревья целей, org chart) НО мы хотим **легковесный Telegram-интерфейс**"

**Принцип TRIZ: Сегментация (Principle 1) + Вынесение (Principle 2)**

Решение: Не воспроизводить web UI в Telegram. Telegram -- это **notification + quick action** канал. Для глубокого анализа отправлять deep link на web UI: `Approve this? [Approve] [Reject] [View in Paperclip ->]`. Сегментировать интерфейс: критичные действия (approve/reject/pause/resume) -- в Telegram, сложные (настройка агента, org chart) -- web only.

### Техническое противоречие #2

> "Мы хотим **безопасность** (только авторизованные операторы) НО мы хотим **простую настройку** (не заставлять пользователя конфигурировать OAuth)"

**Принцип TRIZ: Посредник (Principle 24) + Самообслуживание (Principle 25)**

Решение: Паттерн "pairing code". Оператор в web UI Paperclip генерирует одноразовый код. Отправляет боту `/pair <code>` в Telegram. Плагин верифицирует код через plugin state, привязывает chat_id к userId + companyId. Никакого OAuth, но безопасно -- код одноразовый и time-limited.

### Техническое противоречие #3

> "Мы хотим **real-time уведомления** НО **Plugin SDK не имеет push-подписки на live events WebSocket**"

**Принцип TRIZ: Предварительного действия (Principle 10)**

Решение: Plugin SDK предоставляет `ctx.events.on("approval.created", ...)` -- это уже push-подписка на доменные события. LiveEvents WebSocket (`/api/companies/:id/events/ws`) -- для UI, не для плагинов. Плагину не нужен WebSocket -- Plugin SDK доставляет события через `onEvent` RPC. Противоречия фактически нет: нужная инфраструктура уже существует в виде plugin event system.

---

## MODULE 9: Synthesis -- Конкретная архитектура

### 9.1 Компонентная схема

```
                    Telegram Cloud
                         |
              [Telegram Bot API]
                    |         ^
                    v         |
    webhook POST             sendMessage / editMessage
    /api/plugins/             / answerCallbackQuery
    telegram-hitl/            |
    webhooks/tg-update        |
                    |         |
         +----------+---------+----------+
         |     plugin-telegram-hitl       |
         |  (Paperclip Plugin Worker)     |
         |                                |
         |  onWebhook(tg-update)          |
         |    -> parse command/callback   |
         |    -> ctx.agents.pause(...)    |
         |    -> ctx.http.fetch(approve)  |
         |                                |
         |  onEvent(approval.created)     |
         |    -> lookup chat_ids          |
         |    -> ctx.http.fetch(tg API)   |
         |    -> send with inline kbd     |
         |                                |
         |  state: chat_id mappings       |
         |  entities: pairing codes       |
         +--------------------------------+
                    |
         [Paperclip Plugin Host / SDK]
                    |
         [Paperclip Server APIs]
```

### 9.2 Manifest плагина

```typescript
const manifest: PaperclipPluginManifestV1 = {
  id: "telegram-hitl",
  apiVersion: 1,
  version: "0.1.0",
  displayName: "Telegram HITL Bridge",
  description: "Управление агентами через Telegram: уведомления, approvals, команды",
  author: "Paperclip",
  categories: ["connector", "automation"],
  capabilities: [
    // Чтение данных
    "companies.read",
    "agents.read",
    "issues.read",
    "issue.comments.read",
    "goals.read",
    // Управление агентами
    "agents.pause",
    "agents.resume",
    "agents.invoke",
    "agent.sessions.create",
    "agent.sessions.send",
    "agent.sessions.close",
    // Подписка на события
    "events.subscribe",
    // Хранение состояния (маппинги chat_id)
    "plugin.state.read",
    "plugin.state.write",
    // Исходящий HTTP (Telegram Bot API)
    "http.outbound",
    // Секреты (Telegram Bot Token)
    "secrets.read-ref",
    // Прием webhook (Telegram updates)
    "webhooks.receive",
    // UI для настройки
    "ui.page.register",
    "ui.sidebar.register",
    // Activity logging
    "activity.log.write",
  ],
  entrypoints: {
    worker: "./dist/worker.js",
    ui: "./dist/ui",
  },
  instanceConfigSchema: {
    type: "object",
    properties: {
      botTokenRef: {
        type: "string",
        title: "Telegram Bot Token (secret ref)",
        description: "Ссылка на секрет с токеном бота",
      },
      defaultCompanyId: {
        type: "string",
        title: "Default Company ID",
        description: "Компания по умолчанию для single-company setups",
      },
      notifyOnApproval: { type: "boolean", default: true },
      notifyOnAgentFailure: { type: "boolean", default: true },
      notifyOnAgentComplete: { type: "boolean", default: false },
      notifyOnBudgetExhausted: { type: "boolean", default: true },
    },
    required: ["botTokenRef"],
  },
  webhooks: [
    {
      endpointKey: "tg-update",
      displayName: "Telegram Update Webhook",
      description: "Принимает updates от Telegram Bot API",
    },
  ],
  ui: {
    slots: [
      {
        type: "settingsPage",
        id: "telegram-hitl-settings",
        displayName: "Telegram HITL",
        exportName: "SettingsPage",
      },
    ],
  },
};
```

### 9.3 Модель данных (plugin state + entities)

**Pairing (entities, entityType: "tg-pairing"):**
```
{
  externalId: "<pairing-code>",  // 6-символьный код
  data: {
    userId: "user-uuid",         // Paperclip userId
    companyId: "company-uuid",
    agentIds: ["agent-1", "agent-2"],
    expiresAt: "2026-03-30T12:00:00Z",
    used: false
  }
}
```

**Operator binding (state, scopeKind: "instance"):**
```
stateKey: "operator:<chat_id>"
value: {
  chatId: 123456789,
  userId: "user-uuid",
  bindings: [
    {
      companyId: "company-uuid",
      agentIds: ["agent-1", "agent-2"],  // пустой = все агенты компании
      role: "operator"                    // или "observer" (только чтение)
    }
  ],
  pairedAt: "2026-03-30T10:00:00Z"
}
```

**Reverse index (state, scopeKind: "company"):**
```
stateKey: "tg-operators:<companyId>"
value: {
  operators: [
    { chatId: 123456789, userId: "user-uuid", agentIds: ["agent-1"] },
    { chatId: 987654321, userId: "user-uuid-2", agentIds: ["agent-2", "agent-3"] },
  ]
}
```

### 9.4 Потоки данных

#### Поток 1: Pairing (привязка оператора)

1. Оператор открывает Settings > Telegram HITL в web UI
2. Нажимает "Generate Pairing Code", выбирает агентов
3. UI вызывает `performAction("generate-pairing", { agentIds, companyId })`
4. Worker генерирует 6-символьный код, сохраняет через `ctx.entities.upsert`
5. UI показывает код + QR-ссылку `https://t.me/paperclip_hitl_bot?start=<code>`
6. Оператор открывает бота, отправляет `/start <code>`
7. Worker получает webhook, валидирует код, создает binding через `ctx.state.set`

#### Поток 2: Approval notification

1. Агент создает approval -> Paperclip emits event `approval.created`
2. Plugin worker получает событие через `ctx.events.on("approval.created", handler)`
3. Handler читает `payload.companyId`, `payload.requestedByAgentId`
4. Ищет операторов через reverse index: `ctx.state.get({ stateKey: "tg-operators:<companyId>" })`
5. Фильтрует по agentIds -- находит нужных операторов
6. Для каждого оператора отправляет Telegram-сообщение через `ctx.http.fetch("https://api.telegram.org/bot<token>/sendMessage", { chat_id, text, reply_markup: inline_keyboard })`
7. Inline keyboard: `[Approve | approvalId]` `[Reject | approvalId]` `[View in Paperclip]`

#### Поток 3: Approval decision через Telegram

1. Оператор нажимает "Approve" (callback_query)
2. Telegram отправляет update на webhook `tg-update`
3. Worker парсит callback_data: `approve:<approvalId>`
4. Worker вызывает `ctx.http.fetch("POST /api/approvals/<id>/approve", { decidedByUserId, decisionNote: "Approved via Telegram" })`
5. Worker обновляет Telegram-сообщение через editMessageText (статус: "Approved by you")
6. Paperclip обрабатывает approval -> wakeup agent (уже реализовано в `approvals.ts` строка 151)

#### Поток 4: Команда /pause

1. Оператор отправляет `/pause agent-name`
2. Webhook -> Worker парсит команду
3. Worker резолвит имя агента в agentId через `ctx.agents.list({ companyId })`
4. Worker вызывает `ctx.agents.pause({ agentId, companyId })`
5. Worker отправляет confirmation в Telegram

### 9.5 Обработка множественных операторов

Ключевой аспект задачи: **разные люди назначены на разных агентов**.

Реализация через двухуровневую маршрутизацию:

1. **При событии** -- определяем `companyId` и `agentId` из payload события
2. **Lookup** -- по reverse index `tg-operators:<companyId>` находим всех операторов этой компании
3. **Фильтрация** -- если у оператора `agentIds` не пустой, проверяем совпадение с `agentId` события; если пустой -- оператор получает все уведомления компании
4. **Рассылка** -- отправляем уведомление всем подходящим операторам

Это позволяет:
- Оператору A следить за агентами 1, 2
- Оператору B следить за агентами 3, 4, 5
- Оператору C (CTO) следить за всеми агентами компании
- Все они получают только релевантные уведомления

### 9.6 Безопасность

| Угроза | Митигация |
|---|---|
| Подмена chat_id | Pairing code верифицируется через plugin state; webhook подпись через Telegram Bot API secret token |
| Неавторизованный approve | Проверка, что chat_id привязан к userId с правами board member в companyId |
| Replay attack на callback | Idempotency: проверка статуса approval перед action (уже реализовано в `approvals.ts` строка 44) |
| Bot token leak | Хранение через `ctx.secrets.resolve(config.botTokenRef)` -- Paperclip secret management |
| Telegram webhook forgery | Верификация `X-Telegram-Bot-Api-Secret-Token` header в `onWebhook` |

### 9.7 Что НЕ нужно делать

1. **Не дублировать бизнес-логику** -- все через SDK-вызовы и REST API, не через прямой доступ к БД
2. **Не делать полноценный UI в Telegram** -- это канал quick actions, не замена web UI
3. **Не реализовывать свою систему auth** -- использовать pairing + проверку через существующий userId
4. **Не подписываться на WebSocket** -- Plugin SDK event system (`ctx.events.on`) уже решает задачу push-уведомлений

### 9.8 Ограничение SDK и workaround

**Проблема:** SDK предоставляет `agents.pause`, `agents.resume`, `agents.invoke`, но НЕ предоставляет прямой метод для `approvals.approve/reject`. В `WorkerToHostMethods` (файл `protocol.ts`) нет approval-методов.

**Workaround:** Использовать `ctx.http.fetch` для вызова REST API Paperclip:
```typescript
await ctx.http.fetch(`http://localhost:3100/api/approvals/${approvalId}/approve`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${boardToken}`
  },
  body: JSON.stringify({ decidedByUserId: userId, decisionNote: "Via Telegram" })
});
```

Это требует, чтобы у плагина был board-level токен для компании. Альтернатива: добавить в SDK метод `approvals.resolve` -- одно изменение в `protocol.ts` WorkerToHostMethods + реализация на стороне host.

### 9.9 Оценка сложности

| Компонент | Оценка |
|---|---|
| Manifest + scaffold | 2h |
| Webhook handler (parse Telegram updates) | 4h |
| Pairing flow (generate + verify) | 3h |
| Event subscriptions + notification dispatch | 4h |
| Inline keyboard + callback handling | 3h |
| Command parser (/pause, /resume, /status, /invoke) | 4h |
| Settings UI (React) | 4h |
| Approval integration (с workaround через http.fetch) | 3h |
| Тестирование | 4h |
| **Итого** | **~31h (4 дня)** |

### 9.10 Рекомендация по реализации

Поэтапная стратегия:

**Phase 1 (MVP):** Notification-only. Подписка на `approval.created` и `agent.run.failed`. Отправка уведомлений в Telegram. Pairing flow. Deep links на web UI.

**Phase 2:** Inline actions. Approve/reject через callback buttons. /status и /pause команды.

**Phase 3:** Full HITL. /invoke, agent sessions через Telegram, thread-based диалоги с агентами.

---

## Итог

Paperclip Plugin SDK предоставляет **все необходимые строительные блоки**: event subscriptions, webhook ingestion, agent management, state storage, HTTP outbound. Единственный gap -- отсутствие approval-методов в SDK (обходится через `http.fetch`). Архитектурно задача решается **одним плагином** без модификации core Paperclip, с полной поддержкой множественных операторов через двухуровневую маршрутизацию по (companyId, agentIds[]).
