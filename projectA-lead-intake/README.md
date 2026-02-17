# Lead Intake & Dedup Pipeline

This project automates lead intake and deduplication. It receives lead data from a web form,
normalizes and validates the fields, checks for duplicates in a Google Sheets or Notion table,
and notifies the relevant team via Slack or email.

## Key Features
- Webhook trigger to receive lead data.
- Data normalization and validation using JavaScript functions.
- Duplicate checking and record updating in a database (Google Sheets or Notion).
- Notification to team members with lead details.
- Comprehensive logging for troubleshooting.

## Usage
1. Configure a webhook node in n8n to receive HTTP POST requests from your web form.
2. Normalize and validate fields using JavaScript/Function nodes.
3. Use a database node to query existing leads and update or insert new ones.
4. Send notifications using Slack or email nodes.
5. Log details to a separate table for error tracking and auditing.
