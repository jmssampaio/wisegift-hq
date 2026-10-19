# WiseGift — Record of Processing Activities (GDPR Art. 30)

> **DRAFT — pending PO + attorney review.** Prepared 2026-10-17 by
> WiseGift's internal security-and-privacy function, per `decisions.md`
> 2026-10-17 "Budget event fields approved with conditions; floating nudge
> in memory only; record of processing". It is **informational only and
> does not constitute legal advice**. A qualified attorney must review it
> before it is relied on, shown to a supervisory authority, or handed to a
> Merchant. It is a pilot-gate item (same entry: "a record of processing
> exists").
>
> **Basis.** Everything here is taken from the current HQ documents, not
> from a fresh read of the code:
> - `docs/engineering/architecture.md` §1, §4, §5–§9, §11–§12 (tables, flows, retention);
> - `docs/product/spec.md` "Data & privacy" and "Contact email step and go-live gate";
> - `docs/decisions.md` up to 2026-10-17;
> - `docs/legal/dpa-template.md`, `sub-processors.md`, `consent-position.md`.
>
> Where the documents disagree or a fact is unverified, this record says
> so and marks it `[TO CONFIRM — owner]`. Statements about features that
> are **not shipped** are marked as such. A record that describes planned
> behaviour as current is inaccurate, so it is updated whenever the data
> flows change (same change, same PR as the DPA and sub-processor list).
>
> Fill-in points: `[WISEGIFT LEGAL ENTITY]`, `[WISEGIFT REGISTERED
> ADDRESS]`, `[PRIVACY CONTACT EMAIL]`, `[SUPPORT EMAIL]`, `[RECORD
> VERSION]`, `[LAST REVIEWED]`.

---

## 0. Record holder

| Field | Value |
|---|---|
| Organisation | **[WISEGIFT LEGAL ENTITY]**, incorporated in Spain, registered office **[WISEGIFT REGISTERED ADDRESS]** |
| Role(s) | **Processor** for Merchants' shopper data (Part A, Art. 30(2)). **Controller** for its own merchant-contact and infrastructure data (Part B, Art. 30(1)) |
| Privacy contact | **[PRIVACY CONTACT EMAIL]** (EU-hosted role address; PO to fill) |
| Data protection officer | None designated. **ATTORNEY QUESTION:** is a DPO required under Art. 37 (see §D, A9)? |
| EU representative (Art. 27) | Not applicable: WiseGift is established in the EU |
| Record version / last reviewed | **[RECORD VERSION]** / **[LAST REVIEWED]** |
| Review trigger | Any change to data fields, sub-processors, region, retention or consent behaviour. Reviewed together with `dpa-template.md` and `sub-processors.md` |

---

## Part A — WiseGift as processor (Art. 30(2))

### A.0 Controllers (Art. 30(2)(a))

Each **Merchant** that installs WiseGift and signs the DPA is a separate
controller. WiseGift processes each Merchant's data in a logically
separate tenant (`tenant_id`).

- **Controller register.** Art. 30(2)(a) requires the name and contact
  details of each controller. WiseGift's database holds the shop domain
  (`platform_credentials.platform_shop_domain`), the shop ID and the
  confirmed contact email (`tenants.contact_email`). It does **not** hold
  the Merchant's legal name or registered address. The register must
  therefore be kept as an annex built from signed DPAs: legal name,
  address, signatory, DPA effective date, `tenant_id`, shop domain,
  contact email. `[TO CONFIRM — PO: where the annex lives; it contains
  merchant contact data and must stay EU-hosted and access-restricted]`.
- **Merchant's own DPO / representative:** recorded in the annex where
  the Merchant provides it.
- **Current state.** No paying Merchant and no pilot storefront exists.
  Staging tenants only (`wisegift-dev`, `dummy-store-3x0otvhy`), operated
  by WiseGift itself on test data.

### A.1 Common facts for all Part A entries

These apply to every entry below unless the entry says otherwise.

**Sub-processors** (`sub-processors.md`):

| # | Sub-processor | Role | Region | Third-country transfer |
|---|---|---|---|---|
| 1 | Neon, Inc. | Primary Postgres + pgvector | EU (Neon EU project) | None from the EU region as configured. Neon, Inc. is US-established; see gap G9 |
| 2 | Managed Redis (**not yet selected**: Upstash EU, Redis Cloud EU or Render Redis EU) | Cache, rate-limit and usage counters, kill switch | EU (constraint) | None if an EU region is chosen. Must be selected and listed before the first pilot |
| 3 | Render Services, Inc. | Backend, admin and job hosting; transient processing in memory; request logs | EU (Frankfurt) | None for production data as configured. Render Services, Inc. is US-established; see gap G9 |
| 4 | OpenAI, L.L.C. | Embeddings and LLM ranking (entries A2 recommendation requests and A4 catalogue embedding; **not** A3 orders) | **Currently US** (`api.openai.com`). EU data-residency project decided 2026-10-14, **not switched** | **Yes, today.** SCCs Module 3 + supplementary measures claimed in DPA 10.2. `[TO CONFIRM — PO: OpenAI DPA/SCCs executed; zero data retention status]` |

**Other recipients (not WiseGift sub-processors):**
- **The Merchant** itself, through the admin dashboard (aggregates only),
  CSV export (aggregates only) and the support-run JSONL export (DPA 14.5).
- **Shopify, Inc.**, as the Merchant's own platform: it holds the copy
  of the session identifier written to carts and orders (DPA 6.5). It is
  upstream of WiseGift and governed by the Merchant's contract with
  Shopify.
- No sale or sharing of data with any third party for its own purposes.

**Common TOMs** (DPA clause 11; status as documented on 2026-10-17):

| DPA clause | Measure | Status |
|---|---|---|
| 11.1 | `tenant_id NOT NULL` on every tenant table; every query tenant-scoped; two-tenant isolation tests in CI | In place (per architecture §4) |
| 11.2 | Encryption at rest (Neon-managed) and TLS 1.2+ on every hop | In place per vendor configuration. `[TO CONFIRM — devops: Redis TLS once the provider is selected]` |
| 11.3 | Widget JWT (HS256, 1 h TTL) bound to tenant + session; tenant resolved from `Origin` against the shop's domains; refused with 403 when the tenant is not live | In place. **Known issue G-f** (claims built with string formatting; duplicate `exp` injection, TTL bypass, no cross-tenant leak): fix required before the first pilot |
| 11.4 | Shopify webhook HMAC, fail closed, 401, logged without payload | In place |
| 11.5 | Per-tenant usage cap, per-session rate limit, kill switch | In place |
| 11.6 | Order PII fields dropped at the receiver before log and persistence | In place |
| 11.7 | No secrets, tokens or Personal Data in logs; no internals in error bodies | In place. Confirmed from the backend (2026-10-19): the application logs **no** client IP or user-agent anywhere (G2). Hosting (Render) request logs may hold IP/UA; retention `[TO CONFIRM — devops]` |
| 11.8 | Merchant admin authentication | DPA 11.8 corrected 2026-10-19 to match (G3 resolved). Shopify session tokens (App Bridge ID token, verified HS256, `aud`/`dest`/`exp`/`nbf`), tenant derived only from the verified shop claim; no staff credentials stored; `sub`/`sid`/`jti` never read, stored or logged. `admin_users` is legacy-only and never touched by the embedded path |
| 11.9 | OAuth tokens encrypted at rest; decrypted only in memory | In place |
| 11.10 | Personnel confidentiality and training | `[TO CONFIRM — PO: written confidentiality undertakings exist]` |
| 11.11 | Code review + CI (unit, integration, tenant-isolation tests); human-gated deploys | In place |
| 6.5(e) | Consent gating (Customer Privacy API; `consent_mode_required` default `true`, fail closed) | **Not fully verified.** Running on staging per `decisions.md` 2026-10-15 (scenario B verified; A and C inconclusive; D not run). The DPA 6.5(e) note still says PR #19 is unmerged. `[TO CONFIRM — frontend: merge and release status]`; align the DPA note |
| 14.3 / 14.6 | Tenant hard purge at day 30; 120-day / 18-month purge | **Not enforcing.** Tenant purge exists in dry-run (PR #79). Until the purge runs out of dry-run, nothing is deleted and every row is kept indefinitely (architecture §8). Pilot gate |

**Common retention backstops:**
- Tenant uninstall overrides every period below: soft-delete for 30 days,
  then hard purge of all tenant rows (DPA 14.1–14.3).
- Backups: up to 30 days after hard purge (DPA 14.4). `[TO CONFIRM —
  devops: Neon point-in-time-restore / history window on the production
  plan matches 30 days]`.

---

### A1. Widget sessions and events

| Field | Content |
|---|---|
| **Controller** | The Merchant (A.0) |
| **Data subjects** | Visitors to the Merchant's storefront who render or use the widget **and** for whom tracking is allowed (consent given, no consent required in their region, or — no tenant should use this yet — `consentModeRequired = false` with no consent interface). This includes gift recipients **indirectly**: intent data describes a third person (relationship, occasion) |
| **Data categories** | **Browser:** pseudonymous session ID (UUID v4) in `localStorage` key `wg_session`, 30-day sliding TTL. **Shopify cart:** copy of the ID as cart attribute `_wg_session` (DPA 6.5). **Product links:** `?wg_session=<id>` marker on rec-card links (exposed arm only). **`sessions`:** `session_id`, `tenant_id`, `is_holdout`, `first_seen_at`, `last_seen_at`. **`events`** (per architecture §8): `event_type` (`widget_shown`, `widget_engaged`, `intent_submitted`, `product_clicked`), `placement`, `intent_mode` (self/gift), `is_holdout`, `platform_product_id` (clicks), `rank_score`, `served_from`, `intent_form_schema_version`, `pre_intent_recs` (**built**), `occurred_at` (client clock). Note: `received_at` is a column on `orders_attributed`, **not** on `events` (G7 corrected). **Budget-step fields (OQ-6, not shipped):** `budgetTierSetVersion`, `budgetSuggestedTier`, `budgetChoice` (`tier0`–`tier3`, "Any price" or "not asked"), `budgetInteracted`. No amounts, no free text, no timing or counts (condition C3). `budgetSuggestedTier` coarsely reveals relationship × occasion and is treated as intent data. **Not stored (confirmed from the backend, Flyway V1–V15):** relationship, occasion, budget and free-text interests. There is no intent payload / JSON column. These answers are a transient transfer to OpenAI plus a one-way SHA-256 hash in the cache key (A2). **G1 resolved 2026-10-19:** DPA 6.2 and `consent-position.md` §5 have been corrected to match. **Redis:** session ID in rate-limit keys `wg:rate:{session_id}:{YYYY-MM-DD-HH}`, TTL 1h |
| **Special categories** | None intended. The free-text interests field may incidentally contain Art. 9 data. It is **not stored** with events (G1 resolved 2026-10-19); it is transiently sent to OpenAI (A2) |
| **Purposes (DPA 7.1)** | (b) lift metrics against the holdout (10%, or 20% for design partners who opt in; see G6); (c) per-session rate limiting; (d) budget-step measurement, once shipped: per-Merchant aggregates, plus offline cross-Merchant aggregates with no session ID. **Attorney question A1** applies to the cross-Merchant step |
| **Not used for** | Ranking, prompts or embeddings (budget fields, condition C6); cross-merchant tracking; advertising; building a profile beyond the session; decisions about a visitor |
| **Sub-processors / recipients** | Neon (storage), Render (transient), Redis provider (rate keys). **Not sent to OpenAI.** The Merchant sees aggregates only; budget fields are never exported per session (C6); fleet figures are never shown to another Merchant (C7). Shopify holds the cart/order copy on the Merchant's side |
| **Transfers outside the EU** | None as configured (subject to G9) |
| **Retention** | Browser ID: 30-day sliding TTL; cleared on decline once consent gating is verified. `events`: 120 days from `occurred_at`. `sessions`: 120 days after `last_seen_at`. Redis rate counters: 1 hour. Aggregates (`analytics_kpi_daily`, proposed, no session ID): 18 months. Copies in Shopify carts and orders: Merchant's own retention, outside WiseGift's control (DPA 6.5(d)). **Not enforcing until the purge leaves dry-run** |
| **TOMs (DPA 11)** | A.1 common TOMs, plus: consent gating 6.5(e); the server ignores the client holdout flag; strict event schema with unknown fields rejected; budget fields enum-mapped server-side, validated against the tenant's own tier set per `(tenant_id, version)`, invalid values nulled with only the field name logged (C1, C2); fields travel only in the consent-gated event queue (C5); the widget refuses to run for a tenant that is not live (403, Q-C8, PR #85) |
| **Processing without tracking consent** | No session row, no event, no storage, no cart attribute. Recommendations still render in degraded mode (A3). The server-minted session ID lives in page memory only, to bind the 1-hour token; its holdout bucket is a per-page coin flip. **Pending (widget PR #26):** an inline block's viewport-entry bootstrap before any consent returns copy plus a token bound to a random page identifier, held in memory only. **Attorney question A2** |
| **Known gaps** | Today's bundle still writes `wg_session` / `_wg_session` for a tenant that is not live, before its first API call (architecture §10.3; widget PRs #24/#25 open). See G14 |

### A2. Recommendation requests (including what is sent to OpenAI)

| Field | Content |
|---|---|
| **Controller** | The Merchant |
| **Data subjects** | Storefront visitors who request recommendations, **in every consent state** (with or without tracking consent); gift recipients indirectly |
| **Data categories** | **Request** (`POST /widget/v1/recommendations`): session ID (persisted or in-memory), placement, intent mode, `intent` (relationship, occasion, `budget_min`/`budget_max`, up to 5 free-text interest chips), `context.platform_product_id` on PDPs, limit. **Processing artefacts:** intent embedding vector (in memory), candidate list, ranked response. **Redis cache:** responses keyed `rec:{tenant_id}:{intent_signature}:{context_signature}` (exact budget `min-max` in the key since Decision 3); the signature is a one-way SHA-256 hash of the intent (confirmed from the backend, 2026-10-19); the free-text interests are not held in clear in the key. **Sent to OpenAI:** for embedding, the intent (mode, relationship, occasion, budget, free-text interests) plus the current product title on PDPs; for ranking, the intent including free text plus up to 50 candidate products (ID, title, description truncated to 240 characters, price, tags). **Never sent to OpenAI:** session ID, tenant ID, event history, order data, budget-step event fields, direct identifiers |
| **Special categories** | Possible, incidental, through free text (Art. 9). The "Don't include personal details" notice is **planned, not shipped**. **Attorney question A4** |
| **Purposes (DPA 7.1)** | (a) render personalised recommendations; (c) cost guardrails: cache, usage cap, rate limit, kill switch |
| **Sub-processors / recipients** | OpenAI (embeddings + ranking), Render (transient), Redis provider (cache, counters). The response goes only to the requesting browser |
| **Transfers outside the EU** | **Yes, today: OpenAI, US** (`api.openai.com`). Safeguard claimed: SCCs Module 3 + supplementary measures (DPA 10.2), `[TO CONFIRM — PO: executed]`. **Planned:** OpenAI EU data-residency project (`eu.api.openai.com`), decided 2026-10-14, gated on PO preconditions (eligibility, EU availability of the three pinned models, ZDR, DPA/SCCs). OpenAI remains a US-established entity after the switch; **attorney question A6** |
| **Retention** | WiseGift: request lifetime in memory; cached responses 24 hours; rate counters 1 hour; usage counters reset monthly. No intent payload is persisted (G1 confirmed from the backend, 2026-10-19; no intent column exists). OpenAI: per its API data-retention terms; by default API inputs may be retained for abuse monitoring for a limited period `[TO CONFIRM — PO: current OpenAI terms and whether zero data retention applies to the WiseGift project]` |
| **TOMs (DPA 11)** | A.1 common TOMs, plus: vendor-neutral `RankingClient` port; strict validation of relationship and occasion against the canonical vocabularies (unknown values become null and are logged by field, no 400); interests capped at 5; per-tenant usage cap and kill switch bound volume; API inputs not used for model training (OpenAI API policy); no session ID in prompts (verified in code 2026-10-14, `RankingPromptBuilder`). Recommended: startup log of the effective OpenAI base URLs and a production guard against a non-EU host (2026-10-14 decision, backend follow-up) |

### A3. Attributed orders

| Field | Content |
|---|---|
| **Controller** | The Merchant |
| **Data subjects** | **Every** customer who places an order on a live tenant's storefront, whether or not they saw the widget (unattributed orders are kept for baselines). Also Merchant staff placing Shopify test orders |
| **Data categories** | **Received** (`orders/create` webhook, HMAC-verified): full order payload including customer identity. **Dropped at the receiver, before any log or persistence:** `customer`, `billing_address`, `shipping_address`, `email`, `phone`, `client_details`. **Stored in `orders_attributed`:** `platform_order_id`, `tenant_id`, `session_id` (nullable; strict v4 UUID read from `_wg_session`, legacy `wg_session` or the URL marker), `total_price` (gross), `currency`, `line_items` (`platformProductId`, quantity, price only), `created_at`, `received_at`, the Shopify **`test` flag** (`is_test`, decided 2026-10-16; test orders are stored but excluded from every readout), and **`attribution_source`** (**built, V7**: `note_attribute` / `landing_site` / `referring_site`; G5 corrected 2026-10-19). **Personal-data status:** WiseGift cannot identify the customer, but the Merchant can, through the order ID and the `_wg_session` copy on its own order. The rows are therefore treated as pseudonymous personal data |
| **Not processed** | Orders for a tenant that is not live are ignored: nothing persisted, Shopify still gets 200 (PR #85). Buy-it-now / accelerated-checkout orders carry no cart attribute and stay unattributed |
| **Purposes (DPA 7.1)** | (b) lift metrics: conversion, AOV, revenue per session; merchant-wide baselines from unattributed orders. **Not** used for ranking at MVP |
| **Sub-processors / recipients** | Neon (storage), Render (transient). Not sent to OpenAI. The Merchant sees aggregates and the CSV export (aggregates) |
| **Transfers outside the EU** | None as configured (subject to G9) |
| **Retention** | Order rows: 18 months from `created_at`, then hard-deleted. `session_id` on the row: nulled at 120 days, which breaks the customer ↔ widget-history link. Test orders: same periods `[TO CONFIRM — PO: whether test orders should be deleted sooner, as they serve no readout]`. **Not enforcing until the purge leaves dry-run** |
| **TOMs (DPA 11)** | A.1 common TOMs, especially 11.4 (HMAC, fail closed) and 11.6 (PII drop before log, covered by integration tests); idempotent processing per order ID; strict UUID validation of the session marker. Data-subject requests: GDPR webhooks (`customers/redact`, `customers/data_request`) covering sessions and events linked through order IDs are **planned (M3·2), not built**; they must not be blocked for inactive tenants (DPA 12.2) |

### A4. Catalogue data (not personal data; listed for completeness)

| Field | Content |
|---|---|
| **Controller of the business data** | The Merchant |
| **Personal data?** | **No, in principle.** Products, variants, prices, tags, images, descriptions, the derived budget tier sets, precomputed recommendations and embeddings |
| **Caveats** | (1) Free-form fields (titles, descriptions, tags, image URLs) could incidentally contain a person's name or image, e.g. an artisan or model. Not processed as such. (2) If the deferred social-content work (influencer images) is ever built, it brings personal data and needs its own entry, with the preconditions in `decisions.md` 2026-10-15 "Hooks per placement and niche packs" |
| **Data held** | `tenant_products` (incl. `embedding`), `tenant_product_variants`, `precomputed_recs`, `catalog_sync_runs`, per-tenant tier sets (budget step) |
| **Purposes** | Providing the Services (DPA 4.1, 7.1(a)); the nightly sync runs for every installed tenant, live or not (2026-10-16 decision) |
| **Sub-processors / recipients** | Neon, Render; OpenAI receives product texts for embedding (at ingest) and ranking (candidates) |
| **Transfers** | Product texts to OpenAI, US today (same as A2) |
| **Retention** | Tenant lifetime + 30-day soft-delete, then hard purge |
| **TOMs** | Tenant isolation 11.1; OAuth tokens 11.9; Shopify rate-limit discipline |

### A5. Cost telemetry and guardrail counters

| Field | Content |
|---|---|
| **Controller** | The Merchant for the guardrail counters keyed by session ID; the telemetry rows themselves are operational data |
| **Personal data?** | **`cost_telemetry` rows: no.** Columns are `tenant_id`, `placement`, `model`, `input_tokens`, `output_tokens`, `cache_hit`, `served_from`, `latency_ms`, `occurred_at`; no session ID, no intent. **Redis per-session rate counters: yes, pseudonymous** (session ID in the key; covered in A1/A2). Per-tenant usage counters and the kill-switch flag: no. Confirmed from the backend (2026-10-19): `cost_telemetry` holds **no session ID and no intent**. The budget-step "empty-slate rate" counter is not built |
| **Purposes (DPA 7.1)** | (c) cost guardrails: daily-spend kill switch, usage cap, per-merchant cost reporting |
| **Sub-processors / recipients** | Neon (`cost_telemetry` at MVP), Redis provider (counters). Planned: Grafana Cloud EU post-MVP, which needs a 30-day notice and a sub-processor row |
| **Transfers** | None |
| **Retention** | `cost_telemetry`: tenant lifetime, then hard purge with the tenant. **No fixed period is set and none is enforced yet** (confirmed from the backend, 2026-10-19; G18 open) `[TO CONFIRM — PO: e.g. 18 months, aligned with the aggregates]`. Redis: 1 h (session counters), monthly reset (usage), until cleared (kill switch) |
| **TOMs** | Tenant isolation 11.1; telemetry carries no Personal Data by design (11.7) |

---

## Part B — WiseGift as controller (Art. 30(1))

### B1. Merchant contact email

| Field | Content |
|---|---|
| **Controller** | WiseGift (decision 2026-10-15 "Embedded admin auth", decision 5; spec "Contact email step", privacy flag). **Attorney question A3** confirms the role |
| **Data subjects** | The Merchant's staff member whose address is entered, where the address is personal (e.g. `jane@shop.com`). A role address (`privacy@shop.com`) is often not personal data; the helper text encourages one |
| **Data categories** | `tenants.contact_email` (≤ 254 chars, trimmed, as entered), `tenants.contact_email_confirmed_at`; audit rows `tenant_contact_email_events (tenant_id, action, occurred_at)` with **no address and no staff identity**. **Transient:** the Shopify shop email, read live as a pre-fill suggestion and never stored, cached or logged unless the Merchant confirms it |
| **Purposes** | DPA notices: breach notification (clause 13), sub-processor changes (9.3), region changes (10.3); billing; essential service and security notices. **Never marketing** (Q-C3; attorney to confirm the list) |
| **Legal basis** | **ATTORNEY QUESTION (A3).** Candidates: Art. 6(1)(b), necessary for the Main Agreement and DPA with the Merchant (but the data subject is the staff member, not the contracting party); Art. 6(1)(f), WiseGift's and the Merchant's legitimate interest in reaching a monitored address; Art. 6(1)(c) for the breach-notice element (Art. 33(2) obliges the processor to notify the controller). The in-app checkbox is a confirmation, **not** consent, and should not be presented as consent |
| **Recipients** | WiseGift personnel handling notices and billing. The `[SUPPORT EMAIL]` mailbox provider (not yet chosen; must be EU-hosted). No email-sending provider at MVP. Never sent to OpenAI or any shopper-data sub-processor; never in logs, error bodies, analytics, CSV exports, or `/me` (only the boolean) |
| **Sub-processors / processors of WiseGift** | Neon (storage), Render (transient), the support mailbox provider `[TO CONFIRM — PO: provider and region]` |
| **Transfers outside the EU** | None as configured (subject to G9 and the mailbox provider choice) |
| **Retention** | While the tenant exists, including the 30-day soft-delete window; deleted with the `tenants` row at hard purge. Backups per DPA 14.4. Whether the previous address must be kept as evidence of where a notice was sent: **Q-C5, attorney** |
| **Transparency** | Needs WiseGift's own merchant-contact privacy notice (Q-C4): text and URL **pending attorney wording**; placeholder in the onboarding step |
| **TOMs** | Never logged; not returned by `/me`; CHECK constraint (address and timestamp both set or both null); tenant isolation; encryption at rest |

### B2. Merchant admin access

| Field | Content |
|---|---|
| **Position** | **No merchant-staff personal data is stored** under the embedded-auth model (decision 2026-10-15, decision 4): no emails, password hashes, Google identities, or Shopify user-ID audit log |
| **Transient processing** | Every admin call carries a Shopify App Bridge ID token (JWT). The backend verifies it in memory and derives the tenant only from the verified shop claim. The token carries Shopify staff claims (`sub`/`sid`/`jti`); confirmed from the backend (2026-10-19) that these are **never read, stored or logged** (G3). `[TO CONFIRM — backend: the token URL parameter (plan S1) is never logged]` |
| **Legacy data** | `admin_users` (email, `password_hash`, `google_sub`) remains from the retired invite / cookie-login flow until **Phase 2** removes the code. New tenants get no row (fresh-install E2E, 2026-10-16). Staging rows likely exist for `wisegift-dev`, whose first admin was created through the invite API. Related: Redis-backed Spring Session entries from the legacy cookie chain, including those created by anonymous `/me` probes. `[TO CONFIRM — backend: count of admin_users rows per environment; production must have none]`. **Recommendation:** delete legacy rows and the table in Phase 2, in the same PR that removes the code; until then, no new rows in production |
| **Purposes / legal basis (legacy rows)** | Historical admin login on staging only. If any row ever exists in production, it needs its own legal basis and notice. Recommended outcome: none exist |
| **Retention** | Until Phase 2 deletion, or tenant hard purge, whichever is first |

### B3. Infrastructure and request logs

| Field | Content |
|---|---|
| **Controller** | WiseGift, for the security and operation of its own systems. **Attorney question A5:** is WiseGift controller for these logs, or processor for the Merchant (as for the widget traffic they record)? |
| **Data subjects** | Storefront visitors whose browsers call `/widget/v1/*` (including pre-consent viewport bootstraps once widget PR #26 ships); Merchant staff using the admin; Shopify (webhooks, not personal) |
| **Data categories** | **Render request logs:** client IP address, user agent, request path and query, status, timing `[TO CONFIRM — devops: exact field list, including whether query strings (e.g. id_token on admin URLs) are logged]`. **WiseGift application logs:** structured logs with tenant ID and event codes; **no** IP, user agent, raw `locale`, tokens, intent payloads or Personal Data per policy 11.7, to be verified on the bootstrap path (2026-10-17 condition 2). Webhook rejections are logged without payload |
| **Purposes** | Security, abuse detection, incident investigation, debugging, availability |
| **Legal basis** | Art. 6(1)(f), legitimate interest in securing the service (to be confirmed with A5) |
| **Recipients / processors** | Render Services, Inc. (log storage); planned observability vendor (Grafana Cloud EU) post-MVP. The widget bundle CDN, if any (see G10) |
| **Transfers outside the EU** | Render logs in the EU (Frankfurt) `[TO CONFIRM — devops: log storage region and whether Render staff outside the EU can access them]` |
| **Retention** | Render request logs: **`[TO CONFIRM — devops]`** (depends on plan; to be documented in `sub-processors.md` before the first pilot, 2026-10-17 condition 3). WiseGift application logs: `[TO CONFIRM — devops]` |
| **TOMs** | Log minimisation at source (11.7); `Referrer-Policy: no-referrer` on the admin; widget fetches with `referrerPolicy: "no-referrer"` and `credentials: "omit"` (PR #26 condition 1, before merge); access to the Render dashboard limited to named personnel `[TO CONFIRM — PO: who has access; MFA enabled]` |

### B4. Other controller activities (noted, not yet fully described)

- **Merchant account records.** Shop domain, shop ID, plan, store
  vertical, configuration, OAuth tokens. Mostly business data; a sole
  trader's shop domain can be personal data. Retention: tenant lifetime
  + 30 days. Purpose: providing and billing the Services.
- **Support correspondence.** Merchant emails to `[SUPPORT EMAIL]`,
  which may contain staff or forwarded shopper data (e.g. data-subject
  requests). Provider not chosen. Retention `[TO CONFIRM — PO]`.
- **Support-run data exports** (DPA 14.5: JSONL via signed URL). The
  storage location of the export file is not documented `[TO CONFIRM —
  devops]`. If it is a new vendor (object storage), it is a
  sub-processor and needs the 30-day notice. This is processor activity
  for the Merchant, listed here only because it is undocumented.
- **Billing.** Shopify App Store billing at MVP; no WiseGift billing
  data store yet (Q-C10 open). Add an entry when billing is built.

---

## C. Gaps and inconsistencies to fix

Listed for fixing. None is fixed in this change. Owners in brackets.

| # | Where | Gap | Fix / owner |
|---|---|---|---|
| G1 | DPA 6.2; `consent-position.md` §5; spec "Data & privacy" | **RESOLVED 2026-10-19 (DPA + consent-position).** Backend ground truth (Flyway V1–V15) confirms `events` has no intent columns and no JSON payload; intent (relationship, occasion, budget, free text) is **never stored** — it is a transient transfer to OpenAI plus a one-way hash. DPA 6.2 and `consent-position.md` §5 corrected. **Still open:** spec "Data & privacy" wording ("intent submissions (relationship, occasion, budget, interests)") must be fixed by `product-analyst` |
| G2 | DPA 6.4; spec "What we never store" | **RESOLVED 2026-10-19 (DPA + consent-position).** Backend confirms the application logs no client IP or user-agent anywhere; Render request logs may hold them. DPA 6.4 and `consent-position.md` §4 reworded accordingly. **Still open:** hosting-log retention `[N days — TO CONFIRM — devops]`; spec "What we never store" wording to align (`product-analyst`) |
| G3 | DPA 11.8; `architecture.md`; spec "Out of MVP scope" | **RESOLVED 2026-10-19 (DPA).** Backend confirms embedded Shopify session-token auth; no merchant-staff personal data stored; `sub`/`sid`/`jti` never read/stored/logged; `admin_users` legacy-only. DPA 11.8 corrected. **Still open:** `architecture.md` already fixed per 2026-10-18 commit; spec "Out of MVP scope" wording to align (`product-analyst`) |
| G4 | DPA clause 13 | Does not say **where** breach notices go | Name the confirmed contact email as the notice address (spec privacy flag). `security-and-privacy`, attorney wording |
| G5 | DPA 6.3 | **RESOLVED 2026-10-19.** `attribution_source` is **built (V7)**, not "planned". DPA 6.3 now lists `received_at`, `is_test`, `attribution_source` (note_attribute/landing_site/referring_site), states that unattributed orders (null session ID) are stored for baselines, and that orders for a non-live tenant are dropped |
| G6 | DPA 4.1, 7.1(b); `consent-position.md` §2 | Say "10% holdout"; design partners may opt in to 20% (D4) | "10% holdout by default, or 20% where the Merchant opts in". DPA 4.1 also omits purpose 7.1(d) |
| G7 | DPA 6.2 | **RESOLVED 2026-10-19.** DPA 6.2 now lists `rank_score`, `served_from`, `intent_form_schema_version` and `pre_intent_recs` (**built**). Correction: `received_at` is an `orders_attributed` column, **not** an `events` column |
| G8 | `architecture.md` §8 | **RESOLVED 2026-10-19 (per 2026-10-18 commit "architecture §8 factual correction to match real schema through V15").** Ground truth: `orders_attributed` has `is_test` and `attribution_source` (V7); `events` has `pre_intent_recs` (built) but **not** `received_at` (that is an orders column) and **not** the budget fields (not built). `architecture.md` is not edited here (owned elsewhere; already fixed) |
| G9 | DPA 10.1; `sub-processors.md` rows 1 and 3 ("EU–EU. No third-country transfer") | Neon, Inc. and Render Services, Inc. are US-established. EU hosting does not by itself rule out transfers (remote support access, US-parent access, telemetry) | Check both vendors' DPAs for transfer clauses (SCCs / EU-US Data Privacy Framework); reword if needed. PO + attorney (A6) |
| G10 | `sub-processors.md` row 6 (CDN) | The widget bundle is currently shipped as a Shopify Theme App Extension asset (`extensions/wisegift-widget/assets/widget.js`), so it is served from Shopify's CDN, not a WiseGift CDN. The row's statement that CDN logs are produced "as its own controller" is doubtful for a vendor WiseGift engages | `devops-expert` confirms how the bundle is served in production; `security-and-privacy` rewrites the row; attorney view (A5) |
| G11 | `consent-position.md` §5 | Says server-side session and event rows are kept "for the tenant's active lifetime"; the decided period is 120 days. It also lists "our EU monitoring provider", while `sub-processors.md` row 8 says none is engaged at MVP. Basis line cites `docs/core/data.md`, which is pre-pivot | `security-and-privacy` |
| G12 | DPA 14.6 note; `consent-position.md` §5 note | Say the purge is "planned, not built". Per `decisions.md` 2026-10-16 the tenant purge exists in dry-run (PR #79); the 120-day / 18-month purge status is unclear | `backend-engineer` confirms status; `security-and-privacy` aligns notes. Purge out of dry-run is a pilot gate |
| G13 | DPA 7.1 | No purpose covers security / abuse logging, support, data-subject-request handling, or the support-run export, although WiseGift does all of them for the Merchant | Consider adding "(e) securing the Services and preventing abuse" and "(f) assisting the Merchant under clauses 12 and 14". Attorney to confirm (ties to A5) |
| G14 | DPA 4.2; `architecture.md` §10.3 | DPA says processing begins at install. Server-side shopper processing begins at go-live (Q-C8, PR #85), but today's widget bundle still writes `wg_session` / `_wg_session` for a non-live tenant before its first call (widget #24/#25 open) | Ship #24/#25; reword 4.2 to "catalogue processing begins at install; shopper processing begins at go-live" |
| G15 | `sub-processors.md` row 2 | Redis provider not selected; legacy Spring Session entries in Redis hold admin session data until Phase 2 | Select and list before the first pilot (`devops-expert`); Phase 2 removes the sessions |
| G16 | `sub-processors.md`, DPA 9 | No entry for the support mailbox provider or for export-file storage (B4) | PO chooses providers; `security-and-privacy` decides whether they are sub-processors (export storage: yes, if it holds shopper data) |
| G17 | DPA 6.1 / `consent-position.md` §6 | The pre-consent viewport bootstrap (PR #26) creates a server-side random page identifier inside a token. Not mentioned in DPA 6 | Add once PR #26 ships with its conditions met; depends on A2 |
| G18 | `cost_telemetry` | **Partially resolved 2026-10-19.** Confirmed from the backend: `cost_telemetry` holds **no session ID and no PII**, and no retention is enforced yet. **Still open:** PO decides a retention period (e.g. 18 months) |
| G19 | `CLAUDE.md` | Points to `docs/legal/legal.md`, which does not exist (already flagged 2026-10-14) | PO-owned; not edited here |

---

## D. Attorney questions (consolidated list for hand-over)

Each item is an **ATTORNEY QUESTION**. WiseGift's internal drafts take no
binding position on any of them.

| # | Topic | Question | Source |
|---|---|---|---|
| A1 | **Art. 28(10): fleet-wide recalibration** | `budget-tiers.md` §4.4 compiles budget-step metrics across Merchants (from per-Merchant aggregates with no session ID, offline, never shown to another Merchant) to recalibrate WiseGift's shared suggestion table. If each Merchant authorises this in DPA 7.1(d), does WiseGift remain a processor? Or does determining this purpose make WiseGift a controller for that step, needing its own legal basis, notice and an Art. 30(1) entry? Are the aggregates still personal data at all? | DPA 7.1(d) note; decisions 2026-10-17 |
| A2 | **Art. 5(3) ePrivacy: in-memory token** | Widget PR #26 fetches hook copy on viewport entry before any consent and returns a one-hour token bound to a random server-created page identifier, kept in page memory only (no storage, no cookie, no events). Given EDPB Guidelines 2/2023's broad reading of "storage" and "access", is this covered by the strictly-necessary exemption, or must the pre-consent request return copy only, with the token minted on first interaction? | `consent-position.md` §6; decisions 2026-10-17 |
| A3 | **Contact email: role, legal basis, notice** | (a) Confirm WiseGift is the controller of the Merchant contact email, not the Merchant's processor. (b) Which legal basis: Art. 6(1)(b), (c) or (f)? (c) Wording and URL of the merchant-contact privacy notice (Q-C4). (d) Confirm the permitted uses: DPA notices, billing, essential service notices, never marketing (Q-C3). (e) Must the previous address be kept as evidence of where a notice was sent, and for how long (Q-C5)? | Spec "Contact email step"; decisions 2026-10-16 |
| A4 | **Art. 9: free-text interests** | Visitors may type special-category data about themselves or a gift recipient into the free-text interests field. The text is **not stored** by WiseGift; it is sent **transiently** to a US-established processor (OpenAI; today, US processing) and otherwise reduced to a one-way hash in the cache key. It is not written to the order and not shared with the Merchant. Are a notice and data minimisation enough, or is a further measure needed (fixed vocabulary, filtering before transmission)? | `consent-position.md` §5 |
| A5 | **Role for infrastructure logs** | Are hosting request logs (IP, user agent) of widget traffic processed by WiseGift as controller (own security, Art. 6(1)(f)) or as processor for the Merchant? Same question for any CDN's logs, which `sub-processors.md` row 6 calls "its own controller". Should DPA 7.1 gain a security purpose (G13)? | This record B3; G10, G13 |
| A6 | **International transfers** | (a) Is the "no third-country transfer" statement defensible for Neon, Inc. and Render Services, Inc. (US-established, EU-hosted)? (b) For OpenAI today (US processing): which mechanism (SCCs Module 3, EU-US Data Privacy Framework if certified) and which supplementary measures? (c) After the switch to an OpenAI EU data-residency project, how should DPA 10.2 change, given OpenAI stays a US entity? | DPA 10.2 note; G9 |
| A7 | **Unattributed orders** | WiseGift stores filtered order rows for every order on a live storefront, including customers who never saw the widget, for 18 months, as baselines. Is this proportionate under Art. 5(1)(c) for the Merchant's purpose, and does the Merchant's privacy notice need to cover it? | This record A3 |
| A8 | **Legitimate-interest mode** | Is `consentModeRequired = false` (measurement without a consent signal where no consent interface exists) still defensible given the `_wg_session` join key? No tenant should use it until answered | `architecture.md` §7; `consent-position.md` §3 |
| A9 | **Art. 30 and Art. 37 scope** | Confirm the form and content of this record, that the Art. 30(5) exemption does not apply (processing is not occasional), and whether WiseGift must designate a DPO. Is an Art. 35 DPIA advisable for the free-text and order-linkage processing? | This record §0 |
| A10 | **Consent dependency on the Merchant's banner** | EU compliance of the widget depends on the Merchant enabling its cookie banner (staging finding 2026-10-15). Should the DPA place this responsibility on the Merchant explicitly, and is an onboarding checklist item enough? | decisions 2026-10-15 |

---

## E. Change log

| Date | Change |
|---|---|
| 2026-10-17 | First draft (security-and-privacy). Pending PO + attorney review |
| 2026-10-19 | Reconciled against backend ground truth (Flyway V1–V15): intent not stored (G1), no application IP/UA logging (G2), embedded Shopify session-token auth (G3), order fields + `attribution_source` built in V7 (G5), event fields + `pre_intent_recs` built, `received_at` is an orders column not events (G7/G8), `wg:rate:` key prefix, cache-key intent is a one-way hash, `cost_telemetry` has no PII (G18). DPA, consent-position and sub-processors corrected in the same change. Still pending PO + attorney review |

---

*This record is an informational draft prepared by WiseGift's internal
security-and-privacy function. It does not constitute legal advice. A
qualified attorney must review it before it is relied on, in particular
before the first DPA is signed with a paying Merchant.*
