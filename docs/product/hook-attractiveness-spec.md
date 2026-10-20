# Hook attractiveness — context-adaptive hooks, product preview, trigger config

> **DRAFT — pending PO review.** Written 2026-10-19 by `product-analyst` for the
> "quick win" hook-attractiveness work the PO approved (attractiveness, not
> recommender quality). Nothing here is decided until the PO validates it. Open
> questions for the PO are in §10, each with an owner. This spec is **additive**
> to `spec.md` and `hooks-and-packs.md`; it reuses the existing hooks/packs
> config and does not reinvent it.
>
> **Core premise (from the PO, agreed with the coordinator):** recommendation
> quality is already good. The gap is **attractiveness** — shoppers do not
> notice or engage with the widget. That is a hook problem (copy, trigger,
> visual), not a ranking problem. The shopper's mental state differs by page
> type, so the widget's job and its entry question must differ by placement.
>
> **Inputs read:**
> - `docs/product/spec.md` — Widget — Gift intent hook; Recommendation API; Event API; Analytics dashboard; Attribution & session tracking.
> - `docs/product/hooks-and-packs.md` — §2 per-placement hook layouts, §3 niche packs, §2.7 holdout symmetry / "no pre-intent recs" rule, §5 open questions.
> - `docs/engineering/hooks-and-packs-contract.md` — backend storage model (`placement_hooks` V3/V12), bootstrap `config`, pack defaults, trigger settings, resolution chain.
> - `docs/decisions.md` — 2026-10-07 per-placement hook copy / side-tab trigger; 2026-10-14 M1·2 D1–D9 and holdout; 2026-10-15 packs + "admin is the single source of truth".
>
> **Build ON the existing config, not beside it.** The hook copy already lives
> in `widget_config.placement_hooks` (placement → language → field → string,
> resolved through the pack-default chain and delivered in the session
> bootstrap `config.hook`). The trigger settings already live in typed
> `widget_config` columns (`trigger_mode`, `trigger_position`, colours). This
> spec adds three things on top of that spine: **context tokens in the copy**, a
> **product-preview slot in the hook**, and **trigger activation timing**.

---

## 1. Strategic framing

### 1.1 Three placements, two jobs

The entry question must match where the shopper's head is.

| Placement | Shopper mental state | Job | What is already KNOWN | Entry question must… |
|---|---|---|---|---|
| **PLP / collection** (`collection_header`) | Declared a category intent just by being on the page | **Discovery with a known context** | The category | …**inherit the category as a pre-filled constraint** and never ask the shopper to restate it. Its edge over the store's own facets is the intent/person/occasion/budget dimension the facets lack ("who is it for and what do they like"). |
| **PDP** (`pdp_slot`) | Looking at one specific product | **Conversion / expansion with a known product** | The product | …offer two sub-jobs: (1) complete-the-look / cross-sell (↑ AOV) and (2) **alternative / rescue** ("not quite right? here are 3 like it that fit who/budget better"). |
| **Floating tab** (`floating`) | Varies with the page it floats on | **Trigger, not a third content** | Depends on the page | …**adopt the hook of whatever page it floats on**: PLP-context on a collection, PDP-context on a product, "gift concierge" on homepage/other. |

**Quick-win scope boundary on PDP.** For THIS work the PDP job is primarily the
**alternative / similar + budget-aware** path, which uses **existing
similarity** (the rec API already accepts `context.platform_product_id`).
**Complete-the-look (complementarity)** is a separate, costlier signal being
scoped elsewhere — it is designed here as a **configurable mode** but marked
**dependency-gated** (§4.3, §9). The quick win must be shippable without it.

### 1.2 The three attractiveness levers (ROI order)

1. **Context-adaptive copy** — a hook headline + subcopy that speak to the page,
   via **context tokens** (`{category}`, `{product}`, …) so ONE configured hook
   serves many categories without per-category copy (§2, §6).
2. **Product-preview-in-hook** — show 2–3 **real** products from the context
   (category on PLP, similar-to-this on PDP) **inside** the hook, before the
   shopper answers; the inputs then **refine** that set. People click images,
   not questionnaires (§3). **Gated** — see §3.4 and §9; what populates the
   preview is being confirmed by `recommendations-specialist`.
3. **Trigger config for the floating tab** — immediate / scroll-depth / dwell /
   exit-intent, instead of always-on (§5).

### 1.3 Primary success metric

**Engagement rate = `widget_shown` → `widget_engaged`, measured per placement.**

- This is the explicit target of this work. It is **distinct from downstream
  lift** (conversion / AOV / revenue-per-session), which is unchanged and still
  measured by `m1-2-lift-measurement.md`.
- `widget_engaged` keeps its existing meaning: the shopper activates a choice
  button (the `t₀` trigger). No new event type, no new event field (`spec.md`
  Event API; `hooks-and-packs.md` §2.7).
- **It is an operational pre/post product metric, not a lift claim.** Changing
  copy, preview or trigger moves **both arms together**, so engagement rate is a
  pre/post observation confounded by season and traffic mix (exactly the
  doctrine in `hooks-and-packs.md` §2.7 and M1·2 P2, where Widget CTR is an A/A
  integrity check). Hook A/B testing belongs to the roadmap's experiments item.
- **Caveat:** lever 2 (product preview) can break the A/A symmetry of Widget
  CTR, because showing WiseGift-picked products before intent is a form of
  treatment. This is the central measurement question in §9 / Q-HA1.

---

## 2. Feature: Widget — Context-adaptive hook copy (Lever 1)

### User stories
- As a **widget shopper**, I want the hook to mention the thing I am already
  looking at (this category, this product), so that it feels relevant and I
  understand instantly what the widget does for me here.
- As a **merchant admin**, I want to write ONE hook per placement that adapts to
  any category or product automatically, so that I do not have to author copy
  per collection.

### 2.1 What is new vs what exists
- **Exists:** per-placement `eyebrow` / `headline` / `subline`, editable per
  language, resolved through the pack-default chain, delivered in the bootstrap
  `config.hook` (`hooks-and-packs-contract.md` §1–§3). **No change to storage.**
- **New:** the copy strings may contain **context tokens** that the widget
  interpolates at render time from the storefront context (the collection title
  on a PLP, the product title on a PDP, the store name anywhere). The token
  vocabulary and rules are §6.

### 2.2 Acceptance criteria
- [ ] Any copy field (`eyebrow`, `headline`, `subline`, floating `trigger_title`
  / `trigger_subtitle`) may contain tokens from the §6 vocabulary. A token is
  written `{token}` in the stored string.
- [ ] The widget resolves tokens **at render time** against the page context it
  was given (§6.2). Resolution happens in the widget, after the bootstrap, so
  stored copy stays context-free and cacheable.
- [ ] **Every token must have a fallback** (§6.3). A hook that cannot resolve a
  token (e.g. `{category}` reached from a homepage floating tab) renders the
  field's **tokenless fallback variant**; it never shows a literal `{category}`
  and never shows an empty gap.
- [ ] Token **values are untrusted catalog text** and are rendered as **text
  nodes only**, never as HTML (same rule as merchant copy, contract §5.2). `<`,
  `>`, `&` in a product/collection title are shown literally.
- [ ] A token **value** (not the merchant's own words) may be length-clamped to
  keep the hook readable: a resolved `{product}` / `{category}` longer than the
  §6.4 limit is truncated with an ellipsis. Merchant-authored text is still
  never truncated (`hooks-and-packs.md` §2.2).
- [ ] The admin copy editor (Widget → Configuration, Hook copy tabs) shows the
  **available tokens for that placement** as insertable chips, with a live
  preview that substitutes a **sample** value, and a note on the tokenless
  fallback. (Interaction detail in §8.3.)
- [ ] Holdout symmetry holds: token resolution is identical in both arms and
  does not read `isHoldout` (`hooks-and-packs.md` §2.7).

---

## 3. Feature: Widget — Product-preview-in-hook (Lever 2) — DEPENDENCY-GATED

### User stories
- As a **widget shopper**, I want to see a few real products right away so that I
  can click an image instead of filling in a form, and only answer questions if
  I want to narrow things down.
- As a **merchant admin**, I want the hook to show a taste of what WiseGift can
  surface, so that more shoppers engage.

### 3.1 The preview-first → refine model
- The hook renders, **before any intent is captured**, a slot of **2–3 real
  products** drawn from the page context:
  - on **PLP**: products from the current category (the inherited constraint);
  - on **PDP**: products **similar to** the current product (existing similarity
    over `context.platform_product_id`);
  - on **floating**: whatever the page it floats on resolves to (§5.4).
- The shopper can **click a product** (goes to the merchant PDP, new tab, fires
  `product_clicked` — existing behaviour) OR **answer the hook** (self / gift).
- Answering **refines the already-visible set**: the same slot updates in place
  with intent-aware recommendations. The shopper experiences continuity — the
  products they already saw get better, they do not start from a blank form.

### 3.2 Why this is gated (read before building)
This lever **conflicts with a load-bearing measurement rule** and must not ship
without resolving it:
- `hooks-and-packs.md` §2.7 and M11: **"No pre-intent recommendations on any
  layout."** Adding a recs-before-intent strip makes `widget_shown` a trigger
  (M1·2 P1), which is **gated by G13**: event flag, backend support, and
  **server-side holdout enforcement in the same release**.
- The holdout placebo today only collapses the rec slot **after** intent (M1·2
  D6). A **pre-intent** preview has no placebo defined. If exposed sees WiseGift
  picks before intent and holdout does not, the hook is no longer identical
  across arms and **Widget CTR / engagement rate stops being A/A-comparable**.
- Consent / bootstrap timing: a pre-intent preview needs a `recommendations`
  call **before** the shopper interacts, which needs the widget JWT from the
  session bootstrap, which is **consent-gated** (`spec.md` Attribution &
  session tracking; `consentModeRequired` default `true`). Degraded (no-consent)
  shoppers get **no session**, therefore **no preview** — they see the tokenless
  hook (§2) only. Bootstrap would also have to move from first-interaction to
  viewport entry (`hooks-and-packs-contract.md` §3.3 / Q-C5).

**Therefore:** §3 is specified now (slot, states, refine flow) so it is
build-ready, but it is **blocked** on Q-HA1 (measurement design) + G13 +
the preview source confirmation from `recommendations-specialist`. Levers 1 and
3 ship without it.

### 3.3 Acceptance criteria (design-ready, build-gated)
- [ ] The hook has an optional **preview slot** rendering 2–3 product cards
  (image, title, price — the existing rec-card anatomy, `spec.md` Recommendation
  display), **above or beside** the choice buttons per layout (§8.1).
- [ ] Preview count is **2–3**, configurable per placement within that range
  (distinct from the post-intent rec count 3–10).
- [ ] Clicking a preview card behaves exactly like a recommendation card
  (merchant PDP, new tab, `product_clicked`, accessible "opens in a new tab"
  name). No new event.
- [ ] Answering the hook (self / gift, or any later refine input) **replaces the
  preview set in place** with intent-aware recs; the slot does not disappear and
  reappear. A short loading state covers the swap.
- [ ] If the preview source returns fewer than 2 products, the slot renders what
  it has; if zero, the slot is **omitted** and the hook degrades to the
  copy-only layout (§2) with no empty frame.
- [ ] Preview and refine are **identical in both arms** OR the arm difference is
  explicitly handled server-side under G13 — resolved by Q-HA1. The widget must
  not invent an arm difference client-side.
- [ ] What populates the preview (precomputed cold recs per collection on PLP;
  similar-to-product on PDP; availability and any budget pre-filter) is owned by
  `recommendations-specialist` — see Q-HA2.

### 3.4 Dependency summary
- **Q-HA1 (blocking):** measurement design for pre-intent recs (placebo + G13).
- **Q-HA2 (blocking the preview content):** preview population source and
  signal, owned by `recommendations-specialist`.
- **Bootstrap-on-viewport + consent:** relates to Q-C5 / Q-F1 (bootstrap moment
  and `widget_shown` timing).

---

## 4. Feature: Placement job & mode (admin config)

### User story
- As a **merchant admin**, I want each placement to do the right job for that
  page, so that the widget helps shoppers where they are instead of asking the
  same generic question everywhere.

### 4.1 The per-placement config model
Each placement carries a small **behaviour profile** on top of the existing
copy + trigger config. Proposed shape (backend owns the exact wire/column form;
this is the logical model):

| Config key | PLP `collection_header` | PDP `pdp_slot` | Home `home_hero` | Gift `gift_finder` | Floating `floating` |
|---|---|---|---|---|---|
| **job** (fixed per placement) | `discovery_in_category` | `conversion_on_product` | `discovery_general` | `gift_discovery` | `adopts_page` (§5.4) |
| **mode** (merchant-selectable where >1) | — (single mode) | `alternative` *(default)* / `complete_the_look` *(gated)* | — | — | inherits from adopted page |
| **pre-filled inputs** (not asked) | **category** (inherited constraint) | **product** (+ mode) | none | none | inherited from adopted page |
| **asked inputs** (refine) | person / occasion / budget / interests (per pack visibility) | person / occasion / budget / interests (per pack visibility) | self vs gift, then form | self vs gift, then form | inherited from adopted page |
| **preview slot** (§3) | on *(gated)*, count 2–3 | on *(gated)*, count 2–3 | off (default) | off (default) | inherited |
| **copy** (§2) | eyebrow/headline/subline with tokens | headline/subline with tokens | eyebrow/headline/subline | eyebrow/headline/subline | trigger + drawer, adopted |
| **trigger activation** (§5) | n/a (inline) | n/a (inline) | n/a (inline) | n/a (inline) | immediate / scroll / dwell / exit |

### 4.2 PLP — inherit the category, never restate it
- [ ] On `collection_header`, the current category is a **pre-filled, non-asked
  constraint**. The hook must **not** present a field that asks the shopper
  which category they want — they already declared it by being there.
- [ ] The category is carried into the recommendation request as a context
  constraint (new context dimension — §6.2, Q-HA3). The differentiator the hook
  sells is the **person / occasion / budget / interests** dimension the store's
  own facets do not have.
- [ ] Whether the category is applied as a **hard catalog filter** (recommended)
  or a soft signal is a `recommendations-specialist` + backend decision
  (Q-HA3), and relates to the existing collection-filter follow-up (Q-R1).

### 4.3 PDP — alternative (quick win) vs complete-the-look (gated)
- [ ] **`alternative` (default, ships in this quick win):** uses existing
  similarity over the current product plus the shopper's budget/person signal —
  "not quite right? here are 3 like it that fit who/budget better". No new
  engine signal.
- [ ] **`complete_the_look` (configurable, dependency-gated):** cross-sell /
  complementarity. The admin control exists and is **disabled with an
  explanatory note** until the complementarity signal scoped elsewhere lands
  (§9, Q-HA4). Selecting it while gated is not possible; the control explains
  why and links to status.
- [ ] The PDP card stays **hook-first**. Any pre-intent preview on PDP is the
  gated lever 2 (§3), not a separate always-on cross-sell strip.

---

## 5. Feature: Widget — Floating trigger as context adopter + activation timing (Lever 3)

### User stories
- As a **widget shopper**, I want the floating helper to match the page I am on,
  so that it offers something relevant rather than a generic prompt.
- As a **merchant admin**, I want to choose *when* the floating tab appears
  (immediately, after scrolling, after a dwell, or on exit intent), so that it
  invites rather than nags.

### 5.1 What exists
- The floating trigger exists (`floating_hook.liquid`): `trigger_mode`
  (`side_tab` default / `pill`), `trigger_position`, colours, and per-language
  trigger copy, now owned by the admin (`hooks-and-packs-contract.md` §1.4,
  §4.2; Q-P2 decided yes). A first-visit pulse nudge exists (2026-10-07).

### 5.2 What is new
- **Activation timing** (the lever): a new `trigger.activation` setting
  controlling *when* the floating trigger first appears / nudges.
- **Page-type adoption** (§5.4): the floating drawer renders the hook of the
  page it floats on.

### 5.3 Acceptance criteria — activation timing
- [ ] `trigger.activation` is one of: `immediate` (default, = today's
  always-visible behaviour), `scroll_depth`, `dwell`, `exit_intent`.
- [ ] `scroll_depth` carries a threshold (percent of page scrolled, default
  40%); `dwell` carries a delay in seconds (default 15 s); `exit_intent` has no
  parameter.
- [ ] **`exit_intent` is desktop-pointer only.** On touch / no fine pointer it
  **falls back to `dwell`** at the configured or default delay (there is no
  reliable mobile exit-intent). The admin states this.
- [ ] Whatever the activation, it applies **identically in both arms**, and the
  `widget_shown` timing rule stays the same in both arms (`hooks-and-packs.md`
  §2.7; Q-F1 must document when `widget_shown` fires for a deferred trigger).
- [ ] Auto-surfacing (scroll / dwell / exit) must be **non-disruptive**: it does
  **not** steal focus, does **not** open the drawer by itself, and is
  dismissible; once dismissed it does not reappear on the same page view. All
  motion respects `prefers-reduced-motion` (pulse suppressed), consistent with
  the existing nudge.
- [ ] Activation timing is merchant-configurable per tenant (single floating
  trigger per page at MVP).

### 5.4 Acceptance criteria — page-type adoption
- [ ] The floating trigger **adopts the hook of the page it floats on**:
  - on a **collection** page → the `collection_header` hook + category context;
  - on a **product** page → the `pdp_slot` hook + product context;
  - on **homepage / other** → the `gift_finder` / "gift concierge" hook (no
    category or product token; §6.3 fallback applies).
- [ ] Page type is detected from the storefront (on Shopify, the template /
  `request.page_type`, passed to the widget as context — §6.2). The widget picks
  the adopted placement's resolved copy, preview config and context accordingly.
- [ ] This is a **behaviour change** from today's fixed floating drawer copy
  (`hooks-and-packs.md` §2.3). Confirm with the PO (Q-HA5). The floating-specific
  copy in the packs becomes the **homepage / other** fallback.
- [ ] The trigger **copy** on the pill / side tab still comes from the floating
  copy fields; only the **drawer content** adopts the page hook.

---

## 6. Context-token vocabulary

### 6.1 The tokens (proposed MVP set — kept deliberately small)

| Token | Resolves to | Available on | Example resolved |
|---|---|---|---|
| `{category}` | Current collection / category title | PLP; floating-on-PLP | "Running shoes" |
| `{product}` | Current product title | PDP; floating-on-PDP | "Linen throw blanket" |
| `{store}` | Merchant store / shop name | all placements | "North & Pine" |

- `{category}` and `{product}` are the high-ROI tokens (they carry the page
  context). `{store}` is a convenience token available everywhere.
- **Not in the MVP set** (candidates for later, flagged to avoid scope creep):
  `{vendor}` / `{brand}`, `{product_type}`, `{price}`, `{occasion}`. They add
  resolution and fallback surface for little proven lift; revisit after pilots.

### 6.2 Where values come from
- Values are read from the **storefront page context**, passed into the widget
  at mount (on Shopify via Liquid in the Theme App Extension block:
  `collection.title`, `product.title`, `shop.name`, and the page type via
  `request.page_type` / `{{ template }}`). They are **not** stored in
  `widget_config` and **not** authored by the admin.
- `{category}` as a recommendation **constraint** (not just display text) also
  needs to reach the rec API as a context dimension (collection handle / id).
  That is a **new context field** on `POST /widget/v1/recommendations` (today
  `context` carries only the PDP `platform_product_id`) — Q-HA3.

### 6.3 Fallback rules (required)
- Every token-bearing copy field must resolve to something sensible when the
  token has no value:
  - **Preferred:** the merchant authors a **tokenless fallback variant** of the
    field, used whenever any token in that field is unresolved. (Simplest mental
    model; the admin offers a "fallback wording" field next to the token one.)
  - **Alternative (if the PO prefers one field):** the token resolves to a
    neutral word — `{category}` → "this", `{product}` → "this", `{store}` →
    the merchant's store name is almost always present, so it rarely falls back.
  - Recommendation: **tokenless fallback variant** for `headline` (the load-
    bearing line), neutral-word fallback acceptable for `subline`. Confirm in
    Q-HA6.
- A floating tab on a homepage therefore never shows "Looking for {category}?";
  it shows the homepage / gift-concierge hook (§5.4).

### 6.4 Resolution & safety rules
- [ ] Tokens are matched case-insensitively on the exact spellings in §6.1;
  any unknown `{…}` is left **as literal text is NOT acceptable** — unknown
  tokens are stripped to the fallback wording and logged (admin-side validation
  should prevent them, §8.3).
- [ ] Resolved values render as **text nodes** (no HTML), trimmed.
- [ ] A resolved `{product}` / `{category}` is clamped to **40 code points**
  with an ellipsis if longer (keeps the headline to one or two lines). Merchant-
  authored text around the token is never truncated.
- [ ] Token resolution adds **no bundle weight of note** (string replace over the
  already-delivered copy) and no network call.

---

## 7. Defaults & presets — mapping onto the existing vertical packs

Packs (`general`, `fashion`, `home_decor`) already supply per-placement copy
defaults and are the defaults layer in the resolution chain
(`hooks-and-packs-contract.md` §2, §1.3). This work extends that layer rather
than adding a new one.

- [ ] **Pack defaults gain token-bearing variants** for the placements where a
  context token lifts relevance. The pack default is still a plain string in
  `HookCopyDefaults`; it may now contain `{category}` / `{product}` / `{store}`.
  Each token-bearing default ships with its tokenless fallback (§6.3). Examples
  (EN, illustrative — final copy with the pack copy pass):

  | Placement / field | General | Fashion | Home & Decor |
  |---|---|---|---|
  | `collection_header` headline | "Browsing {category}? Let's find the right one." | "So many {category}. Let's find your pick." | "Lots of {category}? Let's find what fits." |
  | `collection_header` fallback headline | "Too many options? Let's narrow them down." | "So many styles. Let's find the right one." | "Lots to choose from? Let's find what fits." |
  | `pdp_slot` headline | "Buying this {product} for someone?" | "Is this {product} for someone?" | "Is this {product} for someone's home?" |
  | `pdp_slot` fallback headline | "Buying this for someone?" | "Buying this for someone?" | "Buying this for someone's home?" |

  The existing tokenless defaults in `hooks-and-packs.md` §3.2 / contract §2
  become the **fallback** variants, so nothing regresses when a token can't
  resolve.
- [ ] The **job / mode defaults per placement** (§4.1) are code constants, like
  the copy defaults: PLP → `discovery_in_category`; PDP → `alternative`; etc.
  Packs do **not** change the job (a pack is copy + labels + look, not
  behaviour).
- [ ] The **suggested theme preset** per pack (shape tokens only;
  `hooks-and-packs.md` §3.2) is unchanged.
- [ ] Switching pack keeps the §3.3 preview/confirm/undo semantics; a token-
  bearing default is just another field in the "will change / edits kept" diff.

---

## 8. Interaction flows

### 8.1 Preview-first → refine (the key flow, lever 2 — gated)
```
Shopper lands on PLP / PDP
        │
        ▼
Hook renders:  [eyebrow]  [headline with {category}/{product}]  [subline]
               [ preview: 2–3 real products from context ]        ← lever 2 (gated)
               [ For me ]   [ For someone else ]                   ← existing choice
        │
   ┌────┴─────────────────────────────┐
   ▼                                   ▼
Clicks a preview product          Answers the hook (self / gift)
   │                                   │
   ▼                                   ▼
Merchant PDP (new tab),           Intent form (refine): person / occasion /
product_clicked; drawer stays     budget / interests, category/product pre-filled
                                       │
                                       ▼
                                  Same slot updates IN PLACE with
                                  intent-aware recs (loading state covers swap)
                                       │
                                       ▼
                                  product_clicked / intent_submitted as today
```
- The shopper never faces a blank form first; the products are the hook. Inputs
  **refine** an already-visible set.
- When lever 2 is gated off, the flow is the current one: copy-only hook →
  choice → form → recs.

### 8.2 Floating adoption flow (lever 3)
```
Floating trigger appears per activation timing (immediate / scroll / dwell / exit)
        │  (page-type detected)
        ▼
Shopper opens drawer
        │
   ┌────┼───────────────┬─────────────────────┐
   ▼    ▼               ▼                     ▼
 on PLP               on PDP              on home/other
 → collection_header  → pdp_slot          → gift_finder / concierge
   hook + {category}    hook + {product}     hook (no category/product token)
```

### 8.3 Admin — editing context-adaptive copy
- [ ] In Widget → Configuration → Hook copy, each placement tab shows, per field,
  the **tokens available for that placement** as insertable chips (PLP:
  `{category}`, `{store}`; PDP: `{product}`, `{store}`; home/gift: `{store}`).
- [ ] Inserting a token drops `{category}` at the cursor. The **live preview**
  (existing 500 ms Shadow-DOM preview, `hooks-and-packs.md` §3.4) substitutes a
  **sample** value and shows a toggle "preview with a long value" so the merchant
  sees the §6.4 clamp.
- [ ] A field that contains a token shows a **"Fallback wording"** input beneath
  it (§6.3), with helper text: "Shown when there's no {category} — for example on
  your homepage via the floating tab."
- [ ] Validation: a token not in the field's allowed set for that placement is
  flagged inline ("{brand} isn't available here — try {category} or {store}");
  the merchant can still save only after removing/replacing it (unknown tokens
  never reach the storefront, §6.4).
- [ ] Character counts (contract §5.1 limits) count the **authored string
  including the token braces**, not the resolved value. A note explains the
  resolved line may be longer or shorter.

### 8.4 Admin — placement job / mode & preview
- [ ] Each placement tab shows its **job** (read-only label, e.g. "Discovery in
  this category") so the merchant understands what the placement does.
- [ ] PDP shows a **mode** selector: `alternative` (default, enabled) and
  `complete_the_look` (disabled, with "Coming soon — needs the complete-the-look
  signal" note, Q-HA4).
- [ ] Where lever 2 is enabled, a **"Show a product preview in the hook"** toggle
  with a 2–3 count stepper appears; while gated it is disabled with a short note
  linking to status (Q-HA1/Q-HA2).

### 8.5 Admin — trigger activation
- [ ] Widget → Configuration (or Placements, floating row) gains a **"When should
  the floating button appear?"** control: Immediately / After scrolling (%) /
  After a few seconds (s) / On exit intent, with the desktop-only note on exit
  intent and its mobile `dwell` fallback (§5.3).

---

## 9. NEW build vs config-only

### Config-only (reuses existing infra, mostly content/flags)
- Per-placement `eyebrow` / `headline` / `subline` copy, per language — exists
  (`placement_hooks`, admin copy endpoints).
- Token-bearing **pack default strings** and their tokenless fallbacks — content
  change to `HookCopyDefaults` (§7). The *interpolation* is new build (below).
- Trigger `mode` / `position` / colours — exists (contract §1.4, §4.2).
- Post-intent rec count per placement — exists.
- Intent-form field **visibility** per pack — exists.

### NEW build
1. **PLP placement = `collection_header` Theme App Extension app block.** Does
   **not** exist today (`hooks-and-packs.md` §2.3 dependency; contract Q-C4 /
   Q-F4). **This is the single biggest new-build item** and the only one that
   adds a storefront surface. Needs the app block, Dawn colour-scheme wrapper
   check (Q-F4), and a `collection_header` row on the Placements screen.
2. **Context-token interpolation** in the widget: parser, allowed-token set per
   placement, resolution from page context, fallback handling, length clamp,
   text-node rendering (§6). New.
3. **Storefront context passing + rec-API context dimension:** page type and
   collection handle passed into the widget (Liquid), and `{category}` reaching
   `POST /widget/v1/recommendations` as a **new `context` field** (today only
   `platform_product_id`) — backend + `recommendations-specialist` (Q-HA3).
4. **Product-preview-in-hook** (lever 2): the pre-intent slot, the refine-in-
   place swap, and the preview source wiring. New **and gated** on Q-HA1 (holdout
   measurement / G13) + Q-HA2 (preview source).
5. **Floating page-type adoption** (§5.4): page-type detection and rendering the
   adopted placement's hook/context in the drawer. New (behaviour change, Q-HA5).
6. **Trigger activation timing** (`immediate` / `scroll_depth` / `dwell` /
   `exit_intent`) config + widget logic, with the desktop-only exit-intent rule
   (§5.3). New config field + widget behaviour.
7. **Placement job/mode** as a config concept (§4.1), including the gated PDP
   `complete_the_look` mode control. New config + admin UI.
8. **Engagement-rate-per-placement surfacing** as the primary operational metric
   (§1.3): events exist, but presenting `widget_shown → widget_engaged` per
   placement as a pre/post product metric may need a small analytics/dashboard
   addition. Confirm scope (Q-HA7).

### Explicitly NOT in this quick win
- Complete-the-look / complementarity engine signal (gated, scoped elsewhere).
- New intent fields / question types (the "configurable questionnaire" is
  deferred, `hooks-and-packs.md` §6).
- Hook A/B experiments (roadmap item 6).

---

## 10. Open questions for the PO

| # | Question | Recommendation | Owner |
|---|---|---|---|
| **Q-HA1** *(blocking lever 2)* | **Product preview before intent breaks the "no pre-intent recs" rule and the holdout placebo (M1·2 D6, P1, G13).** How do we reconcile engagement lift with valid lift measurement? Options: (a) ship preview behind **G13** with server-side holdout handling (holdout shows an identical, non-WiseGift preview — e.g. the store's own collection products — so arms stay symmetric and WiseGift's lift is still isolated post-intent); (b) exposed-only preview, and accept engagement/Widget CTR is a product metric, not an A/A check; (c) preview = merchant's own collection order (not a WiseGift pick), identical in both arms, with WiseGift ranking kicking in only on refine. | **(c) then (a):** preview from a source that is identical in both arms (store's own collection / precomputed cold recs shown in both arms), so the pre-intent slot is not an arm difference and WiseGift lift stays measured on the refine step onward. Needs M1·2 owner sign-off. | PO + recommendations-specialist + `backend-engineer` (M1·2) |
| **Q-HA2** *(blocking preview content)* | **What populates the preview slot** (PLP: category cold recs? PDP: similar-to-product?), does it respect availability and a budget pre-filter, and is it cheap enough to serve on viewport entry for every PLP/PDP view? | Precomputed cold recs per collection (PLP) and existing similarity (PDP); no live LLM pre-intent, to stay within cost guardrails. | recommendations-specialist |
| **Q-HA3** | **`{category}` as a recommendation constraint** needs a new `context` dimension (collection handle/id) on `POST /widget/v1/recommendations` and a decision: **hard catalog filter** to the collection on PLP, or soft signal? Relates to the existing collection-filter follow-up (Q-R1). | Hard filter to the collection on PLP (the shopper declared it); soft person/occasion/budget on top. | recommendations-specialist + `backend-engineer` |
| **Q-HA4** | **Complete-the-look** PDP mode is dependency-gated on the complementarity signal scoped elsewhere. Confirm the quick win ships **`alternative` only**, with `complete_the_look` present-but-disabled in the admin. | Yes — ship alternative/similar; show complete-the-look as "coming soon". | PO |
| **Q-HA5** | **Floating tab adopts the page hook** (PLP-context on PLP, PDP-context on PDP, gift-concierge on home/other). This changes today's fixed floating drawer copy (`hooks-and-packs.md` §2.3). Accept the behaviour change? | Yes — it is the whole point of "the tab is a trigger, not a third content". | PO |
| **Q-HA6** | **Token fallback model:** per-field tokenless **fallback variant** (merchant authors it), or a **neutral-word** substitution (`{category}` → "this")? | Fallback variant for `headline`; neutral-word acceptable for `subline`. | PO |
| **Q-HA7** | **Engagement rate as the primary success metric:** confirm it is an **operational pre/post product metric per placement** (not a lift claim, consistent with Widget CTR being an A/A integrity check), and confirm whether the merchant dashboard should surface it per placement (small new view) or it stays an internal readout. | Internal readout per placement for the pilots; dashboard surfacing is a fast follow if merchants ask. | PO |
| Q-HA8 | **Context-token vocabulary (§6.1):** approve the MVP set `{category}` / `{product}` / `{store}` and the deferral of `{vendor}` / `{product_type}` / `{price}`. | Approve the 3-token set. | PO |
| Q-HA9 | **Trigger activation default:** keep `immediate` (today's behaviour) as default, or default to `scroll_depth` / `dwell` to reduce nag? And accept **exit-intent is desktop-only** with a `dwell` mobile fallback? | Keep `immediate` default; exit-intent desktop-only with dwell fallback. | PO |
| Q-HA10 | **PLP build priority:** the `collection_header` app block is net-new (the only new storefront surface here). Ship levers 1 & 3 on the existing placements first and PLP second, or hold the quick win until PLP exists? | Ship levers 1 & 3 on existing placements first; PLP block as the immediate fast follow. | PO + `frontend-engineer` |
| Q-HA11 | **Bootstrap-on-viewport + consent:** preview-first needs a rec call before interaction, which needs a session (consent-gated). Degraded shoppers get no preview. Confirm that is acceptable, and that bootstrap moving to viewport entry is in scope (relates to Q-C5 / Q-F1). | Acceptable: degraded shoppers see the copy-only hook; consented shoppers get the preview. | PO + `frontend-engineer` + `security-and-privacy` |

---

## 11. Relationship to existing specs
- This spec **extends** `hooks-and-packs.md` (per-placement hooks, packs) and
  relies on `hooks-and-packs-contract.md` (storage, bootstrap, resolution). It
  does **not** replace them.
- It inherits, unchanged: holdout symmetry (D6/P1), the event schema (no new
  types/fields), cost guardrails (no live LLM pre-intent), consent gating, and
  theming tokens.
- Once the PO validates the open questions, the relevant acceptance criteria
  should be folded into `spec.md` (Widget — Gift intent hook; Recommendation
  API; Merchant admin — Widget configuration / Placements) and the new rec-API
  context field + any measurement changes handed to `backend-engineer` and
  `recommendations-specialist`.
```
