# Fashion Catalogue — Shopify Import Fixture

A realistic contemporary-fashion catalogue (Zara/Mango-style) for a **dedicated
demo dev store**. This is a test bed for the WiseGift recommender: titles, HTML
descriptions, and tags are deliberately rich because the recommender embeds
**title + description + tags**. Products now carry **one real demo image each**
(see [Images](#images)).

- **File:** `fashion-catalogue.csv` (standard Shopify product-import format)
- **Vendor / demo brand:** `Atelier Nord`
- **Products:** 70 — Women (38) and Men (32)
- **Variant rows:** 829 (header + 829 = 830 lines total), 25 columns each
- **Price range:** €16.90 – €129.90
- **All products:** `Status = active`, `Published = TRUE`, inventory **100 per
  variant** (so nothing is filtered out as unavailable)
- **Product Category:** granular Shopify/Google product-taxonomy leaf per Type
  (e.g. Dresses → `… > Clothing > Dresses`, Shoes → `Apparel & Accessories >
  Shoes`).
- **Image Src:** one verified image per product on the product (first) row,
  `Image Position = 1`; variant rows left blank.

## What's in it

| Taxonomy | Covered types |
| --- | --- |
| Tops | T-Shirts (8), Shirts & Blouses (8), Knitwear (7) |
| Dresses | Dresses (5, women) |
| Bottoms | Trousers (8), Denim (6) |
| Outerwear | Outerwear / Jackets (8) |
| Footwear | Shoes (8) |
| Bags | Bags (5) |
| Accessories | Accessories (7) |

Price bands are spread so budget tiers are meaningful for the recommender
(by product): budget €15–24.99 (9), mid-low €25–49.99 (26), mid €50–79.99 (24),
upper €80–109.99 (8), premium €110–130 (3). Tees/accessories sit cheap;
outerwear, boots, and leather bags sit dear.

Each product carries real style/material/fit/occasion/colour/season cues in its
Title, Body (HTML), and Tags (e.g. "relaxed-fit organic cotton", "midi length",
"smart-casual", "high-waisted", "linen blend", "merino wool", "autumn/winter").
These are the signals the recommender learns from — vary or extend them if you
want to test specific ranking behaviour.

### Variants
- **Clothing:** `Size` (XS–XL) and, where natural, `Color` — one CSV row per
  Size×Color combination.
- **Shoes:** `Size` (EU 36–41 women, 40–45 men).
- **Bags:** single `Color` option (no size).
- **Accessories:** `Color`, or `Size`×`Color` for belts.

The first row of each `Handle` carries all product fields (Title, Body, Vendor,
Category, Type, Tags, Published, Option names). Subsequent rows for the same
product carry only the `Handle` plus variant/option/pricing columns — this is
the required Shopify grouping format.

## Import steps

1. In the **demo dev store** Shopify admin, go to **Products → Import**.
2. Upload `fashion-catalogue.csv`. Leave **"Overwrite products with matching
   handles"** unchecked for a first import (check it to re-sync after edits).
3. Review the preview (Shopify reports product/variant counts) and confirm.
4. After import, verify a few products group their variants correctly and show
   `Active` status with stock of 100 per variant.

CSV is UTF-8, uses standard Shopify column headers, and quotes any field
containing commas — it imports as-is with no spreadsheet round-trip needed.
(Opening it in Excel/Numbers and re-saving can corrupt quoting; upload the raw
file.)

## Images

Each product carries **one verified, hotlinkable image** on its product (first)
row, served from the **DummyJSON demo CDN** (`cdn.dummyjson.com`). URLs are
**percent-encoded** (DummyJSON paths contain apostrophes/ampersands/spaces, e.g.
`black-women's-gown` → `black-women%27s-gown`) so Shopify import and browsers
accept them. Every URL was checked to return `200 image/webp`; images vary
within each product Type (no two tees share a photo). Variant rows keep
`Image Src` blank — Shopify attaches the product-row image to all variants.

**Image-to-product mapping** (DummyJSON fashion categories → our Types):

| Our Type | DummyJSON pool | Match |
| --- | --- | --- |
| Dresses | `womens-dresses` | exact |
| Shoes (W / M) | `womens-shoes` / `mens-shoes` | exact |
| Bags (W) | `womens-bags` | exact |
| Bags (M) | `womens-bags` | approx (no `mens-bags` slug) |
| T-Shirts (W / M) | `tops` / `mens-shirts` | exact |
| Shirts & Blouses (W / M) | `tops` / `mens-shirts` | exact |
| Accessories | `sunglasses` / `womens-watches` / `womens-jewellery` | approx (per item) |
| **Knitwear, Trousers, Denim, Outerwear** | `tops` (W) / `mens-shirts` (M) | **approx stand-in** |

The four "gap" Types (Knitwear, Trousers, Denim, Outerwear) have no good
DummyJSON category, so they use a neutral upper-body clothing image as a
placeholder. **These are expected to be swapped later.**

> **Licensing — demo / design-partner use only.** DummyJSON images are a free
> demo/testing resource, fine for an internal dev store and design-partner
> walkthroughs. **Before any public demo or App Store listing, swap them for
> Shopify Burst (free stock, <https://burst.shopify.com>) or properly licensed
> product photography.** Do not treat these as cleared for marketing use.

**Replacing hero images after import:** any product's image can be changed in
the admin (product → Media → replace) or by re-importing with
`Overwrite = on`, a populated `Image Src` for the handle, and
`Image Position = 1`.

The recommender does **not** use images for this milestone — it embeds text
(title + description + tags) only. Images are purely for storefront polish.

## Regenerating / editing

The CSV is generated from a script that holds the hand-authored product content,
expands variant rows, maps the granular `Product Category` per Type, and assigns
a varied DummyJSON image per product. To change the catalogue, edit product
entries (title/body/tags/price/variants) and re-run the generator, then
re-validate (every row 25 columns, handles group correctly, prices/quantities
numeric, SKUs unique, every product row has a resolving `Image Src`). Re-verify
image URLs if the DummyJSON pool changes.
