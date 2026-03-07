# Testing Guide

## 1. Update existing lead

Payload:

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

Expected:
- existing row updated
- `update_lead` log created
- `matchCount` > 0

## 2. Create new lead

Payload:

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

Expected:
- new row added to `Leads`
- `create_lead` log created
- `matchCount` = 0

## 3. Reject missing email

Payload:

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

Expected:
- no write to `Leads`
- `reject_missing_email` appears in `Automation_Logs`
- structured error response

## PowerShell example

```powershell
$body = @{
  name="No Email Lead"; phone="+7 (999) 000-00-00";
  company="Test Co"; role="Ops"; source="website"; status="new";
  interest="Lead intake"; notes="Missing email on purpose."
} | ConvertTo-Json

Invoke-RestMethod -Method Post -Uri "<WEBHOOK_URL>" `
  -ContentType "application/json" `
  -Headers @{ "X-Correlation-Id"="demo-no-email" } `
  -Body $body -TimeoutSec 15
```

## Troubleshooting

### Phone becomes `#ERROR!`
Make sure the value is written as plain text and Google Sheets is not interpreting it as a formula.

### Duplicate leads still appear
Check:
- email normalization
- lookup column = `Email`
- dedup decision uses latest `Last Updated`

### Missing logs
Check:
- each branch is connected to its log nodes
- `Automation_Logs` column names match the Google Sheets node mapping
