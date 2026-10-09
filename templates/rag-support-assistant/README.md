# RAG Support Assistant — Knowledge-Base Q&A

> A webhook that answers a customer question from an inlined knowledge base with an LLM and returns clean JSON for a chat widget.

**Category:** AI assistant · **Nodes:** 5 · **Trigger:** Webhook `POST /rag-ask` · **Tested on:** n8n 1.x self-hosted (October 2026)

## What it does

Turns a plain customer question into a support answer that is grounded in your own knowledge base. The
knowledge base is a text block sitting in a Set node, so the template works with no vector store and no
retrieval service — useful for a small, slowly changing FAQ (a few KB of text). The LLM is instructed to
answer only from that block and to offer a handover to a manager when the answer is not in it.

The chain is short and linear:

1. **`Вопрос клиента (POST)`** — webhook accepts a JSON `POST` on `/rag-ask`. `responseMode` is
   `responseNode`, so the HTTP reply is not sent here but by the Respond node at the end.
2. **`Собрать промпт с базой знаний`** — a Set node takes `body.question` (falling back to
   `Как с вами связаться?` when the field is missing) and glues it to the fixed knowledge-base block:
   delivery times, payment methods, returns, warranty and a contact address. Output fields: `question`
   and `prompt`.
3. **`Ответ по базе знаний`** — Basic LLM Chain (`chainLlm`, `promptType: define`) sends
   `{{ $json.prompt }}` to the attached chat model.
4. **`GPT-модель`** — an OpenAI chat model, `gpt-5-mini`, attached to the chain over the
   `ai_languageModel` connection. Configured with 3 retries, 5 s apart, and `continueErrorOutput`.
5. **`Ответ: ответ ассистента`** — Respond to Webhook returns HTTP 200 with
   `{ ok, question, answer }`, taking `question` from the Set node and `answer` from the chain's `text`
   field.

## Requirements

- n8n, self-hosted, recent 1.x, with the LangChain nodes (`@n8n/n8n-nodes-langchain`) available — they
  ship with current 1.x builds.
- **Credentials to create:** the exported JSON carries none. The `GPT-модель` node needs an **OpenAI**
  credential. Because the node speaks the OpenAI API, a proxy or any OpenAI-compatible gateway works —
  set its base URL in the credential instead of `api.openai.com`.
- **External services:** one LLM endpoint. Nothing else — no database, no vector store, no messenger.

## Setup

1. Import `workflow.json` (n8n → Workflows → ⋯ → **Import from File**).
2. Create the credential and assign it: open **`GPT-модель`** → *Credential* → *Create new* → OpenAI (or
   your OpenAI-compatible proxy). Then pick the model you actually have access to in the *Model* dropdown
   (`gpt-5-mini` is only this template's default).
3. Placeholders: **this template has no stripped secrets** — no chat id and no service token were removed
   from it, and the webhook path is already neutral (`/rag-ask`). What you *must* replace is the content,
   in the next two steps.
4. Set your own knowledge base in the Set node **`Собрать промпт с базой знаний`** — specifically in the
   `prompt` assignment. The sample block is a five-point FAQ for a fictional shop (delivery, payment,
   returns, warranty, `support@example.ru`). Replace it with your own text, keep the numbering, and keep
   the instruction lines above `БАЗА ЗНАНИЙ:` if you want the same guardrails. The customer question is
   appended after the block by the expression, so do not add it yourself.
5. Activate the workflow and test the endpoint:

```bash
curl -X POST https://YOUR-N8N-HOST/webhook/rag-ask \
  -H 'Content-Type: application/json' \
  -d '{"question":"Do you deliver outside Moscow?"}'
```

Expected response (shape taken from the Respond node):

```json
{
  "ok": true,
  "question": "Do you deliver outside Moscow?",
  "answer": "..."
}
```

Send `-d '{}'` to see the fallback question in action — the Set node substitutes `Как с вами связаться?`.

## Nodes

| Node | Type | Purpose |
|---|---|---|
| `Вопрос клиента (POST)` | `n8n-nodes-base.webhook` v2.1 | Accepts the JSON `POST` on `/rag-ask` and defers the reply to the Respond node. |
| `Собрать промпт с базой знаний` | `n8n-nodes-base.set` v3.5 | Reads `body.question` and builds the `question` and `prompt` fields, inlining the knowledge base. |
| `Ответ по базе знаний` | `@n8n/n8n-nodes-langchain.chainLlm` v1.9 | Sends the assembled prompt to the attached chat model and returns its `text`. |
| `GPT-модель` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` v1.3 | The chat model (`gpt-5-mini`) wired into the chain as `ai_languageModel`. |
| `Ответ: ответ ассистента` | `n8n-nodes-base.respondToWebhook` v1.5 | Replies to the webhook caller with `{ ok, question, answer }` and HTTP 200. |

## Notes

- **No retrieval step.** The whole knowledge base is pasted into every prompt. That is cheap to build but
  it costs tokens on every call and it has a hard ceiling: once the block grows past a few pages, move the
  text into a vector store + retriever and keep the same chain node.
- **Keep `Content-Type: application/json`.** The Set node reads `$json.body.question`; it is only
  guaranteed to exist for a JSON body. A POST without a JSON object can make that expression throw.
- **The webhook sets no `authentication`.** The node's parameters carry `httpMethod`, `path`,
  `responseMode` and empty `options` — no auth was configured, so the endpoint is public as exported.
  Every request spends an LLM call, so put something in front of it (n8n's own webhook auth, or a header
  check) before you leave it running.
- **`responseMode: responseNode` means the last node is mandatory.** Delete or rename
  `Ответ: ответ ассистента` and the caller's HTTP request never gets an answer.
- **The model node has a retry policy but no error branch.** `GPT-модель` is set to
  `retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 5000` and `onError: continueErrorOutput`. The
  error output, however, is not connected to anything in this export, so after the third failed attempt
  the chain has no model and nothing answers the webhook. Wire that output to an error Respond node (or
  attach a workflow error handler) if the endpoint is customer-facing.
- **The response body references nodes by name.**
  `$("Собрать промпт с базой знаний").item.json.question` and `$json.text` mean: rename the Set node or
  swap the chain node and the response breaks with a "node not found" style error.
- **`availableInMCP: true`** is set in the workflow settings — the workflow is exposed to n8n's MCP
  server as a callable tool. Turn it off if you do not want that.

## Кратко по-русски

Шаблон «ИИ-ассистент по базе знаний»: вебхук `POST /rag-ask` принимает `{"question": "..."}`, узел Set
подставляет вопрос в текст базы знаний (доставка, оплата, возврат, гарантия, контакты), узел Basic LLM
Chain отправляет промпт в модель `gpt-5-mini`, а Respond-узел возвращает `{ ok, question, answer }`.
База знаний лежит прямо в узле `Собрать промпт с базой знаний` — правьте её там, векторного хранилища
нет. Учётные данные из файла вырезаны: создайте своё подключение OpenAI (или совместимого прокси) и
назначьте его узлу `GPT-модель`. Вебхук без аутентификации — закройте его, если оставляете в бою.

---

Made by [Bowhard](https://bowhard.ru) — automation studio. Template is provided as-is, MIT licensed.
