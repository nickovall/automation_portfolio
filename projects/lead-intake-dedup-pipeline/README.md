# Lead Intake & Dedup Pipeline

## Purpose

This case shows a simple lead intake automation for a small service business. Leads arrive in a Google Sheet from forms, messengers, ads, and manual manager input. The automation validates incoming data, detects duplicates, writes the result to separate review sheets, logs the decision, and notifies the sales team.

## Business problem

Without automation, the team gets duplicate leads, incomplete contacts, late responses, and no clear source attribution. Managers waste time checking the same person across incoming sheets, historical leads, and chats.

## Automated result

The workflow turns raw lead submissions into clean operational records:

- valid new leads go to `valid_leads`;
- duplicates go to `duplicates`;
- invalid rows go to `invalid_leads`;
- every processed row gets an `audit_log` entry;
- source rows in `incoming_leads` are marked as processed.

## Process flow

```text
Manual Trigger
  -> read valid_leads
  -> seed dedup context
  -> read incoming_leads
  -> process lead batch
  -> IF invalid lead
  -> IF duplicate lead
  -> append valid_leads / invalid_leads / duplicates
  -> restore original item context after each sheet write
  -> append audit_log
  -> mark incoming_leads row processed
  -> Telegram LeadGate notification
```

The n8n workflow also includes `TEST - Mock branch input`. This node is not on the normal `Manual Trigger` path. It is used only for branch testing by changing `TEST_CASE` to `invalid`, `duplicate`, or `valid`.

Actors:

- lead source: form, messenger, ads, or manual CSV import;
- automation: validates, deduplicates, routes, logs, and marks rows processed;
- sales manager: receives valid, duplicate, and invalid lead notifications;
- Google Sheets: stores source rows, outputs, and audit history.

## Files

| File | Description |
|---|---|
| `workflow.n8n.json` | Sanitized 29-node n8n workflow export for manual Google Sheets demo |
| `sample_leads.csv` | Test input data with valid leads, duplicates, and invalid rows |

## Input contract

| Field | Type | Required | Notes |
|---|---:|---:|---|
| `created_at` | string | yes | ISO-like date/time from source |
| `source` | string | yes | Form, Telegram, ads, manual import |
| `name` | string | no | Can be empty for raw leads |
| `phone` | string | conditional | Required if email is empty; normalized to digits |
| `email` | string | conditional | Required if phone is empty; normalized to lowercase |
| `message` | string | no | User request text |
| `utm_source` | string | no | Marketing attribution |
| `status` | string | no | Initial status, usually `new` |
| `processed` | string/boolean | no | `FALSE` or empty means the row can be processed |
| `lead_id` | string | no | Filled by the workflow after processing |
| `dedup_key` | string | no | Filled by the workflow after processing |
| `notes` | string | no | Filled by the workflow after processing |

Validation rules:

- `created_at` and `source` must exist.
- A valid row must contain either a usable phone number or a valid email address.
- Phone values are stripped to digits before matching.
- Email values must contain one `@` and a domain.
- Rows with `processed=TRUE` are skipped.
- Invalid rows are logged and are not added to `valid_leads`.

## Output contract

| Field | Type | Notes |
|---|---:|---|
| `lead_id` | string | Deterministic demo id generated from the dedup key |
| `dedup_key` | string | `phone:<digits>` or `email:<lowercase_email>` |
| `action` | string | `valid_lead_created`, `duplicate_detected`, or `invalid_lead` |
| `status` | string | Current processing status |
| `routing_target` | string | `valid_leads`, `duplicates`, or `invalid_leads` |
| `error_reason` | string/null | Present when status is `invalid` or `failed` |

Status lifecycle:

```text
received -> normalized -> validated -> duplicate_checked -> valid -> logged -> processed -> notified
received -> normalized -> invalid -> logged
received -> normalized -> validated -> duplicate -> logged -> processed -> notified
```

## Dedup logic

Priority:

1. Exact phone match after normalization.
2. Exact lowercase email match.
3. Existing key in `valid_leads` or earlier in the same batch goes to `duplicates`.
4. Missing or invalid phone/email goes to the invalid branch.

Edge cases:

- same lead submits twice from different sources;
- same email with updated phone number;
- same phone with corrected spelling of name;
- row has contact data but invalid format;
- Google Sheets append succeeds but Telegram notification fails.

## Error handling

| Error | Handling |
|---|---|
| Missing or invalid contact data | Send to invalid branch and log row |
| Google Sheets read/write failure | Stop the workflow before marking source rows processed |
| Telegram failure | Investigate credential/chat setup; output rows and audit log remain the source of truth |
| Telegram parse error | Use plain text only; dynamic values are sanitized before sending |
| Duplicate found | Log to `duplicates` instead of creating a second valid lead |
| Google Sheets node returns write metadata | Restore original lead item before the next audit/update/Telegram node |

Acceptance criteria:

- valid lead creates one row in `valid_leads`;
- duplicate lead does not create a second valid lead;
- invalid contact data is logged with a reason;
- every processed source row creates an audit row;
- workflow JSON contains no real credentials, sheet IDs, chat IDs, or private endpoints.

## Setup notes

1. Import `workflow.n8n.json` into n8n.
2. Reconnect Google Sheets and Telegram credentials manually.
3. Select the Google Sheet named `lead_intake` in each Google Sheets node.
4. Keep sheet tabs bound by name: `incoming_leads`, `valid_leads`, `invalid_leads`, `duplicates`, `audit_log`.
5. Use Telegram credential `TG_PORTFOLIO_LEADGATE`.
6. Store the LeadGate chat target privately in n8n, for example as `LEADGATE_CHAT_ID`.
7. Do not enable Markdown or HTML parse mode in Telegram nodes.
8. Import `sample_leads.csv` into `incoming_leads`.
9. Run the workflow manually before adding any schedule trigger.

Branch testing:

1. Open `TEST - Mock branch input`.
2. Change `const TEST_CASE = 'invalid';` to `duplicate` or `valid`.
3. Change the mock `source_row_number` if row 2 is not a disposable demo row.
4. Execute the mock node and continue from `IF invalid lead`.
5. Use only fake/disposable demo rows because downstream nodes still write to Google Sheets.

## Demo status

Portfolio demo. The workflow is intentionally sanitized and inactive by default. It shows process design and implementation logic, not production-ready CRM or lead-management software.
