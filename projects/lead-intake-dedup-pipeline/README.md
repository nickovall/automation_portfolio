# Lead Intake & Dedup Pipeline

## Purpose

This case shows a simple lead intake automation for a small service business. Leads arrive from forms, messengers, ads, and manual manager input. The automation validates incoming data, detects duplicates, creates or updates a CRM record, and notifies the sales team.

## Business problem

Without automation, the team gets duplicate leads, incomplete contacts, late responses, and no clear source attribution. Managers waste time checking the same person across sheets, CRM, and chats.

## Result

The workflow turns raw lead submissions into clean CRM-ready records.

Expected output:

- normalized phone/email/name fields;
- duplicate detection by phone and email;
- CRM create/update request;
- Google Sheets log row;
- Telegram notification for a new qualified lead;
- error branch for invalid contacts.

## Process flow

```text
Webhook / CSV import
  -> normalize fields
  -> validate phone or email
  -> dedup by phone/email
  -> CRM create/update
  -> Google Sheets audit log
  -> Telegram sales notification
```

## Files

| File | Description |
|---|---|
| `workflow.n8n.json` | Sanitized n8n-style workflow export |
| `sample_leads.csv` | Test input data with valid leads, duplicates, and invalid rows |

## Input contract

| Field | Type | Required | Notes |
|---|---:|---:|---|
| `created_at` | string | yes | ISO-like date/time from source |
| `source` | string | yes | Form, Telegram, ads, manual import |
| `name` | string | no | Can be empty for raw leads |
| `phone` | string | conditional | Required if email is empty |
| `email` | string | conditional | Required if phone is empty |
| `message` | string | no | User request text |
| `utm_source` | string | no | Marketing attribution |

## Dedup logic

Priority:

1. Exact phone match after normalization.
2. Exact lowercase email match.
3. Same phone + different email updates the existing CRM lead.
4. Same email + different phone updates the existing CRM lead.
5. No phone and no email goes to the invalid branch.

## Error handling

| Error | Handling |
|---|---|
| Missing contact data | Send to invalid branch and log row |
| CRM API 4xx | Log request body without secrets and notify manager |
| CRM API 5xx / timeout | Retry 3 times, then send error notification |
| Duplicate found | Update existing lead instead of creating a new one |

## Setup notes

1. Import `workflow.n8n.json` into n8n.
2. Reconnect CRM, Google Sheets, and Telegram credentials manually.
3. Copy `.env.example` from repo root and fill local values privately.
4. Test with `sample_leads.csv`.
5. Replace mock CRM endpoints with real endpoints before production use.

## Demo status

Portfolio demo. The workflow is intentionally sanitized. It shows process design and implementation logic, not a production-ready CRM integration.
