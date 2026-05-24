# AI Ticket Classifier & Router with n8n + Gemini

A portfolio demo project for support automation built with **n8n** and **Google Gemini**.

The workflow receives a support ticket via webhook, normalizes the input, classifies the message with Gemini, validates the AI output with deterministic business rules, routes the ticket to the correct support queue, and returns a structured JSON response.

## What This Project Demonstrates

This project shows how AI can be used safely inside a support workflow.

Instead of blindly trusting the model output, the workflow:

- receives a support ticket through a webhook API;
- normalizes incoming ticket data;
- uses Gemini to classify the ticket;
- parses the AI response as JSON;
- validates category, intent, priority, sentiment, and confidence;
- applies deterministic business rules after AI classification;
- routes the ticket to the correct support queue and assignee group;
- returns a structured API response.

## Workflow Overview

```text
Incoming Ticket
→ Normalize Incoming Ticket
→ Gemini — Classify Ticket
→ Parse AI Classification
→ Validate Classification
→ Route Ticket
→ Respond to Webhook
```

## Main Features

- Webhook-based API input
- Gemini-powered ticket classification
- Strict JSON parsing
- AI output validation
- Deterministic business rules
- Priority correction
- Sentiment correction
- Human-review fallback logic
- Queue routing
- Structured JSON API response
- LLM-provider-agnostic design

## Use Case

This workflow simulates an internal support automation system.

A ticket can come from a CRM, helpdesk, email parser, messenger, form, or another system. The workflow classifies the customer message and decides where the ticket should be routed.

Example routing logic:

| Classification | Queue | Assignee group |
|---|---|---|
| Billing issue | Billing Support | billing_l1 / billing_l2 |
| Technical issue | Technical Support | tech_l1 / tech_l2 |
| Account access | Account Access | identity_support |
| Feature request | Product Feedback | product_ops |
| Complaint | Escalations | support_l2 |
| Low confidence / unknown | Human Review | support_triage |

## Repository Structure

```text
ai-ticket-classifier-router-n8n/
├── README.md
├── workflows/
│   └── ai-ticket-classifier-router.json
├── docs/
│   ├── workflow.png
│   ├── sample-response-billing.png
│   └── sample-response-login.png
└── examples/
    ├── billing-request.sh
    ├── login-issue.sh
    ├── bug-report.sh
    └── complaint.sh
```

## Screenshots

### Workflow

![Workflow](docs/workflow.png)

### Billing Ticket Response

![Billing response](docs/sample-response-billing.png)

### Account Access Ticket Response

![Login response](docs/sample-response-login.png)

## Input Example

```json
{
  "ticket_id": 3001,
  "customer_id": 7781,
  "customer_tier": "premium",
  "channel": "email",
  "message": "I was charged twice for my subscription. Please refund one of the payments.",
  "created_at": "2026-05-24T20:45:00Z"
}
```

## Output Example

```json
{
  "ticket_id": 3001,
  "customer_id": 7781,
  "customer_tier": "premium",
  "channel": "email",
  "message": "I was charged twice for my subscription. Please refund one of the payments.",
  "created_at": "2026-05-24T20:45:00Z",
  "classification": {
    "category": "billing",
    "intent": "duplicate_charge",
    "priority": "high",
    "sentiment": "negative",
    "language": "en",
    "needs_human": false,
    "confidence": 0.98,
    "summary": "Customer was charged twice for their subscription and is requesting a refund for the duplicate payment."
  },
  "routing": {
    "queue": "Billing Support",
    "assignee_group": "billing_l2",
    "routing_reason": "Billing-related ticket",
    "routed_at": "2026-05-24T17:23:19.742Z"
  }
}
```

## How to Import

1. Open n8n.
2. Import the workflow JSON from the `workflows` directory.
3. Configure Google Gemini credentials in n8n.
4. Open the node `Gemini — Classify Ticket`.
5. Select an available Gemini model manually.
6. Save the workflow.
7. Run the workflow in test mode.
8. Send a POST request to the webhook.

## Important Note About Gemini Credentials and Model Selection

The workflow export does **not** include Gemini credentials.

After importing the workflow, the Gemini node may show an empty model value because credentials and provider-specific settings are removed from the public export.

Before running the workflow:

1. Add your Google Gemini credentials in n8n.
2. Open the node `Gemini — Classify Ticket`.
3. Select an available Gemini model manually, for example:
   - `gemini-3-flash-preview`
   - `gemini-2.0-flash`
   - another Gemini model available in your n8n instance
4. Save the node.

If you use another LLM provider, you can replace the Gemini node with OpenAI, Groq, OpenRouter, Ollama, or another model provider, as long as the model returns the same JSON structure.

## Test Request

For local test webhook:

```bash
curl -s -X POST "http://localhost:5678/webhook-test/classify-ticket" \
  -H "Content-Type: application/json" \
  -d '{
    "ticket_id": 3001,
    "customer_id": 7781,
    "customer_tier": "premium",
    "channel": "email",
    "message": "I was charged twice for my subscription. Please refund one of the payments.",
    "created_at": "2026-05-24T20:45:00Z"
  }' | python3 -m json.tool
```

For production webhook after publishing the workflow:

```bash
curl -s -X POST "http://localhost:5678/webhook/classify-ticket" \
  -H "Content-Type: application/json" \
  -d '{
    "ticket_id": 3001,
    "customer_id": 7781,
    "customer_tier": "premium",
    "channel": "email",
    "message": "I was charged twice for my subscription. Please refund one of the payments.",
    "created_at": "2026-05-24T20:45:00Z"
  }' | python3 -m json.tool
```

## Additional Test Cases

### Account Access

```json
{
  "ticket_id": 3002,
  "customer_id": 7782,
  "customer_tier": "standard",
  "channel": "web",
  "message": "I cannot log into my account. Password reset does not work.",
  "created_at": "2026-05-24T21:10:00Z"
}
```

Expected result:

```text
category: account_access
intent: login_problem
priority: high
sentiment: negative
queue: Account Access
assignee_group: identity_support
```

### Technical Issue

```json
{
  "ticket_id": 3003,
  "customer_id": 7783,
  "customer_tier": "standard",
  "channel": "telegram",
  "message": "The dashboard crashes every time I open the reports page.",
  "created_at": "2026-05-24T21:11:00Z"
}
```

Expected result:

```text
category: technical_issue
intent: bug_report
queue: Technical Support
```

### Feature Request

```json
{
  "ticket_id": 3004,
  "customer_id": 7784,
  "customer_tier": "standard",
  "channel": "email",
  "message": "Can you add dark mode to the admin panel?",
  "created_at": "2026-05-24T21:12:00Z"
}
```

Expected result:

```text
category: feature_request
queue: Product Feedback
assignee_group: product_ops
```

### Complaint

```json
{
  "ticket_id": 3005,
  "customer_id": 7785,
  "customer_tier": "premium",
  "channel": "email",
  "message": "Your support ignored me for three days. This is unacceptable.",
  "created_at": "2026-05-24T21:13:00Z"
}
```

Expected result:

```text
category: complaint
intent: angry_complaint
priority: high
sentiment: negative
queue: Escalations
assignee_group: support_l2
```

## Workflow Logic

### 1. Incoming Ticket

Webhook node that accepts incoming support tickets.

Expected fields:

- `ticket_id`
- `customer_id`
- `customer_tier`
- `channel`
- `message`
- `created_at`

### 2. Normalize Incoming Ticket

Normalizes incoming request data and provides default values for missing fields.

Example defaults:

- `customer_tier`: `standard`
- `channel`: `unknown`
- `created_at`: current timestamp

### 3. Gemini — Classify Ticket

Uses Gemini to classify the support message.

The AI returns JSON with:

- `category`
- `intent`
- `priority`
- `sentiment`
- `language`
- `needs_human`
- `confidence`
- `summary`

### 4. Parse AI Classification

Extracts Gemini text response from:

```text
candidates[0].content.parts[0].text
```

Then parses it as JSON.

The parser also removes accidental Markdown code fences if the model returns them.

### 5. Validate Classification

Validates the AI result against allowed values.

Allowed categories:

```text
billing
technical_issue
account_access
feature_request
complaint
how_to_question
other
```

Allowed intents:

```text
refund_request
payment_failed
duplicate_charge
login_problem
bug_report
setup_question
cancel_subscription
angry_complaint
unknown
```

Allowed priorities:

```text
low
medium
high
urgent
```

Allowed sentiments:

```text
positive
neutral
negative
```

The validation step also applies deterministic business rules:

- billing refund / duplicate charge / payment failure → negative sentiment
- premium billing issues → at least high priority
- login problems → at least high priority
- complaints → high priority and negative sentiment
- low confidence / unknown intent / other category → human review

### 6. Route Ticket

Routes the ticket based on validated classification.

Example routing:

| Category | Queue | Assignee group |
|---|---|---|
| billing | Billing Support | billing_l1 / billing_l2 |
| technical_issue | Technical Support | tech_l1 / tech_l2 |
| account_access | Account Access | identity_support |
| feature_request | Product Feedback | product_ops |
| complaint | Escalations | support_l2 |
| other / low confidence | Human Review | support_triage |

### 7. Respond to Webhook

Returns the final structured JSON response with:

- original ticket data
- validated classification
- routing result

## Why Validation Matters

LLM output can be inconsistent.

For example, the model may classify a duplicate charge correctly as `billing`, but return `sentiment: neutral`.

The validation node corrects this using deterministic business rules.

This makes the workflow safer and more predictable for support operations.

## Possible Improvements

Future versions could add:

- Data Table logging
- CRM API update
- Slack / Telegram notifications
- fallback queue alerts
- confidence threshold configuration
- duplicate ticket detection
- category-specific SLA assignment
- human review dashboard
- multi-language support
- provider switch between Gemini / OpenAI / Groq / Ollama

## Tech Stack

- n8n
- Google Gemini
- Webhook API
- JavaScript Code nodes
- JSON validation
- Support routing logic

## Security Notes

This repository does not include:

- API keys
- Gemini credentials
- production webhook URLs
- private customer data
- real CRM data

Before publishing a workflow export, check that no secrets are included.

## Status

Portfolio MVP completed.

The workflow is designed as a demo project for support automation, CRM operations, and AI-assisted ticket routing.
