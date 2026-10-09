# Website Audit Form

> Public audit endpoint: takes a form submission with a site URL, forwards it to your audit service together with the contact data, and returns the service's JSON verdict straight back to the page.

**Category:** Lead intake · **Nodes:** 3 · **Trigger:** Webhook `POST /site-audit` · **Tested on:** n8n 1.x self-hosted (October 2026)

## What it does

A deliberately thin proxy in front of your own audit service. The page posts a URL plus contact details, the workflow forwards exactly three form fields (and the client IP from the `x-forwarded-for` header) to the audit API, and the response body is passed through unchanged to the browser. It contains no business logic, no storage and no credentials — the audit work happens in the external service.

1. `Запрос аудита (POST)` receives `POST /site-audit` (response mode: Respond to Webhook node).
2. `Проверить сайт (CRM)` sends a JSON POST to the audit API with `url`, `contact`, `name` and `client_ip`.
3. `Ответ странице` returns the audit API's JSON body as-is with HTTP 200.

## Requirements

- n8n self-hosted, recent 1.x (node versions in the export: webhook v2.1, httpRequest v4.5, respondToWebhook v1.5).
- Credentials to create: **none** — no credentials were removed from this template, and none of the three nodes uses authentication.
- External services:
  - your own audit API endpoint that accepts a JSON POST with `url`, `contact`, `name`, `client_ip` and answers with JSON. The export points at the placeholder `https://YOUR-INTERNAL-HOST/api/audit`, which must be replaced (see Setup).

## Setup

1. Import `workflow.json` (n8n → Workflows → Import from File).
2. There are no credentials to assign.
3. Replace the placeholders — this export contains **no** `REPLACE_WITH_YOUR_CHAT_ID` and **no** `REPLACE_WITH_YOUR_TOKEN` values. The one thing that must be changed is the URL of the `Проверить сайт (CRM)` node: `https://YOUR-INTERNAL-HOST/api/audit` stands for the private address of the original deployment (the Docker host gateway) and will not resolve anywhere else — point it at your own audit endpoint.
4. Activate the workflow and call the webhook:

```bash
curl -X POST https://YOUR-N8N-HOST/webhook/site-audit \
  -H 'Content-Type: application/json' \
  -H 'x-forwarded-for: 203.0.113.10' \
  -d '{"url":"https://example.com","contact":"ivan@example.com","name":"Ivan Petrov"}'
```

Only `url`, `contact` and `name` are read from the body; everything else in the payload is ignored, and `client_ip` is taken from the `x-forwarded-for` header rather than from the body.

## Nodes

| Node | Type | Purpose |
|---|---|---|
| `Запрос аудита (POST)` | `n8n-nodes-base.webhook` v2.1 | Entry point: accepts `POST /site-audit` and answers through a Respond to Webhook node. |
| `Проверить сайт (CRM)` | `n8n-nodes-base.httpRequest` v4.5 | Forwards `url`, `contact`, `name` and `client_ip` as a JSON key-pair body to the audit API. |
| `Ответ странице` | `n8n-nodes-base.respondToWebhook` v1.5 | Returns the audit API's JSON body unchanged with HTTP 200. |

## Notes

- **Replace the hard-coded URL.** `https://YOUR-INTERNAL-HOST/api/audit` is a placeholder for the private-network address of the original deployment; as shipped it works only inside that network.
- **`neverError: true`.** The HTTP node ignores HTTP status codes from the audit service, so a 4xx/5xx from your API does not fail the workflow — the page still receives HTTP 200 with whatever body came back. There is no error branch and no separate failure response.
- **`neverError` does not cover timeouts.** The request timeout is 30 s; a slower audit service fails the node, and since the run stops before `Ответ странице`, the page gets n8n's default error response instead of JSON.
- **The response is a pure pass-through:** `respondWith: json` with the body `{{ $json }}`. Whatever contract your audit API has — including its error shape — is the contract the page sees. The node asks for a JSON response, so return JSON from the audit service.
- **`client_ip` is proxy-dependent.** It is read as `{{ $json.headers['x-forwarded-for'] || '' }}`, so it is an empty string when the webhook is called directly without a reverse proxy in front.
- **No authentication anywhere.** The webhook path is open and the HTTP node sends no credentials: anyone who knows the path can trigger audits against your service, and the audit service sees only the payload, not an n8n identity. Add a header check or a proxy rule for a public deployment.

## Кратко по-русски

- Что делает: вебхук `POST /site-audit` принимает заявку с публичной формы, передаёт её в ваш сервис аудита и возвращает ответ сервиса странице без изменений.
- Узлов всего три: приём запроса, POST в сервис аудита, ответ странице (тот же JSON, HTTP 200).
- Передаются только `url`, `contact`, `name`; `client_ip` берётся из заголовка `x-forwarded-for` и будет пустым при прямом вызове без прокси.
- Обязательно заменить адрес `https://YOUR-INTERNAL-HOST/api/audit` — так обозначен внутренний адрес исходной установки, снаружи он не работает. Учётных данных и заглушек `REPLACE_WITH_*` в шаблоне нет.
- В узле HTTP стоит `neverError: true`, поэтому ошибки 4xx/5xx сервиса не ломают процесс: страница всё равно получит 200 с телом ответа, отдельной ветки ошибок нет.
- Таймаут 30 секунд остаётся таймаутом: если сервис медленнее, страница получит стандартную ошибку n8n, а не JSON. Аутентификации нигде нет.

Made by [Bowhard](https://bowhard.ru) — automation studio. Template is provided as-is, MIT licensed.
