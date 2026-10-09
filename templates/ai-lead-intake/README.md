# AI Lead Intake

> One webhook for every channel: accepts a lead, extracts data from an image attachment with a vision LLM, triages it with a second LLM call, writes the card to your CRM and pings the manager in Telegram.

**Category:** Lead intake · **Nodes:** 12 · **Trigger:** Webhook `POST /ai-lead-intake` · **Tested on:** n8n 1.x self-hosted (October 2026)

## What it does

Takes a single JSON POST from any lead source (site form, Telegram, MAX, VK, an e-mail bridge), turns it into a normalised lead card, and answers the caller with the classified lead. A vision model is called only when the payload carries an attachment URL; a second, text-only model call classifies the request, decides the priority and drafts a reply. The card is POSTed to your CRM endpoint and copied to the manager in Telegram.

1. `Заявка из канала` receives `POST /ai-lead-intake` (response mode: Respond to Webhook node).
2. `Настройки` injects four config values: `llm_base_url`, `llm_model`, `crm_url`, `manager_chat_id`.
3. `Нормализация заявки` maps the body into `lead_id` (`L-yyyyMMdd-HHmmss`), `channel`, `name`, `contact`, `message`, `company`, `page`, `attachment_url`, `received_at`.
4. `Есть вложение?` checks that `attachment_url` is not empty.
5. True branch → `Распознать вложение (ИИ)` asks the vision model to extract full name, phone, e-mail, scope, budget and deadline from the attachment.
6. `Склеить ветки` (append merge) joins the attachment branch and the direct branch.
7. `Собрать текст заявки` builds the plain-text lead summary — attachment data, or the literal `вложения нет`.
8. `Разбор заявки (ИИ)` classifies the lead and returns strict JSON: `category`, `priority`, `summary`, `what_needed`, `missing`, `draft_reply`, `needs_human`.
9. `Карточка заявки` (Code) parses that JSON into one flat item.
10. `Записать в CRM` POSTs the card to `crm_url`.
11. `Пинг менеджеру` sends the formatted card to `manager_chat_id`.
12. `Ответ каналу` returns `{ ok, lead_id, category, priority, draft_reply, needs_human }` with HTTP 200.

## Requirements

- n8n self-hosted, recent 1.x (node versions in the export: webhook v2, set v3.4, if v2.2, merge v3.2, code v2, httpRequest v4.2, telegram v1.2, respondToWebhook v1.1).
- Credentials to create:
  - **HTTP Header Auth** — both LLM nodes authenticate with `genericCredentialType` / `httpHeaderAuth` (for example header `Authorization: Bearer <your key>`). No credential values are included in the export.
  - **Telegram** (`telegramApi`) for `Пинг менеджеру`.
- External services:
  - an OpenAI-compatible LLM API at `llm_base_url` — the export points at `https://api.proxyapi.ru/openai/v1` with model `gpt-4o-mini`; the attachment call needs image input support;
  - a CRM intake endpoint reachable at `crm_url` (the placeholder suggests a Bitrix24 webhook, Google Sheets or Airtable).
- `Записать в CRM` has no authentication configured in the export — add a credential or a header yourself if your endpoint requires one.

## Setup

1. Import `workflow.json` (n8n → Workflows → Import from File).
2. Create the credentials above and assign them to `Распознать вложение (ИИ)`, `Разбор заявки (ИИ)` (HTTP Header Auth) and `Пинг менеджеру` (Telegram).
3. Replace the placeholders — all of them live in the `Настройки` node:
   - `manager_chat_id` = `REPLACE_WITH_YOUR_CHAT_ID` — the manager's Telegram chat id;
   - `crm_url` = `<__PLACEHOLDER_VALUE__URL приёма заявок в CRM (вебхук Битрикс24 / Google Sheets / Airtable)__>` — your CRM intake URL;
   - `llm_base_url` and `llm_model` — your provider base URL and model name. Both HTTP nodes append `/chat/completions`, so keep the base URL without that suffix.
   - No `REPLACE_WITH_YOUR_TOKEN` placeholders exist in this export.
4. Activate the workflow and call the webhook:

```bash
curl -X POST https://YOUR-N8N-HOST/webhook/ai-lead-intake \
  -H 'Content-Type: application/json' \
  -d '{"channel":"site","name":"Ivan Petrov","contact":"ivan@example.com","message":"Please automate our invoicing","company":"Example LLC","page":"https://example.com/pricing","attachment_url":""}'
```

Only these body keys are read: `channel` (defaults to `сайт`), `name`, `contact`, `message` (falls back to `text`), `company`, `page`, `attachment_url`. Every missing field becomes an empty string.

## Nodes

| Node | Type | Purpose |
|---|---|---|
| `Заявка из канала` | `n8n-nodes-base.webhook` v2 | Entry point: accepts `POST /ai-lead-intake` and answers through a Respond to Webhook node. |
| `Настройки` | `n8n-nodes-base.set` v3.4 | Holds the four configuration values (LLM base URL, model, CRM URL, manager chat id) and keeps the incoming fields. |
| `Нормализация заявки` | `n8n-nodes-base.set` v3.4 | Maps the request body into fixed lead fields and stamps `lead_id` and `received_at`. |
| `Есть вложение?` | `n8n-nodes-base.if` v2.2 | Routes leads with a non-empty `attachment_url` into the vision branch. |
| `Распознать вложение (ИИ)` | `n8n-nodes-base.httpRequest` v4.2 | Vision LLM call that extracts name, phone, e-mail, scope, budget and deadline from the attachment. |
| `Склеить ветки` | `n8n-nodes-base.merge` v3.2 | Append-mode merge that brings the attachment branch and the direct branch back into one stream. |
| `Собрать текст заявки` | `n8n-nodes-base.set` v3.4 | Builds the plain-text lead summary, including the attachment data, for the triage call. |
| `Разбор заявки (ИИ)` | `n8n-nodes-base.httpRequest` v4.2 | Second LLM call: classifies the lead and returns strict JSON (category, priority, summary, what is needed, what is missing, draft reply, needs_human). |
| `Карточка заявки` | `n8n-nodes-base.code` v2 | Parses the model's JSON answer into one flat lead card and falls back to a safe default. |
| `Записать в CRM` | `n8n-nodes-base.httpRequest` v4.2 | POSTs the lead card as JSON to your CRM intake URL. |
| `Пинг менеджеру` | `n8n-nodes-base.telegram` v1.2 | Sends the formatted lead card to the manager's Telegram chat. |
| `Ответ каналу` | `n8n-nodes-base.respondToWebhook` v1.1 | Returns `{ ok, lead_id, category, priority, draft_reply, needs_human }` with HTTP 200. |

## Notes

- **Mixed error handling.** `Распознать вложение (ИИ)`, `Записать в CRM` and `Пинг менеджеру` are set to `onError: continueRegularOutput`, so their failures do not stop the run. `Разбор заявки (ИИ)` has **no** `onError` — a failed triage call ends the execution before `Ответ каналу`, and the caller receives no JSON body.
- **Retries and timeouts.** The attachment call and the triage call retry twice, 3 s apart; the CRM call retries twice, 3 s apart. Timeouts: 60 s (attachment), 90 s (triage), 30 s (CRM).
- **The attachment call is image-shaped.** It sends the OpenAI `image_url` content part, so `attachment_url` must be a URL your provider accepts as an image — a PDF link usually will not work.
- **A lost attachment degrades gracefully.** When there is no attachment, or the vision call fails, `Собрать текст заявки` still gets an item without `choices` and writes the literal `вложения нет`; triage continues on the text alone.
- **The Code node never throws.** It extracts the first `{...}` block from the model answer and parses it; if that fails it emits `category: "не разобрано"`, `priority: "средний"`, `needs_human: true`.
- **Fixed vocabularies.** The triage prompt hard-codes categories `консультация / заказ / документы / жалоба / партнёрство` and priorities `высокий / средний / низкий`. Both prompts are written in Russian — edit them inside the two HTTP nodes to change language or categories.
- **Field hygiene between the two Set nodes.** `Настройки` keeps the incoming fields (`includeOtherFields: true`), while `Нормализация заявки` drops them (`includeOtherFields: false`) and rebuilds a clean object, so the raw request body is not forwarded to the CRM.
- **The card is forwarded as-is:** `Записать в CRM` posts `{{ $json }}` with no authentication configured.

## Кратко по-русски

- Один вебхук `POST /ai-lead-intake` принимает заявку из любого канала (сайт, Telegram, MAX, ВК, почта).
- Если в теле есть `attachment_url`, вложение разбирает vision-модель: ФИО, телефон, e-mail, задача, бюджет, срок.
- Второй вызов LLM классифицирует заявку, ставит приоритет и пишет черновик ответа; карточка уходит в CRM и менеджеру в Telegram.
- Заменить нужно в узле «Настройки»: `manager_chat_id` (`REPLACE_WITH_YOUR_CHAT_ID`), `crm_url`, `llm_base_url`, `llm_model`; учётные данные — HTTP Header Auth (для LLM) и Telegram.
- Грабли: у «Разбор заявки (ИИ)» нет `onError` — при сбое вызова ответ вебхуку не уйдёт; остальные три узла при ошибке продолжают работу.
- Промпты и словари (категории, приоритеты) лежат прямо в узлах и написаны по-русски — правятся там же.

Made by [Bowhard](https://bowhard.ru) — automation studio. Template is provided as-is, MIT licensed.
