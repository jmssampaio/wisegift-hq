# Fashion Catalogue — Shopify Import Fixture

A realistic contemporary-fashion catalogue (Zara/Mango-style) for a **dedicated
demo dev store**. This is a test bed for the WiseGift recommender: titles, HTML
descriptions, and tags are deliberately rich because the recommender embeds
**title + description + tags**. Images are intentionally omitted (see below).

- **File:** `fashion-catalogue.csv` (standard Shopify product-import format)
- **Vendor / demo brand:** `Atelier Nord`
- **Products:** 70 — Women (38) and Men (32)
- **Variant rows:** 829 (header + 829 = 830 lines total), 25 columns each
- **Price range:** €16.90 – €129.90
- **All products:** `Status = active`, `Published = TRUE`, inventory **100 per
  variant** (so nothing is filtered out as unavailable)
- **Image Src:** blank on every row (deliberate — do not fabricate URLs)

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

## Image note (images added AFTER import)

`Image Src` is **blank on every row by design** — fabricated image URLs would
404 and pollute the store. Add images after the products exist, by either:

- **Shopify Burst** (free stock, <https://burst.shopify.com>) — search e.g.
  "linen shirt", "wool coat", "white trainers", download, and attach to the
  matching product in the admin; or
- a **supplied image set** — place files per product and either attach in the
  admin or re-import with the `Image Src` / `Image Position` columns populated
  (public URLs, one row per image, matched by `Handle`).

The recommender does **not** need images for this milestone — it embeds text
only. Images are purely for the demo storefront's visual polish.

## Regenerating / editing

The CSV was generated from a script that holds the hand-authored product content
and expands the variant rows. To change the catalogue, edit product entries
(title/body/tags/price/variants) and re-run the generator, then re-validate
(every row 25 columns, handles group correctly, prices/quantities numeric, SKUs
unique). Keep `Image Src` blank unless you are intentionally supplying real URLs.
