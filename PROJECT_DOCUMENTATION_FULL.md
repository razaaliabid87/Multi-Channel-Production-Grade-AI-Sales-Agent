# Multi-Channel AI Sales Agent

> A portfolio-scale, multi-channel AI sales automation system built with **n8n**, **HubSpot**, **OpenAI**, **Qdrant**, and **Gemini embeddings**.

The system receives customer conversations through **Web Chat, Email, SMS, and WhatsApp**, resolves the customer to a HubSpot contact, retrieves conversation history and knowledge, qualifies leads, books meetings, and handles quote requests through a human approval workflow.

> **Implementation note:** This README documents the system based on the reviewed workflow files, node code, expressions, and wiring. It describes what is actually implemented rather than relying on stale in-canvas documentation.

---

## Table of Contents

- [Overview](#overview)
- [Core Capabilities](#core-capabilities)
- [Technology Stack](#technology-stack)
- [Architecture](#architecture)
- [Workflow Inventory](#workflow-inventory)
- [Workflow A — Ingestion & Identity](#workflow-a--ingestion--identity)
- [Workflow B — Agent Core](#workflow-b--agent-core)
- [Knowledge Retrieval Tool](#knowledge-retrieval-tool)
- [Update HubSpot Property Tool](#update-hubspot-property-tool)
- [Book Meeting Tool](#book-meeting-tool)
- [Request Quote Draft Tool](#request-quote-draft-tool)
- [Workflow C — Quote Approval & Send](#workflow-c--quote-approval--send)
- [Human-in-the-Loop Controls](#human-in-the-loop-controls)
- [Security & Safety Boundaries](#security--safety-boundaries)
- [Data Contracts](#data-contracts)
- [Conversation History](#conversation-history)
- [Known Issues & Open Verification](#known-issues--open-verification)
- [Deliberately Deferred](#deliberately-deferred)
- [Testing Status](#testing-status)
- [Current Scope & Limitations](#current-scope--limitations)

---

## Overview

The Multi-Channel AI Sales Agent is designed to behave as a sales representative across four asynchronous customer channels.

Its core responsibilities are:

1. Receive an inbound customer message.
2. Validate channel-specific webhook security where applicable.
3. Normalize the incoming payload into a common message structure.
4. Resolve the customer against HubSpot.
5. Detect identity conflicts instead of silently overwriting customer data.
6. Retrieve recent conversation history.
7. Provide the AI agent with business configuration and current date/time.
8. Allow the agent to retrieve verified knowledge instead of relying on memorized pricing or policy information.
9. Allow controlled CRM updates and meeting booking.
10. Convert quote requests into a human-review workflow.
11. Deliver the response through the originating channel where supported.
12. Record successful and failed delivery attempts in HubSpot.

Pricing is deliberately **human-gated**. The AI can request a quote draft, but the customer-facing pricing commitment is not sent until a human approves it through Workflow C.

This is a **portfolio-scale implementation** rather than a claim of complete enterprise infrastructure. Several capabilities are intentionally simplified or deferred.

---

## Core Capabilities

| Capability | Implementation |
|---|---|
| Customer channels | Web Chat, Email, SMS, WhatsApp |
| Workflow platform | n8n |
| CRM | HubSpot |
| AI agent | OpenAI / LangChain Agent node |
| Configured model | `gpt-4o-mini` |
| Knowledge embeddings | Gemini `gemini-embedding-001` |
| Vector database | Qdrant |
| Conversation memory | HubSpot Notes |
| Identity resolution | Phone → Email → Web Session → New Contact |
| CRM write protection | `needs_review` guard + property allowlist |
| Meeting booking | HubSpot Meetings |
| Quote handling | HubSpot Deal + human approval workflow |
| Human notification | Slack |
| Quote approval interface | n8n Form |
| SMS authentication | Twilio HMAC-SHA1 |
| WhatsApp authentication | Meta HMAC-SHA256 |
| Delivery logging | HubSpot Notes |

---

## Technology Stack

- **n8n** — workflow orchestration
- **HubSpot** — CRM, contact records, conversation notes, meetings, and deals
- **OpenAI** — AI sales agent
- **Qdrant** — vector search
- **Gemini Embeddings** — `gemini-embedding-001`
- **Gmail** — email intake and reply delivery
- **Twilio** — SMS intake and delivery
- **Meta Cloud API** — WhatsApp messaging
- **Slack** — human quote-review notification
- **n8n Forms** — quote approval interface

---

# Architecture

## High-Level Flow

```text
┌─────────────┬─────────┬─────────────┬──────────────┐
│  Web Chat   │  Email  │     SMS     │   WhatsApp   │
│  Webhook    │  Gmail  │  Twilio     │  Meta Cloud  │
│             │ Polling │  Webhook    │  API Webhook │
└──────┬──────┴────┬────┴──────┬──────┴───────┬──────┘
       │           │           │              │
       └───────────┴───────────┴──────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │ Workflow A              │
              │ Ingestion & Identity    │
              │                         │
              │ Normalize → Resolve     │
              │ HubSpot Contact         │
              └────────────┬────────────┘
                           │
                           │ Execute Workflow
                           ▼
              ┌─────────────────────────┐
              │ Workflow B              │
              │ Agent Core              │
              │                         │
              │ History → AI Agent      │
              │ → Tools → Reply         │
              └──────┬──────┬──────┬─────┘
                     │      │      │
          ┌──────────┘      │      └─────────────┐
          ▼                 ▼                    ▼
 ┌────────────────┐ ┌────────────────┐ ┌──────────────────┐
 │ Knowledge      │ │ Update HubSpot │ │ Book Meeting /   │
 │ Retrieval      │ │ Property       │ │ Request Quote    │
 │ Qdrant         │ │                │ │ Draft            │
 └────────────────┘ └────────────────┘ └────────┬─────────┘
                                                 │
                                          Slack notification
                                                 ▼
                                    ┌────────────────────────┐
                                    │ Workflow C             │
                                    │ Quote Approval & Send  │
                                    │ Human-gated            │
                                    └────────────────────────┘
```

## Workflow Architecture

The system contains **seven n8n workflows**:

| # | Workflow | Responsibility |
|---:|---|---|
| 1 | **Workflow A — Ingestion & Identity** | Normalizes all four channels and resolves or creates the HubSpot contact |
| 2 | **Workflow B — Agent Core** | Retrieves history, runs the AI agent, executes tools, and routes the reply |
| 3 | **Workflow C — Quote Approval and Send** | Human approval and customer-facing quote delivery |
| 4 | **Tool — Knowledge Retrieval Tool (General Purpose)** | Vector search against the knowledge base |
| 5 | **Tool — Update HubSpot Property** | Low-risk contact-field updates |
| 6 | **Tool — Book Meeting** | Creates a HubSpot meeting |
| 7 | **Tool — Request Quote Draft** | Records a quote request and starts human review |

The tool workflows are intentionally separated from the main agent workflow so they can be independently called and reused.

---

# Workflow A — Ingestion & Identity

Workflow A is the common intake and identity layer for all four customer channels.

## Channel Triggers

### Web Chat

- Webhook: `webchat-inbound`
- `responseMode: responseNode`
- The response is expected to return through the same synchronous webhook connection.

### Email

- Gmail polling trigger
- Checks every minute for unread mail.

### SMS

- Twilio webhook: `twilio-sms-inbound`
- Raw request body capture is enabled for signature verification.

### WhatsApp

- Webhook supports multiple HTTP methods.
- `GET` handles Meta's verification handshake.
- `POST` handles incoming messages.
- Raw request body capture is enabled for signature verification.

---

## Webhook Security

Security validation occurs before normal processing for SMS and WhatsApp.

### Twilio Signature Validation

The workflow:

1. Reconstructs the expected Twilio signature from the webhook URL and sorted POST parameters.
2. Computes the signature using HMAC-SHA1.
3. Compares it with `X-Twilio-Signature`.
4. Sets the signature validation result.
5. Rejects invalid requests with HTTP `403`.

### WhatsApp Signature Validation

The workflow:

1. Reads the raw request body.
2. Uses the Meta App Secret.
3. Computes HMAC-SHA256.
4. Compares the result with `X-Hub-Signature-256`.
5. Rejects invalid requests before normal processing.

Malformed or missing signatures are treated as invalid traffic rather than being allowed to continue.

---

## Message Normalization

Each channel has a dedicated normalization step:

- `Normalize - Web Chat`
- `Normalize - Email`
- `Normalize - SMS`
- `Normalize - WhatsApp`

All four produce the same common structure:

```json
{
  "channel": "...",
  "external_id": "...",
  "contact_name": "...",
  "contact_email": "...",
  "contact_phone": "...",
  "message_text": "...",
  "thread_id": "...",
  "timestamp": "..."
}
```

This allows the downstream agent workflow to operate independently of the original channel payload.

### WhatsApp Payload Validation

The WhatsApp normalizer does not silently accept unknown payload structures. An unrecognized payload, such as a delivery-status callback instead of a customer message, causes the workflow to fail through its downstream guard rather than forwarding empty or invalid message data.

---

## Phone Normalization

Phone normalization is intentionally market-specific.

The current implementation:

1. Removes non-digit characters.
2. Keeps the last 10 digits.
3. Prefixes `+92`.

This supports common Pakistani representations such as:

```text
03001234567
+923001234567
```

being normalized into a comparable format.

This is currently appropriate for the target customer base but would need to change for international expansion.

---

## Identity Resolution

The identity resolution priority is:

```text
Phone
  ↓
Email
  ↓
Web Session ID
  ↓
New HubSpot Contact
```

### 1. Phone Match

If a phone number is available or can be extracted from the message, HubSpot is searched by phone.

### 2. Email Match

If no phone match exists:

- Non-email channels attempt to extract an email from the message.
- Email messages already have the sender's email address.

HubSpot is then searched by email.

### 3. Web Session Match

For Web Chat only, if neither phone nor email identifies the customer, the workflow searches by the stored `web_session_id`.

This supports returning anonymous visitors within the same browser session.

### 4. New Contact

If no identifier matches an existing contact, a new HubSpot contact is created using whichever information is available:

- Phone
- Email
- Name
- Web session ID

The workflow does not require all identifiers to be present.

---

## Identity Conflict Handling

When a contact is successfully resolved:

- A missing email can be added when the current message supplies one.
- A conflicting email is **not automatically overwritten**.
- A conflicting phone number is **not automatically overwritten**.
- A HubSpot note records the conflict.
- `needs_review` becomes `true`.

This creates a human-review boundary for uncertain identity.

A Web Chat contact's stored session ID is refreshed when the customer has already been identified through phone or email but returns with a new browser session.

---

## Workflow A → Workflow B Contract

```json
{
  "channel": "...",
  "hubspot_contact_id": "...",
  "message_text": "...",
  "thread_id": "...",
  "resolution_method": "phone_match | email_match | session_match | new_contact",
  "needs_review": false,
  "phone_number_id": null
}
```

### `resolution_method`

Allowed values:

- `phone_match`
- `email_match`
- `session_match`
- `new_contact`

### `needs_review`

`true` only when a genuine phone/email identity conflict has been detected.

### `phone_number_id`

- Populated for WhatsApp.
- `null` for other channels.
- Represents the Meta identifier of the registered business number that received the WhatsApp message.

---

## Conversation Logging

Every inbound customer message is logged as a HubSpot Note associated with the resolved contact.

There is **no separate conversation-memory store**. HubSpot Notes are the source used by Workflow B for recent conversation history.

---

# Workflow B — Agent Core

Workflow B is the central AI reasoning and orchestration layer.

## Trigger

Workflow B is invoked by Workflow A using an n8n **Execute Workflow** connection.

It receives the Workflow A handoff contract.

---

## Conversation History

Workflow B:

1. Retrieves the contact's HubSpot note associations.
2. Limits the history to the most recent 10 notes.
3. Batch-reads the note content.
4. Formats the notes oldest-first.
5. Provides the formatted history to the AI agent.

For a new contact with no previous notes, the workflow uses:

```text
(no prior conversation on record — this is a new contact)
```

---

## Business Configuration

The workflow retrieves a `business_name` property from a dedicated HubSpot Company record used as a configuration store.

The business name is therefore not hardcoded into the workflow's prompt.

Changing the business name in the configuration record does not require changing the n8n workflow itself.

---

## Runtime Context

A Code node combines:

- Workflow A handoff data
- Conversation history
- Business name
- Current date/time

The current time is exposed as:

```text
current_datetime
```

using ISO 8601 UTC.

This allows the agent to resolve relative dates such as:

- tomorrow
- next Monday
- next week

against the actual current time rather than guessing.

The agent is instructed to interpret relative times using Pakistan Standard Time (UTC+5) unless the customer provides another timezone.

---

## AI Agent

The workflow uses an OpenAI-backed LangChain AI Agent node.

### Configured Model

```text
gpt-4o-mini
```

### Agent Input

The customer message is supplied as the agent's input.

### System Prompt Responsibilities

The system prompt establishes:

- Sales-agent role and scope
- Conversation context
- Business configuration
- Current date/time
- Knowledge-retrieval requirements
- Tool usage rules
- Identity-review behavior
- Delivery-confirmation rules
- Quote approval restrictions
- Language matching

It also instructs the agent to treat the customer's message as **data rather than instructions**, providing an early prompt-injection defense.

---

## Important Agent Rules

The documented agent behavior includes principles such as:

- Never claim an action succeeded before the action actually succeeds.
- Never claim that a quote has been sent when it has only been drafted for approval.
- Respect `needs_review` when identity conflicts exist.
- Do not fabricate a promise to follow up later because there is no autonomous follow-up mechanism.
- Do not answer pricing or policy questions from memory; retrieve the relevant knowledge.
- Match the customer's language.
- Resolve relative dates against the supplied current time.

---

## Agent Tools

The agent has access to four workflow-based tools:

1. Knowledge Retrieval
2. Update HubSpot Property
3. Book Meeting
4. Request Quote Draft

A key security boundary is used for write-capable tools.

The following values are supplied from Workflow B's trusted context:

```text
hubspot_contact_id
needs_review
```

They are **not taken from the model's tool-call arguments**.

This means the model can decide whether to call a tool, but it does not control the identity-safety values that determine whether the tool is permitted to act.

---

## Reply Routing

The response is routed using the channel from the resolved conversation.

| Channel | Delivery |
|---|---|
| Email | Gmail reply |
| SMS | Twilio |
| WhatsApp | Meta Graph API |
| Web Chat | Returned from Workflow B to Workflow A |

The channel switch intentionally has no silent fallback. An unknown channel value fails loudly instead of silently dropping the response.

---

## Delivery Logging

Every delivery attempt is normalized into a common success/failure result.

### Successful delivery

Logged to HubSpot as:

```text
[agent] ...
```

### Failed delivery

Logged to HubSpot as:

```text
[agent - DELIVERY FAILED] ...
```

A delivery note is written only after the send operation has been confirmed as successful or failed. The workflow does not record a message as successfully delivered before the actual send result is known.

---

# Knowledge Retrieval Tool

The Knowledge Retrieval Tool is intentionally designed as a **general-purpose retrieval workflow** rather than a sales-specific component.

Its contract does not depend on CRM or channel-specific fields.

## Contract

### Input

```json
{
  "query": "...",
  "top_k": 5,
  "filters": {}
}
```

### Output

```json
{
  "found": true,
  "chunks": [],
  "chunk_count": 0,
  "best_score": 0,
  "status": "SUCCESS"
}
```

---

## Retrieval Pipeline

```text
Input Validation
      ↓
Gemini Embedding
      ↓
Qdrant Vector Search
      ↓
Similarity Filtering
      ↓
Metadata Filtering
      ↓
Deduplication
      ↓
Adjacent Chunk Merging
      ↓
Text Compression
      ↓
Structured Result
```

### Input Validation

- Empty queries are rejected.
- `top_k` is clamped to the range `1–20`.

### Embeddings

The workflow uses:

```text
gemini-embedding-001
```

### Vector Search

Qdrant performs the vector search.

### Similarity Threshold

A similarity threshold of:

```text
0.50
```

is applied.

Chunks below the threshold are not returned.

### Metadata Filters

The tool supports generic metadata filtering, for example:

```json
{
  "department": "sales"
}
```

The mechanism exists but is currently unused by Workflow B.

### Deduplication

Chunks sharing the same `chunk_uuid` are deduplicated.

### Adjacent Chunk Merging

Consecutive chunks from the same source document are merged when appropriate.

This prevents information split across chunk boundaries from being returned as disconnected fragments.

### Text Compression

Merged content is compressed by removing duplicated section titles and excessive whitespace.

---

## Retrieval Status Values

The tool uses the following status values:

| Status | Meaning |
|---|---|
| `SUCCESS` | Retrieval completed successfully |
| `NO_MATCH` | No sufficiently similar content found |
| `EMPTY_COLLECTION` | Qdrant collection contains no usable data |
| `EMBEDDING_ERROR` | Embedding generation failed |
| `SEARCH_ERROR` | Vector search failed |
| `INVALID_INPUT` | Input validation failed |

Retrieval failures return gracefully with:

```json
{
  "found": false
}
```

rather than throwing an unhandled exception that would terminate the agent turn.

---

# Update HubSpot Property Tool

This tool allows the AI agent to record qualification information as it is discovered during conversation.

Supported information includes:

- Name
- Email
- Phone
- Job title
- Company
- Lead status

## Guard 1 — Identity Review

If:

```text
needs_review = true
```

the update is refused.

The tool returns an explicit reason and instructs the agent not to claim that the update succeeded.

## Guard 2 — Property Allowlist

The model can only update these properties:

```text
email
phone
firstname
lastname
hs_lead_status
jobtitle
company
```

Any other property is rejected.

This prevents the model from writing arbitrary HubSpot fields.

## Risk Classification

This tool is classified as a low-stakes and reversible operation.

Unlike pricing, it does not require a separate human approval step, provided the identity and property guards pass.

---

# Book Meeting Tool

The Book Meeting Tool creates a HubSpot meeting associated with the resolved contact.

## Validation

The workflow first checks:

1. `needs_review`
2. Whether `meeting_datetime_iso` is a valid date.
3. Whether the requested meeting time is in the future.

Invalid or past times are rejected before the request reaches HubSpot.

## Meeting Payload

The meeting creation includes:

```text
hs_timestamp
hs_meeting_title
hs_meeting_start_time
hs_meeting_end_time
```

The default meeting duration is:

```text
30 minutes
```

because the current implementation does not allow the customer or agent to specify a duration.

## Association

The workflow currently uses:

```text
associationTypeId = 200
```

This has worked during testing, but the documentation identifies the association type as an open verification item for environments with custom HubSpot configurations.

---

# Request Quote Draft Tool

The Request Quote Draft Tool intentionally does **not** send a quote to the customer.

HubSpot's native Quotes object requires a configured Products library and Quote template. Those prerequisites are not currently available in the documented HubSpot portal.

Instead, the workflow implements a controlled quote-request process.

## Processing

```text
Agent requests quote
       ↓
Check needs_review
       ↓
Create HubSpot Deal
       ↓
Create quote-request note
       ↓
Notify human through Slack
       ↓
Human opens approval form
       ↓
Workflow C
```

## Actions

### 1. Identity Guard

If `needs_review` is true, the quote request is refused.

### 2. Create Deal

A HubSpot Deal is created using:

```text
Pipeline: default
Stage: presentationscheduled
```

The Deal is associated with the contact.

### 3. Create Quote Request Note

A note is created with the marker:

```text
[QUOTE REQUESTED - NEEDS HUMAN QUOTE CREATION]
```

The note contains the requested package and relevant conversation context.

### 4. Slack Notification

A configured Slack channel receives the quote request.

The notification includes a link to Workflow C's approval form with URL-encoded parameters including:

```text
deal_id
contact_id
package_name
channel
```

## Pricing Safety Boundary

This is the primary pricing human-in-the-loop boundary.

The Request Quote Draft Tool itself sends **nothing** to the customer.

---

# Workflow C — Quote Approval & Send

Workflow C handles the human approval and final quote delivery.

## Trigger

Workflow C uses an n8n Form at:

```text
quote-approval
```

A human opens the form through the link included in the Slack notification.

### Hidden Fields

The following values arrive through the form link:

```text
deal_id
contact_id
package_name
channel
```

### Visible Fields

The human reviewer provides:

- Decision
- Final Price
- Additional Notes to Customer

---

## Rejection Path

If the reviewer rejects the request:

1. A `[quote rejected]` note is logged on the contact.
2. Nothing is sent to the customer.

---

## Approval Path

When approved:

### 1. Fetch Contact

The workflow retrieves:

```text
email
firstname
phone
```

from HubSpot.

### 2. Preserve Originating Channel

Delivery is based on the channel where the original conversation started.

| Originating Channel | Quote Delivery |
|---|---|
| SMS | SMS to originating phone |
| WhatsApp | WhatsApp to originating phone |
| Email | Email |
| Web Chat | Email |

Email acts as the universal fallback for channels other than SMS and WhatsApp.

### 3. Validate Delivery Method

Before sending:

- SMS/WhatsApp require a phone number.
- Email requires an email address.

If the required field is missing, the workflow produces an explicit delivery failure instead of attempting to send with missing information.

### 4. Successful Delivery

On confirmed success:

```text
[quote sent]
```

is logged and the Deal is moved to:

```text
contractsent
```

### 5. Failed Delivery

A failed send is logged as:

```text
[quote send FAILED]
```

along with the reason.

This is intentionally visible to a human because an approved quote that was not actually delivered requires follow-up.

---

## WhatsApp 24-Hour Messaging Constraint

A known expected failure occurs when a free-form WhatsApp message is attempted more than 24 hours after the customer's last message.

The current implementation does not configure a pre-approved WhatsApp template for this situation.

Therefore, Meta can reject the message.

The workflow surfaces this as a logged delivery failure rather than silently losing the event.

---

# Human-in-the-Loop Controls

The system has two distinct human-review boundaries.

## 1. Identity Review

Triggered when a resolved HubSpot contact contains an email or phone value that conflicts with the current customer-provided identifier.

Result:

```text
needs_review = true
```

Write-capable tools respect this state.

## 2. Pricing Review

Triggered by a quote request.

The AI can:

- Recognize the quote request.
- Create a Deal.
- Record the requested package.
- Notify a human.

The AI cannot directly send the pricing commitment.

A human must:

1. Open the approval form.
2. Approve or reject.
3. Enter the final price.
4. Optionally add customer-facing notes.

Only then does Workflow C attempt customer delivery.

---

# Security & Safety Boundaries

The system uses several explicit safety controls.

| Boundary | Protection |
|---|---|
| Twilio webhook | HMAC-SHA1 signature validation |
| WhatsApp webhook | HMAC-SHA256 signature validation |
| Identity conflicts | `needs_review` state |
| CRM writes | Property allowlist |
| Tool identity | Trusted `hubspot_contact_id` from workflow context |
| Tool review state | Trusted `needs_review` from workflow context |
| Meeting booking | Future-date validation |
| Knowledge retrieval | Similarity threshold |
| Agent claims | Do not claim success before confirmed result |
| Pricing | Human approval required |
| Delivery | Success/failure confirmed before logging outcome |

The design intentionally separates **model decision-making** from **authorization-sensitive values**.

The model can decide that a tool should be called, while critical identity and review fields are supplied by the workflow itself.

---

# Data Contracts

## Normalized Inbound Message

```json
{
  "channel": "...",
  "external_id": "...",
  "contact_name": "...",
  "contact_email": "...",
  "contact_phone": "...",
  "message_text": "...",
  "thread_id": "...",
  "timestamp": "..."
}
```

## Workflow A → Workflow B

```json
{
  "channel": "...",
  "hubspot_contact_id": "...",
  "message_text": "...",
  "thread_id": "...",
  "resolution_method": "phone_match",
  "needs_review": false,
  "phone_number_id": null
}
```

## Knowledge Retrieval

### Input

```json
{
  "query": "...",
  "top_k": 5,
  "filters": {}
}
```

### Output

```json
{
  "found": true,
  "chunks": [],
  "chunk_count": 0,
  "best_score": 0,
  "status": "SUCCESS"
}
```

---

# Conversation History

The project deliberately avoids a separate memory database.

Conversation history is stored as HubSpot Notes associated with the customer contact.

The process is:

```text
Customer Message
      ↓
Resolve HubSpot Contact
      ↓
Create HubSpot Note
      ↓
Future Agent Turn
      ↓
Fetch Latest 10 Notes
      ↓
Batch Read Note Content
      ↓
Format Oldest → Newest
      ↓
Provide to AI Agent
```

This gives the agent recent conversational context while keeping customer and conversation information within the CRM.

---

# Deliberately Deferred

The following features are intentionally not part of the current implementation.

## Voice Calling

Live voice support is deferred because it introduces a different real-time architecture involving turn-taking, speech-to-text, and text-to-speech.

## WhatsApp Voice Notes

Voice-note transcription is deferred to be developed alongside the broader voice capability rather than as an isolated feature.

## MCP Exposure

Model Context Protocol exposure is not currently implemented.

The tool workflows are already separated into independently callable sub-workflows, which keeps a future MCP migration relatively isolated.

## Qdrant Metadata Filtering

The Knowledge Retrieval Tool already supports:

```text
filters
```

but Workflow B does not currently use it.

It can be activated when the knowledge base genuinely requires department/category filtering.

## Semantic Caching

Semantic caching is deferred as a performance/cost optimization.

The documented implementation does not currently identify retrieval latency as the active bottleneck.

## Composite Ranking & Diversity Scoring

Advanced composite ranking and diversity scoring were evaluated but deferred because the current knowledge base does not have the scale/diversity problem these techniques were intended to solve.

## Query-Complexity-Based Dynamic Budget Allocation

Also deferred because the current knowledge base does not justify the additional complexity.

## Anonymous Web-Chat Contact Cleanup

Anonymous visitors who never provide identifying information can create separate HubSpot contacts across sessions.

This is considered a low-harm data-cleanup issue rather than a correctness problem for identified customers.

## Native HubSpot Quotes

The current Deal + Note implementation can eventually be replaced with native HubSpot Quotes after the required:

- Products library
- Quote template

are configured in HubSpot.

The upgrade is expected to remain isolated to the quote-request portion of the system.

---

# Testing Status

## Workflow A

The following have been tested end-to-end across all four channels:

- Phone identity matching
- Email identity matching
- Web session matching
- New contact creation
- Identity completion
- Identity conflict handling
- Twilio signature rejection
- WhatsApp signature rejection

Deliberately forged webhook requests were used to confirm invalid signatures are rejected.

## Workflow C

Individual components have been verified in isolation:

- Slack notification delivery
- Form field population
- Channel-specific delivery branches

Workflow C has been reactivated after the earlier inactive-workflow issue was fixed.

A complete end-to-end quote flow remains the next concrete verification step:

```text
Quote request
   ↓
Slack notification
   ↓
Human opens form
   ↓
Approval
   ↓
Customer-facing delivery
   ↓
Deal stage update
```

## Gmail

The threading fix is confirmed for the first reply.

A multi-turn conversation test with at least three back-and-forth messages remains recommended.

## Meeting Booking

Meeting booking has been verified end-to-end, including:

- Date/timezone handling
- `hs_timestamp`
- `hs_meeting_end_time`
- HubSpot meeting-object requirements

---

# Current Scope & Limitations

This implementation intentionally prioritizes **clear boundaries, controlled automation, and demonstrable workflow behavior** over claiming features that are not built.

### Currently implemented

- Four-channel customer intake
- Channel normalization
- Webhook signature validation for SMS and WhatsApp
- HubSpot identity resolution
- Identity conflict detection
- HubSpot-based conversation history
- OpenAI sales agent
- Qdrant knowledge retrieval
- Gemini embeddings
- Controlled HubSpot property updates
- Meeting booking
- Human-gated quote workflow
- Slack human notification
- Multi-channel response delivery
- Delivery success/failure logging

### Currently simplified or deferred

- Native HubSpot Quotes
- Voice calling
- WhatsApp voice-note transcription
- MCP exposure
- Active Qdrant metadata filtering
- Semantic caching
- Advanced retrieval ranking/diversity
- Dynamic retrieval-budget allocation
- Anonymous web-chat contact cleanup

---

## Design Philosophy

The project follows a risk-proportional automation model:

```text
Low-risk / reversible
        ↓
Automate directly
        │
        ├── CRM qualification fields
        └── Meeting booking with validation

Higher-risk / identity uncertainty
        ↓
Block or require review
        │
        └── Conflicting phone/email identity

Financial / pricing commitment
        ↓
Human approval required
        │
        └── Quote creation and delivery
```

The central principle is that the AI should be able to **reason and request actions**, while the workflow layer controls sensitive execution boundaries.

---

## Project Status

**Current status:** Portfolio-scale, multi-channel AI sales automation system with core workflows implemented and tested, human-gated pricing, production-minded safety boundaries, and several clearly documented verification items and deferred capabilities.

The implementation should be evaluated according to what is explicitly built and tested rather than treating deferred features as completed functionality.
