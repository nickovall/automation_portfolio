# Automation Portfolio

Portfolio repository with three practical automation demo cases for an automation specialist / junior AI automation role.

The repository is not presented as production software. It is a public portfolio package that shows how business automation tasks can be decomposed, documented, and implemented with n8n-style workflows, simple data contracts, and safe environment configuration.

## Cases

| Case | Business problem | Main tools | Status |
|---|---|---|---|
| [Lead Intake & Dedup Pipeline](projects/lead-intake-dedup-pipeline/README.md) | Incoming leads arrive from several forms and messengers, duplicates create CRM noise | n8n, Webhook, CRM API, Google Sheets, Telegram | Portfolio demo |
| [Support Ticket Triage with LLM](projects/support-ticket-triage-llm/README.md) | Support requests need fast classification, priority detection, and routing | n8n, LLM API, Helpdesk API, Slack/Telegram | Portfolio demo |
| [Invoice & Payment Reminder Automation](projects/invoice-payment-reminder/README.md) | Unpaid invoices need scheduled reminders and manager summary reports | n8n, Email API, Google Sheets/Airtable, Telegram | Portfolio demo |

## Repository structure

```text
projects/
  lead-intake-dedup-pipeline/
    README.md
    workflow.n8n.json
    sample_leads.csv
  support-ticket-triage-llm/
    README.md
    workflow.n8n.json
    sample_tickets.json
  invoice-payment-reminder/
    README.md
    workflow.n8n.json
    sample_invoices.csv
.env.example
```

## What this portfolio demonstrates

- translating a vague business task into an automation flow;
- defining inputs, outputs, and failure handling;
- using environment variables instead of hardcoded secrets;
- keeping workflow exports sanitized before publishing;
- documenting setup steps so another person can review the solution.

## How to review

1. Open any case from the table above.
2. Read the business context and process flow.
3. Import the sanitized `workflow.n8n.json` into n8n.
4. Reconnect credentials manually.
5. Test with the provided sample data.

## Security notes

- No real API keys are stored in this repository.
- Workflow exports are simplified and sanitized for public sharing.
- `.env.example` contains variable names only.
- Real credentials must be configured inside n8n credentials or a private environment manager.

## Status

This repo is now usable as an early automation portfolio. The cases are demo-level and should be customized before offering them as production client work.
