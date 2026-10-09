# Yandex Disk: upload a file (WebDAV)

> Webhook tool that takes a base64 payload plus a disk path and `PUT`s the decoded file to Yandex Disk over WebDAV — no official Yandex node, no OAuth flow.

**Category:** Yandex · WebDAV · **Nodes:** 4 · **Trigger:** Webhook `POST /yandex-disk-upload` · **Tested on:** n8n 1.x self-hosted (October 2026)

## What it does

Any other workflow (or an external script) can drop a file onto Yandex Disk with one HTTP call: send the path and the file as base64, get back a status verdict. This is the storage step you reach for when a report, a call recording or a generated document has to land somewhere outside n8n.

1. **Запрос загрузки** (`n8n-nodes-base.webhook`, v2.1) — `POST /yandex-disk-upload`, response mode `lastNode` with `responseData: allEntries`; an `onlyRunIf` condition checks the `x-watchdog-token` header, so only your own callers get as far as the disk.
2. **Превратить в файл** (`n8n-nodes-base.convertToFile`, v1.1, operation `toBinary`) — turns the `body.contentBase64` string into a binary item, with `body.fileName` (default `file.bin`) and `body.mimeType` (default `application/octet-stream`).
3. **Загрузить на Диск** (`n8n-nodes-base.httpRequest`, v4.5) — `PUT https://webdav.yandex.ru/` + `body.path`, HTTP Basic Auth, binary body from the `data` field (60 s timeout).
4. **Итог загрузки** (`n8n-nodes-base.code`, v2) — reads `statusCode` and reports what happened: uploaded, replaced, denied, no such folder, disk full.

```json
{ "path": "backups/report.txt", "fileName": "report.txt",
  "status": 201, "ok": true, "note": "файл загружен" }
```

## Why not a native node

n8n has no Yandex node. Yandex Disk exposes a **WebDAV** endpoint next to its REST API, and WebDAV needs nothing more than Basic Auth and a `PUT` — so this template is a straight three-step pipeline: base64 in, binary out, `PUT` to the disk. There is no OAuth dance, no access-token refresh, no polling of an asynchronous upload operation. Authentication is an ordinary **HTTP Basic Auth** credential (Yandex login + app password), and the only Yandex-specific value in the workflow is the base URL `https://webdav.yandex.ru/`.

## Requirements

- n8n, self-hosted, recent 1.x (exported from a self-hosted instance, October 2026).
- **Credential: HTTP Basic Auth** — user = your Yandex login (full address), password = an **app password** for disk/WebDAV access (accounts with two-factor auth cannot use the account password here). Assign it to **Загрузить на Диск**; the exported JSON contains no credentials on purpose.
- The target folder must already exist on the disk — WebDAV `PUT` uploads a file, it does not create directories (that is exactly what the `409` branch in the итог node is for).
- `Content-Type: application/json` and a base64-encoded body. Base64 inflates the payload by roughly a third, so this is a tool for documents and reports, not for multi-gigabyte media.

## Setup

1. Import `workflow.json` (**Workflows → ⋯ → Import from File**).
2. Create the **HTTP Basic Auth** credential and assign it to **Загрузить на Диск**.
3. Replace `REPLACE_WITH_YOUR_TOKEN`. It appears exactly once — in **Запрос загрузки**, *Options → onlyRunIf*:
   `={{ $json.headers['x-watchdog-token'] === 'REPLACE_WITH_YOUR_TOKEN' }}`.
   Use a long random string; every caller must send it as the `x-watchdog-token` header.
4. Set your own disk path. Nothing to edit inside the nodes: `path` is a **request field**, concatenated after the hard-coded base URL `https://webdav.yandex.ru/` in **Загрузить на Диск**. Use a path relative to the disk root, without a leading slash (e.g. `backups/report.txt`).
5. Activate the workflow and test with a real file:

```bash
B64=$(base64 -w0 ./report.txt)        # GNU coreutils; on macOS: base64 -i ./report.txt
curl -X POST https://YOUR-N8N-HOST/webhook/yandex-disk-upload \
  -H 'x-watchdog-token: REPLACE_WITH_YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
        "path": "backups/report.txt",
        "fileName": "report.txt",
        "mimeType": "text/plain",
        "contentBase64": "'"$B64"'"
      }'
```

| Body field | Required | What it does |
|---|---|---|
| `path` | yes | Disk path appended to `https://webdav.yandex.ru/`; the parent folder must exist |
| `contentBase64` | yes | File content; the `toBinary` node decodes it into binary data |
| `fileName` | no | Name used inside the payload — default `file.bin` |
| `mimeType` | no | Payload MIME type — default `application/octet-stream` |

## Nodes

| Node | Type | Purpose |
|---|---|---|
| Запрос загрузки | `n8n-nodes-base.webhook` v2.1 | Receives `POST /yandex-disk-upload`, rejects calls without the `x-watchdog-token` header |
| Превратить в файл | `n8n-nodes-base.convertToFile` v1.1 | Decodes `body.contentBase64` into a binary item with file name and MIME type |
| Загрузить на Диск | `n8n-nodes-base.httpRequest` v4.5 | `PUT` the binary body to Yandex Disk over WebDAV, HTTP Basic Auth |
| Итог загрузки | `n8n-nodes-base.code` v2 | Maps `statusCode` to `ok` + a Russian `note`, returns the normalized JSON |

## Notes

- **The token check is the only guard.** When `onlyRunIf` is false the execution stops at the webhook and Yandex is never touched; the caller simply does not get the итог JSON. There is no second layer, so keep the endpoint behind HTTPS.
- **`path` is concatenated verbatim.** Spaces and non-ASCII characters are not percent-encoded for you — encode them in the caller, or use simple ASCII paths. A leading slash produces a double slash in the URL (`https://webdav.yandex.ru//backups/report.txt`), so send the path without one.
- **A missing `path` is not validated**: the URL collapses to `https://webdav.yandex.ru/` and the request targets the disk root. The failure is reported as a status, not thrown — check `ok`, not the HTTP transport code.
- **Same path = replacement.** Uploading to an existing path returns `204` ("файл заменён"), a new path returns `201` ("файл загружен"); the `ok` flag accepts both, so a retry overwrites instead of creating duplicates.
- **Status mapping lives in the Code node**: `401` → access denied (check the app password), `409` → no such folder (create it first), `507` → the disk is full, anything else → `код N`. `ok` and `status` are the machine-readable fields; `note` is a Russian human string and should not be parsed by callers.
- **`responseFormat: text` + `fullResponse: true` + `neverError: true`** on the HTTP node: the WebDAV reply body lands in `data` and the code in `statusCode`, and a 4xx/5xx does not fail the run. The итог node only looks at `statusCode` — the raw WebDAV response (headers, `ETag`) is discarded, and the caller's answer is the итог JSON.
- **One file per call, whole file in memory.** There is no chunking, no retry and a 60 s timeout (the node has no `retryOnFail`). For large or bulk uploads, loop on the caller side with your own retries.
- **Files up to the JSON body limit of your n8n instance only**: the payload travels as base64 inside a JSON request, so raise `N8N_PAYLOAD_SIZE_MAX` or switch to a binary/multipart variant if you need bigger files.

## Кратко по-русски

Вебхук `POST /yandex-disk-upload` принимает JSON (`path`, `contentBase64`, необязательные `fileName` и `mimeType`), узел `convertToFile` собирает бинарный файл из base64, а HTTP-узел кладёт его на Яндекс.Диск по WebDAV (`PUT https://webdav.yandex.ru/<path>`). Официального узла Яндекса в n8n нет — учётные данные обычные: HTTP Basic Auth (логин Яндекса + пароль приложения), назначить узлу «Загрузить на Диск». Папка назначения должна уже существовать: WebDAV-загрузка каталогов не создаёт (это и означает ответ `409`). Путь подставляется в адрес как есть — без ведущего слэша, пробелы и кириллицу кодировать на стороне вызывающего. Защита — заголовок `x-watchdog-token`, значение `REPLACE_WITH_YOUR_TOKEN` в поле onlyRunIf узла «Запрос загрузки». Ответ узла «Итог загрузки»: `path`, `fileName`, `status` (`201` загружено, `204` заменено), `ok`, `note`.

---

Made by [Bowhard](https://bowhard.ru) — automation studio. Template is provided as-is, MIT licensed.
