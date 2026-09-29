# WiseGift — Specifications

> B2B rewrite. Supersedes the pre-2026-09-29 B2C spec, which described a consumer gift-discovery app on a shared affiliate catalog. Historical context is in `docs/decisions.md` (entries dated 2026-09-29).

## Vision

WiseGift sells **AI recommendation infrastructure to e-commerce merchants**, delivered as an embeddable frontend widget on the merchant's storefront. The widget hooks the shopper with a gift-framed question ("Are you looking for something for you or for someone else?"), captures intent, and calls WiseGift micro-APIs for personalised recommendations drawn from the merchant's own catalog. General recommendations for self-purchase are equally supported — the gift use case is the differentiator, not the whole surface.

**MVP platform: Shopify.** Follow-ons: VTEX, Salesforce Commerce Cloud, custom via REST.

---

## Product surface (MVP at a glance)

Three surfaces, one shared multi-tenant backend:

1. **Widget** — embedded JavaScript on the merchant's storefront (home / PDP / dedicated gift-finder page). Preact + Web Components, ≤ 50 KB gzipped, async, lazy.
2. **Merchant admin dashboard** — web app where the merchant installs, configures placements, edits intent-form copy, and sees the analytics dashboard. Hosted by WiseGift.
3. **Recommendation & event APIs** — the micro-APIs the widget calls to fetch recs and emit events. Same APIs power any future platform integration.

---

## Personas

### Merchant admin
The person at the merchant who installs the Shopify app, configures the widget, and reads the analytics. Usually the e-commerce manager or the growth/marketing lead. Not a developer.

### Widget shopper
Any visitor to the merchant's storefront who sees or interacts with a WiseGift widget. Anonymous by default — WiseGift never receives the shopper's identity.

---

## Platform scope

| Platform | MVP | Post-MVP |
|---|---|---|
| Shopify (App Store public app + Theme App Extension) | ✓ | — |
| Shopify (custom app install for pilots) | ✓ | — |
| VTEX | — | ✓ |
| Salesforce Commerce Cloud | — | ✓ |
| Custom / headless via REST | — | ✓ |

---

## Feature: Merchant onboarding — Shopify app install

### User Story
As a merchant admin, I want to install WiseGift from the Shopify App Store (or as a custom app during pilot) so that I can start showing gift recommendations on my storefront without engineering work.

### Acceptance Criteria
- [ ] Public App Store listing includes app icon, screenshots, pricing tiers, and a data-processing summary.
- [ ] Install flow uses standard Shopify OAuth; the merchant approves the requested scopes (`read_products`, `read_orders` for attribution, `write_theme_extensions` for widget injection).
- [ ] On successful OAuth, a `tenant` record is created in the shared WiseGift database with `tenant_id`, shop domain, Shopify shop ID, install timestamp, region (EU), and a placeholder vertical (defaults to `generic`).
- [ ] The merchant is redirected to the WiseGift admin dashboard's onboarding wizard.
- [ ] During the pilot phase, a merchant can install as a **custom app** created in their Shopify partner dashboard using the same OAuth flow — no App Store listing required.
- [ ] Uninstall via Shopify triggers WiseGift's `app/uninstalled` webhook; tenant is soft-deleted (retained 30 days for reactivation, then hard-purged with all shopper events).
- [ ] All data written during install is stored in the EU region.

---

## Feature: Merchant onboarding — Onboarding wizard

### User Story
As a merchant admin, I want a guided setup after install so that the widget is live on my storefront within 15 minutes without me needing to read documentation.

### Acceptance Criteria
- [ ] Step 1 — **Vertical selection**: the merchant picks their primary vertical from a fixed list (Fashion, Beauty, Home, Tech, Books, Food, Kids, Jewelry, Other). MVP treats all verticals identically at the intent-form level, but the choice is stored on the tenant record for future verticalised packs.
- [ ] Step 2 — **Catalog sync**: WiseGift kicks off the initial catalog import from the Shopify Admin API. A progress indicator shows product count ingested. First-time sync of up to 10k SKUs must complete in under 15 minutes; larger catalogs continue in the background and the wizard proceeds.
- [ ] Step 3 — **Widget placement**: the merchant picks at least one placement (Home hero, PDP recommendation slot, dedicated gift-finder page). Each placement is a Theme App Extension block the merchant enables in the Shopify theme editor via a "Configure in Shopify" deep link.
- [ ] Step 4 — **Preview**: an inline preview of the widget rendered against the merchant's own theme colours and typography. The merchant can adjust the intent-hook copy from a preset library (5 options in EN + ES + PT) or write custom copy.
- [ ] Step 5 — **Go live**: enabling the tenant flips `tenant.active = true`. From this moment the widget renders live for shoppers and events start being recorded.
- [ ] The onboarding wizard is resumable — closing the browser and returning drops the merchant back at the incomplete step.

---

## Feature: Merchant onboarding — Catalog sync

### User Story
As a merchant admin, I want my product catalog to stay in sync with WiseGift automatically so that recommendations always reflect what's actually available in my store.

### Acceptance Criteria
- [ ] Initial full-catalog import at install using Shopify Admin API `products.json` (paginated, respecting Shopify's rate limits).
- [ ] For each product WiseGift stores: `tenant_id`, `shopify_product_id`, `handle`, `title`, `description`, `price`, `currency`, `image_url`, `availability`, `product_type`, `tags`, `variants[]` (id, price, availability), `last_seen_at`.
- [ ] Product embeddings are computed at ingest for each product (title + description + product_type + tags). Vector column on Neon pgvector.
- [ ] The following Shopify webhooks are subscribed and processed within 60 seconds: `products/create`, `products/update`, `products/delete`, `inventory_levels/update` (for availability).
- [ ] On `products/update` or `products/delete` the affected product row is updated; embedding is recomputed only when the description or title changes.
- [ ] On `inventory_levels/update` the `availability` field is updated but embedding is untouched.
- [ ] Unavailable products (all variants out of stock or archived) are excluded from recommendation candidates but retained in the DB.
- [ ] Nightly reconciliation job re-fetches the catalog and diffs against the DB to catch any webhook that was missed.
- [ ] Catalog size at any time is visible to the merchant on the admin dashboard.
- [ ] Merchant-facing sync controls and visibility (manual resync triggers, sync-status panel, webhook health) live in the **Merchant admin — Catalog sync controls** feature.

---

## Feature: Widget — Embed and lifecycle

### User Story
As a widget shopper, I want the widget to load without slowing down the merchant's page so that my browsing experience is not disrupted.

### Acceptance Criteria
- [ ] Widget bundle is served from a WiseGift-controlled CDN with cache headers that permit long TTL + hash-based cache-busting.
- [ ] The Theme App Extension block includes a single `<script async src="…/widget.js">` tag (or Shopify's block-native equivalent). No render-blocking assets.
- [ ] Total gzipped payload of `widget.js` is ≤ **50 KB**.
- [ ] The widget renders inside a `<wisegift-widget>` **Web Component with Shadow DOM** — all styles are isolated from the merchant's theme.
- [ ] Time-to-interactive of the widget (from script fetch complete → widget accepts user input) is < **100 ms** at p95.
- [ ] Recommendation content is **lazy** — no `/recommendations` API call is made until either (a) the widget scrolls into the viewport (IntersectionObserver) or (b) the shopper interacts with the intent hook.
- [ ] The widget passes a Lighthouse Performance audit on a representative merchant page with score ≥ 90 with the widget enabled vs. without.

---

## Feature: Widget — Gift intent hook

### User Story
As a widget shopper, I want a friendly question that helps the widget understand whether I'm shopping for myself or a gift so that the recommendations match my intent.

### Acceptance Criteria
- [ ] The widget renders a hook prompt (default: "Are you looking for something for you, or for someone else?") with two primary buttons: **For me** and **For someone else**.
- [ ] Merchant admins can override the prompt copy from a preset library or with custom text (Merchant admin — Widget configuration feature).
- [ ] Selecting **For me** routes the shopper to the general-recommendations intent flow (budget + interests optional; can also submit with no additional input on a PDP-context placement).
- [ ] Selecting **For someone else** routes to the gift-intent flow (recipient relationship, occasion, budget, optional interests).
- [ ] Both flows share the same downstream API — the intent object carries an `intent_mode: "self" | "gift"` field.
- [ ] The hook is skippable on PDP placements: if the shopper does nothing, the widget still shows recommendations based on the current product context after 3 seconds (fires a widget-shown event but no intent capture).
- [ ] Widget copy is available in EN, ES, PT at MVP.

---

## Feature: Widget — Intent capture form (generic MVP)

### User Story
As a widget shopper, I want a short form to describe what I'm looking for so that recommendations are relevant to me or my gift recipient.

### Acceptance Criteria
- [ ] **Self-intent form** fields (all optional): budget (min/max), interests (free-text chips, max 5).
- [ ] **Gift-intent form** fields: recipient relationship (Partner / Family / Friend / Colleague / Other — required), occasion (Birthday / Anniversary / Wedding / Baby / Christmas / Housewarming / Just because / Other — required), budget min/max (required), interests (free-text chips, max 5, optional).
- [ ] The form is a single scrollable card, not a multi-step wizard — form completion in < 20 seconds is the design target.
- [ ] The form's field schema is stored per tenant and versioned (`intent_form_schema_version` on the tenant record) so verticalised packs (post-MVP) can be introduced without breaking existing widgets.
- [ ] Submit button is disabled until required fields are filled; validation is inline.
- [ ] Submitting fires the recommendation call and shows a loading state; recommendations appear in < 1 second at p95.

---

## Feature: Widget — Recommendation display

### User Story
As a widget shopper, I want to see a small set of relevant product recommendations so that I can quickly decide what to buy.

### Acceptance Criteria
- [ ] Recommendations render as a horizontal scrollable strip of product cards on desktop; vertical stack on mobile.
- [ ] Each card shows product image, title, price, and a **View product** CTA that navigates to the merchant's own product page (`/products/{handle}`) in the same tab.
- [ ] Default set size is 5 products. Configurable per placement (3–10) by the merchant.
- [ ] If fewer than the requested number of candidates pass the ranking threshold, the widget shows the products it has (never pads with irrelevant items).
- [ ] Click on a product card fires a `product-clicked` event before navigating.
- [ ] If the recommendation call fails or returns zero results, the widget renders a graceful fallback (top-N merchant bestsellers or precomputed cold recs for the current PDP), never an error message visible to the shopper.

---

## Feature: Widget — Attribution & session tracking

### User Story
As a merchant admin, I want the widget's impact on my revenue to be measurable so that I can trust WiseGift's lift claims.

### Acceptance Criteria
- [ ] On first widget render for a browser, the widget generates an **anonymous session ID** (UUID v4, 30-day sliding TTL) and stores it in localStorage under a WiseGift-scoped key. No merchant customer identity is attached.
- [ ] The session is deterministically assigned to **exposed** or **holdout** on first render: 90% exposed, 10% holdout. Assignment is stable across placements and page loads for the session lifetime.
- [ ] Holdout sessions see the widget in a **placebo state**: the hook and form render identically, but the recommendation slot is either hidden or filled with a merchant-configured static fallback (never WiseGift-generated recs). Placebo is invisible to the shopper.
- [ ] Every widget interaction fires a typed event to the WiseGift event API: `widget_shown`, `widget_engaged`, `intent_submitted`, `product_clicked`, tagged with `tenant_id`, `session_id`, `placement`, `intent_mode`, `is_holdout`, timestamp.
- [ ] The `orders/create` Shopify webhook is received by the WiseGift backend for every merchant order. WiseGift extracts only `order_id`, `line_items[].shopify_product_id`, `line_items[].quantity`, `total_price`, `currency`, and correlates to a session via a client-side widget-planted marker (a `wg_session=<id>` query param appended to product URLs on click, or Shopify cart attribute if available). Customer PII fields are dropped at ingest and never persisted.
- [ ] Attribution report joins events + orders on `session_id` within a **7-day attribution window** from the last `product_clicked` event.
- [ ] Merchant dashboard renders KPIs against the holdout (see Analytics dashboard feature).

---

## Feature: Widget — Fallbacks and offline behaviour

### User Story
As a widget shopper, I should never see a broken widget so that my confidence in the merchant is not undermined.

### Acceptance Criteria
- [ ] If the `widget.js` script fails to load, the Theme App Extension block renders nothing (empty container). The merchant's page layout is unaffected.
- [ ] If the recommendation API times out (> 2 seconds), the widget serves precomputed cold recs from local widget cache (populated on previous session) or hides the recommendation slot.
- [ ] If the tenant's kill switch is triggered (see Cost guardrails), the widget renders precomputed cold recs only, with no live LLM calls. Shoppers cannot tell.

---

## Feature: Merchant admin — Widget configuration

### User Story
As a merchant admin, I want to control what the widget says and where it appears so that it matches my brand.

### Acceptance Criteria
- [ ] Configuration lives in the WiseGift admin dashboard under Widget → Configuration.
- [ ] The merchant can edit the **hook prompt copy** (up to 200 characters, or pick from 5 language-localised presets).
- [ ] The merchant can toggle the **intent form fields**: relationship, occasion, budget, interests are individually toggleable (required-vs-optional-vs-hidden), with defaults matching the MVP form.
- [ ] The merchant can set the **number of recommendations** shown per widget instance (3–10, default 5).
- [ ] The merchant can pick brand accent colour and border radius; other styling matches the storefront theme via inherited CSS variables.
- [ ] Save persists to the tenant's widget config; changes propagate to live widgets within 5 minutes (config is fetched with the widget bundle and cached).
- [ ] All fields are inline-validated.

---

## Feature: Merchant admin — Placements

### User Story
As a merchant admin, I want to see which placements are live and manage them so that I can experiment with where the widget appears.

### Acceptance Criteria
- [ ] The admin dashboard's Placements screen lists each configured placement (Home hero / PDP slot / Gift-finder page) with status (enabled / disabled), the theme location, and links to the Shopify theme editor for that block.
- [ ] The merchant can enable or disable a placement; disabled placements do not render the widget on the storefront.
- [ ] Each placement carries an independent set of KPIs on the analytics dashboard.
- [ ] Adding a new placement follows the same "Configure in Shopify" deep-link flow used in onboarding.

---

## Feature: Merchant admin — Catalog sync controls

### User Story
As a merchant admin, I want visibility into how my catalog is syncing and the ability to trigger a resync when I need one so that I trust the widget is recommending what is actually in my store.

### Acceptance Criteria
- [ ] A **Catalog sync** panel in the admin dashboard shows: total product count in WiseGift, timestamp of last full sync, timestamp of last webhook received per topic (`products/update`, `inventory_levels/update`, etc.), and a webhook-health indicator (OK / degraded / not receiving) derived from time-since-last-webhook thresholds.
- [ ] A **Refresh prices & stock** button re-pulls variant price and availability from Shopify Admin API without recomputing embeddings. Runs asynchronously with a progress indicator. Rate-limited to one run per tenant per hour.
- [ ] A **Full resync** button re-fetches the full catalog and re-embeds only products whose `content_hash` changed (title / description / tags). Runs asynchronously with a progress indicator. Rate-limited to one run per tenant per 24 hours, with a "Contact support to run more often" affordance.
- [ ] Both buttons disable while a sync is in progress and re-enable on completion; failures show a short error and a "Retry" affordance.
- [ ] Manual syncs are counted against the same Shopify per-app rate-limit budget as automatic syncs — a large-catalog manual run cannot starve other tenants (per `data.md` §3 rate-limit posture).
- [ ] Per-tenant sync cadence (nightly reconciliation frequency) is NOT configurable at MVP; it is fixed for all tenants. Configurable cadence is post-MVP.
- [ ] Per-product force-refresh is NOT exposed in the merchant UI at MVP; it is available support-only via a backend admin path.

---

## Feature: Merchant admin — Analytics dashboard (minimal MVP)

### User Story
As a merchant admin, I want to see the widget's impact on my store in one screen so that I know whether it's worth keeping.

### Acceptance Criteria
- [ ] Default date range: last 30 days. Configurable ranges: 7d, 14d, 30d, 90d.
- [ ] KPI tiles, each shown as `exposed vs holdout` with lift percentage and a significance flag (green if p < 0.05, grey if not enough data yet):
  - **Widget CTR** (widget_engaged / widget_shown)
  - **Add-to-cart rate on recommended products** — sessions where a product-clicked event was followed by a Shopify `carts/update` including that product within 15 minutes.
  - **Conversion rate** on sessions that saw the widget (order within 7-day attribution window / widget_shown sessions).
  - **AOV** for orders from widget-exposed sessions vs holdout.
  - **Revenue per session** (attributed order revenue / widget_shown sessions).
- [ ] A small line chart per KPI shows daily trend over the selected range.
- [ ] A minimum-data callout displays if any KPI has fewer than 5,000 exposed sessions in the selected range ("Not enough data yet — need N more exposed sessions").
- [ ] Data is refreshed hourly; a timestamp shows last refresh.
- [ ] All numbers are export-to-CSV.
- [ ] No cohort analysis, funnel drill-down, or per-recommendation breakdown at MVP — those are post-MVP.

---

## Feature: Merchant admin — Account & billing

### User Story
As a merchant admin, I want to see my plan and usage so that I know when I'm approaching my limits.

### Acceptance Criteria
- [ ] The Account screen shows: plan name (Pilot / Starter / Growth / Scale), monthly recommendation budget included in the plan, current-month recommendations served, days until reset.
- [ ] A visual progress bar shows current usage vs. plan cap. Colour changes to warning at 80% and to error at 100%.
- [ ] Billing is handled via Shopify's built-in App Store billing API for public-app installs (recurring subscription with usage-based overage). Pilot tenants show "Pilot — free through YYYY-MM-DD" with no billing hooks.
- [ ] If the merchant hits their monthly cap, the widget serves precomputed cold recs only (no live LLM) and the admin dashboard shows a persistent banner with an upgrade CTA.
- [ ] Merchant can cancel or downgrade from within Shopify (App Store billing is authoritative).

---

## Feature: Recommendation API — `POST /v1/recommendations`

### User Story
As the widget, I want to fetch personalised recommendations for the current shopper's intent so that I can display relevant products.

### Acceptance Criteria
- [ ] Endpoint: `POST /v1/recommendations`. Auth: tenant public key + short-lived signed request token issued to the widget bundle (rotates hourly).
- [ ] Request body: `session_id`, `placement`, `intent_mode`, `intent` (relationship, occasion, budget_min, budget_max, interests[]), `context` (current `shopify_product_id` if on a PDP), `limit` (default 5).
- [ ] Response body: `recommendations[]` (each: `shopify_product_id`, `handle`, `title`, `price`, `image_url`, `rank_score`), `cache_hit` (boolean), `served_from` (`live_llm` | `cache` | `precomputed` | `fallback`).
- [ ] The endpoint enforces the **per-tenant hard usage cap** — over-cap requests are served from precomputed cold recs, cache hit is set to true, `served_from = precomputed`.
- [ ] The endpoint enforces the **per-session rate limit** (max 20 live-LLM calls per session per hour); over-limit requests are served from cache/precomputed.
- [ ] p95 latency budget: **300 ms** end-to-end.
- [ ] Every response emits cost telemetry (tenant, placement, model, tokens in/out, cache hit/miss, latency) to the ops metrics stream.
- [ ] Errors return a fallback payload with `served_from = fallback` and 200 OK — the widget must never render an error state to the shopper.

---

## Feature: Event API — `POST /v1/events`

### User Story
As the widget, I want to record shopper interactions so that attribution and analytics are accurate.

### Acceptance Criteria
- [ ] Endpoint: `POST /v1/events`. Same auth as the recommendation API.
- [ ] Accepts a batch of events (`events[]`), each: `event_type` (`widget_shown` | `widget_engaged` | `intent_submitted` | `product_clicked`), `session_id`, `placement`, `intent_mode`, `is_holdout`, `timestamp`, optional `shopify_product_id`, optional `rank_score` and `served_from` (echoed from the rec response).
- [ ] Events are validated against a strict schema; unknown fields are rejected (fail fast on client bugs).
- [ ] Events are enqueued and processed asynchronously into the analytics store. p95 write latency < 100 ms.
- [ ] The endpoint is fire-and-forget from the widget's perspective — no retry storms if a request fails.
- [ ] Batching: widget can send up to 10 events per request; widget batches events with a 1-second flush interval.

---

## Feature: Webhook receiver — Shopify order attribution

### User Story
As the WiseGift backend, I want to record order events to attribute revenue lift so that merchant dashboards show accurate KPIs.

### Acceptance Criteria
- [ ] HMAC signature verification against Shopify's shared secret; invalid signatures rejected with 401.
- [ ] Idempotency: repeated webhooks for the same `order_id` are deduplicated.
- [ ] From the order payload we retain only: `order_id`, `line_items[].shopify_product_id`, `line_items[].quantity`, `line_items[].price`, `total_price`, `currency`, `created_at`, `tenant_id` (derived from shop domain).
- [ ] All customer PII fields (`customer`, `billing_address`, `shipping_address`, `email`, `phone`, `client_details`) are dropped at ingest and never persisted, logged, or forwarded.
- [ ] Correlation to a widget session is done via a `wg_session` cart attribute or query-string marker planted by the widget on product-click; unattributed orders are still stored (to compute merchant-wide baselines) but do not contribute to per-session lift.
- [ ] Webhook processing p95 < 500 ms.

---

## Feature: Multi-tenancy — Isolation model

### User Story
As the platform operator, I want strict tenant isolation so that no cross-tenant data leak is possible even in the presence of a bug in application code.

### Acceptance Criteria
- [ ] Every table containing tenant data carries a `tenant_id UUID NOT NULL` column; a composite index `(tenant_id, primary_key)` is present.
- [ ] Every repository method takes a `tenant_id` argument or reads it from a request-scoped context; a lint rule flags any raw SQL that does not include `WHERE tenant_id = ?`.
- [ ] Integration tests seed two tenants, run queries as tenant A, and assert zero rows from tenant B are returned. Rerun in CI on every PR.
- [ ] Every recommendation call and event write is tenant-scoped from the API layer down through embeddings retrieval to the analytics store.
- [ ] Uninstall soft-deletes for 30 days, then a scheduled job hard-purges all tenant-scoped rows (products, embeddings, events, orders, config).

---

## Feature: Cost guardrails (engineering requirements)

### User Story
As the founder, I want the AI cost of running the product to be bounded regardless of merchant traffic so that a pilot can never generate a bill I cannot pay.

### Acceptance Criteria
- [ ] **Per-tenant hard usage cap** — configurable, default 50k live-LLM recommendation calls per calendar month during pilot. Over-cap requests are served from precomputed cold recs. Enforced at the recommendation API layer with a Redis-backed counter.
- [ ] **Response cache** — 24-hour TTL on live recommendations, keyed by `(tenant_id, intent_signature, context_signature)`. Expected hit rate > 60% at steady state.
- [ ] **Nightly precomputed cold recs** — a per-tenant nightly job produces top-N recommendations per SKU without LLM calls (vector retrieval + heuristic ranking only). These serve as PDP-context recommendations when the shopper provides no intent signal, and as the fallback when the live path is unavailable or over-cap.
- [ ] **Model selection** — Claude Haiku 4.5 is the default for re-ranking and intent parsing. Escalation to Sonnet is behind a per-tenant feature flag and used only for full-form gift-intent flows with a rich intent object.
- [ ] **Session-level rate limit** — max 20 live-LLM calls per widget session per hour.
- [ ] **Per-tenant kill switch** — daily-spend threshold triggers an ops alert and flips the tenant to precomputed-only mode until manually reset. Configurable per plan tier.
- [ ] **Cost telemetry** — every rec call is tagged with `tenant_id`, `placement`, `model`, `input_tokens`, `output_tokens`, `cache_hit`, `served_from`, `latency_ms`. Dashboard shows cost per merchant per day.

---

## Data & privacy

### What we store
- **Per tenant:** identity (shop domain, Shopify shop ID, region, plan), catalog snapshot with embeddings, widget config, placement config, feature flags.
- **Per session:** anonymous session ID, exposed/holdout assignment, events (widget_shown, widget_engaged, intent_submitted, product_clicked), intent submissions (recipient relationship, occasion, budget, interests — no free-text PII).
- **Per order:** filtered order payload — `order_id`, line items (product IDs, quantities, prices), total, currency, timestamp, tenant. No customer identifiers.

### What we never store
- Merchant customer names, emails, phone numbers, addresses, IPs.
- Full order payloads with PII (filtered at ingest, before persistence).
- Cross-merchant browsing behaviour or profiles.
- Anything gathered from a source other than the widget interaction, the merchant's Shopify APIs (catalog + orders), or the merchant admin's own inputs.

### Region
All storage and compute in **EU regions** at MVP (Neon EU, application hosting in EU). US region considered only when a US pilot is signed.

### Data-processing role
WiseGift is a **data processor** for the merchant's shopper data (anonymous session events + filtered order records). A **DPA is required** for every merchant, including pilots. Template owned by `privacy-legal-advisor`.

---

## Roadmap — post-MVP (in likely priority order)

1. **Verticalised intent parameter packs** — per-vertical intent forms (Fashion / Tech / Books / Beauty / Home) with merchant-editable style archetypes and reference imagery. Justifies the Growth-tier price step. See `docs/decisions.md` 2026-09-29 entry.
2. **Purchase-outcome learning loop** — the event schema is MVP-ready; the learner (contextual bandit / preference model) lands here to compound per-tenant lift over time.
3. **Analytics — deeper cuts** — per-placement funnel, per-intent-mode conversion, cohort analysis, exportable reports.
4. **VTEX and SFCC integrations** — same core APIs, new adapter modules for catalog sync and order webhooks.
5. **Custom / headless integration** — publish a REST integration guide and a signed-request SDK for merchants on non-supported platforms.
6. **Admin — A/B experiments UI** — let merchants A/B test alternate hook copy, form field sets, and rec-slot sizes with attribution baked in.
7. **Behavioural signals (opt-in)** — if a merchant opts in and has consent-banner infrastructure, ingest cross-page browsing signals for stronger cold-start recs. Not before a legal review.
8. **B2C storefront revival (parked Flutter app)** — resurrect the Flutter consumer app to browse participating merchants' catalogs on a WiseGift-branded surface. Requires per-merchant commercial deal. Not before at least 20 paying merchants.

---

## Out of MVP scope (explicit)

- iOS / Android native mobile apps for merchants.
- Multi-store roll-ups for merchants running several Shopify stores under one brand.
- Free forever tier or self-serve trial without a pilot conversation.
- Rev-share pricing model.
- SSO / SAML for merchant admin login (email + password + Google OAuth are enough at MVP).
- Multi-user seats per merchant tenant (single-user admin at MVP).
- Public API for merchants to hit recommendation endpoints from server-side code (widget-only at MVP).
- Any behavioural tracking beyond widget interactions and the single order-confirmation webhook.

---

## Open questions

- **Language coverage** — MVP ships widget copy in EN, ES, PT. Do we also need DE / FR / IT before the first pilot with a non-Iberian merchant? Route: `functional-analyst` + `marketing-manager`.
- **Widget consent for EU** — some legal regimes require explicit consent for setting a first-party session identifier even without cross-site tracking. Do we need a consent-mode integration with the merchant's cookie banner from day one? Route: `privacy-legal-advisor`.
- **App Store category** — do we list under Marketing → Upselling & Cross-selling, or Store Design → Product Discovery? Positioning matters for organic discovery. Route: `marketing-manager` + `platform-integrations-expert` (proposed agent).
- **Custom-app pilot billing** — during the pilot, merchants install as custom apps (bypassing App Store review). Shopify's App Store billing is not available to custom apps. Do we bill pilot-converted merchants through Stripe until they migrate to the App Store version, or migrate them to the public app at conversion? Route: `devops-expert` + `platform-integrations-expert`.
- **Merchant-side data-clean-room requests** — if a merchant asks for their tenant's raw event data as an export, what SLA and format? Route: `functional-analyst`.

---

## Resolved questions

*(To be populated as MVP scope questions are closed. The seven pivot-era scope decisions are captured in `docs/decisions.md` entry dated 2026-09-29 "Frozen MVP scope".)*
