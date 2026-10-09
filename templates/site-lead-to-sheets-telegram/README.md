# Site Lead to Google Sheets + Telegram

> Takes a website form submission, appends it to a Google Sheets CRM, notifies the manager in Telegram, and answers the page with JSON in both the success and the failure case.

**Category:** Lead intake · **Nodes:** 6 · **Trigger:** Webhook `POST /site-lead` · **Tested on:** n8n 1.x self-hosted (October 2026)

## What it does

A small, production-shaped lead pipeline: the form posts JSON, the payload is normalised into fixed fields, the lead is appended as a row to a Google Sheets spreadsheet used as a CRM, the manager gets a Telegram message, and the page receives a JSON answer. Both external calls have error outputs wired to a dedicated failure response, so the form can show a real error instead of hanging.

1. `Заявка с сайта (POST)` receives `POST /site-lead` (response mode: Respond to Webhook node).
2. `Нормализовать заявку` turns the body into `name`, `phone`, `comment`, `source`, `receivedAt` with defaults.
3. `Сохранить в CRM (Google Sheets)` appends the item as a row to the `Заявки` sheet.
4. `Уведомить менеджера` sends the formatted lead to the manager's Telegram chat.
5. `Ответ: заявка принята` returns `{ ok: true, message: "Заявка принята", lead: <name> }` with HTTP 200.
6. `Ответ: ошибка обработки` returns `{ error: "internal_error", message: … }` with HTTP 500 — it is the target of both error outputs.

## Requirements

- n8n self-hosted, recent 1.x (node versions in the export: webhook v2.1, set v3.5, googleSheets v4.7, telegram v1.2, respondToWebhook v1.5).
- Credentials to create (no credential values are included in the export):
  - **Google Sheets OAuth2** for `Сохранить в CRM (Google Sheets)` — the node is configured with `authentication: oAuth2`;
  - **Telegram** (`telegramApi`) for `Уведомить менеджера`.
- External services:
  - a Google account with access to the spreadsheet you will use as the CRM;
  - a Telegram bot and the chat id that should receive the notifications.

## Setup

1. Import `workflow.json` (n8n → Workflows → Import from File).
2. Create the Google Sheets OAuth2 and Telegram credentials and assign them to `Сохранить в CRM (Google Sheets)` and `Уведомить менеджера`.
3. Replace the placeholders:
   - `Уведомить менеджера` → `chatId` = `REPLACE_WITH_YOUR_CHAT_ID` — replace with your own chat id (this is the only credential-ish placeholder in the export; there is no `REPLACE_WITH_YOUR_TOKEN` here);
   - `Сохранить в CRM (Google Sheets)` → `documentId` is **empty** in the export (it only carries the cached name `CRM — заявки (демо)`): select your own spreadsheet in the Document field.
4. Prepare the sheet: its name must be `Заявки` (that is the fixed `sheetName` value), and its header row must match the auto-mapped fields `name`, `phone`, `comment`, `source`, `receivedAt` — otherwise the columns are not filled.
5. Activate the workflow and call the webhook:

```bash
curl -X POST https://YOUR-N8N-HOST/webhook/site-lead \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ivan Petrov","phone":"+10000000000","comment":"Please call me back","source":"site"}'
```

Only `name`, `phone`, `comment` and `source` are read from the body. Missing values are replaced by the defaults in the next section.

## Nodes

| Node | Type | Purpose |
|---|---|---|
| `Заявка с сайта (POST)` | `n8n-nodes-base.webhook` v2.1 | Entry point: accepts `POST /site-lead` and answers through a Respond to Webhook node. |
| `Нормализовать заявку` | `n8n-nodes-base.set` v3.5 | Normalises the body into `name`, `phone`, `comment`, `source` and stamps `receivedAt` with `$now.toISO()`. |
| `Сохранить в CRM (Google Sheets)` | `n8n-nodes-base.googleSheets` v4.7 | Appends the normalised lead as a row to the `Заявки` sheet, auto-mapping the input fields to columns. |
| `Уведомить менеджера` | `n8n-nodes-base.telegram` v1.2 | Sends the formatted lead (name, phone, comment, source, time) to the manager's Telegram chat. |
| `Ответ: заявка принята` | `n8n-nodes-base.respondToWebhook` v1.5 | Returns `{ ok: true, message, lead }` with HTTP 200 after the lead is stored and the manager is notified. |
| `Ответ: ошибка обработки` | `n8n-nodes-base.respondToWebhook` v1.5 | Returns `{ error: "internal_error", message }` with HTTP 500; both error outputs lead here. |

## Notes

- **Normalisation defaults:** `name` and `phone` fall back to `не указано`, `comment` to an empty string, `source` to `site`. The lead is never rejected for missing fields.
- **Sheets node:** `onError: continueErrorOutput` with retries — 3 tries, 5 s apart. A failed append routes to `Ответ: ошибка обработки` (HTTP 500).
- **Telegram node:** the same settings (`continueErrorOutput`, 3 tries, 5 s apart), and its error output also goes to `Ответ: ошибка обработки`. So if the row was already appended but Telegram fails, the form still gets a 500 while the lead is safely stored.
- **The success response only proves the pipeline ran**, not that Sheets and Telegram both succeeded — the 200 node is reached through the normal output of `Уведомить менеджера`.
- **The sheet name is fixed** (`sheetName` mode `name`, value `Заявки`) while the document is a free choice; change both if you want another layout. Columns are taken from `autoMapInputData` with the schema `name`, `phone`, `comment`, `source`, `receivedAt`.
- **No deduplication:** every POST appends a new row, and the append node retries on failure, so a lost response after a successful write can produce a duplicate row.
- **The webhook is unauthenticated** — anyone who knows the path can add rows. Put it behind a secret path, a proxy rule or a header check if the endpoint is public.
- The Telegram message body reads the values from `Нормализовать заявку` by node name, so it always shows the normalised (defaulted) fields rather than the raw body.

## Кратко по-русски

- Что делает: вебхук `POST /site-lead` принимает заявку с формы, нормализует поля, дописывает строку в Google Sheets и уведомляет менеджера в Telegram.
- Ответ странице: 200 с `{ok, message, lead}` при успехе и 500 с `{error, message}` при сбое — обе ветки ошибок сходятся в узел «Ответ: ошибка обработки».
- Заменить: `chatId` = `REPLACE_WITH_YOUR_CHAT_ID` в узле «Уведомить менеджера» и пустой `documentId` в узле Google Sheets (выбрать свою таблицу).
- Лист должен называться `Заявки`, а шапка — совпадать с полями `name`, `phone`, `comment`, `source`, `receivedAt`, иначе колонки не заполнятся.
- Пустые поля не отбраковываются: имя и телефон станут «не указано», источник — `site`.
- Оба внешних узла повторяют попытку 3 раза с паузой 5 секунд; сбой Telegram после успешной записи всё равно отдаёт странице 500.

Made by [Bowhard](https://bowhard.ru) — automation studio. Template is provided as-is, MIT licensed.
