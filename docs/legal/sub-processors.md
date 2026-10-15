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
| 1 | **Neon, Inc.** (Neon Postgres + pgvector) | Primary durable storage for per-tenant catalog, anonymous session identifiers, widget events, filtered order records, per-tenant configuration, and cost telemetry. | Anonymous session IDs; widget event rows (event type, placement, intent, holdout flag, timestamp); filtered order rows (order ID, line items with `platform_product_id` / quantity / price, totals, currency); per-tenant catalog (product IDs, titles, descriptions, prices, tags, images). No shopper direct identifiers. | EU (Neon EU project) | EU–EU. No third-country transfer. | Active tenant lifetime, plus 30-day soft-delete window, then hard purge. Filtered order rows: 18 months from order creation (*DRAFT — pending PO + attorney review*; purge planned in M3·1). Backups up to 30 days after hard purge. | https://neon.tech/dpa |
| 2 | **Managed Redis provider (TBD — candidate: Upstash EU, Redis Cloud EU, or Render Redis EU)** | Ephemeral response cache (24-hour TTL), per-tenant usage-cap counters, per-session rate-limit counters, per-tenant kill-switch flag. | Anonymous session IDs (as part of rate-limit keys); tenant IDs; hashed intent signatures (not personal data on their own). No direct identifiers. | EU (constraint — must be EU region) | EU–EU. No third-country transfer. | 24 hours for cached responses; 1 hour for session rate counters; end-of-month reset for usage-cap counters; indefinite for kill-switch flag until manually cleared. | TBD at selection |
| 3 | **Render Services, Inc.** (backend + admin dashboard hosting) | Compute host for the WiseGift backend API, admin dashboard, and scheduled jobs. Handles Personal Data transiently in application memory during request processing; no durable Personal Data storage on Render. | All categories in clause 6 of the DPA are handled transiently in application memory during request/response processing. | EU (Render Frankfurt region) | EU–EU. No third-country transfer for production data. | Transient (request lifetime); application logs retained per WiseGift's logging policy with Personal Data filtered out at source. | https://render.com/dpa |
| 4 | **OpenAI, L.L.C.** — (a) `text-embedding-3-small` (embeddings); (b) `gpt-4o-mini-2024-07-18` (default ranker) and `gpt-4o-2024-08-06` (ranker for tenants with the `escalate_ranker` flag) via chat completions | (a) Embeddings of the merchant's product catalog at ingest, **and** of the visitor's intent on every recommendation request (including guardrail-fallback requests), for similarity retrieval. (b) Large-language-model re-ranking of up to 50 candidate products against the visitor's intent, on the live recommendation path only. | **Shopper intent payload:** intent mode (self / gift), recipient relationship, occasion, budget band, and **optional free-text interest chips typed by the visitor (max 5)**. The free text may incidentally contain personal or sensitive information the visitor chooses to type. A "Don't include personal details" notice next to the field is planned, not yet shipped. **ATTORNEY QUESTION (Art. 9):** see `consent-position.md` §5. When the widget is on a product page, the title of that product is added to the embedding input. **Catalog data:** product ID, title, description (truncated to 240 characters for ranking), price and tags. **Not transmitted:** the anonymous session ID, tenant ID, event history, order data, or any direct identifier. | **Currently US:** the backend calls `api.openai.com`. **Planned EU:** on 2026-10-14 the PO decided to move to an OpenAI EU data-residency project (`eu.api.openai.com`). That switch has not been made, and the US endpoint remains live. This cell must keep saying "US" until the switch is made and verified. | Third-country transfer (US). To be covered by Standard Contractual Clauses executed between WiseGift and OpenAI, plus OpenAI's DPA and its API data-usage policy (API inputs are not used to train OpenAI models). **[TO CONFIRM: DPA/SCCs executed with OpenAI.]** | Per OpenAI's API data-retention terms. **[TO CONFIRM: whether zero data retention is enabled on the WiseGift OpenAI project; it cannot be verified from code.]** | https://openai.com/policies/data-processing-addendum |
| 5 | **Anthropic, PBC** | **Not currently engaged.** Anthropic was the planned LLM ranking provider. The backend replaced it with OpenAI in PR 7d, and no shopper or catalog data is sent to Anthropic. Kept here only so readers of earlier drafts see the change. Remove this row before publication. | None. | — | — | — | — |

> *DRAFT — pending PO + attorney review.* Rows 4 and 5 rewritten to match
> the backend as read on 2026-10-14:
> - `recommendation/.../openai/OpenAiRankingClient.java` and
>   `RankingPromptBuilder.java`: the prompt contains the intent fields and
>   candidate product texts, and no session ID.
> - `RecommendationService.java`: `retrieve()` embeds the intent through
>   the OpenAI embedding client on every request.
> - `app/src/main/resources/application.yml`: `wisegift.ranking.*`, with
>   base URL `https://api.openai.com`.
>
> The previous rows wrongly said OpenAI receives catalog data only, and
> named Anthropic as the active ranker. DPA clauses 7.2 and 10.2 have
> been corrected to match this row (2026-10-15). Row 4's region cell and
> its free-text notice were updated on 2026-10-15 per PO decisions.

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

- **The anonymous session identifier is not sent to any
  sub-processor other than WiseGift's own storage.** It is stored in the
  visitor's browser (`localStorage`, per DPA clause 6.1) and in the
  primary Postgres database (row 1). It is not sent to OpenAI (row 4)
  or to any observability vendor.
- **Shopify holds a copy of the identifier on the Merchant's side.**
  Shopify is the Merchant's own platform, not a WiseGift sub-processor.
  On Shopify storefronts, the widget writes the identifier to the
  visitor's cart as the attribute `_wg_session`, and Shopify copies it
  onto resulting orders (DPA clause 6.5). That copy is governed by the
  Merchant's agreement with Shopify.
  > *DRAFT — pending PO + attorney review.* This note replaces the
  > earlier first bullet, which named Anthropic and said the identifier
  > "never leaves WiseGift's backend boundary". The Shopify bullet is new.
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
