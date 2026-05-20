# Support Ticket Triage with LLM

## Purpose

This case shows a safe LLM-assisted support triage workflow for a small SaaS or service business. Tickets arrive in a Google Sheet, the automation classifies them, detects urgency, routes work to the correct team, writes review queues, and creates draft replies for human review.

## Business problem

Support teams often spend the first minutes of each ticket deciding what it is about, how urgent it is, and who should handle it. Manual triage is slow and inconsistent, especially when billing, refunds, account access, and angry customer messages arrive together.

Current manual process:

1. Support agent opens a new ticket.
2. Agent reads the message and chooses a category.
3. Agent decides priority and routing.
4. Agent writes a first reply.
5. Risky cases are escalated only if the agent notices the risk.

## Automated result

The workflow validates each ticket, prepares a safe LLM prompt, validates structured JSON output, applies deterministic safety rules, writes output rows, logs the decision, and saves a draft reply. The LLM never sends final customer replies directly.

The current n8n demo uses a DeepSeek HTTP Request node with a private n8n credential. `Mock LLM classification` remains in the canvas as a disconnected branch-test helper, but it is not on the normal processing path.

Actors:

- customer: submits a ticket through email, chat, or helpdesk form;
- automation: validates, classifies, applies safety rules, routes, and logs;
- LLM: returns structured triage JSON and a safe draft reply;
- support lead: reviews escalated or risky tickets;
- support team: handles routed tickets.

## Process flow

```text
Manual Trigger
  -> read support_triage / incoming_tickets
  -> prepare ticket batch and safe prompt payload
  -> IF tickets to process
     false -> No unprocessed tickets
     true  -> continue
  -> build DeepSeek request
  -> DeepSeek classify ticket
  -> normalize DeepSeek response
  -> validate LLM JSON and apply safety rules
  -> IF review queue required
  -> append triaged_tickets
  -> append human_review_queue when required
  -> append audit_log
  -> mark incoming_tickets row processed
  -> Telegram review alert when required
```

The n8n workflow also includes `TEST - Mock ticket input`. This node is not on the normal `Manual Trigger` path. It is used only for branch testing by changing `TEST_CASE` to `refund`, `account_access`, `technical`, `account_deletion`, `sales`, `complaint`, `security`, or `unclear`.

Acceptance criteria:

- every ticket receives one category from the allowed list;
- if no unprocessed tickets exist, the workflow ends in a clear no-work branch;
- refund, legal, security, payment, account deletion, angry, unclear, and low-confidence cases require human review;
- invalid LLM output is not trusted and goes to manual triage;
- only draft replies are created;
- workflow files contain no real credentials, private endpoints, or customer data.

## Files

| File | Description |
|---|---|
| `sample_tickets.json` | Fake input tickets used for testing |
| `expected_outputs.json` | Expected triage results for the sample tickets |
| `prompt_template.md` | Safe prompt template with JSON-only output rules |
| `workflow.n8n.json` | Sanitized 27-node n8n workflow export for Google Sheets, DeepSeek, review queue, and Telegram demo |

## Input contract

| Field | Type | Required | Notes |
|---|---:|---:|---|
| `ticket_id` | string | yes | Internal helpdesk ticket id |
| `created_at` | string | yes | ISO date/time |
| `channel` | string | yes | `email`, `chat`, `form`, or `social` |
| `customer.name` | string | no | Fake sample name only |
| `customer.email` | string | no | Fake sample email only |
| `customer.company` | string | no | Fake sample company only |
| `subject` | string | yes | Ticket subject |
| `message` | string | yes | Customer message |
| `account_id` | string | no | Demo account id, not a real private id |
| `order_id` | string | no | Demo order or invoice id |
| `locale` | string | no | Example: `en-US` |
| `attachments_count` | number | no | Used only for routing context |
| `status` | string | no | Initial status, usually `new` |
| `processed` | string/boolean | no | `FALSE` or empty means the row can be processed |

Validation rules:

- `ticket_id`, `created_at`, `channel`, `subject`, and `message` must exist.
- `message` must not be empty after trimming.
- Unknown channels are accepted but logged as `other`.
- Attachments are not sent to the LLM in this demo; only metadata is passed.
- Messages with unclear intent are routed to manual review.

## Output contract

LLM output schema:

| Field | Type | Required | Notes |
|---|---:|---:|---|
| `category` | string | yes | `billing`, `technical_issue`, `account_access`, `refund_request`, `cancellation`, `sales_question`, `complaint`, or `other` |
| `priority` | string | yes | `low`, `normal`, `high`, or `urgent` |
| `sentiment` | string | yes | `neutral`, `positive`, `worried`, `angry`, or `unclear` |
| `summary` | string | yes | Short factual summary |
| `routing_team` | string | yes | `billing`, `technical_support`, `account_support`, `sales`, or `support_lead` |
| `needs_human_review` | boolean | yes | True for risky or low-confidence cases |
| `risk_flags` | array | yes | Example: `refund`, `payment_issue`, `angry_customer` |
| `draft_reply` | string | yes | Safe draft for a human reviewer |
| `confidence` | number | yes | 0 to 1 |

Output sheets:

| Sheet | Purpose |
|---|---|
| `triaged_tickets` | One classification row per processed ticket |
| `human_review_queue` | Risky, high-priority, urgent, unclear, or low-confidence tickets |
| `audit_log` | Workflow event log |

Priority rules:

- `urgent`: security issue, active service outage, repeated API 5xx errors, or angry customer requesting manager contact today.
- `high`: refund request, payment issue, account deletion, legal threat, blocked login, or cancellation request.
- `normal`: standard billing, sales, or non-blocking technical question.
- `low`: general information request with no clear customer impact.

Status lifecycle:

```text
received -> validated -> llm_requested -> llm_json_validated -> rules_applied -> routed -> logged
received -> invalid_input -> manual_triage -> logged
received -> validated -> llm_invalid_output -> manual_triage -> logged
received -> validated -> llm_timeout -> retry -> manual_triage -> logged
```

## Error handling

| Error | Handling |
|---|---|
| Missing required input | Route to manual triage and log validation error |
| No unprocessed tickets | End in `No unprocessed tickets`; do not call DeepSeek, write sheets, or send Telegram |
| DeepSeek timeout or transport failure | Retry at the HTTP node level where n8n treats the call as failed |
| DeepSeek non-2xx or empty response | Normalize as `llm_error`, discard model output, and route to manual triage |
| Invalid JSON | Do not parse loosely; route to manual triage |
| Schema mismatch | Route to manual triage with `low_llm_confidence` flag |
| Confidence below `0.75` | Require human review and add `low_llm_confidence` |
| Refund, legal, security, angry customer, payment issue, account deletion, cancellation, or unclear intent | Require human review |
| Google Sheets write failure | Stop before marking the source row processed |
| Telegram failure | Sheet rows and audit log remain the source of truth |

Edge cases:

- one message contains both a refund request and a technical issue;
- customer asks to delete an account;
- message includes legal threat language;
- customer reports suspicious login or security concern;
- LLM returns extra text around JSON;
- ticket has valid metadata but vague message.

## Setup notes

1. Import `workflow.n8n.json` into n8n.
2. Reconnect Google Sheets and Telegram credentials manually.
3. Select the Google Sheet named `support_triage` in each Google Sheets node.
4. Keep sheet tabs bound by name: `incoming_tickets`, `triaged_tickets`, `human_review_queue`, `audit_log`.
5. Use Telegram credential `TG_PORTFOLIO_TRIAGEPILOT`.
6. Store the TriagePilot chat target privately in n8n. If n8n variables are unavailable, set it directly in the Telegram node and keep the value out of the repository.
7. Use a private DeepSeek credential in the `DeepSeek classify ticket` HTTP Request node.
8. Keep `deepseek-chat` as the default model unless testing another model deliberately.
9. Keep `Mock LLM classification` disconnected unless you are testing branch behavior without an LLM call.
10. Test with `sample_tickets.json` and compare with `expected_outputs.json`.

Branch testing:

1. Open `TEST - Mock ticket input`.
2. Change `TEST_CASE` to `refund`, `account_access`, `technical`, `account_deletion`, `sales`, `complaint`, `security`, or `unclear`.
3. Change the mock `row_number` if row 2 is not a disposable demo row.
4. Execute the mock node and continue through the workflow.
5. Use only fake/disposable demo rows because downstream nodes still write to Google Sheets.

Production adaptation:

The portfolio workflow starts with `Manual Trigger` for safe demos. In a client or internal deployment, replace or pair it with one of these triggers:

| Trigger | Use when | Notes |
|---|---|---|
| `Schedule Trigger` | Tickets are collected in Google Sheets or another queue | Simple polling model; keep the no-work branch |
| `Webhook Trigger` | A form, helpdesk, chat widget, or backend can push one ticket at a time | Add secret-header validation, idempotency by `ticket_id`, and `Respond to Webhook` |
| Helpdesk API polling | Zendesk, Freshdesk, Intercom, HelpScout, or CRM is the source of truth | Write tags/queue/status back to the helpdesk; keep Google Sheets as audit mirror only |

Production workflows should also add an error workflow, stricter idempotency checks, rate limits, and a dead-letter/retry queue for repeated DeepSeek or sheet failures. The LLM should still create draft replies only; final customer replies require human review.

## Demo status

Portfolio demo. This is a sanitized, inactive workflow demo, not production-ready customer support software. It demonstrates business logic, safe LLM use, data contracts, routing, and error handling.
