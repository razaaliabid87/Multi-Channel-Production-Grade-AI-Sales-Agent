# Multi-Channel AI Sales Agent

A Multi-channel AI sales agent (n8n, HubSpot, OpenAI, RAG, Qdrant, Gemini, Slack) that unifies Web Chat, Email, SMS & WhatsApp into one customer identity with full cross-channel conversation memory, every message and reply is logged to HubSpot, with anonymous web-chat visitors tracked via a custom Web Session ID property. Retrieves answers via RAG, books meetings, updates CRM records, replies in the customer's language, and gates every quote behind human approval, with signature-verified inbound webhooks.
<table>
  <tr>
    <td><img src="assets/screenshots/slack-channel-notification.png" width="400"></td>
    <td><img src="assets/screenshots/approval-form.png" width="400"></td>
  </tr>
</table>

slack-channel-notification.png
> Full technical documentation — data contracts, node-level behavior, testing status is in
[`PROJECT_DOCUMENTATION_FULL.md`](./PROJECT_DOCUMENTATION_FULL.md).

---

## Overview

Most "AI chatbots" answer questions. This is a sales agent: it remembers who it's talking to across every channel, retrieves real information instead of guessing, takes real CRM actions, and knows exactly where a human needs to step in before it commits the business to anything.

**Core capabilities:**
- **Cross-channel identity resolution:**  phone → email → web-session, in priority order — so a customer who starts on WhatsApp and follows up by email is recognized as the same person, with one shared conversation history. Conflicting identity data is flagged for human review rather than silently overwritten.
- **Retrieval-grounded answers:**  never states a price or policy detail it hasn't just retrieved from the actual knowledge base; deduplicates and merges chunks split across boundaries before the model ever sees them.
- **Tool-using agent:**  updates CRM fields, books meetings, and drafts quote requests as part of natural conversation, with a strict allowlist on which fields it's ever allowed to touch.
- **Two distinct human-in-the-loop boundaries:**  identity conflicts pause automated writes for review; every quote requires explicit human approval (via Slack + an n8n Form) before anything reaches the customer.
- **Signed webhook security:**  inbound Twilio (HMAC-SHA1) and WhatsApp (HMAC-SHA256) traffic is cryptographically verified before it's trusted; forged requests are rejected with a 403 before normalization even runs.

---

## Architecture:

```
Web Chat / Email / SMS / WhatsApp
              │
              ▼
   Workflow A — Ingestion & Identity
   (normalize each channel → resolve/create HubSpot contact)
              │
              ▼
   Workflow B — Agent Core
   (fetch history → OpenAI agent w/ tools → reply routed back to origin channel)
      │            │              │
      ▼            ▼              ▼
  Knowledge    Update HubSpot   Book Meeting /
  Retrieval    Property         Request Quote Draft
  (Qdrant)                            │
                                       ▼
                          Workflow C — Quote Approval & Send
                          (Slack notification → human review via
                           n8n Form → approved delivery)
```

**Seven n8n workflows:**

| # | Workflow | Responsibility |
|---|---|---|
| 1 | Workflow A — Ingestion & Identity | Normalizes all four channels, resolves or creates the HubSpot contact |
| 2 | Workflow B — Agent Core | Retrieves history, runs the AI agent, routes the reply |
| 3 | Workflow C — Quote Approval & Send | Human-gated quote review and delivery |
| 4 | Tool — Knowledge Retrieval (General Purpose) | Vector search, reusable outside this project |
| 5 | Tool — Update HubSpot Property | Guarded, allowlisted contact-field updates |
| 6 | Tool — Book Meeting | Meeting creation with date/time validation |
| 7 | Tool — Request Quote Draft | Flags a quote for human review; sends nothing itself |

The tool workflows are deliberately independent, callable sub-workflows — reusable across future agent projects without rework.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Automation platform | n8n |
| LLM / reasoning | OpenAI (LangChain Agent node, tool-calling) |
| CRM | HubSpot (contacts, deals, meetings, notes, custom properties) |
| Vector search | Qdrant |
| Embeddings | Google Gemini (`gemini-embedding-001`) |
| SMS | Twilio |
| WhatsApp | Meta WhatsApp Business Cloud API (direct integration) |
| Email | Gmail API |
| Human approval routing | Slack API |
| Approval interface | n8n Forms |
| Web chat | Custom webhook endpoint |

---

## How It Works

**1. Ingestion & Identity:**  every channel normalizes into one common message shape. SMS and WhatsApp webhooks are signature-verified before anything else runs. Identity resolves by phone, then email, then (web chat only) a stored session ID, before creating a new contact — never blocking creation on having every identifier.

**2. Agent Core:**  pulls the last 10 conversation notes for the resolved contact (HubSpot Notes *are* the memory store — no separate database), builds a prompt carrying real-time date/timezone context and the business's own name (pulled from a HubSpot config record, not hardcoded), and hands the conversation to an OpenAI agent with four tools. Two values — `hubspot_contact_id` and `needs_review` — are supplied to every write-capable tool from Workflow B's trusted context, never from the model's own tool-call output, so the model can decide *when* to act but never *whether it's authorized* to.

**3. Quote Approval & Send:**  a quote request never reaches the customer directly. It creates a flagged HubSpot Deal + note, notifies a human via Slack, and only routes delivery back through the customer's original channel once a human has explicitly approved a final price through the form.

---

## Setup

Requires your own credentials/instances for: HubSpot (private app token — contacts/deals/meetings/notes scopes), OpenAI, Twilio, a Meta Developer app with WhatsApp Business Cloud API access, Gmail OAuth, a Qdrant instance, a Google Gemini API key, and a Slack app (`chat:write` scope, bot invited to the target notification channel). Each workflow's credential fields must be pointed at your own instances before import.

---

## Roadmap

Deliberately scoped for a working v1, with a clear path forward: voice channel support and WhatsApp voice-note transcription, MCP exposure of the tool layer, active knowledge-base metadata filtering, semantic caching for high-volume queries, native HubSpot Quotes (once Products/template configuration exists), and phone-normalization support beyond the current single-market scope.

---

## Author

Built by Raza Ali as part of AI Automation Engineering | AI Agents.
