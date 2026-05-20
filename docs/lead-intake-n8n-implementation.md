# Lead Intake n8n Implementation

Status: current working implementation guide for a sanitized portfolio demo.

This document describes the n8n workflow used for the Lead Intake & Dedup Pipeline case. It does not contain real sheet IDs, chat IDs, credential IDs, API keys, tokens, private URLs, or customer data.

## Business Logic

Business flow:

```text
incoming_leads -> normalize -> validate -> deduplicate -> write result sheet -> audit log -> mark source row -> Telegram alert
```

The workflow is built for a small service business where leads arrive from landing forms, Telegram, ads, referrals, and manual imports.

Actors:

| Actor | Role |
|---|---|
| Lead source | Adds raw lead rows to `incoming_leads` |
| n8n workflow | Normalizes, validates, deduplicates, writes outputs, logs, and notifies |
| Sales manager | Reviews valid, duplicate, and invalid notifications |
| Google Sheets | Stores source rows, result tables, and audit log |
| Telegram bot | Sends internal LeadGate alerts |

## Private n8n Configuration

| Item | Private n8n value |
|---|---|
| Google Sheets credential | `GSHEETS_PORTFOLIO_AUTOMATION` |
| Telegram credential | `TG_PORTFOLIO_LEADGATE` |
| Google Sheets document | Select `lead_intake` from the n8n document list |
| Telegram chat target | Store privately in n8n, for example as `LEADGATE_CHAT_ID` |

Do not commit real credential IDs, tokens, chat IDs, sheet IDs, OAuth values, or private URLs. The repository workflow export uses placeholders only.

## Required Sheet Tabs

| Sheet | Purpose |
|---|---|
| `incoming_leads` | Source rows waiting for processing |
| `valid_leads` | Valid new leads |
| `invalid_leads` | Rows rejected by validation |
| `duplicates` | Leads already known by phone or email |
| `audit_log` | Workflow event history |
| `config` | Optional demo configuration |

## Required Sheet Columns

### `incoming_leads`

```text
created_at
source
name
phone
email
message
utm_source
status
processed
lead_id
dedup_key
notes
```

### `valid_leads`

```text
processed_at
lead_id
source
name
phone_normalized
email_normalized
message
utm_source
status
dedup_key
source_row
telegram_notified
```

### `invalid_leads`

```text
processed_at
source
name
phone
email
reason
raw_payload
source_row
```

### `duplicates`

```text
processed_at
dedup_key
existing_lead_id
new_source
action
source_row
notes
telegram_notified
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

## Current Node Sequence

The current n8n workflow has 29 nodes.

```text
Manual Trigger
  -> Read valid_leads
  -> Seed valid context
  -> Read incoming_leads
  -> Process lead batch
  -> IF invalid lead

TEST - Mock branch input
  -> IF invalid lead

IF invalid lead true
  -> Append invalid_leads
  -> Restore invalid context after append
  -> Append invalid audit_log
  -> Restore invalid context after audit
  -> Mark invalid processed
  -> Restore invalid context after update
  -> Telegram invalid alert

IF invalid lead false
  -> IF duplicate lead

IF duplicate lead true
  -> Append duplicates
  -> Restore duplicate context after append
  -> Append duplicate audit_log
  -> Restore duplicate context after audit
  -> Mark duplicate processed
  -> Restore duplicate context after update
  -> Telegram duplicate alert

IF duplicate lead false
  -> Append valid_leads
  -> Restore valid context after append
  -> Append valid audit_log
  -> Restore valid context after audit
  -> Mark valid processed
  -> Restore valid context after update
  -> Telegram new lead alert
```

## Node Purposes

| Node | Purpose |
|---|---|
| `Manual Trigger` | Starts the demo manually; workflow is not activated yet |
| `Read valid_leads` | Reads existing valid leads for deduplication context |
| `Seed valid context` | Stores existing `dedup_key`, phone, and email keys in a context object |
| `Read incoming_leads` | Reads source rows from `incoming_leads` |
| `Process lead batch` | Skips processed rows, normalizes fields, validates contact data, detects duplicates, builds output rows and Telegram messages |
| `TEST - Mock branch input` | Manual branch-test helper for invalid, duplicate, and valid paths |
| `IF invalid lead` | Sends invalid rows to the invalid branch |
| `IF duplicate lead` | Sends duplicate rows to the duplicate branch |
| `Append ...` Google Sheets nodes | Write result and audit rows |
| `Restore ... context ...` Code nodes | Restore the original processed item after Google Sheets writes, because write nodes return sheet write metadata instead of the original lead item |
| `Mark ... processed` Google Sheets nodes | Update source row status in `incoming_leads` by `row_number` |
| `Telegram ... alert` nodes | Send internal LeadGate notifications after sheet writes and source update |

## Processing Logic

`Process lead batch` handles the main logic in one Code node so the workflow stays readable and avoids fragile multi-input joins.

Core rules:

```javascript
const isProcessed = (value) => {
  if (value === true) return true;
  const normalized = String(value ?? '').trim().toLowerCase();
  return ['true', 'yes', '1', 'processed', 'done'].includes(normalized);
};

const text = (value) => String(value ?? '').trim();
const email = (value) => text(value).toLowerCase();
const phone = (value) => text(value).replace(/\D/g, '');
const validEmail = (value) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
```

Dedup priority:

1. Existing `dedup_key` in `valid_leads`.
2. Exact normalized phone match.
3. Exact lowercase email match.
4. Earlier new valid lead in the same batch.

Validation rules:

| Rule | Result |
|---|---|
| `processed` is true-like | Skip row |
| `created_at` is missing | Invalid |
| `source` is missing | Invalid |
| Phone has fewer than 7 digits and email is invalid or empty | Invalid |
| Phone is valid | Primary `dedup_key` is `phone:<digits>` |
| Phone is not valid but email is valid | Primary `dedup_key` is `email:<lowercase_email>` |

The Code node also builds parse-safe Telegram text. Dynamic values are sanitized before sending to Telegram so underscores, brackets, punctuation, or copied text do not break Telegram entity parsing.

## Google Sheets Operations

All Google Sheets nodes use:

```text
Resource: Sheet Within Document
Document: lead_intake selected from list
Sheet: tab name
Cell format: RAW
```

Important setup rule: bind the document from the n8n list and bind tabs by name, not by a pasted private sheet URL.

### Invalid branch writes

`Append invalid_leads`:

| Column | Source |
|---|---|
| `processed_at` | processed timestamp |
| `source` | lead source |
| `name` | raw name |
| `phone` | raw phone |
| `email` | raw email |
| `reason` | validation reason |
| `raw_payload` | compact source row payload |
| `source_row` | Google Sheets row number |

`Append invalid audit_log`:

| Column | Value |
|---|---|
| `workflow_name` | `Lead Intake & Dedup Pipeline` |
| `event_type` | `invalid_lead` |
| `status` | `invalid` |
| `message` | validation reason |

`Mark invalid processed` updates `incoming_leads`:

```text
processed = TRUE
status = invalid
notes = validation reason
```

### Duplicate branch writes

`Append duplicates`:

| Column | Source |
|---|---|
| `processed_at` | processed timestamp |
| `dedup_key` | current lead dedup key |
| `existing_lead_id` | matched lead id |
| `new_source` | current lead source |
| `action` | `duplicate_detected` |
| `source_row` | Google Sheets row number |
| `notes` | duplicate reason |
| `telegram_notified` | demo notification marker |

`Mark duplicate processed` updates `incoming_leads`:

```text
processed = TRUE
status = duplicate
lead_id = existing lead id
dedup_key = current dedup key
notes = Duplicate lead detected
```

### Valid branch writes

`Append valid_leads`:

| Column | Source |
|---|---|
| `processed_at` | processed timestamp |
| `lead_id` | deterministic demo id |
| `source` | lead source |
| `name` | normalized name |
| `phone_normalized` | digits-only phone |
| `email_normalized` | lowercase email |
| `message` | lead message |
| `utm_source` | attribution |
| `status` | `valid` |
| `dedup_key` | primary dedup key |
| `source_row` | Google Sheets row number |
| `telegram_notified` | demo notification marker |

`Mark valid processed` updates `incoming_leads`:

```text
processed = TRUE
status = valid
lead_id = new demo lead id
dedup_key = current dedup key
notes = New valid lead
```

## Telegram Messages

Telegram nodes use credential `TG_PORTFOLIO_LEADGATE`. The chat target is configured privately in n8n and must not be committed.

The workflow sends plain text messages with webpage preview disabled.

New lead:

```text
New qualified lead
Name: <name>
Contact: <phone_normalized> / <email_normalized>
Source: <source>
UTM: <utm_source>
Action: added to valid leads
```

Duplicate:

```text
Duplicate lead detected
Name: <name>
Dedup key: <dedup_key>
Matched lead: <existing_lead_id>
Action: logged to duplicates
```

Invalid:

```text
Invalid lead
Source: <source>
Name: <name>
Reason: <reason>
Action: logged to invalid leads
```

Avoid Markdown or HTML parse mode for this workflow. Plain text is more stable for demo data copied from sheets.

## Mock Branch Testing

The workflow includes `TEST - Mock branch input` for manual testing.

How to use it:

1. Open `TEST - Mock branch input`.
2. Change the first setting in the JavaScript:

```javascript
const TEST_CASE = 'invalid';
```

Allowed values:

```text
invalid
duplicate
valid
```

3. Execute the test node.
4. Continue from `IF invalid lead` to test the selected branch.

The mock node is not connected to `Manual Trigger`, so normal manual workflow runs still read Google Sheets. Its default `source_row_number` is `2`; change it to a disposable demo row if needed. Use it only on test data, because downstream update nodes still mark the selected `incoming_leads` row as processed.

## Error Handling

| Failure | Handling |
|---|---|
| Missing phone and invalid/missing email | Append to `invalid_leads`, append audit row, mark source row `invalid`, send invalid alert |
| Duplicate phone/email | Append to `duplicates`, append audit row, mark source row `duplicate`, send duplicate alert |
| Google Sheets append fails | Workflow stops before marking source row processed |
| Google Sheets update fails | Output and audit rows may exist; review source row manually |
| Telegram alert fails | Sheet rows and audit log remain the source of truth; check Telegram credential/chat setup |
| Telegram parse error | Use plain text and sanitized values; do not enable Markdown/HTML parse mode |
| `row_number` missing | Make sure Google Sheets read node returns row metadata before enabling source row updates |
| Schema mismatch after changing sheet headers | Refresh columns in affected Google Sheets nodes |

## Test Plan

Before testing, use fake sample rows only and keep the workflow inactive.

### Test 1: New valid lead

Input:

```text
Unique phone or email, usable name/source/message
```

Expected:

- one row appended to `valid_leads`;
- one `valid_lead_created` row appended to `audit_log`;
- source row marked `processed=TRUE`, `status=valid`;
- Telegram new lead alert sent.

### Test 2: Duplicate lead

Input:

```text
Same normalized phone or same lowercase email as an existing valid lead
```

Expected:

- one row appended to `duplicates`;
- no second valid lead created;
- one `duplicate_detected` audit row;
- source row marked `processed=TRUE`, `status=duplicate`;
- Telegram duplicate alert sent.

### Test 3: Invalid lead

Input:

```text
No usable phone and no valid email
```

Expected:

- one row appended to `invalid_leads`;
- one `invalid_lead` audit row;
- source row marked `processed=TRUE`, `status=invalid`;
- Telegram invalid alert sent.

### Test 4: Re-run safety

Run the workflow again without adding new unprocessed rows.

Expected:

- no new output rows;
- no new Telegram alerts;
- already processed rows are skipped.

### Test 5: Mock branch node

Run `TEST - Mock branch input` with each value:

```text
invalid
duplicate
valid
```

Expected:

- selected branch receives one item;
- corresponding Google Sheets writes use the mock row fields;
- Telegram node uses `TG_PORTFOLIO_LEADGATE`;
- no Markdown/HTML parse error occurs.

## Expected Output Summary

For `projects/lead-intake-dedup-pipeline/sample_leads.csv`:

| Source row | Expected status | Output sheet |
|---|---|---|
| Anna Kim from landing form | `valid` | `valid_leads` |
| Anna K. from Telegram | `duplicate` | `duplicates` |
| Max Lee from Facebook ads | `valid` | `valid_leads` |
| Manual import with no valid contact | `invalid` | `invalid_leads` |
| Sergey Ivanov email-only lead | `valid` | `valid_leads` |

Expected audit count:

```text
5 audit_log rows for 5 processed source rows
```

Expected Telegram alerts:

```text
3 new lead alerts
1 duplicate lead alert
1 invalid lead alert
```

## Build Notes

- The n8n workflow is a portfolio demo, not production CRM software.
- Keep `Manual Trigger` until manual tests are stable.
- Do not activate the workflow until the user explicitly decides the demo is ready.
- Do not commit real credentials, chat IDs, sheet IDs, tokens, private URLs, VPS details, or n8n credential IDs.
- If exporting from n8n, sanitize the workflow before saving it in the repository.
