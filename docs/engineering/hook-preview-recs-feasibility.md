# Hook preview recs — feasibility note (DRAFT)

Owner: recommendations-specialist · Date: 2026-10-19 · Status: DRAFT, for PO validation

Scope: the "hook-attractiveness quick win" wants 2–3 REAL products shown INSIDE
the hook BEFORE the shopper gives any intent input. Intent inputs then REFINE
the preview. Two placements:

- **PLP / collection page** — preview = products representative of the
  category/collection being viewed.
- **PDP** — preview = similar/alternative products to the one being viewed
  (pure embedding similarity; complement / complete-the-look is explicitly
  out of scope here, see §5).

This note answers only: does a serving path exist, what is the cheapest way to
fill each gap, how it interacts with the cache key, and how we'd measure it.
No code. Nothing here is "done".

---

## 1. Does a pre-intent preview serving path already exist?

**Short answer: PDP is a small, mostly-wiring gap; PLP is a larger gap (missing
data + missing widget→backend signal). Neither is served today.**

### What exists in `wisegift-backend`

- `POST /widget/v1/recommendations` → `RecommendationService.recommend(...)`.
  This path **always requires an `IntentPayload`** (`intentMode` is mandatory)
  and **always embeds the intent string** to retrieve. There is no pre-intent
  entry point. `context` on the request carries **only `platform_product_id`**
  (`ContextDto`), nothing for a collection.
- `precomputed_recs` table (Flyway V1) exists and is keyed exactly for the PDP
  case: `(tenant_id, source_platform_product_id, rank) → recommended_platform_product_id`.
  **But it has no writer and no reader in main code** — the only reference is
  `TenantPurgeRepository` (deletes it on tenant purge). It is a dormant schema.
- `tenant_products.embedding vector(1536)` is already populated per product by
  the catalog module. `CatalogQueryPort.findNearest(tenantId, queryVector, k[, budget])`
  already does tenant-scoped pgvector cosine NN.
- `events.pre_intent_recs BOOLEAN` (Flyway V7, G13) already exists: the widget
  flags a `widget_shown` whose mount renders recs before any intent. The
  measurement hook is already in the schema.
- Event vocabulary already includes `widget_shown`, `widget_engaged`,
  `product_clicked`, `intent_submitted` (`EventType`).

### What happens today if you call the endpoint with no intent

A call with bare `intentMode` + a context product → signatures computed → cache
miss → `RealIntentCheck` trips `NO_INTENT` → `pgvectorSlate` embeds the string
`"intent: self\ncontext product: <title>"` and returns pgvector NN, **labelled
`served_from=precomputed`** (misleading — it is live pgvector, not precomputed).

So a crude PDP-ish result falls out of the existing fallback, but it is the
wrong mechanism: it re-embeds a string on every call even though the product's
embedding is already stored, and for PLP it has nothing to anchor on (no
context product, no collection id) → it would embed just `"intent: self"` and
return products nearest to the word "self", i.e. noise. **Not usable as-is.**

### Verdict per placement

| Placement | Pre-intent preview served today? | Nature of the gap |
|-----------|----------------------------------|-------------------|
| **PDP (similars)** | No — only an accidental, mislabeled live-pgvector fallback | Small: table + embeddings exist; need a writer + a clean reader |
| **PLP (category)** | No | Larger: **no collection-membership data ingested** and **no collection id in the widget→backend contract** |

---

## 2. Cheapest path to fill each gap

### 2a. PDP — similar/alternative products

The product's embedding already exists, so the cheapest correct path does **not
re-embed anything**.

**Recommended: precompute, don't serve live.** The candidate set is bounded
(the tenant's catalog), there is no intent variation, and the answer is stable
until the catalog or embeddings change. This is the textbook case for the
dormant `precomputed_recs` table.

- **Writer (nightly / post-catalog-sync, per tenant):** for each active,
  embedded product, self-join pgvector NN (`embedding <=> embedding`, exclude
  self, `available = true`, `excluded_reason IS NULL`), take top-N (N≈5), write
  `(tenant_id, source_platform_product_id, rank, recommended_platform_product_id, computed_at)`.
  This is a query already expressible with the same SQL shape as `findNearest`;
  it needs a new `CatalogQueryPort` method (NN from a stored product's vector,
  excluding itself) plus a scheduled writer command.
- **Reader (serve time):** lookup by `(tenant_id, source_platform_product_id)`,
  hydrate cards from `tenant_products`. **Zero live model cost, single indexed
  read, sub-millisecond.** Label `served_from=precomputed` — and here the label
  is finally honest.
- **Cost/latency:** serve = one indexed SELECT, no embedding, no LLM. Nightly
  compute = pgvector NN over the catalog (seconds per tenant for typical SMB
  catalogs). This is the cheapest possible serving posture.
- **Live fallback (only if precompute is stale/missing):** pgvector NN using the
  stored product embedding directly (still **no re-embed**). Needs a
  `findNearestToProduct` port method. Cheap (one vector query), but precompute
  should cover the overwhelming majority.

### 2b. PLP — category/collection-representative products

**Blocker first:** the backend currently has **no way to know which products
belong to the collection being viewed.** `tenant_products` has `product_type`
and `tags`, but no collection membership; and the widget sends no collection
identifier. So before any ranking question, two inputs are missing. Options,
cheapest first:

1. **`product_type` as a proxy for category (cheapest, no new ingestion).**
   The widget passes the viewed collection's dominant `product_type` (or the
   backend derives a representative type). Serve top-N products of that
   `product_type`, filtered to `available = true`. Needs: a widget→backend
   signal (new context field) and a simple typed query. Cheap, but
   `product_type` is coarse and not always aligned with how the merchant
   merchandises collections.
2. **Embed the collection title/handle → pgvector NN (cheap, live or precomputed).**
   The widget passes the collection title/handle; backend embeds that short
   string once and runs `findNearest`. One embedding call (~fractions of a
   cent, not an LLM rank). **Precomputable per collection nightly** → then serve
   time is a plain read. Reuses `findNearest` wholesale. Good default IF we can
   get the collection title to the backend.
3. **Centroid-of-collection similarity (best representativeness, needs
   membership).** Compute the centroid of the member products' embeddings and
   return the nearest members → "most typical products of this collection."
   This is the most faithful "representative" signal but **requires ingesting
   collection membership** (a Shopify collection → product mapping), which is a
   catalog-data-engineer work item, not a quick win.

**Recommendation for the quick win:** ship **option 2** (embed collection
title/handle, precomputed per collection), because it reuses `findNearest`, is
cheap, and precomputes cleanly. It still needs the **one contract change**:
the widget must send a collection identifier (handle + title) in the request
context, and we should add a small `collection_recs` read model (or extend
`precomputed_recs` with a collection-keyed sibling) so serve time is a read, not
a live embed. Flag to catalog-data-engineer: if we want option 3 later,
collection membership must be ingested.

**No popularity signal at cold start.** Every new tenant has an empty behavioural
log, so "top by popularity" is unavailable on day 1. Use
availability + similarity/representativeness only; revisit a popularity re-rank
once the purchase-outcome loop produces signal.

---

## 3. Cache-key interaction

Current key: cache is `(tenant_id, intent_signature, context_signature)`.
- `intent_signature` = SHA-256(`intentMode | relationship | occasion | exact_budget | sorted(interests)`)
- `context_signature` = SHA-256(`placement | platform_product_id`)

Implications for a pre-intent preview:

- **The preview has no intent inputs → `intent_signature` collapses to a single
  constant per `intentMode`** (e.g. everything null → `"self||...|none|"`). All
  variability lives in the context half. That is a strong signal that the
  preview should be served from the **precomputed / context read model**, not
  the live-LLM + 24h-response-cache path. The 24h response cache would sit on a
  tiny key space with a near-100% hit rate — precompute is strictly simpler and
  cheaper, and avoids labelling confusion.
- **PDP preview:** the context half (`placement | platform_product_id`) already
  keys naturally per product. If we serve from `precomputed_recs` keyed by
  product, we bypass `SignatureComposer` entirely for the preview — cleaner.
- **PLP preview:** `context_signature` today knows only `platform_product_id`;
  it has **no collection dimension**. To key a PLP preview we must add a
  collection identifier to the context (e.g. `context_signature = placement |
  collection_id`), OR key a dedicated collection read model directly. Either
  way this is a **context-key change**, and `ContextDto` must carry the
  collection id. No change to `intent_signature`.

**Net:** do not route the preview through the live+cache LLM path. Serve it from
a precomputed context read model (PDP: existing `precomputed_recs`; PLP: a new
collection-keyed read model). The intent-based cache key is for the refined,
post-intent slate — which continues to work exactly as today once the shopper
submits inputs.

---

## 4. Measurement — what makes a preview "good enough"

This work's KPI is **upper-funnel engagement, distinct from downstream lift**.

- **Primary online KPI:** `widget_shown → widget_engaged` rate, on mounts where
  `pre_intent_recs = true`. Both events and the flag already exist (`EventType`,
  `events.pre_intent_recs` from V7). "Engaged" = first meaningful interaction
  (expand / first intent input / `product_clicked` on a preview card).
- **Experiment design:** preview-populated hook vs. empty/copy-only hook. The
  existing 10% session holdout + `is_holdout` on events/sessions supports this;
  alternatively a config-flagged on/off per placement. Graduation bar is the
  standard one: statistically significant lift (p < 0.05, ≥ 5,000 exposed
  sessions per placement) before preview-on becomes default.
- **Secondary (guard against a pretty-but-useless preview):** preview-card CTR
  (`product_clicked` where the clicked product is a preview card), and that
  engaging with the preview does not cannibalise `intent_submitted` (we want
  the preview to pull shoppers INTO intent, not replace it). Watch
  `served_from` distribution stays `precomputed`-dominant (cost guardrail).
- **Offline proxy before shipping (cheap sanity gate, not the real KPI):**
  - PDP: precision@3 that preview items are genuine similars (curated/synthetic
    pairs, or cosine-threshold spot checks per tenant).
  - PLP: precision@3 that preview items actually belong to / match the viewed
    collection (against `product_type`/collection membership where available).
  Offline precision guards against obviously-wrong previews; the engagement
  delta is the metric that decides.
- **Reproducibility:** precompute writer must log model + feature (embedding)
  version + `computed_at` so a shown preview can be traced to the snapshot that
  produced it.

"Good enough" bar to state to the PO: a preview is worth shipping if it lifts
`widget_shown → widget_engaged` by a practically meaningful margin at the
significance bar above, without reducing `intent_submitted` rate and without
pushing `served_from` off precomputed-dominant.

---

## 5. Complement / complete-the-look (explicitly out of scope, noted)

PDP preview here is **similarity** only ("more like this / alternatives"), which
embeddings + pgvector already express. **Complementarity** ("goes with this" —
e.g. belt for trousers) is a different and costlier signal: embedding cosine
actively works against it (complements are often semantically distant). It would
additionally require one of: co-purchase / co-view signal (unavailable at
cold-start per merchant), a merchant-curated complement mapping, or an
LLM/taxonomy step to infer category-complement rules — plus its own evaluation
(is the complement genuinely cross-category and relevant). Out of scope for this
quick win; flag as a separate, later work item.

---

## 6. Summary for the PO

- **PDP preview = small gap.** Table + embeddings already exist; needs a nightly
  writer into `precomputed_recs` (self-NN per product) and a clean reader. No
  live model cost at serve time. Cheapest and recommended.
- **PLP preview = larger gap.** Needs (a) a widget→backend collection signal
  (new context field) and (b) a representativeness source. Cheapest viable path
  = embed the collection title/handle → pgvector NN, precomputed per collection.
  Centroid-of-collection (most faithful) needs collection-membership ingestion —
  a catalog-data-engineer item, not a quick win. No popularity signal exists at
  cold-start.
- **Cache:** do NOT route the preview through the live-LLM + 24h response cache;
  the intent half of the key is constant with no intent, so serve from a
  precomputed context read model. PDP keys by product (exists); PLP needs a new
  collection dimension in the context key / read model.
- **Measurement:** primary KPI `widget_shown → widget_engaged` on
  `pre_intent_recs=true` mounts (events + flag already in schema); experiment
  vs. empty hook at the standard graduation bar; offline precision@3 as a
  cheap pre-ship sanity gate.
