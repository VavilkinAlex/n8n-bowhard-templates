# Yandex Contacts Actions (CardDAV)

> Reads a Yandex address book over CardDAV with plain HTTP Request nodes and returns contacts as clean JSON — list or search.

**Category:** Yandex (CardDAV) · **Nodes:** 5 · **Trigger:** Webhook `POST /yandex-contacts` · **Tested on:** n8n 1.x self-hosted (October 2026)

## What it does

n8n ships no Yandex node, so this template talks to the Yandex address book the way a CardDAV client does:
`PROPFIND` to enumerate the cards, a `REPORT` with `addressbook-multiget` to fetch them in one round trip,
then vCard parsing in a Code node. Two Code nodes do the heavy lifting — one builds the multiget request from
the returned `href`s, the other unfolds and parses the vCards and applies the requested action. A single
webhook drives both `list` and `search`, so one endpoint can serve a CRM lookup, an autocomplete field or an
agent tool.

Chain, node by node:

1. **Запрос контактов** (`webhook`) — accepts `POST /yandex-contacts`, guarded by the `x-watchdog-token` header.
   Response mode is `lastNode`: the JSON produced by the last node is the HTTP response.
2. **Список карточек** (`httpRequest`) — `PROPFIND` with `Depth: 1` against the address book URL
   (`.../addressbook/YOUR-LOGIN%40yandex.ru/<book-id>/`), body `<d:propfind><d:prop><d:getetag/><d:getcontenttype/></d:prop></d:propfind>`,
   Basic Auth, `neverError: true`, 30 s timeout.
3. **Собрать запрос карточек** (`code`) — scans the XML for `href` values, drops the address book itself, and
   builds a `<c:addressbook-multiget>` body listing every card plus `getetag` and `address-data`. Emits
   `multiget`, `cardCount` and the PROPFIND status in `listStatus`.
4. **Забрать карточки** (`httpRequest`) — `REPORT` with `Depth: 1`, raw `application/xml` body
   `{{ $json.multiget }}`, Basic Auth, `neverError: true`, 60 s timeout.
5. **Разобрать ответ** (`code`) — reads the original webhook body back with `$("Запрос контактов").first().json.body`,
   unfolds vCard continuation lines, parses `FN` / `N` / `EMAIL` / `TEL` / `ORG` / `TITLE`, sorts by name
   (Russian locale), applies `action` / `query` / `limit`, and returns the report below.

Request body fields: `action` (`list` — default, or `search`), `query` (substring, case-insensitive),
`limit` (default 20, clamped to 1…200). `search` matches against name, organization, title, e-mails and phones.

Response shape:

```json
{
  "action": "search",
  "query": "acme",
  "total": 137,
  "matched": 4,
  "returned": 4,
  "truncated": false,
  "listStatus": 207,
  "reportStatus": 207,
  "note": "ответ получен",
  "contacts": [
    { "name": "Jane Doe", "emails": ["jane@example.com"], "phones": ["+10000000000"], "organization": "Acme" }
  ]
}
```

Each returned contact carries at most 3 e-mails and 3 phones. The workflow is read-only — it never writes to
the address book.

## Requirements

- n8n (self-hosted, recent 1.x). No community nodes — only `webhook`, `httpRequest` and `code`.
- **HTTP Basic Auth** credential for Yandex Contacts: your Yandex login and an **app password** created for
  CardDAV (a regular account password will be rejected), both cut out of the published JSON.
- A Yandex address book shared with that login. The book id is the last path segment of the collection URL.
- Outbound HTTPS access to `carddav.yandex.ru` from the n8n host.

## Setup

1. Import `workflow.json` (Workflows → ⋯ → Import from File).
2. Create the credentials and assign them to **Список карточек** and **Забрать карточки**
   (both nodes use type `HTTP Basic Auth`):
   - *User*: `YOUR-LOGIN@yandex.ru`, *Password*: the app password for CardDAV.
3. Replace the placeholder token in **Запрос контактов** → Options → *Only Run If*:
   `{{ $json.headers['x-watchdog-token'] === 'REPLACE_WITH_YOUR_TOKEN' }}`.
   Pick a long random value; there is no other authentication on this webhook.
4. Point both HTTP nodes at **your** address book. Replace the URL
   `https://carddav.yandex.ru/addressbook/YOUR-LOGIN%40yandex.ru/1/` in **Список карточек** and
   **Забрать карточки** with your own collection address (find it in the Yandex 360 Contacts → export/CardDAV
   settings). Keep the `@` percent-encoded as `%40`. Also update the `book` constant at the top of the
   **Собрать запрос карточек** Code node, otherwise its filter compares against the old path — the `REPORT`
   body is built from absolute `href`s, so the two must stay consistent.
5. Activate the workflow and test it:

```bash
curl -X POST https://YOUR-N8N-HOST/webhook/yandex-contacts \
  -H 'x-watchdog-token: REPLACE_WITH_YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"action":"list","limit":5}'
```

```bash
curl -X POST https://YOUR-N8N-HOST/webhook/yandex-contacts \
  -H 'x-watchdog-token: REPLACE_WITH_YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"action":"search","query":"example.com","limit":20}'
```

## Nodes

| Node | Type | Purpose |
|---|---|---|
| Запрос контактов | `n8n-nodes-base.webhook` | `POST /yandex-contacts` entry point with the shared-secret header check; responds with the last node's JSON. |
| Список карточек | `n8n-nodes-base.httpRequest` | CardDAV `PROPFIND` (`Depth: 1`) listing every card in the address book. |
| Собрать запрос карточек | `n8n-nodes-base.code` | Extracts card `href`s from the multistatus XML and builds the `addressbook-multiget` REPORT body. |
| Забрать карточки | `n8n-nodes-base.httpRequest` | CardDAV `REPORT` that returns the full vCard data for all cards in one request. |
| Разобрать ответ | `n8n-nodes-base.code` | Unfolds and parses vCards, sorts contacts, applies `list`/`search` + `limit`, returns the report. |

## Notes

- **`webdavMethods: true` is required** on both HTTP nodes. Without it the node refuses the non-standard
  `PROPFIND` / `REPORT` verbs. Their XML body is sent as raw content (`application/xml; charset=utf-8`),
  not as a JSON or form body.
- **`neverError: true` is deliberate.** A wrong app password returns `401`, which would normally abort the
  whole execution; instead the status is carried into the report as `listStatus` / `reportStatus`. Treat `207`
  (Multi-Status) as success — that is what CardDAV returns. The Code node translates `401` into
  *"отказ в доступе — проверить пароль приложения для контактов"* and any other non-`207` code into `"код N"`.
- **`responseFormat: autodetect` + `fullResponse: true`.** CardDAV replies with `text/xml`; with an explicit
  `text` or `json` format the node breaks on this content type. With `fullResponse` the XML arrives in the
  `data` field, which is what both Code nodes read — not in a body you can address directly.
- **The Code nodes avoid array/string helpers on purpose** (built with character loops instead of
  `.filter` / `.map` / `.join`), so they can also be round-tripped through the n8n workflow SDK when this
  file is regenerated. Keep that style if you edit them.
- **vCard unfolding matters.** Long values are wrapped with a leading space or tab on the next line; if that
  is not undone, telephones and long organization names come back truncated and sometimes get parsed as
  separate broken lines.
- **Absolute `href`s are assumed.** `Собрать запрос карточек` filters `href` values containing `/addressbook/`
  and drops the trailing-slash duplicate of the collection itself. If Yandex ever returns relative hrefs, the
  multiget body must be rewritten to prefix them with the book path.
- **`limit` is clamped to 1…200 and applied after sorting**, so `total` is the size of the whole address book,
  `matched` is what survived the `search` filter, and `truncated` tells you that `returned < matched`.
- Both Code nodes run in **runOnceForAllItems** mode: the workflow is one item deep, so per-item semantics do
  not apply.
- Timeouts are 30 s for the listing and 60 s for the multiget. Very large address books (thousands of cards)
  can exceed the latter — raise it and consider paging the multiget.

## Кратко по-русски

Шаблон читает книгу контактов Яндекса по CardDAV обычными HTTP-узлами, потому что узла Яндекса в n8n нет:
`PROPFIND` перечисляет карточки, `REPORT` с `addressbook-multiget` забирает их за один запрос, два Code-узла
разбирают XML и vCard. Один вебхук `POST /yandex-contacts` обслуживает и список (`action: list`), и поиск
(`action: search`, `query`, `limit` до 200); ответ — JSON с полями `name`, `emails`, `phones`, `organization`
(до трёх значений каждого) и служебными `total` / `matched` / `truncated`.

Что заменить: пароль приложения Яндекса для CardDAV (обычный пароль не подойдёт) в обеих HTTP-узлах, адрес
книги в них и константу `book` в первом Code-узле, токен `REPLACE_WITH_YOUR_TOKEN` в условии вебхука.
Запись в книгу шаблон не делает — только чтение.

---

Made by [Bowhard](https://bowhard.ru) — automation studio. Template is provided as-is, MIT licensed.
