# LeadFlow

**Production-style lead intake automation built with n8n, Google Sheets and Discord.**

LeadFlow receives website leads through an n8n webhook, validates and normalizes the payload, detects duplicates, stores valid leads, calculates a simple priority score, routes notifications to Discord, logs execution outcomes, and returns meaningful HTTP status codes.

## What it does

- receives website leads through a webhook
- validates and normalizes incoming data
- checks required fields and email format
- rejects malformed requests with HTTP `400`
- generates a unique lead ID for every request
- detects duplicate leads by email
- returns HTTP `409` for duplicate leads
- stores valid leads in Google Sheets
- calculates lead priority using urgency, budget and high-intent terms
- classifies leads as `HIGH` or `NORMAL`
- routes high-priority and normal leads to separate Discord channels
- retries failed Google Sheets and Discord operations
- logs successful, duplicate, invalid and Discord-error executions
- handles Google Sheets lookup/store errors separately
- returns HTTP `201`, `400`, `409`, `500` or `502` depending on the result

## Workflow overview

```text
Website
   |
   v
Webhook
   |
   v
Validate + Normalize
   |
   +---- invalid --------------------> Log Invalid -> HTTP 400
   |
   v
Find Duplicate by Email
   |
   +---- lookup error -----------------------------> HTTP 500
   |
   +---- duplicate -------------------> Log Duplicate -> HTTP 409
   |
   v
Store Lead
   |
   +---- store error ------------------------------> HTTP 500
   |
   v
Determine Priority
   |
   +---- HIGH ------> Discord #priority --+
   |                                      |
   +---- NORMAL ----> Discord #leads -----+
                                          |
                           +--------------+--------------+
                           |                             |
                        success                     Discord error
                           |                             |
                      Log Success                 Log Discord Error
                           |                             |
                       HTTP 201                       HTTP 502
```

## Lead scoring

The workflow uses a small rule-based scoring system:

| Rule | Score |
| --- | ---: |
| Urgency is `high`, `urgent`, `asap`, `sofort` or `dringend` | +3 |
| Budget is at least `5000` | +2 |
| Message/company contains a high-intent term | +2 |

High-intent terms currently include:

```text
angebot, proposal, demo, termin, meeting,
call, diese woche, this week, urgent, asap
```

A score of `3` or more is classified as `HIGH`. Lower scores are classified as `NORMAL`.

## Expected request

Send a `POST` request to the n8n webhook.

Required fields:

- `name`
- `email`
- `message`

Optional fields:

- `phone`
- `company`
- `source`
- `budget`
- `urgency`

Example payload:

```json
{
  "name": "Max Mustermann",
  "email": "max@example.com",
  "phone": "+49 170 1234567",
  "company": "Muster GmbH",
  "source": "website",
  "budget": 7500,
  "urgency": "high",
  "message": "We need a proposal and a call this week."
}
```

The same payload is available in [`examples/sample-request.json`](examples/sample-request.json).

## Google Sheets structure

Create one spreadsheet with two tabs.

### `Leads`

```text
receivedAt, leadId, name, email, phone, company, source, budget, urgency, message
```

### `Logs`

```text
timestamp, leadId, email, status, detail, priority
```

## Setup

1. Import `workflow.json` into n8n.
2. Create a Google Sheet with the `Leads` and `Logs` tabs and headers shown above.
3. Replace every `REPLACE_WITH_GOOGLE_SHEET_ID` value with your spreadsheet ID.
4. Select your own Google Sheets OAuth2 credential in every Google Sheets node.
5. Create/select one Discord webhook credential for the priority channel.
6. Create/select another Discord webhook credential for the normal leads channel.
7. Test the workflow using n8n's Webhook Test URL.
8. Activate/publish the workflow and use its Production URL for real requests.

## HTTP responses

| Status | Meaning |
| ---: | --- |
| `201` | Lead created and routed successfully |
| `400` | Request validation failed |
| `409` | Duplicate email already exists |
| `500` | Google Sheets lookup or storage failed |
| `502` | Discord notification failed |

## Security

The public `workflow.json` is sanitized for repository use:

- no Google Sheets credential IDs
- no Discord credential IDs
- no Discord webhook IDs
- no real Google Sheet ID
- no n8n instance ID
- no workflow ID or version ID

Credentials must be configured locally after importing the workflow.

## Tech stack

- n8n
- Google Sheets
- Discord webhooks
- HTTP webhooks
- JavaScript in n8n Code nodes

## Scope

This repository is intentionally limited to lead intake and routing. It demonstrates validation, deduplication, persistence, scoring, notification routing, retry/error handling, logging and HTTP response handling without attempting to become a full CRM.
