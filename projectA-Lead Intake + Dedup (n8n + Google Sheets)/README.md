# Lead Intake + Dedup (n8n + Google Sheets)

Webhook-based lead intake workflow for a small B2B ops / automation scenario.

## Why this case matters

This project is not just a form-to-sheet integration. It demonstrates a realistic low-code operations workflow:

- inbound lead intake via webhook
- field normalization
- validation path for missing email
- duplicate detection by `Email`
- canonical record selection
- create/update branching
- structured audit logging in Google Sheets
- API-style JSON responses

## Stack

- n8n Cloud
- Google Sheets
- Webhooks
- IF / Merge / Code / Respond to Webhook nodes
- JavaScript in n8n Code node

## Sheets used as a mini ops system

### Leads
- Lead ID
- Name
- Email
- Phone
- Company
- Role
- Source
- Status
- Interest
- Created At
- Last Updated
- Notes

### Automation_Logs
- Timestamp
- Run ID
- Workflow
- Step
- Result
- Error Message
- Input Ref
- Output
- Duration (sec)
- Match Count
- Duplicate Rows

## Workflow summary

1. **Webhook** receives a lead payload.
2. **Edit Fields** normalizes the data:
   - lowercases email
   - keeps phone as text
   - applies default values for `Status` and `Source`
   - adds service fields (`CorrelationId`, `StartMs`, `ReceivedAt`)
3. **IF - Email Present** validates that email exists.
4. **Reject branch**:
   - writes a reject log entry
   - responds with a structured error JSON
5. **Lookup branch**:
   - searches `Leads` by normalized `Email`
   - merges matching rows with the input
   - uses a Code node to:
     - count matches
     - choose the canonical row by latest `Last Updated`
     - decide between `create` and `update`
6. **Create / Update branch**:
   - appends a new row or updates an existing row
7. **Logging branch**:
   - writes structured records to `Automation_Logs`
   - stores duration and duplicate metrics
8. **Respond to Webhook** returns a JSON result to the caller.

## Architecture

```text
Webhook
  -> Edit Fields
  -> IF - Email Present
      -> false:
         Log - Reject Missing Email
         -> GS - Logs Reject
         -> Respond - Reject

      -> true:
         -> GS - Leads Lookup
         -> Merge
         -> Dedup Decision
         -> IF - Create or Update
             -> true:
                GS - Leads Create
                -> Log - Create
                -> GS - Logs Create
                -> Respond to Webhook

             -> false:
                GS - Leads Update
                -> Logs - Update
                -> GS - Logs Update
                -> Respond to Webhook
```

## Deduplication logic

Primary key: **Email**

Rules:
- no matches -> `create`
- one match -> `update`
- multiple matches -> choose the row with the latest `Last Updated`

The workflow also logs:
- `Match Count`
- `Duplicate Rows`

This makes duplicate handling visible in audit logs.

## Test scenarios

### 1. Update existing lead
Use an email that already exists in `Leads`.

Expected:
- canonical row is updated
- log step = `update_lead`
- response includes `action = update`

### 2. Create new lead
Use an email that does not exist in `Leads`.

Expected:
- new row is appended
- log step = `create_lead`
- response includes `action = create`

### 3. Reject missing email
Send payload without `email`.

Expected:
- no write to `Leads`
- reject log step = `reject_missing_email`
- response returns a structured error JSON

## Example payloads

### Update payload
```json
{
  "name": "Sergey Ivanov",
  "email": "sergey.ivanov@biz.test",
  "phone": "+7 (999) 111-22-33",
  "company": "ZenFit Gym",
  "role": "Operations Manager",
  "source": "website",
  "status": "new",
  "interest": "Invoice reminders",
  "notes": "Please contact via email."
}
```

### Create payload
```json
{
  "name": "Anna Petrova",
  "email": "anna.petrova@biz.test",
  "phone": "+7 (999) 222-33-44",
  "company": "Orbit Studio",
  "role": "Founder",
  "source": "website",
  "status": "new",
  "interest": "Lead intake automation",
  "notes": "Interested in quick demo."
}
```

### Missing email payload
```json
{
  "name": "No Email Lead",
  "phone": "+7 (999) 000-00-00",
  "company": "Test Co",
  "role": "Ops",
  "source": "website",
  "status": "new",
  "interest": "Lead intake",
  "notes": "Missing email on purpose."
}
```

## PowerShell test example

```powershell
$body = @{
  name="Anna Petrova"; email="anna.petrova@biz.test"; phone="+7 (999) 222-33-44";
  company="Orbit Studio"; role="Founder"; source="website"; status="new";
  interest="Lead intake automation"; notes="Interested in quick demo."
} | ConvertTo-Json

Invoke-RestMethod -Method Post -Uri "<WEBHOOK_URL>" `
  -ContentType "application/json" `
  -Headers @{ "X-Correlation-Id"="demo-002" } `
  -Body $body -TimeoutSec 15
```

## What to show in an interview

- workflow canvas with validation, dedup, branching, and logging
- existing lead update example
- new lead create example
- reject example for missing email
- audit log entries with duplicate metrics

## Repo structure

```text
/workflows
  lead-intake-dedup.json

/docs
  case-01-lead-intake-dedup.md
  testing.md

/assets
  01-workflow-overview.png
  02-automation-logs.png
  03-leads-create.png
  04-leads-update.png
```

## Skills demonstrated

- webhook intake
- payload normalization
- Google Sheets CRUD
- deduplication logic
- branching with IF
- Merge + Code node usage
- operational logging
- validation / reject-path design

## Future improvements

- validate email format, not only presence
- add a dedicated Error Trigger workflow for technical failures
- notify Slack/Telegram on write failures
- replace Google Sheets with a database for larger volumes
