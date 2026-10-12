# AI FAQ Chatbot for Immigration Services (n8n + OpenAI + Google Sheets)

An automated FAQ chatbot that answers public questions about Indonesian immigration services (passports, visas, stay permits, APEC card, Work & Holiday Visa support letter, foreigner reporting, and office service hours).

The bot is built as an **n8n workflow** exposed through a webhook, so it can be connected to any front end (website widget, messaging bot, etc.).

> This repository contains a **sanitized export** of the workflow: credentials, spreadsheet IDs, webhook path, and contact numbers have been removed. The FAQ spreadsheet itself is not included.

---

## How it works

```mermaid
flowchart LR
    A[Client] -->|POST message| B[Webhook]
    B --> C[Clean Input]
    C --> D[Intent Classifier<br/>gpt-4o-mini]
    D --> E[Parse Intent]
    E --> F[(Google Sheets<br/>FAQ database)]
    F --> G[Format Data<br/>match by intent]
    G --> H[Response Agent<br/>gpt-4o-mini]
    H --> I[Format Log]
    I --> J[Respond to Webhook]
```

1. **Webhook** receives a `POST` request with the user's message.
2. **Clean Input** lowercases and trims the text.
3. **Intent Classifier** (LLM) maps the question to exactly one label from a fixed taxonomy of about 80 intents (e.g. `paspor_syarat`, `apec_biaya`, `lapor_apoa`, `mpp_blitar_jam`). Special labels handle greetings (`sapaan`) and anything out of scope (`tidak_diketahui`), such as live status checks, personal data, or app bugs.
4. **Google Sheets lookup**: the FAQ sheet is read and the row whose `intent` column matches the predicted label is selected.
5. **Response Agent** (LLM) rewrites the FAQ answer in a friendly customer-service tone, strictly based on the retrieved text. If nothing matches, it returns a polite fallback message that points the user to the official WhatsApp contact.
6. **Respond** returns `{ "reply": "..." }`.

### Design choices

- **Classify first, then retrieve**: the LLM never answers from its own knowledge. The final answer is grounded in a curated FAQ row, which reduces hallucination on policy-sensitive topics such as fees and requirements.
- **Closed intent list + explicit out-of-scope rules**: the classifier is told to return `tidak_diketahui` instead of guessing for off-topic questions, live data requests (queue/status lookups), and application errors.
- **Fallback path**: unmatched intents produce a fixed "information not available" reply with a human contact channel.
- **Non-technical maintenance**: staff can update answers by editing the Google Sheet, with no code changes.

## Tech stack

n8n, OpenAI API (`gpt-4o-mini`), Google Sheets API, Webhook/REST

## Setup

1. Import `faq_chatbot_workflow.sanitized.json` into n8n.
2. Create a Google Sheet with these columns: `intent`, `question`, `answer`, `url`. Each row's `intent` must match one label in the classifier prompt.
3. In the **Google Sheets (FAQ)** node, select your spreadsheet and add your Google credential.
4. In both **OpenAI Chat Model** nodes, add your OpenAI credential.
5. In the **AI Agent** node, replace `https://wa.me/62XXXXXXXXXX` with your own contact link.
6. Activate the workflow and send a test request:

```bash
curl -X POST https://<your-n8n-host>/webhook/faq-chatbot \
  -H "Content-Type: application/json" \
  -d '{"message": "apa saja syarat membuat paspor?"}'
```

Expected response shape:

```json
{ "reply": "..." }
```

## Intent taxonomy (overview)

| Area | Examples |
|------|----------|
| Passport | requirements, process, fees, processing time, lost/damaged passport, children's passport, identity changes, affidavit for dual-citizenship children |
| Visa & stay permits | visa basics, overstay, ITK / ITAS / ITAP, status conversion |
| APEC Business Travel Card | definition, requirements, process, fees, validity |
| Work & Holiday Visa support letter (SDUWHV) | definition, criteria, requirements, process |
| Foreigner reporting | APOA, obligations, procedures |
| Service info | office hours, address, public service mall (MPP) hours and addresses |
| Special | `sapaan` (greeting), `tidak_diketahui` (out of scope / live data) |

## Limitations & roadmap

- Conversation logging is only formatted in this export; persisting it (e.g. appending to a sheet or database) is the next step for monitoring.
- The webhook has no authentication or rate limiting. Add a shared secret header and throttling before exposing it publicly.
- Intent accuracy has not been benchmarked on a labeled test set. Building one (and tracking accuracy and fallback rate) would make improvements measurable.
- The Google Sheets node reads all rows on every request; filtering by intent would reduce latency as the FAQ grows.

## Author

**Dhimas Rama Anthony Navy Pratama**  
[LinkedIn](https://www.linkedin.com/in/dhimaspratama1157) | dhimasrama38@gmail.com
