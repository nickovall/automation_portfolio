# Support Ticket Triage with LLM

This workflow classifies incoming support tickets using a large language model (LLM) and routes them
based on category and priority. It generates summaries and draft responses while logging all data
for quality improvement.

## Key Features
- Webhook trigger for incoming tickets.
- Integration with an LLM API (e.g., OpenAI) for categorization and priority assignment.
- Routing logic to send tickets to appropriate channels (Slack or separate tables).
- Error handling and fallback category assignment.
- Detailed logging of raw and processed ticket data.

## Usage
1. Set up a webhook node to receive ticket data.
2. Call the LLM API with instructions to output JSON fields (category, priority, summary).
3. Use conditional logic to route tickets according to category/priority.
4. Generate draft replies and summaries.
5. Log each processed ticket for later analysis and prompt refinement.
