# Yandex Calendar: list events (CalDAV)

> Webhook tool that runs a CalDAV `calendar-query` (`REPORT`) over a time window and returns Yandex Calendar events as a clean, sorted JSON list — no native n8n node.

**Category:** Yandex · CalDAV · **Nodes:** 4 · **Trigger:** Webhook `POST /yandex-event-list` · **Tested on:** n8n 1.x self-hosted (October 2026)

## What it does

Your system asks "what is in this calendar between these dates" and gets back events, not XML. One Code node builds the CalDAV query, an HTTP Request node sends it with the non-standard `REPORT` method, and a second Code node turns the multistatus response into flat objects.

1. **Запрос событий** (`n8n-nodes-base.webhook`, v2.1) — `POST /yandex-event-list`, response mode `lastNode` with `responseData: allEntries`; an `onlyRunIf` condition checks the `x-watchdog-token` header before anything else runs.
2. **Собрать запрос событий** (`n8n-nodes-base.code`, v2) — resolves the requested calendar name to a collection path through a hard-coded allow-list, normalizes the time window (`from` / `to`, with defaults) and builds the `calendar-query` XML with a `<c:time-range>` filter on `VEVENT`.
3. **Список событий** (`n8n-nodes-base.httpRequest`, v4.5) — `REPORT https://caldav.yandex.ru` + collection path, raw `application/xml; charset=utf-8` body, headers `Depth: 1` and `Content-Type`, with the `webdavMethods` option enabled (60 s timeout).
4. **Разобрать события** (`n8n-nodes-base.code`, v2) — splits the response on `BEGIN:VEVENT`, pulls out `SUMMARY`, `DTSTART`, `DTEND`, `LOCATION`, `UID`, sorts by start and returns the first 50 events with a count and a verdict.

```json
{ "calendar": "Personal",
  "range": { "start": "20261015T000000Z", "end": "20261114T000000Z" },
  "count": 2, "status": 207, "note": "ответ получен",
  "events": [ { "summary": "Demo event", "start": "20261015T100000Z", "end": "20261015T110000Z",
                "allDay": false, "location": "", "uid": "demo-event-0001" } ] }
```

## Why not a native node

n8n has no Yandex node, and Yandex Calendar is reached over **CalDAV** — so the whole integration is one HTTP Request node plus two small Code nodes. The `REPORT` verb and the `Depth` header are WebDAV-level details the HTTP Request node supports natively (via `webdavMethods`), and authentication is an ordinary **HTTP Basic Auth** credential (Yandex login + app password). Nothing here needs an SDK, an OAuth app or a token refresh loop.

## Requirements

- n8n, self-hosted, recent 1.x (exported from a self-hosted instance, October 2026).
- **Credential: HTTP Basic Auth** — user = your Yandex login (full address), password = an **app password** issued for CalDAV access. Assign it to **Список событий**; the exported JSON contains no credentials on purpose.
- **Your calendars, edited in the workflow.** The `allowed` map inside **Собрать запрос событий** maps friendly calendar names to CalDAV collection paths. The map shipped in the file still points at the exporter's own account — replace all three paths (each of the form `/calendars/you@example.com/events-0000000/`, trailing slash required) before you activate.
- Read-only usage: the workflow only sends `REPORT` queries, it never writes to a calendar.

## Setup

1. Import `workflow.json` (**Workflows → ⋯ → Import from File**).
2. Create the **HTTP Basic Auth** credential and assign it to **Список событий**.
3. Replace `REPLACE_WITH_YOUR_TOKEN` in **Запрос событий**, *Options → onlyRunIf*. Note the short expression form used here — `=$json.headers['x-watchdog-token'] === 'REPLACE_WITH_YOUR_TOKEN'` (no `{{ }}`); it evaluates as a single expression, but you can rewrite it as `={{ … }}` if you prefer the common style.
4. Set your own calendars: open **Собрать запрос событий** and rewrite the `allowed` object — key = the calendar name callers will send, value = your collection path ending with `/`. The **first** entry is also the fallback for unknown names.
5. Activate the workflow and test — the request takes the calendar *name*, not the path:

```bash
curl -X POST https://YOUR-N8N-HOST/webhook/yandex-event-list \
  -H 'x-watchdog-token: REPLACE_WITH_YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{ "calendar": "Personal", "from": "2026-10-01", "to": "2026-10-31" }'
```

| Body field | Required | What it does |
|---|---|---|
| `calendar` | no | Allow-list key; omitted or unknown → the first entry (default calendar) |
| `from` | no | Window start, `YYYY-MM-DD` or full ISO; omitted/invalid → today 00:00 UTC |
| `to` | no | Window end, same formats; omitted/invalid → `from` + 30 days |

## Nodes

| Node | Type | Purpose |
|---|---|---|
| Запрос событий | `n8n-nodes-base.webhook` v2.1 | Receives `POST /yandex-event-list`, rejects calls without the `x-watchdog-token` header |
| Собрать запрос событий | `n8n-nodes-base.code` v2 | Calendar allow-list, time-window defaults, `calendar-query` XML with `<c:time-range>` |
| Список событий | `n8n-nodes-base.httpRequest` v4.5 | `REPORT` with `Depth: 1`, HTTP Basic Auth, raw XML body (`webdavMethods` enabled) |
| Разобрать события | `n8n-nodes-base.code` v2 | Extracts VEVENT fields from the multistatus body, sorts them, returns up to 50 |

## Notes

- **`webdavMethods: true` is mandatory** on the HTTP node: `REPORT` is not in n8n's default method list, so the node would reject it otherwise. Headers are sent as a key/value pair list and the body is raw XML produced by the Code node.
- **The response is read from `$json.data`, not from a parsed JSON field.** `responseFormat: autodetect` plus `fullResponse: true` means the CalDAV body is passed through as text while `statusCode` carries the HTTP code — the parser reads exactly those two. A successful CalDAV multistatus is `207`, which the Code node treats as the healthy code; `401` becomes `отказ в доступе — проверить пароль приложения для календаря`.
- **`neverError: true`** means an auth failure or a rejected query still returns HTTP 200 to the caller, with the real code inside `status` / `note`. Callers should check `status === 207` rather than the transport status.
- **Unknown calendar names fail silently.** `allowed[wanted] || allowed[<first key>]` returns the default calendar for any typo, so a wrong `calendar` value gives you events from the wrong place instead of an error.
- **Unparsable dates fall back too.** `parseWhen()` keeps its fallback when `new Date(...)` yields `NaN`, so `from: "next week"` becomes "from today" — no field-level validation errors are surfaced.
- **The parser is deliberately small, and it has limits worth knowing:** only the first 50 events come back (`events.slice(0, 50)`); folded iCalendar lines (continuations beginning with a space) are not unfolded, so long summaries can arrive truncated; ICS escapes (`\n`, `\,`) are not decoded; all-day events keep CalDAV's raw `DATE` form (e.g. `20261015`) with `allDay: true` set by the `VALUE=DATE` marker. Events without a summary and without a start are dropped.
- **Timestamps in the output are CalDAV's own compact UTC form** (`YYYYMMDDTHHMMSSZ`), not ISO — the `stamp()` helper only formats the query window, it does not reformat event values.
- **No pagination and no sync token**: this is a window query, not an incremental sync. For "what changed since last time", compare `UID` + `DTSTART` across two calls, or extend the workflow with a `sync-collection` REPORT.

## Кратко по-русски

Вебхук `POST /yandex-event-list` принимает `{calendar, from, to}`, узел-код по списку календарей находит путь коллекции и собирает CalDAV-запрос `calendar-query` с фильтром по времени, HTTP-узел отправляет его методом `REPORT` (включена опция `webdavMethods`, заголовок `Depth: 1`), а второй код-узел разбирает ответ и отдаёт отсортированный список событий (до 50 штук). Официального узла Яндекса в n8n нет — учётные данные обычные: HTTP Basic Auth (логин Яндекса + пароль приложения), назначить узлу «Список событий». Защита — заголовок `x-watchdog-token`, значение `REPLACE_WITH_YOUR_TOKEN` в поле onlyRunIf узла «Запрос событий». **Перед запуском обязательно замените три пути календарей в коде узла «Собрать запрос событий»** — в файле остались пути аккаунта-экспортёра. Ответ: `calendar`, `range`, `count`, `status` (успех — 207), `note`, `events`.

---

Made by [Bowhard](https://bowhard.ru) — automation studio. Template is provided as-is, MIT licensed.
