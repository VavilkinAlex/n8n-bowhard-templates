# Voice Assistant — Knowledge-Base Brain

> The "brain" behind a phone assistant: takes a recognized phrase from a telephony scenario and answers it from a hard-coded topic table, with no LLM and no external API.

**Category:** AI · Voice · **Nodes:** 7 (6 working nodes + 1 sticky note) · **Trigger:** Webhook `POST /voice-brain` · **Tested on:** n8n 1.x self-hosted (October 2026)

## What it does

This is the answer engine of a voice bot. A telephony scenario (the template was built for VoxEngine /
Voximplant, but any HTTP client fits) sends the speech-to-text result of the caller's phrase; the workflow
returns the phrase to say back. All the intelligence is one Code node: input validation, keyword scoring
against a table of topics, a fallback ladder, and phone-number capture. Nothing is sent to an external
service, so a reply is deterministic, instant and free.

1. **`Принять запрос ассистента`** — webhook accepts `POST /voice-brain`. The node has
   `authentication: none` and `responseMode: responseNode`: the token check happens inside the code, and
   one of the three Respond nodes sends the reply.
2. **`Ответ по базе знаний`** — the Code node. It compares the `x-brain-token` header (or `body.token`)
   with the hard-coded `TOKEN`, validates `text`, then scores the phrase against a 42-entry knowledge-base
   table. Returns `valid`, `httpStatus`, `errorCode`, `errorMessage`, `intent`, `answer`, `score`,
   `contact`, `heard`.
3. **`Запрос корректен?`** — an If node checks `{{ $json.valid }}` is `true` (strict type validation on a
   boolean).
4. **`Ответ клиенту`** (true branch) — HTTP 200 with `{ answer, intent, contact }`. The caller reads
   `answer` and speaks it.
5. **`Ошибка запроса`** (false branch) — HTTP status taken from the data
   (`responseCode: ={{ $json.httpStatus }}`, i.e. 401 or 400) with `{ error, message }`.
6. **`Внутренняя ошибка`** — if the Code node itself throws (`onError: continueErrorOutput`), this node
   answers HTTP 500 with `{ error: "internal_error", message: "Сбой в работе ассистента" }`.

Behaviour baked into the code:

- **Phone capture wins over everything.** Any phrase containing 10+ digits is treated as a phone number,
  normalized to `7XXXXXXXXXX` (a leading `8` becomes `7`, a bare 10-digit number gets a `7` prefix, longer
  values are cut at 20 characters) and returned as `contact` with `intent: phone_given`.
- **Two-topic answers.** If the caller asks about two things at once ("how much and how fast"), the top
  topic's full answer is followed by a one-line `brief` of the second topic — but only while the result
  stays within `COMBINE_LIMIT = 280` characters. Otherwise both topics are answered briefly.
- **A fallback ladder instead of one "I don't understand".** A short unrecognized phrase (≤ 2 words) gets
  `FALLBACK_SHORT` (an invitation to clarify), a longer one gets `FALLBACK`, and when the caller's
  `fallbacks` counter is already ≥ 1 the assistant stops guessing and asks for a phone number
  (`FALLBACK_ESCALATED`).

## Requirements

- n8n, self-hosted, recent 1.x. No LangChain nodes are needed.
- **Credentials:** none. The workflow calls no external service at all — no LLM, no database, no messenger.
- **External services:** none, apart from the telephony side that calls the webhook.
- **One shared secret of your own choosing**, placed in the Code node as `TOKEN` and sent by the caller in
  the `x-brain-token` header. That is the only access control this template has.
- A caller that can POST `{ text, callId, fallbacks }` — a VoxEngine scenario, a script, or `curl`.

## Setup

1. Import `workflow.json` (n8n → Workflows → ⋯ → **Import from File**).
2. Create credentials: there are none to create. Assign nothing; the template runs as imported.
3. Replace the placeholder. In the Code node **`Ответ по базе знаний`** the first statements are:

   ```js
   const TOKEN = 'REPLACE_WITH_YOUR_TOKEN';
   ```

   Put a long random string there, and give the same value to your telephony scenario, which must send it
   as the `x-brain-token` request header (a `token` field in the JSON body is also accepted as a
   fallback). While the placeholder is unchanged, only a request carrying the literal string
   `REPLACE_WITH_YOUR_TOKEN` is accepted.
4. Set your own knowledge base in the **same Code node**. It is the `KB` array of 42 entries; each entry
   is `{ intent, topic, keywords, answer, brief }` plus optional `solo`, `final`, `onlyShort` and
   `maxWords`. Edit the answers and keywords in place, and replace the three `FALLBACK*` strings and
   `COMBINE_LIMIT` if your voice and pacing differ. There is no other place where answers live.
5. Activate the workflow and test it:

```bash
curl -X POST https://YOUR-N8N-HOST/webhook/voice-brain \
  -H 'Content-Type: application/json' \
  -H 'x-brain-token: REPLACE_WITH_YOUR_TOKEN' \
  -d '{"text":"сколько стоит голосовой ассистент","callId":"demo-1","fallbacks":0}'
```

Success (HTTP 200), shape taken from the `Ответ клиенту` node:

```json
{ "answer": "...", "intent": "voice", "contact": "" }
```

Failure — wrong or missing token (HTTP 401), shape from the `Ошибка запроса` node:

```json
{ "error": "unauthorized", "message": "Неверный токен запроса" }
```

## Nodes

| Node | Type | Purpose |
|---|---|---|
| `Принять запрос ассистента` | `n8n-nodes-base.webhook` v2.1 | Accepts `POST /voice-brain` with no node-level auth and defers the reply to a Respond node. |
| `Ответ по базе знаний` | `n8n-nodes-base.code` v2 | Validates token and text, scores the phrase against the 42-topic knowledge base and composes the answer. |
| `Запрос корректен?` | `n8n-nodes-base.if` v2.3 | Routed on the boolean `valid` field returned by the Code node. |
| `Ответ клиенту` | `n8n-nodes-base.respondToWebhook` v1.5 | Returns HTTP 200 with `{ answer, intent, contact }` for a valid request. |
| `Ошибка запроса` | `n8n-nodes-base.respondToWebhook` v1.5 | Returns the validation error with the status the Code node chose (401 unauthorized, 400 bad `text`). |
| `Внутренняя ошибка` | `n8n-nodes-base.respondToWebhook` v1.5 | Returns HTTP 500 `{ error: "internal_error" }` when the Code node throws. |
| `Как это работает` | `n8n-nodes-base.stickyNote` v1 | Canvas documentation of the contract: `{ text }` in, `x-brain-token` header, `{ answer }` out. |

## Notes

- **The webhook node itself does not authenticate.** `authentication` is `none` on
  `Принять запрос ассистента`; the `if (token !== TOKEN)` check inside the Code node is the only gate,
  and it answers 401 before any scoring happens. Keep the token long, and treat the webhook path as
  semi-public.
- **Hard input limits in the code:** an empty `text` returns HTTP 400 `validation_error`
  "Нужно непустое поле text", and a `text` longer than 500 characters returns 400 as well. Speech
  recognizers occasionally emit a long mis-transcription, and it is rejected by design.
- **Validation errors are data, not exceptions.** `bad(...)` returns `valid: false` with an `httpStatus`,
  the If node takes the false branch, and `Ошибка запроса` sets `responseCode: ={{ $json.httpStatus }}`.
  `onError: continueErrorOutput` on the Code node only catches thrown script errors and routes them to
  `Внутренняя ошибка` (HTTP 500).
- **The If node is strict.** `typeValidation: strict` with a boolean comparison on `$json.valid` means the
  Code node must return a real boolean in every branch — returning `"true"` as a string makes the node
  error out instead of taking a branch.
- **The workflow is stateless; `fallbacks` comes from the caller.** n8n does not count consecutive
  misunderstandings — the telephony scenario does, and passes the number in. With `fallbacks >= 1` the
  assistant switches to `FALLBACK_ESCALATED` and asks for a phone number instead of trying again.
- **Phone detection is deliberately blunt.** Any phrase with 10 or more digits is captured as a phone
  number and short-circuits topic scoring, so a caller reading out an order number may be treated as a
  contact. The number is normalized to `7XXXXXXXXXX` and returned in `contact` for the CRM.
- **Answer text is written for text-to-speech, and the constraints are load-bearing.** The comments in the
  code record live synthesis/recognition tests (06.10.2026): digits are never used (only words such as
  "от пятнадцати тысяч"), "ИИ" is never written (it is read as "и"), "1С" is always spelled "один-эс", and
  URLs or e-mail addresses are never dictated. Keep these rules when you edit the answers.
- **Length limits come from real calls.** Every answer is kept under roughly 250 characters (~15 seconds
  of speech) and a combined answer is capped by `COMBINE_LIMIT = 280`. The code comments note a live call
  where a 400-character (26-second) answer made the caller hang up, and a second where two consecutive
  26-second price answers ended the call.
- **Keyword matching is anchored at word start, and some topics are intentionally restricted.** "заявк"
  matches "заявка/заявки" but "бот" does not match "работа"; multi-word phrases score double. `solo`
  topics (a human request, "wait", "are you a bot", "repeat") are never merged into a combined answer —
  the comments record a 22-second tautology that this prevents. `onlyShort` and `maxWords` keep a topic
  tied to short utterances, so "да" is consent while "давно хотел спросить" is not, and "пока" is a
  goodbye while "покажите примеры" is not.
- **There is no LLM here, by design.** Answers are exact strings from the table, so nothing can be
  hallucinated and nothing is billed — but every new topic is a code change, and the assistant cannot
  handle a phrasing that its keyword lists miss.

## Кратко по-русски

Это «мозг» голосового ассистента: вебхук `POST /voice-brain` принимает `{ text }` от сценария телефонии
и отдаёт `{ answer, intent, contact }`. Вся логика — в узле Code: проверка токена `x-brain-token`,
ограничение длины реплики, скоринг по таблице из 42 тем и подсказка после повторного непонимания.
Внешних сервисов и учётных данных нет: отвечает не нейросеть, а заранее написанные фразы, поэтому
ответы детерминированы и бесплатны. Замените заглушку `REPLACE_WITH_YOUR_TOKEN` в узле `Ответ по
базе знаний`, перепишите там же базу знаний (массив `KB`): без цифр, без «ИИ», до ~250 знаков.

---

Made by [Bowhard](https://bowhard.ru) — automation studio. Template is provided as-is, MIT licensed.
