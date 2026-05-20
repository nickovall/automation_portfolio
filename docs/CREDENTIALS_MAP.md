# Credentials Map

Real tokens and secrets are stored only inside n8n credentials, private environment files, or the operator's private notes.

Do not commit token values, credential IDs, private URLs, chat IDs, API keys, OAuth tokens, VPS credentials, or customer data.

## Telegram

| Case | Bot / channel role | n8n credential name | Purpose |
|---|---|---|---|
| Lead Intake & Dedup Pipeline | LeadGate Automator | `TG_PORTFOLIO_LEADGATE` | New lead alerts, duplicate alerts, invalid lead alerts |
| Support Ticket Triage with LLM | TriagePilot AI | `TG_PORTFOLIO_TRIAGEPILOT` | Human review alerts, urgent ticket escalation, LLM triage summaries |

## Google Sheets

| Case | Spreadsheet | n8n credential name | Purpose |
|---|---|---|---|
| Lead Intake & Dedup Pipeline | `lead_intake` | `GSHEETS_PORTFOLIO_AUTOMATION` or the existing Google Sheets credential | Read incoming leads, write outputs, write audit log |
| Support Ticket Triage with LLM | `support_triage` | `GSHEETS_PORTFOLIO_AUTOMATION` or the existing Google Sheets credential | Read tickets, write triage outputs, write review queue, write audit log |
| Invoice Reminder Automation | `invoice_reminders` | `GSHEETS_PORTFOLIO_AUTOMATION` or the existing Google Sheets credential | Read invoices, write reminder queue, write payment status log, write audit log |

The Google account connected in n8n must have access to each spreadsheet. Public exports may use placeholder locator values, but live n8n nodes should be selected through `From list` so n8n stores its internal spreadsheet locator.

## LLM

| Case | Provider | n8n credential name | Purpose |
|---|---|---|---|
| Support Ticket Triage with LLM | DeepSeek | `DeepSeek account` | Classify support tickets into structured JSON and generate draft replies for human review |

## Email

| Case | n8n credential name | Purpose |
|---|---|---|
| Invoice Reminder Automation | `<EMAIL_CREDENTIAL_NAME>` | Optional reviewed email-send step after finance approval. The public demo does not send customer email directly. |

## Public Export Rules

- Workflow JSON may include credential names because they help reviewers understand setup.
- Workflow JSON must not include real credential IDs; use `CRED_ID` in public exports.
- Workflow JSON must not include real API keys, chat IDs, sheet IDs, webhook URLs, or private hostnames.
- `.env.example` must contain placeholders or public-safe names only.
- n8n workflows should stay inactive until credentials, sheets, and test data are reviewed manually.
