# Yandex Calendar: create event (CalDAV)

> Webhook tool that turns a JSON payload into an `.ics` event and `PUT`s it into a Yandex Calendar collection over CalDAV — plain HTTP Request node, no Yandex SDK and no native node.

**Category:** Yandex · CalDAV · **Nodes:** 4 · **Trigger:** Webhook `POST /yandex-event-create` · **Tested on:** n8n 1.x self-hosted (October 2026)

## What it does

Your own system posts an event (title, time window, all-day flag, UID, calendar path) and gets back a small JSON verdict. The workflow builds an iCalendar file with the built-in `convertToFile` node and writes it into the CalDAV collection named in the request, so a single endpoint can serve several calendars and several callers.

1. **Запрос события** (`n8n-nodes-base.webhook`, v2.1) — `POST /yandex-event-create`, response mode `lastNode` with `responseData: allEntries`. An `onlyRunIf` condition compares the `x-watchdog-token` header with the token stored in the node, so anonymous calls never reach Yandex.
2. **Собрать событие** (`n8n-nodes-base.convertToFile`, v1.1, operation `iCal`) — builds the `.ics` from `body.title`, `body.start`, `body.end`, `body.allDay`, adds `UID` from `body.uid` and `DESCRIPTION` from `body.description` (empty string when the field is absent), and names the file `<uid>.ics`.
3. **Положить в календарь** (`n8n-nodes-base.httpRequest`, v4.5) — `PUT https://caldav.yandex.ru` + `calendarPath` + `<uid>.ics`, HTTP Basic Auth, the generated file sent as binary data (30 s timeout).
4. **Итог** (`n8n-nodes-base.code`, v2) — maps the CalDAV status code to a human verdict and returns the normalized result.

```json
{ "title": "Demo event", "start": "2026-10-15T10:00:00Z", "end": "2026-10-15T11:00:00Z",
  "allDay": false, "status": 201, "ok": true, "note": "событие создано" }
```

## Why not a native node

n8n ships no Yandex node at all, and Yandex Calendar speaks **CalDAV** — an open standard, not a proprietary API. So instead of waiting for an integration, the template does what n8n is good at: it builds the iCalendar payload with `convertToFile` and issues a plain `PUT` through the HTTP Request node, authenticating with an **HTTP Basic Auth** credential (Yandex login + app password). The only Yandex-specific values in the whole workflow are the base URL `https://caldav.yandex.ru` and the collection path you pass in the request — everything else is generic iCal and HTTP.

## Requirements

- n8n, self-hosted, recent 1.x (exported from a self-hosted instance, October 2026).
- **Credential: HTTP Basic Auth** — user = your Yandex login (full address), password = an **app password** issued for CalDAV access. Assign it to **Положить в календарь**; the exported JSON contains no credentials on purpose.
- A CalDAV collection path of your own, ending with `/` — e.g. `/calendars/you@example.com/events-0000000/`. It is **not** stored in the workflow: the caller sends it in every request, together with the event `uid`.
- The calling system should generate a stable `uid` per event (any unique string works).

## Setup

1. Import `workflow.json` (n8n: **Workflows → ⋯ → Import from File**).
2. Create the **HTTP Basic Auth** credential and assign it to the node **Положить в календарь**.
3. Replace `REPLACE_WITH_YOUR_TOKEN`. It appears exactly once — in **Запрос события**, *Options → onlyRunIf*:
   `={{ $json.headers['x-watchdog-token'] === 'REPLACE_WITH_YOUR_TOKEN' }}`.
   Use a long random string; every caller must send it as the `x-watchdog-token` header.
4. Set your own calendar path. Nothing to edit inside the nodes: `calendarPath` is a **request field**, concatenated after the hard-coded base URL `https://caldav.yandex.ru` in **Положить в календарь**. It must start and end with `/`.
5. Activate the workflow (a webhook only answers in production) and test:

```bash
curl -X POST https://YOUR-N8N-HOST/webhook/yandex-event-create \
  -H 'x-watchdog-token: REPLACE_WITH_YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
        "calendarPath": "/calendars/you@example.com/events-0000000/",
        "uid": "demo-event-0001",
        "title": "Demo event",
        "start": "2026-10-15T10:00:00Z",
        "end": "2026-10-15T11:00:00Z",
        "allDay": false,
        "description": "Created by n8n"
      }'
```

| Body field | Required | What it does |
|---|---|---|
| `calendarPath` | yes | CalDAV collection path, appended to `https://caldav.yandex.ru` |
| `uid` | yes | iCal `UID`, also the file name (`<uid>.ics`) and the last URL segment |
| `title` | yes | iCal `SUMMARY` |
| `start`, `end` | yes | Event window, passed to `convertToFile` as-is |
| `allDay` | yes | All-day flag; the result reports `true` only for a literal boolean `true` |
| `description` | no | iCal `DESCRIPTION`; omitted → empty string |

## Nodes

| Node | Type | Purpose |
|---|---|---|
| Запрос события | `n8n-nodes-base.webhook` v2.1 | Receives `POST /yandex-event-create`, rejects calls without the `x-watchdog-token` header |
| Собрать событие | `n8n-nodes-base.convertToFile` v1.1 | Builds the `.ics` file (`operation: iCal`) with UID, description and file name |
| Положить в календарь | `n8n-nodes-base.httpRequest` v4.5 | `PUT` the file into the Yandex CalDAV collection, HTTP Basic Auth |
| Итог | `n8n-nodes-base.code` v2 | Reads `statusCode`, sets `ok` and a Russian `note`, returns the normalized JSON |

## Notes

- **The token check is the only guard.** When `onlyRunIf` is false the workflow starts and stops at the webhook — Yandex is never called, and the caller does not get the `Итог` JSON. There is no second authentication layer: keep the endpoint behind HTTPS and treat the token as a secret.
- **Status handling depends on `neverError: true`.** The HTTP node is configured with `fullResponse: true`, `neverError: true` and `responseFormat: text`, so a `401` or `403` does not fail the run — it lands in `Итог` as `status` plus a `note` (`отказ в доступе — проверить учётные данные`, `календарь только для чтения`). Change the response format and the mapping in **Итог** breaks.
- **Same UID = update, not duplicate.** `PUT` to an existing `<uid>.ics` returns `204` and overwrites the event; a new UID returns `201`. The `ok` flag accepts both, so retrying a request is safe.
- **Missing `calendarPath` is not validated**: `undefined` is concatenated into the URL (`https://caldav.yandex.ruundefined.ics`). A missing `uid` likewise produces `undefined.ics`. Both fields are effectively mandatory even though nothing checks them.
- **`ok` and `status` are the machine-readable fields**; `note` is a Russian human string from the Code node (`событие создано`, `событие обновлено`, …) and should not be parsed by callers.
- **No retry, no queue**: the HTTP node has a 30 s timeout and no `retryOnFail`. For bulk imports, add batching and retries on your side rather than hammering the calendar endpoint.

## Кратко по-русски

Вебхук `POST /yandex-event-create` принимает JSON (`calendarPath`, `uid`, `title`, `start`, `end`, `allDay`, необязательный `description`), узел `convertToFile` собирает файл `.ics`, а HTTP-узел кладёт его в Яндекс.Календарь через CalDAV (`PUT https://caldav.yandex.ru…`). Официального узла Яндекса в n8n нет — здесь работают обычные узлы и учётные данные HTTP Basic Auth (логин Яндекса + пароль приложения), назначить их нужно узлу «Положить в календарь». Защита вебхука — заголовок `x-watchdog-token`, значение `REPLACE_WITH_YOUR_TOKEN` в поле onlyRunIf узла «Запрос события». Тот же `uid` перезаписывает событие (ответ `204`), новый — создаёт (`201`); ответ отдаёт узел «Итог» с полями `status`, `ok`, `note`.

---

Made by [Bowhard](https://bowhard.ru) — automation studio. Template is provided as-is, MIT licensed.
