# Per-placement hooks and niche packs

> **DRAFT — pending PO review.** Written 2026-10-15 by `product-analyst` for the two improvements the PO approved on 2026-10-15. Nothing here is decided until the PO validates it. Open questions are listed in §5, each with an owner.
>
> Linked from `docs/product/spec.md` (Widget — Gift intent hook; Merchant onboarding — Onboarding wizard; Merchant admin — Widget configuration; Roadmap item 1).
>
> **Inputs:**
> - `decisions.md` 2026-10-07 "Per-placement hook copy"
> - `decisions.md` 2026-10-07 "Widget v1: side-tab trigger … progressive disclosure"
> - `decisions.md` 2026-10-07 floating-hook entry
> - `decisions.md` 2026-10-15 "Merchant admin is the single source of truth for widget configuration"
> - M1·2 D1–D9 (2026-10-14) and P1–P3 (2026-10-15)
> - `architecture.md` §9.2 (session bootstrap) and §10.5 (theming)
> - the PO's HTML mockups: `Main`, `HomePill`, `PlpHook` and `PdpHook` in scope; `Question1`, `Question2`, `HomeRoom` and `HomeNeed` as style and tone references only

---

## 1. Scope

**In scope**
1. **Per-placement hook layouts.** The hook is the first screen of the widget: the self / gift choice. It gets a distinct layout for each placement: `home_hero`, `collection_header` (PLP), `pdp_slot`, `gift_finder` and `floating` (collapsed pill, expanded drawer).
2. **Niche packs as starter presets.** The merchant picks "What do you sell?" (Fashion, Home & Decor or General). The pack pre-fills **existing** configuration only:
   - per-placement hook copy;
   - labels and options of the existing intent questions;
   - a suggested theme preset.

**Not in scope**
- New question types, new intent fields, or any change to the recommendation engine.
- Anything after the hook: the intent form's structure, recommendation display and the placebo all stay as specified in `spec.md`. Packs change form **labels and option lists only** (§3.2).
- Looks, influencer posts and a configurable questionnaire (§6, Deferred).

**Baseline this builds on (no change)**

| Area | Current behaviour |
|---|---|
| Hook copy chain (2026-10-15) | 1. `placement_hooks[placement]` → 2. `hook_copy` → 3. backend per-placement default → 4. generic fallback. The backend resolves the chain and returns `copyForPlacement` in the session bootstrap (`POST /widget/v1/session` → `config`). The widget falls back to `copy`, then to its built-in default. |
| Current backend defaults | `home_hero` "Looking for a gift, or something for yourself?"; `pdp_slot` "Thinking about this as a gift? Let's find the right pick."; `collection_header` "Not sure which one? We can match a gift to the person."; `gift_finder` "Tell us about them — we'll find something perfect."; `floating` "Not sure what to get them? Let's narrow it down together." (the `floating` key was planned as a follow-up in the 2026-10-07 entry; see Q-B4) |
| Gift form | Relationship (Partner / Family / Friend / Colleague / Other), occasion, budget bands and interest chips (`spec.md` "Intent capture form"). Visual picker cards, budget range buttons and optional interests, per 2026-10-07 "Widget v1" |
| Self form | Budget and interest chips |
| Events | `widget_shown`, `widget_engaged`, `intent_submitted`, `product_clicked`. Strict schema: unknown fields are rejected |
| Holdout | Server-assigned `isHoldout`. The hook and form render identically in both arms. After intent, holdout sees the neutral collapsed rec slot (D6, P1) |
| Theming | 8 tokens plus `border_radius_px`, applied as `--wg-*` on the host. Chain: override → Shopify theme variable → WiseGift default (`architecture.md` §10.5) |

---

## 2. Feature: Widget — Per-placement hook layouts

### User stories
- As a **widget shopper**, I want the "for me or for someone else?" question to fit where I am in the store (home page, product list, product page, gift page, or the floating button), so that it feels like part of the store and takes me seconds to answer.
- As a **merchant admin**, I want each placement's hook to use wording that suits that page, so that it reads naturally to shoppers and matches my brand voice.
- As **WiseGift**, I want every hook layout to behave identically for exposed and holdout shoppers, so that lift measurement stays valid.

### 2.1 Shared anatomy

Every placement is assembled from the same parts. Placements differ in which parts they show and how they arrange them.

| Part | Description | Editable? |
|---|---|---|
| **Eyebrow** | Small uppercase label with a sparkle icon (for example "Shopping guide") | Merchant-editable per placement. Empty hides it. Hard limit 24 characters |
| **Headline** | The hook prompt. This is the existing `placement_hooks[placement]` / `hook_copy` value | Merchant-editable per placement. Hard limit **90** for per-placement values. Global `hook_copy` keeps its existing 200 limit, and the admin warns above 90 |
| **Subline** | One supporting sentence | Merchant-editable per placement. Empty hides it. Hard limit 120 |
| **Choice buttons** | Two buttons: **For me** and **For someone else**, each with an icon (person, gift) and, on large layouts, a one-line sublabel | **Fixed** (localised) |
| **PDP choice buttons** | **Yes, it's a gift** and **No, it's for me** | **Fixed** (localised) |
| **Reassurance line** | Clock icon plus "Takes about 30 seconds" | **Fixed** (localised). Shown only on layouts that include it (§2.3) |
| **Trigger (floating only)** | Collapsed pill: gift icon, title, subtitle, chevron | Title and subtitle are merchant-editable: title ≤ 24, subtitle ≤ 32, and an empty subtitle hides it |
| **Icons** | Inline SVG: sparkle, person, gift, clock, filter-lines, chevron, close, check | Fixed |

**Why the choice labels are fixed:** they are the measured interaction that sets `intent_mode`. Keeping them identical across tenants keeps the accessibility labels, the localisation and the per-placement funnel comparable. Brand voice lives in the eyebrow, headline and subline.

**Fixed strings.** EN is final. ES and PT are drafts for PO review (PT assumes European Portuguese; Q-P6).

| Key | EN | ES | PT |
|---|---|---|---|
| choice.self | For me | Para mí | Para mim |
| choice.self.sub | Find what suits you | Encuentra lo que te va | Encontra o que combina contigo |
| choice.gift | For someone else | Para otra persona | Para outra pessoa |
| choice.gift.sub | Find the right gift | Encuentra el regalo perfecto | Encontra o presente certo |
| pdp.gift | Yes, it's a gift | Sí, es un regalo | Sim, é um presente |
| pdp.self | No, it's for me | No, es para mí | Não, é para mim |
| reassure | Takes about 30 seconds | Solo unos 30 segundos | Demora cerca de 30 segundos |
| error.retry | That didn't work. Please try again. | No ha funcionado. Inténtalo de nuevo. | Não funcionou. Tente novamente. |
| drawer.close | Close | Cerrar | Fechar |

### 2.2 Copy model and resolution

- [ ] Each placement resolves **three** copy fields: eyebrow, headline and subline. The headline follows the existing 2026-10-15 chain unchanged: `placement_hooks[placement]` → `hook_copy` → per-placement default → generic fallback. The eyebrow and subline follow the same chain except the global `hook_copy` step, because there is no global eyebrow or subline.
- [ ] With niche packs (§3), **step 3 ("per-placement default") resolves from the tenant's active pack**. The defaults in §3.2 replace the current backend constants. General-pack defaults become the WiseGift defaults for all tenants. No pilots exist yet, so no merchant sees a silent change.
- [ ] The backend resolves all three fields per placement and language, and returns them in the session bootstrap `config`. `copyForPlacement` keeps its meaning (the resolved headline) for older bundles. Field names and shape belong to `backend-engineer` (Q-B1, Q-B2).
- [ ] Copy is resolved for the storefront's language (EN, ES or PT at MVP). If a merchant edits one language only, the other languages keep their own resolved values. Edits are never auto-translated.
- [ ] The widget never truncates merchant copy with an ellipsis. Text wraps, and the layout grows vertically.
- [ ] On `pdp_slot` the choice buttons read "Yes, it's a gift" and "No, it's for me", so the headline must be a yes/no question. The admin shows helper text under the PDP headline field: "Phrase this as a yes/no question. The buttons read 'Yes, it's a gift' and 'No, it's for me'."

### 2.3 Layouts per placement

Measurements come from the mockups and describe intent, not pixel law. Spacing maps to the `spacing_scale` token (§2.6).

#### `home_hero` — expanded card (`Main.dc.html`)
- [ ] A standalone card shows, in order:
  - the eyebrow, with its sparkle icon;
  - the headline, as a heading (§2.5);
  - the subline;
  - a **two-column grid of large choice buttons**: a 48 px icon tile, the label, and the sublabel underneath;
  - the reassurance line.
- [ ] Choice order: **For me** first, then **For someone else**.
- [ ] Default emphasis: the **For someone else** icon tile is filled with the accent colour (white glyph). For me uses a tinted tile. Both buttons have the same neutral border in the default state, so neither looks pre-selected (§7, M4).
- [ ] The card uses `card_shadow`.
- [ ] **≤ 400 px:**
  - the choice grid becomes one column of full-width buttons, each at least 64 px tall;
  - the headline steps down one size (mockup 26 px → 22 px);
  - the eyebrow, subline and reassurance line stay.
- [ ] Default (General pack) copy:
  - eyebrow: "Shopping guide";
  - headline: "Not sure what to pick? Tell us who it's for.";
  - subline: "A few quick questions, no account needed."

#### `collection_header` — product-list band (`PlpHook.dc.html`)
- [ ] A horizontal band sits above the product grid. Left to right:
  - a 52 px icon tile (filter-lines);
  - a text block: the headline, with the subline below it;
  - two compact buttons, 44 px tall: **For me** (outline) and **For someone else** (primary, accent fill).
- [ ] No eyebrow and no reassurance line, because the band must stay short.
- [ ] No shadow: border only, so the band sits quietly in the page flow.
- [ ] **≤ 400 px:**
  - the icon tile is hidden;
  - the text block takes the full width;
  - the buttons sit below it in a two-column grid, each at least 44 px tall;
  - if either label wraps, as with longer ES or PT strings, the buttons stack one per row.
- [ ] Default copy:
  - headline: "Too many options? Let's narrow them down.";
  - subline: "Tell us who it's for and we'll pick the ones worth a look."
- [ ] **Dependency:** no Theme App Extension block exists for collection templates. The 2026-10-07 floating-hook entry deferred it "if merchants ask". Shipping this layout needs a new `app_block` and a `collection_header` row on the Placements screen (Q-F4).

#### `pdp_slot` — card under Add to bag (`PdpHook.dc.html`)
- [ ] A compact card sits below the merchant's Add to bag button. It shows:
  - a top row with a 40 px gift icon tile, the headline and the subline;
  - a two-column row of 44 px buttons: **Yes, it's a gift** (primary, accent fill) **first**, then **No, it's for me** (outline).
- [ ] No eyebrow, no reassurance line, no shadow.
- [ ] The card must not compete visually with Add to bag. It uses the widget's own background and a 1 px border, and its primary button is never larger than the merchant's Add to bag.
- [ ] **No recommendations are rendered before intent.** The PDP card is a hook only. Pre-intent PDP recs are gated by P1 / G13 and are not part of this work (§2.7).
- [ ] **≤ 400 px:** two columns are kept down to 340 px; below that, the buttons stack.
- [ ] Default copy:
  - headline: "Buying this for someone?";
  - subline: "Tell us a little about them and we'll suggest a few they'd love."
  - This differs from the mockup on purpose: §7, M7.

#### `gift_finder` — dedicated gift page (no mockup)
- [ ] Uses the `home_hero` anatomy at the page's content width, with the eyebrow, the two large choice buttons and the reassurance line. Both choices stay, so `intent_mode` is still captured, but the copy is gift-led (Q-P3).
- [ ] Default copy:
  - eyebrow: "Gift finder";
  - headline: "Find a gift they'll actually love.";
  - subline: "Answer a few quick questions and we'll pick from the whole store."

#### `floating` — collapsed pill and expanded drawer (`HomePill.dc.html`)
- [ ] **Collapsed (pill):** a rounded button showing, in order:
  - a 36 px circular gift icon in the accent colour;
  - a title, with an optional subtitle on a second line;
  - a chevron.
- [ ] Default trigger copy:
  - title: "Need help choosing?";
  - subtitle: "Tell us who it's for".
- [ ] **Position — needs a PO decision (Q-P1).** The mockup puts the pill bottom-right. The 2026-10-07 "Widget v1" decision made the **side tab** the default (`trigger_mode = side_tab`) precisely because bottom-corner pills collide with cart drawers and chat widgets.
  - Recommendation: ship this pill as the visual for the existing **`pill`** trigger mode, and keep `side_tab` as the default.
  - The side tab keeps its title-only label (the existing ≤ 32-character "Gift finder" default), because a vertical tab cannot fit a subtitle.
- [ ] **≤ 400 px:** the pill collapses to a 56 px round icon-only button (the existing mobile behaviour). Its accessible name is the title plus the subtitle.
- [ ] **Expanded (drawer):** activating the trigger opens the existing right-side drawer: `min(420px, 92vw)` wide, full width on small screens.
  - The drawer content uses the `home_hero` anatomy. At drawer width the choice grid keeps two columns above 400 px and stacks below.
  - Drawer default copy: eyebrow "Shopping guide"; headline "Who are we shopping for today?"; subline "A few quick questions, no account needed."
- [ ] The first-visit pulse nudge (2026-10-07) is kept. It is suppressed under `prefers-reduced-motion: reduce`.
- [ ] The trigger is rendered outside the Shadow DOM (2026-10-07 floating-hook entry), so the host's `--wg-*` variables do not reach it. Its colours today come from theme-block settings. Per the 2026-10-15 rule ("admin is the single source of truth for appearance"), these should move to the admin (Q-P2).
- [ ] **Co-existence:** if a page shows both an inline hook (`home_hero`, `gift_finder` or `collection_header`) and the floating pill, the pill stays visible. Two hooks on one screen is acceptable at MVP. Hiding the pill while an inline hook is in view is a candidate refinement (Q-F8).

### 2.4 States (all placements)

| State | Behaviour |
|---|---|
| **Default** | As laid out above. Both choices are enabled. |
| **Hover** (pointer devices) | The border darkens (text colour at about 40% mix), the background takes a light accent tint, and the cursor becomes a pointer. A 120 ms transition, removed under `prefers-reduced-motion`. |
| **Focus** (keyboard) | A `:focus-visible` ring: 2 px outline, 2 px offset, in the accent colour. If the accent is below 3:1 against the card background, the ring uses `text_color` instead. Never removed. |
| **Selected / pressed** | The activated choice shows a 2 px accent border, a light accent tint and a check icon, so the state does not rely on colour alone. It is visible during loading, and again when the shopper returns to the hook with Back from the form; focus then returns to that button. |
| **Loading** | The first activation triggers the session bootstrap (2026-10-07). The chosen button shows a spinner after 300 ms (no flicker on fast responses) and sets `aria-busy="true"`; the other button is disabled. When the bootstrap returns, the intent form for the chosen mode renders and focus moves to its first field or heading. |
| **Error** | If the bootstrap fails, the buttons return to default and a single polite status line appears under them: "That didn't work. Please try again." No technical detail is shown. Activating a choice again retries. The line is announced through a `role="status"` region. |
| **Collapsed / expanded** | `floating` only (§2.3). `home_hero`, `gift_finder`, `collection_header` and `pdp_slot` have no collapsed state at MVP (Q-P7). |

- [ ] Every state above renders identically for exposed and holdout sessions (§2.7).

### 2.5 Accessibility

- [ ] Choice buttons and the trigger are real `<button type="button">` elements, not links or `div`s.
- [ ] Every interactive target is at least **44 × 44 px** on all placements and viewports. The large `home_hero` buttons are at least 64 px tall on mobile.
- [ ] The two choices sit in a `role="group"` labelled by the headline (`aria-labelledby`). Each button's accessible name is its label plus its sublabel where shown. Icons are `aria-hidden="true"`.
- [ ] The headline is an `<h2>` on `home_hero` and `gift_finder`. On `collection_header` and `pdp_slot` it is a non-heading text element, so the widget does not inject headings into the merchant's product-page outline. In the drawer it is the dialog's label.
- [ ] Trigger: `aria-haspopup="dialog"`, `aria-expanded` reflecting the drawer, and `aria-controls` pointing to the drawer. Drawer: `role="dialog"`, `aria-modal="true"`, labelled by its headline. Focus is trapped inside while it is open. Esc and the Close button close it, and focus returns to the trigger.
- [ ] Contrast:
  - body and label text at least **4.5:1** against the card background;
  - non-text UI that identifies a control (outline-button borders, the selected border, the focus ring) at least **3:1**;
  - the derived neutral border is clamped to at least 3:1 (§2.6).
- [ ] **Text on accent fills** (the primary button and the filled icon tile) is chosen automatically, white or near-black, whichever has the higher contrast against the accent. No new token. If the merchant's accent is light (for example yellow), the text becomes dark.
- [ ] **Eyebrow** text uses the accent colour only if the accent reaches 4.5:1 on the background; otherwise it uses `text_color`.
- [ ] Layouts reflow without loss of content at 320 px width and at 200% text zoom.
- [ ] All motion (pulse, hover transitions, spinner easing) respects `prefers-reduced-motion`. The spinner may stay, but without decorative easing.

### 2.6 Theming: existing tokens only

No new tokens. Values the mockups hard-code map to existing tokens or are derived from them in CSS.

| Mockup value | Maps to |
|---|---|
| Accent (`{{accent}}`) | `--wg-accent-color` (→ `--color-accent` → WiseGift default) |
| `#141418` headline and labels | `--wg-text-color` |
| `#5b5b66` secondary text | Derived from `text_color` at about 72% over the background. Must still reach 4.5:1; frontend-engineer clamps it |
| `#ffffff` card background | `--wg-background-color` |
| `#f0f1f7` icon-tile tint | Derived: accent mixed about 8% into the background |
| `#e4e4ea` / `#d9d9e1` borders | Derived: text mixed into the background, **clamped to ≥ 3:1 for control borders** (the mockup's `#d9d9e1` on white is about 1.4:1) |
| `#141418` primary fill on the PLP band and the pill | **Accent**, for consistency with `home_hero` and `pdp_slot` (§7, M3) |
| Manrope | `--wg-heading-font-family` for the headline; `--wg-font-family` for everything else. **No web font is loaded by the widget** (§7, M2) |
| Card radius 14–18 px; fully rounded buttons | `--wg-border-radius` for cards and controls alike. Controls are fully rounded only when the radius is ≥ 22 px, so one token produces a coherent look (§7, M9) |
| Paddings and gaps | `--wg-space-sm` / `-md` / `-lg` from `spacing_scale` |
| Card shadow | `--wg-card-shadow` on standalone cards (`home_hero`, `gift_finder`, the drawer); none on the inline `collection_header` and `pdp_slot` cards |

- [ ] Derived colours and the on-accent text rule use CSS (`color-mix()`) or a few lines of mount-time code. They count against the existing theming sub-budget of < 1 KB gzipped (`architecture.md` §10.5). Browsers without `color-mix()` fall back to fixed neutral values that meet the contrast rules on a white background.

### 2.7 Events, KPIs and holdout symmetry

- [ ] **No new event types and no new event fields.** The event API's strict schema rejects unknown fields, and this work must not need a contract change. Hook layout is not an event dimension.
- [ ] `placement` on every event takes one of the existing values: `home_hero`, `collection_header`, `pdp_slot`, `gift_finder` or `floating`. Per-placement KPIs and the analytics `placement` filter therefore keep working. `collection_header` and `floating` become selectable in the filter once those placements produce traffic.
- [ ] **When events fire does not change.** `frontend-engineer` documents the current mapping per placement before build (Q-F1): which moment emits `widget_shown` (viewport entry or first interaction) and which emits `widget_engaged` (proposed: activating either choice). The questions to settle:
  - Does showing the collapsed pill emit `widget_shown`, or only opening the drawer? Today the bootstrap happens on first interaction, so the pill alone has no session and emits nothing.
  - Whatever the answer, the same rule must apply in both arms.
- [ ] **Holdout symmetry (D6, P1):** layout, copy, pack, theme, states and timings of the hook are identical for exposed and holdout. The widget must not read `isHoldout` to change anything before intent submission. The only arm difference stays the post-intent neutral collapsed rec slot (D6).
- [ ] **No pre-intent recommendations** on any layout, including the PDP card. Adding any recs-before-intent strip would make `widget_shown` a trigger (P1), and that is gated by G13: event flag, backend support and server-side holdout enforcement in the same release.
- [ ] **Widget CTR stays an A/A integrity check (P2).** A layout change moves both arms together, so the check stays valid. Before/after comparisons of hook layouts or copy are **not** lift claims. They are pre/post observations confounded by season and traffic mix. Copy experiments belong to the roadmap's A/B item.
- [ ] Changing pack or copy mid-pilot changes the treatment for both arms at once. The readout notes the change dates (Q-B5, Q-R4).

### 2.8 Bundle

Today the bundle is about 20 KB gzipped; the budget is 50 KB.

| Item | Estimate (gzipped) | Note |
|---|---|---|
| Four layout variants sharing one choice-button component | 2–4 KB | Mostly CSS and markup |
| 8 inline SVG icons | < 1 KB | Already partly present |
| Derived-colour and on-accent rules | Within the existing < 1 KB theming sub-budget | |
| Pack content | **0 KB** | Resolved server-side and delivered in the bootstrap. Packs must **not** ship as string tables in the bundle |

- [ ] Expected total after this work: about 23–25 KB gzipped. CI's 50 KB assertion stays.
- [ ] **Heavy items to keep out:**
  - web fonts (Manrope);
  - room or lifestyle photos (as in `HomeRoom`);
  - Lottie or animation libraries;
  - per-pack or per-locale string tables inside the bundle.

---

## 3. Feature: Merchant admin — Niche packs (starter presets)

### User stories
- As a **merchant admin**, I want to say what my store sells, so that the widget starts with wording and choices that fit my shoppers without me writing copy.
- As a **merchant admin**, I want to change my pack later without losing the copy I wrote myself.
- As a **merchant admin**, I want to reset any field to my pack's default when I no longer want my edit.

### 3.1 What a pack is
A pack is a named set of **defaults** for configuration that already exists. It adds no question types, no intent fields, and no ranking or retrieval change. The only new stored value is which pack is active (proposed `widget_config.vertical_pack`: `fashion | home_decor | general`, default `general`; Q-B3).

**Model: defaults layer, not copy-in.** Pack values are not copied into the merchant's override fields. They sit in the resolution chain as the per-placement default (step 3 of the hook-copy chain; the active form schema for the intent questions). Overrides store only what the merchant edited. This is the same pattern as `theme_tokens`. As a result:
- switching pack updates every field the merchant never edited, automatically;
- fields the merchant edited are kept;
- "Reset to pack default" means clearing that override;
- no per-field "was edited" tracking is needed: an override existing is the edit.

### 3.2 Pack contents

**Hook copy per placement** (EN; ES and PT need a translation pass, Q-P6)

| Placement / field | General | Fashion | Home & Decor |
|---|---|---|---|
| `home_hero` eyebrow | Shopping guide | Style guide | Home guide |
| `home_hero` headline | Not sure what to pick? Tell us who it's for. | Find the piece that fits. Tell us who it's for. | Make a space feel finished. Tell us who it's for. |
| `home_hero` subline | A few quick questions, no account needed. | A few quick questions about style and budget. | A few quick questions about style and budget. |
| `collection_header` headline | Too many options? Let's narrow them down. | So many styles. Let's find the right one. | Lots to choose from? Let's find what fits. |
| `collection_header` subline | Tell us who it's for and we'll pick the ones worth a look. | Tell us who you're shopping for and we'll pick the pieces worth a look. | Tell us who it's for and we'll pick the pieces worth a look. |
| `pdp_slot` headline | Buying this for someone? | Buying this for someone? | Buying this for someone's home? |
| `pdp_slot` subline | Tell us a little about them and we'll suggest a few they'd love. | Tell us a little about them and we'll suggest pieces that suit their style. | Tell us a little about them and we'll suggest a few pieces they'd love. |
| `gift_finder` eyebrow | Gift finder | Gift guide | Gift guide |
| `gift_finder` headline | Find a gift they'll actually love. | Gifts they'll actually wear. | Gifts for the home they'll keep for years. |
| `gift_finder` subline | Answer a few quick questions and we'll pick from the whole store. | Tell us about them and we'll pick from our collection. | Housewarming, wedding or just because. Tell us about them and we'll pick from our range. |
| `floating` trigger title | Need help choosing? | Need a style hand? | Need a hand choosing? |
| `floating` trigger subtitle | Tell us who it's for | Tell us who it's for | Tell us who it's for |
| `floating` drawer eyebrow | Shopping guide | Style guide | Home guide |
| `floating` drawer headline | Who are we shopping for today? | Who are we dressing today? | Who are we shopping for today? |
| `floating` drawer subline | A few quick questions, no account needed. | A few quick questions, no account needed. | A few quick questions, no account needed. |

**Intent questions**

The field labels, the option labels, the option **order and visibility**, the budget bands and the interests helper text are what change. **Wire values do not change:** `relationship` and `occasion` send the same values as today, and budget still sends `budget_min` / `budget_max`. The ranker sees no new value.

| Field | General | Fashion | Home & Decor |
|---|---|---|---|
| Relationship: label | Who's it for? | Who's it for? | Who's it for? |
| Relationship: options, in display order (wire value → label) | Partner → My partner · Family → Family · Friend → A friend · Colleague → A colleague · Other → Someone else | Partner → My partner · Friend → A friend · Family → Family · Colleague → A colleague · Other → Someone else | Family → Family · Friend → A friend · Partner → My partner · Colleague → A colleague · Other → Someone else |
| Occasion: label | What's the occasion? | What's the occasion? | What's the occasion? |
| Occasion: options shown, in order | Birthday · Anniversary · Christmas · Wedding · Baby · Housewarming · Just because · Other | Birthday · Anniversary · Christmas · Wedding · Baby · Just because · Other (Housewarming hidden) | Housewarming · Wedding · Christmas · Birthday · Anniversary · Baby · Just because · Other |
| Budget bands (shop currency) | Under 50 · 50–100 · 100–200 · Over 200 (today's bands) | Under 30 · 30–60 · 60–120 · Over 120 | Under 40 · 40–100 · 100–250 · Over 250 |
| Gift interests: label | What are they into? | Anything about their style? | Any room or style in mind? |
| Gift interests: helper text | e.g. cooking, hiking, coffee, board games | e.g. minimal, earthy colours, loves linen, sporty | e.g. bedroom, Scandinavian, warm lighting, linen |
| Self budget: label | What's your budget? | What's your budget? | What's your budget? |
| Self interests: label | What are you looking for? | What are you looking for? | What are you looking for? |
| Self interests: helper text | e.g. something for the weekend, a new hobby | e.g. a light jacket for spring, black ankle boots | e.g. a reading lamp for the living room, linen bedding |

Notes:
- Helper text is shown as visible text under the field, not only as a placeholder, which disappears on input and has weak contrast.
- The occasion list follows `spec.md`. The widget's 2026-10-07 picker listed "Holiday" in place of Christmas, Wedding and Housewarming. Which enum is canonical needs confirming before packs ship (Q-F3).
- **Fashion "Who is it for?" → Women / Men / Kids is not included.** That is a department dimension, not a relationship, and no intent field carries it today. See Q-R1. Without it, the Fashion pack can recommend menswear to someone shopping for a female partner. This is the pack's biggest quality risk, and it is flagged to the PO (Q-P4).

**Suggested theme preset.** It sets **shape tokens only**. Colours and fonts are left alone so the merchant's brand and theme inheritance stay intact.

| Token | General | Fashion | Home & Decor |
|---|---|---|---|
| `border_radius_px` | no suggestion (keeps current) | 2 | 14 |
| `spacing_scale` | no suggestion | roomy | cozy |
| `card_shadow` | no suggestion | none | subtle |

### 3.3 Applying, changing and resetting a pack

- [ ] **Applying a pack** sets `vertical_pack`. Fields without a merchant override immediately resolve to the new pack's defaults: hook copy for every placement and language, and the intent question labels, options and bands.
- [ ] **Fields the merchant edited are kept by default.** Before confirming, the merchant sees a **preview** listing:
  - "Will change": every unedited field, with its before → after value;
  - "Your edits, kept": every edited field, with a checkbox "Replace with [Pack] default", unchecked by default.
- [ ] Checked edits are cleared, so they resolve to the new pack. Nothing else is touched.
- [ ] **Merchant visibility toggles are never changed by a pack.** That covers relationship, occasion, budget and interests as required, optional or hidden. A pack changes labels and options, not which fields are on.
- [ ] **Theme preset is opt-in.** The confirm step has a checkbox "Also apply the suggested look (corners, spacing, shadow)":
  - checked by default during onboarding, where there are no overrides to lose;
  - unchecked by default when changing pack later.
  - Applying it writes the shape tokens as overrides, exactly like the existing Theming presets, and the existing per-token Reset and "Auto-inherit from theme" preset undo it.
- [ ] The confirm button names the action: "Switch to Fashion". After applying, a toast reads "Switched to Fashion" with **Undo**, available until the merchant navigates away (same pattern as the Theming presets).
- [ ] **Reset to pack default (per field):** every edited copy field shows an "Edited" badge and a "Reset to [Pack] default" link. Unedited fields show a muted "[Pack] default" label.
- [ ] **Reset all copy:** "Reset all copy to [Pack] defaults" opens a confirmation that lists every field it will clear.
- [ ] **Global hook copy interaction:** the global `hook_copy` sits above pack defaults in the chain (step 2), so a non-empty global value hides every pack headline that has no per-placement override. The preview and the Configuration screen state this ("Your default hook copy is used on 3 placements instead of the Fashion headlines"), with a one-click "Clear default hook copy" (Q-P5).
- [ ] Changing intent labels, options or bands produces a **new intent-form schema version**. `intent_submitted` events already carry `intentFormSchemaVersion`, so analysis can tell the forms apart. A pack change that touches only hook copy does not bump the form schema.
- [ ] Changes propagate to live widgets within 5 minutes (existing config cache).
- [ ] Every pack change is recorded with a timestamp for readout notes (Q-B5).

### 3.4 Admin UX

**Picker component, "What do you sell?":** a radio-card group of three cards. Each card has a name, a one-line description and a sample headline.

| Card | Description | Sample |
|---|---|---|
| **Fashion** | Clothing, shoes and accessories | "Find the piece that fits. Tell us who it's for." |
| **Home & Decor** | Furniture, textiles, lighting and decor | "Make a space feel finished. Tell us who it's for." |
| **General** | A mix of products, or something else | "Not sure what to pick? Tell us who it's for." |

- [ ] The picker is a native radio group (arrow keys move between cards, Space selects) with a visible selected state that does not rely on colour alone (check icon plus border). Each card is at least 44 px tall.

**Onboarding (Step 1, replaces "Vertical selection")**
- [ ] Step 1 asks "What do you sell?" with the three cards. Choosing **General** reveals an optional "Tell us more" select with the remaining verticals: Beauty, Tech, Books, Food, Kids, Jewelry, Other. This keeps `tenants.vertical` populated for future packs and analytics (Q-P8).
- [ ] The choice sets `vertical_pack` and `tenants.vertical`. Fashion maps to `fashion`, Home & Decor to `home`, and General to the "Tell us more" value, or `other` if none is chosen.
- [ ] The suggested look is applied by default, and the merchant sees it in Step 4 (Preview), where the existing preset library and custom copy remain available. Step 4's preset library is now the active pack's copy, plus the merchant's own edits.
- [ ] The step is resumable like the rest of the wizard.

**Configuration (Widget → Configuration)**
- [ ] A **"Store type"** card at the top shows the active pack and a **Change** button. Change opens the picker, then the preview and confirm step from §3.3.
- [ ] Below it, a **"Hook copy" section with one tab per placement** (Home, Collection, Product page, Gift finder, Floating button). Only enabled placements are shown by default; a "Show all" toggle reveals the rest. Each tab has:
  - eyebrow, headline and subline fields (trigger title and subtitle on Floating), with live character counts and the limits from §2.1;
  - a language switcher (EN / ES / PT);
  - the Edited / [Pack] default markers and reset links;
  - a **live preview** of that placement's layout, rendered in the real Shadow DOM with the tenant's theme and updated within 500 ms of typing (same rule as the Theming preview);
  - preview toggles for desktop and ≤ 400 px widths, and for the default and focus states.
- [ ] The **intent questions** are shown read-only with the active pack's labels and options ("Set by your store type"), next to the existing visibility toggles. Merchant editing of labels and options is not part of this work.
- [ ] Every control is keyboard-reachable and labelled for screen readers.

### 3.5 Commercial angle (for `marketing-manager`)
Packs let WiseGift show up as **"WiseGift for Fashion"** or **"WiseGift for Home & Decor"** rather than a generic recommendation app. The App Store listing, demo store and Design Partner outreach can lead with copy and question wording that already sound like that merchant's category, and the install-to-live time stays under 15 minutes because nothing has to be written. This is honest only if the pack really is a starting point and not a vertical engine. Positioning should say "set up for fashion stores in one click", not "AI trained for fashion", until Q-R1 and vertical ranking work exist. Packs also prepare the ground for the roadmap's verticalised intent packs (the Growth-tier step), so the pricing question (Q-P9) should be settled before the listing names them.

---

## 4. Acceptance summary for the human validator

Checklist the PO can validate on staging once built:
- [ ] Each of the five placements renders the layout in §2.3 at desktop width and at ≤ 400 px. On Dawn, the choice buttons are real buttons, at least 44 px, and keyboard-operable.
- [ ] Editing a placement's headline in the admin changes only that placement on the storefront within 5 minutes.
- [ ] An exposed session and a holdout session (forced through a test tenant) show a pixel-identical hook and form. They differ only after intent submission (neutral collapsed slot).
- [ ] Switching General → Fashion changes unedited copy and labels, keeps an edited headline, and Undo restores the previous state.
- [ ] The CI bundle report shows ≤ 50 KB gzipped, and the widget makes no font or image requests.

---

## 5. Open questions

| # | Question | Owner |
|---|---|---|
| Q-B1 | Storage for the eyebrow and subline per placement and language: extend `placement_hooks` values from a string to `{eyebrow, headline, subline}` per language, or add a sibling column? Keep `PUT /admin/v1/widget/config` backward compatible. | backend-engineer |
| Q-B2 | Bootstrap `config` additions: the resolved `{eyebrow, headline, subline}` for the session's placement (keep `copyForPlacement`), floating trigger copy, and the active intent-form labels, options and bands. The form schema is not in the bootstrap today, so the widget presumably hard-codes it. Payload size per bootstrap? | backend-engineer |
| Q-B3 | `widget_config.vertical_pack` (`fashion | home_decor | general`, default `general`) and pack defaults in code (for example `PlacementHooks.defaultFor(placement, pack, lang)`). Is a separate column justified, or can the pack be derived from `tenants.vertical`? (Recommendation: separate column, because a jewelry store may want the Fashion pack.) The onboarding vertical endpoint is also not built (`architecture.md` §9.3). | backend-engineer |
| Q-B4 | Confirm `floating` and `collection_header` are accepted keys in `placement_hooks` and `placements_enabled`, and that `floating` has a backend default (planned in 2026-10-07 as a follow-up one-liner). | backend-engineer |
| Q-B5 | Record pack and copy changes with a timestamp (audit row) so readouts can note them, alongside the D4 holdout-split audit. | backend-engineer |
| Q-B6 | Pack labels and options as `intent_form_schemas` versions: a new version on each pack switch? There is no admin route for the schema today. | backend-engineer |
| Q-R1 | **Fashion department (Women / Men / Kids).** Not representable today: `relationship` is a relationship enum, and the self flow has no such field at all. Options: (a) defer to the configurable questionnaire; (b) seed it as an interests chip (soft LLM signal, no hard filter, so it would be inconsistent); (c) a new intent field plus a hard catalog filter on `product_type`/tags (an engine change, outside this scope). Recommendation: (a), and accept the risk for the pilot. **PO decision 2026-10-15:** option (a) for the first Fashion release. Next step: a "Who is it for?" question whose options are **merchant-selected Shopify collections** (department, age band, etc.), applied as a hard collection filter. See decisions.md 2026-10-15 "Hooks per placement and niche packs". | recommendations-specialist (decision: PO) |
| Q-R2 | Confirm the ranker prompt and the cache intent signature use **wire values**, not display labels, so relabelling ("My partner") cannot change ranking or fragment the cache. | recommendations-specialist |
| Q-R3 | Pack-specific budget bands change the shape of `budget_min` / `budget_max` across tenants. Any effect on cache hit rate, or on the planned band bucketing of the intent signature? | recommendations-specialist |
| Q-R4 | How readouts treat a pack or copy change during a measurement window (note only, or segment by period like D4). | recommendations-specialist |
| Q-F1 | Document today's event mapping per placement: when `widget_shown` and `widget_engaged` fire, and whether the collapsed pill emits anything before the drawer opens. Confirm it is identical in both arms. | frontend-engineer |
| Q-F2 | Are the form labels, options and budget bands hard-coded in the widget today? What is needed to render them from bootstrap config? | frontend-engineer |
| Q-F3 | Occasion enum mismatch: `spec.md` lists Birthday / Anniversary / Wedding / Baby / Christmas / Housewarming / Just because / Other; the 2026-10-07 widget picker lists Birthday / Anniversary / Just because / Holiday / Baby / Other. Which is live, and which does the backend accept? | frontend-engineer + backend-engineer |
| Q-F4 | New Theme App Extension `app_block` for collection templates (`collection_header`). Does it sit inside Dawn's colour-scheme wrapper, so auto-inherit works (2026-09-29 finding)? | frontend-engineer |
| Q-F5 | The trigger renders outside the Shadow DOM: how does it receive admin theme values (accent, radius, font) once they move to the admin (Q-P2)? | frontend-engineer |
| Q-F6 | Confirm the bundle estimate in §2.8 after a spike. | frontend-engineer |
| Q-F7 | The automatic on-accent text colour and the derived neutrals: CSS `color-mix()` plus contrast clamping in under 1 KB? Fallback for older browsers? | frontend-engineer |
| Q-F8 | Hide the floating pill while an inline hook on the same page is in the viewport? Two widget instances have no coordination today. | frontend-engineer (decision: PO) |
| Q-P1 | Floating trigger default: keep the **side tab** (2026-10-07, avoids cart-drawer and chat collisions) and offer the mockup pill as the `pill` mode, or make the bottom-right pill the default again? Recommendation: keep the side tab. | PO |
| Q-P2 | Move the trigger settings (copy, mode, position, colours) from the theme block to the admin, under the 2026-10-15 "admin is the single source of truth" rule? Recommendation: yes. **PO decision 2026-10-15: yes.** | PO |
| Q-P3 | `gift_finder` has no mockup. Is the `home_hero` anatomy with gift-led copy acceptable? | PO |
| Q-P4 | Ship the Fashion pack without a department question (see Q-R1), or hold Fashion until it exists? **PO decision 2026-10-15:** ship Fashion without a department question. The pack copy invites the shopper to say who it's for in the free-text field. A collection-based "Who is it for?" question follows as the next step. | PO |
| Q-P5 | Keep the global "Default hook copy" in the admin once packs exist, or demote it to "Use the same headline everywhere (optional)"? | PO |
| Q-P6 | ES and PT copy for the packs and fixed strings: translation pass and PO review. Confirm European Portuguese and the register (informal "tu" vs neutral). **PO decision 2026-10-15:** European Portuguese, informal "tu". | PO |
| Q-P7 | Did the "Home hook, expanded" mockup name imply a collapsed state for the inline home card (for example collapsing to the pill after dismissal or scroll)? Recommendation: no collapsed state for inline hooks at MVP. | PO |
| Q-P8 | The onboarding vertical list (9 values) vs packs (3): accept the mapping in §3.4? | PO |
| Q-P9 | Packs on all plans, or reserved for a tier (the roadmap tied verticalised packs to the Growth-tier step)? **PO decision 2026-10-15:** all plans during the pilots. Revisit with marketing-manager when pricing is set. | PO + marketing-manager |

---

## 6. Deferred

| Item | Mockups | Why deferred |
|---|---|---|
| **Configurable questionnaire** (multi-step questions, per-vertical questions such as department, age band, style toggles, room, product category; a merchant-editable question set) | `Question1`, `Question2` (the questions themselves), `HomeRoom`, `HomeNeed` | These need new intent fields, so the ranker and retrieval must use them (hard filters, prompt changes, cache-signature changes) and quality must be evaluated per field. That is ranking work owned by `recommendations-specialist`, outside the "existing configuration only" constraint. |
| **Looks** (curated outfit or room sets as answers or results) and the admin look editor | `LookQuestion`, `AdminLookEditor`, `AdminAddLook` | Needs a look entity, ranking over sets rather than single products, and an evaluation method. Ranking work, same reason as above. |
| **Influencer and social posts** in the widget | `LookPosts` | Rights and platform terms (embedding third-party posts, image licensing), API rate limits and access changes, and moderation. **Contracted influencers are acceptable later**, provided each asset carries a recorded **licence scope**, an **expiry date** and a **takedown** path. |
| Hook copy A/B experiments | — | Roadmap item 6. Needs experiment assignment orthogonal to the holdout. |
| Merchant-editable intent labels and options | — | Packs cover the starting point. Free editing adds validation and localisation burden; revisit after pilot feedback. |

---

## 7. Recommended changes to the mockups

| # | Mockup | Change | Reason |
|---|---|---|---|
| M1 | `HomePill` | Use as the `pill` trigger mode; keep the side tab as the default (Q-P1) | 2026-10-07 decision: bottom-corner pills collide with cart drawers and chat widgets |
| M2 | All | Do not load Manrope or any web font; inherit the storefront's fonts via tokens | Bundle and TTI budget; Google Fonts requests send shopper IPs to a US provider (an EU privacy exposure); brand consistency |
| M3 | `PlpHook`, `HomePill` | Primary fill uses the accent, not hard-coded `#141418`; the pill background maps to admin settings | Consistency across placements; one theming source (2026-10-15) |
| M4 | `Main` | "For someone else" default state: remove the accent border and keep only the filled icon tile as emphasis | The accent border reads as "already selected" and is indistinguishable from the selected state |
| M5 | All | White text on accent is chosen automatically by contrast; the eyebrow falls back to the text colour on light accents | Merchant accents can be light; WCAG 4.5:1 |
| M6 | All | Raise outline-button borders from `#d9d9e1` (about 1.4:1 on white) to at least 3:1 | WCAG 1.4.11 non-text contrast for control boundaries |
| M7 | `PdpHook` | Subline "Let's check it's the right pick, or see a few alternatives they'd love" → "Tell us a little about them and we'll suggest a few they'd love." | The engine does not evaluate the current product for a recipient; the copy over-promises |
| M8 | `Main` | Subline "3 quick questions" → "A few quick questions" | The self flow has 2 fields and the gift flow 4; numbered steps belong to the deferred questionnaire |
| M9 | `PlpHook`, `PdpHook`, `Main` | Accept that a single `border_radius_px` drives cards and controls; fully rounded buttons only at a high radius | No new radius token; one token gives a coherent look across themes |
| M10 | `HomePill` | On ≤ 400 px, collapse to a 56 px icon-only button | The two-line pill is about 230 px wide and crowds mobile screens |
| M11 | `PdpHook` | Keep the card hook-only; no pre-intent recs strip | P1 / G13 measurement gate |
| M12 | `Question1`, `Question2` (for when the questionnaire returns) | The Back button (36 px) and age chips (40 px) need ≥ 44 px; "Next" and "Skip this" are links styled as buttons and should be `<button>`s | Accessibility (touch target size, correct semantics) |
| M13 | `HomeRoom` | Room photos are reference only | Image weight, licensing, and they need a room field that does not exist |
