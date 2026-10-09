# AI Video Generation via ProxyAPI → Yandex Disk → Telegram

> Text-to-video in one webhook: submits a generation job, polls until it finishes, downloads the clip, stores it on Yandex Disk and sends it to Telegram.

**Category:** AI media · **Nodes:** 13 · **Trigger:** Webhook `POST /ai-video` · **Tested on:** n8n 1.x self-hosted (October 2026)

## What it does

A single `POST` starts a text-to-video generation and returns the stored clip's metadata. Because video
generation is **asynchronous**, the workflow cannot answer immediately: the provider returns a job id within
seconds, while the clip itself takes minutes. So the flow is split into *create → wait → poll → branch*, and
the HTTP response is sent later, from a **Respond to Webhook** node, once the file is on disk.

The loop is built from an ordinary **Wait** node (10 seconds) plus two If nodes, not from a Code loop:

1. **Запрос видео** (`webhook`) — `POST /ai-video`, guarded by the `x-watchdog-token` header.
   Response mode is `responseNode`, so nothing is returned until a Respond-to-Webhook node runs.
2. **Параметры** (`set`) — normalizes the body: `prompt` (required), `model` (default `google/veo-3.1-lite`),
   `duration` (default `4`), `resolution` (default `720p`) and `fileName` (default `clip-<yyyyMMdd-HHmmss>`).
   `includeOtherFields` keeps the raw webhook item available downstream.
3. **Создать задачу** (`httpRequest`) — `POST https://api.proxyapi.ru/v1/videos` with the header credential;
   JSON body `{ model, prompt, duration, resolution }`. Returns the job object (`id`, possibly `polling_url`).
4. **Ждать** (`wait`) — pauses the execution 10 seconds.
5. **Проверить статус** (`httpRequest`) — `GET` the job. On the first pass it polls
   `{{ $json.polling_url }}` (or falls back to `https://api.proxyapi.ru/v1/videos/<id>`); on retries the URL is
   rebuilt from **Создать задачу**, because the Wait node emits an empty item and that data no longer exists.
6. **Готово?** (`if`) — is `status` equal to `completed`?
   - **true** → **Скачать файл**.
   - **false** → **Отказ?**.
7. **Отказ?** (`if`) — did the job reach `failed`?
   - **true** → **Ответ об ошибке** responds `{ ok: false, status, error, jobId }` and the run ends.
   - **false** (still `queued` / `processing`) → back to **Проверить статус**, which closes the polling loop.
8. **Скачать файл** (`httpRequest`) — `GET /v1/videos/<id>/content` with the header credential, response format
   **file** into the binary property `data`, 300 s timeout.
9. From here the branch **splits into two parallel consumers of the same binary**:
   - **На Яндекс Диск** (`httpRequest`) — `PUT https://webdav.yandex.ru/Video/<fileName>.mp4` with the Basic
     Auth credential and the binary `data` as the body (300 s timeout, `neverError: true`).
   - **Отправить в Telegram** (`telegram`) — `sendVideo` with `binaryPropertyName: data`, a caption built from
     model / duration / resolution / prompt, and the file name `<fileName>.mp4`. This runs **in parallel with
     the upload**, not after it — Telegram gets the clip even if the Disk upload fails.
10. **Итог** (`set`) — after the Disk upload, builds the response object: `ok` (`statusCode` is `201` or `204`),
    numeric `status`, `diskPath`, `prompt`, `model`, `seconds`, `jobId`.
11. **Ответ** (`respondToWebhook`) — returns exactly that JSON, overwriting the webhook item, which ends the run.

Request body fields: `prompt` (required), `model`, `duration`, `resolution`, `fileName`.

Response on success:

```json
{
  "ok": true,
  "status": 201,
  "diskPath": "Video/clip-20261001-120000.mp4",
  "prompt": "a drone shot over a misty forest at sunrise",
  "model": "google/veo-3.1-lite",
  "seconds": 4,
  "jobId": "job_abc123"
}
```

Response on failure (job status `failed`):

```json
{ "ok": false, "status": "failed", "error": "генерация не удалась", "jobId": "job_abc123" }
```

## Requirements

- n8n (self-hosted, recent 1.x). No community nodes — `webhook`, `set`, `httpRequest`, `wait`, `if`,
  `respondToWebhook`, `telegram`.
- **HTTP Header Auth** credential for ProxyAPI (`Authorization: Bearer <PROXYAPI-KEY>`), removed from the
  published JSON.
- **HTTP Basic Auth** credential for Yandex Disk (Yandex login + **app password**, not the account password),
  removed from the published JSON.
- **Telegram** credential (a bot token from @BotFather) plus the chat id that should receive the clip.
- A ProxyAPI account with access to the chosen video model, and a `Video/` folder on Yandex Disk.
- Outbound HTTPS to `api.proxyapi.ru`, `webdav.yandex.ru` and `api.telegram.org`.

## Setup

1. Import `workflow.json` (Workflows → ⋯ → Import from File).
2. Create the credentials and assign them to the nodes that lost them:
   | Credential | Type | Assign to |
   |---|---|---|
   | ProxyAPI key | HTTP Header Auth | **Создать задачу**, **Проверить статус**, **Скачать файл** |
   | Yandex Disk app password | HTTP Basic Auth | **На Яндекс Диск** |
   | Telegram bot | Telegram API | **Отправить в Telegram** |
3. Replace the placeholders:
   - **Запрос видео** → Options → *Only Run If*:
     `{{ $json.headers['x-watchdog-token'] === 'REPLACE_WITH_YOUR_TOKEN' }}` — set your own long random secret.
   - **Отправить в Telegram** → *Chat ID*: `REPLACE_WITH_YOUR_CHAT_ID` → your numeric chat id (send the bot a
     message first, then read `getUpdates`).
4. Replace the customer-specific paths and hosts:
   - `YOUR-N8N-HOST` in the curl examples below (the webhook path stays `/ai-video` unless you rename it).
   - the Yandex Disk folder in **На Яндекс Диск**: `https://webdav.yandex.ru/Video/…` — use a folder that
     exists and that the app password can write to.
   - `example.com`-style values do not appear here, but if you swap ProxyAPI for another OpenAI-compatible
     provider, change the three `api.proxyapi.ru` URLs consistently (**Создать задачу**, **Проверить статус**,
     **Скачать файл**).
5. Activate the workflow and run it:

```bash
curl -X POST https://YOUR-N8N-HOST/webhook/ai-video \
  -H 'x-watchdog-token: REPLACE_WITH_YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
        "prompt": "a drone shot over a misty forest at sunrise",
        "model": "google/veo-3.1-lite",
        "duration": 4,
        "resolution": "720p",
        "fileName": "forest-clip"
      }'
```

Expect the request to stay open for the whole generation (minutes, not seconds) — the HTTP client sees the
response only after the clip is uploaded. For fire-and-forget use, drop the two Respond-to-Webhook nodes and
let the workflow answer immediately.

## Nodes

| Node | Type | Purpose |
|---|---|---|
| Запрос видео | `n8n-nodes-base.webhook` | `POST /ai-video` entry point with the shared-secret check; replies via response nodes. |
| Параметры | `n8n-nodes-base.set` | Applies defaults for `model`, `duration`, `resolution`, `fileName` and keeps the raw body. |
| Создать задачу | `n8n-nodes-base.httpRequest` | `POST /v1/videos` — submits the generation job and returns its id. |
| Ждать | `n8n-nodes-base.wait` | 10-second pause between status polls. |
| Проверить статус | `n8n-nodes-base.httpRequest` | `GET` the job; uses `polling_url` on the first pass, the id-based URL afterwards. |
| Готово? | `n8n-nodes-base.if` | Branches on `status === "completed"`. |
| Скачать файл | `n8n-nodes-base.httpRequest` | Downloads `/v1/videos/<id>/content` into the binary property `data`. |
| На Яндекс Диск | `n8n-nodes-base.httpRequest` | WebDAV `PUT` of the binary into `Video/<fileName>.mp4`. |
| Отправить в Telegram | `n8n-nodes-base.telegram` | `sendVideo` of the same binary with a caption, in parallel with the upload. |
| Итог | `n8n-nodes-base.set` | Builds the success payload (`ok`, `status`, `diskPath`, `jobId`, request echo). |
| Ответ | `n8n-nodes-base.respondToWebhook` | Returns the success JSON and ends the execution. |
| Отказ? | `n8n-nodes-base.if` | Branches on `status === "failed"`. |
| Ответ об ошибке | `n8n-nodes-base.respondToWebhook` | Returns `{ ok: false, status, error, jobId }` and ends the execution. |

## Notes

- **The polling loop can run forever.** `Отказ?` sends anything that is not `failed` back to **Проверить
  статус**, and the Wait node is a fixed 10 s. A job stuck in `queued` (moderation, provider incident, quota)
  keeps the execution alive: set *Workflow Settings → Timeout* or add an attempt counter. There is no
  "max tries" here on purpose — the template stays minimal.
- **The Wait node loses upstream data.** That is why **Проверить статус** re-derives the URL from
  `$('Создать задачу').first().json.id` instead of relying on the current item. If you replace the Wait node
  with an HTTP poll, keep that reference — otherwise the retries hit an empty URL.
- **The ternary has no null guard** (`$json.polling_url || fallback`): if the status response comes back
  without `status`, **Готово?** and **Отказ?** both fall through to **false**, and the second one loops.
  Add a schema check if you point this at a different provider.
- **`responseFormat: file` + `outputPropertyName: "data"`.** The download node writes the clip into the binary
  property named `data`; both consumers (**На Яндекс Диск** and **Отправить в Telegram**) read that exact name.
  Rename it in one place and you must rename it in the other two.
- **The Yandex Disk node uses `neverError: true` with a `text` response format** — an upload failure does not
  abort the run, it just makes `Итог.ok` false (`statusCode` is neither `201` nor `204`). Note that WebDAV
  answers `201 Created` for a new file and `204 No Content` when it overwrites an existing one, which is why
  both codes count as success. A `PUT` with `Content-Range` can also return `206` — not treated as success here.
  The same request has `responseFormat: 'text'`, so any response body lands in the `data` field, not in the item root.
- **Telegram runs before the upload finishes.** Both nodes are fed by **Скачать файл** in parallel, so the
  Telegram message proves the clip was generated, not that it reached the Disk. Wire it after **На Яндекс
  Диск** if you need that guarantee.
- **Telegram bot upload limit is ~50 MB**, and it is enforced by Telegram, not by n8n. Generated clips are
  small, but a longer/higher-resolution job can exceed it and make the Telegram node fail — the workflow
  would then error after the Disk upload already succeeded.
- **Yandex Disk path is built from user input**: `fileName` comes straight from the request body, so it can
  contain `/` or `..`. Sanitize it (or hard-code a prefix) before exposing this webhook to anyone but yourself.
- Timeouts: 60 s for task creation and status polls, 300 s for download and upload. Long generations do not
  need a longer *node* timeout because the waiting happens in the **Wait** node, not inside a request — but the
  *webhook* response is still pending meanwhile, so put a reverse proxy read timeout in front of n8n.
- `availableInMCP` and `executionOrder: v1` come from the export; nothing in the logic depends on them.
- Only two failure paths are handled: provider-reported `failed`, and a non-2xx Disk upload. A failed Telegram
  send, a download error or an HTTP error from the provider will surface as a normal node error (no error
  branch is wired).

## Кратко по-русски

Шаблон делает клип из текстового описания одним вызовом вебхука `POST /ai-video`: создаёт задачу в ProxyAPI,
в цикле «Ждать 10 секунд → Проверить статус → If» ждёт готовности, скачивает файл, кладёт его на Яндекс Диск
по WebDAV и параллельно отправляет в Telegram. Ответ отдаётся не сразу, а в конце узлом Respond to Webhook,
поэтому HTTP-клиент ждёт всю генерацию (минуты). Тело запроса: `prompt` (обязателен), `model`, `duration`,
`resolution`, `fileName`.

Что заменить: Basic Auth Яндекса (пароль приложения), Header Auth ключа ProxyAPI, бота и chat id Telegram,
токен `REPLACE_WITH_YOUR_TOKEN` в условии вебхука, папку `Video/` на Диске. Грабли: цикл опроса бесконечный —
ставьте таймаут workflow; узел Wait теряет данные, поэтому URL опроса собирается заново из «Создать задачу».

---

Made by [Bowhard](https://bowhard.ru) — automation studio. Template is provided as-is, MIT licensed.
