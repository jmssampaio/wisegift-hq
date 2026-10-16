# Guided budget step: catalogue-derived tiers and the suggestion rule

> **DRAFT — pending PO review.** Written 2026-10-17 by `recommendations-specialist` for `decisions.md` 2026-10-17 "Budget step: guided named tiers from the catalogue" (Decision 1, phase 1). Nothing here is decided until the PO validates it. The wire contract in §3 is a **proposal**: `backend-engineer` owns the contract (`hooks-and-packs-contract.md` §3.4, PR 2).
>
> **Inputs:**
> - `decisions.md`:
>   - 2026-10-17 (variant C phased; Q-F3 vocabulary; exact-budget cache key)
>   - 2026-10-09 (overlap budget filter, a revision of 2026-10-08 "never over budget"; self + budget is real intent)
>   - the M1·2 entries of 2026-10-14 to 2026-10-16
> - `hooks-and-packs.md` §3.2 (pack budget bands) and §5 "Q-F3 resolution"
> - `hooks-and-packs-contract.md` §3.4 (`config.intentForm`)
> - `m1-2-lift-measurement.md`
> - the mockups `V2BudgetSlider`, `V2BudgetTiers` and `V2BudgetGuided` (variant C, Guided, was chosen)
> - code, read-only: `wisegift-backend` `origin/develop` at `bb94e0e`
>
> **Data caveat.** I could not read the staging catalogues: there was no database access in this session, and both storefronts are password-protected. The worked examples in §7 therefore use **illustrative** prices:
> - The 17-product store is reconstructed from the Shopify generated sample catalogue (snowboards, ski wax, gift card). `decisions.md` 2026-10-06 ("Staging deploy wrap") says that catalogue was synced, and its "$9.95 wax / $2629 snowboard" prices match.
> - The 47-product `wisegift-dev` catalogue is invented to be consistent with the PO's finding that "100–200 €" returned nothing.
>
> All numbers were computed with a reference script (§7.4), not by hand. **Before PO sign-off, `backend-engineer` should run the §7.4 query on staging and replace the examples.**

---

## 0. Summary

1. **Tiers come from the tenant's own prices.** We sort the in-stock products by the price a shopper would actually pay, which is the cheapest **available** variant. We then cut the list into up to 4 groups of roughly equal size, and move each cut to the nearest "nice" price (20, 25, 50, 75, 100, 150, …).
2. **Every tier must be able to fill a full row of recommendations:** at least `max(5, recs_per_widget, 10% of products)` products per tier. If 4 tiers can't meet that, we use 3, then 2. With fewer than about 10 priced, in-stock products, the budget step is hidden. With no catalogue yet, the pack's §3.2 bands are used.
3. **Tiers are stable.** They are recomputed after every successful full sync and price refresh, but the old tiers are **kept** unless:
   - a tier falls below the minimum;
   - a tier drifts more than 15 points away from an equal share;
   - the catalogue has grown enough, with margin, for an extra tier.
4. **The same price edges are used for gift and self.** Only the names differ (per pack × intent).
5. **Suggestion (gift only):** a fixed relationship × occasion table selects a tier in advance.
   - The suggestion is **never** the open-ended top tier.
   - Without relationship and occasion, nothing is suggested.
   - With only 2 tiers, nothing is suggested.
   - **Self: no suggestion.**
   - The copy says "We suggest" and nothing about what other people choose.
6. **Measurement:** four new nullable fields on `intent_submitted`, plus one server-side counter (empty-slate rate per budget type). Phase 2 (the slider) is triggered by the "Any price" share or by budget changes on re-submission, read from a minimum sample (§4.4).
7. **Cache and filter:** only tier edges are ever sent, so a tenant has at most 5 budget keys (4 tiers plus "Any price"), the same as with bands. Two code fixes are prerequisites: the exact-budget key (Decision 3) is **not yet on `origin/develop`**, and the price rollups include unavailable variants (§8, F1 and F3).

---

## 1. Tiering algorithm

Version label: **`tiers-v1`**. It is stored with every tier set and logged (§1.8).

### 1.1 Input population and the price per product

**Eligible product.** All of the following must hold:
- `tenant_products.tenant_id = :tenant`;
- `available = true`;
- at least one row in `tenant_product_variants` (same `tenant_id`) with `available = true` and `price > 0`.

**Price per product:** `p = MIN(variant.price)` over that product's **available** variants with `price > 0`. This is the cheapest price a shopper can actually buy today.

- **Not `tenant_products.price_min`.** That rollup includes unavailable variants (F1). A product whose only €20 variant is sold out would otherwise count in a €0–20 tier.
- **No embedding requirement.** Embeddings are written asynchronously (`EmbeddingBatcher` is `@Async`). Tiers computed right after a first sync would otherwise see zero products and fall back to pack bands for a new tenant. The lag is minutes; the nightly 02:30 backfill catches failures.
- **The guarantee still holds against the hard filter.** The retrieval filter is `price_min <= budgetMax AND COALESCE(price_max, price_min) >= budgetMin` (2026-10-09). For any product counted in tier `[a, b]` we have `price_min ≤ p ≤ b` and `price_max ≥ p ≥ a`, so it **passes the filter for that tier**. The filtered set is a superset of the tier's count.

Let `x₁ ≤ … ≤ x_N` be the sorted prices, and `N` the number of eligible products.

**Reference SQL** (non-normative; tenant-scoped in every clause):
```sql
SELECT MIN(v.price) AS p
  FROM tenant_products tp
  JOIN tenant_product_variants v
    ON v.tenant_id = tp.tenant_id AND v.tenant_product_id = tp.id
 WHERE tp.tenant_id = :tenant
   AND tp.available = true
   AND v.available = true
   AND v.price > 0
 GROUP BY tp.id
 ORDER BY p;
```

### 1.2 Number of tiers and minimum products per tier

- **Minimum per tier:** `M = max(5, recs_per_widget, ceil(0.10 × N))`.
  - **5 and `recs_per_widget`:** every tier can fill a full slate, which is the PO's "every tier has products" promise in measurable form.
  - **10%:** on large catalogues, no tier is a sliver.
- Try `k = 4`, then `3`, then `2`. Take the first `k` for which `N ≥ k × M` **and** a valid edge set exists (§1.4).
- If no `k ≥ 2` is valid, the source is `none` (§1.5).

### 1.3 "Nice" edges: the price ladder

Edges are **whole numbers in the shop currency's major unit**, because `budgetMin` / `budgetMax` are `Integer` in `IntentPayload` and `NearestProductFinder`.

**Ladder:** every integer of the form `m × 10ⁿ` (n ≥ 0) with `m ∈ {1, 1.5, 2, 2.5, 3, 4, 5, 6, 7.5}`:
`1, 2, 3, 4, 5, 6, 10, 15, 20, 25, 30, 40, 50, 60, 75, 100, 150, 200, 250, 300, 400, 500, 600, 750, 1000, 1500, …`

- Non-integers such as 1.5, 2.5 and 7.5 at n = 0 drop out.
- The ladder is currency-agnostic: JPY or HUF simply land in higher decades.

**Niceness rank** (used only to break ties; lower is nicer):

| Rank | Mantissas | Examples |
|---|---|---|
| 0 | 1, 2, 5 | 10, 20, 50, 100, 200, 500 |
| 1 | 2.5, 3 | 25, 30, 250, 300 |
| 2 | 1.5, 4, 6, 7.5 | 15, 40, 60, 75, 150, 750 |

**Why a ladder and not "round to a step by magnitude".** A step rule such as "round to 5 below 100, to 25 up to 500" produces edges like 35, 45, 65 or 175. A ladder of about 9 values per decade gives labels that read like retail price points. The cost is that the nearest ladder value can be up to about 25% from the raw quantile. §1.4 compensates by choosing edges for balance, not for proximity.

### 1.4 Choosing the edges (quantile target, brute force)

For a given `k`, tiers are `[0, e₁) [e₁, e₂) … [e_{k−1}, ∞)` for counting purposes. A product with price exactly `eⱼ` counts in the upper tier.

1. **Candidates:** the ladder values `v` with `x₁ < v ≤ x_N`.
2. **Enumerate** every strictly increasing combination of `k − 1` candidates. That is at most a few thousand combinations, which is microseconds.
3. **Reject** a combination if:
   - any tier has fewer than `M` products; or
   - any **middle** tier is narrower than ×1.5 (`e_{j+1} < 1.5 × eⱼ`). This avoids tiers like "25–30 €" that would be meaningless as named tiers. The first and the open top tier have no width rule.
4. **Score** the rest and keep the lowest, comparing lexicographically:
   1. **imbalance** = `max_j |countⱼ − N/k|`. The target is **equal product counts per tier**: the raw `j/k` quantiles (P25/P50/P75 for 4 tiers, P33/P67 for 3, P50 for 2).
   2. the sum of the niceness ranks of the edges;
   3. `Σ |ln(eⱼ / qⱼ)|`, where `qⱼ` is the raw `j/k` quantile (linear interpolation);
   4. the edges in ascending order (smallest wins). This makes the result fully deterministic, with no randomness to seed.

**How this handles skew.** Real catalogues are right-skewed: many cheap items and a long expensive tail. Quantiles are rank-based, so a few €2,000 items do not stretch the tiers; they simply sit in the open top tier. Gaps in the price list (for example nothing between €95 and €219) produce a wide tier ("50–500 €") instead of an empty one. Clusters of identical prices (many items at 29.95) stay together, because half-open intervals never split equal prices.

### 1.5 Sources and fallbacks

| Situation | Source | Tiers | Suggestion |
|---|---|---|---|
| A valid `k ∈ {4,3,2}` exists (§1.2–1.4) | `catalogue` | `k` tiers | §2 (none if `k = 2`) |
| `N = 0`: no successful sync yet, or no priced, in-stock product | `pack_default` | The pack's §3.2 bands as edges: General 50/100/200, Fashion 30/60/120, Home & Decor 40/100/250 | §2 |
| `N ≥ 1` but no valid `k ≥ 2` (fewer than about 10 products, or a very narrow price range) | `none` | `[]`. The widget **skips the budget step** and sends no budget, the same as "Any price". | none |
| A merchant override exists (later, §1.9) | `custom` | the merchant's edges | §2 |

- **Narrow range.** No special case is needed: when all prices sit within about ×1.5 of each other, the width and minimum rules leave no valid combination for `k ≥ 3`, and often none for `k = 2`. The result is fewer tiers, or `none`.
- **Why `none` and not pack bands for tiny catalogues.** With fewer than 10 products, a budget filter mostly removes products from a slate that is already short. The pack bands are exactly what produced the empty "100–200 €" result. The admin shows: "The budget question is hidden until your catalogue has at least 10 priced, in-stock products" (copy for `product-analyst`).
- **Merchant visibility `budget = hidden`** always wins: no step, whatever the source. `required` means "choose a tier or Any price".

### 1.6 When to recompute, and stability (hysteresis)

**Triggers.** Tiers are recomputed as a command, never inside a ranking call (CQRS):
- a **successful** full sync: nightly at 02:00 UTC, manual, or at install (`CatalogSyncService.syncFullCatalog` with status `OK`);
- a successful price and stock refresh (`CatalogRefreshService.refreshPricesAndStock` with status `OK`).

`FAILED` and `PARTIAL` runs never recompute: a broken sync must not reshape tiers. Product and inventory webhooks do not recompute either; their drift is absorbed by the nightly run. That needs 5 or more products in one tier to go out of stock within a day.

**Rule.** Let `T_old` be the active set and `T_new` the result of §1.4 on today's prices.
1. **No active set**, or `T_old.source ∈ {pack_default, none}` → adopt `T_new`.
2. **Invalid:** any `T_old` tier has fewer than `M` products today → adopt `T_new`. This is the only way `k` can **decrease**.
3. **Drift:** `max_j |shareⱼ − 1/k| > 0.15` for `T_old` on today's prices, and `T_new`'s drift is lower → adopt `T_new`. For 4 tiers, that means a tier now holds under 10% or over 40% of products.
4. **Growth:** `k_new > k_old` **and** `N ≥ 1.25 × k_new × M` → adopt `T_new`. The 25% margin stops a catalogue hovering around 20 products from flipping between 3 and 4 tiers.
5. Otherwise **keep `T_old`**, and log `kept` with the drift value.
6. `custom` sets are never replaced automatically (§1.9).

**Effect.** Most recomputes change nothing, because rounding to the ladder already absorbs small changes and the rules above absorb the rest (example in §7.2). A change creates a new tier-set version (§1.8), which readouts note like a form change (Q-R4).

### 1.7 Gift vs self

**The same edges.** There is one price distribution per tenant.

Why:
- one read model;
- comparable chosen-tier distributions across intents (§4);
- no data suggesting self shoppers need different cut points.

**What differs:** tier names (per pack × intent, §3.3) and the suggestion (gift only, §2.5). Revisit if §4 shows clearly different tier usage, for example self "Any price" share far above gift.

### 1.8 Storage (read model; proposal for `backend-engineer`)

New table `budget_tier_sets`. It is tenant-scoped and added to `TenantPurgeRepository.TENANT_SCOPED_TABLES`.

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | `UUID NOT NULL REFERENCES tenants` | |
| `version` | `INTEGER NOT NULL` | Per tenant, `max + 1`. PK `(tenant_id, version)` |
| `source` | `VARCHAR(16) NOT NULL` | `catalogue`, `pack_default`, `custom` or `none` (CHECK) |
| `edges` | `INTEGER[] NOT NULL` | Inner edges, ascending. `[]` means 1 tier (not used) or `none` |
| `currency` | `VARCHAR(3) NULL` | Shop currency (§1.10) |
| `product_count` | `INTEGER NOT NULL` | `N` at compute time |
| `tier_counts` | `INTEGER[] NOT NULL` | Products per tier at compute time |
| `algorithm_version` | `VARCHAR(16) NOT NULL` | `tiers-v1` |
| `active` | `BOOLEAN NOT NULL` | Exactly one active per tenant (partial unique index) |
| `computed_at` | `TIMESTAMPTZ NOT NULL` | |

- **Inactive rows are kept,** so that `budgetTierSetVersion` on events (§4.2) maps back to the actual edges. A row is a few hundred bytes, and a new one is created only on change.
- A `pack_default` set records the pack's edges at that moment, so a later pack switch creates a new version.
- **Placement:** catalogue owns prices, so a pure `BudgetTierCalculator` (mechanism) and a `BudgetTierPolicy` (the §1.6 rule) live in `catalog`. A `BudgetTierQueryPort` is consumed by the bootstrap. The module split is the backend's call.

### 1.9 Merchant overrides (later; shape only)

Not in phase 1. The shape, so that phase 1 doesn't block it:
- Admin "Budget tiers": **Automatic (recommended)** | **Custom**.
- Custom stores 1–3 inner edges as a `custom` tier set: positive integers, strictly increasing.
- The admin shows the live product count per tier from the same SQL, and warns, without blocking, when a tier has fewer than `M`.
- Optional per-language name overrides follow the hook-copy pattern (an override exists = an edit; "Reset to pack default").
- "Back to automatic" adopts `T_new` immediately.
- Endpoint sketch: `GET /admin/v1/widget/budget-tiers` (active set plus counts) and `PUT /admin/v1/widget/budget-tiers { mode, edges? }`.

### 1.10 Currency and formatting

- **Edges are in the shop's currency**, the currency the catalogue prices and the hard filter use.
- **Gap:** the backend does not know the shop currency today (F2).
  - **Proposal:** read `currency` from Shopify `shop.json` at install and on `shop/update` (`PlatformStoreInfo` gains `currency`), and store it as `tenants.currency VARCHAR(3)`.
  - Until it exists, the fallback is the modal `orders_attributed.currency` for the tenant, else `null`.
- **The widget formats** with `Intl.NumberFormat(<storefront locale>, { style: "currency", currency, maximumFractionDigits: 0 })`. The locale is the full `data-locale`, e.g. `es-ES`, so symbol placement and grouping follow the store ("30–60 €" in ES/PT, "€30–€60" in EN).
  - Ranges use `formatRange` where available, otherwise each end is formatted separately.
  - With `currency: null` the widget shows bare numbers.
- **Patterns** (EN; ES/PT through the Q-P6 translation pass):

  | Tier | Pattern | Example |
  |---|---|---|
  | first (`min = null`) | "Up to {max}" | Up to €20 |
  | middle | "{min}–{max}" | €20–€50 |
  | top (`max = null`) | "{min} and up" | €500 and up |
  | Any price | "Any price" | |

  "Up to" (not "Under") matches the filter, which is inclusive at the maximum.
- **Multi-currency storefronts** (Shopify Markets: `Shopify.currency.active ≠ currency`): see §6.3 and open question OQ-3.

---

## 2. Suggestion rule (MVP, rule-based)

Version label: **`suggest-v1`**.

### 2.1 Scale

The suggestion is a **level** on a 4-step scale: 0 "small treat", 1 "something special", 2 "to impress", 3 "go all out". It is then mapped to the tenant's actual tier count (§2.3).

**Level 3 is never suggested.** The top tier is open-ended. Pre-selecting "€500 and up" on a shopper's behalf puts unbounded prices in front of them that they did not ask for, which is a trust risk. The maximum suggestion is level 2.

### 2.2 Table: relationship × occasion → level

Canonical Q-F3 values. Derivation: a **base per relationship** (partner 2; family, friend, client and other 1; colleague 0), **+1 for wedding**, **−1 for just_because**, then clamped to 0–2, plus the hand adjustments marked **J**.

| relationship \ occasion | birthday | anniversary | christmas | wedding | baby | housewarming | just_because | other |
|---|---|---|---|---|---|---|---|---|
| **partner** | 2 | 2 | 2 | 2 | 2 **J** | 2 **J** | 1 | 2 |
| **family** | 1 | 2 **J** | 1 | 2 | 1 | 1 | 0 | 1 |
| **friend** | 1 | 1 | 1 | 2 | 1 | 1 | 0 | 1 |
| **colleague** | 0 | 0 | 0 | 1 | 0 **J** | 0 | 0 | 0 |
| **client** | 1 **J** | 1 **J** | 1 **J** | 1 **J** | 1 **J** | 1 **J** | 0 **J** | 1 **J** |
| **other** | 1 | 1 | 1 | 2 | 1 | 1 | 0 | 1 |

**Judgement calls (J), for the PO to accept or change:**
- **family × anniversary = 2:** typically a parents' or grandparents' milestone.
- **partner × baby / housewarming = 2:** these are rare combinations. Kept at the partner level rather than lowered.
- **colleague × baby = 0:** often a group collection, with individual gifts small. Lowered from the +0 default.
- **The client row is flat at 1, never 2,** even for a wedding. Corporate-gift norms and anti-bribery policies often cap value. "Something special" is the safe default.
- **just_because = 0 for everyone but partner:** by definition it is not an occasion.
- Everything else follows the base + modifier rule with no adjustment.

### 2.3 Mapping a level to the tenant's tiers

| Tenant tiers | Level 0 | Level 1 | Level 2 |
|---|---|---|---|
| 4 | tier 0 | tier 1 | tier 2 |
| 3 | tier 0 | tier 1 | tier 1 |
| 2 | **no suggestion** | **no suggestion** | **no suggestion** |
| `none` | no step | | |

- **With 3 tiers,** levels 1 and 2 both go to the middle tier, so the open top tier is never pre-selected.
- **With 2 tiers there is no suggestion.** One half of the catalogue against the other is too coarse to suggest honestly, and the only non-top tier would be suggested every time.

### 2.4 Missing relationship or occasion

| Relationship | Occasion | Rule |
|---|---|---|
| set | set | Table cell |
| set | missing (optional or hidden) | The relationship's `other` column |
| missing | set | The `other` row for that occasion |
| missing | missing | **No suggestion.** Nothing is pre-selected. |

Values the backend nulled as unknown (Q-F3 validation) count as missing.

### 2.5 Self intent: no suggestion (recommendation)

Self intent gets **no suggestion and no pre-selection.**

1. **There is no signal.** Without a recipient or an occasion, a middle-tier default would be an arbitrary anchor, not advice.
2. **Cost.** Self + any budget passes `RealIntentCheck` (2026-10-09), so a pre-selected tier would turn every self submission into a live LLM call, including shoppers who never chose a budget. Without pre-selection, self + "Any price" + no interests is served from `precomputed`, as today.
3. **Measurement.** An unprompted self choice is the cleanest demand signal for the phase 2 slider decision (§4.4).

### 2.6 Widget behaviour and copy rule

**Behaviour**
- The suggested tier is shown **selected**, with a visible badge reading **"We suggest"** (ES "Te sugerimos", PT "Sugerimos"; drafts under Q-P6).
- Optional helper line: "Based on who it's for and the occasion. You can change it."
- **The suggestion updates live** while relationship or occasion change, **until the shopper touches the budget control.** From then on it is never changed automatically.
- **Never:**
  - "Most people choose", "Popular", "Best value";
  - any count or percentage of other shoppers;
  - any claim about what others spend.
  This holds until real data exists **and** the PO approves the copy (Decision 1).
- **Accessibility:**
  - The tiers form a native radio group.
  - The badge is text, not colour alone.
  - The suggested option's accessible name includes "We suggest".
  - Pre-selection sets `checked` without moving focus.

### 2.7 Holdout: no lift bias

- The tier set, the suggestion matrix, the badge and the update behaviour are part of the bootstrap `intentForm`. That is identical for both arms (`hooks-and-packs-contract.md` §6), and the widget must not read `isHoldout` to change the budget step.
- Everything happens **before** `intent_submitted`, the trigger (`m1-2-lift-measurement.md` §1.3). The triggered population is therefore not selected by the arm, and **the lift comparison stays unbiased.**
- **Limit, stated honestly:** the suggestion is part of the treatment. Holdout shoppers also see it, but they get no recommendations, so its effect on revenue flows only through the exposed arm's recommendations.
  - The holdout measures **WiseGift including the suggestion vs no WiseGift.** It can **not** isolate the suggestion's own contribution.
  - That needs an A/B test inside the exposed arm, with a per-experiment salt (G12; roadmap item 6). It is not phase 1.
- The deploy of the step, and every tier-set version change, is noted in readouts like any form change (Q-R4).

---

## 3. Wire contract proposal (bootstrap)

> **Proposal.** The backend owns the contract. This slots into PR 2's `config.intentForm` (`hooks-and-packs-contract.md` §3.4) and **replaces `budget.bands`**. Wire semantics stay `budget_min` / `budget_max` (Decision 1).

### 3.1 Shape

```json
"intentForm": {
  "schemaVersion": 4,
  "budgetTiers": {
    "version": 7,
    "source": "catalogue",
    "currency": "EUR",
    "tiers": [
      { "min": null, "max": 20 },
      { "min": 20,   "max": 50 },
      { "min": 50,   "max": 500 },
      { "min": 500,  "max": null }
    ],
    "suggestion": {
      "ruleVersion": "suggest-v1",
      "byRelationship": {
        "partner": { "birthday": 2, "anniversary": 2, "christmas": 2, "wedding": 2, "baby": 2, "housewarming": 2, "just_because": 1, "other": 2 },
        "family":  { "birthday": 1, "…": 1 },
        "other":   { "birthday": 1, "…": 1 }
      }
    }
  },
  "gift": {
    "budget": {
      "label": "What's your budget?",
      "visibility": "required",
      "tierLabels": [
        { "labelKey": "budget.gift.general.treat",   "label": "A small treat" },
        { "labelKey": "budget.gift.general.special", "label": "Something special" },
        { "labelKey": "budget.gift.general.impress", "label": "To impress" },
        { "labelKey": "budget.gift.general.all_out", "label": "Go all out" }
      ],
      "anyPriceLabel": "Any price",
      "suggestedBadge": "We suggest",
      "suggestionHelper": "Based on who it's for and the occasion. You can change it."
    }
  },
  "self": {
    "budget": {
      "label": "What's your budget?",
      "visibility": "required",
      "tierLabels": [ "… self labels …" ],
      "anyPriceLabel": "Any price"
    }
  }
}
```

### 3.2 Field rules

- **`budgetTiers` sits at the form level** because the edges are shared by gift and self (§1.7). The labels sit per intent.
- **`source`** is one of `catalogue | pack_default | custom | none`. With `none`, `tiers` is `[]`, `suggestion` is `null`, and the widget skips the step.
- **`tiers`:**
  - `min: null` on the first tier; `max: null` on the top tier.
  - Integers in `currency`'s major unit.
  - 2 to 4 entries, ordered.
- **`currency`** is ISO 4217, or `null` (§1.10).
- **`suggestion`:**
  - It is `null` when `k = 2`, when the source is `none`, or for self, which has no entry.
  - Values are **already mapped to the tenant's tier indices** (§2.3), so the widget does only a lookup: `byRelationship[rel ?? "other"][occ ?? "other"]`, with no suggestion when both are missing.
  - The backend sends only the visible relationship × occasion keys for the pack. The size is under 0.5 KB.
- **Why there is no single `suggestedTierIndex`:** the suggestion depends on answers given in the same form after the bootstrap. A single value would need a second request. The resolved matrix keeps the policy server-side (versioned, testable) and the widget logic trivial.
- **`tierLabels`** has the same length and order as `tiers`. When `k < 4`, the backend selects names:
  - `k = 3`: `[treat, special, all_out]`;
  - `k = 2`: `[special, all_out]`.
  `labelKey` is stable for tests and analytics; `label` is resolved for pack × intent × language (EN, with ES/PT falling back to EN until translated, as with hook copy).
- **What the widget sends** on `/recommendations` (canonical, so the cache keys line up):
  - for a tier: `budget_min = tier.min`, `budget_max = tier.max`, **omitting** null bounds;
  - for "Any price": neither bound.
  - The backend also canonicalises `budget_min = 0` to absent.
- **Holdout:** `budgetTiers` is identical for both arms. `isHoldout` is never read here.
- **`schemaVersion` does not bump on a tier recompute.** Tiers are catalogue data, not form content. `budgetTiers.version` tracks them instead, and it is echoed on events (§4.2).

### 3.3 Tier names per pack × intent (EN drafts, for `product-analyst`)

| Pack / intent | Tier 0 (`treat`) | Tier 1 (`special`) | Tier 2 (`impress`) | Tier 3 (`all_out`) |
|---|---|---|---|---|
| General · gift | A small treat | Something special | To impress | Go all out |
| Fashion · gift | A little something | Something special | A statement piece | Go all out |
| Home & Decor · gift | A small touch | Something special | A real centrepiece | Go all out |
| General · self | Keep it simple | A step up | Treat yourself | Top of the range |
| Fashion · self | Everyday | A step up | An investment piece | Top of the range |
| Home & Decor · self | Small touches | A step up | A statement piece | Top of the range |

- The General · gift row is the PO's wording from Decision 1.
- The other rows are drafts and are not ranker inputs: the ranker sees only `budget_min` / `budget_max`. Changing a name has no effect on ranking or the cache.
- Budget field label: "What's your budget?" for both intents, per §3.2 of the packs spec.

---

## 4. Measurement plan

### 4.1 Questions we need to answer
1. **Does every tier return a full slate?** This is the original bug.
2. **Do the tiers fit shoppers?** Shown by the "Any price" share and the tier usage spread.
3. **Is the suggestion helpful or just an anchor?** Shown by acceptance, passive vs active, and the direction of overrides.
4. **Is there demand for exact amounts (phase 2)?** Shown by "Any price" and budget changes on re-submission.
5. **Integrity:** is the step symmetric across arms?

### 4.2 New fields on `intent_submitted` (event schema change)

The events API is strict (`FAIL_ON_UNKNOWN_PROPERTIES=true`). Today `EventRequest` carries no budget information, so **new fields are required**. Four nullable fields, accepted **only on `intent_submitted`** (ignored on other types, like `preIntentRecs`):

| Wire field | Column on `events` | Type and validation | Why it is needed |
|---|---|---|---|
| `budgetTierSetVersion` | `budget_tier_set_version INTEGER NULL` | > 0 | Maps tier indices back to edges, separates `catalogue` / `pack_default` / `custom`, and splits readouts across recomputes |
| `budgetSuggestedTier` | `budget_suggested_tier SMALLINT NULL` | 0–3, null = no suggestion shown | Without it, acceptance cannot be told apart from coincidence |
| `budgetChoice` | `budget_choice VARCHAR(10) NULL` | `tier0`, `tier1`, `tier2`, `tier3`, `any_price` or `not_asked` (CHECK) | The chosen tier and the "Any price" share. `not_asked` covers source `none` and budget hidden |
| `budgetInteracted` | `budget_interacted BOOLEAN NULL` | | `true` if the shopper changed the budget selection at least once. Separates passive acceptance of the pre-selection from an active confirmation. Without it, a high acceptance rate is uninterpretable |

**Not added:**
- **Amounts:** derivable from the version and the index.
- **Relationship and occasion:** not needed for these metrics. Adding them is a separate privacy decision (Q-F3 resolution, "Future logging").
- **A switch counter:** re-submissions already produce extra `intent_submitted` events (§4.3).

**No new PII.** The fields are coarse indices with no free text and no amounts, but they are shopper-provided intent attached to a pseudonymous session. **`security-and-privacy` must review them before the widget sends them.** Retention is the same as `events` (120 days, D5). Older bundles send none of these fields, so they are stored as nulls.

### 4.3 Server-side counter (not an event change)

- **What:** the empty-slate and short-slate rate per budget kind.
- **Where:** the recommendation service, which already knows the budget and the slate size.
- **Proposal:** two nullable columns on `cost_telemetry`:
  - `budget_kind VARCHAR(8)`: `none`, `tier` or `custom`, where `tier` means the bounds equal an edge pair of the active set;
  - `result_count SMALLINT`.
- Equivalent alternative: a structured log line. That is the backend's call.

### 4.4 Metrics, success criteria, phase 2 trigger

Rules that apply to every metric:
- **First submission per session**, unless stated otherwise. It happens before any treatment, so both arms pool for descriptive metrics.
- Gift and self are reported **separately**.
- Each metric is reported per tenant and fleet-wide.

| # | Metric | Definition | Phase 1 success | Reads as |
|---|---|---|---|---|
| B1 | **Short-slate rate (tier)** | tier-bounded rec requests with `result_count < recs_per_widget` ÷ tier-bounded rec requests (exposed, server-side) | **≤ 2%** (guaranteed by construction, except embedding lag) | the bug is fixed |
| B2 | **"Any price" share** | `budgetChoice = any_price` ÷ submissions where the step was shown | **≤ 25%** | tiers fit shoppers |
| B3 | Tier usage spread | share per chosen tier | no tier below 5% for 4 tiers (diagnostic only) | tiers are used |
| B4 | **Suggestion acceptance** | chosen = suggested ÷ gift submissions with a suggestion, split by `budgetInteracted` | none (descriptive). Flag if passive acceptance is above 85% (pure anchoring) or overall acceptance is below 30% (wrong rule) | default helpful vs ignored |
| B5 | Override direction | among overrides, the share going **lower** than the suggestion | recalibrate the table if above 70% or below 30% | the rule is too high or too low |
| B6 | **Budget change on re-submission** (exposed only) | exposed triggered sessions with 2 or more `intent_submitted` where `budgetChoice` changed ÷ exposed triggered sessions | **≤ 15%** | shoppers are not hunting for a different range |
| B7 | **A/A integrity** | B2 and B4 compared between arms on **first** submissions, with a chi-square test | p ≥ 0.01. Below that, suspect instrumentation, as with CTR (P2) | symmetric step |

**Why B6 is exposed-only:** re-submission happens *after* the arms diverge (exposed sees recs, holdout sees the collapsed slot), so its rate differs by arm by design. It is a treatment-response diagnostic, not an A/A check.

**Phase 2 (slider) trigger.** Evaluated fleet-wide, and per tenant where the tenant alone has the sample. **Any one of:**
- B2 "Any price" share **> 25%**;
- B6 **> 15%**;
- at least 2 pilot merchants ask for exact amounts.

The B2 and B6 conditions must hold over **2 consecutive 30-day windows**, each with **≥ 400 first submissions** for that intent.
- **Why 400:** with 400 answers, a share around 25% is known to about **±4 points** (95%). "100 of 400 picked Any price" means "between about 21% and 29%". Below 400, a 30% reading could be noise.
- **The 25% and 15% thresholds are judgement calls.** There is no baseline yet. They mean "1 in 4 shoppers decline our ranges" and "1 in 7 hunt for a different range after seeing results". Revisit them after the first 400 submissions.

**What does not trigger the slider:** a low B4 or a lopsided B5. Those recalibrate the **suggestion table** (`suggest-v2`), not the step design.

**What phase 1 does not claim.** No revenue claim is made for tiers or for the suggestion. Holdout lift measures WiseGift as a whole (§2.7). Before/after comparisons of empty-slate or conversion are confounded by season and catalogue changes, so they are reported as observations only.

### 4.5 Offline evaluation (before the first pilot)

1. **Unit tests pin:**
   - the ladder and the niceness ranks;
   - the §7 worked examples, as golden tests;
   - every fallback branch;
   - every hysteresis branch;
   - the full §2.2 matrix and the §2.3 mapping;
   - the canonical `budget_min` / `budget_max` the widget sends per tier.
2. **Property tests** over random catalogues (seeded), which must always hold:
   - every adopted `catalogue` tier has at least `M` products;
   - for each tier, the hard-filter SQL returns at least `M` products from the same fixture;
   - edges are strictly increasing ladder values.
3. **Staging check:** run §7.4 on both staging tenants and record the tiers in the PR.

---

## 5. Interaction with the hard budget filter and the cache key

### 5.1 Hard filter (2026-10-09 overlap semantics)
- **Unchanged.** Tiers produce `budget_min` / `budget_max`, and the filter applies them as before.
- **Coverage is guaranteed.** Every product counted in a tier passes that tier's filter (§1.1), so the filtered candidate set is at least `M`. B1 monitors this.
- **Boundary prices.** A product priced exactly at an edge (e.g. €50.00 with edge 50) passes the filter of both adjacent tiers, because the filter is inclusive on both ends. This is harmless: it is never over budget, and it slightly enlarges both sets.
- **"Never over budget" is currently weakened** by F1. A product whose cheap variant is sold out still has that cheap `price_min`, so it passes a low tier's ceiling while only expensive variants are buyable, and the card shows the sold-out price. Fixing F1 is a **prerequisite** for honest tiers.

### 5.2 Cache key (Decision 3: exact `min-max`)
- **Bounded keys.** In phase 1 the widget sends only tier edges or no budget. Per tenant and active tier set there are therefore at most **k + 1 ≤ 5 distinct budget components**: the same count as the old 4 bands plus "Any price". Bucketing stays reasonable, with **no fragmentation.**
- **Recompute.** A new tier set changes the keys of the tiers whose edges moved. Old entries expire within the 24 h TTL. Expect a cold cache for those tiers for one day, and only on actual changes, which hysteresis makes rare.
- **Older bundles** that still send pack bands get exact keys of their own. These are bounded too, and short-lived.
- **Prerequisite:** `SignatureComposer` on `origin/develop` (`bb94e0e`) **still uses the 0/25/75/200/500 buckets** (F3). With buckets, catalogue tiers from different recomputes or tenants can share a bucket and serve over-budget cached products, which is the leak Decision 3 closed. The exact-budget key must ship **before or with** this step.
- **Phase 2 note.** A free slider would make budget keys unbounded. When phase 2 is built, the slider should snap to ladder values so keys stay bounded. That is a decision for then.

### 5.3 Serving policy (cost)
- **Gift:** budget alone is not real intent (2026-10-09), so this step does not change gift LLM volume.
- **Self:** a chosen tier makes the request live-LLM eligible, as today with bands. With no pre-selection for self (§2.5), self LLM volume stays tied to deliberate choices.
- `precomputed_recs` has no intent dimension and is unaffected.

---

## 6. Edge cases

### 6.1 Products without a price
- **Excluded from tiering:** a variant price that is `null` or `≤ 0` never counts. That covers free samples and "price on request".
- A product with no priced, available variant is not eligible (§1.1).
- **Retrieval:** budget-bounded requests already exclude `price_min IS NULL`. But `NearestProductFinderAdapter` maps a `null` `price_min` to `0` for "Any price" slates, so such a product renders as free (F4). That is a catalogue-quality fix, flagged to `catalog-data-engineer` and `backend-engineer`.

### 6.2 Sale price vs regular price
- **Use the current selling price: Shopify variant `price`.** On a sale, `price` is the reduced price and `compare_at_price` is the struck-through "was" price.
- The shopper's budget is what they pay, and the filter and the card already use `price`. The backend does not read `compare_at_price`, and should not for tiering.
- Market- or B2B-specific price lists are not ingested. The base price is used.
- A sale run through a price refresh can move tiers only through the §1.6 rules. A store-wide sale that shifts every price by the same factor causes no drift (shares unchanged), so tiers stay put.

### 6.3 Multi-currency stores (Shopify Markets)
- The catalogue, the filter and the edges are in the **shop currency**.
- **When the storefront's active currency equals the shop currency** (both staging stores, and likely most EU single-market pilots), nothing special happens.
- **When they differ** (for example an EUR shop selling to the UK in GBP):
  - **Proposal (OQ-3):** the widget converts **for display only** with `Shopify.currency.rate`, re-rounds to the ladder, and prefixes "about" ("Up to about £20").
  - It **sends the shop-currency edges unchanged**, so the filter and the cache stay exact in shop currency.
  - The shopper-visible range is approximate, to within the ladder rounding.
  - Alternative: hide the amounts and show tier names only when the currencies differ.
  - This needs PO input before a multi-market pilot.
- **No FX in the backend**, consistent with M1·2 §3.1.

### 6.4 Out-of-stock items
- **Excluded,** at both the product and the variant level (§1.1).
- **Caveat (F5):** the parser marks a variant unavailable when `inventory_quantity ≤ 0`, even if the merchant sells on backorder (`inventory_policy = continue`). Pre-order-heavy stores would get thinner tiers and recs. This is a catalogue question for `catalog-data-engineer`; tiering follows whatever `available` means.

### 6.5 Other cases
- **Unpublished products (F6):** sync does not filter on sales-channel publication, so products not published to the Online Store may count in tiers but cannot be bought. They inflate counts slightly. To verify with `backend-engineer`; fixing it fixes retrieval too.
- **Gift cards:** they count like any product (e.g. Shopify's €10–100 gift card has price €10). Whether gift cards should be recommendable is a catalogue policy question, not a tiering one.
- **Pack switch:** tiers are unaffected when the source is `catalogue`. Labels change with the pack. A `pack_default` set is recomputed on a pack switch, and the readout notes it.
- **Very large catalogues** (10k+ products): one aggregate query per recompute, `O(N log N)`, at sync time only. The bootstrap reads one row.

---

## 7. Worked examples (illustrative prices; see the data caveat)

### 7.1 New dev store `dummy-store-3x0otvhy` (17 products, Shopify sample catalogue; EUR assumed)
- **Products:** 17. One ("Out of Stock Snowboard", €885.95) is excluded, so `N = 16`.
- **Prices `p`:** 9.95 (ski wax), 10.00 (gift card, cheapest variant), then 14 snowboards: 600.00, 629.95, 629.95, 699.95, 729.95, 749.95, 749.95, 785.95, 885.95, 885.95, 949.95, 1025.00, 2629.95, 2629.95.
- `M = max(5, 5, ceil(1.6)) = 5`.
- **k = 4:** needs `N ≥ 20`. Not possible.
- **k = 3:** needs 15 ≤ 16.
  - The first tier needs 5 products, which forces `e₁ ≥ 750` (9 products below 750).
  - The middle-tier width rule then forces `e₂ ≥ 1125`, i.e. 1500 on the ladder. That leaves 2 products on top (fewer than 5).
  - **No valid set.**
- **k = 2:** the only valid edge is **750** (9 / 7 products; the raw median is 749.95).
- **Result:** `source = catalogue`, 2 tiers:
  - **"Something special: Up to €750"**
  - **"Go all out: €750 and up"**
  - plus "Any price".
- **Suggestion:** none (`k = 2`).
- **For comparison, the General pack bands on the same prices:** 2 / **0** / **0** / 14 products for Under 50 / 50–100 / 100–200 / Over 200. That is exactly the empty "100–200 €" the PO saw. This catalogue is bimodal (two cheap items, then nothing until €600), and the algorithm refuses to invent a tier in the gap.

### 7.2 `wisegift-dev` (47 products; prices invented, consistent with "no products at 100–200")
- **Products:** 47. The 17 sample products above, plus 30 accessories and hardware. Excluded: 1 out-of-stock snowboard, 1 out-of-stock accessory and 1 product priced €0. So `N = 44`.
- **Accessory prices:** 8.50, 12, 14.95, 15, 18, 19.95, 22, 24.95, 25, 29, 29.95, 32, 34.95, 39.95, 45, 49, 49.95, 55, 59.95, 65, 79, 89.95, 95, 219, 249, 279, 329, 449.
- `M = max(5, 5, ceil(4.4)) = 5`.
- Raw quartiles: 28.00 / 72.00 / 647.45.
- **k = 4 is valid.** Best edges **20 / 50 / 500**, with 8 / 11 / 11 / 14 products (imbalance 3).
  - `25 / 50 / 500` ties on imbalance (10 / 9 / 11 / 14) but loses on niceness (25 has rank 1).
- **Result:**
  - "A small treat: Up to €20"
  - "Something special: €20–€50"
  - "To impress: €50–€500"
  - "Go all out: €500 and up"
  - "Any price"
- "To impress" is wide because the catalogue has a real gap between €95 and €219.
- **Suggestions** (4 tiers, so level = tier):
  - partner × birthday → tier 2 "To impress";
  - friend × christmas → tier 1;
  - colleague × just_because → tier 0;
  - relationship and occasion both missing → none.
- **Stability check.** The next sync adds 3 products (€42, €120, €160) and removes 2 (€55, €279), so `N = 45`.
  - `T_new` would be 25 / 50 / 500.
  - `T_old` (20 / 50 / 500) on the new data holds 8 / 12 / 11 / 14 products, every tier ≥ 5. Its drift is 7.2 points, under the 15-point limit, and `k` is unchanged.
  - **Kept.** No new version, no cache churn.

### 7.3 A typical mid-size store (for contrast; synthetic)
- 400 products, log-normal around a €45 median (seed 42), so `M = 40`.
- **Edges 30 / 50 / 75,** with 115 / 94 / 81 / 110 products.
- The middle tiers sit exactly at the ×1.5 width minimum (50/30 = 1.67; 75/50 = 1.5).

### 7.4 Reproducing on staging

Run the §1.1 SQL per tenant, then the reference script with those prices. `backend-engineer` should paste the real outputs into this section before sign-off.
- Reference script: `tiers_ref.py`, in this session's scratchpad. It is **not** a deliverable; the Java `BudgetTierCalculator` plus golden tests replace it.

---

## 8. Findings in the current code (`origin/develop` `bb94e0e`)

| # | Finding | Where | Effect | Proposed owner |
|---|---|---|---|---|
| **F1** | `price_min` / `price_max` roll up over **all** variants, including unavailable ones | `ShopifyPlatformCatalogSource.parseProduct`, `ShopifyWebhookController` product parsing, `CatalogRefreshService.updateProductVariants` | **Over-budget leak:** a product passes a low tier's ceiling because of a sold-out cheap variant, and the card shows that price. Weakens "never over budget". **Prerequisite** | backend-engineer |
| **F2** | The shop currency is never stored. `tenant_products.currency` reads a `currency` field that Shopify REST `/products.json` does not return. `PlatformStoreInfo` has only the shop id and domain | platform-integrations, catalog | Tier labels cannot be formatted. Proposal: `tenants.currency` from `shop.json` | backend-engineer |
| **F3** | The exact-budget cache key (Decision 3) is not implemented: `SignatureComposer` still buckets at 0/25/75/200/500 | recommendation | Bucket collisions between tiers lead to over-budget cached slates. **Prerequisite** | backend-engineer |
| F4 | A `null` `price_min` is mapped to `0` in retrieval results | `NearestProductFinderAdapter` | Unpriced products render as free on "Any price" slates | backend-engineer / catalog-data-engineer |
| F5 | A variant is unavailable when `inventory_quantity ≤ 0`, regardless of `inventory_policy = continue` | `ShopifyPlatformCatalogSource.variantAvailable` | Backorder stores get thinner catalogues | catalog-data-engineer |
| F6 | No publication or status filter on the product fetch | `ShopifyPlatformCatalogSource` | Unpublished products may be counted and recommended | backend-engineer (verify) |

---

## 9. Open questions

| # | Question | Recommendation | Owner |
|---|---|---|---|
| OQ-1 | Accept the minimum per tier `M = max(5, recs_per_widget, 10% of N)` and "hide the step below about 10 products" (instead of pack bands)? | Yes | PO |
| OQ-2 | Accept "never pre-select the open-ended top tier" and "no suggestion with 2 tiers"? | Yes | PO |
| OQ-3 | Multi-currency storefronts: approximate converted labels ("about £20") or tier names only? | Converted labels with "about" | PO + frontend-engineer |
| OQ-4 | The judgement cells in §2.2, especially the flat client row and colleague × baby = 0 | As drafted | PO |
| OQ-5 | Self: no suggestion (§2.5)? | Yes | PO |
| OQ-6 | Four new `intent_submitted` fields (§4.2): privacy review | Ship after `security-and-privacy` sign-off | security-and-privacy |
| OQ-7 | Phase 2 thresholds (25% "Any price", 15% re-submission change, 400 first submissions × 2 windows) | Accept as provisional; revisit after the first 400 | PO |
| OQ-8 | Tier names for Fashion, Home & Decor and self (§3.3), and their ES/PT translations | `product-analyst` to finalise | product-analyst + PO (Q-P6) |
| OQ-9 | Should gift cards and €0 or unpublished products be recommendable at all (F4, F6, §6.5)? | Catalogue policy, outside tiering | catalog-data-engineer |
| OQ-10 | Sequencing: F1 and F3 before the step ships; F2 before labels can show a currency | Yes | backend-engineer |
