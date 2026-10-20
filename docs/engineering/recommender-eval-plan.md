# Recommender evaluation plan — fashion demo store

> **Status: DRAFT for PO validation.** Owner: `recommendations-specialist`.
> Not committed by this draft; the coordinator handles git. Nothing here is
> settled policy until the PO signs off; `decisions.md` is not edited.
>
> **Purpose.** A working test plan to exhaustively exercise and polish the
> WiseGift recommender on the upcoming dedicated fashion demo store (Zara/Mango
> style catalogue, being generated at
> `docs/engineering/staging-fixtures/fashion-catalogue.csv`). Per CLAUDE.md,
> quality is **measured before we polish**: this doc defines the bars and the
> scripts, so "is it good enough / is it attractive enough" becomes a checklist,
> not a vibe.
>
> **Scope of surfaces under test.**
> 1. The **intent flow** (self + gift) — hook, form, routing.
> 2. The **recommendation slate** from `POST /widget/v1/recommendations`
>    (OpenAI ranker + embeddings over the tenant catalogue, budget filter,
>    `served_from`).
> 3. The neutral **PDP product-preview** (pgvector self-NN similars,
>    precomputed, no intent).
>
> **Out of scope here (referenced, not redone).** Exposed-vs-holdout lift
> measurement — conversion, AOV, revenue-per-session, with Widget CTR as the
> A/A integrity check — is fully specified in
> `docs/engineering/m1-2-lift-measurement.md`. This plan uses that as the
> *online* ground truth and does **not** re-derive it. This plan owns the
> **pre-demo / offline / scripted** layer that sits in front of it.
>
> **Sources read:** `docs/core/recommendations.md` (§2 measurement, §3
> cold-start, §4 serving policy, §5 retrieve+rank), `docs/product/spec.md`
> (Recommendation API, Widget intent/display, Cost guardrails),
> `docs/engineering/m1-2-lift-measurement.md`, `docs/engineering/budget-tiers.md`,
> `docs/engineering/staging-fixtures/gift-catalogue.csv` (format reference — the
> fashion CSV is the same Shopify product-export shape).

---

## 0. TL;DR — the "good enough" bars

The demo is judged on three tiers. A rec surface **ships to a merchant demo**
only when all **Tier-1** bars pass. Tier 2 is the quality bar we polish toward.
Tier 3 is the online truth, which only real traffic can settle.

**Tier 1 — Correctness / "not embarrassing" (hard pass/fail, no shoppers needed; §2).**
- B1 **Budget respected.** Every slate for a budgeted intent is 100% within the
  overlap budget rule (`decisions.md` 2026-10-09). Zero out-of-band products. Hard fail if any.
- B2 **Coherence.** Gift/self slates are category- and gender-coherent with the
  intent; the PDP preview similars are same-category or sensibly complementary,
  never random. Measured by precision@5 ≥ 0.7 on the labelled scenario set (§3).
- B3 **Cold-start coverage.** Every in-stock SKU yields a non-empty, sensible
  PDP-preview slate (≥ 3 items, de-duplicated, in stock). Zero empty slates.
- B4 **No cross-tenant leakage.** No slate on the fashion tenant ever returns a
  product from any other tenant. Hard fail, P0, if any.
- B5 **No error state ever shown.** Timeouts/zero-candidate fall back to
  precomputed/bestsellers; the shopper never sees an error (`spec.md`).

**Tier 2 — "Attractive enough" (the PO's actual question; §3).**
- A1 The scripted scenario matrix (§3) produces the **expected qualitative
  outcome** on ≥ 90% of cells, judged against the pre-written expectations, not
  live opinion.
- A2 Gift intent feels **gift-like** (giftable, presentable items surface for
  gift framings; not just "similar to what you browsed").
- A3 Slate **diversity** holds: ≤ 2 items per `product_type`, no near-duplicate
  colourways stacked.
- A4 `served_from` mix is healthy on the demo: live_llm only fires on real
  intent; passive PDP views serve `precomputed`; repeats hit `cache`.

**Tier 3 — Lift (online, post-demo).** Per `m1-2-lift-measurement.md`:
conversion / AOV / revenue-per-session exposed-vs-holdout, with CTR as the A/A
check. Not settleable on a demo store with no real shoppers — see §1.3.

**Headline numeric bars to quote back:** precision@5 ≥ **0.7** and MRR ≥ **0.8**
on the labelled scenario set; **0** budget violations; **0** empty PDP slates;
**0** cross-tenant items; scenario-matrix pass rate ≥ **90%**.

---

## 1. What "good" means operationally for this demo

### 1.1 Separate the measurable from the subjective

The PO's question — *"is it good enough / is it attractive enough?"* — bundles
two things that must be judged with different instruments:

| Question | Instrument | Where |
|---|---|---|
| Is it **correct**? (in budget, in category, in stock, no leaks, no errors) | Deterministic pass/fail checks | §2 |
| Is it **relevant**? (right items for the intent) | precision@5 / MRR against a human-labelled relevance set | §2.3, §3 |
| Is it **attractive**? (feels right, gift-like, well-merchandised) | Scripted scenario matrix with pre-written expected outcomes | §3 |
| Does it **sell more**? | Exposed-vs-holdout lift | `m1-2-lift-measurement.md` |

"Attractive" is the fuzziest, and it is exactly where polish burns time if it is
unmeasured. The discipline (§3): **write the expected outcome before looking at
the slate.** The PO scores the live slate against a fixed expectation sheet. A
disagreement is a logged insight (§5), not a silent re-tune.

### 1.2 Why a labelled set, in plain terms

precision@5 = "of the 5 products shown, how many a human would call a reasonable
fit for this intent." If 4 of 5 fit, precision@5 = 0.8. MRR = "how high up is
the first genuinely good item" (1.0 = the top card is great; 0.5 = the first
good one is in position 2). We label a small fixed set of intents once, then
every ranker/prompt/embedding change is re-scored against the **same** labels,
so we can see improvement or regression as a number instead of an argument.

### 1.3 What the demo store can and cannot prove

- **Can prove:** correctness (Tier 1), relevance/attractiveness (Tier 2). These
  need only a catalogue and a human judge — no shopper traffic.
- **Cannot prove:** lift (Tier 3). A demo/staging store has no real buyers, so
  conversion / AOV / RPS are not measurable. Do **not** quote lift off the demo.
  The demo de-risks relevance so that the *first real pilot* spends its scarce
  sessions on a recommender we already believe is good. This is the honest
  framing to give any design partner.

---

## 2. Offline / pre-demo sanity checks (no shoppers)

Run these the moment the fashion catalogue is ingested and embeddings +
precomputed cold bank are built, **before** anyone opens the demo widget. Each
is a concrete pass/fail a human (or a small script) can run against staging.

### 2.1 Catalogue ingest sanity
- **S1 Row integrity.** Every active SKU from `fashion-catalogue.csv` has a
  `tenant_products` row with: non-empty title, `product_type`, ≥ 1 in-stock
  priced variant, and a non-null embedding. Pass = 100% of active rows.
- **S2 Price sanity.** `price_min`/`price_max` per product are populated from the
  cheapest **available** variant (per `budget-tiers.md`); no zero/null prices on
  active products.
- **S3 Availability filter.** Products with all variants out of stock/archived
  are excluded from candidate retrieval but retained in the DB (`spec.md`).
  Spot-check: force one product out of stock → it disappears from slates.

### 2.2 PDP self-NN similars (precomputed) — the product-preview surface
- **S4 Not random.** For 10 hand-picked source products spanning categories
  (dress, knitwear, shoe, bag, accessory, outerwear), the top-5 precomputed
  similars are **same category or a sensible complement**, not scattered.
  Human precision@5 ≥ **0.7** averaged over the 10 (B2).
  - *Worked example:* source = "Silk Scarf, Terracotta" → expect other scarves,
    accessories, soft-goods; a reasonable complement is a coordinating knit or
    bag. A **fail** looks like: a snowboard, a men's hiking boot, a gift card.
- **S5 Gender coherence.** A women's product does not return predominantly men's
  products (and vice versa) unless the item is genuinely unisex. Spot-check 5
  gendered sources; ≥ 4/5 gender-coherent in the top 5.
- **S6 No near-duplicate stacking.** The same product in another colourway does
  not fill the slate. ≤ 2 items of the same `product_type` (`recommendations.md`
  §5.4 penalty). Spot-check the most colour-variant-heavy product.
- **S7 Full coverage (cold-start).** Every in-stock SKU has ≥ 3 precomputed
  similars. Query: count source SKUs with `< 3` rows in `precomputed_recs` →
  must be **0** (B3). This is the day-1 cold-start guarantee.
- **S8 Determinism.** Re-running the nightly cold build on an unchanged
  catalogue yields identical slates (seeded tie-breaks, `recommendations.md`
  §2.4). Diff two runs → empty.

### 2.3 Live slate (`POST /widget/v1/recommendations`) with synthetic intent
Run the labelled synthetic intent set (§3 matrix, as API payloads — no widget
needed) directly against the endpoint.
- **S9 Budget strictly respected.** For every budgeted payload, **0** returned
  products violate the overlap budget rule (`decisions.md` 2026-10-09; enforced
  post-LLM as a hard filter per `recommendations.md` §5.2). This is a hard gate:
  one violation fails the build. Check both `budget_min`/`budget_max` and the
  slate never pads with out-of-band items.
- **S10 Category/gender coherence vs intent.** Gift-for-her-birthday does not
  return men's workwear; "running" interest does not return formalwear.
  precision@5 ≥ **0.7**, MRR ≥ **0.8** over the labelled set (B2).
- **S11 Diversity constraint.** ≤ 2 per `product_type` per slate; slate size
  honours `limit` (default 5) and never pads with irrelevant items (A3).
- **S12 Self vs gift branching actually differs.** For the same context SKU and
  budget, the self slate and gift slate are **not identical** — gift framing
  surfaces more giftable/presentable items (`recommendations.md` §5.3). If they
  are byte-identical, the prompt branching is not wired; flag it.
- **S13 `served_from` is correct.** Real-intent payload → `live_llm` (first
  call) then `cache` (repeat identical payload). No-intent PDP payload →
  `precomputed`. Forced LLM timeout → `fallback`, 200 OK, non-empty slate.
- **S14 Traceability.** Every response carries `model_version`, `prompt_version`,
  `feature_version`, `served_from` (`recommendations.md` §2.4). Missing any → fail;
  we cannot trace a bad slate back without them.

### 2.4 Isolation and safety
- **S15 No cross-tenant leakage (P0).** Create a second throwaway tenant with a
  disjoint catalogue. Every fashion-tenant slate returns only fashion-tenant
  products. Also verify the cache key is tenant-scoped (`recommendations.md`
  §4.2): identical intent on two tenants must not share a cached slate (B4).
- **S16 Guardrail fallbacks.** Flip the tenant kill switch → every call serves
  `precomputed`, widget renders normally (`spec.md` cost guardrails). Exceed the
  per-session rate limit → `cache`/`precomputed`. No error state (B5).

> **Pass/fail roll-up.** S1–S16 all green = Tier-1 cleared, the demo is safe to
> open. Any red in S9 (budget) or S15 (leakage) is a blocking stop.

---

## 3. Scripted test scenarios (PO runs by hand on the demo store)

A fixed script so polishing is measured against a constant, not re-litigated
each session. The PO walks the widget exactly as a shopper would, for each cell,
and scores the slate against the **pre-written expected outcome**.

### 3.1 How to run
1. For each row: open the chosen **entry product** PDP (or home hero for
   no-context rows), click the hook, pick the intent mode, fill the form as
   specified, submit.
2. Record: the 5 product titles returned, `served_from` (visible in dev tools /
   network), and a 0–5 "fit" count (how many of 5 match the expected outcome).
3. Score the cell **pass** if fit ≥ 4/5 **and** budget 100% respected **and** no
   obvious coherence break (wrong gender/category). Otherwise **fail** + a
   one-line reason → that reason is a §5 insight.
4. Target: ≥ **90%** of cells pass (A1).

### 3.2 The matrix

Dimensions (keep it to a runnable size — ~24 cells, not a combinatorial
explosion):

- **Intent mode:** self, gift
- **Relationship (gift only):** Partner, Friend, Colleague, Family
- **Occasion (gift only):** Birthday, Anniversary, Christmas, Just because
- **Budget tier:** low / mid / high (use the tenant's own guided tiers from
  `budget-tiers.md`), plus one "Any price"
- **Entry product:** a dress PDP, a knitwear PDP, an accessory PDP, home hero
  (no context)

Suggested concrete cells (PO can extend):

| # | Mode | Relationship | Occasion | Budget | Entry | Expected qualitative outcome |
|---|---|---|---|---|---|---|
| 1 | self | — | — | mid | dress PDP | Items the browser would buy for herself near the dress: coordinating dresses/tops/accessories, same gender, in budget. |
| 2 | self | — | — | low | knitwear PDP | Budget knitwear/basics; nothing above the low tier; no luxury outerwear. |
| 3 | self | — | — | Any price | accessory PDP | Accessory-led slate, broad price, still category-coherent. |
| 4 | self | — | — | high | home hero | Premium self-purchase picks, diverse categories, all high tier. |
| 5 | gift | Partner | Anniversary | high | dress PDP | Romantic/presentable gift items for a partner; giftable (scarf, jewellery, premium knit), not a near-clone of the dress. |
| 6 | gift | Partner | Birthday | mid | accessory PDP | Giftable accessories + a complementary piece; feels like a gift, in mid budget. |
| 7 | gift | Friend | Birthday | mid | knitwear PDP | Safe, likeable friend-gift items (scarf, beanie, small leather goods); unisex-leaning ok. |
| 8 | gift | Friend | Just because | low | home hero | Small, affordable giftable items; nothing over the low tier. |
| 9 | gift | Colleague | Christmas | low | accessory PDP | Neutral, low-risk colleague gifts (card holder, beanie, scarf); not intimate apparel. |
| 10 | gift | Colleague | Just because | mid | home hero | Tasteful neutral gifts; no overly personal/sized clothing. |
| 11 | gift | Family | Christmas | mid | knitwear PDP | Warm, universally-liked family gifts (cashmere beanie, scarf); in budget. |
| 12 | gift | Family | Birthday | high | dress PDP | Premium giftable pieces; presentable; not a workwear dump. |
| 13 | gift | Partner | Anniversary | Any price | accessory PDP | Partner gift, price-agnostic; still relationship-appropriate. |
| 14 | gift | Friend | Christmas | high | dress PDP | Higher-budget friend gift; giftable, not identical-category clones. |
| 15 | self | — | — | mid | accessory PDP + interests="running" | Sporty/athleisure-leaning items in budget; coherence with the "running" interest, not formalwear. |
| 16 | self | — | — | low | home hero + interests="minimal, leather" | Minimal leather goods / neutral basics; interest terms clearly reflected. |

Add gift cells for the remaining relationship × occasion corners as time allows;
keep every added cell's expected outcome **written before** the run.

### 3.3 Interest free-text probes (self + gift)
Because interests are free-text (`recommendations.md` §4.2), add 4 probes that
stress the embedding input: a single clear interest ("cashmere"), two interests
("silk, scarves"), a vague interest ("cosy"), and a near-miss/absent interest
("snowboarding" — expect graceful, not empty). Expected: the first three bias
the slate visibly; the near-miss falls back to coherent in-budget items without
erroring.

---

## 4. What the catalogue itself needs for good recs

Hand this checklist back to whoever tunes `fashion-catalogue.csv`. The
recommender is only as good as the text it embeds: embeddings are built from
`title + description + product_type + tags` (`spec.md`, `recommendations.md`
§5.1). Thin or inconsistent fields blind it.

**Required for every active product:**
- **C1 Rich title** — includes the garment noun and a distinguishing attribute
  ("Silk Scarf, Terracotta"), not a bare SKU or style code.
- **C2 Real description** — ≥ ~1–2 sentences covering material, use, and style.
  Median description length must clear the ingest floor (`recommendations.md`
  §3.3 / `data.md`); one-word descriptions produce weak, clustering vectors.
- **C3 Specific `product_type`** — the diversity constraint and self-NN
  coherence both key off it. "Scarves", "Knitwear", "Shoes" — not a catch-all
  "Apparel" for everything (which would collapse the ≤2-per-type rule).
- **C4 Gender tag** — `women` / `men` / `unisex` in tags. Without it, S5 gender
  coherence cannot hold and gift-for-her returns menswear.
- **C5 Giftability / occasion signal in tags** — `gift`, material, season,
  style (`silk`, `winter`, `minimal`). The gift prompt leans on these to surface
  presentable items; a catalogue with no gift/style tags makes gift and self
  slates converge (S12 fails).
- **C6 Accurate stock + price** — in-stock flag and a real price per variant;
  these drive the availability filter (S3) and the budget filter (S9).

**Coverage checks on the catalogue as a whole:**
- **C7 Category spread** — enough products per `product_type` to fill a diverse
  slate; avoid a catalogue that is 90% dresses (`recommendations.md` §3.3
  category imbalance). Aim ≥ 5 categories, each with several SKUs.
- **C8 Price spread across tiers** — each guided budget tier (`budget-tiers.md`)
  must hold ≥ `max(5, recs_per_widget, 10% of catalogue)` products, or the tier
  returns thin/empty slates. Verify before the demo.
- **C9 Size** — ≥ 200 SKUs (pilot floor, `recommendations.md` §3.3). For a
  convincing Zara/Mango feel, aim higher so slates don't visibly repeat.
- **C10 Gift-appropriate inventory** — the catalogue must actually contain
  giftable items (scarves, accessories, small leather goods, knitwear) so gift
  intent has something to return; a pure dress catalogue has no good gift answer.

**Gaps that would blind the recommender (flag to catalogue tuner):** missing
gender tags (C4), catch-all `product_type` (C3), one-line descriptions (C2), no
gift/style tags (C5), a single price band (C8). Any of these turns a "ranker
problem" into a "data problem" — fix the data first, or §5 insights will chase
the wrong lever.

---

## 5. The polish loop (insight → re-testable change → re-measure)

The iteration protocol that turns *"this rec is wrong because…"* into a measured
change. No ranking change without a metric (CLAUDE.md, `recommendations.md` §2).

1. **Capture the insight precisely.** Log the failing cell: intent payload,
   `served_from`, the 5 returned titles, `model_version` + `prompt_version` +
   `feature_version` (always present, §2.3 S14), and the one-line "wrong
   because…". This makes it reproducible.
2. **Classify the cause — which lever.** Route the insight to exactly one lever;
   most "bad rec" reports are mis-attributed:
   - **Data** → fix the catalogue (§4). (Wrong gender shown → missing C4 tag.)
   - **Budget logic** → the overlap filter / tier edges (`budget-tiers.md`).
   - **Retrieval** → embedding input or K (`recommendations.md` §5.1).
     (Relevant item never appears → it is not in the top-K candidates.)
   - **Ranking** → the self/gift prompt template (`recommendations.md` §5.3).
     (Right candidates retrieved but ordered badly / wrong framing.)
   - **Filters/diversity** → post-LLM slate constraints (§5.2).
   - **Precomputed similars** → the cold-bank scoring (§5.4) for PDP preview.
   A quick triage: is the good item **in the top-50 candidates** at all? If no →
   retrieval/data. If yes but not shown → ranking/filters.
3. **Make the smallest change on that one lever.** One concern per branch
   (CLAUDE.md). Prompt text is diff-reviewable; bump `prompt_version`.
4. **Re-score on the fixed labelled set BEFORE merge.** Re-run §2.3 S9–S12 +
   the §3 matrix. The change merges only if precision@5 / MRR are **≥** the
   prior baseline on the full set — not just the one cell it was meant to fix.
5. **Guard against silent regression.** If the change fixes the target cell but
   drops precision@5 on the set beyond the agreed threshold (TBD at first eval
   PR, `recommendations.md` §2.3), it does **not** merge without an explicit
   accepted-regression note (the PO decides the trade; append to
   `decisions.md` only if the human asks).
6. **Seed everything.** Random tie-breaks are seeded (`recommendations.md`
   §2.4), so a re-run is comparable. If a slate changes between identical runs
   with no code change, that is a bug, not noise.
7. **Only then consider the next insight.** One lever, one measurement, repeat.

This keeps "polishing with insights" honest: every insight ends as either a
merged measured improvement, an accepted-regression note, or a catalogue fix —
never an un-recorded prompt tweak.

---

## 6. Complete-the-look (complementarity) — a different signal, evaluated separately

The PDP preview today is **similarity** (self-NN: "more things like this"). The
complete-the-look differentiator is **complementarity** ("goes *with* this"): a
belt and shoes for a dress, not more dresses. These are different objectives and
**must not be measured with the same labels**.

- **Why similarity metrics mislead here.** A great complement (dress → clutch)
  has *low* embedding cosine to the source, so it scores badly on self-NN
  precision@5 and would be penalised by the same-`product_type` rule
  (`recommendations.md` §5.4) that is correct for the similarity surface. Judging
  complementarity by similarity precision would reward the wrong thing.
- **The signal it would need (when it lands).** One of: (a) co-purchase / outfit
  co-occurrence from order line-items (not available cold — needs the v2
  purchase-outcome loop, `recommendations.md` §8); (b) explicit
  outfit/complementary-category rules (dress→{shoes, bag, outerwear,
  jewellery}); or (c) an LLM "style a full look" prompt over the candidate set.
  At cold start only (b) and (c) are available — no behavioural signal exists.
- **How we would evaluate it separately.**
  - A **distinct labelled set**: for each source product, a human marks which
    candidates *complete the look* (different category, coherent style/colour,
    sensible to wear together) — explicitly **not** "is similar".
  - Metrics on that set: complementary-precision@k and **category-diversity of
    the complement slate** (a good complete-the-look slate is high-diversity by
    design — the opposite of the similarity slate's low-diversity bar).
  - A **"would you wear these together?"** human pass/fail per slate, scripted
    like §3.
  - Online, if it ships to a pilot: evaluate as its own placement in the lift
    framework (`m1-2-lift-measurement.md`), with add-to-cart-of-complement as the
    exposed-only diagnostic — never folded into the similarity surface's numbers.
- **Recommendation:** keep complete-the-look **out of the MVP demo's scored
  surfaces**. If shown at all, label it clearly and score it on its own sheet.
  Mixing it into the §2/§3 bars would corrupt both.

---

## 7. Open questions for the PO

| # | Question | My recommendation |
|---|---|---|
| Q1 | precision@5 ≥ 0.7 / MRR ≥ 0.8 as the Tier-2 bars — accept, or tune after the first labelled run? | Accept as provisional; re-baseline once the fashion labels exist. |
| Q2 | Who labels the relevance set — PO + recommendations-specialist, as `recommendations.md` §2.3 says? | Yes; label once, freeze, re-use for every change. |
| Q3 | Scenario-matrix pass bar at 90% — right for a demo? | Yes for demo-ready; raise to 95% before a paying pilot. |
| Q4 | Is complete-the-look in or out of the demo? | Out of scored surfaces for MVP (§6); revisit when a complement signal exists. |
| Q5 | The accepted-regression threshold on precision@5 (still TBD, `recommendations.md` §2.3) — set it at the first eval PR? | Yes; propose ≤ 0.03 absolute drop needs a note, anything larger blocks. |
```
