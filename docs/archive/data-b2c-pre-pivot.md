# WiseGift — Catalog Data

## Schema & taxonomy

The canonical product record lives in PostgreSQL (`products` table). Fields and types are
defined authoritatively in `docs/engineering/architecture.md` Section 5. Summary of
matching-relevant columns:

| Field | Role in matching |
|---|---|
| `name` | Primary text signal; embedded into pgvector |
| `description` | Secondary text signal; embedded; truncated to 1000 chars |
| `category` | Hard filter and ranking signal |
| `price` / `currency` | Hard filter (`maxBudget`) |
| `country` | Hard filter (`residenceCountry`) |
| `image_url` | Required for card rendering; nil products are suppressed |
| `affiliate_url` | Required for monetisation; nil products are suppressed |
| `min_age` / `max_age` | Soft filter on recipient age range |
| `gender` | Soft filter on recipient gender preference |
| `delivery_days` | Soft filter on collection `max_delivery_days` |
| `active` | Hard gate — only `active = true` products are served |
| `embedding` (vector 1536) | Cosine-similarity retrieval in RAG pipeline |
| `provider_id` | Source provenance; used for feed refresh routing |
| `provider_product_id` | Deduplication key within a provider |
| `provider_categories` (text[]) | Raw source taxonomy preserved before AI normalisation |

### Occasion taxonomy (gifting context)

The `category` column uses a normalised WiseGift taxonomy independent of source
taxonomy. AI classifier maps provider categories to these values at ingestion time.

Canonical categories (v1):
- `Technology`
- `Home & Living`
- `Beauty & Wellness`
- `Fashion & Accessories`
- `Food & Drink`
- `Sports & Outdoors`
- `Books & Media`
- `Toys & Games`
- `Experiences`
- `Other`

Occasion types (used in `RecipientProfile.eventType` and collection filtering):
`Birthday`, `Christmas`, `Wedding`, `Anniversary`, `Baby Shower`, `Graduation`,
`Housewarming`, `Other`

---

## Attributes used for matching

The RAG pipeline (see architecture.md Section 4) embeds `name + description` into
a 1536-dim vector and uses cosine similarity against the recipient's interest vector.
Hard filters applied post-retrieval: `price <= maxBudget`, `country = residenceCountry`,
`active = true`. Soft signals used in LLM ranking prompt: `category`, `gender`,
`min_age`/`max_age`, `delivery_days`.

Fields that must be non-null for a product to enter the active catalog:
- `name`, `price`, `currency`, `category`, `image_url`, `affiliate_url`, `country`

Fields that improve match quality but are nullable:
- `description`, `min_age`, `max_age`, `gender`, `delivery_days`

---

## Merchant onboarding model

**Scope:** how new merchants land in the catalog. Cross-links:
[`giftability-rules.md`](./giftability-rules.md) (Layer A universal rules),
[`../decisions.md`](../decisions.md) (per-merchant Layer B lists and the
2026-07-10 strategic decision).

Prior context that is not re-litigated here:
[2026-07-02 sourcing plan](../decisions.md#2026-07-02--product-sourcing-strategy-awin-primary-amazon-pa-api-secondary-ebay-partner-network-tertiary),
[2026-07-02 Awin + Amazon pipeline design](../decisions.md#2026-07-02--ingestion-pipeline-design-awin-product-feeds-primary--amazon-pa-api-secondary),
[2026-07-02 image / health monitoring plan](../decisions.md#2026-07-02--catalog-sourcing-pipeline-amazon-pa-api-setup-tradedoubler-image-handling-health-monitoring-and-schema-migrations).

### One Awin ingestor, N merchant programmes

There is a **single** `AwinFeedIngestor` running against the Awin Product Data
API v3. It iterates the list of approved merchant programmes and pulls each in
sequence within the daily 02:00 UTC scheduled run. New merchants (e.g. El
Corte Inglés ES, Bikila) are added by dropping a row into the
`merchant_programmes` registry — no new code, no new adapter, no bespoke
per-merchant classes.

The same pattern applies to Tradedoubler (one `TradedoublerFeedIngestor`,
N feed URLs). Amazon PA-API is a single-merchant source and does not use this
model.

### `merchant_programmes` registry

v1 lands as a **PostgreSQL table** (not a config file). Rationale: (a) PO owns
the list day-to-day and editing YAML deployed via a build is friction we don't
need; (b) the health monitor and the purge job both read from it; (c) the
promotion threshold ("~10 merchants → move to DB") is trivial to cross once
ECI + Bikila + the existing PT/ES targets are all onboarded — better to start
in the right place than migrate mid-flight.

| Column | Type | Purpose |
|---|---|---|
| `merchant_id` | `VARCHAR(64) PRIMARY KEY` | Stable slug we control (e.g. `awin-eci-es`, `awin-bikila-es`). Written into `products.merchant_id`. |
| `network` | `VARCHAR(20) NOT NULL` | `awin` \| `tradedoubler` \| `amazon` \| `other`. |
| `network_programme_id` | `VARCHAR(64) NOT NULL` | Awin `merchantId`, Tradedoubler `programmeId`. Amazon PA-API is a single-merchant source and does not use `merchant_programmes` at all, so no nullable case exists. |
| `display_name` | `VARCHAR(200) NOT NULL` | Human-readable name for admin UI (e.g. `"El Corte Inglés ES"`). |
| `country` | `CHAR(2) NOT NULL` | ISO 3166-1 alpha-2 (`ES`, `PT`, …); one row per country per merchant if multi-country. |
| `poll_cadence` | `VARCHAR(20) NOT NULL DEFAULT 'DAILY'` | `DAILY` \| `WEEKLY` \| `PAUSED`. Weekly is for low-volatility catalogs; PAUSED disables ingest without deleting the row. |
| `layer_b_ref` | `VARCHAR(200)` | Pointer to the decision-log anchor holding the merchant's Layer B category allow/deny list. |
| `active` | `BOOLEAN NOT NULL DEFAULT true` | Kill switch. When `false`, the ingestor skips and the deactivation sweep marks all `products` for that merchant `active=false`. |
| `created_at` / `updated_at` | `TIMESTAMPTZ` | Audit. |

**Who edits it:** PO adds / retires rows via the admin UI (backend-expert to
scope the endpoint). catalog-data-engineer proposes edits when a Layer B
change or a poll-cadence tweak is warranted. Editing this table is a
first-class product action — not a code change.

Flagged to backend-expert: schema migration (V6 or later), admin endpoint
(`GET /admin/merchant-programmes`, `POST`, `PATCH /{merchantId}`), read-side
integration into `AwinFeedIngestor` and `TradedoublerFeedIngestor` scheduled
runs.

### `merchant_id` on `products`

Adds provenance beyond the existing `provider_id` (which only distinguishes
`awin` vs `amazon` vs `tradedoubler`).

| Column | Type | Purpose |
|---|---|---|
| `merchant_id` | `VARCHAR(64) NOT NULL` | FK-ish reference to `merchant_programmes.merchant_id`. Required on all new rows; back-fill script for legacy rows sets `'legacy-unknown'` until re-ingest. |

Uses:
- **Provenance** — which merchant does this row belong to.
- **Filtering** — recommendations can prefer/avoid a merchant (e.g. quality
  score falls below threshold → temporary exclusion) without deleting rows.
- **Deactivation scope** — when a merchant is paused or removed from
  `merchant_programmes`, we mark all their products `active=false` in one
  UPDATE.
- **Layer B enforcement** — the ingest-time filter looks up the merchant's
  Layer B list by `merchant_id`, not by network programme ID.

Flagged to backend-expert: V6 migration adding the column, back-fill,
NOT NULL constraint, index on `(merchant_id, active)`.

### Two-layer giftability filter

- **Layer A — universal.** Applies to every product from every merchant. Full
  ruleset in [`giftability-rules.md`](./giftability-rules.md). PO-approved
  editorial exclusions (adult content, weapons, alcohol, gambling, basic
  utensils, groceries, cleaning supplies, hardware/DIY consumables, tobacco,
  prescription/medical, live animals, hazardous chemicals, financial products,
  funeral supplies, third-party gift cards, digital-only intangibles) plus
  universal quality gates (required fields, price band, image reachable,
  description length ≥ 50, in-stock, supported currency/country).
- **Layer B — per-merchant.** Category allow/deny lists specific to each
  merchant's catalog structure. Because Awin merchants each expose their own
  taxonomy (ECI's `Menaje` vs Bikila's running-shoe categories), Layer B is
  drafted per merchant by catalog-data-engineer, approved by PO, and versioned
  in [`decisions.md`](../decisions.md) under that merchant's onboarding entry.
  `merchant_programmes.layer_b_ref` points to the anchor.

Both layers run at ingest, Layer A first then Layer B, first-match rejection.

### Ingest-time filtering, not query-time

Rejected rows **persist** with `active=false` and `rejection_reason=<rule id>`.
This costs some rows in the table but buys us: (a) a rejection audit —
per-rule, per-merchant volumes tell us if a rule is too tight or a merchant's
taxonomy has drifted; (b) we don't re-evaluate a rejected item on every
subsequent feed run — the dedup check on `(provider_id, provider_product_id)`
already recognises it and skips embedding.

Requires a new column:

| Column | Type | Purpose |
|---|---|---|
| `rejection_reason` | `VARCHAR(64)` | Populated only when `active=false` and the deactivation happened at ingest or health check. Values are the rule IDs from `giftability-rules.md` (e.g. `A-EX-03`, `A-Q-02`) plus health-monitor statuses (`LINK_BROKEN`, `IMAGE_BROKEN`, `UNAVAILABLE`, `MERCHANT_PAUSED`). NULL for `active=true` rows. |

Also add `deactivated_at TIMESTAMPTZ` (see purge section).

Flagged to backend-expert: V6/V7 migration adding both columns.

### 30-day purge

Overrides the earlier "keep forever for auditing" recommendation. PO chose
small DB > full audit trail.

**Rule:** any `products` row where `active = false` AND
`deactivated_at < now() - interval '30 days'` is **hard-deleted**. This
applies uniformly — whether the row was rejected at ingest, deactivated by the
health monitor, or paused by a merchant kill switch.

**Job:**
- `@Scheduled(cron = "0 30 3 * * *")` — daily at 03:30 UTC, between the Awin
  feed run (02:00) and the Amazon sweep (05:00), well outside the health
  monitor window (04:00–05:00).
- Batch DELETE with an upper bound per run (e.g. 10,000 rows) to keep the
  transaction small; if more remain, they clear on the next day.
- Must also DELETE from `collection_products` where a purged product is
  referenced (same pattern as `AmazonProductEnricher`).
- Logs summary counts per `rejection_reason` — first-class quality signal.

**`deactivated_at` semantics:** set (a) at ingest when a rule matches or a
quality gate fails; (b) by the health monitor when it flips `active` from
`true` to `false`; (c) by the merchant-pause sweep. Cleared when a row is
re-activated (rare — usually a manual admin action).

Flagged to backend-expert: `PurgeJob` `@Component` in
`infrastructure/scheduling/`, admin endpoint to inspect pending-purge counts.

### Cross-merchant dedup — Path C (post-hoc grouping)

Same product sold by ECI and by Bikila stays as two separate `products` rows
(one per merchant, each with its own `affiliate_url`). Clustering happens at
recommendation time, not at ingest.

**Fingerprint recipe (v1):**

```
fingerprint = sha256(lower(
    normalize(brand)
    + "|" + normalize(title_core)
    + "|" + normalize(key_attribute_set)
))
```

Where:
- `brand` — best available: EAN-derived brand > feed `brand` field >
  regex-extracted first token of `name` (fallback, noisy).
- `title_core` — title with marketing noise stripped: remove common
  punctuation, remove size/colour tokens (moved into `key_attribute_set`),
  collapse whitespace, lowercase, strip stop words (`the`, `for`, `de`, `la`,
  `el`, `para` etc.), truncate to first 12 tokens.
- `key_attribute_set` — deterministic serialisation of the small set of
  discriminating attributes that ARE identity, not variation: for now,
  `model_number` (if present) and `EAN`/`GTIN` (if present). Explicitly
  excluded from the fingerprint in v1: `size`, `colour`, `capacity` — see
  size-critical filter deferral (2026-07-10 decision).

Stored as `products.fingerprint VARCHAR(64)` (SHA-256 hex). Nullable — if we
can't compute a meaningful fingerprint (missing brand AND missing model AND
generic title), we skip clustering for that row and it stays a singleton in
recommendations.

**Cluster at recommendation time:** the ranking layer groups the retrieved
set by `fingerprint`, picks one representative per cluster (highest
merchant-quality or lowest price — recommendations-specialist owns the
tiebreak), and returns clusters, not raw rows. See
[`recommendations.md`](./recommendations.md).

**v1 caveat:** the fingerprint is noisy. EAN coverage on Awin feeds is
inconsistent; some merchants supply it, most don't. Where EAN is present the
fingerprint is high-confidence; where it isn't, the brand+title normalisation
does the work and false positives (two different products clustered together)
and false negatives (same product not clustered) both happen. Monitor and
tune.

**Path A migration (future):** once EAN coverage on active merchants exceeds
~60% AND we have >10 merchants selling overlapping SKUs, migrate to a
canonical `product_catalog` table with per-merchant `product_offers` rows.
The v1 fingerprint seeds this — canonical rows are created by clustering on
today's fingerprint and choosing a representative. Explicitly not scheduled;
tracked as a future decision.

Flagged to recommendations-specialist: fingerprint recipe review, cluster
representative tiebreak, slate diversity constraint (one row per fingerprint
in the final slate).

### Refresh cadence + `last_seen_at` integration

Per-merchant `poll_cadence` on `merchant_programmes` drives which merchants
are hit on which run:

- `DAILY` — pulled every 02:00 UTC run.
- `WEEKLY` — pulled Mondays only; skipped other days. Use for merchants with
  low SKU churn (e.g. curated boutique catalogs where the feed rarely changes).
- `PAUSED` — skipped entirely; deactivation sweep marks all their products
  `active=false, rejection_reason=MERCHANT_PAUSED`.

Interaction with V5 health columns (`last_seen_at`, `last_checked_at`,
`last_check_status`):

- `last_seen_at` is set on every ingest touch. For `WEEKLY` merchants the
  staleness threshold is tightened to `> 16 days` (2 missed weekly runs)
  rather than the default 48 h.
- `last_checked_at` and `last_check_status` are managed by the health
  monitor and are independent of poll cadence.
- The 30-day purge is driven by `deactivated_at`, NOT `last_seen_at` — a
  product deactivated for staleness gets `deactivated_at = now()` at the
  moment of deactivation, and its 30-day clock starts then.

### Size-critical filter (deferred)

Not enforced in v1. A "size / fit-critical" product (running shoes, apparel)
currently ships as a single row even though the recipient's size is unknown
at recommendation time. Revisit when slate quality issues surface — the
trigger is "> 5% of surfaced products in a category are size-critical and
the merchant does not offer free/easy returns."

### Follow-on schema summary (flagged to backend-expert)

New / changed columns needed to support this model (target migrations V6+):

| Table | Column | Notes |
|---|---|---|
| `merchant_programmes` (new) | see table above | New DB table. |
| `products` | `merchant_id VARCHAR(64) NOT NULL` | FK-ish to `merchant_programmes.merchant_id`. Index on `(merchant_id, active)`. |
| `products` | `rejection_reason VARCHAR(64)` | Nullable; set on any deactivation. |
| `products` | `deactivated_at TIMESTAMPTZ` | Drives the 30-day purge clock. |
| `products` | `fingerprint VARCHAR(64)` | SHA-256 hex; nullable. Index (non-unique) on `fingerprint`. |
| Purge job | — | New scheduled `@Component`. |

---

## Ingestion & enrichment

### Sourcing stack (v1 — see decision 2026-07-02)

| Priority | Source | Type | PT/ES coverage | Access gate |
|---|---|---|---|---|
| 1 — Primary | Awin | Product Data API v3 (JSON, paginated) | Strong (dedicated PT/ES programmes) | Publisher account active |
| 2 — Secondary | Amazon PA-API 5.0 | REST API (SearchItems + GetItems) | Good (amazon.es; PT routed via amazon.es) | Associates account active; sales-velocity ramp |
| 3 — Tertiary | eBay Partner Network (EPN) | REST API (Browse API) | Moderate (ebay.es, cross-border) | Explicitly out of scope for now |

### Module placement

Ingestion lives **inside the existing `catalog` Spring module** — no new module or CLI. The
`AffiliateClient` port, `ProductFeeder` orchestrator, `ProductEmbeddingClassifierImpl`,
and JPA persistence layer are already in place. New adapters slot in under
`infrastructure/out/affiliate/awin/` and `infrastructure/out/affiliate/amazon/`.

The existing `AwinApiClient` (keyword-search mode) becomes the foundation for a feed-oriented
ingestor. The existing `AmazonCuratedClient` (static YAML stub) is superseded by a live PA-API
adapter and should be removed or kept only under `affiliate.amazon.mock.enabled`.

### Awin ingestion design

**API choice: Product Data API v3 (JSON) over flat-file feeds**

Flat-file feeds (SFTP/FTP, gzipped XML or CSV delivered per-merchant) are the right
choice above ~200k products/day or ~50 approved merchant programmes. At this stage
(target 5k products, handful of PT/ES programmes) the Product Data API avoids SFTP
infrastructure, credential rotation, and per-merchant feed URL bookkeeping. Revisit
when merchant programme count exceeds ~30 and daily product volume exceeds 100k.

Flat-file tradeoffs to keep in mind:
- Pro: lower latency per record, no API rate concern, richer field set (stock level, EAN).
- Con: requires an SFTP client + cron job, per-merchant feed URL management, XML parsing
  pipeline, and a separate process to decompress/stream large gzipped files.

**Merchant programme priority list (PT/ES, gift verticals)**

Prioritise programmes where (a) the merchant has PT or ES localisation, (b) the feed
has description quality above 50 characters, and (c) the category aligns with the five
target gift verticals. Indicative targets — confirm programme IDs in the Awin publisher
UI after approval:

| Vertical | Candidate merchants (PT/ES) |
|---|---|
| Beauty & Wellness | Sephora ES, Douglas ES, Rituals ES, Kiehl's ES |
| Home & Living | El Corte Inglés ES, Casa ES, Zara Home ES, Leroy Merlin ES |
| Fashion & Accessories | Zara ES, Mango ES, ASOS ES, El Corte Inglés ES |
| Technology | MediaMarkt ES, El Corte Inglés ES, PcComponentes ES |
| Food & Drink | Lavinia ES, Palacio de la Cerveza, Chocolates Valor ES |

Note: most large merchants offer dual PT/ES coverage under one programme. Confirm
country availability in the programme detail page before approving.

**Ingestion flow**

```
[Awin Product Data API v3]
  GET /publishers/{publisherId}/products
    ?countryCode=ES|PT
    &merchantId={merchantId}
    &pageSize=100
    &page={n}
    → paginate until empty page

  Per page:
    → pre-filter (see gates below)
    → normalise AwinProduct → Product domain object
    → classify: ProductEmbeddingClassifierImpl maps provider_categories[] → canonical category
    → deduplicate: existsByProviderIdAndProviderProductId()
        INSERT new records | UPDATE price/imageUrl on existing records
    → validate image URL (HTTP HEAD, expect 200)
    → embed: name + ". Category: " + category + ". " + description
    → set active = true; save
```

**Pre-filter rules (Awin)**

Reject at ingest time (do not embed, do not persist):
- `searchPrice` is null, blank, zero, or unparseable.
- Parsed price < 5.00 EUR or > 2000.00 EUR (outside plausible gifting range).
- `awImageUrl` is null or blank.
- `merchantDeepLink` is null or blank.
- `productName` is null or shorter than 5 characters.
- `description` is null or shorter than 50 characters (A-Q-04; PO-approved threshold as of 2026-07-10).
- `categoryName` not in the gift-relevant allow-list (see taxonomy section).
- Product is marked out-of-stock where the feed provides a stock status field.

**Feed update strategy**

Full feed per merchant daily (not delta). Awin's Product Data API does not expose a
reliable delta/changed-since mechanism on the v3 endpoint. The deduplication check
(`existsByProviderIdAndProviderProductId`) handles re-runs without duplicate inserts.
Price and image URL are always overwritten on the existing record on re-sync.

Products that disappear from a merchant feed across two consecutive daily runs should
be soft-deactivated (`active = false`). Implement this by tracking `lastSeenAt`
timestamp on the `Product` entity (requires a schema migration — product owner decision
on priority).

**PostgreSQL write strategy**

- Use Spring Data JPA `saveAll()` in batches of 50 within a `@Transactional` method.
- Deduplication check is `existsByProviderIdAndProviderProductId` before each record —
  for large feeds replace with a bulk `findAllByProviderIdAndProviderProductIdIn` to
  reduce N+1 queries.
- Document ID: PostgreSQL auto-increment `BIGINT` (existing schema). Composite unique
  index on `(provider_id, provider_product_id)` must be present — verify with Flyway.
- Embedding is stored inline in the `products` table in the `vector(1536)` column.

**Scheduling**

- Daily at 02:00 UTC: `@Scheduled(cron = "0 0 2 * * *")` inside `catalog` process.
- Each merchant programme is processed sequentially within the run to avoid hitting
  Awin API rate limits (Product Data API: 10 req/s by default).

### Amazon PA-API ingestion design

**Operations to use**

- `SearchItems`: discover ASINs by keyword + browse node within amazon.es.
- `GetItems`: fetch full item detail (title, features, images, offers) by ASIN batch
  (up to 10 ASINs per call).

Do not use `GetVariations` or `GetBrowseNodes` in v1 — adds complexity without
proportionate catalog coverage gain.

**Keyword / browse node search strategy (ES locale)**

The existing `ProductFeeder.keywords` list is generic. Replace it with a structured
matrix mapping each WiseGift canonical category to 3–5 Spanish-language search terms
and the corresponding Amazon browse node ID for amazon.es:

| WiseGift category | Search terms | amazon.es Browse Node |
|---|---|---|
| Technology | "auriculares inalámbricos", "smartwatch", "altavoz bluetooth", "cámara digital" | 667049031 (Electrónica) |
| Home & Living | "decoración hogar", "velas aromáticas", "set cocina regalo", "difusor aromas" | 599369031 (Hogar) |
| Beauty & Wellness | "set cuidado piel regalo", "perfume mujer", "kit maquillaje" | 117332031 (Belleza) |
| Fashion & Accessories | "bolso mujer regalo", "pañuelo seda", "cartera hombre piel" | 1571280031 (Moda) |
| Food & Drink | "vino tinto regalo", "cesta navidad gourmet", "chocolates artesanos" | 4666196031 (Alimentación) |
| Sports & Outdoors | "esterilla yoga", "mochila senderismo", "kit fitness regalo" | 2454219031 (Deportes) |
| Books & Media | "novela regalo", "libro cocina español", "libro ilustrado" | 599364031 (Libros) |
| Toys & Games | "juego de mesa familia", "juguete educativo niños", "lego regalo" | 1626220031 (Juguetes) |

Browse node IDs must be verified against the amazon.es node tree — treat the above as
starting points.

**Handling missing `description`**

Amazon's PA-API `SearchItems` / `GetItems` does not return a `description` field.
Synthesise it from the `Features` resource (`ItemInfo.Features.DisplayValues[]`):

```
description = String.join(". ", features).substring(0, min(1000, length))
```

If `features[]` is null or empty and no description can be synthesised, set
`description = null` and flag the product for review (it will still pass the required-
field gate since `description` is nullable, but embedding quality will be lower —
name + category only). Products with null description should be weighted lower in
LLM ranking; flag this to the recommendations-specialist.

**Rate limit management**

New PA-API accounts start at 1 request/second. Use a `ScheduledExecutorService`-backed
token bucket or a simple `Thread.sleep(1100)` between API calls. Track the following:

- One `GetItems` call per 10 ASINs — batch ASINs from `SearchItems` results.
- `SearchItems` returns up to 10 results per call; paginate with `ItemPage` (max 10 pages
  = 100 results per keyword).
- At 1 req/s: 100 keywords × 10 pages (SearchItems) + 100 keywords × 10 batches
  (GetItems) = 2,000 calls = ~33 minutes. This is within the daily window.
- Rate limit ramps to 10 req/s after qualifying sales. Do not assume this during v1 build.

**Pre-filter rules (Amazon)**

Same gates as Awin, plus:
- Reject ASINs where `Offers.Listings` is empty (no current offer = not buyable).
- Reject ASINs where `PrimaryImage` is null (required for card rendering).
- Reject if synthesised description (from features) is shorter than 50 characters after
  joining (A-Q-04 threshold, aligned with Awin path).

**PostgreSQL write strategy**

Identical to Awin: `saveAll()` in batches of 50, dedup by `(provider_id="amazon",
provider_product_id=asin)`. `affiliateUrl` format:
`https://www.amazon.es/dp/{asin}?tag={storeId}` (existing `AmazonCuratedClient`
pattern is correct).

**Scheduling**

- Daily at 03:00 UTC: `@Scheduled(cron = "0 0 3 * * *")`, offset from Awin to avoid
  simultaneous database pressure. The rate-limited PA-API sweep will run for ~33 minutes
  at 1 req/s.

### Shared normalisation layer

**WiseGift canonical category taxonomy (v1)**

The categories in `ProductCategoryKeywords.java` are misaligned with the canonical
taxonomy in `data.md`. This must be reconciled before live ingestion; it is the highest-
priority pre-condition. Canonical set (these are the `KEYWORDS` map keys that must exist):

| Canonical value (stored in `products.category`) | Replaces in current code |
|---|---|
| `Technology` | `Electronics` |
| `Home & Living` | `Home & Kitchen` |
| `Beauty & Wellness` | `Beauty & Personal Care` |
| `Fashion & Accessories` | `Fashion` |
| `Food & Drink` | `Food & Drink` (matches) |
| `Sports & Outdoors` | `Sports`, `Travel & Outdoors` (merge) |
| `Books & Media` | `Books & Media` (matches) |
| `Toys & Games` | `Toys & Kids` |
| `Experiences` | not present — add |
| `Other` | `Uncategorized` (rename) |

Remove `Digital Services`, `Finance & Investment`, `Automotive` from `ProductCategoryKeywords`
— these are not gift categories and will pollute the classifier. The classifier will
fall back to `Other` for products that genuinely do not fit.

**Awin → WiseGift category mapping**

`AwinProduct.categoryName` is a free-text merchant-supplied string. The embedding
classifier (`ProductEmbeddingClassifierImpl`) already handles the mapping — no explicit
lookup table is required at the adapter level. Pass `categoryName` as the sole element
of `providerCategories[]`; the classifier will cosine-match it against category keyword
embeddings. Retain the raw value in `provider_categories` for auditability.

**Amazon → WiseGift category mapping**

`SearchItems` returns `BrowseNodeInfo.BrowseNodes[].DisplayName`. Pass the first-level
display name as `providerCategories[0]`. The embedding classifier handles the rest.
For `GetItems`, use `Classifications.ProductGroup` + `Classifications.Binding` as
additional signal in `providerCategories[]`.

**Field mapping table**

| WiseGift `Product` field | Awin source field | Amazon PA-API source field |
|---|---|---|
| `providerId` | `"awin"` (constant) | `"amazon"` (constant) |
| `providerProductId` | `aw_product_id` | ASIN |
| `providerCategories` | `[category_name]` | `[BrowseNodes[0].DisplayName, ProductGroup]` |
| `name` | `product_name` (truncate 200) | `ItemInfo.Title.DisplayValue` (truncate 200) |
| `description` | `description` (truncate 1000) | join(`ItemInfo.Features.DisplayValues`, ". ") truncate 1000 |
| `price` | `search_price` (parse BigDecimal) | `Offers.Listings[0].Price.Amount` |
| `currency` | `currency` | `Offers.Listings[0].Price.Currency` |
| `imageUrl` | `aw_image_url` | `Images.Primary.Large.URL` |
| `affiliateUrl` | `merchant_deep_link` | `https://www.amazon.es/dp/{asin}?tag={storeId}` |
| `country` | `country` field or config default | `"ES"` (amazon.es locale; PT products served from ES) |
| `deliveryDays` | `delivery_time` (parse with `AwinApiClient.parseDeliveryDays`) | null (PA-API does not reliably expose this) |
| `category` | classifier output | classifier output |
| `minAge` / `maxAge` | null (Awin does not supply) | null (derive from category heuristic in v2) |
| `gender` | null | null (derive from keyword signal in v2) |

**Products that fail validation gates**

- Hard reject (missing required field, price out of range, image null): discard silently;
  log at DEBUG with provider ID and product ID for auditability.
- Soft flag (description too short, price suspicious): persist with `active = false`;
  these are available for manual review and reactivation via
  `PATCH /api/v1/admin/products/{id}`.
- Image URL returns non-200: persist with `active = false`; re-check on next daily run.

### Ingestion pipeline (target design — full sequence)

1. **Fetch** — `@Scheduled` cron in `catalog` calls `AwinFeedIngestor` (02:00 UTC) and
   `AmazonPaApiClient` (03:00 UTC).
2. **Pre-filter** — reject records that fail hard gates before any DB or AI operation.
3. **Normalise** — map source fields to `Product` domain object per field mapping table above.
4. **Classify** — `ProductEmbeddingClassifierImpl.classifyProduct()` maps
   `providerCategories[]` + name + description to canonical category. Embeddings for
   category keyword vectors are initialised once at startup.
5. **Deduplicate** — `existsByProviderIdAndProviderProductId()` check; INSERT new /
   UPDATE price+image on existing.
6. **Image validate** — HTTP HEAD on `imageUrl`; set `active = false` if non-200.
7. **Embed** — `name + ". Category: " + category + ". " + description` sent to
   `text-embedding-3-small`; result stored in `embedding vector(1536)`.
8. **Activate** — set `active = true` when all required fields present and image valid.

### Feed refresh cadence

- Price and availability: daily minimum (prices shift frequently on affiliate feeds).
- New products: daily (same run as price refresh).
- Full re-embed: only when the embedding model changes — this requires a migration job
  that iterates all active products and re-embeds; plan before upgrading the model.

---

## Tradedoubler ingestion design

Tradedoubler is needed specifically for Fnac ES and Decathlon ES. It is not a live API
like the Awin Product Data API — it delivers gzipped XML or CSV files per programme.

### Feed format

Tradedoubler uses its own XML schema ("Datafeed"). A programme feed URL looks like:
`https://feeds.tradedoubler.com/ES/{programmeId}/product-feed.xml.gz`

Key XML fields (mapped to WiseGift Product):

| WiseGift field | Tradedoubler XML element |
|---|---|
| `providerProductId` | `<Id>` or `<EAN>` (use `<Id>`) |
| `name` | `<Name>` |
| `description` | `<Description>` |
| `price` | `<Price>` (decimal, EUR) |
| `currency` | `<Currency>` or hardcode `EUR` for ES programmes |
| `imageUrl` | `<ImageUrl>` |
| `affiliateUrl` | `<TrackingUrl>` (Tradedoubler's affiliate-tracked link) |
| `providerCategories` | `<CategoryName>` |
| `country` | hardcode `ES` for Fnac/Decathlon ES programmes |
| `deliveryDays` | `<DeliveryTime>` (when present; parse same as AwinApiClient.parseDeliveryDays) |

The CSV variant uses column headers matching the same concepts; XML is preferred because
the structure is unambiguous with namespacing.

### Difference from Awin feeds

| Dimension | Awin (API mode) | Tradedoubler (flat-file mode) |
|---|---|---|
| Delivery | REST API, JSON, paginated | HTTPS download, gzipped XML/CSV |
| Rate concern | 10 req/s API limit | None — one file download per programme per run |
| Field set | Rich (delivery_time, country, merchant fields) | Similar; field names differ |
| Delta support | None (full re-fetch) | None (full re-fetch) |
| Infrastructure | None beyond RestClient | Requires gzip stream parsing (GZIPInputStream + StAX or JAXB) |
| Auth | Bearer token in Authorization header | Credentials embedded in feed URL (or Basic auth on some programmes) |

The `TradedoublerFeedIngestor` adapter is a separate class from `AwinApiClient`. It does
NOT implement `AffiliateClient` (which is keyword-search-oriented); instead it follows the
feed-pull pattern and is called directly by the scheduled orchestrator.

### Publisher onboarding steps (product owner action required)

1. Register at tradedoubler.com — choose "Publisher" account type.
2. Submit site/app details for review (usually 1-3 business days).
3. Search the Programmes directory for "Fnac" (ES) and "Decathlon" (ES); apply to each.
4. After approval, find the "Product feeds" or "Datafeed" section in the programme detail.
5. Copy the feed download URL for each programme; store in application config as
   `affiliate.tradedoubler.feeds[0].url` etc.
6. Tradedoubler provides a Publisher ID and API key — these may be embedded in feed URLs
   or used for the Management API. For flat-file ingestion only the feed URL is needed.

### Scheduling

Daily at 02:30 UTC (30 minutes after Awin to avoid simultaneous DB pressure):
`@Scheduled(cron = "0 30 2 * * *")`.

---

## Health monitoring pipeline

### New `products` table columns (V5 Flyway migration)

```sql
-- V5__add_catalog_health_columns.sql
ALTER TABLE products
    ADD COLUMN IF NOT EXISTS last_seen_at     TIMESTAMPTZ,
    ADD COLUMN IF NOT EXISTS last_checked_at  TIMESTAMPTZ,
    ADD COLUMN IF NOT EXISTS last_check_status VARCHAR(20) NOT NULL DEFAULT 'UNCHECKED';
```

`last_seen_at`: set by feed ingestors (Awin, Tradedoubler) each time the product
appears in a feed run. Not set by Amazon PA-API (health check covers that).
`last_checked_at`: set by `CatalogHealthMonitor` after each check cycle.
`last_check_status`: enum values `OK | LINK_BROKEN | IMAGE_BROKEN | UNAVAILABLE | UNCHECKED`.

Add these fields to the `Product` entity:

```java
private Instant lastSeenAt;
private Instant lastCheckedAt;
private String lastCheckStatus = "UNCHECKED";
```

This migration must be applied before the health monitor or Tradedoubler adapter are
deployed. Flag to backend-expert for the Flyway migration file.

Also add a composite unique index (missing from V1 schema — required for deduplication
correctness):

```sql
-- include in V5 or as a separate V5b migration
CREATE UNIQUE INDEX IF NOT EXISTS ux_products_provider
    ON products(provider_id, provider_product_id);
```

### `CatalogHealthMonitor` design

Location: `infrastructure/out/affiliate/health/CatalogHealthMonitor.java`
Annotation: `@Component`, `@RequiredArgsConstructor`, `@Slf4j`

```java
@Scheduled(cron = "0 0 4 * * *")   // 04:00 UTC — link + image check
public void checkLinksAndImages() { ... }

@Scheduled(cron = "0 0 5 * * *")   // 05:00 UTC — Amazon availability via PA-API
public void refreshAmazonAvailability() { ... }
```

**Link check logic (`checkLinksAndImages`)**

Iterate all active products in batches of 100 using `ProductJpaRepository.findAll(Pageable)`.
For each product:

```
1. HTTP HEAD on affiliateUrl (5s timeout, no redirects beyond 5 hops)
   - 200/301/302 → OK
   - 404/410 → LINK_BROKEN → set active=false, lastCheckStatus="LINK_BROKEN"
   - Redirect loop (> 5 hops) → LINK_BROKEN
   - Timeout / connection refused → leave active unchanged; log WARN; skip

2. HTTP HEAD on imageUrl (5s timeout)
   - 200 → OK
   - 404/403 → IMAGE_BROKEN → set active=false, lastCheckStatus="IMAGE_BROKEN"
   - Content-Type must start with "image/" — if it doesn't, treat as IMAGE_BROKEN

3. If both pass → set lastCheckStatus="OK", lastCheckedAt=now()
   Persist only if status changed (avoid unnecessary UPDATE churn)
```

Use `java.net.http.HttpClient` with `HttpClient.Redirect.NORMAL` (follows up to 5 redirects).
Do not use `RestClient` for HEAD requests — it throws on 4xx by default. Use the raw
`HttpClient` with `HttpResponse.BodyHandlers.discarding()` and inspect `statusCode()`.

**Amazon availability refresh (`refreshAmazonAvailability`)**

Batch Amazon products in groups of 10 (PA-API GetItems limit). Per batch:

```
GetItems(ItemIds=[asin1..asin10],
         Resources=["Offers.Listings.Availability.Message",
                    "Offers.Listings.Price.Amount",
                    "Offers.Listings.Price.Currency"])
```

Response handling:
- `ItemsResult.Items[].Offers.Listings` empty → `active=false`, `lastCheckStatus="UNAVAILABLE"`
- Price returned → update `product.price` with new value; `lastCheckStatus="OK"`
- Honour 1 req/s rate limit: `Thread.sleep(1100)` between batch calls.

**Deactivation and collection cleanup**

When a product is set `active=false` by the health monitor, also delete it from
`collection_products` — the same pattern already used in `AmazonProductEnricher`:

```java
collectionProductRepo.deleteByProductId(product.getId());
```

**`last_seen_at` staleness deactivation**

At the end of each Awin + Tradedoubler feed run, the feed ingestor calls:

```java
// Deactivate products not seen in the last 2 consecutive runs (48 h)
productRepository.deactivateNotSeenSince(Instant.now().minus(48, HOURS), "awin");
productRepository.deactivateNotSeenSince(Instant.now().minus(48, HOURS), "tradedoubler");
```

This requires a new method on `ProductRepository` port:

```java
void deactivateNotSeenSince(Instant threshold, String providerId);
```

Implemented in `ProductRepositoryImpl` via a JPA `@Modifying @Query`:

```java
@Modifying
@Query("UPDATE Product p SET p.active = false WHERE p.providerId = :providerId " +
       "AND p.lastSeenAt < :threshold AND p.active = true")
void deactivateNotSeenSince(@Param("threshold") Instant threshold,
                             @Param("providerId") String providerId);
```

### Run cadence summary

| Check | Schedule | Method | What triggers deactivation |
|---|---|---|---|
| Affiliate link validity | 04:00 UTC daily | HEAD request | 404 / 410 / redirect loop |
| Image URL validity | 04:00 UTC daily (same run) | HEAD request | 404 / non-image content-type |
| Amazon price + availability | 05:00 UTC daily | PA-API GetItems | Empty Listings array |
| Awin/TD price refresh | 02:00 / 02:30 UTC daily | Feed re-ingest | Not applicable (price updated) |
| Feed staleness | End of each Awin/TD run | `last_seen_at` comparison | Not seen in > 48 h |

---

## Image handling verdict

**Recommendation: direct merchant URLs with HEAD validation. No CDN re-hosting at this stage.**

Rationale:

1. Legal risk of re-hosting: affiliate programme terms (Awin, Amazon Associates,
   Tradedoubler) permit displaying merchant images only in the context of promoting
   their products. Copying images to your own CDN (Cloudinary, Firebase Storage) crosses
   into re-hosting territory and requires explicit written licence from each merchant.
   At 5k products across many merchants, obtaining that licence is not feasible. The
   legal risk is asymmetric: a single merchant complaint can result in programme termination.

2. Cost and complexity: Cloudinary free tier covers ~25k transformations/month; beyond
   that it becomes a recurring cost. Firebase Storage is cheap per GB but adds a second
   ingestion step (download + upload), a storage management layer, and a URL lifecycle to
   manage. None of this is warranted for 5k products.

3. Merchant CDN reliability: Amazon images are served from `m.media-amazon.com` (very
   reliable). Awin serves images via merchant CDNs — quality varies, which is exactly
   what the HEAD validation at ingestion time and the nightly health monitor address.
   The correct response to a broken image URL is to deactivate the product and wait for
   the next feed run to supply a valid URL, not to cache a stale copy.

4. What validation buys you: the ingestion-time HEAD check ensures no product enters the
   active catalog with a broken image. The nightly health check catches post-ingestion
   breakage. This covers the material risk without the CDN overhead.

Revisit threshold: if the rolling 30-day image-broken rate among active products exceeds
5%, evaluate a selective image proxy (cache only the most-surfaced products in
recommendations, not the full catalog).

-
## Search / indexing

- pgvector cosine-similarity index on `embedding` (IVFFlat or HNSW depending on catalog
  size; HNSW preferred above ~50k products for recall stability).
- B-tree indexes on `country`, `category`, `price`, `active` for hard-filter performance.
- Text search (keyword) uses PostgreSQL `tsvector` on `name + description`; consider
  upgrading to a dedicated search index (Typesense/Meilisearch) if Discover tab latency
  degrades above 200 ms p95.

---

## Data quality & freshness

### Quality gates (enforced at ingestion)


- Reject products missing `name`, `price`, `image_url`, or `affiliate_url`.
- Reject products where `image_url` returns non-200 at ingestion time.
- Reject products with `description` shorter than 50 characters (A-Q-04 —
  insufficient signal for embedding-based retrieval / LLM summarization).
- Flag products with price = 0 or price > 10000 for review.

### Known quality risks by source

| Source | Risk |
|---|---|
| Awin | Feed quality varies widely by merchant; some feeds have truncated descriptions or missing images. Must filter by merchant data quality score. |
| Amazon PA-API | Strict rate limits (1 req/s on new accounts, up to 10/s after qualifying). Description field is often absent; must fall back to `features[]` array. Sales-velocity gate delays access for new accounts. |
| eBay Partner Network | Listings are user-generated; title quality is inconsistent. Category mapping requires aggressive normalisation. Listings can go stale/sold without feed update. |

### Freshness vs. cost tradeoff

Daily feed refresh is the minimum viable cadence. Real-time availability checking
is not economical at early stage — use the `GET /api/v1/products/availability` pattern
(Flutter-side pre-render check) as a soft freshness layer without re-ingesting the
whole catalog on every request.

### Scale warning

At ~100k products the IVFFlat index recall degrades and must be replaced with HNSW.
Awin alone can deliver millions of product records across all merchants — the feed
must be filtered by relevance score and merchant quality before storage, not ingested
in full. Plan a relevance pre-filter (price range plausibility, gift-category relevance
score) at the Normalise step before volume becomes a problem.
