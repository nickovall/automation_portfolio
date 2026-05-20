# Invoice Reminder Automation

## Purpose

This case shows a payment reminder workflow for a small service business. The automation checks invoice due dates, prepares reminder messages, avoids duplicate reminders, writes an audit trail, and routes risky cases to manager review.

## Business problem

Small teams often track invoices in spreadsheets and send reminders manually. This creates missed follow-ups, duplicate messages, and unclear payment status history.

Current manual process:

1. Finance owner opens the invoice tracker.
2. Owner checks which invoices are due soon or overdue.
3. Owner writes a reminder message manually.
4. Owner tries to remember whether the customer was already contacted today.
5. Old overdue or invalid-contact cases are escalated informally.

## Automated result

The workflow reads unpaid invoices from Google Sheets, decides whether a reminder is needed, writes a prepared reminder to `reminders_queue`, logs the decision, and updates invoice reminder metadata. It does not send customer emails directly in the public demo.

Actors:

- finance owner: reviews reminder queue and escalation cases;
- automation: checks due dates, reminder stage, duplicate-send risk, and logs;
- customer: receives a reminder only after the finance owner approves or after an approved email-send node is added;
- Google Sheets: stores invoices, reminder queue, status log, and audit history.

## Process flow

```text
Manual Trigger
  -> read invoice_reminders / reminders_queue
  -> seed anti-duplicate send context
  -> read invoice_reminders / invoices
  -> prepare invoice reminder batch
  -> IF invoices need reminders
     false -> No invoice reminder work
     true  -> IF manager review required
              true  -> append reminders_queue as manager_review_required
              false -> append reminders_queue as queued_for_review
  -> append payment_status_log
  -> append audit_log
  -> update invoices reminder metadata
```

The n8n workflow also includes `TEST - Mock invoice input`. This node is not on the normal `Manual Trigger` path. It is used only for branch testing by changing `TEST_CASE` to `due_soon`, `overdue`, `escalation`, `invalid_contact`, `paid`, or `not_due`.

Acceptance criteria:

- paid, cancelled, void, and refunded invoices are skipped;
- due-soon and overdue invoices produce reminder queue rows;
- old overdue, high reminder count, missing email, or missing payment link cases require manager review;
- the same invoice/reminder type/date cannot be queued twice;
- every processed invoice creates payment status and audit log rows;
- workflow files contain no real credentials, sheet IDs, customer data, or private URLs.

## Files

| File | Description |
|---|---|
| `sample_invoices.csv` | Fake input invoices used for testing |
| `expected_outputs.json` | Expected queue and status results for the sample invoices |
| `workflow.n8n.json` | Sanitized n8n workflow export for Google Sheets reminder queue demo |

## Input contract

| Field | Type | Required | Notes |
|---|---:|---:|---|
| `invoice_id` | string | yes | Unique invoice id |
| `client_name` | string | yes | Fake sample name only |
| `client_email` | string | conditional | Required before customer reminder can be sent |
| `amount` | number | yes | Invoice amount |
| `currency` | string | yes | Example: `USD` or `KRW` |
| `issue_date` | string | yes | ISO date |
| `due_date` | string | yes | ISO date |
| `status` | string | yes | `sent`, `overdue`, `paid`, `cancelled`, `void`, or `refunded` |
| `reminders_sent` | number | no | Number of previous reminders |
| `last_reminder_at` | string | no | Last reminder date |
| `paid_at` | string | no | Payment date when already paid |
| `days_overdue` | number | no | Filled by the workflow |
| `days_to_due` | number | no | Filled by the workflow |
| `processed` | string/boolean | no | Demo tracking flag from the source sheet |

Validation rules:

- `invoice_id`, `due_date`, `amount`, and `currency` must exist.
- Customer reminders need a valid-looking `client_email`.
- Paid or closed invoices are skipped.
- If `last_reminder_at` is today, the invoice is skipped to prevent duplicate customer contact.
- Existing `reminders_queue` rows are used to build an anti-duplicate key from `invoice_id`, `reminder_type`, and queue date.

## Output contract

Reminder queue fields:

| Field | Type | Notes |
|---|---:|---|
| `created_at` | string | Workflow processing timestamp |
| `invoice_id` | string | Source invoice id |
| `client_name` | string | Customer name from `invoices` |
| `client_email` | string | Customer email from `invoices` |
| `reminder_type` | string | `due_soon`, `overdue`, `manager_review`, or `invalid_contact` |
| `message` | string | Draft reminder or manager review note |
| `status` | string | `queued_for_review` or `manager_review_required` |
| `sent_at` | string | Blank in the public demo |
| `telegram_notified` | string | `FALSE` in the public demo |

Output sheets:

| Sheet | Purpose |
|---|---|
| `reminders_queue` | Prepared reminder drafts and manager review items |
| `payment_status_log` | Invoice reminder status history |
| `audit_log` | Workflow event log and run summary |

Status lifecycle:

```text
issued -> due_soon -> reminded -> paid
issued -> overdue -> reminded -> paid
overdue -> escalated -> paid
issued -> review_required -> corrected -> reminded
```

## Error handling

| Error | Handling |
|---|---|
| No invoices need reminders | End in `No invoice reminder work`; do not write queue rows |
| Missing invoice id or invalid due date | Route to manager review and log validation error |
| Missing customer email | Queue as `manager_review_required`; do not allow customer send |
| Reminder already queued today | Skip invoice to avoid duplicate-send |
| Paid or closed invoice | Skip invoice |
| Google Sheets write failure | Stop before updating invoice metadata |
| Partial downstream failure | Audit and payment status logs remain the source of truth for what completed |

Edge cases:

- invoice is overdue but was already reminded today;
- customer email is missing;
- invoice is more than 14 days overdue;
- reminder count is already 3 or higher;
- invoice was paid after it was queued but before review;
- workflow is restarted after a partial run.

## Setup notes

1. Import `workflow.n8n.json` into n8n.
2. Reconnect Google Sheets credentials manually.
3. Select the Google Sheet named `invoice_reminders` in each Google Sheets node using `From list`.
4. Keep sheet tabs bound by name: `invoices`, `reminders_queue`, `payment_status_log`, `audit_log`.
5. Import `sample_invoices.csv` into `invoices`.
6. Run manually first and compare with `expected_outputs.json`.
7. Review `reminders_queue` before sending any real customer email.
8. Add a Schedule Trigger only after the manual run is verified.

n8n stores an internal spreadsheet locator when `From list` is selected. The public workflow keeps this value as `INVOICE_REMINDER_SPREADSHEET_LOCATOR`; do not replace it with a real Sheet ID in the repository. The Google Sheets credential used in n8n must have access to the `invoice_reminders` spreadsheet before the workflow can run.

Branch testing:

1. Open `TEST - Mock invoice input`.
2. Change `const TEST_CASE = 'due_soon';` to `overdue`, `escalation`, `invalid_contact`, `paid`, or `not_due`.
3. Change the mock `row_number` if row 2 is not a disposable demo row.
4. Execute the mock node and continue from `Prepare invoice reminder batch`.
5. Use only fake/disposable demo rows because downstream nodes still write to Google Sheets.

Production adaptation:

The portfolio workflow writes reminder drafts to a queue instead of sending email directly. In a production version, add an approved email send branch after review, plus unsubscribe/business-hours rules, bounce handling, and a stronger source-of-truth system such as accounting software or a CRM.

## Demo status

Portfolio demo. This is a sanitized, inactive workflow demo, not production-ready billing software. It demonstrates process design, reminder cadence, anti-duplicate-send logic, review routing, data contracts, and audit logging.
