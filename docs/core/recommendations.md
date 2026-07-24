# WiseGift — Recommendation Design & Evaluation

## 1. What "good" means (evaluation targets)

A recommendation is good when a user, presented with the top-5 gift suggestions for
a specific recipient and occasion, finds at least one item they consider genuinely
appropriate to buy. That is the human ground truth. Everything below is a proxy for it.

### Primary metrics

| Metric | Definition | Target (v1 baseline) |
|---|---|---|
| Precision@5 | Fraction of the 5 results rated relevant by the gifter after seeing them | >= 0.6 (3 of 5 relevant) |
| Click-through rate (CTR) | Fraction of results where the affiliate link is opened | Track; no v1 target yet |
| Wishlist save rate | Fraction of recommended products saved by the gifter | Track; no v1 target yet |
| Per-category coverage | Count of active products per canonical category | >= 100 per category before launch |
| Intra-list diversity | Mean pairwise cosine distance of the 5 result embeddings | >= 0.25 (results should not cluster) |

CTR and save rate require instrumentation that does not yet exist. Define the
instrumentation contract with the backend-expert before shipping the AI Curator
to a user base large enough to generate signal.

Precision@5 can be measured in early access by presenting users with a 3-tap
"was this gift relevant?" prompt after each curator session. Collect a minimum of
200 rated sessions before drawing conclusions.

### Secondary metrics (leading indicators)

- Budget hit rate: fraction of results with `price <= maxBudget`. Should be 100 %;
  any miss is a pipeline bug.
- Country filter compliance: fraction of results with `country = residenceCountry`.
  Should be 100 %; any miss is a pipeline bug.
- Null explanation rate: fraction of LLM responses where the per-gift explanation
  is empty or generic ("This is a great gift"). Should be < 5 %.

### What we are NOT measuring (v1)

- Collaborative filtering signals (no purchase history, no engagement graph yet).
- A/B test lift. No traffic volume to split at this stage.
- Conversion to purchase. We do not have affiliate network conversion data at launch.

---

## 2. Inputs (gifting context)

The `RecipientProfile` POSTed to `POST /api/v1/gifts/recommendations` carries:

| Field | Role |
|---|---|
| `name` | Used in LLM prompt for personalisation of explanation |
| `eventType` | Maps to occasion type for prompt framing and soft category signal |
| `maxBudget` | Hard filter on `price` post-retrieval |
| `residenceCountry` | Hard filter on `country` post-retrieval |
| `relationship` | Soft signal in LLM ranking prompt |
| `gender` | Soft filter; passed to LLM; used as soft gate against `products.gender` |
| `birthday` (age derivable) | Soft filter against `min_age`/`max_age` |
| `interests` | Primary semantic signal; interests text is embedded to form the Interest Vector |

The cache key is a SHA-256 digest of all eight fields (sorted interests list).
Two requests with identical profiles share a cache entry by design. See decision
2026-07-01 (cache-key collision fix).

---

## 3. Matching & ranking approach (v1 — RAG pipeline)

The current pipeline is a retrieve-then-rank RAG:

```
RecipientProfile
  → embed(interests + eventType + relationship) → Interest Vector (1536-dim)
  → pgvector cosine similarity search, top-K candidates (K=20 currently)
  → hard filter: price <= maxBudget, country = residenceCountry, active = true
  → soft filter candidates: gender, age range (applied in LLM prompt, not SQL)
  → LLM reasoning prompt: top candidates + full profile context
  → LLM returns top 5 picks with per-gift personalised explanation
```

### Assumptions made (state explicitly)

1. The recipient's `interests` text is the primary discriminator. Two recipients
   with identical demographics but different interests should get meaningfully
   different results. This is the central bet of the semantic approach.
2. Product `name + description` embeddings are a reasonable proxy for gift
   suitability. This breaks down for products with minimal or generic descriptions
   (a known quality risk — see data.md).
3. The LLM ranking step adds genuine personalisation beyond what cosine similarity
   provides. This is an assumption not yet measured. It should be validated by
   comparing LLM-ranked results against straight cosine-sorted results using
   Precision@5 on a held-out sample.
4. Five results is the right output size. This is a UX decision (spec.md), not a
   quality decision. Track whether users feel the number is too few or too many
   when qualitative feedback is available.

### Justify before adding complexity

The current pipeline has no collaborative filtering, no user-to-user similarity,
no popularity signal, and no re-ranking model. These are intentional omissions, not
gaps. Adding them is only justified if:
- Precision@5 is measured below 0.6 on a statistically significant sample AND
- The proposed addition has a clear, measurable improvement hypothesis AND
- The added signal does not introduce privacy risks that require legal review.

Document any such decision in `docs/decisions.md` before building.

---

## 4. Cold-start handling

### Cold product (new product, no engagement history)

The current pipeline has no engagement signals. Every product — new or established —
is ranked purely on semantic relevance and metadata quality. Cold-product handling is
therefore identical to warm-product handling, and the pipeline already handles it.

The risk is not cold-start in the classical sense; it is cold-product metadata
quality. A product with a poor or absent description produces a low-quality embedding
and may appear in irrelevant result sets. Mitigation:
- Enforce description quality gates at ingestion (see data.md pre-filter rules).
- Products with `description = null` or description under 50 characters should receive
  lower LLM ranking weight. Pass a `metadata_quality: low` flag alongside such
  products in the LLM prompt so the model can de-prioritise them.
- Track null-explanation rate as a proxy: if the LLM cannot generate a specific
  explanation, the product is probably not well-matched.

### Cold user (new user, no history)

New users provide interests at registration (Step 4) or enter them in the AI Curator.
If interests are empty or skipped, the Interest Vector has low discriminating power
and retrieval will surface popular products across the budget/country constraint.

Mitigation for empty interests:
- Fall back to `eventType` + `relationship` as the primary retrieval signal.
  Embed a synthetic interest string such as `"gift for {relationship} {eventType}"`
  to produce a reasonable Interest Vector.
- Log the empty-interests rate. If it is above 30 % of curator sessions, the
  registration interests prompt needs a UX revision (flag to functional-analyst).

### Cold catalog (new category with few products)

If a canonical category has fewer than 20 active products, retrieval will surface
the same items repeatedly. Track active product count per category. Categories below
50 active products should be flagged to the catalog-data-engineer for sourcing action
before that category is promoted to recommendation surfaces.

---

## 5. Catalog source treatment in recommendations

### Affiliate products (Awin, Amazon PA-API)

Primary recommendation population. Have passed all ingestion quality gates.
Embedding quality is generally high because description is validated at ingest.
These products should be served without additional constraints beyond the standard
pipeline.

### User-contributed products (browser extension / URL clipper)

NOT YET ENABLED. The privacy-legal-advisor has flagged the browser extension feature
as not cleared to build in its current form (decision 2026-07-02). The analysis below
is recorded for when/if this feature proceeds in a reduced-risk form.

If user-contributed products enter the catalog, they must be treated differently:

**Engagement gate (hard requirement before serving in recommendations)**

A user-contributed product must not appear in any recommendation result until it has
received at least one save or wishlist-add from a user other than the contributor.
This gate:
- Prevents a contributor from gaming recommendations by clipping self-promotional
  products.
- Ensures a minimum cross-user relevance signal before the product is served to
  others.
- Is implementable today using `active = false` at ingest + a Firestore trigger or
  backend job that sets `active = true` once the cross-user save threshold is met.

**Quality gate (same as affiliate, enforced before activation)**

Every user-contributed product must pass the same pre-filter gates as affiliate
products: name, price, image_url, affiliate_url non-null; description >= 20
characters; category classifiable to a canonical value; image URL returns HTTP 200.
Products failing hard gates are discarded. Products failing soft gates land in
`active = false` for manual moderation.

**Recommendation slate diversity constraint**

When user-contributed products are serving, cap them at 2 of the 5 recommendation
results. Enforce this as a post-ranking filter before the LLM output is returned.
This prevents a cluster of user-contributed clips on a trending product from
dominating the slate.

**Source tracking**

The `provider_id` column already carries source provenance. Set `provider_id =
"user_clip"` for all user-contributed products. This enables per-source quality
analysis and the diversity constraint above.

**Bias risk**

User-contributed products reflect what users browse, not what recipients want.
They will over-represent Technology, Fashion, and Beauty and under-represent
Experiences and long-tail categories. Track per-category, per-source product counts
as a monitoring metric. If user-contributed products constitute more than 40 % of
any category's active products, halt ingestion for that category until affiliate
coverage catches up.

---

## 6. Category coverage requirements

Before the AI Curator is promoted to any marketing surface, each canonical category
must have a minimum active product count. These are requirements on the catalog,
not on the recommendation pipeline, but they gate recommendation quality:

| Category | Minimum active products for launch | Current status |
|---|---|---|
| Technology | 100 | Unknown — measure after Awin/Amazon ingestion |
| Home & Living | 100 | Unknown |
| Beauty & Wellness | 100 | Unknown |
| Fashion & Accessories | 100 | Unknown |
| Food & Drink | 50 | Unknown — fewer PT/ES affiliate sources |
| Sports & Outdoors | 50 | Unknown |
| Books & Media | 50 | Unknown |
| Toys & Games | 50 | Unknown |
| Experiences | 20 | Not yet sourced — no affiliate pathway defined |
| Other | No minimum | Catch-all; high counts here indicate taxonomy gaps |

The Experiences category has no affiliate ingestion pathway in v1. This is a
structural coverage gap. It must be resolved before birthday and anniversary
occasions can be well-served. Flag to catalog-data-engineer.

---

## 7. Known limitations & biases

### Semantic embedding limitations

- Products with sparse descriptions produce low-quality embeddings. Any product
  where the embedding was generated from name + category only (no description)
  should be flagged with a `embedding_quality: low` marker and de-weighted in the
  LLM prompt.
- The embedding model (text-embedding-3-small) was not trained on gifting-specific
  corpora. The relevance signal for unusual gift categories may be weaker than for
  common e-commerce terms.

### Catalog composition bias (affiliate sources)

- The current affiliate stack (Awin PT/ES merchant list + Amazon ES) biases toward
  Spanish-market mainstream retail. This is correct for the target market but limits
  diversity of unique or artisanal gifts.
- Technology and Fashion categories will be densest (most affiliate coverage) and
  will dominate cosine retrieval for any recipient profile with tech or fashion
  interests. Diversity constraints in the LLM prompt help but do not eliminate this.

### LLM ranking is a black box

- The LLM ranking step (Step 6 of the RAG pipeline) is not explainable in the same
  way as a scored retrieval model. We cannot audit why a specific product ranked
  above another for a specific profile. This is acceptable for v1 but becomes a
  problem if bias complaints arise. Plan to log the full LLM prompt and response for
  a sample of requests (with appropriate data minimisation — see privacy-legal-advisor
  on logging PII from the RecipientProfile).

### Gender and age soft filters are null-heavy

- `min_age`, `max_age`, and `gender` are nullable in the product schema and are
  not supplied by Awin or Amazon PA-API at ingestion time. The LLM prompt carries
  these signals from the RecipientProfile but cannot use them for structured
  filtering if the product data is absent. This weakens age and gender
  personalisation until the catalog-data-engineer derives these from category
  heuristics (planned for v2 per data.md).

### Cache sharing across users

- Two users gifting a recipient with identical profile fields (same name, budget,
  country, eventType, relationship, gender, age, interests) will receive the same
  5 results. This is intentional for cost reasons but means that cached results
  do not reflect catalog updates until the TTL expires. Monitor cache TTL vs.
  daily feed refresh cadence to ensure stale results are not served after a
  significant catalog update.

---

## 8. Instrumentation required (not yet built)

These are pre-conditions for measuring recommendation quality. None currently exist.

1. **Session logging**: log `recommendationSessionId`, `profileHash` (existing),
   `resultProductIds[]`, `timestamp` on every curator call. Store in a separate
   analytics table or log stream, not in PostgreSQL products.
2. **Relevance feedback**: after a curator session, prompt the user with a 3-tap
   rating per result (relevant / not relevant / skip). Store against `sessionId`.
3. **Click-through logging**: log when a user opens an affiliate URL from a
   recommendation card. Associate with `sessionId` and `productId`.
4. **Wishlist-from-recommendation logging**: flag whether a wishlist save originated
   from a recommendation result (pass a `source=curator` tag on the save event).

Without (1) and (2), Precision@5 cannot be measured. Without (1) and (3), CTR
cannot be measured. All four are needed before the first quality review cycle.

Coordinate with the backend-expert on the analytics logging contract and with the
privacy-legal-advisor on whether RecipientProfile fields may be retained in session
logs (they contain personal data about third parties — the recipient, not the user).
