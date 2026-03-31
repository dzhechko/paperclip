# Анализ плагинной системы Paperclip

**Дата исследования:** 2026-03-30

---

## 1. Общая архитектура

Плагинная система Paperclip построена на модели **отдельных процессов (worker)**. Каждый плагин запускается как отдельный Node.js-процесс, который общается с хостом через **JSON-RPC 2.0 поверх stdio** (newline-delimited JSON / NDJSON).

### Ключевые компоненты

| Компонент | Расположение | Описание |
|-----------|-------------|----------|
| Plugin SDK | `packages/plugins/sdk/` | Публичный API для авторов плагинов |
| Plugin examples | `packages/plugins/examples/` | 4 примера плагинов |
| create-paperclip-plugin | `packages/plugins/create-paperclip-plugin/` | Генератор скаффолда плагина |
| Server routes | `server/src/routes/plugins.ts` | REST API управления плагинами |
| Plugin UI static | `server/src/routes/plugin-ui-static.ts` | Раздача UI бандлов плагинов |
| Event bus | `server/src/services/plugin-event-bus.ts` | Шина событий для плагинов |
| Lifecycle manager | `server/src/services/plugin-lifecycle.ts` | Управление жизненным циклом |
| Worker manager | `server/src/services/plugin-worker-manager.ts` | Управление worker-процессами |
| Plugin spec | `doc/plugins/PLUGIN_SPEC.md` | Полная спецификация системы |

### Структура пакета SDK (`@paperclipai/plugin-sdk`)

```
@paperclipai/plugin-sdk          -- worker SDK: definePlugin, runWorker, контекст
@paperclipai/plugin-sdk/ui       -- UI SDK: React хуки (usePluginData, usePluginAction, usePluginStream)
@paperclipai/plugin-sdk/testing  -- тестовый harness (in-memory)
@paperclipai/plugin-sdk/bundlers -- пресеты для esbuild/rollup
@paperclipai/plugin-sdk/dev-server -- dev-сервер с hot reload
@paperclipai/plugin-sdk/protocol -- JSON-RPC типы (продвинутый уровень)
```

---

## 2. Жизненный цикл плагина (install, configure, run)

### Машина состояний

```
installed --> ready --> disabled
    |           |         |
    |           |--> error|
    |           v         |
    |    upgrade_pending  |
    |           |         |
    v           v         v
         uninstalled
```

### Этапы жизненного цикла

1. **Установка (install)**
   - Через REST API: `POST /api/plugins/install`
   - Поддерживает npm-пакеты и локальные пути
   - Манифест считывается и валидируется
   - Статус: `installed`

2. **Активация (enable)**
   - `POST /api/plugins/:pluginId/enable`
   - Переводит в статус `ready`, запускает worker-процесс
   - Worker получает вызов `initialize` (JSON-RPC) с манифестом и конфигурацией
   - Затем хост вызывает `setup(ctx)` -- основной хук плагина

3. **Конфигурация (configure)**
   - `GET/POST /api/plugins/:pluginId/config` -- чтение/сохранение конфигурации
   - `POST /api/plugins/:pluginId/config/test` -- тест конфигурации через RPC `validateConfig`
   - Конфигурация описывается JSON Schema в `instanceConfigSchema` манифеста
   - При изменении конфига хост вызывает `onConfigChanged()` или перезапускает worker

4. **Работа (run)**
   - Worker обрабатывает события, джобы, вебхуки, bridge-запросы от UI
   - Хост периодически проверяет здоровье через `health` RPC

5. **Деактивация / удаление**
   - `POST /api/plugins/:pluginId/disable` -- деактивация, worker останавливается
   - `DELETE /api/plugins/:pluginId` -- удаление (мягкое или полное с `?purge=true`)
   - При остановке хост вызывает `onShutdown()` с таймаутом 10 секунд

### Lifecycle hooks (definePlugin)

| Хук | Обязательный | Описание |
|-----|-------------|----------|
| `setup(ctx)` | Да | Основная точка входа. Регистрация обработчиков |
| `onHealth?()` | Нет | Диагностика для health dashboard |
| `onConfigChanged?(newConfig)` | Нет | Горячее применение новой конфигурации |
| `onShutdown?()` | Нет | Очистка перед остановкой |
| `onValidateConfig?(config)` | Нет | Валидация конфига для UI / Test Connection |
| `onWebhook?(input)` | Нет | Обработка входящих вебхуков |

---

## 3. Система событий (hooks/events)

### Доступные события (PLUGIN_EVENT_TYPES)

Плагины могут подписываться на следующие доменные события:

| Событие | Описание |
|---------|----------|
| `company.created` | Создана компания |
| `company.updated` | Обновлена компания |
| `project.created` | Создан проект |
| `project.updated` | Обновлен проект |
| `project.workspace_created` | Создано рабочее пространство |
| `project.workspace_updated` | Обновлено рабочее пространство |
| `project.workspace_deleted` | Удалено рабочее пространство |
| **`issue.created`** | **Создана задача (issue)** |
| **`issue.updated`** | **Обновлена задача** |
| `issue.comment.created` | Создан комментарий к задаче |
| `agent.created` | Создан агент |
| `agent.updated` | Обновлен агент |
| `agent.status_changed` | Изменен статус агента |
| `agent.run.started` | Запущен прогон агента |
| `agent.run.finished` | Завершен прогон агента |
| `agent.run.failed` | Прогон агента провалился |
| `agent.run.cancelled` | Прогон агента отменен |
| **`approval.created`** | **Создан запрос на одобрение** |
| **`approval.decided`** | **Решение по одобрению принято** |
| `cost_event.created` | Создано событие стоимости |
| `activity.logged` | Записана активность |

### Механизм подписки

```typescript
// Подписка на конкретное событие
ctx.events.on("issue.created", async (event) => {
  // event.eventId, event.eventType, event.entityId, event.companyId, event.payload
});

// С серверным фильтром (фильтрация на стороне хоста, до передачи worker-у)
ctx.events.on("issue.created", { companyId: "...", projectId: "..." }, async (event) => {});

// Plugin-to-plugin события (автоматический namespace: "plugin.<pluginId>.<name>")
ctx.events.emit("sync-done", companyId, { count: 42 });
ctx.events.on("plugin.acme.linear.sync-done", async (event) => {});

// Wildcard подписка на plugin события
ctx.events.on("plugin.acme.linear.*", async (event) => {});
```

### Event Bus (серверная часть)

Файл: `server/src/services/plugin-event-bus.ts`

- In-process шина событий
- Каждый плагин получает scoped handle через `bus.forPlugin(pluginId)`
- Подписки изолированы по плагинам
- Поддержка wildcard `.*` для plugin-namespace
- Ошибки в обработчиках собираются, не блокируют доставку другим плагинам
- Серверная фильтрация по `companyId`, `projectId`, `agentId`

---

## 4. Возможности плагинов (capabilities)

### Система capability (разрешений)

Каждый плагин в манифесте декларирует нужные capability. Хост проверяет их перед выполнением каждого вызова worker-->host. Без capability вызов отвергается с ошибкой `CAPABILITY_DENIED`.

### Полный список capability

**Data Read:**
- `companies.read`, `projects.read`, `project.workspaces.read`
- `issues.read`, `issue.comments.read`, `issue.documents.read`
- `agents.read`, `goals.read`, `activity.read`, `costs.read`

**Data Write:**
- `issues.create`, `issues.update`
- `issue.comments.create`, `issue.documents.write`
- `agents.pause`, `agents.resume`, `agents.invoke`
- `agent.sessions.create/list/send/close`
- `goals.create`, `goals.update`
- `activity.log.write`, `metrics.write`

**Plugin State:**
- `plugin.state.read`, `plugin.state.write`

**Runtime / Integration:**
- `events.subscribe`, `events.emit`
- `jobs.schedule`, `webhooks.receive`
- **`http.outbound`** -- исходящие HTTP-запросы
- `secrets.read-ref`

**Agent Tools:**
- `agent.tools.register`

**UI:**
- `instance.settings.register`
- `ui.sidebar.register`, `ui.page.register`, `ui.detailTab.register`
- `ui.dashboardWidget.register`, `ui.commentAnnotation.register`, `ui.action.register`

---

## 5. Ответы на ключевые вопросы

### 5.1. Можно ли слушать создание/обновление issue?

**ДА.** Полностью поддерживается через события:

```typescript
ctx.events.on("issue.created", async (event) => { /* ... */ });
ctx.events.on("issue.updated", async (event) => { /* ... */ });
ctx.events.on("issue.comment.created", async (event) => { /* ... */ });
```

Capability: `events.subscribe`

### 5.2. Можно ли слушать запросы на одобрение (approvals)?

**ДА.** Есть два события:

```typescript
ctx.events.on("approval.created", async (event) => { /* ... */ });
ctx.events.on("approval.decided", async (event) => { /* ... */ });
```

Capability: `events.subscribe`

### 5.3. Можно ли слушать heartbeat агентов?

**ЧАСТИЧНО.** Прямого события `agent.heartbeat` нет в `PLUGIN_EVENT_TYPES`. Однако есть:

- `agent.status_changed` -- изменение статуса агента
- `agent.run.started` / `agent.run.finished` / `agent.run.failed` / `agent.run.cancelled`

В системе существует понятие heartbeat (в shared-типах: `HeartbeatRun`, `HeartbeatRunEvent`, `lastHeartbeatAt` у агента), но это внутренний механизм, **не выставленный как plugin-событие**. Плагин может получить `lastHeartbeatAt` из объекта `Agent` через `ctx.agents.get()`.

### 5.4. Можно ли отправлять внешние уведомления?

**ДА.** Несколько механизмов:

1. **`ctx.http.fetch()`** -- прямые HTTP-запросы к любым внешним сервисам (Telegram Bot API, Slack API, email API, любые вебхуки). Capability: `http.outbound`

2. **Scheduled jobs** -- периодические задачи для polling/push. Capability: `jobs.schedule`

3. **Webhooks (входящие)** -- принимать входящие вебхуки по URL `POST /api/plugins/:pluginId/webhooks/:endpointKey`. Capability: `webhooks.receive`

```typescript
// Пример отправки в Telegram
const config = await ctx.config.get();
const token = await ctx.secrets.resolve(config.botTokenRef as string);
await ctx.http.fetch(`https://api.telegram.org/bot${token}/sendMessage`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ chat_id: config.chatId, text: "Новая задача!" }),
});
```

### 5.5. Можно ли создавать/изменять issues через API?

**ДА.** Полноценный CRUD:

```typescript
// Создание
const issue = await ctx.issues.create({
  companyId, projectId, title: "Bug fix", description: "...", priority: "high"
});

// Обновление
await ctx.issues.update(issueId, { status: "in_progress", title: "Updated" }, companyId);

// Чтение
const issues = await ctx.issues.list({ companyId, projectId, limit: 50 });
const issue = await ctx.issues.get(issueId, companyId);

// Комментарии
await ctx.issues.createComment(issueId, "Обновление статуса", companyId);
const comments = await ctx.issues.listComments(issueId, companyId);

// Документы к задачам
await ctx.issues.documents.upsert({ issueId, key: "plan", body: "...", companyId });
```

Capabilities: `issues.create`, `issues.update`, `issues.read`, `issue.comments.create`, `issue.comments.read`, `issue.documents.write`, `issue.documents.read`

### 5.6. Можно ли approve/reject approvals из плагина?

**НЕТ.** В текущем SDK (`WorkerToHostMethods`) **нет методов для управления approvals**. Плагин может:
- **Слушать** события `approval.created` и `approval.decided`
- Но **не может** программно одобрить/отклонить approval

Approvals управляются через REST API сервера (`server/src/routes/approvals.ts`), но этот API не выставлен в plugin SDK. Теоретически плагин может использовать `ctx.http.fetch()` чтобы вызвать REST endpoint напрямую (если доступен localhost), но это обходной путь, не предусмотренный архитектурой.

---

## 6. Полный Plugin Context (ctx)

Объект `PluginContext`, передаваемый в `setup(ctx)`:

| Клиент | Описание |
|--------|----------|
| `ctx.manifest` | Манифест плагина (read-only) |
| `ctx.config` | Чтение конфигурации плагина |
| `ctx.events` | Подписка на события + эмит plugin-событий |
| `ctx.jobs` | Регистрация обработчиков scheduled jobs |
| `ctx.launchers` | Регистрация UI launcher-ов |
| `ctx.http` | Исходящие HTTP-запросы |
| `ctx.secrets` | Разрешение секретных ссылок |
| `ctx.activity` | Запись в лог активности |
| `ctx.state` | Key-value состояние плагина (scoped) |
| `ctx.entities` | Сущности плагина (upsert/list) |
| `ctx.projects` | Чтение проектов и workspaces |
| `ctx.companies` | Чтение компаний |
| `ctx.issues` | CRUD задач, комментариев, документов |
| `ctx.agents` | Чтение/управление агентами (pause/resume/invoke) |
| `ctx.agents.sessions` | Chat-сессии с агентами (create/send/close) |
| `ctx.goals` | CRUD целей (goals) |
| `ctx.data` | getData handlers для UI |
| `ctx.actions` | performAction handlers для UI |
| `ctx.streams` | SSE push от worker к UI |
| `ctx.tools` | Регистрация инструментов для агентов |
| `ctx.metrics` | Запись метрик |
| `ctx.logger` | Структурированный логгер |

---

## 7. Worker <--> Host протокол (JSON-RPC 2.0)

### Host --> Worker методы

| Метод | Обязательный | Описание |
|-------|-------------|----------|
| `initialize` | Да | Инициализация с манифестом и конфигурацией |
| `health` | Да | Проверка здоровья |
| `shutdown` | Да | Грациозная остановка |
| `validateConfig` | Нет | Валидация конфигурации |
| `configChanged` | Нет | Уведомление о смене конфига |
| `onEvent` | Нет | Доставка доменного события |
| `runJob` | Нет | Выполнение scheduled job |
| `handleWebhook` | Нет | Обработка входящего вебхука |
| `getData` | Нет | UI bridge: получение данных |
| `performAction` | Нет | UI bridge: выполнение действия |
| `executeTool` | Нет | Вызов инструмента от агента |

### Worker --> Host методы

Полный перечень (через `WorkerToHostMethods`):

- `config.get`
- `state.get/set/delete`
- `entities.upsert/list`
- `events.emit/subscribe`
- `http.fetch`
- `secrets.resolve`
- `activity.log`
- `metrics.write`
- `log`
- `companies.list/get`
- `projects.list/get/listWorkspaces/getPrimaryWorkspace/getWorkspaceForIssue`
- `issues.list/get/create/update/listComments/createComment`
- `issues.documents.list/get/upsert/delete`
- `agents.list/get/pause/resume/invoke`
- `agents.sessions.create/list/sendMessage/close`
- `goals.list/get/create/update`

### Worker --> Host нотификации (fire-and-forget)

- `streams.emit` -- push события в SSE канал для UI
- `streams.open` -- открытие канала
- `streams.close` -- закрытие канала

---

## 8. REST API плагинов (server/src/routes/plugins.ts)

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/plugins` | Список всех плагинов |
| GET | `/plugins/examples` | Встроенные примеры |
| GET | `/plugins/ui-contributions` | UI слоты от ready-плагинов |
| GET | `/plugins/:pluginId` | Получить плагин по ID или key |
| POST | `/plugins/install` | Установить плагин |
| DELETE | `/plugins/:pluginId` | Удалить плагин |
| POST | `/plugins/:pluginId/enable` | Включить |
| POST | `/plugins/:pluginId/disable` | Выключить |
| GET | `/plugins/:pluginId/health` | Диагностика |
| POST | `/plugins/:pluginId/upgrade` | Обновить |
| GET | `/plugins/:pluginId/jobs` | Список джобов |
| POST | `/plugins/:pluginId/jobs/:jobId/trigger` | Запустить джоб |
| POST | `/plugins/:pluginId/webhooks/:endpointKey` | Входящий вебхук |
| GET | `/plugins/tools` | Список инструментов |
| POST | `/plugins/tools/execute` | Выполнить инструмент |
| GET/POST | `/plugins/:pluginId/config` | Чтение/запись конфигурации |
| POST | `/plugins/:pluginId/config/test` | Тест конфигурации |
| POST | `/plugins/:pluginId/bridge/data` | UI bridge: getData |
| POST | `/plugins/:pluginId/bridge/action` | UI bridge: performAction |
| GET | `/plugins/:pluginId/bridge/stream/:channel` | SSE стрим |
| GET | `/plugins/:pluginId/dashboard` | Дашборд здоровья |

---

## 9. UI система плагинов

### Доступные UI слоты (extension points)

| Тип слота | Описание |
|-----------|----------|
| `page` | Полноценная страница |
| `detailTab` | Вкладка на странице деталей (issue/project) |
| `taskDetailView` | Кастомный вид задачи |
| `dashboardWidget` | Виджет на дашборде |
| `sidebar` | Элемент в боковой панели |
| `sidebarPanel` | Панель в боковой панели |
| `projectSidebarItem` | Элемент в навигации проекта |
| `globalToolbarButton` | Кнопка в глобальном тулбаре |
| `toolbarButton` | Кнопка в тулбаре сущности |
| `contextMenuItem` | Пункт контекстного меню |
| `commentAnnotation` | Аннотация к комментарию |
| `commentContextMenuItem` | Пункт меню комментария |
| `settingsPage` | Страница настроек плагина |

### Launchers

Плагины могут регистрировать launcher-ы -- UI точки входа (модалки, drawer-ы), с настройками размера, зоны размещения, типа действия (`openModal`).

### UI hooks (React)

- `usePluginData(key, params)` -- запрос данных от worker
- `usePluginAction(key)` -- вызов действия в worker
- `usePluginStream(channel)` -- SSE подписка на стрим от worker
- `useHostContext()` -- получение контекста хоста

---

## 10. Примеры плагинов

### plugin-hello-world-example
Минимальный плагин. Только `setup()` с логом и `onHealth()`. Образец для быстрого старта.

### plugin-file-browser-example
Добавляет навигацию "Files" в проект и вкладку file browser.

### plugin-kitchen-sink-example
**Полноценная демонстрация всех возможностей:**
- 22+ UI слота (page, sidebar, dashboard widget, detail tabs, context menus, launchers)
- Подписка на события: `issue.created`, `issue.updated`, plugin-to-plugin
- Scheduled job (heartbeat каждые 15 минут)
- Webhook endpoint
- 3 agent tools (echo, company summary, create issue)
- State management (scoped key-value)
- Entity store
- HTTP fetch, secrets resolve
- Process execution (curated commands)
- Streams (SSE push)
- Metrics

### plugin-authoring-smoke-example
Smoke-тест для проверки процесса authoring-а.

---

## 11. Вебхуки (webhooks)

### Входящие вебхуки

Плагин может принимать входящие HTTP-запросы через:
```
POST /api/plugins/:pluginId/webhooks/:endpointKey
```

Вебхук-эндпоинты декларируются в манифесте:
```typescript
webhooks: [{
  endpointKey: "demo",
  displayName: "Demo Ingest",
  description: "Accepts arbitrary webhook payloads"
}]
```

Обработчик в worker:
```typescript
async onWebhook(input: PluginWebhookInput) {
  // input.endpointKey, input.headers, input.rawBody, input.parsedBody, input.requestId
}
```

Требуется capability: `webhooks.receive`

### Исходящие вебхуки / HTTP

Через `ctx.http.fetch()` (capability: `http.outbound`) можно отправлять запросы на любые внешние URL, включая webhook-endpoint-ы сторонних сервисов.

---

## 12. Паттерны Notification/Communication

### Текущее состояние

**В кодовой базе НЕ найдено готовых плагинов уведомлений.** Нет реализованных плагинов для Telegram, Slack, Discord или email. Поиск по `packages/plugins/` не выявил ни одного упоминания этих сервисов (за исключением общих документационных комментариев в SDK).

### Как реализовать уведомления

Архитектура полностью поддерживает создание notification-плагина:

1. **Подписка на события** (`events.subscribe`) -- ловить `issue.created`, `approval.created`, `agent.run.failed` и т.д.
2. **Отправка уведомлений** (`http.outbound`) -- вызовы Telegram Bot API, Slack Webhook, SendGrid и пр.
3. **Секреты** (`secrets.read-ref`) -- безопасное хранение API-ключей/токенов
4. **Конфигурация** (`instanceConfigSchema`) -- настройки через UI (chat ID, канал, фильтры)
5. **UI виджеты** -- настройки плагина, статус отправки
6. **Webhooks** (`webhooks.receive`) -- приём обратных callback-ов от внешних сервисов
7. **State** (`plugin.state`) -- хранение состояния (последнее отправленное уведомление, дедупликация)
8. **Jobs** (`jobs.schedule`) -- периодические сводки/дайджесты

---

## 13. Что отсутствует в текущей системе

1. **Approval management API** -- плагин не может программно approve/reject approvals
2. **Agent heartbeat events** -- нет `agent.heartbeat` в PLUGIN_EVENT_TYPES (есть `agent.run.*`)
3. **Готовые notification-плагины** -- ни одного плагина для внешних уведомлений
4. **Shared React component kit** -- UI плагины используют обычные React-компоненты, нет предоставленной библиотеки UI-компонентов от хоста
5. **Assets API** -- `ctx.assets` упоминается как "не поддерживается в текущей сборке"
6. **Multi-instance deployment** -- runtime-установка плагинов работает только на single-node

---

## 14. Выводы для создания notification-плагина

Система плагинов Paperclip **хорошо подходит** для создания плагина уведомлений (например, Telegram). Все необходимые строительные блоки присутствуют:

- Богатый набор событий для подписки (issues, approvals, agents, goals)
- Полноценный HTTP-клиент для исходящих запросов
- Управление секретами для API-токенов
- Конфигурируемые настройки через UI
- State management для отслеживания состояния
- Scheduled jobs для дайджестов/healthcheck
- Входящие webhooks для callback-ов
- UI слоты для интерфейса настроек

**Ограничения:**
- Нельзя approve/reject approvals -- только уведомлять о них
- Нет прямого доступа к heartbeat-событиям агентов
- Нет двусторонней интеграции (Telegram --> Paperclip actions) без дополнительного REST-слоя
