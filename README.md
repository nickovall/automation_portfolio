# Automation Portfolio

Portfolio repository with practical automation demo cases for an automation specialist / junior AI automation role.

The repository is not presented as production software. It is a public portfolio package that shows how business automation tasks can be decomposed, documented, and implemented with n8n-style workflows, simple data contracts, and safe environment configuration.

## Cases

| Case | Business problem | Main tools | Status |
|---|---|---|---|
| [Lead Intake & Dedup Pipeline](projects/lead-intake-dedup-pipeline/README.md) | Incoming leads arrive from several forms and messengers, duplicates create sales noise | n8n, Google Sheets, Telegram | Portfolio demo |
| [Support Ticket Triage with LLM](projects/support-ticket-triage-llm/README.md) | Support requests need fast classification, priority detection, and routing | n8n, Google Sheets, LLM, Telegram | Portfolio demo |
| [Invoice Reminder Automation](projects/invoice-reminder-automation/README.md) | Unpaid invoices need consistent follow-up without duplicate customer reminders | n8n, Google Sheets | Portfolio demo |

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
    expected_outputs.json
    prompt_template.md
  invoice-reminder-automation/
    README.md
    workflow.n8n.json
    sample_invoices.csv
    expected_outputs.json
docs/
  CREDENTIALS_MAP.md
  lead-intake-n8n-implementation.md
  support-ticket-triage-n8n-implementation.md
  invoice-reminder-n8n-implementation.md
.env.example
.gitignore
AGENTS.md
README.md
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

## Implementation docs

| Document | Purpose |
|---|---|
| [Lead Intake n8n Implementation](docs/lead-intake-n8n-implementation.md) | Node-by-node build guide for the Google Sheets and Telegram workflow |
| [Support Ticket Triage n8n Implementation](docs/support-ticket-triage-n8n-implementation.md) | Node-by-node build guide for the Google Sheets, DeepSeek, review queue, no-work branch, and Telegram workflow |
| [Invoice Reminder n8n Implementation](docs/invoice-reminder-n8n-implementation.md) | Node-by-node build guide for the Google Sheets invoice reminder queue workflow |
| [Credentials Map](docs/CREDENTIALS_MAP.md) | Public-safe map of credential names and where real secrets belong |

## Security notes

- No real API keys are stored in this repository.
- Workflow exports are simplified and sanitized for public sharing.
- `.env.example` contains variable names only.
- Real credentials must be configured inside n8n credentials or a private environment manager.

## Status

This repo is now usable as an early automation portfolio. The published cases are demo-level and should be customized before using them for real client workflows.
