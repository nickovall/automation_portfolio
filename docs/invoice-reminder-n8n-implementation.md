# Invoice Reminder Automation - n8n Implementation

This document describes the n8n build for the portfolio invoice reminder case.

The workflow is a sanitized portfolio demo. It does not include real credentials, real Sheet IDs, real customer data, private URLs, or email API keys.

## Business Process

Business problem:

- unpaid invoices are tracked manually;
- reminders can be missed or duplicated;
- overdue escalation is inconsistent;
- the team lacks a clear audit log of reminder decisions.

Current manual process:

1. Finance owner opens the invoice sheet.
2. Owner checks due dates and previous reminders.
3. Owner writes customer reminders by hand.
4. Owner updates invoice status manually.
5. Manager escalation happens only when somebody notices an old overdue invoice.

Target automated process:

1. Workflow reads previous reminder queue rows.
2. Workflow builds anti-duplicate keys from `invoice_id`, `reminder_type`, and queue date.
3. Workflow reads invoice rows.
4. Workflow filters out paid/closed/not-due/already-reminded-today rows.
5. Workflow creates reminder drafts for due-soon and overdue invoices.
6. Risky or invalid rows go to manager review.
7. Workflow writes reminder queue, payment status log, audit log, and invoice metadata updates.
8. Workflow ends in a visible no-work branch when nothing needs action.

Actors:

- finance owner;
- account manager;
- customer;
- n8n workflow;
- Google Sheets.

## Data Model

Google Sheet document:

```text
invoice_reminders
```

In n8n, select this document with `From list`. The live workflow stores the internal spreadsheet locator returned by n8n, while the repository export keeps `INVOICE_REMINDER_SPREADSHEET_LOCATOR` as a public-safe placeholder.

Required sheets:

```text
invoices
reminders_queue
message_templates
payment_status_log
audit_log
config
```

The current public workflow uses:

```text
invoices
reminders_queue
payment_status_log
audit_log
```

## Sheet Columns

`invoices`:

```text
invoice_id
client_name
client_email
amount
currency
issue_date
due_date
status
reminders_sent
last_reminder_at
paid_at
days_overdue
days_to_due
processed
```

`reminders_queue`:

```text
created_at
invoice_id
client_name
client_email
reminder_type
message
status
sent_at
telegram_notified
```

`payment_status_log`:

```text
timestamp
invoice_id
old_status
new_status
source
notes
```

`audit_log`:

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

## Node Sequence

| # | Node | Purpose |
|---:|---|---|
| 1 | `Manual Trigger` | Safe manual demo start |
| 2 | `Read reminders_queue` | Load previous queue rows |
| 3 | `Seed reminder send context` | Build anti-duplicate keys |
| 4 | `Read invoices` | Load invoice rows |
| 5 | `Prepare invoice reminder batch` | Validate, calculate due state, build queue/audit/update payload |
| 6 | `IF invoices need reminders?` | Visible no-work gate |
| 7 | `No invoice reminder work` | Clean end state when no invoice needs action |
| 8 | `IF manager review required` | Split normal drafts from risky review items |
| 9 | `Append reminders_queue (review)` | Write risky/invalid rows to reminder queue |
| 10 | `Append reminders_queue (standard)` | Write normal reminder drafts to reminder queue |
| 11 | Restore context nodes | Recover original invoice payload after Google Sheets write metadata |
| 12 | `Append payment_status_log` | Write status transition history |
| 13 | `Append audit_log` | Write structured audit event |
| 14 | `Update invoices reminder metadata` | Update source invoice row |
| 15 | `TEST - Mock invoice input` | Disconnected branch testing helper |

## Code Node Logic

`Seed reminder send context`:

```javascript
const text = (value) => String(value ?? '').trim();
const dateOnly = (value) => text(value).slice(0, 10);
const keys = [];

for (const item of items) {
  const invoiceId = text(item.json?.invoice_id);
  const reminderType = text(item.json?.reminder_type);
  const createdDate = dateOnly(item.json?.created_at);
  if (invoiceId && reminderType && createdDate) {
    keys.push(`${invoiceId}:${reminderType}:${createdDate}`);
  }
}

return [{ json: { existing_send_keys: [...new Set(keys)] } }];
```

`Prepare invoice reminder batch`:

- reads each invoice row;
- skips paid or closed rows;
- skips rows already reminded today;
- calculates days until due and days overdue;
- creates `due_soon`, `overdue`, `invalid_contact`, or `manager_review` reminder types;
- builds the duplicate key as `invoice_id + ':' + reminder_type + ':' + as_of_date`;
- skips rows whose key already exists in `reminders_queue`;
- sets `needs_manager_review=true` for invalid contact, severe overdue, or high reminder count;
- returns a control item with `has_work=false` when no invoice needs action.

## Reminder Rules

| Condition | Result |
|---|---|
| `status` is `paid`, `cancelled`, `void`, or `refunded` | Skip |
| Due date is more than 3 days away | Skip |
| Due date is within 3 days | `due_soon`, normal review queue |
| 1-14 days overdue and reminders sent below 3 | `overdue`, normal review queue |
| More than 14 days overdue or reminders sent 3+ | `manager_review`, manager review required |
| Missing client email | `invalid_contact`, manager review required |
| Existing same invoice/type/date key | Skip duplicate queue write |

## Google Sheets Operations

`Read reminders_queue`:

- document: `invoice_reminders`;
- sheet: `reminders_queue`;
- operation: read rows;
- uses `created_at`, `invoice_id`, and `reminder_type` to prevent same-day duplicate queue rows.

`Read invoices`:

- document: `invoice_reminders`;
- sheet: `invoices`;
- operation: read rows.

`Append reminders_queue`:

- sheet: `reminders_queue`;
- one row per reminder decision;
- writes only columns that exist in the current sheet.

`Append payment_status_log`:

- sheet: `payment_status_log`;
- one row per processed invoice.

`Append audit_log`:

- sheet: `audit_log`;
- one row per processed invoice.

`Update invoices reminder metadata`:

- sheet: `invoices`;
- match on `row_number`;
- update `status`, `reminders_sent`, `last_reminder_at`, `days_overdue`, and `days_to_due`.

## Error Handling

| Error | Handling |
|---|---|
| Empty invoice set | `No invoice reminder work` branch |
| Missing invoice id or invalid due date | Manager review item |
| Duplicate same-day queue key | Skip, no write |
| Paid or closed invoice | Skip |
| Missing client email | Manager review item |
| Google Sheets append fails | Stop before source row update |
| Google Sheets update fails | Existing queue/status/audit rows show what happened |
| Google credential has no access to `invoice_reminders` | Share the spreadsheet with the Google account connected to n8n, or reconnect the credential to the account that owns the sheet |
| Spreadsheet locator is pasted as a literal title | Re-select `invoice_reminders` with `From list`; do not type the title into the locator value |

The workflow intentionally does not send customer email. This avoids accidental customer contact in a portfolio demo and keeps the review queue as the safe final output.

## Test Plan

Manual test:

1. Import `sample_invoices.csv` into `invoices`.
2. Confirm `reminders_queue`, `payment_status_log`, and `audit_log` headers exist.
3. Run `Manual Trigger`.
4. Compare results with `expected_outputs.json`.

Branch tests:

1. Open `TEST - Mock invoice input`.
2. Set `TEST_CASE` to `due_soon`.
3. Execute from the mock node and verify standard branch.
4. Set `TEST_CASE` to `escalation` or `invalid_contact`.
5. Verify manager review branch.
6. Set `TEST_CASE` to `paid` or `not_due`.
7. Verify no-work branch.

Expected outputs:

- due-soon invoice goes to `queued_for_review`;
- overdue invoice goes to `queued_for_review`;
- severe overdue invoice goes to `manager_review_required`;
- invalid contact invoice goes to `manager_review_required`;
- paid and not-due invoices are skipped.

## Production Notes

A production workflow can add:

- Schedule Trigger, for example every weekday morning;
- source-of-truth integration with accounting software;
- approved email send branch after finance review;
- unsubscribe/business-hours rules;
- bounce handling;
- error workflow;
- stronger locking for concurrent runs.

Do not activate a production schedule until the anti-duplicate-send check has been tested with repeated runs.
