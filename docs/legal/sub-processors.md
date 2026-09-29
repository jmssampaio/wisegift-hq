# WiseGift — Sub-processor list

**Sub-processors of [WISEGIFT LEGAL ENTITY] as of [EFFECTIVE DATE].
Updated version at [SUB-PROCESSOR LIST URL].**

> **Informational only. Not legal advice.** This document is a working
> list prepared by WiseGift's internal security-and-privacy function for
> onboarding conversations with pilot merchants and their advisors. It
> is not a legal opinion. A qualified attorney in the merchant's
> jurisdiction should confirm the transfer mechanisms and the accuracy
> of the sub-processor descriptions before a paying contract is signed.
>
> Fill-in points are marked as `[WISEGIFT LEGAL ENTITY]`, `[EFFECTIVE
> DATE]`, and `[SUB-PROCESSOR LIST URL]`.

---

## 1. Purpose

This document lists every third party that processes Personal Data on
WiseGift's behalf in the delivery of the WiseGift Services, as
referenced in the WiseGift Data Processing Addendum (`dpa-template.md`,
clause 9). It also lists infrastructure providers that do **not**
process Personal Data but are relevant to a full picture of the data
flow (marked "No personal data").

**Changes to this list are communicated to Merchants at least 30 days
before taking effect, per DPA clause 9.3.** Merchants are responsible
for subscribing to updates at the URL above.

---

## 2. Sub-processors that process Personal Data

| # | Sub-processor | Purpose | Data categories | Region | Legal basis for transfer | Retention | DPA URL |
|---|---|---|---|---|---|---|---|
| 1 | **Neon, Inc.** (Neon Postgres + pgvector) | Primary durable storage for per-tenant catalog, anonymous session identifiers, widget events, filtered order records, per-tenant configuration, and cost telemetry. | Anonymous session IDs; widget event rows (event type, placement, intent, holdout flag, timestamp); filtered order rows (order ID, line items with `platform_product_id` / quantity / price, totals, currency); per-tenant catalog (product IDs, titles, descriptions, prices, tags, images). No shopper direct identifiers. | EU (Neon EU project) | EU–EU. No third-country transfer. | Active tenant lifetime, plus 30-day soft-delete window, then hard purge. Backups up to 30 days after hard purge. | https://neon.tech/dpa |
| 2 | **Managed Redis provider (TBD — candidate: Upstash EU, Redis Cloud EU, or Render Redis EU)** | Ephemeral response cache (24-hour TTL), per-tenant usage-cap counters, per-session rate-limit counters, per-tenant kill-switch flag. | Anonymous session IDs (as part of rate-limit keys); tenant IDs; hashed intent signatures (not personal data on their own). No direct identifiers. | EU (constraint — must be EU region) | EU–EU. No third-country transfer. | 24 hours for cached responses; 1 hour for session rate counters; end-of-month reset for usage-cap counters; indefinite for kill-switch flag until manually cleared. | TBD at selection |
| 3 | **Render Services, Inc.** (backend + admin dashboard hosting) | Compute host for the WiseGift backend API, admin dashboard, and scheduled jobs. Handles Personal Data transiently in application memory during request processing; no durable Personal Data storage on Render. | All categories in clause 6 of the DPA are handled transiently in application memory during request/response processing. | EU (Render Frankfurt region) | EU–EU. No third-country transfer for production data. | Transient (request lifetime); application logs retained per WiseGift's logging policy with Personal Data filtered out at source. | https://render.com/dpa |
| 4 | **Anthropic, PBC** (Claude Haiku 4.5 / Sonnet — recommendation re-ranking and intent parsing) | Large-language-model inference for re-ranking candidate products against the visitor's submitted intent. Called on the live recommendation path only. | Intent payload only: recipient relationship, occasion, budget band, optional free-text interest chips (max 5), and the candidate product texts (title, description, tags) drawn from the merchant's catalog. **No anonymous session ID is transmitted. No direct identifiers are transmitted.** | Anthropic API endpoints are US-based. WiseGift enables Anthropic's zero-retention configuration where available. | Third-country transfer (US). Covered by Standard Contractual Clauses (Module 3 — Processor to Processor) executed between WiseGift and Anthropic, plus Anthropic's DPA and enterprise data-handling commitments. | No durable retention by Anthropic under zero-retention configuration; transient during inference. | https://www.anthropic.com/legal/dpa |
| 5 | **OpenAI, L.L.C.** (`text-embedding-3-small` — product embeddings at catalog ingest) | Compute vector embeddings for the merchant's product catalog at ingest time, for downstream similarity retrieval. | **Product catalog data only** (title, description, product type, tags). **No shopper session data, no shopper events, no order data, no direct identifiers are transmitted.** Merchant product catalog content may itself be considered the Merchant's business data rather than Personal Data of Data Subjects. | OpenAI API endpoints are US-based. | Third-country transfer (US). Covered by Standard Contractual Clauses executed between WiseGift and OpenAI, plus OpenAI's DPA and its API data-usage policy (API inputs are not used to train OpenAI models). | No durable retention by OpenAI for API inputs; transient during embedding computation. | https://openai.com/policies/data-processing-addendum |

## 3. Infrastructure providers that do NOT process Personal Data

Listed for completeness. These vendors do not process Personal Data as
defined in the DPA; no transfer mechanism is required for shopper
Personal Data.

| # | Provider | Purpose | Data handled | Region |
|---|---|---|---|---|
| 6 | **Widget CDN (TBD — Cloudflare or Fastly)** | Distribution of the static widget JavaScript bundle to shopper browsers via edge cache. | Static JavaScript bundle only. No Personal Data. Widget-to-API traffic bypasses the CDN and terminates directly at the WiseGift API. Standard CDN request logs (IP, user-agent, path) may be produced by the CDN as its own controller for security purposes and are not accessed by WiseGift as processor. | Global edge with the origin located in the EU. |
| 7 | **Shopify, Inc.** (e-commerce platform integration — source system) | Shopify is the **source** of the Merchant's catalog and order data via OAuth-authorised Admin API calls and webhooks. Shopify is the Merchant's own processor under a separate agreement between Merchant and Shopify; it is **not** a sub-processor of WiseGift. | N/A — Shopify is upstream of WiseGift, not downstream. | Per Merchant's Shopify contract. |
| 8 | **Observability / log vendor** | TBD at MVP. WiseGift stores structured logs and cost telemetry in the primary Postgres database at MVP (Neon EU, listed at row 1). A dedicated observability vendor (candidate: Grafana Cloud EU) is planned post-MVP. When adopted, this row will be updated with the vendor's name and DPA URL, and Merchants will receive 30-day advance notice per DPA clause 9.3. | Structured application logs (Personal Data filtered at source per DPA clause 11.7); cost telemetry (tenant ID, model, tokens, latency — no Personal Data). | EU (constraint). |

---

## 4. Notes on the data flow

- **The anonymous session identifier never leaves WiseGift's backend
  boundary in a form linkable to a shopper.** It is stored in the
  visitor's browser (`localStorage`, per DPA clause 6.1) and in the
  primary Postgres database (row 1). It is not sent to Anthropic
  (row 4), not sent to OpenAI (row 5), and not sent to any observability
  vendor.
- **No sub-processor receives merchant customer identity data.** The
  filtered order webhook (DPA clause 6.3) drops customer name, e-mail,
  address, phone, and IP at the WiseGift receiver, before persistence
  and before any log write. No downstream sub-processor sees those
  fields because WiseGift never holds them beyond the milliseconds it
  takes to build the filtered order row.
- **EU region is a load-bearing commitment.** If WiseGift adds a
  non-EU processing region, this list will be updated and the DPA
  re-executed per DPA clause 10.3. As of the Effective Date, no such
  addition is planned.

---

## 5. Change log

| Date | Change |
|---|---|
| [EFFECTIVE DATE] | Initial publication. |

---

*This document is informational only and does not constitute legal
advice. Attorney review is recommended before relying on the transfer
mechanisms described above. Route binding questions to a qualified
attorney in the applicable jurisdiction.*
