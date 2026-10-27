# 200-product fashion catalogue — design + generation spec

> **Status: DRAFT for PO validation.** Owner: `recommendations-specialist`.
> Not committed by this draft; the coordinator handles git. Nothing here is
> settled until the PO signs off; this draft does not edit `decisions.md` or
> `CLAUDE.md`.
>
> **Purpose.** Replace the 70-product "Atelier Nord" fixture
> (`fashion-catalogue.csv`) with a **200-product adult fashion catalogue
> (100 men + 100 women)** built to *demonstrate the recommender is powerful*:
> coherent, impressive PDP similars; clean gender / age / style / category
> filters; and ≥ 3 style-tiles that separate visibly. This doc is a **design +
> precise generation spec** that feeds a separate content step which emits the
> Shopify-import CSV. **It does not generate the 200 rows** — it locks the
> structure, distribution, tag vocabulary, showcase neighbourhoods, eval plan,
> and 4 exemplar rows so the content step is deterministic.
>
> **Pipeline-aligned (read before generating).** The structure below is derived
> from the live backend so the import **auto-classifies with 0 unknowns**:
> - `GenderClassifier` (`catalog/domain/model/GenderClassifier.java`): gender is
>   derived from **tags first, then `product_type`** — never the title. Synonym
>   sets: men ← `men/man/mens/male/boys/boy/menswear/him`; women ←
>   `women/woman/womens/ladies/lady/female/girls/girl/womenswear/her`; unisex ←
>   `unisex/all-gender/all-genders/genderless`. Both men+women signals → unisex.
> - `AgeGroupClassifier` (`catalog/domain/model/AgeGroupClassifier.java`): age is
>   derived tags→type, **baby > kids > adult**, else falls back to **adult when
>   gender is confidently men/women**. ADULT set = `adult/mens/womens/men/women`.
>   **CRITICAL TRAP:** `boys`/`girls` are in BOTH the gender men/women set AND the
>   KIDS set. A `boys` tag would classify the product as **KIDS** and the audience
>   filter would hide it from an adult self/gift flow. **This catalogue NEVER uses
>   `boys`/`girls`/`kids`/`teen`/`junior`/`youth`/`child`.** Gender cue is always
>   the plain `men` / `women` token, which also lands age = adult.
> - Embedding input (`EmbeddingService.composeEmbedInput`, **BINDING format**):
>   `title + "\n" + description + "\n" + product_type + "\n" + sorted(tags)`. Tags
>   feed the vector, so the tag schema below is a first-class retrieval signal,
>   not just a filter. Rich, varied, non-templated **title + description** are
>   what make similars impressive; templated copy collapses the vectors.
> - PDP similars (`similars-gender-category-ranking.md`): **gender is a hard
>   filter**, **category (`product_type`) is a soft/emergent signal** with a
>   diversity cap (≤ 2 per `product_type`). Cross-category same-gender recs are
>   *wanted* (trousers for a shirt), so categories must be distinct and populated.
> - Styles (`style-anchored-recommendations.md`): a style = **2–3 anchor
>   products** (`StyleAnchoring.MIN_ANCHORS_TO_ENABLE = 2`, `MAX_ANCHORS = 3`);
>   matching = **centroid (mean-pool) of the anchors' stored text-embeddings**.
>   Style quality gates: anchor-coherence (mean pairwise cosine ≥ 0.45 clean),
>   catalogue-fit (best match ≥ 0.60), separation (pairwise top-10 Jaccard < 0.5).
> - CSV shape + variant grouping follows the existing `fashion-catalogue.csv`
>   (25 Shopify product-import columns; first row per handle carries all product
>   fields, subsequent rows carry only handle + variant/option/pricing).
>
> **Vendor / demo brand:** reuse **`Atelier Nord`** (single vendor, so vendor is
> not a confound in similarity — the signal is garment/style, not brand).
> **Currency:** EUR. **All products:** `Status = active`, `Published = TRUE`,
> inventory 100/variant (nothing filtered as unavailable, matching the current
> fixture). **Region:** EU demo store (unchanged).

---

## 0. TL;DR for the PO

- **200 adult products, 100 men + 100 women**, single brand, 8 categories each
  side, every category deep enough (**min 8 products per gender×category cluster**,
  most 10–15) that PDP similars always have strong near-neighbours.
- **3 style axes** woven through every category: **casual / classic / sporty**.
  Each (gender × category × style) sub-cluster holds **≥ 3 products** so a style
  tile's neighbourhood is never thin.
- **4 named demo styles** (anchor sets) to create after sync: *Smart Office*
  (classic men), *Weekend Casual* (casual men), *Studio Minimal* (classic/minimal
  women), *Active Edit* (sporty, cross-gender-capable). Each has its 2–3 anchors
  pre-identified below.
- **0-unknown classification is engineered in:** every product tag list starts
  with the plain `men` or `women` token, and the catalogue uses **none** of the
  kids/baby/boys/girls vocabulary — so gender resolves and age resolves to adult
  for all 200.
- **8–10 showcase PDP paths** are pre-designed (open the PDP, look): classic
  men's oxford → classic shirts/chinos/derbies; women's sporty capsule; a
  perfume cluster; two style-anchored cross-category-but-coherent paths.
- **Eval reuses the existing bars** (`recommender-eval-plan.md` S1–S16,
  `similars-gender-category-ranking.md` §6): **0 cross-gender leaks on gendered
  sources, 0 `unknown`, 100% adult, precision@5 ≥ 0.7, every SKU ≥ 3 similars**.
- This doc stops at **4 exemplar rows** to lock the format; the content step
  writes the other 196 following the per-category schema.

---

## 1. Category + count distribution (sums to 100 each)

**Design rule — minimum cluster size.** A PDP-similars slate is `k` (default 3,
configurable to 10) same-gender neighbours, with ≤ 2 per `product_type`
(diversity cap) and candidate-side availability filter. For a slate to be
*strong and non-repetitive*, a source product needs **enough same-gender
neighbours that the top-k are genuinely close, not scraped from a thin pool**.
I set the **minimum (gender × category) cluster = 8 products**, target 10–15 for
the hero categories. Rationale:

- With ≤ 2 per `product_type` in a slate, a 3-card slate can draw ≤ 2 from the
  source's own category and must find ≥ 1 complementary same-gender item — so
  the *catalogue as a whole* per gender (100 items, 8 categories) is the pool for
  the complementary card, and the *category* (≥ 8) is the pool for the
  same-category cards. 8 gives several near-neighbours after the top-1 is the
  source itself.
- For the **style-tiles** feature, each style's centroid NN must return a
  coherent, non-thin slate **after** the gender + budget hard filters. A style
  spanning a few categories needs **≥ 3 products per (gender × category × style)
  sub-cluster** so the centroid sits inside a populated region (catalogue-fit
  ≥ 0.60) and the slate fills.
- 8 is also the `recommender-eval-plan.md` C7 "≥ 5 categories, several SKUs
  each" bar, comfortably exceeded.

### 1.1 Men — 100 products, 8 categories

| # | Category (`product_type`) | Count | Shopify `Product Category` leaf | Style mix (casual / classic / sporty) |
|---|---|---|---|---|
| M1 | Shirts (camisas) | 14 | `Apparel & Accessories > Clothing > Shirts & Tops` | 5 / 6 / 3 |
| M2 | Outerwear (casacos + coats) | 13 | `Apparel & Accessories > Clothing > Outerwear` | 4 / 5 / 4 |
| M3 | Trousers (calças) | 13 | `Apparel & Accessories > Clothing > Pants` | 5 / 5 / 3 |
| M4 | T-Shirts (básicos/tees) | 13 | `Apparel & Accessories > Clothing > Shirts & Tops` | 6 / 3 / 4 |
| M5 | Sneakers (sapatilhas) | 12 | `Apparel & Accessories > Shoes` | 5 / 3 / 4 |
| M6 | Loungewear (pijamas) | 10 | `Apparel & Accessories > Clothing > Sleepwear & Loungewear` | 6 / 2 / 2 |
| M7 | Fragrance (perfumes) | 11 | `Health & Beauty > Personal Care > Cosmetics > Perfume & Cologne` | 4 / 5 / 2 |
| M8 | Accessories (acessórios) | 14 | per-item leaf (belts/scarves/hats/small leather goods — see §3.5) | 5 / 6 / 3 |
| | **Total men** | **100** | | 40 / 35 / 25 |

### 1.2 Women — 100 products, 8 categories

| # | Category (`product_type`) | Count | Shopify `Product Category` leaf | Style mix (casual / classic / sporty) |
|---|---|---|---|---|
| W1 | Dresses (vestidos) | 14 | `Apparel & Accessories > Clothing > Dresses` | 5 / 6 / 3 |
| W2 | Outerwear (casacos + coats) | 13 | `Apparel & Accessories > Clothing > Outerwear` | 4 / 5 / 4 |
| W3 | Tops & Basics (básicos) | 13 | `Apparel & Accessories > Clothing > Shirts & Tops` | 6 / 3 / 4 |
| W4 | Shoes (sapatos) | 13 | `Apparel & Accessories > Shoes` | 4 / 5 / 4 |
| W5 | Bags (malas) | 12 | `Apparel & Accessories > Handbags, Wallets & Cases > Handbags` | 4 / 5 / 3 |
| W6 | Fragrance (perfumes) | 11 | `Health & Beauty > Personal Care > Cosmetics > Perfume & Cologne` | 4 / 5 / 2 |
| W7 | Accessories (acessórios) | 14 | per-item leaf (scarves/hats/jewellery/small leather goods — see §3.5) | 5 / 6 / 3 |
| W8 | Loungewear / Knitwear-lounge | 10 | `Apparel & Accessories > Clothing > Sleepwear & Loungewear` | 6 / 2 / 2 |
| | **Total women** | **100** | | 38 / 37 / 25 |

**Why these two columns differ slightly** (men lead with shirts/trousers, women
with dresses/bags): it mirrors a real contemporary-fashion mix and gives the gift
flow genuinely giftable inventory on both sides (fragrance, accessories, bags,
scarves). Every category clears the **8-minimum**; hero categories (shirts,
dresses, outerwear, accessories) sit at 13–14 for depth.

**`product_type` discipline (C3).** The bracketed English noun in the table is
the **exact `product_type` string** the content step writes (e.g. `Shirts`,
`Trousers`, `T-Shirts`, `Dresses`, `Shoes`, `Sneakers`, `Bags`, `Outerwear`,
`Loungewear`, `Fragrance`, plus the accessory leaf types in §3.5). It must be
**specific, never a catch-all `Apparel`** — the ≤ 2-per-`product_type` diversity
cap and the same-category soft signal both key off it. Portuguese terms are the
PO's shorthand; the stored fields are English (artifacts in English per project
rule).

---

## 2. Style taxonomy + demo styles

### 2.1 The three style axes (every product carries exactly one)

Each product is tagged with exactly **one** of `casual` / `classic` / `sporty`
as its primary style token (plus finer style descriptors, §3.4). One primary
style per product keeps the style-tiles separable (§5.3 separation check) and
makes the labelled style set unambiguous.

| Style | Aesthetic the embeddings should cluster on | Example cues in copy/tags |
|---|---|---|
| **casual** | relaxed, everyday, soft, weekend | relaxed fit, washed cotton, everyday, weekend, denim, jersey |
| **classic** | tailored, timeless, smart, refined | tailored, oxford, merino, leather, smart, office, structured |
| **sporty** | athletic, technical, athleisure, active | technical, performance, mesh, stretch, running, gym, athleisure |

Distribution target across the 200: **~39% casual, ~36% classic, ~25% sporty**
(from the per-category mixes above). Every (gender × category × style)
sub-cluster holds **≥ 3 products** — this is the hard constraint the content
step must satisfy when it assigns the per-category style mix, so no style tile is
thin. Where a (gender × category × style) cell would fall below 3 (e.g. sporty
dresses), the mix tables above already round that style down to 3 and never below.

### 2.2 The 4 named demo styles (anchor sets to create after sync)

Each style = a `name` + a tile image (reuse an anchor's product image, per
`style-anchored-recommendations.md` §3.3) + **2–3 anchors** from THIS catalogue.
Anchors are chosen to be mutually coherent (high pairwise cosine → passes the
≥ 0.45 enable gate) yet to span a couple of categories (so the centroid is a
*taste*, not a single-product clone). The content step must guarantee the named
anchor handles exist with exactly the described character.

| Style name | Vertical | Gender lean | Anchors (3) — handle intents | What its slate should return |
|---|---|---|---|---|
| **Smart Office** | fashion | men | `m-classic-oxford-shirt`, `m-wool-tailored-trousers`, `m-leather-derby-shoes` | Classic men's shirts, tailored trousers, leather shoes, a merino knit, a structured coat — the "dress for work" edit. |
| **Weekend Casual** | fashion | men | `m-washed-relaxed-tee`, `m-slim-casual-chinos`, `m-canvas-low-sneakers` | Casual men's tees, chinos/denim, canvas sneakers, a casual overshirt — relaxed everyday. |
| **Studio Minimal** | fashion | women | `w-minimal-shift-dress`, `w-fine-merino-crew`, `w-leather-tote-bag` | Clean minimal women's dresses, fine knits, leather bags, minimal trainers/flats — understated, tonal. |
| **Active Edit** | fashion | cross | `w-performance-leggings`, `m-technical-running-tee`, `w-cushioned-runner-sneakers` | Sporty/athleisure both genders: technical tees, leggings, runners, track outerwear. *Note:* with no audience pick this slate spans genders in proportion to anchors; with a women's/men's pick the hard filter scopes it — a deliberate demo of the §1.3 "style = taste, filter = constraint" property. |

**Why 4, not more.** 4 covers the three style axes, both genders, and one
deliberate cross-gender style for the audience-filter demo, while staying at the
2–3 anchor sweet spot. The separation check (§5.3) is easy to pass with 4
well-separated styles; adding near-duplicate styles would fail it and demo badly.

> **Ambiguity flagged to PO.** Styles are created in the **admin after sync**
> (they are tenant data, not CSV rows). This doc *pins the anchor handles* so the
> content step produces products that match; the actual style creation + tile
> image is a follow-up admin step for `frontend-engineer` / the PO. If you want
> the styles seeded programmatically alongside the CSV, that is a separate small
> seed script — flag it and I will spec it.

---

## 3. Tag schema (drives embeddings AND classifiers)

### 3.1 Per-product fields (what the content step writes)

| CSV field | Rule | Why |
|---|---|---|
| `Handle` | `m-` or `w-` + kebab descriptor (e.g. `m-classic-oxford-shirt`). Unique. | gender-readable handle; matches anchor handles in §2.2 |
| `Title` | Natural, specific, carries garment noun + 1–2 distinguishing attributes. Gender word **optional** in title (classifier ignores title). E.g. "Men's Classic Oxford Shirt, Sky Blue". | C1 rich title; readable demo |
| `Body (HTML)` | **2–4 sentences of VARIED natural copy** + a `<ul>` of 3 bullets (material, fit/silhouette, use/occasion/season). **Never templated** — vary sentence structure, vocabulary, and attribute order across products or the embeddings collapse into one blob. | C2; embeddings quality — this is the single biggest lever on similars |
| `Vendor` | `Atelier Nord` (all 200) | brand not a confound |
| `Product Category` | the Shopify taxonomy leaf from §1 tables (per-item for accessories, §3.5) | storefront polish; not used by recommender |
| `Type` (`product_type`) | the exact English `product_type` string from §1 / §3.5 | diversity cap + soft category signal |
| `Tags` | ordered list per §3.2–3.4 | gender/age classifiers + embedding signal |
| `Variant Price` | per the price band for the category/style (§4) | budget filter |
| variant columns | Size/Color grouping per §3.6, inventory 100, SKU unique | Shopify grouping |
| `Status` / `Published` | `active` / `TRUE` | nothing filtered as unavailable |

### 3.2 Tag ordering + required cue (the 0-unknown contract)

Every product's `Tags` list is composed in this fixed order so the result is
deterministic and the classifiers resolve every row:

```
<gender-cue>, <category-noun>, <primary-style>, <material...>, <colour>, <fit/silhouette>, <occasion/season...>, <giftability>
```

1. **`<gender-cue>` (position 1, REQUIRED):** exactly `men` for all 100 men's
   products, `women` for all 100 women's products. This single token drives
   `GenderClassifier` → `men`/`women` and `AgeGroupClassifier` → `adult`
   (the ADULT set contains `men`/`women`; no kids/baby signal present). **Do not
   use `boys`/`girls`** (they flip age to KIDS). Fragrance and genuinely
   genderless accessories: see §3.3.
2. **`<category-noun>`:** e.g. `shirt`, `trousers`, `t-shirt`, `dress`, `shoes`,
   `sneakers`, `bag`, `coat`/`jacket`, `loungewear`, `fragrance`, `scarf`, etc.
3. **`<primary-style>`:** exactly one of `casual` / `classic` / `sporty`.
4. **material / colour / fit / occasion / season / giftability:** the controlled
   vocabulary in §3.4 — these enrich the embedding and power interest/occasion
   probes and the gift flow.

### 3.3 Gender for fragrance and unisex accessories

- **Fragrance is split, not unisex here.** To keep the men/women audience filter
  crisp for the demo, each fragrance is tagged `men` or `women` (men's colognes
  vs women's perfumes), counting toward the per-gender 100. This avoids a pile of
  `unisex` fragrances that would appear in *both* audience flows and muddy the
  gender-coherence check. A couple of deliberately `unisex` scents MAY be added
  as a showcase of the unisex path (§5), but they would be **extra** and are not
  in the 200 count unless the PO wants them — flag.
- **Accessories are gendered too** (men's belt vs women's scarf) for the same
  reason, each tagged `men`/`women`. If a genuinely unisex accessory is wanted to
  demo the "unisex is eligible for both source genders" rule
  (`similars-gender-category-ranking.md` §2.4), tag it `unisex` — but note it
  then classifies gender=unisex, age=**unknown** unless it ALSO carries `adult`
  or a men/women token. **To keep 0 unknowns, any `unisex` product MUST also
  carry the literal `adult` tag** (ADULT set includes `adult`). The default
  recommendation is: **no unisex in the 200**, all gendered, 0 ambiguity.

### 3.4 Controlled tag vocabulary (deterministic — content step picks from these)

Pick tokens ONLY from these lists so the catalogue is internally consistent and
the labelled eval sets are derivable by rule. Spelling is fixed (lowercase,
hyphenated as shown).

- **gender (pos.1):** `men` | `women` *(never `boys`/`girls`/`kids`/`teen`/`unisex`\*)*
  — \*`unisex` only with an added `adult` tag, and only if the PO opts in (§3.3).
- **category noun:** `shirt`, `overshirt`, `t-shirt`, `polo`, `trousers`,
  `chinos`, `jeans`, `coat`, `jacket`, `overcoat`, `parka`, `blazer`, `knit`,
  `dress`, `top`, `blouse`, `shoes`, `sneakers`, `trainers`, `loafers`,
  `derby`, `boots`, `heels`, `flats`, `bag`, `tote`, `crossbody`, `backpack`,
  `loungewear`, `pyjamas`, `robe`, `fragrance`, `scarf`, `belt`, `hat`, `beanie`,
  `cap`, `gloves`, `wallet`, `cardholder`, `sunglasses`, `jewellery`, `necklace`,
  `watch`.
- **primary style (pos.3, exactly one):** `casual` | `classic` | `sporty`.
- **material:** `cotton`, `organic-cotton`, `linen`, `wool`, `merino`,
  `cashmere`, `denim`, `leather`, `suede`, `silk`, `viscose`, `recycled-polyester`,
  `technical-fabric`, `mesh`, `cotton-twill`, `corduroy`, `nylon`, `canvas`.
- **colour:** `black`, `white`, `navy`, `grey`, `charcoal`, `beige`, `camel`,
  `brown`, `tan`, `olive`, `khaki`, `burgundy`, `sky-blue`, `sage`, `cream`,
  `terracotta`, `pink`, `red`, `green`, `blue`.
- **fit / silhouette:** `relaxed-fit`, `regular-fit`, `slim-fit`, `tailored`,
  `oversized`, `straight-leg`, `tapered`, `high-waisted`, `cropped`, `fitted`,
  `a-line`, `shift`, `structured`, `low-profile`, `boxy`.
- **occasion / season:** `everyday`, `weekend`, `office`, `smart-casual`,
  `occasion`, `evening`, `formal`, `travel`, `summer`, `spring`, `autumn`,
  `winter`, `all-season`, `resort`, `commute`, `running`, `gym`, `athleisure`,
  `lounge`, `sleep`.
- **giftability (gift-flow signal, C5):** `gift`, `giftable`, `premium`,
  `minimal`, `statement`. Apply `gift`/`giftable` to the genuinely giftable
  items (fragrance, accessories, bags, scarves, knitwear) — NOT to sized
  intimate items like loungewear-pyjamas, so the gift vs self branching differs
  (S12).

> **Why this matters for the embedding.** Because tags are concatenated into the
> embed input (`EmbeddingService`), the style/material/occasion tokens sharpen
> the vector *beyond* the free-text description — two casual cotton weekend tees
> land near each other; a classic merino office knit lands away from them. The
> controlled vocabulary makes those clusters tight and the demo predictable.

### 3.5 Accessory `product_type` + category leaf (per item)

Accessories are heterogeneous; give each its own `product_type` (so the
diversity cap doesn't treat a belt and a scarf as the same type) and taxonomy
leaf:

| Accessory | `product_type` | `Product Category` leaf |
|---|---|---|
| Belt | `Belts` | `Apparel & Accessories > Clothing Accessories > Belts` |
| Scarf | `Scarves` | `Apparel & Accessories > Clothing Accessories > Scarves & Shawls` |
| Hat / beanie / cap | `Hats` | `Apparel & Accessories > Clothing Accessories > Hats` |
| Gloves | `Gloves` | `Apparel & Accessories > Clothing Accessories > Gloves & Mittens` |
| Wallet / cardholder | `Small Leather Goods` | `Apparel & Accessories > Handbags, Wallets & Cases > Wallets & Money Clips` |
| Sunglasses | `Sunglasses` | `Apparel & Accessories > Clothing Accessories > Sunglasses` |
| Jewellery (women) | `Jewellery` | `Apparel & Accessories > Jewelry` |
| Watch | `Watches` | `Apparel & Accessories > Jewelry > Watches` |

The 14 men's + 14 women's accessories spread across these so each accessory
`product_type` has 3–5 items (meets the sub-cluster floor; men lean
belts/wallets/scarves/caps/sunglasses/watch, women lean scarves/jewellery/
sunglasses/hats/gloves/small-leather-goods).

### 3.6 Variants

Follow the existing fixture convention:
- **Clothing (shirts, tees, trousers, dresses, outerwear, loungewear, tops):**
  `Size` (XS–XL) × `Color` where natural; one CSV row per combination.
- **Shoes/sneakers:** `Size` only — EU 40–45 (men), EU 36–41 (women).
- **Bags:** single `Color` option, no size.
- **Fragrance:** `Size` = `50 ml` / `100 ml` (two variants), no colour.
- **Accessories:** `Color`, or `Size`×`Color` for belts; one option for
  sunglasses/jewellery/watch.

Keep the variant count sane (the current fixture is ~830 variant rows for 70
products; 200 products at a similar ratio is ~2,000–2,500 variant rows — fine for
Shopify import but note it to whoever runs the import). The content step also
regenerates `inventory.csv` (1 row per variant) as the README describes.

---

## 4. Price bands (EUR, per category — realistic, tier-spread)

Prices are chosen so the guided **budget tiers** (`budget-tiers.md`) each hold a
healthy pool (C8: each tier ≥ max(5, recs_per_widget, 10% of catalogue)), and so
style correlates loosely with price (sporty basics cheaper, classic/premium
dearer) without collapsing any tier.

| Category | Low band | Typical band | Premium band | Notes |
|---|---|---|---|---|
| T-Shirts / Tops & Basics | €16.90–24.90 | €25–39.90 | — | casual cheap; a merino/silk top can reach €49.90 |
| Loungewear / Pyjamas | €24.90–34.90 | €35–59.90 | €60–79.90 | silk/cashmere robe premium |
| Shirts | €29.90–39.90 | €40–69.90 | €70–89.90 | oxford/linen mid; tailored premium |
| Trousers | €34.90–49.90 | €50–79.90 | €80–99.90 | wool-tailored premium |
| Dresses | €34.90–54.90 | €55–89.90 | €90–129.90 | occasion/evening premium |
| Knitwear (within tops/lounge) | €39.90–59.90 | €60–89.90 | €90–119.90 | cashmere premium |
| Sneakers / Shoes | €49.90–69.90 | €70–99.90 | €100–139.90 | leather derby/boots premium |
| Outerwear | €69.90–99.90 | €100–159.90 | €160–229.90 | wool overcoat / technical parka premium |
| Bags | €49.90–79.90 | €80–129.90 | €130–199.90 | structured leather premium |
| Fragrance | €39.90–59.90 | €60–89.90 | €90–119.90 | 50 ml cheaper, 100 ml dearer |
| Accessories | €14.90–29.90 | €30–59.90 | €60–119.90 | scarf/belt cheap; watch/jewellery premium |

**Spread target across the 200** (so every budget tier is demo-able):
- **budget** (≤ €39.99): ~60 products (tees, basics, scarves, belts, caps,
  cheaper loungewear, entry fragrance 50 ml).
- **mid** (€40–89.99): ~90 products (shirts, trousers, dresses, knits, sneakers,
  bags, most fragrance).
- **premium** (≥ €90): ~50 products (outerwear, leather shoes/boots, premium
  dresses, structured bags, 100 ml fragrance, watches/jewellery).

The content step must keep **every category present in ≥ 2 tiers** so a budgeted
intent on any category-context PDP still fills.

---

## 5. Showcase neighbourhoods (the "open this PDP and look" moments)

These are pre-designed so the demo is reliable. Each names an **entry PDP** and
the **expected slate character** (gender-hard, category-soft, ≤ 2 per type). The
content step must ensure the named anchors exist with the stated character and
enough same-gender neighbours.

**N1 — Classic men's oxford shirt (hero PDP).** Open `m-classic-oxford-shirt`.
Expect: 1–2 other classic men's shirts (`m-poplin-dress-shirt`,
`m-linen-smart-shirt`) + ≥ 1 complementary classic men's piece (tailored
trousers `m-wool-tailored-trousers`, derby shoes `m-leather-derby-shoes`, or a
merino crew). **No women's items, nothing sporty/casual dominating.** This is the
gender-hard + category-soft showcase.

**N2 — Women's sporty capsule.** Open `w-performance-leggings`. Expect a sporty
women's slate: `w-technical-running-tee`/`w-seamless-sports-top`,
`w-cushioned-runner-sneakers`, a `w-track-jacket` — same gender, athleisure
coherent, cross-category. Demonstrates the sporty style cluster holds together.

**N3 — Men's weekend casual path.** Open `m-washed-relaxed-tee`. Expect casual
men's tees/overshirt + `m-slim-casual-chinos`/`m-straight-jeans` +
`m-canvas-low-sneakers`. A clean casual outfit, no tailoring, no sportswear.

**N4 — Women's occasion dress → accessories (gift-adjacent).** Open
`w-evening-midi-dress`. Expect other occasion/evening women's dresses + ≥ 1
giftable complement (`w-silk-scarf`, `w-statement-necklace`, a `w-heels`). Shows
category-soft complementarity and feeds the gift flow.

**N5 — Fragrance cluster (men).** Open `m-cedar-eau-de-cologne`. Expect other
men's fragrances (`m-amber-eau-de-parfum`, `m-citrus-cologne`) — a tight
same-`product_type` cluster capped at 2, plus 1 giftable men's accessory
(`m-leather-cardholder`). Demonstrates a clean within-category cluster + the
diversity cap preventing an all-fragrance slate.

**N6 — Fragrance cluster (women).** Open `w-rose-eau-de-parfum`. Mirror of N5 for
women; good for the gift-for-her scenario.

**N7 — Women's leather bag → leather goods.** Open `w-leather-tote-bag`. Expect
other women's bags (`w-leather-crossbody`, `w-structured-shopper`) + a leather
complement (`w-leather-flats`, `w-leather-belt`). Material + gender coherence.

**N8 — STYLE-ANCHORED: *Smart Office* (men, cross-category coherent).** Tap the
**Smart Office** style tile (no context PDP). Expect a cross-category men's edit:
a classic shirt, tailored trousers, leather derby, merino knit, structured coat —
**different categories, one coherent classic aesthetic.** This is the
style-centroid showcase: coherent *across* categories, which pure PDP similars
cannot do as directly.

**N9 — STYLE-ANCHORED: *Active Edit* + audience filter (cross-gender → filtered).**
Tap **Active Edit** with **no audience** → a sporty slate spanning both genders.
Then tap it again with **Women's** selected → the SAME taste, now women-only
(leggings, runner sneakers, track jacket, seamless top). Demonstrates
`style = taste, filter = constraint` (`style-anchored-recommendations.md` §1.3)
— a strong "wow" for the PO.

**N10 — Out-of-stock sale-recovery (optional).** Force one hero product out of
stock (e.g. `m-leather-derby-shoes`), open its PDP. Expect available same-gender
alternatives (other men's shoes/sneakers). Demonstrates the all-source /
available-candidate rule (`similars-gender-category-ranking.md` §5). Optional
because it needs a manual stock flip post-import.

---

## 6. Eval / showcase plan (verify after sync)

Reuses `recommender-eval-plan.md` (S1–S16) and
`similars-gender-category-ranking.md` §6 — the **same bars**, run against the new
200-product catalogue. Run the moment the catalogue is ingested, embeddings
built, and the cold PDP bank precomputed, **before** opening the demo.

### 6.1 Classification coherence (hard gates — the 0-unknown promise)

- **E1 — 0 `unknown` gender.** `SELECT count(*) FROM tenant_products WHERE
  tenant_id = :t AND gender = 'unknown'` → **must be 0** (every product carries a
  `men`/`women` pos-1 tag). (eval-plan S5 / similars §6.4; target was < 10%, we
  engineer **0**.)
- **E2 — 0 `unknown`/non-adult age.** `age_group` must be `adult` for all 200
  (no kids/baby vocabulary used). `count WHERE age_group <> 'adult'` → **0**.
  This is the explicit guard against the `boys`/`girls` trap.
- **E3 — gender split 100/100.** `count GROUP BY gender` → men 100, women 100
  (plus any opted-in unisex, which must also be age=adult).
- **E4 — 0 cross-gender leaks on gendered sources (hard, P0-style).** For every
  gendered source, 100% of precomputed PDP similars are gender-compatible
  (men→men/unisex, women→women/unisex). SQL over `precomputed_recs` ⋈
  `tenant_products.gender`. One wrong-gender card fails. (similars §6.1.)

### 6.2 Coverage + cold-start (every SKU demos)

- **E5 — every SKU ≥ 3 similars.** `count` of source SKUs with `< 3` rows in
  `precomputed_recs` → **0** (eval-plan S7, B3). The day-1 cold-start guarantee;
  the 8-per-cluster minimum is what makes this hold with the gender filter on.
- **E6 — row integrity.** 100% of active SKUs have non-empty title,
  `product_type`, ≥ 1 in-stock priced variant, non-null embedding (S1).
- **E7 — price/tier spread.** Each budget tier holds ≥ max(5, recs_per_widget,
  10% of catalogue = 20) products (C8); each category present in ≥ 2 tiers (§4).

### 6.3 Per-source neighbour quality (is it impressive?)

- **E8 — precision@5 ≥ 0.7, MRR ≥ 0.8** on a labelled set of ~15 source products
  spanning all categories both genders (S4/S10, B2). Labels derivable by rule
  from the controlled tags (same gender + same-or-complementary category + style
  proximity = relevant). Label once, freeze, re-use.
- **E9 — no near-duplicate stacking / diversity.** ≤ 2 per `product_type` per
  slate; no two colourways of the same product stacked (S6, A3). Spot-check the
  most colour-variant-heavy products (a tee, a dress).
- **E10 — cross-category presence.** Across gendered sources, a meaningful share
  of slates contain ≥ 1 different-`product_type` same-gender card (confirms
  category stays *soft*, trousers-for-a-shirt works; similars §6.3).
- **E11 — walk N1–N10.** Open each showcase PDP/style tile; confirm the slate
  matches the pre-written expected character. ≥ 90% of the showcase paths pass
  (A1). Record failures as `recommender-eval-plan.md` §5 insights.

### 6.4 Style-tiles quality (the new surface)

- **E12 — anchor-coherence per style.** Mean pairwise anchor cosine ≥ **0.45**
  for all 4 styles (enable without warning; `style-anchored-recommendations.md`
  §5.1). If *Active Edit* (cross-gender) dips to 0.30–0.45, that is acceptable
  with the admin warning and is itself a demo of the coherence panel.
- **E13 — catalogue-fit per style.** Centroid's best available-product match
  cosine ≥ **0.60** (§5.2) — the 3-per-sub-cluster minimum ensures the centroid
  sits in a populated region.
- **E14 — separation.** Pairwise top-10 Jaccard between the 4 styles < **0.5**
  for most pairs (§5.3). Smart Office vs Weekend Casual (both men) is the tightest
  pair — verify they still separate (classic vs casual tags should do it).
- **E15 — styled precision@5 ≥ 0.70 and > unstyled self baseline** on labelled
  styled sets (§5.4); with an audience pick, 100% of styled results
  gender/age-compatible.

### 6.5 Isolation / safety (unchanged bars)

- **E16 — no cross-tenant leakage (P0).** Second throwaway tenant with a disjoint
  catalogue; every 200-catalogue slate returns only this tenant's products; cache
  key tenant-scoped (S15, B4).
- **E17 — determinism.** Re-running the nightly cold build on the unchanged
  catalogue yields identical slates (S8).

**Pass/fail roll-up.** E1–E4 (classification + leaks) and E16 are blocking. E5
(coverage) and E8/E11 (quality + showcase) are the "is it impressive" bars. All
green ⇒ the 200-catalogue demo is safe and compelling to open.

---

## 7. Exemplar CSV rows (lock the format — 4 products)

Exact Shopify product-import shape (25 columns, same header as
`fashion-catalogue.csv`). First row per handle carries all product fields;
subsequent rows carry handle + variant/option/pricing only. The content step
copies this shape for the other 196. Header, for reference:

```
Handle,Title,Body (HTML),Vendor,Product Category,Type,Tags,Published,Option1 Name,Option1 Value,Option2 Name,Option2 Value,Variant SKU,Variant Grams,Variant Inventory Tracker,Variant Inventory Qty,Variant Inventory Policy,Variant Fulfillment Service,Variant Price,Variant Compare At Price,Variant Requires Shipping,Variant Taxable,Image Src,Image Position,Status
```

**Exemplar A — men, classic shirt, a *Smart Office* anchor (clothing, Size×Color):**

```csv
m-classic-oxford-shirt,"Men's Classic Oxford Shirt, Sky Blue","<p>A buttoned-down oxford cut from a mid-weight washed cotton that softens with every wear. The regular fit sits clean through the body without pulling, and the button-down collar holds its shape from desk to dinner.</p><ul><li>100% washed oxford cotton</li><li>Regular fit, button-down collar</li><li>Smart-casual staple for the office and beyond</li></ul>",Atelier Nord,Apparel & Accessories > Clothing > Shirts & Tops,Shirts,"men, shirt, classic, cotton, sky-blue, regular-fit, office, smart-casual, all-season",TRUE,Size,S,Color,Sky Blue,MSH-OXF-S-SKYB,250,shopify,100,deny,manual,59.90,,TRUE,TRUE,,,active
m-classic-oxford-shirt,,,,,,,,,M,,Sky Blue,MSH-OXF-M-SKYB,250,shopify,100,deny,manual,59.90,,TRUE,TRUE,,,active
m-classic-oxford-shirt,,,,,,,,,L,,Sky Blue,MSH-OXF-L-SKYB,250,shopify,100,deny,manual,59.90,,TRUE,TRUE,,,active
m-classic-oxford-shirt,,,,,,,,,M,,White,MSH-OXF-M-WHIT,250,shopify,100,deny,manual,59.90,,TRUE,TRUE,,,active
```

**Exemplar B — women, sporty leggings, an *Active Edit* anchor (clothing, Size only):**

```csv
w-performance-leggings,"Women's High-Waisted Performance Leggings, Black","<p>Sculpting high-rise leggings in a four-way-stretch technical knit that moves with you and wicks sweat on the run. A wide bonded waistband stays put through burpees and long miles alike, with a hidden key pocket at the back.</p><ul><li>Four-way-stretch technical fabric</li><li>High-waisted, squat-proof</li><li>Built for running, gym and studio</li></ul>",Atelier Nord,Apparel & Accessories > Clothing > Pants,Leggings,"women, leggings, sporty, technical-fabric, black, high-waisted, running, gym, athleisure",TRUE,Size,XS,Color,Black,WLG-PRF-XS-BLAC,180,shopify,100,deny,manual,44.90,,TRUE,TRUE,,,active
w-performance-leggings,,,,,,,,,S,,Black,WLG-PRF-S-BLAC,180,shopify,100,deny,manual,44.90,,TRUE,TRUE,,,active
w-performance-leggings,,,,,,,,,M,,Black,WLG-PRF-M-BLAC,180,shopify,100,deny,manual,44.90,,TRUE,TRUE,,,active
w-performance-leggings,,,,,,,,,L,,Black,WLG-PRF-L-BLAC,180,shopify,100,deny,manual,44.90,,TRUE,TRUE,,,active
```

**Exemplar C — women, premium fragrance (Size = volume, giftable):**

```csv
w-rose-eau-de-parfum,"Women's Rose & Peony Eau de Parfum","<p>A luminous floral built on fresh-cut rose and peony, warmed by a soft musk base that lingers without overpowering. Presented in a weighted glass flacon that looks considered on any dresser — an easy, elegant gift.</p><ul><li>Rose, peony and soft musk</li><li>Long-wear eau de parfum concentration</li><li>A refined present for her</li></ul>",Atelier Nord,Health & Beauty > Personal Care > Cosmetics > Perfume & Cologne,Fragrance,"women, fragrance, classic, floral, premium, gift, giftable, occasion, evening",TRUE,Size,50 ml,,,WFR-ROS-50ML,350,shopify,100,deny,manual,69.90,,TRUE,TRUE,,,active
w-rose-eau-de-parfum,,,,,,,,,100 ml,,,WFR-ROS-100ML,600,shopify,100,deny,manual,99.90,,TRUE,TRUE,,,active
```

**Exemplar D — men, accessory / small leather goods (giftable, single option):**

```csv
m-leather-cardholder,"Men's Slim Leather Cardholder, Brown","<p>A slim cardholder in full-grain vegetable-tanned leather that develops a rich patina over time. Four card slots and a central sleeve keep the everyday essentials to hand in a jacket pocket. Boxed and ready to give.</p><ul><li>Full-grain vegetable-tanned leather</li><li>Four card slots, slim profile</li><li>A classic, giftable everyday carry</li></ul>",Atelier Nord,"Apparel & Accessories > Handbags, Wallets & Cases > Wallets & Money Clips",Small Leather Goods,"men, cardholder, classic, leather, brown, everyday, gift, giftable, premium",TRUE,Color,Brown,,,MSL-CRD-BROW,120,shopify,100,deny,manual,39.90,,TRUE,TRUE,,,active
```

**Format notes for the content step (binding):**
- Pos-1 tag is `men`/`women` on EVERY row — this is the 0-unknown contract.
- No `boys`/`girls`/`kids`/`baby`/`teen`/`junior`/`youth` anywhere.
- One primary style token (`casual`/`classic`/`sporty`) on every row.
- `Type` is the specific `product_type`; accessories use the §3.5 types.
- `Image Src` left blank here (images are storefront-only, not used by the
  recommender this milestone; fill later per the README image policy if wanted).
- Vary the `Body (HTML)` prose genuinely — the four exemplars above use different
  sentence shapes on purpose; do NOT template.
- Prices respect the §4 band for the category/style.

---

## 8. Open questions for the PO

| # | Question | My recommendation |
|---|---|---|
| Q1 | 200 = 100 men + 100 men, single vendor `Atelier Nord`, 8 categories/side, min 8 per gender×category — accept the distribution in §1? | Accept; it clears every eval coverage bar and keeps similars strong. |
| Q2 | All products gendered (incl. fragrance + accessories), **no unisex** in the 200 — or add a few unisex items to demo the unisex path? | Default: no unisex (0 ambiguity, crisp gender filter). If you want the unisex demo, I add 2–3 unisex scents/accessories tagged `unisex, adult` as **extras** (beyond 200). |
| Q3 | The 4 demo styles (Smart Office, Weekend Casual, Studio Minimal, Active Edit) + their anchors — right set for the demo? | Yes; covers 3 style axes, both genders, and the cross-gender audience-filter showcase (N9). |
| Q4 | Styles are created in the admin after sync. Seed them programmatically alongside the CSV, or create by hand? | Hand-create for the demo (small); if you want them seeded, I'll spec a tiny seed script as a separate task. |
| Q5 | Images blank at import (recommender is text-only this milestone), filled later via Burst per the README policy? | Yes; keep the content step focused on text quality, which is what drives similars. |
| Q6 | Replace the 70-product `fashion-catalogue.csv` or stand the 200 up as a new fixture file alongside it? | New file (e.g. `fashion-catalogue-200.csv`) so the 70-set stays reproducible; retire the 70-set only after the 200 passes E1–E17. |
```
