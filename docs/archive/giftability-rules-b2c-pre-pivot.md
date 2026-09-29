# WiseGift — Giftability Rules (Layer A: Universal)

Cross-links: [`data.md` — Merchant onboarding model](./data.md#merchant-onboarding-model)
| Decision log entry [2026-07-10](../decisions.md#2026-07-10--multi-merchant-awin-ingestion-pipeline-and-giftability-model).

## Editorial statement

**Compass: "gift-worthy, not everyday."** WiseGift surfaces items a giver would
be proud to hand over on a birthday, wedding, or Christmas — not commodity
purchases someone would buy for themselves without thought. When a rule is
borderline, apply this test: *"Would a thoughtful gift-giver actually pick this
for someone else on a real occasion?"* If the honest answer is "only under
niche circumstances," the rule should keep it out.

Two-layer model:
- **Layer A (this doc): universal exclusions and quality gates.** Applied to
  every product from every source, before Layer B.
- **Layer B: per-merchant category allow/deny lists.** Owned per merchant,
  drafted by catalog-data-engineer, approved by PO, versioned in
  [`decisions.md`](../decisions.md) at each merchant onboarding.

Both layers run at **ingest time** — rejected rows persist with
`active=false, rejection_reason=<rule id>` and are subject to the 30-day
purge (see [`data.md`](./data.md#30-day-purge)).

---

## 1. Universal exclusion table (topical)

All rules below are PO-approved. No pending items.

Legend for **Source**:
- **PO** — explicitly named by Product Owner on 2026-07-10.
- **PO (approved 2026-07-10)** — proposed by catalog-data-engineer, approved by PO on 2026-07-10.

Matching order: rules run top-to-bottom; first match wins and sets
`rejection_reason`. Patterns are illustrative — the actual regex lives in code
and can be tuned per feed's language mix (ES/PT/EN).

| ID | Rule name | Pattern (keyword / category / regex) | Rationale | Source |
|---|---|---|---|---|
| A-EX-01 | Adult content | Category names: `adult`, `sex toys`, `erotic`, `lingerie erótica`, `juguetes sexuales`, `+18`. Keyword regex on title/desc: `\b(sex toy\|vibrator\|dildo\|erotic\|adulto/a explicit)\b`. | Explicitly out of scope; App Store / Play Store safety; gifting context inappropriate. | PO |
| A-EX-02 | Weapons | Categories: `weapons`, `firearms`, `armas`, `caza`, `hunting`, `knives — combat`, `tactical`. Keywords: `\b(firearm\|gun\|rifle\|pistol\|ammunition\|munición\|arma de fuego\|navaja táctica)\b`. Kitchen knives excluded from this rule (Layer B decides). | Legal restrictions vary by market (ES/PT strict); not gift-appropriate. | PO |
| A-EX-03 | Alcohol | Categories: `wine`, `spirits`, `beer`, `champagne`, `vinos`, `licores`, `cervezas`. Keywords: `\b(whisky\|whiskey\|vodka\|gin\|ron\|rum\|tequila\|licor\|aguardiente\|vermouth\|wine\|vino\|cava\|champagne\|beer\|cerveza)\b`. | PO decision — universal exclusion (may become Layer B allow in future). | PO |
| A-EX-04 | Gambling | Categories: `gambling`, `lottery`, `betting`, `apuestas`, `lotería`. Keywords: `\b(lottery\|betting\|apuesta\|casino\|poker chips\|scratch card\|rasca\|quiniela)\b`. | Legal + editorial; addiction risk. | PO |
| A-EX-05 | Basic utensils / everyday commodities | Categories: `utensils`, `basic kitchen`, `menaje básico`. Keywords: `\b(dish sponge\|estropajo\|toilet paper\|papel higiénico\|rubbish bag\|bolsa de basura\|paper towels?\|papel de cocina)\b`. | Commodity purchase, not gift-worthy. | PO |
| A-EX-06 | Groceries / fresh food | Categories: `grocery`, `fresh food`, `frutas`, `verduras`, `carne`, `pescado`, `lácteos`, `pantry staples`. Keywords: `\b(milk\|leche\|bread\|pan\|eggs\|huevos\|fresh produce\|raw meat\|carne fresca)\b`. Pattern is deliberately narrowed to commodity groceries: gourmet chocolates, artisan foods, and curated gift hampers do NOT match these categories or keywords and pass through Layer A naturally (Layer B in Food & Drink merchants then decides). No override mechanism. | Perishability + commodity; not the WiseGift promise. | PO |
| A-EX-07 | Cleaning supplies | Categories: `cleaning`, `laundry`, `limpieza`, `detergentes`, `productos de limpieza`. Keywords: `\b(bleach\|lejía\|detergent\|detergente\|floor cleaner\|limpiacristales\|mop\|fregona\|scouring pad)\b`. | Commodity + hazardous; not gift-worthy. | PO |
| A-EX-08 | Hardware / DIY consumables | Categories: `hardware`, `DIY consumables`, `ferretería`, `bricolaje consumibles`. Keywords: `\b(screws?\|nails?\|tornillos\|clavos\|drill bits\|brocas\|sandpaper\|lija\|silicone sealant\|silicona\|paint stripper\|decapante)\b`. **Note:** curated tool sets / power tools as gifts may live in Layer B allow lists per merchant. | Commodity consumable, not gift-worthy. | PO |
| A-EX-09 | Tobacco / vaping | Categories: `tobacco`, `vape`, `e-cigarettes`, `tabaco`, `cigarrillos`, `vapeadores`. Keywords: `\b(cigarette\|cigar\|cigarro\|puro\|vape\|e-liquid\|nicotine\|tabaco de liar\|shisha\|cachimba)\b`. | Health + market restrictions (ES/PT age-gated advertising rules); consistent with alcohol exclusion. | PO (approved 2026-07-10) |
| A-EX-10 | Prescription / medical / pharma | Categories: `prescription`, `medical devices`, `pharmacy`, `farmacia`, `medicamentos`, `parafarmacia`. Keywords: `\b(prescription\|receta\|antibiotic\|antibiótico\|insulin\|thermometer digital\|blood pressure monitor\|tensiómetro\|pill organiser\|pastillero)\b`. **Exception:** wellness / self-care items (essential oils, massage tools) belong to Beauty & Wellness, not this rule. | Regulated products; unsuitable as surprise gifts; recipient sensitivity. | PO (approved 2026-07-10) |
| A-EX-11 | Live animals & livestock | Categories: `live animals`, `pets — live`, `mascotas vivas`, `livestock`. Keywords: `\b(live puppy\|live kitten\|goldfish live\|hermit crab live\|cachorro vivo)\b`. Pet accessories, food, toys are NOT excluded here (may sit in Layer B). | Ethical + logistical; live animals should never be a surprise gift. | PO (approved 2026-07-10) |
| A-EX-12 | Hazardous chemicals / dangerous goods | Categories: `hazardous`, `flammables`, `chemicals — industrial`, `productos peligrosos`. Keywords: `\b(sulfuric acid\|ácido sulfúrico\|lye\|sosa cáustica\|drain opener industrial\|fuel canister\|gasoline)\b`. | Safety + shipping restrictions; not gift-worthy. | PO (approved 2026-07-10) |
| A-EX-13 | Financial / investment products | Categories: `insurance`, `loans`, `investment products`, `crypto`, `seguros`, `préstamos`, `criptomoneda`. Keywords: `\b(insurance policy\|loan\|mortgage\|investment fund\|ISA\|pension plan)\b`. **Note:** hardware crypto wallets (e.g., Ledger) as a physical *device* gift are INCLUDED — they do not match the category or keyword patterns above and are Layer-B allow-listable via Tech merchants. The rule targets financial-product categories only (ISAs, tokens, investment funds, insurance), not the physical device. | Not a physical gift; regulatory risk; misaligned with product promise. | PO (approved 2026-07-10) |
| A-EX-14 | Funeral / grief supplies | Categories: `funeral`, `mourning`, `funeraria`, `urnas`. Keywords: `\b(urn\|coffin\|ataúd\|mourning wreath\|corona funeraria\|memorial plaque)\b`. | Editorial — misaligned with occasion set (birthday/Christmas/wedding etc.). | PO (approved 2026-07-10) |
| A-EX-15 | Gift cards / vouchers of other retailers | Categories: `gift cards — third-party`, `vouchers`, `tarjetas regalo terceros`. Keyword regex must include a third-party retailer signal (e.g. `\b(third[- ]party gift card\|multi[- ]retailer voucher\|tarjeta regalo terceros)\b`) — a merchant's own-brand gift card (issuer == redemption target) does not match this pattern and passes through Layer A naturally, where the merchant's Layer B decides. No override mechanism. | Cannibalises the WiseGift value prop (we curate; gift cards defer curation to recipient). | PO (approved 2026-07-10) |
| A-EX-16 | Digital-only / intangibles at ingest | Category: `digital downloads`, `software licence keys`, `ebooks`, `descargas digitales`. Keywords: `\b(ebook\|software licence\|licencia software\|game key\|steam key\|digital download)\b`. **Exception:** Products in the Experiences category (services with a physical or scheduled component: spa, dining, activities, workshops) are excluded from A-EX-16 and allow-listable via Layer B under `Experiences`. Purely digital fulfilment (ebooks, software keys, downloads with no physical or scheduled component) remains rejected. | Fulfilment complexity + returns/refunds risk; UX unclear for surprise gifting. | PO (approved 2026-07-10) |

---

## 2. Universal quality gates (structural, non-topical)

These are Layer A but not "topical exclusions" — they enforce data shape so
the recommendation slate never surfaces broken cards.

| ID | Gate | Condition | On failure |
|---|---|---|---|
| A-Q-01 | Required fields | `name`, `price`, `currency`, `image_url`, `affiliate_url`, `country`, `merchant_id` all non-null / non-blank | `active=false`, `rejection_reason=MISSING_REQUIRED_FIELD` |
| A-Q-02 | Price plausibility | `5.00 <= price <= 2000.00` in EUR (or currency-equivalent post-conversion at time of ingest) | `active=false`, `rejection_reason=PRICE_OUT_OF_RANGE` |
| A-Q-03 | Image URL shape | `image_url` is non-blank and starts with `http://` or `https://`. **MVP-scoped** — the original doc mandated an HTTP HEAD returning 200 + `Content-Type: image/*`, deferred because 300k+ HEAD calls per sync need a bounded thread pool, cache, and timeouts — its own future PR. URL-shape check catches malformed values only. | `active=false`, `rejection_reason=INVALID_IMAGE_URL` |
| A-Q-04 | Description length | `length(description) >= 50` after trim (or synthesised description for Amazon PA-API) | `active=false`, `rejection_reason=DESCRIPTION_TOO_SHORT` |
| A-Q-05 | Title length | `5 <= length(name) <= 200` | `active=false`, `rejection_reason=TITLE_INVALID` |
| A-Q-06 | Currency supported | `currency IN ('EUR', 'GBP', 'USD')` — expand as markets expand | `active=false`, `rejection_reason=CURRENCY_UNSUPPORTED` |
| A-Q-07 | Country supported | `country IN ('ES', 'PT', 'US', 'CO', NULL)` — NULL is accepted (Amazon PA-API path) | `active=false`, `rejection_reason=COUNTRY_UNSUPPORTED` |
| A-Q-08 | In-stock | Rejects when the feed **explicitly** reports out-of-stock via **either** signal: `in_stock = false` OR `stock_quantity <= 0`. **MVP-scoped: fail-open on double-null** — many Awin feeds (ECI observed 2026-07) leave `in_stock` blank; checking `stock_quantity` as a fallback catches the common ECI case. When both signals are null (unknown), the row is accepted. Further per-merchant availability heuristics belong in Layer B. | `active=false`, `rejection_reason=OUT_OF_STOCK` |
| A-Q-09 | Active-in-date | Feed's `validFrom`/`validTo` window (when present) covers `now()` | `active=false`, `rejection_reason=OFFER_EXPIRED` |
| A-Q-10 | Deep-link parseable | `affiliate_url` matches the expected pattern for the merchant network (Awin deep-link format, Amazon `/dp/{ASIN}?tag=`, Tradedoubler tracking URL) | `active=false`, `rejection_reason=AFFILIATE_URL_INVALID` |

---

## 3. How rules combine

- **All-must-pass.** Both the topical exclusion set (Section 1) and the quality
  gates (Section 2) must all pass for a product to reach `active=true`.
- **First-match rejection.** As soon as one rule matches or one gate fails, the
  record is written with `active=false` and the corresponding
  `rejection_reason` set. Ingest does not evaluate subsequent rules — this
  keeps the audit trail deterministic and the pipeline cheap.
- **Rejected rows are persisted** (not silently dropped) so we can inspect
  per-rule reject volumes. They are hard-deleted 30 days after their
  `deactivated_at` timestamp — see [`data.md` — 30-day purge](./data.md#30-day-purge).
- **Layer B runs after Layer A.** A record that passes Layer A is then
  evaluated against the merchant's Layer B category allow/deny list. Same
  semantics: first Layer B rejection wins.

---

## 4. How to add a rule

Owner: catalog-data-engineer proposes, PO approves, decision logged.

1. Decide the layer. Universal (any merchant, any market) → Layer A here. Merchant-scoped → Layer B in a decision-log entry at that merchant's onboarding.
2. Pick a rule ID. Layer A topical exclusions use `A-EX-NN`; quality gates use `A-Q-NN`. Increment the next free number in this file.
3. Write the pattern in code-agnostic terms (category names, keyword regex, numeric threshold). Keep patterns declarative — implementation lives in the ingestion adapter, not in this doc.
4. Justify it against the "gift-worthy, not everyday" compass. If the justification is thin, it probably belongs in Layer B.
5. Update this file (append a row; do not renumber existing rules).
6. Append a `docs/decisions.md` entry citing the rule ID and the rationale.
7. Flag to backend-expert if a code change is needed (usually yes for a new pattern; usually no for a threshold tweak already parameterised).

**Removing / relaxing a rule** follows the same path — never delete a row from
Section 1 or 2; instead add a "superseded by A-EX-NN on YYYY-MM-DD" note in
the rationale column and log the reversal in `decisions.md`.

---

## 5. Open items (PO review needed on this doc)

- A-EX-09 through A-EX-16 approval — **closed 2026-07-10** (all seven approved; Source column updated).
- A-EX-13 hardware crypto wallet decision — **closed 2026-07-10** (flipped to INCLUDED; Layer-B allow-listable via Tech merchants).
- A-EX-16 Experiences carve-out language — **closed 2026-07-10** (rewritten to distinguish physical/scheduled Experiences from purely digital fulfilment).
- Quality-gate thresholds — **closed 2026-07-10** (price band €5–€2000 kept; description length raised to ≥ 50 chars).
