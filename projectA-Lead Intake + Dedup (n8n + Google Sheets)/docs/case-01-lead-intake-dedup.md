# Case 01 — Lead Intake + Dedup

## Business context

Small B2B teams often collect leads from multiple sources: forms, landing pages, manual entries, and chat requests. Without automation, duplicate records appear, lead history becomes inconsistent, and ops teams lose time cleaning the data.

## Goal

Build a low-code intake workflow that:

- accepts inbound leads via webhook
- normalizes incoming fields
- rejects incomplete payloads
- checks for existing leads by email
- updates the canonical record or creates a new one
- logs every execution in a dedicated audit sheet

## Outcome

The workflow acts like a lightweight intake layer on top of Google Sheets and shows how no-code automation can support basic CRM-style operations.

## Workflow behavior

### Validation
The flow checks whether `Email` is present. If not:

- it skips lookup and write operations
- records a reject event in `Automation_Logs`
- returns a structured error response

### Lookup and dedup
For valid leads:

- `GS - Leads Lookup` searches by `Email`
- `Merge` enriches the input with matching rows
- `Dedup Decision` determines:
  - `matchCount`
  - `duplicateRows`
  - `action = create | update`
  - canonical row by latest `Last Updated`

### Write path
- `create` -> append a new row to `Leads`
- `update` -> update the canonical row in `Leads`

### Logging
Every branch writes to `Automation_Logs`, including:

- run ID
- step name
- result
- duration
- duplicate metrics

## Why this is portfolio-worthy

This case shows more than basic automation. It includes:

- validation logic
- deterministic deduplication
- conditional branching
- operational audit logging
- API-style response design

## Demo evidence to include

- workflow overview screenshot
- updated lead row
- created lead row
- reject log example
- duplicate metrics in logs
