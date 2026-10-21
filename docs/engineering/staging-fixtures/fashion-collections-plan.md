# Fashion Collections Plan — Demo Store Navigation

Shopify **smart (automated) collections** for the Atelier Nord demo store,
keyed to the **exact `Type` values and `women`/`men` + style tags** authored in
`fashion-catalogue.csv` (70 products). These power the store's nav and PLP
(product-listing) pages. Every rule below was verified against the CSV and
carries its real product count.

> **WiseGift scope note:** WiseGift does **not** ingest collection membership
> today. Collections exist purely for the demo storefront's navigation and PLP
> pages. The recommender still keys off product **Type + Tags** (which these
> collections are derived from), so the two stay consistent without a separate
> feed.

## Key Shopify constraint (drives the design)

A smart collection has **one** match mode — *all* conditions **or** *any*
condition — with **no parentheses / grouping**. So you **cannot** express
`gender = women AND (Type = T-Shirts OR Type = Knitwear)` in a single
collection: the gender `AND` and the type `OR` can't coexist.

Consequences, reflected below:
- **Gender + a single Type** → native, match **all** (e.g. Women's Dresses). ✅
- **Gender + multiple Types** (e.g. "Women's Tops & Knitwear") → **not** a
  single native collection. We implement these as a **navigation menu group**
  over several per-Type collections (recommended, zero fixture change), or
  optionally as a merged collection via an added category tag (see
  [Optional: merged PLPs](#optional-merged-category-plps)).
- **Style/occasion edits** (tag `OR` tag) → native, match **any**. ✅

## Setup path (per collection, ~30s each)

1. Shopify admin → **Products → Collections → Create collection**.
2. Set the **Title** (display name) and **Handle** (edit under *Search engine
   listing*).
3. Collection type → **Automated**.
4. Choose the **match mode** (all / any) and add the **conditions** exactly as
   listed, then **Save**.

> **No CSV import for collections.** Core Shopify has **no native
> collections-CSV import** — automated collections are created in-admin (fast:
> ~30s each). Membership then populates automatically from the product data.

Condition fields used: **Product type** (`is equal to`) and **Product tag**
(`is equal to`) — both exact-match, so values like `Shirts & Blouses` (with the
ampersand) must be typed exactly.

---

## Tier 1 — Top-level gender collections

| Display name | Handle | Match | Conditions (field · relation · value) | Products |
| --- | --- | --- | --- | --- |
| Women | `women` | all | Product tag · is equal to · `women` | **38** |
| Men | `men` | all | Product tag · is equal to · `men` | **32** |

---

## Tier 2 — Main nav (gender × category)

These are the **native per-Type gendered collections** (match **all**:
gender tag + Product type). They are the collections a merchant actually
creates, and they roll up into the menu groups shown in
[Navigation menu](#navigation-menu-what-the-shopper-sees). All conditions use
match mode **all**.

### Women

| Display name | Handle | Conditions | Products |
| --- | --- | --- | --- |
| Women's Dresses | `women-dresses` | tag=`women` · type=`Dresses` | 5 |
| Women's T-Shirts | `women-tshirts` | tag=`women` · type=`T-Shirts` | 4 |
| Women's Shirts & Blouses | `women-shirts-blouses` | tag=`women` · type=`Shirts & Blouses` | 4 |
| Women's Knitwear | `women-knitwear` | tag=`women` · type=`Knitwear` | 4 |
| Women's Trousers | `women-trousers` | tag=`women` · type=`Trousers` | 4 |
| Women's Denim | `women-denim` | tag=`women` · type=`Denim` | 3 |
| Women's Outerwear | `women-outerwear` | tag=`women` · type=`Outerwear` | 4 |
| Women's Shoes | `women-shoes` | tag=`women` · type=`Shoes` | 4 |
| Women's Bags | `women-bags` | tag=`women` · type=`Bags` | 3 |
| Women's Accessories | `women-accessories` | tag=`women` · type=`Accessories` | 3 |

Women total: **38** ✓

### Men

| Display name | Handle | Conditions | Products |
| --- | --- | --- | --- |
| Men's T-Shirts & Polos | `men-tshirts` | tag=`men` · type=`T-Shirts` | 4 |
| Men's Shirts | `men-shirts` | tag=`men` · type=`Shirts & Blouses` | 4 |
| Men's Knitwear | `men-knitwear` | tag=`men` · type=`Knitwear` | 3 |
| Men's Trousers | `men-trousers` | tag=`men` · type=`Trousers` | 4 |
| Men's Denim | `men-denim` | tag=`men` · type=`Denim` | 3 |
| Men's Outerwear | `men-outerwear` | tag=`men` · type=`Outerwear` | 4 |
| Men's Shoes | `men-shoes` | tag=`men` · type=`Shoes` | 4 |
| Men's Bags | `men-bags` | tag=`men` · type=`Bags` | **2 ⚠** |
| Men's Accessories | `men-accessories` | tag=`men` · type=`Accessories` | 4 |

Men total: **32** ✓

**⚠ Flag — tiny collection:** `men-bags` has only **2** products. It is **not**
used as a standalone nav link; it sits inside the **Men's Bags & Accessories**
menu group (6 products), which reads as a healthy PLP. (No collection is empty.)

### Navigation menu (what the shopper sees)

The 6/5 category groups are built in **Online Store → Navigation** as menu
headings linking the collections above. Group totals (distinct products):

**Women (6 groups)**
- **Dresses** → `women-dresses` *(5)*
- **Tops & Knitwear** → `women-tshirts`, `women-shirts-blouses`, `women-knitwear` *(12)*
- **Trousers & Denim** → `women-trousers`, `women-denim` *(7)*
- **Outerwear** → `women-outerwear` *(4)*
- **Shoes** → `women-shoes` *(4)*
- **Bags & Accessories** → `women-bags`, `women-accessories` *(6)*

**Men (5 groups — no Dresses)**
- **Tops & Knitwear** → `men-tshirts`, `men-shirts`, `men-knitwear` *(11)*
- **Trousers & Denim** → `men-trousers`, `men-denim` *(7)*
- **Outerwear** → `men-outerwear` *(4)*
- **Shoes** → `men-shoes` *(4)*
- **Bags & Accessories** → `men-bags`, `men-accessories` *(6)*

---

## Tier 3 — Curated "edit" collections

Cross-category style/occasion edits from the exact tags in the CSV. All use
match mode **any** (tag `OR` tag) — natively expressible. These span both
genders by design.

| Display name | Handle | Match | Conditions (Product tag · is equal to · …) | Products |
| --- | --- | --- | --- | --- |
| Workwear Edit | `workwear-edit` | any | `office`, `workwear`, `tailored`, `smart` | **17** |
| Autumn/Winter Edit | `autumn-winter-edit` | any | `autumn`, `winter` | **24** |
| Occasion Edit | `occasion-edit` | any | `occasion`, `evening`, `wedding guest`, `party`, `festive` | **12** |

**Swap-in alternates** (verified, if a different angle is wanted):
`Summer / Holiday Edit` → any of `summer`, `holiday`, `vacation`, `resort` =
**12**; `Weekend / Casual Edit` → any of `weekend`, `casual`, `loungewear`,
`athleisure`, `streetwear` = **24**; `Linen Shop` → any of `linen`,
`linen blend` = **5** (small but coherent).

---

## Optional: merged category PLPs

If you want the six groups to be **single browsable collections** (e.g. one
"Women's Tops & Knitwear" PLP) rather than menu groups, Shopify's one-match-mode
limit means you must add **one category tag per product** — the only native way
to merge Types under a gender. Suggested tags: `cat-tops`
(T-Shirts + Shirts & Blouses + Knitwear), `cat-bottoms` (Trousers + Denim),
`cat-bags-acc` (Bags + Accessories). Each merged collection is then match
**all**: gender tag + category tag. Verified resulting counts (all healthy, none
empty):

| Merged collection | Match | Conditions | Products |
| --- | --- | --- | --- |
| Women's Tops & Knitwear | all | tag=`women` · tag=`cat-tops` | 12 |
| Women's Trousers & Denim | all | tag=`women` · tag=`cat-bottoms` | 7 |
| Women's Bags & Accessories | all | tag=`women` · tag=`cat-bags-acc` | 6 |
| Men's Tops & Knitwear | all | tag=`men` · tag=`cat-tops` | 11 |
| Men's Trousers & Denim | all | tag=`men` · tag=`cat-bottoms` | 7 |
| Men's Bags & Accessories | all | tag=`men` · tag=`cat-bags-acc` | 6 |

**This is a fixture change (not applied):** it requires adding those 3 tags
across the relevant rows in `fashion-catalogue.csv` and re-importing. The
default plan above needs **no** fixture change. Flag to the coordinator which
path is preferred.

---

## Verification summary

- Counts computed by parsing `fashion-catalogue.csv` (70 products, first row per
  handle) and applying each rule's logic.
- **No empty collections.** Smallest standalone: `men-bags` = 2 (⚠ flagged;
  absorbed into the Men's Bags & Accessories group of 6).
- Gender totals reconcile: Women 38, Men 32 = 70. Each gender's per-Type
  collections sum back to its total.
- Edit collections all land in a useful 12–24 range.
