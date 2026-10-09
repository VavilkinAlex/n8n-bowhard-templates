# Daily Deals Report to Telegram

> Every morning at 09:00 pulls yesterday's deals from a CRM API, computes the totals and posts a formatted report to Telegram.

**Category:** Reporting · **Nodes:** 4 · **Trigger:** Schedule (daily at 09:00) · **Tested on:** n8n 1.x self-hosted (October 2026)

## What it does

Scheduled reporting without a BI tool. The workflow fetches yesterday's deals from an HTTP API, reduces
them to four numbers and sends one Telegram message. There is no database, no dashboard and no state — each
run is independent, which is exactly why it can be trusted to keep running unattended.

1. **`Каждый день в 09:00`** — Schedule Trigger, `days` interval of 1, `triggerAtHour: 9`,
   `triggerAtMinute: 0`.
2. **`Забрать сделки из CRM`** — HTTP Request, `GET https://api.example-crm.ru/v1/deals?period=yesterday`
   (placeholder host — replace it). 3 retries, 5 s apart.
3. **`Свести отчёт`** — a Set node that keeps only four computed fields: `totalLeads`
   (`items.length`), `totalAmount` (sum of `amount`), `wonDeals` (count of `status === "won"`) and
   `period` (`$now.minus({ days: 1 })` formatted as `dd.MM.yyyy`).
4. **`Отправить отчёт в Telegram`** — Telegram `sendMessage` with the ready-made text template:
   `📊 Отчёт за {{ $json.period }}`, then the lead count, the sum in roubles and the won-deal count.

## Requirements

- n8n, self-hosted, recent 1.x.
- **Credentials to create:** a **Telegram API** credential (`telegramApi`). The original workflow had one
  attached to the Telegram node and it was stripped from the export before publishing, so the imported node
  has no credential — create your own bot token (a bot from @BotFather, added to the chat you report to)
  and assign it to `Отправить отчёт в Telegram`.
- **External services:**
  - a CRM or any HTTP API that returns yesterday's deals as a JSON object with an `items` array;
  - Telegram, for delivery.
  - No LLM, no Google Sheets, no database.

## Setup

1. Import `workflow.json` (n8n → Workflows → ⋯ → **Import from File**).
2. Create the Telegram credential and assign it to **`Отправить отчёт в Telegram`** (node → *Credential* →
   *Create new* → Telegram API → paste the bot token).
3. Replace the placeholders and the sample data:
   - **`chatId`** in the Telegram node holds the placeholder **`REPLACE_WITH_YOUR_CHAT_ID`**. Replace it with
     your numeric user id, group id or channel id. This is the only placeholder in the file; no service
     token was stripped into the workflow, only the credential.
   - **`url`** in `Забрать сделки из CRM` is the sample
     `https://api.example-crm.ru/v1/deals?period=yesterday`. Point it at your own API. If that API needs
     authentication, add it on the node (*Authentication* / *Header Auth*) — none is configured as
     exported.
   - If your API expects a different period argument, align it with the `period` expression in the Set
     node, which always computes *yesterday*.
4. Set the report content where it lives: the four metrics in **`Свести отчёт`**, the message wording in
   **`Отправить отчёт в Telegram`**.
5. Activate the workflow and test it. This template has **no webhook to curl** — the trigger is a
   schedule:
   - in the editor, click **Execute workflow** to run it immediately;
   - or temporarily replace the Schedule Trigger with a Manual Trigger while you are tuning the
     expressions.
   To check the API response shape first, call your endpoint directly:

```bash
curl 'https://api.example-crm.ru/v1/deals?period=yesterday'
```

The Set node expects this envelope:

```json
{
  "items": [
    { "amount": 12000, "status": "won" },
    { "amount": 3000,  "status": "open" }
  ]
}
```

## Nodes

| Node | Type | Purpose |
|---|---|---|
| `Каждый день в 09:00` | `n8n-nodes-base.scheduleTrigger` v1.4 | Starts the workflow once a day at 09:00 in the instance timezone. |
| `Забрать сделки из CRM` | `n8n-nodes-base.httpRequest` v4.5 | `GET`s yesterday's deals from the CRM endpoint, with 3 retries 5 s apart. |
| `Свести отчёт` | `n8n-nodes-base.set` v3.5 | Reduces the payload to `totalLeads`, `totalAmount`, `wonDeals` and `period`. |
| `Отправить отчёт в Telegram` | `n8n-nodes-base.telegram` v1.2 | Sends the formatted report with `sendMessage`, with 3 retries 5 s apart. |

## Notes

- **09:00 means 09:00 in the instance timezone.** The workflow's `settings` block contains only
  `executionOrder: v1` and no `timezone`, so the trigger hour follows the n8n instance (and the container's
  `TZ`). A server on UTC sends the report at 09:00 UTC, not Moscow time.
- **The Set node hard-codes the response shape.** `$json.items.length`, `$json.items.reduce(...)` and
  `$json.items.filter(...)` all assume `items` sits at the root of the HTTP response. A CRM that wraps the
  list (for example `{ "data": { "deals": [] } }`) breaks all four assignments — adjust the paths in
  `Свести отчёт` before activating.
- **`includeOtherFields: false`.** The raw CRM payload is discarded; only the four computed fields reach
  Telegram. That is good for privacy, but if you want to attach the raw deal list or a link, you have to
  re-enable the flag or add the fields explicitly.
- **Aggregation details worth knowing:** `amount` is coerced with `Number(i.amount || 0)`, so numeric
  strings work and missing amounts count as zero; `wonDeals` counts only an exact, case-sensitive
  `status === "won"` — `Won`, `success` or a localized status counts as zero.
- **Retries but no error branch.** Both the HTTP Request and the Telegram node have `retryOnFail: true`,
  `maxTries: 3`, `waitBetweenTries: 5000`. There is no error output and no error workflow configured, so
  if the CRM is still down after the third attempt the execution simply fails and the morning report never
  arrives — attach an error workflow if the report matters.
- **"Yesterday" is computed twice and must agree.** The URL asks the API for `period=yesterday` while the
  Set node computes the displayed date with `$now.minus({ days: 1 })`. If your API interprets the period
  differently (a rolling 24 hours, a working-day calendar), the label and the numbers will drift apart.
- **The message uses the `₽` sign and a 📊 emoji** with `replyMarkup: none`. Some Telegram clients and
  channels render the emoji differently, and a channel post with a `replyMarkup` is not possible anyway —
  leave it at `none`.

## Кратко по-русски

Шаблон «Ежедневный отчёт по сделкам в Telegram»: планировщик запускает процесс каждый день в 09:00,
HTTP-узел забирает вчерашние сделки из API CRM, узел Set считает число заявок, сумму и число успешных
сделок, а узел Telegram присылает готовый отчёт в чат. Учётные данные Telegram из файла вырезаны —
создайте своё подключение с токеном бота и назначьте его узлу отправки. В узле Telegram осталась
заглушка `REPLACE_WITH_YOUR_CHAT_ID`, адрес `api.example-crm.ru`
— пример, замените на свой. Вебхука нет: запускайте кнопкой «Execute workflow» или ручным триггером.

---

Made by [Bowhard](https://bowhard.ru) — automation studio. Template is provided as-is, MIT licensed.
