> Sanitized mirror of Forgejo `serhii/comms-net-pi`. Source code is not published here.
>
> Commit texts: `commits/`. Need the code? Email: sergeyalpatov1@gmail.com
> Source: Forgejo `serhii/comms-net-pi` | Synced: 2026-10-05T01:37:38Z

---

# coms-net-pi

Связь между pi-агентами через лёгкий HTTP/SSE хаб. Позволяет нескольким
инстансам pi coding-agent общаться друг с другом — на одной машине или по сети.

Это порт расширения `coms-net` + хаба из репозитория
[`disler/pi-vs-claude-code`](https://github.com/disler/pi-vs-claude-code), адаптированный
под пакетный namespace `@earendil-works/pi-*`, а хаб-сервер **переписан с Bun на Node**
(Bun не нужен — работает на Node ≥ 20).

## Как это работает (модель)

Hub-and-spoke, не чистая mesh-сеть:

```
  Агент A ──┐                        ┌── Агент B
            │  REST: register/send/  │
            ├──►   ХАБ-СЕРВЕР   ◄─────┤   (один процесс)
            │  SSE: prompts/replies  │
  Агент C ──┘   обновления пула      └── Агент D (другой ПК)
```

Каждый pi-агент подгружает небольшое **клиентское расширение**, которое при старте
сессии регистрируется на хабе, открывает SSE-стрим и шлёт heartbeat-ы. LLM получает
шесть инструментов: `coms_net_list`, `coms_net_dialog`, `coms_net_command`, `coms_net_get`, `coms_net_await`.

Когда агент A шлёт сообщение агенту B, хаб пушит его B через SSE как входящий prompt,
который **запускает ход у B**. B просто отвечает как обычно; хук `agent_end` у
расширения перехватывает ответ B и отправляет обратно на хаб, который пересылает
его A. Вызов `coms_net_await` у A резолвится этим ответом. Ограничение hops
(по умолчанию 5) не даёт уйти в бесконечный ping-pong.

### Двух типов сообщений: `dialog` и `command`

- **`coms_net_dialog`** — диалог: отправитель ожидает ответ. Хаб гарантирует
  доставку: если получатель временно отключён, сообщение дожидается его
  переподключения (ретранслируется при открытии SSE-стрима). На принимающем
  агенте хук `agent_end` захватывает ответ и отправляет его обратно.
  Параметр `expect_reply` (по умолчанию `true`): `true` — обычный вопрос-ответ;
  `false` — fire-and-forget с кратким отчётом (токен-минимально
  на стороне отправителя): принимающему агенту говорится «убедись, что работа
  сделана, и отчитайся коротко о проделанном». На `agent_end` этот краткий отчёт
  захватывается как обычный ответ сообщения (доступен отправителю через
  `coms_net_get`/`coms_net_await` без LLM-раунд-трипа). Отправитель **не** получает
  LLM-ход с телом отчёта — лишь дешёвая отметка `pending.result` и короткая
  TUI-нотификация «reported done». Так отчёт пира **не въезжает** в следующий
  reasoning-шаг отправителя автоматически — он лежит в `pending.result` и его можно
  вытащить по требованию. Полезно для фоновых делегирований («сделай и коротко
  доложи статус»), где важен и факт выполнения, и короткий отчёт, но не нужен
  разбор простыни ответа в следующем ходе.

Обратно-совместимо: старые клиенты/серверы без поля трактуются как `expect_reply=true`.

## Требования

- **Node ≥ 20** (использует глобальные `Request`/`Response`/`ReadableStream` +
  нативное стрипание типов через `--experimental-strip-types`). npm-зависимостей нет.
- **pi** (coding agent) — нужен только на машинах с агентами, не на выделенном хаб-хосте.

## Быстрая установка

Установщик написан на Node (`install.js`) и работает на Linux, macOS и Windows —
никаких bash/PowerShell-зависимостей. `install.sh` сохранён как тонкая обёртка
(`exec node install.js "$@"`) для обратной совместимости.

Из директории этого репозитория:

```bash
node install.js              # полная установка: расширение + хаб-сервер
node install.js --client-only   # только pi-расширение (машина, подключающаяся к удалённому хабу)
node install.js --server-only   # только хаб (выделенный хаб-хост)
node install.js --uninstall     # убрать всё
# или на POSIX:
./install.sh ...               # то же самое через bash-обёртку
```

Установщик:
1. Копирует `client/coms-net.ts` → `~/.pi/agent/extensions/` (pi сам её находит).
2. Копирует `server/coms-net-server.ts` → `~/.pi/coms-net/bin/` + launcher `coms-net-server`
   на `PATH` (по умолчанию `~/.local/bin` на POSIX, `%USERPROFILE%\.pi\bin` на Windows;
   переопределить через `--bin-dir <path>`).
3. Добавляет `coms_net_list/send/get/await` в allow-list `"active"` в
   `~/.pi/agent/tools.json`. **Этот шаг обязателен** — pi фильтрует инструменты
   расширения, которых нет в этом списке.

### Windows

На Windows установщик кладёт `coms-net-server.cmd` и `coms-net-server.ps1` в
`%USERPROFILE%\.pi\bin` и напоминает добавить эту директорию в `PATH`
(`setx PATH "%PATH%;%USERPROFILE%\.pi\bin"`). `chmod 0600` для `server.secret.json`
на Windows неприменим (нет POSIX-режимов), поэтому проверка режима в клиенте
отключается на `win32` — loopback auto-token работает как обычно. `fs.renameSync`
перезаписывается через `rmSync`-приём, т.к. на Windows rename поверх существующего
файла падает с `EPERM`. Для завершения сервера помимо `SIGINT`/`SIGTERM`
регистрируется `SIGBREAK`.

## Использование — локально (одна машина)

```bash
coms-net-server                       # запускаем хаб (авто-токен, авто-порт)
pi --cname planner --project default  # агент 1 (сам находит локальный хаб)
pi --cname coder    --project default  # агент 2
```

В любом pi попроси LLM связаться с другим агентом, например:
> «Свяжись с агентом coder: вызови coms_net_dialog(target='coder', prompt='...'),
> затем coms_net_await с полученным msg_id.»

Принимающий агент просто отвечает нормально — его ответ автоматически
перехватывается и уходит обратно. Под редактором также появится виджет «пула»,
показывающий, кто онлайн.

## Использование — по сети

Хаб на хосте A (не-loopback **требует** явный токен):
```bash
# POSIX:
PI_COMS_NET_HOST=0.0.0.0 PI_COMS_NET_PORT=7780 \
PI_COMS_NET_AUTH_TOKEN=<shared-secret> coms-net-server
# Windows (PowerShell):
$env:PI_COMS_NET_HOST='0.0.0.0'; $env:PI_COMS_NET_PORT=7780; `
$env:PI_COMS_NET_AUTH_TOKEN='<shared-secret>'; coms-net-server
```

Агенты на хостах B/C:
```bash
pi --cname <имя> --project default \
   --server-url http://<hostA>:7780 --auth-token <shared-secret>
```

## Безопасность

Хаб аутентифицирует только по bearer-токену и говорит по чистому HTTP. Для работы
по сети запускайте его поверх доверенной сети (Tailscale/WireGuard) либо за TLS-реверс-
прокси. Токен никогда не пишется в логи и вычищается из клиентских ошибок.

## Флаги и переменные окружения

Флаги клиентского расширения (на запуск `pi`): `--cname`, `--purpose`, `--project`,
`--color`, `--explicit`, `--server-url`, `--auth-token`, `--soul`. Env-фолбэки:
`PI_COMS_NET_SERVER_URL`, `PI_COMS_NET_AUTH_TOKEN`, `PI_COMS_NET_PROJECT`.

Env хаба: `PI_COMS_NET_HOST`, `PI_COMS_NET_PORT` (0 = назначается ОС),
`PI_COMS_NET_PUBLIC_URL`, `PI_COMS_NET_PROJECT`, `PI_COMS_NET_AUTH_TOKEN`,
`PI_COMS_NET_MAX_HOPS`, `PI_COMS_NET_MESSAGE_TTL_MS`, `PI_COMS_NET_HEARTBEAT_MS`,
`PI_COMS_NET_STALE_AFTER_MS`, `PI_COMS_NET_OFFLINE_AFTER_MS`,
`PI_COMS_NET_INTRO_MAX_LENGTH`, `PI_COMS_NET_LOG_QUIET`, `PI_COMS_NET_LOG_HEARTBEAT`,
`PI_COMS_NET_LOG_RETENTION_DAYS` (сколько дней хранить .jsonl логи, по умолчанию 7).
`PI_COMS_NET_PARALLELISM` (глобальный лимит параллельных DIALOG-сообщений в обработке;
см. «Настройка параллелизма» ниже).

### Настройка параллелизма

`PI_COMS_NET_PARALLELISM` контролирует, сколько DIALOG-сообщений могут
обрабатываться одновременно **во всей системе** (все проекты, все агенты).
Значение указывается при запуске хаба:

```bash
PI_COMS_NET_PARALLELISM=1 coms-net-server   # строго последовательно (default)
PI_COMS_NET_PARALLELISM=3 coms-net-server   # до 3 сообщений одновременно
PI_COMS_NET_PARALLELISM=0 coms-net-server   # безлимит (старое поведение)
```

**Как это работает.**
Хаб отслеживает количество DIALOG-сообщений в статусе `delivered` (доставлено
агенту, но ответ ещё не получен). Когда лимит достигнут, новые сообщения
остаются в очереди (`queued`) и не отправляются агенту. Как только агент
отвечает (или сообщение протухает по TTL), слот освобождается, и следующее
сообщение из очереди доставляется (FIFO — самые старые первыми).

COMMAND-сообщения (fire-and-forget) не учитываются в лимите и доставляются
всегда.

Slash-команды в pi:
- `/coms-net` — обновить виджет; `--all`, `--server`, `--reconnect`, `--project <name>`.
- `/introduce-me` — сгенерировать короткое описание агента по его `SOUL.md` и
  опубликовать его на хабе для других агентов. Флаги: `--show` (показать уже
  опубликованное), `--regenerate` (перезаписать существующее).

Инструменты, доступные LLM:
- `coms_net_dialog` — диалог: отправить сообщение и получить ответ (гарантированная доставка)
- `coms_net_command` — команда: однонаправленное уведомление, ответ не нужен (fire-and-forget)
- `coms_net_list` — список пиров
- `coms_net_get` — неблокирующая проверка ответа на `coms_net_dialog`
- `coms_net_await` — блокирующее ожидание ответа на `coms_net_dialog`
- `coms_net_introduce` — публикация короткого представления агента (используется `/introduce-me`).

## Agent SOUL.md (персонаж агента)

`SOUL.md` — файл-персонаж агента, отдельный от `SYSTEM.md`/`--system-prompt`
pi (те задают базовый системный промпт; `SOUL.md` дополняет его). Discovery:
`--soul <path>` → `<cwd>/.pi/SOUL.md` → `~/.pi/agent/SOUL.md` (берётся первый
найденный). Поддерживает frontmatter `name` / `description` / `color`, который
встраивается в цепочку приоритета идентичности агента (`--cname` > SOUL >
system-prompt frontmatter > автогенерацию имени).

Тело `SOUL.md` (без frontmatter) дописывается в системный промпт через
расширение события `before_agent_start` — оно **не запускает ход само по себе**,
а лишь дополняет промпт для того хода, который уже инициирован пользователем
или входящим сообщением от пира. Таким образом `SOUL.md` — тихое персонажное
содержание, а не фоновая работа.

`/introduce-me` — LLM-управляемая команда: она выставляет агенту сообщение
(`followUp`, `triggerTurn: true`) с инструкцией прочитать свой `SOUL.md` (или
свой name/purpose как фолбэк) и сразу вызвать `coms_net_introduce` с коротким
описанием. Хаб хранит представление в `AgentCard.introduction`, рассылает его
пирами через SSE-событие `agent_intro` (другие агенты получаютего
автоматически при подключении и в `coms_net_list`). Хаб отклоняет перезапись
непустого представления без `update=true`, чтобы пиры не спамили. Только
короткое описание покидает машину — полное тело `SOUL.md` никуда не
отправляется. Пример:

```bash
pi --cname planner --soul ./planner-soul.md --project default
# внутри агента: /introduce-me
# в другом агенте: «Свяжись с агентом planner» — его capabilities уже видны в
# coms_net_list (строка с описанием после `| `).
```

## Структура проекта

```
coms-net-pi/
├── install.js               кросс-платформенный установщик / деинсталлятор (Node)
├── install.sh               тонкая bash-обёртка → install.js (обратная совместимость)
├── package.json
├── bin/coms-net-server.js   кросс-платформенный launcher (npm bin; Node)
├── bin/coms-net-server      bash-шим (для прямого запуска на POSIX)
├── client/coms-net.ts       расширение pi (порт из upstream coms-net.ts)
└── server/coms-net-server.ts хаб на Node (переписан транспорт с Bun.serve)
```

## Что портировано, а что оставлено

- **Клиент** (`coms-net.ts`): почти механический порт — переименованы импорты
  `@mariozechner/pi-*` → `@earendil-works/pi-*`; убран опциональный загрузчик темы
  `themeMap.ts` (используется ваша глобальная тема pi). Логика без изменений.
- **Сервер** (`coms-net-server.ts`): вся логика роутера/обработчиков/SSE/await —
  оригинальный upstream-код as-is, поскольку он написан против стандартного Web Fetch
  API (`Request`/`Response`/`ReadableStream`/`Headers`/`TextEncoder`), который Node ≥ 20
  предоставляет глобально. Переписан только транспортный слой
  (`Bun.serve` → мост `node:http`) и entrypoint-гард (`import.meta.main` → проверка argv).

## Проверено

End-to-end протестировано на Node 22.22.3 / pi 0.80.5: два агента обмениваются
сообщением через хаб — `coms_net_await` у отправителя резолвится авто-перехваченным
ответом получателя.
