# PDP similars — gender-hard, category-soft ranking (design)

> **Status: DRAFT for PO + backend-engineer validation.** Owner:
> `recommendations-specialist`. Not committed by this draft; the coordinator
> handles git. Nothing here is settled until the PO signs off; this draft does
> not edit `decisions.md` or CLAUDE.md.
>
> **Why this exists.** Live PO testing of the PDP product-preview on the fashion
> demo surfaced four quality rules. The current surface is pure pgvector self-NN
> similarity, which ignores gender and can return women's products for a men's
> PDP. A **throwaway manual DB patch** (gender via `title LIKE 'Men''s %'`,
> top-3, all-sources) was applied to the dummy store so the PO could see the
> target behaviour. That patch is a hack and will be reverted. This doc designs
> the **real** mechanism on true product attributes + configuration.
>
> **Desired behaviour (from the PO).**
> 1. **Gender is a hard priority** — men's → men's, women's → women's.
> 2. **Category is a softer priority** — same-gender cross-category is *good*
>    (trousers that go with a shirt). Do not over-constrain to same type.
> 3. **Count is configurable, default 3** — tied to admin config, not hardcoded.
> 4. **Source = all products** (incl. out-of-stock), recommending only available
>    products — an out-of-stock PDP showing available alternatives is a
>    sale-recovery play.
>
> **Code read (read-only), `wisegift-backend`:** `catalog` module
> (`TenantProduct`, `CatalogQueryPort`, `NearestProduct`,
> `CatalogIngestService`, `EmbeddingService`, `NearestProductFinderAdapter`),
> `recommendation` module (`PdpPreviewPrecomputeService`, `PdpPreviewService`,
> `PrecomputedRecsWriter/Reader`), `platform-integrations`
> (`ShopifyPlatformCatalogSource.parseProduct`, `PlatformProductData`,
> `ShopifyProductClassifier`, `ShopifyWebhookController`), Flyway `V1`, `V16`.
> **Companion:** `docs/engineering/recommender-eval-plan.md` (the surface under
> test), `docs/core/recommendations.md` §5.4 (precomputed path).

---

## 0. TL;DR for the PO

- **Gender is not a stored field today.** `tenant_products` has `product_type`
  (free-text) and `tags` (text[]) but **no gender column**. Gender lives only
  inside `tags` (the fashion fixture uses `women` / `men` / `unisex`). The
  title-string hack worked because titles happened to carry "Men's"; that is
  not reliable across merchants. **We must capture a normalized `gender` field
  at ingest** — the proper fix.
- **Category is fine as-is.** `product_type` is stored and is the right signal
  for a *soft* same-category boost. We do **not** hard-filter on it (that would
  kill the complementary cross-category recs the PO wants).
- **Shopify's standardized Product Category (taxonomy) is NOT captured** — the
  sync only reads merchant free-text `product_type`. Capturing the taxonomy is
  an optional later enhancement (§1.4); not required for this change.
- **The self-join already supports out-of-stock sources** — only the writer's
  source list filters them out. A one-line source-list change plus keeping the
  candidate-side availability filter delivers rule 4.
- **Count** reuses the existing `widget_config` config pattern; add a
  PDP-preview count (default 3) and have the serve path read it instead of a
  hardcoded `TOP_N`.
- **Net backend work:** 1 Flyway migration + ingest derivation (gender), 1
  Flyway migration (count config), 2 catalog port/query changes (gender filter +
  all-source list), writer + serve-path wiring. No LLM, no re-embedding, stays
  `served_from = precomputed`.

---

## 1. The gender / category signal in the data model (and the gap)

### 1.1 What is stored today (`tenant_products`, Flyway V1)
- `product_type VARCHAR(255)` — Shopify's merchant-editable `product_type`
  (fixture: "Scarves", "Hats", "Small leather goods"). **Category signal.**
- `tags TEXT[]` — Shopify tags (fixture: `women,accessories,gift,silk` /
  `unisex,winter,cashmere,gift`). **The only place gender appears today.**
- `title`, `description`, `price_min/max`, `available`, `excluded_reason`,
  `embedding vector(1536)`.
- **No `gender` column. No standardized category column.**

### 1.2 What the Shopify sync captures (`ShopifyPlatformCatalogSource.parseProduct`
and `ShopifyWebhookController`)
- Reads `product_type` and `tags` (both REST comma-string and webhook array
  forms handled). Maps to `PlatformProductData.productType` / `.tags`.
- **Does NOT read** Shopify's standardized `category` / product-taxonomy node.
  Only free-text `product_type` is captured.
- The embedding source string (`EmbeddingService`) is
  `title + description + product_type + sorted(tags)` — so gender influences the
  vector *only* weakly and implicitly, via whatever gender word sits in tags.
  That implicit signal is exactly what fails: cosine similarity does not respect
  it as a hard constraint.

### 1.3 The gap and the decision
**Decision: add a normalized `gender` attribute to `tenant_products`, derived at
ingest. Do not key gender off the title string, and do not rely on raw tags at
query time.** Rationale:
- The title hack is merchant-formatting-dependent and silently wrong when titles
  omit gender ("Silk Scarf, Terracotta" has no gender in the title).
- Raw tags are free-text and inconsistent (`women` vs `womens` vs `ladies`);
  filtering on `'women' = ANY(tags)` at query time bakes that fragility into the
  hot path and repeats it everywhere gender is needed.
- A **normalized, controlled-vocabulary column** computed once at ingest is the
  clean, reusable signal: the gift/self live ranker (`recommendations.md` §5)
  will want the same gender constraint later, so solving it in the data model
  pays off beyond the PDP preview.

**Proposed column:** `gender VARCHAR(10)` with a controlled vocabulary
`{ men, women, unisex, unknown }` (store `unknown` explicitly, never null-as-
magic). Derivation is a small pure classifier at ingest (§1.4).

### 1.4 Gender derivation at ingest (`GenderClassifier`, new, in `catalog` or
`platform-integrations`)
A deterministic, testable mapping from the product payload to the vocabulary,
checked in priority order:
1. **Tags (primary).** Normalize each tag (lowercase, trim, singularize common
   forms) and match against synonym sets:
   - men ← `men, man, mens, male, boys, menswear, him`
   - women ← `women, woman, womens, ladies, female, girls, womenswear, her`
   - unisex ← `unisex, all-gender, genderless`
2. **product_type (secondary),** same synonym match, if tags gave nothing.
3. **Conflict rule.** If both men and women match → `unisex` (a product tagged
   for both is genuinely cross-gender; do not pick one).
4. **Default** → `unknown` when nothing matches.

Notes:
- The synonym vocabulary is a small config/constant, reviewable in the PR and
  extendable per merchant feedback. Start English; the fashion demo is English.
- `gender` joins the **content hash** inputs (`ContentHash.compute`) so a tag
  edit that changes gender re-triggers the derivation and (where relevant) the
  downstream precompute. It does **not** need to change the embedding source —
  gender becomes a *filter*, not part of the vector.
- **Optional later enhancement:** capture Shopify's standardized `category`
  (taxonomy GID + name) in the sync and store it as `std_category`. It would
  sharpen the category soft-boost (§2.3) and give a reliable gender hint for
  apparel taxonomy nodes. Out of scope for this change; note as a follow-up for
  `backend-engineer` + catalog-data owner.

> **Honest caveat for the PO.** Gender quality is now a *catalogue data* quality
> problem, surfaced cleanly instead of hidden in embeddings. If a merchant tags
> nothing, every product is `unknown` and the hard filter degrades to "no gender
> constraint" (§2.2 fallback). This is the right failure mode (no worse than
> today) and it is measurable (§6, gender-coherence check), which the title hack
> never was. The catalogue checklist in `recommender-eval-plan.md` §4 (C4 gender
> tag) is the lever to fix it.

---

## 2. Retrieval / ranking design

The PDP preview stays **precomputed, no LLM, no re-embedding** — we only change
*which* neighbours survive and *how* they are ordered. Mechanism changes; policy
(precomputed, neutral, symmetric across holdout arms) is untouched.

### 2.1 Gender as a HARD filter (rule 1)
Within the pgvector self-join candidate set, keep only candidates whose gender
is **compatible** with the source product's gender. Compatibility (not strict
equality, so unisex works both ways):

| Source gender | Eligible candidate genders |
|---|---|
| men | men, unisex |
| women | women, unisex |
| unisex | men, women, unisex (no gender constraint) |
| unknown | no gender constraint (see 2.2 fallback) |

This is a hard `WHERE`-clause filter in the self-join, applied **before** the
cosine ordering, so a closer-but-wrong-gender neighbour can never outrank a
correct-gender one. This is the fix for the men's-tee-returns-women's bug.

### 2.2 Fallback when gender is `unknown`
- **Unknown source** → no gender filter (behaves like today; we cannot constrain
  what we cannot classify). The candidate pool is unfiltered by gender.
- **Known source, too few eligible candidates** (fewer than the configured
  count survive the gender filter) → **do not** pad with wrong-gender products.
  Return the short (gender-correct) slate; a 2-card correct slate beats a 3-card
  slate with one wrong-gender card. The serve path already tolerates short
  slates (`PdpPreviewService` returns whatever is serveable). Track short-slate
  frequency as a signal the catalogue is thin for that gender (§6).

### 2.3 Category as a SOFT boost, never a hard constraint (rule 2)
The PO explicitly wants same-gender **cross-category** recs (trousers for a
shirt). So category must **not** filter. Two viable mechanisms; recommend A for
MVP:

- **Option A (recommend): keep pure cosine order within the gender-filtered
  pool, plus the existing same-`product_type` diversity cap.** Embedding
  similarity already surfaces a mix of same-type and complementary same-gender
  items; the `recommendations.md` §5.4 same-type penalty (≤ 2 per `product_type`)
  already prevents a slate of 3 near-identical shirts. This is the smallest
  change, fully precomputable, and directly delivers "same gender, mixed
  category". No new scoring code.
- **Option B (later, if A under-delivers): an explicit soft re-rank.** Score =
  `cosine_similarity + w · same_category_bonus` with a small `w`, or a
  category-diversity-aware MMR. This needs a tuned weight and a labelled set to
  justify it (§6). Do not build it until A is measured as insufficient — added
  complexity must beat measured gains, not novelty (CLAUDE.md).

**MVP decision: Option A.** Gender is the only hard change; category stays an
emergent + diversity-capped soft signal.

### 2.4 Unisex / accessories handling
- Unisex products (scarf, beanie, card holder in the fixture) are eligible for
  **every** source gender (§2.1), which is correct: a unisex scarf is a fine
  complement to a men's shirt or a women's dress.
- A unisex *source* gets no gender constraint — its neighbours can be men's,
  women's or unisex. This is acceptable for genuinely genderless items;
  accessories are the common case and benefit from the wide pool.
- Items that are gendered but mis-tagged as unisex are a *catalogue* fix (C4),
  not a ranker fix.

---

## 3. How it plugs into the precompute writer and the serve path

Isolate policy from mechanism: the gender filter is a **mechanism** change,
contained in the catalog self-join; the writer/serve policy barely moves.

### 3.1 Precompute writer (`PdpPreviewPrecomputeService` + `CatalogQueryPort`)
Concrete changes for `backend-engineer`:
1. **Gender filter in the self-join** (`NearestProductFinderAdapter
   .STORED_PRODUCT_NN_SQL`): add a gender-compatibility predicate on the
   candidate `p` against the source `src.gender`. Because `src` is already
   joined in the query, the source gender is available without an extra read.
   Keep the existing candidate filters (`p.available = true`,
   `p.excluded_reason IS NULL`, `p.embedding IS NOT NULL`, `p.platform_product_id
   <> src`). The predicate implements the §2.1 table; when `src.gender` is
   `unknown` or NULL the predicate is a no-op (§2.2).
2. **Source list = all embedded, non-excluded products** (rule 4; §5): replace
   `findEmbeddedRecommendableProductIds` (which filters `available = true`) with
   a source list that **drops the availability filter** but keeps
   `excluded_reason IS NULL AND embedding IS NOT NULL`. Out-of-stock PDPs now get
   a preview; gift cards / unpublished products (no buyable PDP) still don't.
3. **Storage ceiling `TOP_N`** stays a *buffer*, not the serve count. Keep it ≥
   the maximum configurable serve count plus headroom for the serve-time
   availability trim (today `TOP_N = 5`, serve `FETCH_CEILING = 10`). If the
   configurable count can exceed 5, raise `TOP_N` to `maxConfigurableCount +
   buffer`. The writer stores gender-correct neighbours only, so the serve path
   never has to re-apply the gender filter.
4. Idempotency, per-source transaction, tenant-scoping: unchanged.

### 3.2 Serve path (`PdpPreviewService`)
- **No gender logic at serve time** — the stored preview is already
  gender-correct (precompute did the filtering). Serve keeps only the existing
  availability re-check (`isServeable`) and the availability-trim buffer.
- **One change:** the serve **count** comes from config, not a client-supplied
  or hardcoded `limit` (§4).
- `served_from = precomputed`, no cache, no holdout branch, neutral/symmetric —
  all unchanged. The lift-neutrality property (`decisions.md` 2026-10-19) is
  preserved.

### 3.3 Freshness interaction
Gender is derived at ingest and stored, so a re-tag flows: webhook/sync →
`gender` recomputed (content-hash includes it) → product re-embedded if content
changed → nightly (or threshold-triggered) PDP precompute rewrites that
product's neighbours and every slate that could include it. No live-path cost.

---

## 4. Configurable count (default 3), per placement (rule 3)

### 4.1 What exists
- `widget_config.recs_per_widget INTEGER NOT NULL DEFAULT 5` (Flyway V1) — the
  general "recommendations per widget" count, already consumed by budget-tier
  sizing. Spec range 3–10.
- `widget_config.pdp_preview_enabled BOOLEAN` (Flyway V16) — the PDP preview
  on/off toggle.
- `PdpPreviewService.preview(tenant, productId, limit)` already takes a `limit`
  param; `PdpPreviewPrecomputeService.TOP_N = 5` is a hardcoded storage ceiling.

### 4.2 Recommendation: a dedicated PDP-preview count column
The PDP preview is a distinct placement with distinct UX (in-hook preview, PO
wants **3**) from the main rec slate (default 5). Overloading `recs_per_widget`
would couple two independent knobs. Proposed:
- **New Flyway migration:** `widget_config.pdp_preview_count INTEGER NOT NULL
  DEFAULT 3`, with a `CHECK (pdp_preview_count BETWEEN 1 AND 10)`.
- **Serve path reads it:** `PdpPreviewService` (or its controller) resolves
  `pdp_preview_count` for the tenant and uses it as the slate size, rather than
  trusting the client `limit`. If a client `limit` is still passed, **clamp** it
  to the config value (config is the authority; never exceed it).
- **Admin surface:** expose it in the existing widget-config admin editor
  alongside `recs_per_widget` / `pdp_preview_enabled` (hand to
  `frontend-engineer` + `product-analyst` for the admin control; backend owns
  the config contract).
- **Writer ceiling:** keep `TOP_N ≥ max(pdp_preview_count range high, serve
  buffer)` so the stored buffer always covers the configured serve count after
  the availability trim. With the 1–10 range, set `TOP_N = 10` (or a small
  constant above the max) — still tiny (`sources × 10` rows).

Default 3 satisfies the PO's immediate ask without a code change per tenant; a
merchant can raise it to fill a wider strip.

---

## 5. Source = all products, recommend only available (rule 4)

- **Candidate (recommended) side stays available-only.** The self-join keeps
  `p.available = true AND p.excluded_reason IS NULL`; the serve path re-checks
  `isServeable`. We never recommend an out-of-stock product. Unchanged.
- **Source side drops the availability filter.** Today the writer's source list
  (`findEmbeddedRecommendableProductIds`) requires `available = true`, so an
  out-of-stock product gets **no** precomputed preview. Change the source list to
  all **embedded, non-excluded** products regardless of `available` (§3.1-2).
  The self-join's `src` join already imposes no availability predicate, so only
  the source-id list needs changing.
- **Result:** a shopper landing on an out-of-stock PDP sees available
  alternatives (same gender) — the sale-recovery play. This pairs naturally with
  gender-hard filtering: the alternatives are same-gender substitutes.
- **Edge:** an out-of-stock source with no embedding (never embedded while
  unavailable) still yields no preview — acceptable; embedding is the
  precondition for any similarity. Note for the catalog owner: ensure products
  are embedded even while out of stock if sale-recovery coverage matters.

---

## 6. Measurement / validation

Ties into `docs/engineering/recommender-eval-plan.md` (the PDP self-NN sanity
checks S4–S8 and the Tier-1/Tier-2 bars). **No ranking change without a metric**
(CLAUDE.md). Define these BEFORE the change merges:

1. **Gender-coherence check (new hard bar, extends eval-plan S5).** For a
   labelled set of gendered source products, **100%** of recommended cards must
   be gender-compatible per §2.1 (men source → men/unisex only, etc.). This is a
   hard pass/fail; one wrong-gender card on a known-gender source fails the
   build. Run as a SQL/spot check over `precomputed_recs` joined to
   `tenant_products.gender`.
2. **precision@k with the gender constraint on (eval-plan S4, B2).** On the
   fashion labelled set, precision@3 ≥ **0.7** and MRR ≥ **0.8** for the PDP
   preview *after* the gender filter — confirm the hard filter did not gut
   relevance (it should improve it).
3. **Cross-category presence (confirms rule 2 is not over-constrained).** Across
   the labelled gendered sources, a meaningful share of slates contain ≥ 1
   **different-`product_type`** same-gender card (the complementary case). If
   every slate is same-type, the diversity cap or the pool is too tight —
   investigate. This guards against accidentally turning category into a hard
   constraint.
4. **Gender classifier coverage.** Report the share of catalogue products
   classified `unknown`. A high share means the hard filter is silently
   degrading to no-op; route to the catalogue checklist (C4). Target for the
   demo: `unknown` share < **10%** of active products.
5. **Short-slate rate.** Share of known-gender sources whose gender-correct pool
   yields fewer than the configured count. A high rate on a specific gender
   flags catalogue imbalance (eval-plan §4 C7), not a ranker bug.
6. **Out-of-stock source coverage (rule 4).** Count out-of-stock sources that
   now have a non-empty preview vs. before (was 0). Confirms the sale-recovery
   change landed.
7. **Regression guard.** Re-run the full eval-plan S4–S12 on the fashion
   fixture; precision@k must be **≥** the pre-change baseline. A drop beyond the
   accepted-regression threshold (`recommendations.md` §2.3) blocks the merge
   without an explicit accepted-regression note.
8. **Determinism.** The gender filter is a deterministic SQL predicate; two
   precompute runs on an unchanged catalogue must produce identical
   `precomputed_recs` (eval-plan S8).

---

## 7. Concrete backend change list (for `backend-engineer`)

| # | Change | Where | Notes |
|---|---|---|---|
| 1 | Add `gender VARCHAR(10) NOT NULL DEFAULT 'unknown'` to `tenant_products` | new Flyway migration | controlled vocab `men/women/unisex/unknown` |
| 2 | `GenderClassifier` — derive gender from tags → product_type synonym sets, conflict→unisex, else unknown | new class, `catalog` or `platform-integrations` | pure, unit-tested; English vocab |
| 3 | Set `gender` on upsert; add it to `ContentHash.compute` inputs | `CatalogIngestService`, `ContentHash` | re-derive on tag/type edits |
| 4 | Map `gender` through `PlatformProductData` (or derive post-map in catalog) | `PlatformProductData`, Shopify parse | keep platform-neutral |
| 5 | Add gender-compatibility predicate to the self-join (no-op when source gender unknown/null) | `NearestProductFinderAdapter.STORED_PRODUCT_NN_SQL` | §2.1 table |
| 6 | New all-source list: embedded + `excluded_reason IS NULL`, **drop** `available` filter | `CatalogQueryPort` / `CatalogQueryService` (replaces/augments `findEmbeddedRecommendableProductIds`) | rule 4 source side |
| 7 | Add `pdp_preview_count INTEGER NOT NULL DEFAULT 3 CHECK 1..10` to `widget_config` | new Flyway migration | rule 3 |
| 8 | Serve count from config, clamp client `limit` to it | `PdpPreviewService` + controller / config reader | authority = config |
| 9 | Raise writer storage ceiling `TOP_N` to cover max config count + buffer | `PdpPreviewPrecomputeService` | e.g. 10; table stays tiny |
| 10 | Expose `pdp_preview_count` in widget-config admin contract | admin config API | hand UI to frontend-engineer/product-analyst |
| 11 | *(optional, later)* capture Shopify standardized `category` in sync → `std_category` | Shopify parse + migration | sharpens soft category + gender hint |

**Not changed:** `served_from = precomputed`, no cache, no LLM, no re-embedding,
holdout-neutral symmetry, tenant isolation, per-tenant kill-switch/cap guard.
One concern per branch (CLAUDE.md): suggest splitting into (a) gender ingest +
column, (b) self-join gender filter + all-source list, (c) count config.

---

## 8. Open questions for the PO

| # | Question | My recommendation |
|---|---|---|
| Q1 | Normalized `gender` column vs. query-time tag matching? | Column, derived at ingest — reusable by the live ranker, testable, not title-dependent. |
| Q2 | Category soft-boost: Option A (cosine + diversity cap) or B (explicit weight) for MVP? | A now; B only if measurement shows A under-delivers complementarity. |
| Q3 | Dedicated `pdp_preview_count` (default 3) vs. reuse `recs_per_widget`? | Dedicated column — the PDP preview is a distinct placement with a distinct count. |
| Q4 | Capture Shopify standardized category now or later? | Later; `product_type` is adequate for the soft boost. Flag to catalog owner. |
| Q5 | Unknown-gender source: no filter (proposed) vs. hide the preview? | No filter (degrade gracefully, same as today); measure the unknown share and fix via catalogue tags. |
| Q6 | Embed out-of-stock products so sale-recovery previews exist? | Yes if sale-recovery coverage matters; confirm with catalog owner (embedding cost is one-time). |
```
