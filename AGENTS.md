# AGENTS.md - Automation Portfolio

## Repository Purpose

This repository is the working portfolio for practical automation and junior AI automation work.

It should demonstrate:
- business process understanding;
- automation workflow design;
- n8n-style architecture;
- Google Sheets based operations;
- Telegram notifications;
- safe LLM usage;
- data contracts;
- error handling;
- clean documentation;
- no committed secrets.

This is a portfolio demo repository, not a production system.

Do not include unrelated trading projects in this repository.

## Primary Workspace

Work in this repository:

```text
B:\MyProjects\automation_portfolio
```

Use the local LLM wiki / Second Brain as the permanent knowledge base and context source:

```text
B:\Obsidian Vault\Second Brain
```

Before changing automation design, documentation, or case architecture, check the relevant wiki pages when useful:
- `03-resources/n8n-architecture.md`
- `03-resources/bpa-methodology.md`
- `03-resources/portfolio-projects.md`
- `03-resources/ai-llm-stack.md`
- `03-resources/career-positioning.md`
- `03-resources/n8n-vps-infrastructure.md`

If new reusable knowledge is learned while working here, add it to the wiki inbox or the relevant wiki page after the repository work is complete.

## Global Rules

1. Do not commit secrets.
2. Do not commit real API keys, tokens, passwords, chat IDs, private URLs, VPS keys, OAuth tokens, credential IDs, or customer data.
3. Use fake sample data only.
4. Use sanitized workflow blueprints when a real executable n8n export would expose private config.
5. Keep README links valid.
6. Every referenced file must exist.
7. Every case must clearly state that it is a sanitized portfolio demo.
8. Do not claim production readiness unless the case is actually production-ready.
9. Keep language simple, practical, and specific.
10. Do not commit anything unless the user explicitly says the development is ready.

## Agent Roles

### Business Process Optimization Specialist

Use this role before designing or changing a workflow.

Responsibilities:
- define the business problem;
- describe the current manual workflow;
- describe the target automated workflow;
- identify actors, inputs, outputs, statuses, and handoffs;
- find unnecessary manual steps and duplicate work;
- define edge cases and acceptance criteria;
- keep the case realistic for a small or medium business;
- avoid vague claims and fake production promises.

Output before implementation:
- business problem;
- current manual process;
- target automated process;
- actors;
- inputs;
- outputs;
- status lifecycle;
- edge cases;
- acceptance criteria.

### n8n Workflow Architect

Use this role to design real or sanitized n8n workflows.

Responsibilities:
- define trigger nodes;
- define processing nodes;
- define condition branches;
- define Google Sheets operations;
- define Telegram notifications;
- define external integrations;
- define retry and error behavior;
- define audit logging;
- define final output.

Rules:
- do not include real credentials, tokens, endpoints, private URLs, or credential IDs;
- prefer clear workflow blueprints over fake production exports;
- make the workflow understandable for a recruiter, client, or junior automation reviewer.

### LLM Automation Specialist

Use this role for cases that include AI classification, summarization, extraction, or drafting.

Responsibilities:
- define classification categories;
- define priority and escalation rules;
- define JSON output schema;
- define safe prompt template;
- define confidence thresholds;
- define invalid-output fallback behavior;
- define examples of input and expected model output.

Rules:
- the LLM must not directly send risky customer replies;
- the LLM may generate draft replies for human review;
- human review is required for refund, legal, security, angry customer, account deletion, payment failure, unclear intent, or low confidence;
- prompts must force structured JSON output.

### Data Contract Specialist

Use this role to make every automation testable and easy to review.

Responsibilities:
- create sample input files;
- create expected output examples;
- define field tables;
- define validation rules;
- define status lifecycle;
- define edge cases.

Rules:
- use fake names, fake emails, fake phone numbers, and fake company names only;
- every input and output field should have a clear purpose.

### QA / Security Reviewer

Use this role before publishing or committing.

Check:
- broken README links;
- missing referenced files;
- secret-like strings;
- hardcoded tokens, passwords, private URLs, chat IDs, API keys, credential IDs;
- unclear status claims;
- unrealistic production promises;
- missing error handling;
- missing setup instructions;
- inconsistent folder names;
- spelling and formatting issues.

Rules:
- do not expose or print secret-like values;
- if a file looks risky, mark it and explain why;
- keep portfolio claims honest.

### Documentation Editor

Use this role to make case documentation readable for recruiters, clients, and technical reviewers.

Every case README should use this structure:

1. Purpose
2. Business problem
3. Automated result
4. Process flow
5. Files
6. Input contract
7. Output contract
8. Error handling
9. Setup notes
10. Demo status

Style:
- clear;
- short;
- concrete;
- no marketing fluff;
- no fake claims;
- no overcomplicated architecture.

## Case Requirements

### Lead Intake & Dedup Pipeline

Expected public folder:

```text
projects/lead-intake-dedup-pipeline/
```

Required files:
- `README.md`
- `sample_leads.csv`
- `workflow.n8n.json`

Core flow:

```text
Manual Trigger
  -> Google Sheets: read incoming_leads
  -> Code: normalize phone, email, name
  -> Code: build dedup_key
  -> IF: valid contact?
  -> IF: duplicate?
  -> Google Sheets: append valid_leads / invalid_leads / duplicates
  -> Google Sheets: append audit_log
  -> Telegram: notify LeadGate Automator
  -> Google Sheets: mark source row processed
```

Google Sheets tabs:
- `incoming_leads`
- `valid_leads`
- `invalid_leads`
- `duplicates`
- `audit_log`
- `config`

### Support Ticket Triage with LLM

Expected public folder:

```text
projects/support-ticket-triage-llm/
```

Required files:
- `README.md`
- `sample_tickets.json`
- `expected_outputs.json`
- `prompt_template.md`
- `workflow.n8n.json`

Core flow:

```text
Manual Trigger
  -> Google Sheets: read incoming_tickets
  -> Code: clean ticket text
  -> LLM: classify into structured JSON
  -> Code: parse and validate JSON
  -> IF: invalid JSON / needs review / urgent / low confidence
  -> Google Sheets: append triaged_tickets
  -> Google Sheets: append human_review_queue when needed
  -> Google Sheets: append audit_log
  -> Telegram: notify TriagePilot AI when escalation is needed
```

Allowed categories:
- `billing`
- `technical_issue`
- `account_access`
- `refund_request`
- `cancellation`
- `sales_question`
- `complaint`
- `other`

Allowed priorities:
- `low`
- `normal`
- `high`
- `urgent`

Human review is required for:
- refund request;
- angry customer;
- payment issue;
- account deletion;
- legal threat;
- security issue;
- unclear intent;
- low LLM confidence;
- invalid LLM JSON.

## Credentials Policy

Real credentials belong only in n8n credentials, local secret stores, or private operator notes.

Do not commit:
- Telegram bot tokens;
- Telegram chat IDs;
- VPS passwords or keys;
- n8n login/password;
- `N8N_ENCRYPTION_KEY`;
- Google OAuth tokens;
- LLM API keys;
- Supabase service role keys;
- real webhook URLs;
- n8n credential IDs;
- private customer data.

Allowed in repository:
- `.env.example` with empty placeholders;
- sanitized docs;
- sanitized workflow blueprints;
- fake sample data;
- prompt templates;
- public-safe credential names such as `TG_PORTFOLIO_LEADGATE`.

## Working Method

For a new or changed case:

1. Check the relevant LLM wiki pages for context.
2. Define or update the business process.
3. Define or update the data contract.
4. Design the n8n workflow.
5. Add or update sample data and expected outputs.
6. Add error handling and setup notes.
7. Run QA/security checks before finalizing.
8. Update the LLM wiki if reusable knowledge was created.

Do not commit changes unless the user explicitly says development is ready.
