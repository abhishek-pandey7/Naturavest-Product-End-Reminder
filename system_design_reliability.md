# System Design, Reliability, and Operations Guide

This document outlines the detailed system design, tech stack decisions, failure analysis, optimizations, and operational diagnostics for the **Naturavest WhatsApp Automation System**.

---

## 1. Complete Architecture & Tech Stack Rationale

### The Tech Stack
* **n8n (Workflow Automation Platform):** Chosen as the core orchestration engine. n8n provides visual debugging, automatic retries, native credential management (OAuth2/Header Auth), and robust scheduling out of the box, avoiding the need to maintain custom server infrastructure (e.g., Node.js/Express, cron daemons, db queues).
* **Shopify REST & GraphQL APIs:** 
  * **REST API** is used to fetch specific product metafields and save order metafields.
  * **GraphQL API** is used for batch querying orders via tag filtering (`tag:reminder1_YYYY-MM-DD` or `tag:reminder2_YYYY-MM-DD`) and performing mutations (adding/removing tags). GraphQL is chosen for its efficiency, allowing us to retrieve multiple orders, line items, handles, images, and customer data in a single payload, reducing Shopify API rate-limit consumption.
* **JavaScript (n8n Code Node):** Used for custom date and pack size calculations. Standardizing on JS allows for clean regex matching (e.g., parsing variant packs like "Pack of 3"), type checking (e.g., parsing integers), and fallback default handling (e.g., default 30-day lifespan) that is easily readable and maintainable compared to complex n8n expression syntax.
* **Messaging.digital (WhatsApp Business API Gateway):** Utilized for delivering WhatsApp template notifications. Templates ensure compliance with Meta's business policies, avoiding spam flags and ensuring high deliverability.

### End-to-End Pipeline
```
[Shopify Checkout] 
       │ (Webhook: orders/create)
       ▼
[n8n Workflow Part A]
       │
       ├──► Fetch Product Metafields (REST)
       ├──► JS: Calculate totalLifespan = baseLifespan * quantity * packMultiplier
       ├──► JS: Calculate date1 (Lifespan - 2 days) & date2 (Lifespan + 2 days)
       ├──► Write order metafields "reminder_date_1" & "reminder_date_2" (REST)
       └──► Tag Order "reminder1_YYYY-MM-DD" & "reminder2_YYYY-MM-DD" (GraphQL)
       
[Daily Cron (8:00 AM)]
       │
       ▼
[n8n Workflow Part B]
       │
       ├──► Fetch all orders tagged "reminder1_today" or "reminder2_today" (GraphQL)
       ├──► Split Out into individual threads
       ├──► Switch: Route by tag type
       │      ├──► Route 1 (Pre-Runout): Send WhatsApp template 'purchase_reminder'
       │      └──► Route 2 (Post-Runout): Send WhatsApp template 'restock_reminder_post'
       └──► Untag Order "reminder1_today" and "reminder2_today" (GraphQL)
```

---

## 2. Failure Modes & Operational Resilience

Below is an analysis of what can go wrong, the impact, mitigation strategies, and how to diagnose issues.

### A. Webhook Delivery Failures (Shopify -> n8n Part A)
* **What happens:** Shopify fails to send the `orders/create` webhook (e.g., n8n instance is down, network timeout).
* **Impact:** The reminder dates are never calculated, and the order is never tagged. The customer will not receive any reminders.
* **Mitigation:** Shopify webhooks have built-in retry mechanisms (retrying 19 times over 48 hours).
* **Diagnosis:** Check the Shopify Admin -> Settings -> Notifications -> Webhooks section to inspect delivery failures, or check n8n's execution history for missing trigger payloads.

### B. Missing Product Lifespan Metafield
* **What happens:** A customer buys a product that does not have the `custom.lifespan_days` metafield populated.
* **Impact:** Without a fallback, the JavaScript node would crash or evaluate to `NaN`, halting the workflow.
* **Mitigation (Implemented):** The JavaScript code includes a resilient fallback block:
  ```javascript
  let baseLifespan = 30; // Default fallback to 30 days
  ```
* **Diagnosis:** Monitor executions of the JavaScript node in n8n. If the output payload uses exactly 30 days as base lifespan, check if the product in Shopify has `custom.lifespan_days` set.

### C. WhatsApp API Failure / Rate Limiting (n8n Part B -> Messaging.digital)
* **What happens:** The WhatsApp API is down, credentials expire, or rate limits are hit.
* **Impact:** Messages fail to send.
* **Mitigation:** By default, n8n's HTTP Request node can be configured to retry on failure (e.g., 3 retries with exponential backoff).
* **Diagnosis:** Inspect Part B execution logs. Look for non-200 responses from `https://messagingapi.charteredinfo.com`.

### D. Tag Removal Failure (Critical Failure Mode)
* **What happens:** The WhatsApp message is successfully sent, but the Shopify GraphQL mutation to remove `reminder1_YYYY-MM-DD` or `reminder2_YYYY-MM-DD` fails.
* **Impact:** The order remains tagged. On the next daily run, the workflow will fetch the same order again and send a **duplicate WhatsApp message** to the customer.
* **Mitigation:** 
  1. Configure n8n's GraphQL untag node to retry on failure.
  2. Implement a validation check: log execution IDs in a database or google sheet to prevent duplicate runs for the same order.
* **Diagnosis:** Customer complaints of duplicate messages, or checking Shopify order timelines to see if a tag removal mutation failed.

---

## 3. Performance & System Optimizations

1. **Decoupled Architecture (Latency Minimization):** We do not block the checkout flow. The order webhook is accepted immediately by n8n. The expensive process of sending notifications is shifted to an off-peak daily cron job (8:00 AM).
2. **GraphQL Batching:** Part B fetches up to 50 orders in a single API call instead of executing 50 individual database lookups.
3. **Advanced Lifespan Calculation:** Programmatically parses variant titles (e.g. "Pack of 3") and quantities:
   ```javascript
   let packMultiplier = 1;
   let packMatch = variantName.match(/Pack of (\d+)/i);
   if (packMatch) packMultiplier = parseInt(packMatch[1]);
   let totalLifespan = baseLifespan * quantity * packMultiplier;
   ```
4. **Data Sanitization:** Phone numbers are cleaned programmatically inside n8n to strip any formatting, spaces, or country-code symbols (e.g., `+`, `-`, `()`) before calling the WhatsApp API:
   ```javascript
   {{ ($json.node.phone || $json.node.customer?.phone || '').replace(/\D/g, '') }}
   ```
5. **Fallback Chain:** Checks `order.phone` first, falling back to `customer.phone` to guarantee maximum deliverability.

---

## 4. Prompt Caching & Hallucination Reduction (AI/LLM System Design)

Although this n8n system uses deterministic logic nodes, if LLMs/AI features are introduced in the future (e.g., writing custom reminders), the following guardrails must be applied:

* **Hallucination Prevention:** Do not allow LLMs to calculate dates or customer numbers. Keep the core state machine (date math, customer ID matching) in deterministic code (JavaScript). Only use LLMs for content personalization, wrapped in strict JSON schemas.
* **Prompt Caching & Latency:** If using an LLM to generate custom message copy, construct prompts with a static system prompt and dynamic variables at the end. This allows engines (like Gemini or Claude) to cache the system prompt, reducing latency and input token costs.
* **Tool Calling Errors:** Any AI-agent workflow interacting with Shopify must implement schema-based tool validation to prevent LLMs from passing invalid arguments to mutations.
