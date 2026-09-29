# WiseGift — Recommendation Engine

> B2B rewrite. Supersedes `docs/archive/recommendations-b2c-pre-pivot.md`. That
> doc described a shared affiliate-catalog engine with user-clip diversity
> constraints, popularity-bias mitigations, and consumer-side relevance prompts —
> none of which apply here. Per-tenant catalog and merchant-scoped attribution
> replace all of it.
>
> Companion docs (do not duplicate — reference):
> - `docs/product/spec.md` — product surface (widget, admin, APIs, KPIs)
> - `docs/engineering/architecture.md` — modules, schema, serving pipeline shape
> - `docs/growth/gtm-b2b.md` — cost model, pilot playbook, significance bar
> - `docs/core/data.md` — catalog ingestion, embeddings, quality gates (drafted in parallel)

---

## 1. Positioning

Recommendation quality is **the** differentiator of WiseGift — not the widget,
not the Shopify install flow, not the analytics dashboard. Those are table
stakes. Merchants keep paying only if attributed lift on revenue-per-session
holds up against the 10% holdout.

The engine is **general-purpose over per-tenant catalogs**: it serves both
self-intent ("for you") and gift-intent ("for someone else") through a single
pipeline. Gift-intent is the highest-value use case — it's what the widget hook
sells, it's what generic behavioural engines (Nosto, Rebuy, Klevu) don't do —
but the mechanism underneath is one retrieve+rank pipeline that branches on the
intent payload, not two engines glued together.

Every merchant is a tenant. Every tenant starts with **zero behavioural
signal**. The engine must produce respectable recs on day 1 from catalog
metadata + widget-provided intent alone. Cold-start-per-merchant is the
first-order problem, not an edge case.

**No cross-tenant learning at MVP.** Multi-tenant isolation is a load-bearing
invariant (see `spec.md` Feature: Multi-tenancy). Ranking never mixes signal
from tenant A into tenant B's serving path, even indirectly.

---

## 2. How quality is measured

Every ranking change ships with a metric — offline, online, or both — declared
before the code. No unmeasurable cleverness.

### 2.1 Online metrics (pilot floor)

Every pilot ships a **90% exposed / 10% holdout** session split from day one
(see `spec.md` Feature: Widget — Attribution & session tracking).
Holdout assignment is:

- **Session-level**, keyed on the widget's anonymous `session_id`.
- **Sticky** — assignment is computed once on first `widget_shown` (deterministic
  from `hash(session_id) mod 100 < 10`), stored on the `sessions` row, and
  never re-rolled. Same shopper, same bucket, across placements and page loads.
- **Tenant-scoped** — a 10% holdout on tenant A is drawn from tenant A's
  sessions only. Never a cross-tenant pool.
- **Placebo** — holdout sessions render the widget UI identically; the rec
  slot is either hidden or filled with the merchant's static fallback. Invisible
  to the shopper.

The metric hierarchy (in order of business weight):

| # | Metric | Definition | Where it lives |
|---|---|---|---|
| 1 | Widget CTR | `widget_engaged` / `widget_shown` | Attention floor. Broken hook or dead widget shows up here first. |
| 2 | Add-to-cart rate on recommended SKU | Sessions where `product_clicked` on a recommended SKU is followed by a Shopify `carts/update` including that SKU within 15 min | First real quality signal — did the shopper act on a rec? |
| 3 | Conversion rate | Attributed orders / `widget_shown` sessions, 7-day attribution window | Bridges intent to revenue. |
| 4 | AOV | Attributed order value / attributed orders | Detects whether the widget shifts basket composition (up or down). |
| 5 | Revenue per session | Attributed order revenue / `widget_shown` sessions | **The composite metric that decides case studies and renewals.** |

All five are reported per placement, always as `exposed vs holdout` with lift %
and a significance flag (see `spec.md` Feature: Merchant admin — Analytics
dashboard).

### 2.2 Significance bar

Aligned with `gtm-b2b.md`:

- **p < 0.05**, two-sample test on the exposed vs holdout ratio.
- **Minimum ~5,000 exposed sessions per placement** before a lift number is
  case-study-ready or a ranking change graduates from experiment to default.
- Below the threshold the admin dashboard renders the "Not enough data yet"
  callout for that KPI (per `spec.md`).
- Test choice (two-proportion z-test for CTR / add-to-cart / conversion,
  Welch's t-test on log-revenue for AOV / revenue-per-session) — **TBD at first
  analytics PR**. Consistency across KPIs matters more than picking the
  fanciest test.

### 2.3 Offline evaluation (before pilot exposure)

Online metrics can't guide day-0 ranking choices — there's no traffic yet. For
pre-pilot changes:

- Synthetic intent set per vertical (5–10 hand-crafted `intent` payloads:
  budget × occasion × interest combos) run against a seeded fixture catalog.
- Metrics: precision@5 and MRR against a human-labelled relevance set (labels
  from the recommendations-specialist and PO, not the model).
- **Purpose is regression detection, not absolute quality**. A change that
  drops precision@5 on the fixture set by more than a small margin (threshold
  **TBD at first eval PR**) does not merge without an explicit
  accepted-regression note in `docs/decisions.md`.
- Seeded fixture catalog lives in the backend test resources; per-tenant real
  catalogs are never used for offline eval.

### 2.4 Reproducibility & traceability

Every `/widget/v1/recommendations` response is tagged with:

- `model_version` (Anthropic model ID + snapshot date)
- `prompt_version` (monotonic per ranking-prompt change)
- `feature_version` (monotonic per intent-signature / context-signature schema change)
- `served_from` (`live_llm` | `cache` | `precomputed` | `fallback`, per `spec.md`)

Tags are emitted with cost telemetry (per `architecture.md` §5) and echoed on
the `product_clicked` event (per `spec.md` Event API). Any bad-outcome session
can be traced back to the exact model + prompt + feature that produced its
slate. Random components (candidate tie-breaks, prompt sampling if any) are
seeded from `hash(tenant_id, session_id, intent_signature)` for
reproducibility.

---

## 3. Cold-start per merchant

Every new tenant lands with zero events. The engine must serve reasonable recs
from day 1 using only catalog + widget-provided intent.

### 3.1 Precomputed cold rec bank (nightly, per tenant)

Per `architecture.md` §5, a nightly job at `03:00 UTC` (offset by tenant hash to
smear load) produces:

- For each active SKU in `tenant_products`, top-N recommendations computed
  from **vector nearest-neighbour** (pgvector cosine) + **heuristic ranking**.
- Heuristics at MVP: price-band adjacency (same order-of-magnitude to the
  source SKU), category diversity (don't return 5 near-duplicates), and
  in-stock filter. No LLM calls.
- Stored in `precomputed_recs (tenant_id, source_shopify_product_id, rank,
  recommended_shopify_product_id, computed_at)`.
- Re-run on-demand when catalog change exceeds a per-tenant threshold (e.g.
  >5% of SKUs updated in a day). **Threshold TBD at first pilot** — start
  conservative, tune from observed rebuild latency.

The cold bank is the **default serve** for any request without real intent
signal, and the **fallback** for any request that fails guardrails or LLM
availability.

### 3.2 Cold → warm transition

Per SKU:
- Cold recs remain the source of truth for **PDP-context requests** (shopper
  landed on a product page, has not engaged the intent form).
- The instant real intent lands (widget-engaged with a submitted intent form),
  the live LLM path takes over for that request. There is no "warm-up
  threshold" — intent is the switch, not accumulated data.

Per tenant:
- MVP has **no per-tenant warm-up** — the ranking prompt is the same on day 1
  and day 90 of a pilot.
- The v2 feedback loop (see §7) is what turns accumulated tenant-specific
  outcomes into per-tenant ranking bias. That is v2 explicitly.

### 3.3 Cold-start risks flagged

- **Sparse embeddings** — a catalog with short, low-quality product
  descriptions produces weak vectors. Cold recs will cluster on whichever
  descriptive dimension survives. Coordination point with catalog engineer:
  reject catalogs at ingest where median description length is below a
  threshold (see `data.md`).
- **Small catalogs** — tenants with <100 active SKUs produce shallow rec
  slates. Below-threshold catalogs at ingest should trigger an admin warning
  ("catalog too small for personalisation quality"). **Threshold TBD** — likely
  around 100 SKUs based on ICP; confirm at first pilot.
- **Category imbalance within a tenant** — a fashion tenant whose catalog is
  90% dresses will over-return dresses regardless of intent. The category
  diversity heuristic caps this at slate level, but the tenant should surface
  it to the merchant admin. Post-MVP.

---

## 4. Live vs precomputed serving policy

The single most important cost + quality lever in the system. Policy is small
and reviewable; mechanism (the LLM call, the vector search) is complex and
testable — kept isolated per CLAUDE.md.

### 4.1 Decision tree (per `architecture.md` §5, restated for clarity)

Every `/widget/v1/recommendations` call flows through:

```
1. Cache lookup (Redis, 24h TTL)
     key = (tenant_id, intent_signature, context_signature)
     → HIT  → return cached response, served_from = cache
     → MISS → continue

2. Guardrail checks (Redis-backed counters)
     - Per-tenant monthly usage cap reached?          → precomputed fallback
     - Per-session hourly rate limit (20 calls) hit?  → precomputed fallback
     - Per-tenant kill switch flipped?                → precomputed fallback

3. Intent signal check
     - No real intent (PDP-context view, no form submitted)?
       → serve precomputed cold recs for current SKU, served_from = precomputed

4. Live LLM path
     - Embed intent (interests + budget + occasion + relationship + context SKU)
     - pgvector top-K on tenant_products.embedding (K = 50)
     - Rank via Claude Haiku 4.5 (or Sonnet if tenant flag on)
     - Return top-N (default 5), served_from = live_llm, write to cache

5. Fallback
     - LLM error, timeout > 2s, or zero candidates
       → precomputed cold recs
       → if empty, top-N merchant bestsellers
       → last resort, hide the rec slot (widget never renders an error)
```

### 4.2 Intent signature — what collapses into one cache key

The `intent_signature` is a SHA-256 of a **canonical** intent payload:

- `intent_mode` (`self` | `gift`)
- `budget_min`, `budget_max` **bucketed to price bands** (bands TBD at first
  pilot — likely quartiles of the tenant catalog's price distribution)
- `occasion` (verbatim from the fixed enum in `spec.md`)
- `relationship` (verbatim from the fixed enum)
- `interests[]` — **sorted, lowercased, whitespace-normalised**

The `context_signature` is a SHA-256 of:

- `placement` (`home_hero` | `pdp_slot` | `gift_finder`)
- `shopify_product_id` if PDP, otherwise null

Pattern mirrors `RecipientProfileHasher` from the pre-pivot codebase — same
discipline: canonicalise inputs before hashing so trivially-different requests
collapse. Two shoppers with identical intent + identical context share a cache
entry within the same tenant. **Never across tenants** — `tenant_id` is part
of the cache key, not the signature.

Open question: **how aggressively to bucket interests**. Stemming? Synonym
collapse? Free-text field carries meaningful signal but also creates cache
fragmentation. MVP: no stemming, just normalise. Revisit at first pilot if
cache hit rate is below the 60% target.

### 4.3 Real intent — the definition

"Real intent" gates the live LLM path. Definition for MVP:

- **Gift-intent flow**: intent form submitted with all required fields
  (relationship, occasion, budget). Interests optional but included in the
  signature if present.
- **Self-intent flow**: intent form submitted with **at least one** of budget
  or interests. Bare "For me" click with no additional input on a PDP counts
  as **passive** — serves precomputed cold recs for that PDP.
- **Bare PDP view** (no widget engagement): passive → precomputed.

Rationale: the LLM's added value over vector search is largest when the intent
payload is rich. Cheap, generic queries don't earn a live LLM call.

### 4.4 Session rate limit interaction

20 live-LLM calls per session per hour (per `spec.md`). Above the limit,
subsequent requests serve from cache (if hot) or precomputed. The cap is
generous — a shopper who genuinely re-tunes intent 20 times in an hour is
edge-case; anything higher is likely bot traffic that would otherwise drain
budget.

---

## 5. Retrieve + rank pipeline

### 5.1 Retrieval

- **Store**: `tenant_products.embedding` (pgvector column, per-tenant scoped
  via composite index `(tenant_id, ...)`).
- **Query vector**: computed from the canonical intent payload — concatenated
  `interests + occasion + relationship + intent_mode + context_product_title`
  → embedding call.
- **K**: 50 candidates by cosine similarity. **Hard filters applied in SQL**:
  `tenant_id = ?`, `available = true`, budget band on `price_min`/`price_max`.
- **Embedding cost**: intent embeddings are computed live on live-LLM-path
  requests only. Cache-hit and precomputed paths don't embed. Catalog
  embeddings are ingest-time only (per `architecture.md` §5).

### 5.2 Ranking

- **Input to LLM**: top-K candidates (compact: id, title, price, tags,
  product_type, first ~200 chars of description) + the intent payload +
  slate constraints.
- **Output**: ordered list of top-N (default 5) SKU IDs. **No free-text
  explanation returned to the widget at MVP** — the widget doesn't render one.
  If the merchant admin wants per-product rationale for debugging, that's a
  separate internal endpoint (post-MVP).
- **Slate constraints in the prompt**:
  - Return exactly N or fewer (never pad with irrelevant items — per `spec.md`
    Feature: Widget — Recommendation display).
  - Category diversity: no more than 2 products from the same
    `product_type`.
  - Budget compliance: 100% within the requested band. Enforced post-LLM as a
    hard filter, not trusted from the model — any out-of-band pick is dropped
    and the slate is padded from the retrieved candidates.
- **Slate size**: merchant-configurable 3–10, default 5.

### 5.3 Self vs gift branching

Both intent modes flow through the same retrieve+rank pipeline. Branching
happens at **prompt template selection**, not at pipeline level:

- **Self prompt**: framing is "find products this shopper will want to buy for
  themselves." Uses interests + budget + context SKU. Simpler; recipient
  fields not present.
- **Gift prompt**: framing is "find gift ideas for this recipient." Injects
  relationship + occasion into the framing. Interests are treated as
  *recipient* interests, not shopper interests.

Both templates version-bumped together and tagged in `prompt_version`. Prompt
text itself is out of scope for this doc — lives in the backend module's
resources and is diff-reviewable in each ranking PR.

### 5.4 Precomputed path (no LLM)

For the precomputed fallback path, ranking is:

1. Top-K vector neighbours of the source SKU (K=20).
2. Score = cosine similarity, penalised by:
   - Same-`product_type` penalty (drop rank of the N+1..M near-duplicates)
   - Price-band penalty (products more than one band away from source SKU price)
3. Top-N returned.

No LLM. Deterministic. Serves as the honest floor of quality — if
precomputed-served sessions and holdout sessions don't diverge on any KPI,
the widget is contributing zero value on those requests and the merchant is
being told the truth.

---

## 6. Model policy

Per `architecture.md` and `spec.md` cost guardrails:

- **Default: Claude Haiku 4.5** for both intent parsing (if needed for future
  free-text handling) and ranking. Cheap, fast, sufficient for structured
  intent → ranked slate.
- **Escalation: Claude Sonnet 4.6**, behind a **per-tenant feature flag**.

### 6.1 What triggers Sonnet escalation

MVP policy:

- **Off by default** for every tenant, including pilots.
- **Manually flipped on** by ops for a specific tenant when either:
  - Merchant is on the Scale or Enterprise tier (pricing structure funds it), or
  - A pilot merchant has demonstrated CTR / revenue-per-session lift below
    target on Haiku and Sonnet-vs-Haiku is being A/B tested as a lift lever.
- **Scope**: per tenant, not per request. Sub-tenant scoping (e.g. Sonnet only
  for gift-intent flows with 5+ interests) is possible via the same flag
  mechanism but **not designed in MVP** — flag is boolean per tenant.
- **Flag storage**: `tenants.feature_flags` JSONB (schema TBD at first tenant
  module PR, currently the schema in `architecture.md` §8 doesn't have this
  column yet — flagged below).

### 6.2 Model version pinning

Every response tags the exact model ID + snapshot date. Model upgrades
(Haiku 4.5 → Haiku 5) are a **coordinated rollout**, not silent drift:

1. New model tested offline against the fixture eval set.
2. Rolled out to internal test tenants first.
3. A/B tested per pilot tenant against the incumbent for at least the
   significance bar (§2.2).
4. Graduated to default only if lift is non-negative.

---

## 7. Per-tenant kill switch integration

Per `spec.md` Cost guardrails and `gtm-b2b.md`:

- Before any live-LLM path decision, check Redis flag `kill:{tenant_id}`.
- Flag is set by the cost-telemetry alerting layer when a tenant's
  **daily-spend threshold** is crossed. Thresholds are per-plan-tier
  (defaults **TBD at first pilot**; pilot default should be conservative —
  e.g. daily budget = monthly cap / 30 × 1.5x tolerance).
- Flag is **auto-set**, **manually reset**. No auto-recovery — ops looks at
  why the tenant tripped before flipping back.

### 7.1 Behaviour when tripped

1. Every rec call for that tenant is served from **precomputed cold recs**.
2. `served_from = precomputed` on every response — the widget renders normally,
   the shopper sees products, no error state.
3. The admin dashboard renders a **persistent banner** ("You've hit your daily
   AI budget — recommendations are serving from the nightly cache. Upgrade to
   restore live personalisation."). Wording TBD with marketing.
4. **PO is alerted** — ops on-call channel + email. Alert includes tenant,
   trigger time, spend at trigger, projected month-end spend if not stopped.
5. Cost telemetry continues to record `served_from = precomputed`, so the
   duration of the degraded state is measurable and reported in the day-90
   pilot review.

### 7.2 Non-behaviour

- **No cross-tenant kill switch spillover.** Tenant A tripping the kill switch
  never affects tenant B's serving path.
- **No lift-metric contamination.** Precomputed-served requests during a
  kill-switch window are still exposed sessions and still counted in the
  holdout comparison. If precomputed is significantly worse than live, this
  will show in the metric — that's the truthful reading.

---

## 8. v2: purchase-outcome feedback loop (framing only)

Per `gtm-b2b.md` and `spec.md` roadmap. **MVP does not build this.** MVP does
build the event schema that v2 will consume, so v2 doesn't need a backfill.

### 8.1 What MVP emits (data.md is authoritative)

- `widget_shown` — placement + session + holdout + tenant.
- `widget_engaged` — hook clicked.
- `intent_submitted` — intent payload snapshot.
- `product_clicked` — with `rank_score`, `served_from`, `model_version`,
  `prompt_version` tagged so v2 can attribute outcomes back to the exact
  ranking decision.
- Filtered `orders/create` webhook — line items + total, correlated via
  `wg_session` marker (per `spec.md` Widget — Attribution).

Together these form the (context, action, outcome) tuples a bandit or
preference model needs.

### 8.2 What v2 looks like at a high level

- **Per-tenant learner** — a contextual bandit or preference-learning model
  fit on that tenant's own event stream. **No cross-tenant learning at v2
  either** unless a specific decision revisits the isolation invariant.
- **Learning signal** — a weighted combination of `product_clicked`,
  add-to-cart, and attributed order, with the strongest weight on attributed
  order revenue.
- **Serving integration** — the learner produces a per-(tenant, intent
  signature) ranking bias vector that's blended with the LLM's ranking at
  serve time. The LLM stays in the loop for cold intents and slate diversity;
  the learner tilts within its output.
- **Evaluation** — same 90/10 exposed/holdout split, same KPI hierarchy,
  same significance bar. Compounding lift over pilot tenure is the
  hypothesis; measure it, don't assume it.

Design of the learner itself, feature engineering, retraining cadence, and
regression protection are **not designed here**. They belong in a v2 doc
opened when v2 is scoped.

---

## 9. Explicitly NOT MVP

To keep the surface reviewable and to avoid re-litigating settled scope:

- **Verticalised intent packs** (fashion style archetypes, tech platforms,
  book format preferences) — first post-MVP release per `spec.md` roadmap.
- **Purchase-outcome feedback loop / bandit learner** — v2 per §8.
- **Rev-share attribution model** — Enterprise-only, revisited when Enterprise
  pilots are on the table (per `gtm-b2b.md`).
- **Cross-tenant learning / transfer / warm-start-from-similar-merchant** —
  violates the multi-tenant isolation invariant; not on the roadmap.
- **Merchant-facing per-product rationale** ("why did the widget recommend
  this?") — post-MVP admin analytics feature, not shopper-facing.
- **A/B experiments UI for merchants** — post-MVP per `spec.md` roadmap.
- **Behavioural signals beyond widget interactions** — post-MVP and gated on
  a legal review (per `spec.md` roadmap).
- **Bare "For me" click as live-LLM trigger** — treated as passive intent at
  MVP; served from precomputed. Revisit if pilots show significant self-intent
  volume without richer input.

---

## 10. Open questions for PO validation

Called out inline above; consolidated here:

1. **Cache hit rate target of 60%+** — is that acceptable, given it means up to
   40% of engaged requests go live to Claude? Alternative is a coarser
   intent signature (bucket interests harder) that raises hit rate but
   collapses more nuance. Trade-off is quality vs cost. Currently biased
   toward quality — confirm.
2. **Threshold for "catalog too small for personalisation"** — I've suggested
   ~100 SKUs but this is a guess. What's the smallest catalog we want to say
   yes to in a pilot?
3. **Sonnet escalation policy** — per-tenant boolean flag is what
   `architecture.md` says. Should there be a **per-flow** override
   (e.g. Sonnet only for gift-intent, Haiku for self)? That's more nuanced and
   more expensive to build; MVP proposes boolean-per-tenant only.
4. **Daily-spend threshold defaults per plan tier** — pilot default of
   `monthly_cap / 30 × 1.5` is a reasonable starting point but not derived
   from data. What's the PO's ceiling for a single bad day on a pilot tenant
   before ops intervenes?
5. **Significance-test choice** — z-test for rate KPIs and Welch's t on log-
   revenue is my proposal. Any preference from PO or is this fine for the
   dashboard team to lock in at the first analytics PR?
6. **Merchant-facing kill-switch banner copy** — needs marketing review.
   Suggested language above is placeholder.
7. **`tenants.feature_flags` column** — needed for the Sonnet flag and any
   future per-tenant experiment toggles. Not in `architecture.md` §8 today.
   Flag to add in the first tenant-module PR, or open a specific decision
   entry?
8. **Offline eval regression margin** — I don't want to pick a threshold
   without at least one calibration run. Propose: land the fixture set +
   harness first, run it against the initial ranker, then set the margin
   from observed variance. Confirm approach.

---

*This doc is not "done" — it's the reviewable draft. PO validation gates any
of the above from becoming settled policy. Changes to this doc are logged in
`docs/decisions.md` when the PO signs off, not on every draft edit.*
