# Merchant admin — Configuration redesign v3 (for the EDITORIAL "Shopping Guide" widget)

> **VALIDATED by the PO (2026-10-29)** as the admin direction, off the HTML mockup
> built from this doc. All open questions (§6) are **RESOLVED** — the locked
> decisions are folded into the spec below. Written by `product-analyst` for
> `frontend-engineer` (admin UI) and `backend-engineer` (contracts, §8) + the PO.
> Design + interaction + configuration model + contracts + acceptance criteria only
> — **no implementation, no code.** Individual PRs are still human-gated and "done"
> only on the PO's per-PR validation, but the **direction is locked.**
>
> **What this does:** adapts the merchant admin to the **PO-approved EDITORIAL
> design direction** — the **4 hooks + a stepped "Shopping Guide" wizard** (the
> PNGs in `docs/engineering/ui-designs/hooks/` and `.../form/`). It (1) inventories
> exactly what the widget becomes so the admin can configure all of it, (2) defines
> the full configuration model reusing existing plumbing where it fits and flagging
> genuinely-new config the backend doesn't model yet, (3) proposes a **more usable
> admin IA** (the PO's explicit goal: "an admin that's easier to use"), and (4)
> gives acceptance criteria + decision-focused open questions.
>
> **Inputs read (HQ):**
> - Designs: `docs/engineering/ui-designs/hooks/{floating_button,landing,collection,pdp}_hook.png`,
>   `docs/engineering/ui-designs/form/{form_step1_gift_enable,form_step2,form_step2_for_me,form_step3,form_step3_for_me,formt_step3_form_me_desktop,form_step4,form_results_mobile}.png`
> - Current state: `docs/product/product-finder-simplified-spec.md` (the 4-hook scope),
>   `docs/product/admin-config-ia-v2.md` (admin PR #23: one "Configuration" page,
>   sections Appearance / Hooks / Form; text-size slider → `font_scale`;
>   editable-strings editor over `pf.*` via `/admin/v1/widget/strings`),
>   `docs/decisions.md` (declutter to one Configuration page, budget-selector
>   removal, 4-hook simplification).
>
> **This design REVERSES some recent decisions — confirmed by the PO as the new
> direction.** Called out explicitly so the walk-backs are deliberate (§1.2):
> the **budget SLIDER is back** (reverses the budget-selector removal / range
> tiles), **occasion is back** (reverses the occasion drop, Q-PF1), the **form is
> a step WIZARD** (reverses the single-scroll card + in-form self/gift toggle),
> the **"add a note" field is dropped**, and the **PDP gift REFINE link is in**
> (admin #23 deferred it).

---

## Locked decisions (PO, 2026-10-29) — the spec builds to these

1. **Naming.** The shopper-facing default title is **"Shopping Guide"** (replaces
   the internal "Product Finder"). Default `pf.title` / eyebrow = "Shopping Guide",
   merchant-overridable. (Resolves Q-A8.)
2. **Step counter.** Count **only the required steps** per mode: Audience / Budget /
   Style show **"1 / 3 … 3 / 3"** in **both** self and gift. **Step 4 (gift
   details) is OPTIONAL** — it shows an **"OPTIONAL" eyebrow, no counter number**.
   (Resolves Q-A1; fixes the PNGs' inconsistent 1/2 vs 1/3.)
3. **Audience is catalog-derived, dynamic.** Audience = the **primary categories /
   genders the merchant actually sells**, emitted by the backend with a presence
   floor — **not** a fixed Women/Men/Girl/Boy/Baby set. Only audiences present in
   the catalog appear. The admin shows them **read-only**. (Resolves Q-A10, Q-WZ2;
   reframes N1.)
3b. **Collection (PLP) skips Step 1 entirely.** The collection context
   (**gender + category**, e.g. Men → Shirts) is preserved as the shopper's intent:
   it **pre-fills the audience**, the guide **starts at Budget**, and
   **recommendations filter/rank on that gender + category context**. (Resolves the
   PLP pre-fill path; see §8 contract.)
4. **Relationship list is neutral + editable.** Default **Partner / Parent /
   Sibling / Friend / Child / Colleague** (no female skew). **Occasion** stays
   **Birthday / Christmas / Anniversary / Just because**, editable. (Resolves Q-A6.)
5. **PDP similars default 6, range 4–8.** The discreet **"Buying this for someone?
   REFINE"** gift link is **IN** (toggle default on). (Resolves Q-A4.)
6. **Style "pick up to 2"** is fixed at **2** (not merchant-configurable at MVP).
   **Per-style tile image** = pick-from-catalog by default, optional upload.
   (Resolves Q-A5, Q-A7.) **Note field removed; budget slider + occasion re-added**
   confirmed as the new direction (Q-A9).

---

## 0. TL;DR for the PO

- **The widget changed shape, so the admin must too.** The form is now a
  **5-screen wizard** (Audience+gift → Budget slider → Style tiles → Gift details
  (optional) → Results), and the four hooks each carry **distinct** copy/options
  now (landing/PLP are "FOR ME / FOR SOMEONE ELSE" links; floating is an icon-only
  side tab; PDP is similars + a "REFINE" gift link). The flat `pf.*` string list
  from #23 does not map cleanly onto this.
- **Keep ONE Configuration page** (do not re-litigate the one-tab win), but
  **re-organize it to mirror the shopper journey**: *Appearance → Entry points
  (the 4 hooks) → Shopping Guide (the wizard, one card per step) → Results*. Each
  hook and each step is a **self-contained card** (enable + options + copy +
  "Preview this"), collapsed by default with a one-line summary and a "Using
  defaults" badge — so a merchant can ship **without touching anything**, and
  power users expand only what they want. This is the "easier to use" win: config
  is grouped the way the shopper experiences it, not as one long string table.
- **Reuse the plumbing we have:** the `pf.*` editable-strings system (extended
  with new keys, grouped per step), the 8 theme tokens + `font_scale`, the 4
  placements, and the existing Styles card.
- **Flag NEW config the backend doesn't model yet** (each a backend follow-up):
  **per-style tile images**, **budget-slider range (min/max/step/currency + "any
  budget")**, **configurable relationship list**, **configurable occasion list**,
  the **audience taxonomy change (Kids → GIRL/BOY split)**, **multi-select "pick
  up to 2"** on style, and the **results editable-chips** model.
- **Preview must become step-aware and hook-aware**, not just placement-aware:
  a surface switcher (Floating / Landing / Collection / PDP) **plus** a step
  switcher for the wizard (Audience / Budget / Style / Gift / Results).

---

## 1. Widget delta inventory (what the admin must now configure)

This is the scope anchor. Everything in §2–§3 exists to configure these.

### 1.1 The 4 hooks (entry points)

| Hook | What it renders (from the PNGs) | What the merchant configures |
|---|---|---|
| **Floating side tab** (`floating_button_hook.png`) | Icon-only tab flush to the right edge, **44×56** on mobile, persists across pages. **Desktop reveals the label on hover** ("HELP ME CHOOSE" with a gift icon). | Enable, trigger icon, **hover/expanded label** text, position (left/right), style tokens. The icon-only vs label behaviour is the shipped side-tab-label semantics (unset→default, explicit-empty→icon-only). |
| **Landing block** (`landing_hook.png`) | Inline block on any page. Eyebrow "SHOPPING GUIDE", headline "Not sure what to choose?", subline "Tell us who it's for. Three questions, thirty seconds.", two underlined links **FOR ME / FOR SOMEONE ELSE**. | Enable, eyebrow, headline, subline, the two link labels. The link chosen sets the gift intent and **enters the wizard** (FOR SOMEONE ELSE → gift on). |
| **Collection / PLP** (`collection_hook.png`) | Inline band in the collection header. Category breadcrumb ("WOMAN / COATS"), count ("312 items"), headline "312 coats. Who is it for?", subline "We keep your filters and show the ones worth a look.", **FOR ME / FOR SOMEONE ELSE** links on the right. | Enable, headline pattern (uses the live count + category), subline, link labels. **Audience is pre-filled from the collection** → the wizard **skips the audience question** (§1.3). |
| **PDP** (`pdp_hook.png`) | Below the buy box: "PICKED FOR YOU" similars (grid, title + price; **default 6, range 4–8** — the mock shows 3 for space), and a discreet **"Buying this for someone? REFINE"** gift link. **No wizard by default.** | Enable, similars heading ("PICKED FOR YOU"), similars **count (default 6, 4–8)**, the gift-link prompt + **REFINE** label (toggle default on). Tapping REFINE opens the wizard in **gift** mode. |

### 1.2 The Shopping Guide wizard (the form — now a 5-screen stepper)

Header on every step: centred eyebrow **"SHOPPING GUIDE"**, **X** close (top-right),
**back chevron** (top-left, from step 2 on). A **step counter** ("1 / 3") and an
**accumulating facet breadcrumb** of chips (e.g. `GIFT  WOMEN  UP TO €120  TAILORED`)
sit under the header — these chips become the **editable chips** on Results.

| Step | PNG | Control | Rules / copy | Self vs gift |
|---|---|---|---|---|
| **1 · Audience + gift** | `form_step1_gift_enable.png` | **Single-select** list of the **catalog-derived audiences** (the primary categories/genders the merchant sells — the mock shows Women/Men/Girl/Boy/Baby, but the real list is dynamic per catalog), checkmark on selected, + an **"It's a gift"** toggle ("We'll ask about them, not you"). | Heading "Who are we dressing today?". Counter **"1 / 3"**. **SKIP** (→ no audience filter) + **NEXT**. | The gift toggle sets intent. On landing/PLP the FOR ME / FOR SOMEONE ELSE link pre-sets it. **PLP skips this step** (audience pre-filled from the collection). |
| **2 · Budget** | `form_step2.png`, `form_step2_for_me.png` | **SLIDER** — big current value "€ 120", "Drag to set your budget", track **€40 → €350+**. | Heading "Up to how much for them?" (gift) / "Up to how much?" (self). **ANY BUDGET** (skip) + **NEXT**. | Heading copy differs by mode. |
| **3 · Style** | `form_step3.png`, `form_step3_for_me.png`, `formt_step3_form_me_desktop.png` | **Image tiles**, **multi-select max 2**: EVERYDAY / TAILORED / EVENING / WEEKEND / SPORT / OCCASION. Selected = black border + checkmark. | Heading "Which is more her?" (gift) / "Which is more you?" (self). "Pick up to two. Not sure? Skip it." **NONE OF THESE** + **NEXT/SEE PICKS**. Desktop = side panel over the storefront, tiles in a 3-up grid. | Heading copy differs by mode. |
| **4 · Gift details** (GIFT only, OPTIONAL) | `form_step4.png` | Two pill groups: **THEY ARE MY** relationship (default neutral: **Partner / Parent / Sibling / Friend / Child / Colleague**) + **OCCASION** (Birthday / Christmas / Anniversary / Just because). Single-select each (selected = filled). | **"OPTIONAL" eyebrow — no counter number**, heading "A little more about them?". **SKIP** + **SEE PICKS**. | **Only shown in gift mode.** Self skips straight to Results after step 3. |
| **Results** | `form_results_mobile.png` | **Editable facet chips** (WOMEN · UP TO €120 · EVERYDAY · EVENING — tap a chip to jump back and change it) + **product grid 2-up** with title, price, **ADD**. | Heading "Our picks for you", subline "8 pieces · tap a chip to change". | Same for both; gift shows the gift facets. |

### 1.3 Path differences (gift vs self, PLP pre-fill)

- **Self path:** Audience → Budget → Style → **Results** (4 screens; step 4 skipped).
  Counter shows **"1 / 3 … 3 / 3"**.
- **Gift path:** Audience(+gift on) → Budget → Style → **Gift details (OPTIONAL)** →
  Results. The **counter is the same "1 / 3 … 3 / 3"** across the three required
  steps in both modes; **step 4 shows an "OPTIONAL" eyebrow and no counter number**
  (locked decision 2 — this fixes the PNGs' inconsistent 1/2 vs 1/3).
- **PLP skips Step 1 entirely (locked decision 3b):** the collection supplies the
  **gender + category** context (e.g. Men → Shirts). That context (a) pre-fills the
  audience, (b) starts the wizard **at Budget** (counter still "1 / 3" relative to
  the three required steps), and (c) is passed to the backend so
  **recommendations filter/rank on gender + category** — not just the audience
  chip. The FOR ME / FOR SOMEONE ELSE link sets gift intent. The breadcrumb shows
  the pre-filled audience/category chip (overridable on Results). See §8 for the
  context contract.
- **Dropped:** the **"+ Add a note" / free-text** field is **gone** from every
  path (reverses the simplified-spec note). No interests/note config in the admin.
- **Two ways to set gift intent** now coexist — the hook link (landing/PLP) and
  the step-1 toggle (floating). Confirm this is intended (Q-A2).

---

## 2. Admin configuration model

Organized by surface. For each item: **[REUSE]** = existing plumbing covers it;
**[NEW]** = config the backend doesn't model yet (a backend follow-up, listed in
§2.5). Copy strings all flow through the **`pf.*` editable-strings system** unless
noted; the new keys are proposed per step.

### 2.1 Global appearance (unchanged)

- **8 theme tokens** (accent, background, text, radius, spacing, fonts, shadow) —
  **[REUSE]** drive the editorial look; selected-state fill + primary CTA (the
  black ADD/NEXT buttons and selected tiles) resolve from `--wg-accent`.
- **Text size** scale slider → `--wg-font-scale` (0.85×–1.25×, default 1.0×) —
  **[REUSE]**, shipped in #23. Unchanged.

### 2.2 Per-hook config

| Hook | Config | Plumbing |
|---|---|---|
| **Floating** | Enable; position (left/right); trigger **icon**; **hover/expanded label** (`pf.floating.label`, e.g. "HELP ME CHOOSE"; explicit-empty → icon-only; unset → default). | **[REUSE]** placement + side-tab-label semantics + `pf.*`. |
| **Landing** | Enable; eyebrow (`pf.landing.eyebrow` = "SHOPPING GUIDE"); headline (`pf.landing.headline`); subline (`pf.landing.subline`); link labels (`pf.choice.self` / `pf.choice.gift` — reused as "FOR ME" / "FOR SOMEONE ELSE"). | **[REUSE]** placement + `pf.*` (choice strings reused as link labels). |
| **Collection/PLP** | Enable; headline pattern (`pf.plp.headline`, interpolates count + category, e.g. "{count} {category}. Who is it for?"); subline (`pf.plp.subline`); link labels (shared with landing). | **[REUSE]** copy; **[NEW]** the count/category **interpolation tokens** in the string (templated string support) — flag Q-A3. |
| **PDP** | Enable; similars heading (`pf.pdp.similarsHeading` = "PICKED FOR YOU"); **similars count** (4–8, default per mock = 3 shown; confirm Q-A4); gift-link prompt (`pf.pdp.giftPrompt` = "Buying this for someone?") + **REFINE** label (`pf.pdp.refine`). | **[REUSE]** similars enable + count; **[REUSE]** `pf.*` copy. The REFINE gift link is **new to the admin** (deferred in #23) but needs no new backend model beyond a toggle + its copy. |

### 2.3 Per-step (wizard) config

| Step | Config | Plumbing |
|---|---|---|
| **1 · Audience + gift** | Step heading (`pf.step.audience.heading`); audience **options source = catalog-derived, dynamic** (read-only note, not an editable list — the primary categories/genders the merchant actually sells, with a presence floor); gift-toggle label + helper (`pf.gift.toggleLabel` / `pf.gift.toggleHelper`); SKIP label (`pf.common.skip`); NEXT label (`pf.common.next`). | **[NEW]** **catalog-derived audience contract** — dynamic list + presence floor (§2.5 N1, §8). |
| **2 · Budget** | Step heading per mode (`pf.step.budget.heading.self` / `.gift`); **slider range: min / max / step / currency**; "any budget" label (`pf.budget.anyLabel`); the big-value format (e.g. "€ {n}", "€ {max}+"). | **[NEW]** budget-slider **range model** (§2.5 N2) — replaces the removed budget tiers; copy via `pf.*`. |
| **3 · Style** | Step heading per mode (`pf.step.style.heading.self` / `.gift`); **which styles are enabled** + per-style **label** + per-style **tile image**; the **"pick up to 2"** rule (max-select, confirm Q-A5 whether the "2" is configurable); "pick up to two…" helper (`pf.style.helper`); NONE OF THESE label (`pf.style.none`). | **[REUSE]** the Styles card as the base; **[NEW]** per-style **tile image** (§2.5 N3) and **multi-select max-2** (§2.5 N4). |
| **4 · Gift details** | Step eyebrow "OPTIONAL" (`pf.step.gift.eyebrow`, **no counter number**); heading (`pf.step.gift.heading`); **relationship list** (default neutral: Partner / Parent / Sibling / Friend / Child / Colleague) under "THEY ARE MY" (`pf.gift.relationshipHeading`); **occasion list** (Birthday / Christmas / Anniversary / Just because) under "OCCASION" (`pf.gift.occasionHeading`); SKIP + SEE PICKS labels. | **[NEW]** configurable **relationship list** (§2.5 N5) and **occasion list** (§2.5 N6) — both were fixed/dropped before. |
| **Results** | Heading (`pf.results.heading` = "Our picks for you"); subline w/ count (`pf.results.subline` = "{n} pieces · tap a chip to change"); **editable-chip** behaviour (tap-to-edit returns to the owning step); ADD label (`pf.results.add`). | **[REUSE]** copy; **[NEW]** the **editable-chips** interaction model + the chip↔step mapping (§2.5 N7). |

### 2.4 What is REMOVED from the admin vs #23

- The **"+ Add a note" / free-text** field config (note is dropped from the widget).
- Any **single-scroll form / in-form self-gift toggle** framing — replaced by the
  per-step cards + the hook-level FOR ME / FOR SOMEONE ELSE links.
- The flat, ungrouped `pf.*` string table — replaced by **per-step / per-hook
  grouped editors** (the strings persist; only the admin grouping changes).

### 2.5 NEW backend config (flag to `backend-engineer` + `recommendations-specialist`)

> These are config models the current backend does not have. The admin IA below
> assumes them; if any is deferred, the corresponding admin card falls back to a
> read-only "coming soon" note rather than blocking the rest.

| # | New config | Why it's new | Owner |
|---|---|---|---|
| **N1** | **Catalog-derived audience list** (dynamic): the backend emits the **primary categories/genders the merchant actually sells**, with a **presence floor**. Not a fixed Women/Men/Girl/Boy/Baby set — whatever the catalog contains. Admin shows read-only. | Audience must reflect each merchant's real catalog; a fashion store may have Women/Men only, a kids store Girl/Boy/Baby, etc. | backend (derivation) + recs (presence floor) |
| **N2** | **Budget-slider range model**: `min`, `max`, `step`, `currency`, and an "any budget" sentinel. | Budget was tiers, then removed; the slider needs a continuous range with sensible per-tenant defaults (e.g. derived from catalog price distribution). | backend (+ recs for sensible defaults) |
| **N3** | **Per-style tile image** (one representative image per enabled style). | The editorial tiles show a photo per style; the Styles card stored anchors, not a display image. Needs an image field (upload or pick-from-catalog). | backend + frontend (asset handling) |
| **N4** | **Style multi-select "pick up to 2"** (was single-select `style_token`). | Editorial design allows two styles. Payload becomes a small array; ranker must accept 0–2 style tokens. | backend + recs |
| **N5** | **Configurable relationship list** (labels + order): default **neutral** — Partner / Parent / Sibling / Friend / Child / Colleague (no female skew). | Richer than the old fixed enum; merchant may edit/trim within a cap. | backend |
| **N6** | **Configurable occasion list** (labels + order): BIRTHDAY/CHRISTMAS/ANNIVERSARY/JUST BECAUSE. | Occasion was dropped; it's back as an optional gift field — needs a (re)modelled, editable list. | backend + recs |
| **N7** | **Results editable-chips model**: each applied facet (audience/budget/style×2/relationship/occasion) renders as a chip that maps back to its owning step for editing, and re-queries on change. | New interaction; needs the facet→step mapping and a "re-rank with changed facet" path. | frontend + recs |
| **N8** | **Templated hook headline** for PLP ("{count} {category}. Who is it for?"). | Needs interpolation of live collection count + category into a `pf.*` string. | backend (string templating) + frontend |

---

## 3. A more usable admin IA (the PO's "easier to use" goal)

### 3.1 Critique of the shipped #23 one-page Configuration

#23 is **one Configuration page, three flat sections (Appearance / Hooks / Form)**,
with a **text-size slider** and an **editable-strings editor over the `pf.*` keys**.
Against the richer editorial surface it strains in four ways:

1. **The flat `pf.*` string editor doesn't scale.** The wizard adds ~5 step
   headings, per-mode variants, step-skip labels, budget/any-budget copy, style
   helper + none-of-these, gift eyebrow/headings, two editable **lists**
   (relationship, occasion), results heading/subline/ADD, plus per-hook copy. As
   one undifferentiated list this is a wall of keys a merchant can't navigate.
2. **"Hooks" now hides real per-hook differences.** In #23 "Hooks" was mostly the
   floating button + enable toggles. Now **each hook has distinct copy and
   options** (landing links + headline/subline; PLP templated headline; PDP
   similars heading + count + REFINE). They need **per-hook cards**, not one lump.
3. **The form is no longer "a form" — it's a 5-screen journey.** A single "Form"
   section can't express step order, per-step options, per-mode copy, or the
   self/gift branch. The natural unit of config is now **the step**.
4. **The preview is placement-aware but not step-aware.** A merchant editing
   "Style step, gift heading" has no way to see *that screen*. The preview must
   gain a **step switcher**.

### 3.2 Proposed structure — one page, journey-ordered cards

Keep **one Configuration page** (do not reverse the one-tab consolidation — that
churned once already). Re-order it to **follow the shopper journey**, with a
sticky jump-bar and one shared Save bar:

```
Widget → Configuration
┌──────────────────────────────────────────────────────────────────────┐
│ [ Appearance ] [ Entry points ] [ Shopping Guide ] [ Results ] [ Advanced ]  ← jump-bar
│                                                                        │
│ ┌────────────────────────────────────┐  ┌──────────────────────────┐  │
│ │ 1. APPEARANCE (global)             │  │  LIVE PREVIEW (sticky)    │  │
│ │    Presets · Colour · Shape ·      │  │  Surface: [Float][Land]   │  │
│ │    Typography · Text size · Effects│  │          [Collection][PDP]│  │
│ │                                    │  │  Step:   [1][2][3][4][✓]  │  │
│ │ 2. ENTRY POINTS (the 4 hooks)      │  │  Mode:   ( ) Self (•) Gift │  │
│ │    ▸ Floating side tab   [on ·…]   │  ├──────────────────────────┤  │
│ │    ▸ Landing block       [on ·…]   │  │  real widget render of    │  │
│ │    ▸ Collection (PLP)    [on ·…]   │  │  the selected surface +   │  │
│ │    ▸ PDP similars+refine [on ·…]   │  │  step + mode, live tokens │  │
│ │                                    │  │  + strings applied        │  │
│ │ 3. SHOPPING GUIDE (wizard, by step)│  │                          │  │
│ │    ▸ Step 1 · Audience + gift      │  └──────────────────────────┘  │
│ │    ▸ Step 2 · Budget (slider)      │                               │
│ │    ▸ Step 3 · Style (tiles)        │                               │
│ │    ▸ Step 4 · Gift details (opt.)  │                               │
│ │ 4. RESULTS                         │                               │
│ │    ▸ Picks heading + chips + ADD   │                               │
│ │ 5. ADVANCED                        │                               │
│ │    ▸ All strings (raw pf.* table)  │                               │
│ └────────────────────────────────────┘                               │
│                                   [ Save bar: Discard · Save ]         │
└──────────────────────────────────────────────────────────────────────┘
```

**Why this is easier to use**

- **Journey order = the merchant's mental model.** They set the brand look once,
  decide **where** the widget appears, then tune **what the guide asks** step by
  step, then the **results**. Increasing specificity, matching the shopper's path.
- **One card per hook / per step.** Each card holds *everything* for that surface:
  enable + options + copy (as **structured fields**, not raw keys) + a **"Preview
  this"** button that jumps the live preview to that surface/step/mode. No hunting
  through a string table.
- **Progressive disclosure + sensible defaults.** Cards are **collapsed by
  default**, each showing a one-line summary and a **"Using defaults"** badge. A
  merchant can **ship without opening a single card**. Expanding a card reveals
  its structured editor; a per-card **Reset** returns it to defaults. A top-level
  **"Ready to ship — all defaults"** indicator reassures the zero-config merchant.
- **Structured editors for the new lists.** Relationship and occasion are **list
  editors** (reorder, rename, show/hide, add/remove within a cap), not free-text
  keys. Style is a **tile editor** (toggle each style, set its label + image).
  Budget is a **range editor** (min/max/step/currency + any-budget copy). These
  are far more usable than editing `pf.gift.relationship.3`.
- **Raw-strings escape hatch stays.** The flat `pf.*` table from #23 lives under
  **Advanced** for anyone who wants direct access or a string not yet surfaced —
  so nothing regresses, but it's no longer the primary surface.

### 3.3 Card contents (layout-ready)

**Entry points — one card each (collapsed → summary; expanded → fields):**
- *Floating side tab:* Enable toggle · Position (left/right) · Icon (default gift)
  · Hover label (text; "Leave empty to show only the icon" helper; shows **"Icon
  only"** state when cleared) · Preview this.
- *Landing block:* Enable · Eyebrow · Headline · Subline · FOR ME label · FOR
  SOMEONE ELSE label · Preview this.
- *Collection (PLP):* Enable · Headline template (with `{count}`/`{category}`
  chips to insert) · Subline · link labels (shared) · note: "Audience is filled
  from the collection; the guide skips the audience step here." · Preview this.
- *PDP:* Enable similars · Similars heading · Similars count (stepper 4–8) ·
  Gift-link toggle · Gift prompt · REFINE label · Preview this.

**Shopping Guide — one card per step:**
- *Step 1 Audience + gift:* Heading · read-only "Audiences come from your catalog —
  only the categories you actually sell appear here" note (renders the live derived
  list) · Gift toggle label + helper · SKIP/NEXT labels · Preview this (mode switch
  Self/Gift in the preview).
- *Step 2 Budget:* Heading (Self) · Heading (Gift) · **Range editor** (min, max,
  step, currency) · Any-budget label · Preview this.
- *Step 3 Style:* Heading (Self) · Heading (Gift) · **Tile editor** (per style:
  on/off, label, image) · max-select = 2 (Q-A5) · helper · NONE OF THESE label ·
  Preview this.
- *Step 4 Gift details:* Eyebrow · Heading · **Relationship list editor** ·
  **Occasion list editor** · SKIP/SEE PICKS labels · Preview this (gift only).

**Results — one card:** Picks heading · subline (with `{n}`) · ADD label · a
read-only explainer of the editable-chips behaviour · Preview this.

### 3.4 Preview (step-aware + hook-aware)

Extend the #23 preview. Controls above the canvas:
- **Surface switcher** (radio): Floating / Landing / Collection / PDP.
- **Step switcher** (segmented, only when a form-rendering surface is active):
  Audience / Budget / Style / Gift / Results.
- **Mode switch** (Self / Gift) so per-mode copy and the gift-only step 4 are
  visible.
- The active card **biases** the switchers: opening the PDP card selects PDP;
  opening "Step 2 Budget" selects the Budget step; the merchant can still override.
- **Live, ≤ 500 ms, sample data only** (no tenant rec path, no network/auth/
  analytics), deterministic; shopper-facing controls inside the preview are
  inert/non-tabbable. Holds at 320 px and 200% zoom.
- **Dependency (same as #23):** the **widget-preview bundle must be updated to the
  new editorial Shopping Guide layout** (wizard steps + the 4 hooks) with a
  per-surface/per-step render API. Critical path for the preview; **flag to
  `frontend-engineer`** (widget-side).

---

## 4. Acceptance criteria

> Structural and testable. Widget invariants (holdout symmetry, ≤ 50 KB bundle,
> 44 px targets, token-driven theming) are unchanged and out of scope here.

**IA / structure**
- [ ] **AC-IA1** — `Widget → Configuration` is **one page** ordered **Appearance →
  Entry points → Shopping Guide → Results → Advanced**, with a sticky jump-bar and
  one shared Save bar (Discard reverts the whole draft; Save persists it).
- [ ] **AC-IA2** — Every hook and every wizard step is a **collapsible card** with
  a one-line summary, a **"Using defaults"** badge when untouched, a per-card
  **Reset**, and a **"Preview this"** control. A merchant can **save and ship with
  all cards collapsed/untouched** (defaults only).
- [ ] **AC-IA3** — The flat `pf.*` string table is **demoted to the Advanced
  section**; all primary copy is edited through **structured, per-surface/per-step
  fields** (not raw keys).

**Hooks**
- [ ] **AC-HK1** — Each of the four hooks has its own card configuring exactly the
  items in §2.2: floating (position, icon, hover label w/ icon-only state), landing
  (eyebrow/headline/subline + two link labels), collection (templated headline +
  subline + links + the audience-pre-fill note), PDP (similars heading + count +
  gift prompt + REFINE label).
- [ ] **AC-HK2** — The **PDP REFINE gift link** is configurable (toggle + prompt +
  label) and, when on, opens the wizard in **gift** mode (admin copy only; behaviour
  is widget-side).

**Wizard**
- [ ] **AC-WZ1** — Section 3 has **one card per step** (Audience+gift, Budget,
  Style, Gift details) with per-mode heading fields where the mocks differ by mode,
  plus the Results card.
- [ ] **AC-WZ2** — **Audience is read-only** (catalog-derived; shown as Women /
  Men / Girl / Boy / Baby) — no editable audience list.
- [ ] **AC-WZ3** — **Budget** is a **range editor** (min/max/step/currency +
  any-budget label); **Style** is a **tile editor** (per style: on/off, label,
  image) with the **max-2** rule surfaced; **Relationship** and **Occasion** are
  **list editors** (reorder/rename/show-hide/add-remove within a cap).
- [ ] **AC-WZ4** — The **"+ Add a note" / free-text** config is **absent** (the
  field is dropped from the widget).

**Preview**
- [ ] **AC-PV1** — The live preview renders the **real editorial widget** per
  surface via a **surface switcher** (Floating/Landing/Collection/PDP) **and**, for
  form surfaces, a **step switcher** (Audience/Budget/Style/Gift/Results) and a
  **Self/Gift mode** switch; the active card biases the selection.
- [ ] **AC-PV2** — Token changes **and** every structured-field edit repaint the
  relevant preview within **≤ 500 ms**, no save; sample data only; deterministic;
  inert shopper controls; holds at 320 px / 200% zoom.
- [ ] **AC-PV3** — The preview depends on the **widget-preview bundle updated to
  the editorial layout** (widget-side) — flagged, sequenced before the admin
  preview work.

**New config surfaced (each falls back to a read-only "coming soon" if deferred)**
- [ ] **AC-NC1** — The admin surfaces editors for N2 (budget range), N3 (style tile
  images), N5 (relationship list, neutral default), N6 (occasion list); and
  read-only notes for N1 (catalog-derived audiences) and N4 (fixed max-2). Each
  unbuilt backend model degrades to a disabled card with a note, never a broken
  control.

---

## 5. Reuse vs new — quick reference

| Area | Reuse (existing plumbing) | New (backend follow-up) |
|---|---|---|
| Appearance | 8 theme tokens, `--wg-font-scale` | — |
| Hook copy | `pf.*` strings, placement set, side-tab-label semantics | PLP templated headline (N8) |
| Audience | catalog-derived audiences | **Girl/Boy split (N1)** |
| Budget | `pf.*` copy | **slider range model (N2)** |
| Style | the Styles card base | **per-tile image (N3)**, **max-2 multi-select (N4)** |
| Gift details | `pf.*` copy | **relationship list (N5)**, **occasion list (N6)** |
| Results | `pf.*` copy | **editable-chips model (N7)** |

---

## 6. Open questions — ALL RESOLVED (PO, 2026-10-29)

| # | Question | Resolution (locked) |
|---|---|---|
| **Q-A1** | Step **counter** denominator per mode. | **Required steps only**: Audience/Budget/Style = "1/3…3/3" in **both** modes; step 4 shows **"OPTIONAL" eyebrow, no number**. (Locked decision 2.) |
| **Q-A2** | Gift intent set **two ways** (hook link + step-1 toggle). | **Keep both**; the hook link **pre-sets** the toggle; the toggle is the single source of truth. |
| **Q-A3 / N8** | PLP **templated** headline editable? | **Editable template** with insertable `{count}`/`{category}` chips; fixed default. Backend adds string templating. |
| **Q-A4** | PDP similars count + default. | **Range 4–8, default 6.** REFINE link **in** (default on). (Locked decision 5.) |
| **Q-A5 / N4** | Style "pick up to 2" fixed or configurable? | **Fixed at 2** at MVP. (Locked decision 6.) |
| **Q-A6 / N5-N6** | Relationship + occasion editable, and the gendered skew? | **List editors** (reorder/rename/show-hide/add-remove within a cap). Relationship default is **neutral**: Partner/Parent/Sibling/Friend/Child/Colleague. Occasion: Birthday/Christmas/Anniversary/Just because. (Locked decision 4.) |
| **Q-A7 / N3** | Per-style tile image: upload or pick-from-catalog? | **Pick-from-catalog** default, optional upload. (Locked decision 6.) |
| **Q-A8** | Rename to "Shopping Guide"? | **Yes** — default title/eyebrow "Shopping Guide", overridable. (Locked decision 1.) |
| **Q-A9** | Accept the five reversals? | **Yes** — slider back, occasion back, step wizard, note dropped, PDP REFINE in. |
| **Q-A10 / N1** | Audience taxonomy. | **Catalog-derived, dynamic** list with presence floor — not a fixed Girl/Boy split. Admin read-only. (Locked decision 3.) |

---

## 7. Hand-off

- **`frontend-engineer`** — design the admin UI on top of §3 (journey-ordered
  cards, structured editors, step-aware preview); and the **widget-preview bundle
  update** to the editorial layout (critical path for AC-PV3).
- **`backend-engineer`** — the NEW config models N1–N8 (§2.5): Girl/Boy audience
  derivation, budget-slider range, per-style image, style max-2 payload,
  relationship + occasion lists, results-chips/facet model, PLP templated headline;
  plus the new `pf.*` keys as overridable defaults.
- **`recommendations-specialist`** — audience presence floors (which catalog-derived
  audiences qualify to show), budget slider sensible defaults from price
  distribution, 0–2 style tokens in the ranker, occasion back as an optional signal,
  the **PLP gender+category context** consumption (filter/rank), and the
  re-rank-on-chip-change path.
- **PO** — **VALIDATED (2026-10-29).** Direction locked; build to §8 contracts and
  §9 phasing, human-gated per PR.
- **Two called-out reductions/additions:** note field **removed**; occasion and
  budget slider **re-added**. Both deliberate, flagged, not silent.

---

## 8. Backend contract requirements (for `backend-engineer`)

> Concrete, buildable contracts for N1–N8. Each states the **field/endpoint shape**,
> how it **fits the existing plumbing** (the `pf.*` editable-strings store behind
> `/admin/v1/widget/strings`; the theme tokens incl. `font_scale` behind the
> theme-tokens endpoint; the placement set; the session bootstrap + `/recommendations`
> contract), and whether it **needs a migration**. `backend-engineer` owns the exact
> wire naming; these are the shapes the widget + admin assume. Reuse the
> existing **defaults-layer pattern** (ship a server default, let the merchant
> override) everywhere a value is editable.

### 8.0 What is pure REUSE (no new contract)

- **All copy** (step headings per mode, hook headlines/sublines, link labels, gift
  toggle label/helper, SKIP/NEXT/NONE/ADD, results heading/subline, REFINE prompt)
  → **existing `pf.*` string store**. This change only **adds keys** (and their
  server defaults) and **re-groups them in the admin**; the store, the
  `/admin/v1/widget/strings` read/write, and the per-language model are unchanged.
  New default `pf.title` = **"Shopping Guide"**.
- **Appearance** → the 8 theme tokens + `--wg-font-scale`, unchanged.
- **Placement set** → `{floating, landing, plp, pdp}`, unchanged; no new placement.

### 8.1 N1 — Catalog-derived audience list (dynamic + presence floor)

- **Contract:** the **session bootstrap** carries, per tenant, the **present
  audiences** derived from the catalog — e.g.
  `config.guide.audiences: [{ id, label, genderContext, ageContext }]`, ordered,
  already filtered by a **presence floor** (an audience appears only if ≥ N
  in-stock products map to it; N owned by `recommendations-specialist`). The widget
  renders exactly this list in Step 1; the admin reads it **read-only**.
- **Fits plumbing:** extends the existing bootstrap `config` object (same place the
  present-audience work was already scoped); labels remain overridable via `pf.*`
  if a merchant wants to rename a derived audience (optional, Phase 3+).
- **Migration:** none for storage (derived at read time from the catalog + an
  aggregate/cache); refresh on catalog sync. No schema change required.

### 8.2 N2 — Budget-slider range model

- **Contract:** per tenant in bootstrap `config.guide.budget: { min, max, step,
  currency, anyBudget: true }`, with a **server default derived from the catalog
  price distribution** (e.g. min ≈ floor, max ≈ a high percentile, "+" above max).
  Merchant override stored on `widget_config` (admin editor writes it). The intent
  payload on `/recommendations` keeps sending `budgetMin`/`budgetMax` (unchanged
  hard filter, per the 2026-10-08 budget-filter decision); "any budget" sends both
  null.
- **Fits plumbing:** reuses the existing budget hard-filter at pgvector retrieval;
  only the **input control** changes (tiers → slider range). Copy via `pf.*`.
- **Migration:** small — a nullable `budget_range` JSONB (or 4 columns) on
  `widget_config` for the merchant override; null → derived default. Additive.

### 8.3 N3 — Per-style tile image

- **Contract:** extend the existing **Styles** config so each enabled style carries
  a **display image**: `styles[i].imageRef` = either a catalog product image id
  (**pick-from-catalog**, default) or an uploaded asset URL (optional). Bootstrap
  emits the resolved image URL per enabled style for the widget tiles.
- **Fits plumbing:** builds on the shipped Styles card / style-token model; adds one
  field per style. The anchor-based `style_token` on `/recommendations` is
  unchanged.
- **Migration:** additive column on the style config row (`image_ref`) or a JSONB
  field on the existing styles structure. Upload path reuses whatever asset handling
  the admin already has (or defer upload to a later phase; pick-from-catalog needs
  no storage).

### 8.4 N4 — Style multi-select "pick up to 2"

- **Contract:** the intent payload's `style_token` becomes **`styleTokens: string[]`
  (0–2 entries)** on `/recommendations` (keep a back-compat single-value read).
  Fixed max 2 (not configurable). Cache signature includes the sorted token array.
- **Fits plumbing:** extends the already-approved `style_token` field to an array;
  ranker accepts 0–2 (recs owns how two tokens combine). Same signature composer,
  array-aware.
- **Migration:** none (payload + cache-key shape only); confirm the rec DTO tolerates
  the array.

### 8.5 N5 / N6 — Relationship + occasion lists (editable)

- **Contract:** per tenant in bootstrap `config.guide.relationships: [{id,label}]`
  (neutral default: Partner/Parent/Sibling/Friend/Child/Colleague) and
  `config.guide.occasions: [{id,label}]` (Birthday/Christmas/Anniversary/Just
  because). Merchant override (reorder/rename/show-hide/add-remove within a cap ~6)
  stored on `widget_config`. The intent payload sends the selected `relationshipId`
  and `occasionId` (occasion is **optional**, gift-only).
- **Fits plumbing:** labels can live in `pf.*` while the **list membership/order**
  lives in `widget_config` (a list is structural, not a single string). Recs consume
  `relationship` (already modelled) and **occasion re-added as an optional signal**.
- **Migration:** additive JSONB (`relationship_list`, `occasion_list`) on
  `widget_config`; null → server default. `/recommendations` DTO re-admits the
  optional `occasion` field (it was dropped) — confirm graceful absence.

### 8.6 N7 — Results editable-chips / facet model

- **Contract:** the Results screen renders a chip per applied facet
  (audience/category, budget, style×0–2, relationship, occasion). Tapping a chip
  returns to the **owning step**; changing it **re-queries** `/recommendations` with
  the amended intent. No new endpoint — it is a **re-submit of the existing intent
  payload** with one facet changed; the widget holds the facet→step mapping.
- **Fits plumbing:** reuses `/recommendations`; the only backend need is that a
  **partial intent re-submit** is cheap (the 24h response cache keys on the intent
  signature, so changed-facet re-queries hit cache when repeated).
- **Migration:** none.

### 8.7 N8 + PLP context — gender+category collection context

- **Contract (the load-bearing one for decision 3b):**
  1. **Widget → backend:** on the **PLP placement**, the session bootstrap /
     `/recommendations` request carries a **`collectionContext: { gender, category }`**
     (e.g. `{ gender: "men", category: "shirts" }`), resolved by the widget from the
     collection page (handle/metadata) or passed from the theme block. The guide
     **starts at Budget** (Step 1 skipped).
  2. **Backend → recs:** the recommendation pipeline treats `collectionContext` as a
     **hard filter on gender + category** (like the audience hard filter) **and** as a
     ranking signal, in addition to budget/style. This preserves the shopper's
     collection intent rather than discarding it at the audience step.
  3. **PLP templated headline (N8):** a `pf.plp.headline` string supports
     `{count}` + `{category}` interpolation; the widget fills `{count}` from the live
     collection and `{category}` from `collectionContext.category`; backend only
     stores/serves the template string (string templating support in the `pf.*`
     render path).
- **Fits plumbing:** `collectionContext` is a new request field on the existing
  session/rec contract; gender+category map to catalog attributes the ranker already
  filters on. Headline templating extends the `pf.*` render, not the store.
- **Migration:** none for context (request field + existing filters); the templating
  is render-layer.

### 8.8 Contract summary table

| # | Shape | Plumbing | Migration |
|---|---|---|---|
| N1 | bootstrap `config.guide.audiences[]` (derived, floored) | bootstrap config | none (derived) |
| N2 | `config.guide.budget{min,max,step,currency,anyBudget}` + `widget_config` override; payload `budgetMin/Max` unchanged | existing budget hard-filter | small additive |
| N3 | `styles[i].imageRef` (catalog pick / upload) | Styles config | additive column |
| N4 | payload `styleTokens: string[]` (0–2) | style-token → array; cache sig | none |
| N5/N6 | `config.guide.relationships[]` / `occasions[]` + overrides; payload `relationshipId`, optional `occasionId` | `pf.*` labels + `widget_config` lists; recs re-admit occasion | additive JSONB |
| N7 | re-submit intent with one changed facet | existing `/recommendations` + cache | none |
| N8 / PLP | request `collectionContext{gender,category}`; `pf.plp.headline` templating | session/rec request + existing filters | none |

---

## 9. Phased implementation plan (contract-first)

> **Principle:** backend contract → widget consumption → admin configuration, with
> each phase **independently shippable and testable** behind the human deploy gate.
> The **widget form becoming a step wizard is the biggest risk** — Phase 2 lands it
> on a **static/default config** (no new admin, no new backend lists) so the wizard
> shell is proven before any dynamic config rides on it. Every PR targets
> `develop` via a `feature/*` branch; `develop → main` stays a separate human PR.
> Each phase is small enough to review/merge/deploy one at a time.

**Dependency order:** P0 → P1 → P2 → (P3, P4 parallel) → P5 → P6 → P7.

| Phase | Ships | Repos | Depends on | Independently testable? |
|---|---|---|---|---|
| **P0 · String + naming groundwork** | Add the new `pf.*` keys with server defaults (incl. `pf.title`="Shopping Guide"), re-grouped in the admin strings editor; no behaviour change. Rename default title across widget + admin. | backend (defaults) + admin (grouping) | — | Yes — strings render/override; name shows "Shopping Guide". |
| **P1 · Contract stubs (backend)** | Bootstrap `config.guide` skeleton (audiences derived, budget range derived, relationships/occasions defaults, styles image resolve) + `/recommendations` payload accepts `styleTokens[]`, optional `occasionId`, `collectionContext` — all **tolerated, defaulted**, no widget change yet. | backend | P0 | Yes — contract tests on bootstrap + payload tolerance; old widget still works. |
| **P2 · Widget wizard shell** | The **step wizard UX** (header, counter "1/3…3/3", OPTIONAL step 4, back/skip, breadcrumb chips, Results with editable chips + ADD) rendering off **P1's config/defaults** — budget slider, style tiles (multi-select max 2), gift toggle, relationship/occasion pills. Self + gift paths. No admin editing yet (uses defaults). | widget | P1 | Yes — the full shopper flow on default config, on staging. **De-risks the wizard here.** |
| **P3 · Catalog-derived audience + PLP context** | Backend presence-floored audience derivation (N1) + `collectionContext` consumption (hard filter + rank, N8/3b). Widget: Step 1 renders the dynamic list; **PLP skips Step 1** and starts at Budget with gender+category applied. | backend + recs + widget | P2 | Yes — audience list reflects a test catalog; a PLP session filters on gender+category. |
| **P4 · Budget range + style images + style multi-select** | Backend: budget range model (N2), per-style image (N3), `styleTokens[]` ranking (N4). Widget: slider honours tenant range + "any budget"; tiles show per-style images; 0–2 selection to recs. | backend + recs + widget | P2 | Yes — slider range + tile images + 2-style payload verifiable per tenant. |
| **P5 · Relationship/occasion lists + results re-rank** | Backend: editable relationship/occasion lists (N5/N6) + occasion as optional rec signal; re-rank-on-chip-change path/caching (N7). Widget: pills render the configured lists; editable chips re-query. | backend + recs + widget | P3, P4 | Yes — custom lists appear; changing a results chip re-queries. |
| **P6 · Admin: journey-ordered IA + per-surface/step cards** | The admin re-org (§3): Appearance → Entry points → Shopping Guide → Results → Advanced; collapsible cards with "Using defaults" badges, structured editors (budget range, style tile editor, relationship/occasion list editors, hook cards incl. PLP templated headline + PDP REFINE), raw `pf.*` under Advanced. | admin | P1–P5 (configs exist to edit) | Yes — each card writes its config; merchant can ship zero-config. |
| **P7 · Admin: step-aware + hook-aware live preview** | Surface switcher (Floating/Landing/Collection/PDP) + step switcher (Audience/Budget/Style/Gift/Results) + Self/Gift mode, biased by the active card; depends on the **widget-preview bundle updated to the editorial layout** (widget-side, sequence its build before this). | widget (preview bundle) + admin | P2, P6 | Yes — preview repaints per surface/step/mode ≤ 500 ms on sample data. |

**Phasing rationale**

- **P0/P1 are pure contract + defaults** — zero user-visible risk, unblock everyone.
- **P2 proves the wizard** on defaults before dynamic config depends on it — the
  single biggest UX change is isolated and testable on its own.
- **P3/P4 are parallelizable** once the shell exists (audience/PLP vs
  budget/style are independent slices), each shippable alone.
- **P5** needs both the lists (P3-style config pattern) and the facet model (P4
  style multi-select) in place for the Results chips to cover every facet.
- **Admin (P6/P7) comes after the config exists** so every card edits a real,
  already-consumed value — no admin control ships ahead of its effect. P7 is gated
  on the widget-preview bundle, kept last so the preview reflects the finished widget.
- **Fallback discipline:** if any backend model (P3–P5) slips, the corresponding
  admin card degrades to a read-only "coming soon" note (AC-NC1) and the widget
  uses the P1 defaults — no phase hard-blocks shipping the others.
