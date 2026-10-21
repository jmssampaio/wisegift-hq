# Fashion Catalogue — Shopify Import Fixture

A realistic contemporary-fashion catalogue (Zara/Mango-style) for a **dedicated
demo dev store**. This is a test bed for the WiseGift recommender: titles, HTML
descriptions, and tags are deliberately rich because the recommender embeds
**title + description + tags**. Products in **matched Types carry one real demo
image**; all other Types are intentionally blank (see [Images](#images)).

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
- **Image Src:** verified image on the product (first) row for **matched Types
  only** (`Image Position = 1`); unmatched Types are blank. **24 products keep an
  image, 46 are blank.** Variant rows are always blank.

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

**Policy: an image is kept only where it genuinely depicts the product's garment
type; every other Type is intentionally blank** (to be filled later via Shopify
Burst). **24 products keep a verified image; 46 are blank.** Kept images are
served from the **DummyJSON demo CDN** (`cdn.dummyjson.com`), **percent-encoded**
(DummyJSON paths contain apostrophes/ampersands/spaces, e.g. `black-women's-gown`
→ `black-women%27s-gown`), each checked to return `200 image/webp`, and varied
within a Type. Variant rows always keep `Image Src` blank — Shopify applies the
product-row image to all variants.

**Coverage by Type:**

| Type | Image | DummyJSON pool / why blank |
| --- | --- | --- |
| Dresses | **kept** | `womens-dresses` (exact) |
| Shoes (W / M) | **kept** | `womens-shoes` / `mens-shoes` (exact) |
| Women's Bags | **kept** | `womens-bags` (exact) |
| Men's T-Shirts & Shirts | **kept** | `mens-shirts` (genuine shirts/tees) |
| **Women's T-Shirts & Shirts** | **blank** | DummyJSON `tops` is entirely frocks/dresses — no genuine top, so blanked |
| Men's Bags | blank | only women's-handbag images available |
| Accessories | blank | belts/scarves/hats had sunglasses/watch/jewellery stand-ins |
| Knitwear, Trousers, Denim, Outerwear | blank | no matching DummyJSON category (shirt/top stand-ins removed) |

Net kept (24): Women's Dresses 5, Women's Shoes 4, Women's Bags 3, Men's
T-Shirts 4, Men's Shirts & Blouses 4, Men's Shoes 4.

> **Licensing — demo / design-partner use only.** DummyJSON images are a free
> demo/testing resource, fine for an internal dev store and design-partner
> walkthroughs. **Before any public demo or App Store listing, swap them for
> Shopify Burst (free stock, <https://burst.shopify.com>) or properly licensed
> product photography.** Do not treat these as cleared for marketing use.

**Adding/replacing images after import:** blank Types (and any hero swap) are
filled in the admin (product → Media → add/replace) or by re-importing with
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
numeric, SKUs unique, kept Types have a resolving `Image Src` + `Image
Position = 1`, blanked Types have both empty). Re-verify image URLs if the
DummyJSON pool changes.
