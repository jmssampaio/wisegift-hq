# WiseGift — Data model (B2B recommendation platform)

> B2B rewrite. Supersedes `docs/archive/data-b2c-pre-pivot.md`. That doc
> described a global affiliate catalog with universal + per-merchant
> giftability rules and cross-merchant fingerprint dedup — none of that
> applies here. Cross-links:
> [spec](../product/spec.md), [architecture](../engineering/architecture.md),
> [cost model](../growth/gtm-b2b.md), [decisions](../decisions.md) (2026-09-29).
>
> This is a reasoning doc. Table shapes and column names are authoritative in
> `architecture.md` Section 8; do not duplicate them here — reference and
> justify.

---

## 1. Framing

The whole data model exists to serve one live-path decision: **for this
tenant, for this shopper's intent, on this placement, return N products from
this tenant's own catalog inside 300 ms — or fall back to something
precomputed for this tenant that is not embarrassing.**

That collapses to four data surfaces:

1. **Per-tenant catalog** — the merchant's own products, embedded, kept
   fresh.
2. **Per-shopper session + event stream** — attribution today, learning loop
   tomorrow, same schema.
3. **Filtered order webhook** — the only outcome signal we accept, PII-free.
4. **Cost + guardrail state** — how the platform stays solvent per tenant.

Everything else (widget config, intent-form versioning, kill-switch state)
falls out of these four.

---

## 2. Load-bearing invariants

These are the invariants every table, every query, and every ingest path has
to honour. They are non-negotiable and any violation is a P0.

- **Every row that describes tenant data carries `tenant_id UUID NOT NULL`**
  and is indexed with a composite index leading with `tenant_id`. Applies to
  catalog rows, embeddings, precomputed recs, sessions, events, attributed
  orders, cost telemetry, intent-form schema versions, widget config. Only
  `tenants` itself and platform-side OAuth records key off `tenant_id` as PK
  rather than carrying it (they *are* the tenant identity).
- **Tenant isolation is enforced at the repository layer, not the query
  layer.** Every repository method takes `tenant_id` explicitly or reads it
  from a request-scoped `TenantContext`. An architecture test rejects raw
  SQL that is missing `WHERE tenant_id = ?`. Integration tests seed at least
  two tenants and assert zero cross-leakage on every read path.
- **No shopper PII, ever.** The widget generates its own anonymous session
  ID; the order webhook is filtered *at ingest*, before persistence, before
  any log write. The set of accepted order fields is fixed and small
  (see §4). Everything else is dropped in the receiver.
- **Embeddings are computed at ingest, never at serve time.** Serve-time
  cost is bounded by retrieval + rerank, not by embedding round trips.
- **Region is EU-only at MVP.** Neon EU, Redis EU, hosting EU, telemetry EU.
  US region requires an explicit PO decision + DPA update.

---

## 3. Per-tenant catalog

**Owner:** Catalog module. **Shape:** `tenant_products` +
`tenant_product_variants` in `architecture.md` §8. **Ingestion source:**
Shopify Admin API + product webhooks. **Post-MVP:** VTEX and SFCC land
behind the same `PlatformCatalogSource` port; the DB shape does not change.

### What we store per product

The set from `architecture.md` §8: identity (`shopify_product_id`, `handle`),
merchandising text (`title`, `description`, `product_type`, `tags`), price
range across variants (`price_min`, `price_max`, `currency`), media
(`image_url`), availability rollup (`available` — true if any variant in
stock), a `content_hash` that drives embed-recompute, and the pgvector
`embedding` + `embedding_model_version` pair. Variant-level price and
availability live in the child table so the parent row does not churn on
inventory-only updates.

Deliberately NOT stored: SEO handle history, per-variant option metadata
(size/colour breakdowns), inventory location, per-market pricing, Shopify
metafields. The MVP ranking use case is served by title / description /
tags / price band. Anything else is future scope and adding it early is
noise for isolation review.

### Why per-tenant, not shared

The archived B2C model had a single global `products` table with a
`country` filter and a two-layer giftability filter. In B2B, the merchant's
own catalog IS the scope — they have already curated what they sell, in the
regions they sell it. There is no cross-tenant recommendation surface, so
there is no reason to physically colocate two merchants' rows. Isolation is
easier when they simply do not share a row space.

### Ingestion path

Three entry points, all idempotent per tenant:

1. **Initial full sync** at Shopify OAuth completion (spec: Merchant
   onboarding → Catalog sync). Paginated `products.json` fetch, batched
   normalise → embed → upsert. First-time sync of up to 10k SKUs completes
   in under 15 minutes; larger catalogs continue in the background while
   the wizard proceeds.
2. **Deltas via webhook** — `products/create`, `products/update`,
   `products/delete`, `inventory_levels/update`. Handled inside 60 s at p95.
3. **Manual sync triggered by the merchant admin** (spec: Merchant admin →
   Catalog sync controls). Two shapes:
   - **Refresh prices & stock** — re-pulls variant `price` and
     `available` only. Skips the embed step entirely (no content change,
     no vector recompute). Rate-limited per tenant to one run per hour.
   - **Full resync** — same code path as (1), reusing `content_hash` to
     avoid re-embedding unchanged rows. Rate-limited per tenant to one
     run per 24 h.

All three paths converge on a small pipeline of single-responsibility steps
(fetch → parse → filter → normalise → embed if needed → upsert). A row
failing one step is skipped and counted (`rejection_reason` counter tagged
by `tenant_id`), not fatal to the batch. Batches commit per row or per
small batch — no long-held transactions across a paginated catalog fetch.
Manual runs share the same per-tenant token bucket and fleet-wide 429
guard as automatic syncs (see **Rate limit posture** below), so a
large-catalog manual resync cannot starve other tenants.

Idempotency comes from `(tenant_id, shopify_product_id)` unique and from
`content_hash`. A rerun with unchanged content produces the same DB state
and skips the embed call. `last_seen_at` is bumped on every touch; the
nightly reconciliation job re-fetches the catalog and diffs, catching any
webhook we missed and deactivating products whose `last_seen_at` has
fallen behind.

### HMAC + dedup on webhooks

Every Shopify webhook is HMAC-verified against the per-app shared secret
before any parsing. Invalid signatures return 401 and are logged with the
shop domain and event topic — never with payload contents. Webhook
dedup on the receiver: Shopify's `X-Shopify-Webhook-Id` header (or the
`(shop_domain, topic, resource_id, updated_at)` tuple as a fallback) is
kept for a bounded window in Redis; repeats are ack'd 200 and dropped.

### Rate limit posture

Shopify enforces a leaky-bucket rate limit (2 req/s baseline, 40 burst)
per app-per-shop. A large-catalog tenant's full sync must not starve
smaller tenants sharing the app credentials. Concretely: the fetcher runs
per tenant with a per-tenant token bucket, and the app-wide budget is
shared with a small guard that pauses per-tenant fetch when the fleet-wide
error rate on `429` climbs. Details of the throttle live in the ingestion
PR, not this doc.

### Deactivation policy (settled 2026-09-29)

A product is marked `available=false` after **2 consecutive successful
reconciliation runs** in which the product was not observed. A run is
"successful" only if `catalog_sync_runs.status='ok'` and
`products_seen_count` is within ±10 % of the last known catalog size.
Failed or partial runs are no-ops for this counter — a crashed nightly
job or a Shopify 5xx cannot self-inflict a fleet-wide deactivation.

Deactivated rows remain in the DB; the §7 30-day soft-delete window
governs hard removal.

`catalog_sync_runs` is a small operational table
(`tenant_id`, `started_at`, `finished_at`, `status`, `products_seen_count`,
`pages_fetched`) written once per reconciliation run. It also backs the
webhook-health indicator in the admin sync-status panel (`spec.md` →
Merchant admin → Catalog sync controls).

---

## 4. Filtered order webhook

**Owner:** Events module (spec: Webhook receiver — Shopify order attribution).
**Shape:** `orders_attributed` in `architecture.md` §8.

We accept `orders/create` from Shopify and immediately drop everything we
do not need. The set we keep is exactly:

- `order_id` (the Shopify order ID, primary key)
- `line_items[]` — only `{shopify_product_id, quantity, price}` per line
- `total_price`, `currency`
- `received_at` (server-side timestamp, not the shopper's clock)
- `tenant_id` (derived from `shop_domain`)
- `session_id` (nullable — set only if the widget planted a correlator)

We drop, in the receiver, before any log line or persistence:
`customer`, `billing_address`, `shipping_address`, `email`, `phone`,
`client_details`, discount codes with shopper-identifying names, any
Shopify note attributes we did not set ourselves. The receiver is the only
place these fields ever exist in memory, and only for the milliseconds it
takes to build the filtered record. This is codified in the Frozen MVP
scope entry (2026-09-29) and is a P0 to regress on.

Idempotency: repeated deliveries of the same `order_id` for the same
tenant are dedup'd on the PK. Unattributed orders (no `session_id`) are
still stored — they contribute to the merchant-wide baseline but never to
per-session lift.

Session correlation is a widget concern (`?wg_session=<id>` on product
links or a Shopify cart attribute — see `architecture.md` §7). The data
model just receives whichever the widget managed to plant.

---

## 5. Widget event schema

**Owner:** Events module (spec: Widget → Attribution & session tracking,
Event API). **Shape:** `sessions`, `events` in `architecture.md` §8.

Four event types emitted from day one:

- `widget_shown` — the widget rendered in the viewport for this session +
  placement.
- `widget_engaged` — the shopper interacted with the hook or the intent
  form.
- `intent_submitted` — the intent form was submitted. Carries an explicit
  `intent_form_schema_version` column on the event row (settled 2026-09-29)
  so the v2 learning loop can read historical payloads across schema
  upgrades without a backfill. Payload shape described by the tenant's
  active `intent_form_schema` — see §8.
- `product_clicked` — a recommendation card was clicked; carries the
  clicked `shopify_product_id`, its `rank_score`, and the `served_from`
  echoed from the rec response.

Every event carries `tenant_id`, `session_id`, `placement`, `intent_mode`,
`is_holdout` (denormalised from `sessions` for read speed), and
`occurred_at`. Nothing else. There is no free-text field; the intent
payload lives on the `intent_submitted` row, versioned by intent-form
schema.

### Why these four, and why now

Two independent consumers both need this event stream:

1. **Attribution today** — the analytics dashboard computes the five KPIs
   (widget CTR, add-to-cart rate on recommended products, conversion rate
   on engaged sessions, AOV, revenue per session) as
   `exposed vs holdout` with significance, against a 7-day attribution
   window from the last `product_clicked`.
2. **Purchase-outcome learning loop tomorrow** — the v2 contextual-bandit
   / preference-learning path replays this exact stream + the filtered
   order webhook to update per-tenant ranking. The spec locks this in as
   the reason we ship the schema at MVP even though the learner is v2.

Because both consumers read the same events, there is **no migration when
v2 lands**. This is a hard requirement: adding a fifth event type is
cheap; renaming or reshaping the existing four after we have live
merchants is not.

### Ingest discipline

Widget batches up to 10 events per request with a 1-second flush; the
API validates against a strict schema and rejects unknown fields (fail
fast on client bugs). Writes are enqueued and processed async into the
`events` table with a p95 write latency under 100 ms. Fire-and-forget
from the widget — no retry storms on the merchant's storefront.

Holdout assignment is deterministic on `session_id`: `hash(session_id)
mod 100 < 10 → holdout`. Assignment is stored on the `sessions` row on
first `widget_shown` and **never re-rolled** — if it were, retroactive
holdout membership would poison every lift number we ever computed.

---

## 6. Embeddings

**Where:** pgvector column on `tenant_products.embedding`, alongside
`embedding_model_version` and `last_embedded_at`.

Embeddings are per-tenant by virtue of living on a per-tenant table. There
is no shared embedding space and no cross-tenant retrieval. This is
enforced by the same `WHERE tenant_id = ?` rule as every other read.

### When we (re)embed

- On initial catalog sync.
- On `products/create` and on `products/update` **only when the
  `content_hash` (SHA-256 of title + description + product_type + tags)
  changes**. Price and inventory updates skip embedding entirely — they
  cost real money and do not change the vector.
- Never at recommendation-serve time.

Embed calls are batched per tenant and tagged with `tenant_id` for cost
attribution — same as every LLM call.

### Model versioning

Each vector is stored with the `embedding_model_version` that produced it.
This lets us:

- Detect drift when a model is upgraded (mixed-version tenant → schedule a
  per-tenant re-embed job, not a silent full-fleet rebuild).
- Attribute cost by model at the tenant level.
- Roll out a new embedding model per-tenant with a shadow-index period if
  we need to evaluate quality first.

### Provider choice

**TBD at the first ingestion PR.** `architecture.md` §1 lists Anthropic or
OpenAI as candidates and notes the decision drives the vector column
dimensionality. This doc deliberately does not pick — the choice is a
recommendations-specialist + backend joint decision, not a data-model
one. What this doc commits to: whichever provider is picked, the model
version lives on every row, and switching is a per-tenant re-embed job.

---

## 7. Retention & purge

Three retention regimes:

1. **Active tenant data** — kept as long as the tenant is active. No TTL
   on catalog rows, events, orders, or precomputed recs while
   `tenants.active = true`.
2. **Soft-deleted tenant (uninstall)** — Shopify `app/uninstalled` sets
   `tenants.soft_deleted_at`. Widget stops rendering, catalog sync stops,
   webhooks are still HMAC-verified but no-op for a soft-deleted tenant.
   Data is kept for **30 days** to permit reactivation on reinstall
   (spec: Merchant onboarding — install/uninstall).
3. **Hard purge** — scheduled daily job at 03:00 UTC deletes every
   `tenant_id`-scoped row across every module's tables for tenants whose
   `soft_deleted_at` is older than 30 days. Explicit list of tables lives
   with the purge job, not here; the invariant is *every* table carrying
   `tenant_id` is included, and CI has a test that fails if a new
   tenant-scoped table is added without being registered with the purge
   job.

Events and orders are append-only within the tenant lifetime — we do not
prune old events per-tenant at MVP. If storage cost becomes a driver
(pilot volumes suggest it will not for the first year), a rolling window
on `events` older than ~13 months is the natural next step so
year-over-year comparisons still work.

The archived B2C doc had a 30-day post-deactivation purge on individual
product rows. That does not apply here — the merchant's catalog is the
truth; when they deactivate a product we mark `available=false` and keep
the row so historical events still resolve to a product name.

---

## 8. Intent-form schema versioning

**Owner:** Recommendation module. **Shape:** `intent_form_schemas` in
`architecture.md` §8 — composite PK `(tenant_id, version)`, exactly one
`active = true` row per tenant, JSONB `schema_json` defining fields,
labels, and options.

Versioning per tenant from day one is a spec requirement (Widget → Intent
capture form). The rationale is roadmap-driven: verticalised intent packs
(Fashion style archetypes, Tech platforms owned, Beauty skin type,
Books format, etc. — see `gtm-b2b.md`) land as a post-MVP release and
must not break widgets on live merchants that predate them.

What this buys us at MVP:

- A merchant on the generic form (`version = 1`) keeps rendering it while
  we introduce `version = 2` for a Fashion pack in their tenant.
- Migration between versions is a per-tenant admin action, not a fleet
  operation.
- Events carrying an intent payload can be interpreted against the
  schema version that was active when they fired, without introducing a
  new event shape.

What lives on the schema JSON is the field set, labels, option lists, and
required/optional flags. It does not include styling — that is
`widget_config`. It does not include the ranking prompt — that is the
Recommendation module's concern.

**Open question for PO:** the `intent_submitted` event does not currently
carry the intent-form schema version it was answered against. If we plan
to run the learning loop across a tenant's schema upgrade, we need it.
Cheap to add on the event; expensive to backfill. Recommend adding
`intent_form_schema_version` to the `intent_submitted` event payload from
day one.

---

## 9. Cost telemetry

**Owner:** Recommendation module + shared telemetry infrastructure.
**Shape:** either a stream (Prometheus / OpenTelemetry — decision TBD in
`architecture.md` §14) or a durable `cost_telemetry` table (also in §8).

Every recommendation call emits a tagged sample, unsampled — no
statistical sampling. The tag set is fixed: `tenant_id`, `placement`,
`model`, `input_tokens`, `output_tokens`, `cache_hit`, `served_from`
(`live_llm | cache | precomputed | fallback`), `latency_ms`.

Two consumers:

1. **Per-tenant cost dashboard** — daily-spend threshold triggers the
   kill switch (see §10). Live-tuned per tenant.
2. **Post-hoc unit-economics analysis** — cost per merchant per rec, per
   placement, per model. Feeds the pricing tier revisit conversation in
   `gtm-b2b.md`.

Why unsampled: pilot volumes are low enough that sampling loses fidelity
on outlier tenants, and outlier tenants are exactly the ones we need to
see (a runaway cost is a per-tenant event, not a fleet-average event).

**Decision deferred:** stream vs durable table. `architecture.md` marks
this as TBD at the first devops PR. This data doc treats them as
equivalent for now — the field set is the same either way.

---

## 10. Kill switch and guardrail state

**Owner:** Recommendation module. **Backing store:** Redis for the live
counters; DB for the configured thresholds.

Redis holds the ephemeral counters and flags:

- `cap:{tenant_id}:{YYYY-MM}` — monthly usage counter for the per-tenant
  hard usage cap (default 50k live-LLM recs during pilot, configurable
  per tenant via the `tenants.monthly_usage_cap` column).
- `rate:{session_id}:{YYYY-MM-DD-HH}` — per-session live-LLM call counter
  (max 20 / session / hour).
- `kill:{tenant_id}` — bool + reason string. When set, every rec call
  falls through to precomputed cold recs for that tenant regardless of
  intent signal. Set automatically when the daily-spend threshold from
  cost telemetry is breached; only cleared by an ops action.

Why Redis and not the DB: these are read on every rec call inside the
300 ms latency budget, and the write pattern is per-request. A DB round
trip on the hot path is out.

Durable state that shapes the guardrails lives on the tenant record:
`plan`, `monthly_usage_cap`, and a nullable `daily_spend_cap` column
(settled 2026-09-29). `daily_spend_cap` is NULL by default and falls
through to the per-plan constant at check time; support sets it
explicitly only when a pilot merchant needs custom headroom or an
outlier tenant needs a bespoke cap. No runtime lookup complexity —
NULL means "use plan default." The per-plan constants themselves are
still TBD at first pilot.

The kill switch flip is a P0 alert to ops. Recovery is manual by design:
the widget stays live serving precomputed recs (shopper cannot tell) and
ops decides whether to raise the cap, tune the model choice, or contact
the merchant.

---

## 11. Anonymous session ID

**Owner:** Widget (generation) + Events module (persistence). **Shape:**
`sessions` in `architecture.md` §8.

Generation is client-side: on first widget render on a browser, the
widget mints a UUID v4 and writes it to localStorage under a
WiseGift-scoped key with a 30-day sliding TTL. That ID is the widget's
identity for its own analytics, and it is *not* linked to:

- The merchant's own customer database. WiseGift never receives it.
- The Shopify customer ID. WiseGift never asks for it.
- Any cross-domain identifier. The session ID is scoped to the
  merchant's origin because it lives in the merchant's localStorage.
- Any other WiseGift tenant. A shopper who visits two merchants using
  WiseGift generates two independent session IDs, one per merchant's
  origin.

The server-side `sessions` row holds: the UUID (PK), `tenant_id`, the
sticky `is_holdout` flag (see §5), `first_seen_at`, `last_seen_at`.
Nothing else. There is no IP, no user-agent, no referrer. The row is
created lazily on the first `widget_shown` event, not on session-token
issuance.

### On EU consent

Whether a first-party session identifier requires explicit consent under
some EU regimes even without cross-site tracking is an open question
routed to security-and-privacy (see `spec.md` and `architecture.md` open
questions). This data model does not pre-decide the answer — if consent
is required, the widget delays session-ID creation until consent is
granted, and the data model is unchanged (rows just get created later or
not at all).

---

## 12. What we deliberately left behind from the B2C model

Called out so no one carries a pre-pivot habit forward:

- **Global `products` table with a `country` filter.** Gone. The tenant
  IS the scope. No cross-tenant retrieval, no country column on catalog
  rows (the merchant sells where they sell).
- **Universal + per-merchant giftability rules (Layer A / Layer B) and
  fingerprint-based cross-merchant dedup.** Gone. In B2B the merchant
  has already curated their assortment; we do not overlay a giftability
  filter, and there is no cross-merchant space to dedup across.
- **Per-row 30-day purge after deactivation with a `deactivated_at`
  column on products.** Not applied to catalog rows. `available=false`
  is the deactivation state; the row stays so historical events resolve
  to a product name.
- **Affiliate provenance columns (`provider_id`,
  `provider_product_id`, `provider_categories`).** Gone. Products come
  from one place per tenant (their platform), identified by
  `shopify_product_id`.
- **A canonical WiseGift category taxonomy (Technology / Home & Living
  / Beauty & Wellness etc.).** Not applied. Ranking uses the merchant's
  own `product_type` and `tags`; verticalised intent packs (post-MVP)
  are per-vertical *of the tenant*, not a normalised category applied
  to catalog rows.

---

## 13. PO validation needed

Before this doc is treated as the source of truth for the first
ingestion / events / attribution PRs, please review:

1. **Deactivation threshold on missed nightly reconciliation.** §3
   proposes "2 consecutive misses → `available=false`, do not delete
   the row." Confirm or override. Decision affects merchant-visible
   catalog-size numbers on the admin dashboard.
2. **`intent_form_schema_version` on the `intent_submitted` event.**
   §8 recommends adding it now so the v2 learning loop can interpret
   old intent payloads across a schema upgrade. Cheap to add today,
   expensive to backfill. Confirm to include from day one.
3. **Custom per-tenant kill-switch daily-spend threshold.** §10 assumes
   the threshold is a per-plan constant, not per-tenant. If any pilot
   is likely to want a bespoke number (e.g. a Design Partner running
   an unusually large campaign), we need a `daily_spend_cap` column on
   `tenants` — trivial to add later but flag if it is a day-one need.
4. **Event retention past 13 months.** §7 flags no MVP TTL on `events`.
   If the answer is "we want at least N months of history for
   year-over-year comparisons," name N so the archive plan is not a
   surprise.
5. **Session-ID consent posture.** §11 defers to the open
   security-and-privacy question. Confirm this doc does not need to
   pre-commit to a stance before that legal review lands.
6. **Embeddings provider.** §6 leaves the choice to the first
   ingestion PR. Confirm no data-model decision needs to be made
   before that PR is scoped.

No section of this doc should be treated as validated until the PO has
signed off on the items above.
