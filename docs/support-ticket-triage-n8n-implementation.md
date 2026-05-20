# Support Ticket Triage n8n Implementation

Status: current working implementation guide for a sanitized portfolio demo.

This document describes the n8n workflow used for the Support Ticket Triage with LLM case. It does not contain real sheet IDs, chat IDs, credential IDs, API keys, tokens, private URLs, or customer data.

## Business Logic

Business flow:

```text
incoming_tickets -> prepare prompt payload -> LLM JSON classification -> schema validation -> safety rules -> output sheets -> audit log -> Telegram review alert
```

The workflow is built for a small SaaS or service business where support tickets arrive from email, chat, forms, or social messages.

The LLM may generate a draft reply, but it must never send the final reply directly to a customer.

## Current Demo Mode

The current n8n workflow uses DeepSeek through a sanitized HTTP Request node.

Rules:

- the DeepSeek API key stays only inside a private n8n credential;
- the repository stores only the credential type/name placeholder, never the credential ID or token;
- `Mock LLM classification` remains in the canvas as a disconnected branch-test helper;
- customer replies remain drafts for human review.

## Private n8n Configuration

| Item | Private n8n value |
|---|---|
| Google Sheets credential | `GSHEETS_PORTFOLIO_AUTOMATION` or the existing Google Sheets credential |
| Telegram credential | `TG_PORTFOLIO_TRIAGEPILOT` |
| LLM credential | `DeepSeek account` or the private DeepSeek credential configured in n8n |
| Google Sheets document | Select `support_triage` from the n8n document list |
| Telegram chat target | Store privately in n8n; use `TRIAGEPILOT_CHAT_ID` if variables are available, otherwise set it directly in the Telegram node |

Do not commit real credential IDs, tokens, chat IDs, sheet IDs, LLM API keys, OAuth values, or private URLs.

## Required Sheet Tabs

| Sheet | Purpose |
|---|---|
| `incoming_tickets` | Source tickets waiting for triage |
| `triaged_tickets` | Classification result for every processed ticket |
| `human_review_queue` | Tickets requiring review before customer response |
| `Priority_Matrix` | Optional priority reference table |
| `SLA_Policy` | Optional SLA reference table |
| `Queue_Routing` | Optional routing reference table |
| `audit_log` | Workflow event history |
| `config` | Optional demo configuration |

## Required Sheet Columns

### `incoming_tickets`

```text
ticket_id
created_at
customer_name
customer_email
subject
message
channel
status
processed
```

### `triaged_tickets`

```text
triaged_at
ticket_id
category
priority
sentiment
routing_team
needs_human_review
risk_flags
confidence
summary
draft_reply
source_row
telegram_notified
```

### `human_review_queue`

```text
created_at
ticket_id
priority
risk_flags
summary
draft_reply
review_status
review_owner
telegram_message_id
```

### `audit_log`

```text
timestamp
workflow_name
source_sheet
source_row
entity_id
event_type
status
message
```

The imported `support_triage` sheet also has optional result columns in `incoming_tickets`:

```text
category
priority
needs_human_review
notes
```

## Current Node Sequence

The current sanitized n8n workflow has 27 nodes.

```text
Manual Trigger
  -> Read incoming_tickets
  -> Prepare ticket batch
  -> IF tickets to process?
     true  -> continue
     false -> No unprocessed tickets
  -> Build DeepSeek request
  -> DeepSeek classify ticket
  -> Normalize DeepSeek response
  -> Validate LLM JSON and apply safety rules
  -> IF review queue required

TEST - Mock ticket input
  -> Prepare ticket batch

IF review queue required true
  -> Append triaged_tickets (review)
  -> Restore review context after triage
  -> Append human_review_queue
  -> Restore review context after queue
  -> Append review audit_log
  -> Restore review context after audit
  -> Mark review processed
  -> Restore review context after update
  -> Telegram review alert

IF review queue required false
  -> Append triaged_tickets (standard)
  -> Restore standard context after triage
  -> Append standard audit_log
  -> Restore standard context after audit
  -> Mark standard processed
```

## Node Purposes

| Node | Purpose |
|---|---|
| `Manual Trigger` | Starts the demo manually; workflow is not activated |
| `Read incoming_tickets` | Reads unprocessed tickets from `support_triage` |
| `TEST - Mock ticket input` | Manual branch-test helper for common ticket scenarios |
| `Prepare ticket batch` | Skips processed rows, validates required fields, creates the prompt payload, or returns a no-work control item |
| `IF tickets to process?` | Routes real ticket items to DeepSeek and no-work control items to the no-work branch |
| `No unprocessed tickets` | Cleanly ends a run when there is nothing to process |
| `Build DeepSeek request` | Builds the safe JSON-only chat completion request |
| `DeepSeek classify ticket` | Calls DeepSeek with a private n8n credential and retry behavior |
| `Normalize DeepSeek response` | Extracts model content and preserves the original ticket context |
| `Mock LLM classification` | Disconnected fallback helper for branch testing without an LLM call |
| `Validate LLM JSON and apply safety rules` | Parses JSON, validates schema, applies deterministic escalation rules, builds output rows |
| `IF review queue required` | Splits review-required tickets from standard triage tickets |
| `Append ...` Google Sheets nodes | Write triage, review, and audit rows |
| `Restore ... context ...` Code nodes | Restore the original ticket item after Google Sheets writes |
| `Mark ... processed` Google Sheets nodes | Update source row `status` and `processed` in `incoming_tickets` by `row_number` |
| `Telegram review alert` | Sends internal TriagePilot alert only for review-required tickets |

## Trigger Architecture

The portfolio workflow uses `Manual Trigger` because it is safer for demos, imports, and reviewer testing. It prevents an imported workflow from running before credentials and sheet tabs are bound.

The no-work branch is intentional:

```text
Prepare ticket batch
  -> has_work=true  -> DeepSeek path
  -> has_work=false -> No unprocessed tickets
```

This follows the same principle as an `ok_empty` state in production automation: empty input is a valid outcome, not an error. The workflow must not create fake placeholder rows, call DeepSeek, write output sheets, or send Telegram when there is nothing to process.

Production trigger options:

| Trigger | Best fit | Required additions |
|---|---|---|
| `Schedule Trigger` | Google Sheets remains the ticket queue | Keep no-work branch, add schedule window, add global error workflow |
| `Webhook Trigger` | Forms, backend, chat widget, or helpdesk can POST each ticket | Verify a secret header, validate payload schema, check idempotency by `ticket_id`, add `Respond to Webhook` |
| Helpdesk API polling | Zendesk, Freshdesk, Intercom, HelpScout, or CRM is source of truth | Read only new tickets, write tags/status back to helpdesk, keep Sheets as audit mirror |

Recommended production webhook flow:

```text
Webhook Trigger
  -> Verify webhook secret
  -> Validate payload schema
  -> Check idempotency by ticket_id
  -> Build DeepSeek request
  -> DeepSeek classify ticket
  -> Normalize DeepSeek response
  -> Validate LLM JSON and apply safety rules
  -> IF review queue required
  -> Write triage/review/audit records
  -> Telegram alert for review cases
  -> Respond to Webhook with internal triage result
```

Do not send final customer replies from this workflow. Draft replies are internal review artifacts only.

## Allowed LLM Output

The model must return exactly one JSON object with this schema:

```json
{
  "category": "billing",
  "priority": "high",
  "sentiment": "angry",
  "summary": "Customer was charged twice and asks for refund.",
  "routing_team": "billing",
  "needs_human_review": true,
  "risk_flags": ["refund", "angry_customer"],
  "draft_reply": "Short safe draft reply for human review.",
  "confidence": 0.86
}
```

Allowed categories:

```text
billing
technical_issue
account_access
refund_request
cancellation
sales_question
complaint
other
```

Allowed priorities:

```text
low
normal
high
urgent
```

Allowed risk flags:

```text
refund
payment_issue
account_deletion
legal_threat
security_issue
angry_customer
unclear_intent
low_llm_confidence
cancellation
manager_requested
```

## Safety Rules

`Validate LLM JSON and apply safety rules` enforces deterministic rules after the model response.

Human review is required for:

| Trigger | Handling |
|---|---|
| Refund request | Add `refund`, route to billing/review |
| Payment issue | Add `payment_issue`, route to review |
| Account deletion | Add `account_deletion`, route to support lead |
| Legal threat | Add `legal_threat`, route to support lead |
| Security issue | Add `security_issue`, route to review |
| Angry customer | Add `angry_customer`, route to support lead |
| Unclear intent | Add `unclear_intent`, route to support lead |
| Cancellation | Add `cancellation`, route to support lead |
| Confidence below `0.75` | Add `low_llm_confidence`, route to review |
| Priority `high` or `urgent` | Put into review queue even when the model did not set a risk flag |

The LLM draft is saved as an internal draft only. No customer message is sent.

## Mock Branch Testing

The workflow includes `TEST - Mock ticket input` for manual testing.

How to use it:

1. Open `TEST - Mock ticket input`.
2. Change the first setting in the JavaScript:

```javascript
const TEST_CASE = 'refund';
```

Allowed values:

```text
refund
account_access
technical
account_deletion
sales
complaint
security
unclear
```

3. Change the mock `row_number` if row 2 is not a disposable demo row.
4. Execute the test node.
5. Continue through `Prepare ticket batch` and the rest of the workflow.

Use only fake/disposable demo rows because downstream update nodes still mark the selected `incoming_tickets` row as processed.

## Telegram Message

Telegram uses credential `TG_PORTFOLIO_TRIAGEPILOT`. The chat target is configured privately in n8n and must not be committed.

Message template:

```text
Support ticket needs review
Ticket: <ticket_id>
Priority: <priority>
Category: <category>
Risk flags: <risk_flags>
Summary: <summary>
Draft reply: <draft_reply>
```

Use plain text only. Do not enable Markdown or HTML parse mode.

## Error Handling

| Failure | Handling |
|---|---|
| Missing required input | Create fallback triage output and route to `human_review_queue` |
| No unprocessed tickets | Route to `No unprocessed tickets`; no DeepSeek, sheet write, or Telegram call |
| DeepSeek timeout or transport failure | Retry at the HTTP node level where n8n treats the call as failed |
| DeepSeek non-2xx or empty response | Normalize as `llm_error`, discard model output, create fallback output, route to review |
| Invalid LLM JSON | Discard model result, create fallback output, route to review |
| Unknown enum value | Discard model result, create fallback output, route to review |
| Confidence below threshold | Keep safe output, add `low_llm_confidence`, route to review |
| Google Sheets append fails | Workflow stops before marking source row processed |
| Google Sheets update fails | Output and audit rows may exist; source row needs manual review |
| Telegram alert fails | Sheet rows and audit log remain the source of truth |
| Schema mismatch after changing sheet headers | Refresh columns in affected Google Sheets nodes |
| Google credential has no access to `support_triage` | Share the spreadsheet with the Google account connected to n8n, or reconnect the credential to the account that owns the sheet |

## Test Plan

Before testing, use fake sample rows only and keep the workflow inactive.

### Test 0: No unprocessed tickets

Expected:

- `Prepare ticket batch` returns one control item with `has_work=false`;
- `IF tickets to process?` routes to `No unprocessed tickets`;
- DeepSeek, Google Sheets write nodes, and Telegram are not executed.

### Test 1: Refund / payment issue

Expected:

- row appended to `triaged_tickets`;
- row appended to `human_review_queue`;
- `risk_flags` includes `refund` and `payment_issue`;
- Telegram review alert sent;
- source row marked `processed=TRUE`, `status=review_required`.

### Test 2: Sales question

Expected:

- row appended to `triaged_tickets`;
- no row appended to `human_review_queue`;
- no Telegram alert sent;
- source row marked `processed=TRUE`, `status=triaged`.

### Test 3: Security issue

Expected:

- row appended to `triaged_tickets`;
- row appended to `human_review_queue`;
- `risk_flags` includes `security_issue`;
- Telegram review alert sent.

### Test 4: Unclear ticket

Expected:

- row appended to `triaged_tickets`;
- row appended to `human_review_queue`;
- `risk_flags` includes `unclear_intent` and `low_llm_confidence`;
- confidence is below `0.75`;
- Telegram review alert sent.

### Test 5: DeepSeek isolated check

Expected:

- DeepSeek returns HTTP `200`;
- `Normalize DeepSeek response` extracts JSON text;
- `Validate LLM JSON and apply safety rules` returns `validation_status=valid_llm_json`;
- a sales question is classified as `sales_question`, `priority=normal`, `routing_team=sales`;
- no Google Sheets or Telegram nodes are reached during the isolated check.

### Test 6: Invalid LLM output

Expected:

- invalid model output is discarded;
- fallback classification is created;
- ticket is routed to `support_lead`;
- audit row explains fallback reason.

## Expected Output Summary

For `projects/support-ticket-triage-llm/sample_tickets.json`, expected classifications are documented in:

```text
projects/support-ticket-triage-llm/expected_outputs.json
```

The real LLM may phrase summaries and drafts differently. The important acceptance criteria are valid JSON, allowed enum values, correct escalation flags, and no direct customer send.

## Build Notes

- The n8n workflow is a portfolio demo, not production helpdesk software.
- Keep `Manual Trigger` until manual tests are stable.
- Do not activate the workflow until the user explicitly decides the demo is ready.
- Do not commit real credentials, chat IDs, sheet IDs, tokens, private URLs, VPS details, LLM API keys, or n8n credential IDs.
- If exporting from n8n, sanitize the workflow before saving it in the repository.
