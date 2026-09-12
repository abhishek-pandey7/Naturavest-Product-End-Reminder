# Naturavest Product End Reminder

Two n8n workflows that track how long a purchased product will last and send the customer two WhatsApp reminders: one shortly before the product runs out, and one shortly after, prompting a restock.

The system is built on Shopify (webhooks, metafields, tags), n8n (orchestration) and Messaging.digital (WhatsApp Business API).

## How it works

```
[Shopify Checkout]
       | (Webhook: orders/create)
       v
[Workflow 1a: Calculate Date & Save to Shopify]
       |
       +--> Fetch product metafield custom.lifespan_days (REST)
       +--> totalLifespan = baseLifespan * quantity * packMultiplier
       +--> date1 = today + totalLifespan - 2   (pre-runout)
       +--> date2 = today + totalLifespan + 2   (post-runout)
       +--> Save order metafields reminder_date_1 and reminder_date_2 (REST)
       +--> Tag order reminder1_YYYY-MM-DD and reminder2_YYYY-MM-DD (GraphQL)

[Daily cron, 08:00]
       |
       v
[Workflow 1b: Send WhatsApp & Tag Order]
       |
       +--> Fetch orders tagged reminder1_today or reminder2_today (GraphQL, up to 50)
       +--> Split out into individual items
       +--> Switch on tag
       |      +--> reminder1: send template 'purchase_reminder' (text)
       |      +--> reminder2: send template 'restock_reminder_post' (image + reorder button)
       +--> Remove both reminder tags from the order (GraphQL)
```

### Workflow 1a: Order intake and scheduling

Triggered in real time by the Shopify `orders/create` webhook.

1. Fetch the first line item's product metafields via REST.
2. Read `custom.lifespan_days` (defaults to 30 if missing).
3. Parse the variant title for a pack multiplier, for example "Pack of 3".
4. Compute `totalLifespan = baseLifespan * quantity * packMultiplier`.
5. Write `custom.reminder_date_1` (runout minus 2 days) and `custom.reminder_date_2` (runout plus 2 days) to the order.
6. Tag the order with `reminder1_{date1}` and `reminder2_{date2}`.

### Workflow 1b: Daily dispatch and cleanup

Runs once a day at 08:00.

1. Query Shopify GraphQL for orders tagged `reminder1_{today}` or `reminder2_{today}`.
2. Process each order independently.
3. Route by tag:
   - `reminder1`: send the `purchase_reminder` template with first name and product title.
   - `reminder2`: send the `restock_reminder_post` template with the product image as header, first name, product title, and the product handle as the button URL for quick reordering.
4. Remove both tags so the customer is not messaged again for this order.

## Repository contents

| File | Purpose |
| --- | --- |
| `Naturavest - Product End Reminder - 1a (Calculate Date & Save to Shopify).json` | n8n export of the intake workflow |
| `Naturavest - Product End Reminder - 1b (Send WhatsApp & Tag Order).json` | n8n export of the daily dispatch workflow |
| `architecture.md` | Component architecture with Mermaid sequence diagrams and the full data schema |
| `system_design_reliability.md` | Tech stack rationale, failure modes, diagnostics and optimisations |

## Setup

### 1. Shopify

Create the following metafield definitions:

| Owner | Namespace.key | Type | Purpose |
| --- | --- | --- | --- |
| Product | `custom.lifespan_days` | Integer | Days a single unit of the product lasts |
| Order | `custom.reminder_date_1` | Date | Scheduled pre-runout reminder date |
| Order | `custom.reminder_date_2` | Date | Scheduled post-runout reminder date |

Populate `custom.lifespan_days` on every product. Products without it fall back to 30 days.

Order tags use the formats `reminder1_YYYY-MM-DD` and `reminder2_YYYY-MM-DD`. They exist purely for fast indexed GraphQL lookups in workflow 1b and are removed after sending.

### 2. WhatsApp templates (Messaging.digital)

Approve two templates in your Messaging.digital account:

| Template | Header | Body params | Button |
| --- | --- | --- | --- |
| `purchase_reminder` | none | `{{1}}` first name, `{{2}}` product title | none |
| `restock_reminder_post` | Image (product featured image) | `{{1}}` first name, `{{2}}` product title without variant suffix | URL with product handle |

### 3. n8n

1. Import both JSON files into your n8n instance.
2. Attach credentials: Shopify Admin API access token (REST and GraphQL, API version `2026-01`) and the Messaging.digital header auth.
3. Activate workflow 1a. n8n registers the `orders/create` webhook with Shopify automatically.
4. Activate workflow 1b. Adjust the schedule trigger if 08:00 in the instance timezone is not what you want.

## Operations

### Failure modes to watch

| Failure | Impact | Mitigation | Where to look |
| --- | --- | --- | --- |
| Webhook not delivered (n8n down, timeout) | Order never scheduled, no reminders | Shopify retries 19 times over 48 hours | Shopify Admin > Settings > Notifications > Webhooks; n8n execution history |
| Product missing `lifespan_days` | Would crash the code node | Falls back to 30 days | JS node output shows base lifespan exactly 30 |
| WhatsApp API down or rate limited | Message not sent | Enable retry on the HTTP Request node (3 retries, exponential backoff) | Non-200 responses from `messagingapi.charteredinfo.com` in 1b logs |
| Tag removal fails after a successful send | **Duplicate message** on the next daily run | Enable retry on the untag node; optionally log sent order IDs externally as an idempotency check | Customer reports of duplicates; Shopify order timeline |

### Design notes

- **Decoupled phases.** The checkout webhook is accepted immediately and only writes metadata. Message sending is deferred to the off-peak daily cron, so nothing in the checkout path waits on the WhatsApp API.
- **GraphQL batching.** Workflow 1b pulls up to 50 due orders with customer, line item, handle and image data in a single request, which keeps Shopify rate-limit usage low.
- **Deterministic date maths.** All lifespan and date calculations live in a plain JavaScript code node. Regex parsing of pack sizes, integer coercion and default handling are easier to read and test there than in n8n expression syntax.
- **Phone sanitisation.** Numbers are stripped to digits before calling the API, and `order.phone` falls back to `customer.phone`.

## Further reading

- [architecture.md](architecture.md) for sequence diagrams of both workflows and the complete API and schema definitions.
- [system_design_reliability.md](system_design_reliability.md) for tech stack rationale, a deeper failure analysis and guardrails to apply if LLM-generated copy is ever introduced.
