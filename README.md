# n8n Workflow Templates by Bowhard

[![n8n](https://img.shields.io/badge/n8n-1.x-EA4B71)](https://n8n.io) [![License: MIT](https://img.shields.io/badge/License-MIT-green)](LICENSE)

Production-tested [n8n](https://n8n.io) workflows for lead intake, AI assistants, Yandex services and
business reporting. Every template here runs (or ran) in a real client project at
**[Bowhard](https://bowhard.ru)** — an automation studio that builds lead intake, voice assistants and
reporting on n8n, Voximplant and the Yandex ecosystem.

> Шаблоны n8n, проверенные в работе: приём заявок, ИИ-ассистенты, Яндекс (Диск/Календарь/Контакты) и
> отчётность. Русское описание — [в конце файла](#по-русски).

## Templates

| Template | Category | Nodes | Trigger | What it does |
|---|---|---|---|---|
| [ai-lead-intake](templates/ai-lead-intake) | AI · Lead intake | 12 | Webhook | One intake endpoint for website, Telegram, MAX, VK and e-mail: normalizes the payload, reads attachments with an LLM, classifies the request (category, priority, whether a human is needed), builds a CRM card and pings the manager. |
| [site-lead-to-sheets-telegram](templates/site-lead-to-sheets-telegram) | Lead intake | 6 | Webhook | Classic "website → storage → messenger": accepts a form POST, normalizes fields, appends a row to Google Sheets, notifies the manager in Telegram and answers the site with JSON. |
| [website-audit-form](templates/website-audit-form) | Lead intake | 3 | Webhook | Public "audit my site" form: validates the submitted address, calls the audit backend, returns the report to the page and stores the request. |
| [rag-support-assistant](templates/rag-support-assistant) | AI assistant | 5 | Webhook | Answers a customer question from a knowledge base with an LLM: builds the prompt from the KB block, calls an OpenAI-compatible endpoint and returns JSON for the chat widget. |
| [voice-assistant-knowledge-base](templates/voice-assistant-knowledge-base) | AI · Voice | 7 | Webhook | The "brain" of a phone assistant: takes a recognized phrase from VoxEngine, scores it against a topic table, answers from the knowledge base and asks for a human when it keeps failing. |
| [daily-deals-report](templates/daily-deals-report) | Reporting | 4 | Schedule | Every morning pulls deals from a CRM API, computes the summary and sends a formatted report to Telegram — scheduled reporting without a BI tool. |
| [yandex-calendar-create-event](templates/yandex-calendar-create-event) | Yandex · CalDAV | 4 | Webhook | Creates an event in Yandex Calendar over CalDAV: builds an iCal file and PUTs it into the calendar. No official Yandex node needed. |
| [yandex-calendar-list-events](templates/yandex-calendar-list-events) | Yandex · CalDAV | 4 | Webhook | Reads events from Yandex Calendar over CalDAV: builds a calendar-query for a time window, sends REPORT and parses VEVENTs into a clean list. |
| [yandex-disk-upload-file](templates/yandex-disk-upload-file) | Yandex · WebDAV | 4 | Webhook | Uploads a file (base64 in, WebDAV out) to Yandex Disk — handy as a storage step for reports, call recordings and generated media. |
| [yandex-contacts-actions](templates/yandex-contacts-actions) | Yandex · CardDAV | 5 | Webhook | Reads a Yandex address book over CardDAV: PROPFIND, multiget REPORT and vCard parsing into name/e-mail/phone. |
| [ai-video-generation-proxyapi](templates/ai-video-generation-proxyapi) | AI · Media | 13 | Webhook | End-to-end AI video: creates a generation task, polls its status, downloads the result, uploads it to Yandex Disk and reports to Telegram. |

**Highlights**

- **Yandex without native nodes.** n8n has no Yandex nodes — these templates talk CalDAV, CardDAV and
  WebDAV through plain HTTP Request nodes, with HTTP Basic Auth credentials.
- **Webhook tools with a shared secret.** The internal tools check an `x-watchdog-token` header in an
  `onlyRunIf` condition before doing any work, so a public endpoint cannot burn your API budget.
- **AI where it pays off.** LLM calls are used for classification and knowledge-base answers, and plain
  expressions/Code nodes for the rest.

## Bowhard services for n8n workflows

Two services the studio runs next to these templates, both useful as a step in an n8n flow:

- **[Audio and video to text](https://bowhard.ru/rasshifrovka)** — upload a recording (or a public link, up to
  2 GB) and get plain text, `.srt` subtitles and word-level `.json` timecodes. 15 minutes free every day,
  then 25 ₽ per hour of audio; files are deleted after 24 hours. Handy for call recordings and voice messages
  before they reach an LLM node.
- **[Text to speech](https://bowhard.ru/ozvuchka)** — Russian voice-over for notifications and reports:
  5 000 characters free, 3.9 ₽ per 1 000 characters.

## How to use a template

1. Open the template folder and download `workflow.json` (or copy the raw file).
2. In n8n: **Workflows → ⋯ → Import from File**. Alternatively copy the JSON and paste it onto the canvas.
3. Create the credentials listed in that template's README and assign them to the nodes
   (the exported JSON has **no** credentials attached — on purpose).
4. Replace the placeholders (see below), set your own URLs and ids.
5. Activate the workflow and test the trigger — every README has a `curl` example.

### Placeholders

| Placeholder | Meaning |
|---|---|
| `REPLACE_WITH_YOUR_CHAT_ID` | Telegram chat id that receives the notifications |
| `REPLACE_WITH_YOUR_TOKEN` | shared secret that protects the webhook (`x-watchdog-token` / `x-brain-token`) |
| `YOUR-N8N-HOST` | your n8n host, e.g. `n8n.example.com` |
| `example.com` | stands for a customer-specific address (CRM endpoint, audited site, LLM base URL) |

## Security notes

These files are cleaned before publishing, and you should keep them clean:

- credential blocks are stripped from every node — assign your own;
- live webhook paths are renamed to neutral ones, so an imported copy never calls our production URLs;
- long service tokens and real chat ids are replaced with placeholders;
- in production, always put authentication in front of a webhook that spends money (an LLM call, an SMS,
  a video generation task). The `onlyRunIf` header check used here is the minimum.

## Requirements

- n8n 1.x, self-hosted (templates were exported from a self-hosted instance in October 2026).
- Credentials for the services a template uses (Telegram bot, Google Sheets, HTTP Basic/Header Auth…).
- An OpenAI-compatible LLM endpoint for the AI templates — a proxy such as ProxyAPI works fine from
  regions where direct API access is blocked.

## Contributing

Found a bug or made a template better? Issues and pull requests are welcome. Please keep the format:
one folder per template with `workflow.json` plus `README.md`, no credentials in the JSON.

## License

[MIT](LICENSE). Templates are provided as-is, without warranty — read them before pointing them at
production data.

---

## По-русски

Здесь лежат шаблоны n8n, снятые с боевых проектов студии **[Bowhard](https://bowhard.ru)**: приём
заявок (сайт, Telegram, MAX, ВК, почта) с ИИ-разбором, публичная форма аудита сайта, ИИ-ассистент по
базе знаний, «мозг» голосового ассистента для телефонии, ежедневный отчёт по сделкам в Telegram и
работа с Яндексом — Календарь (CalDAV), Контакты (CardDAV), Диск (WebDAV) — обычными HTTP-узлами,
потому что официальных узлов Яндекса в n8n нет.

Как поставить: скачать `workflow.json` из папки шаблона → в n8n «Workflows → Import from File» →
создать и назначить учётные данные (в файлах их нет специально) → заменить заглушки
`REPLACE_WITH_YOUR_CHAT_ID` и `REPLACE_WITH_YOUR_TOKEN` → активировать и проверить вызовом из README.
Боевые адреса вебхуков и служебные токены из шаблонов вырезаны.

Лицензия MIT, без гарантий. Вопросы и доработки — через issues и pull request.
