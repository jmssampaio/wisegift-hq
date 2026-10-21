# Intent-form personalization — intent-adaptive copy + gender/audience filter

> **DRAFT — pending PO review (2026-10-21).** Written by `product-analyst` for
> the PO-approved direction from live testing: (1) the self/gift intent form
> should **adapt its field copy by intent mode** (the "interests" field reads
> wrong for SELF), (2) add a **gender / audience filter** shown conditionally by
> the merchant's vertical, mapped onto the same normalized `gender` signal that
> now filters PDP similars, and (3) keep **configurable "looks" / complete-the-
> look** as the deferred expansion. Nothing here is "done" until the PO
> validates it. Open questions for the PO are in §7, each with a recommendation.
>
> **Scope:** spec (user stories + acceptance criteria) and interaction/UX for the
> widget intent form + the admin config that drives it. No implementation. The
> product-data signal (gender/age columns, derivation) and the `/recommendations`
> filter are **flags/change-lists** for `backend-engineer` + `recommendations-
> specialist`, not settled here.
>
> **Inputs read:**
> - `docs/product/spec.md` — Widget — Gift intent hook; Intent capture form
>   (generic MVP); Widget configuration (intent form fields); Recommendation API.
> - `docs/product/hooks-and-packs.md` §3.2 (pack intent-question labels, option
>   lists, interests helper text), §5 Q-R1/Q-P4 (the deferred Fashion department
>   question), §6 Deferred (configurable questionnaire + looks).
> - `docs/engineering/hooks-and-packs-contract.md` §3.4 (`config.intentForm`),
>   §4.3 (store-type write path), §6 (events, holdout symmetry).
> - `docs/engineering/similars-gender-category-ranking.md` — the normalized
>   `gender` column proposal (`{men, women, unisex, unknown}`), the GenderClassifier,
>   the §2.1 gender-compatibility table, and the hard-filter mechanism.
> - `docs/engineering/budget-tiers.md` §1.7, §3 (the guided budget step; the
>   per-intent label pattern the audience field reuses).
> - `docs/product/hook-attractiveness-spec.md` (§4.2 PLP inherits category;
>   §4.3 complete-the-look gated) and `docs/product/pdp-similars-redesign.md`
>   (§4 complete-the-look as a deferred second module).
> - Widget code (read-only, `../wisegift-widget`): `src/components/SelfIntentForm.tsx`,
>   `GiftIntentForm.tsx`, `src/types/intent.ts`, `src/api/types.ts` (`Intent`,
>   `SessionConfig.intentForm`), `src/widget.tsx`.
> - Backend state via `similars-gender-category-ranking.md` §1 (gender column NOT
>   yet present; `product_type` + `tags` are).

---

## 0. TL;DR for the PO

- **The "interests" field already has the right per-intent label in the config**
  (`self.interests.label = "What are you looking for?"`, `gift.interests.label =
  "What are they into?" / "Anything about their style?"`), **but the widget
  ignores it** and hardcodes "Interests (optional, up to 5)" for both modes. The
  first fix is small and mostly wiring: render the field label/helper from
  `config.intentForm[mode]` so copy adapts self vs gift. §2 gives the exact copy.
- **Add one shopper-facing "Audience" field** (Women / Men / Kids / Baby / Any),
  shown **only for apparel-type stores**, defaulting to **Any (no filter)** so we
  never wrongly constrain. It is the live-path twin of the PDP similars gender
  filter and the long-promised fix for the Fashion "menswear-to-a-female-partner"
  risk (hooks-and-packs Q-R1/Q-P4). §3.
- **Yes, the gender column needs extending for baby/kids** — but not by cramming
  age into `gender`. Keep `gender {men, women, unisex, unknown}` as the similars
  doc defines it and **add a separate orthogonal `age_group {adult, kids, baby,
  unknown}`** column, derived the same way at ingest. The Audience field composes
  the two. §3.4.
- **Conditional display** keys off the merchant's store type (`vertical_pack` /
  `vertical`): a default-visibility table (Fashion → shown, Home & Decor /
  General → hidden) plus the existing per-field merchant override in
  `intent_form_fields`. §1.
- **Looks / complete-the-look stays DEFERRED.** It is a results-side capability
  needing a complementarity signal, not an intent-form field. §5 sketches where
  it slots and flags the dependency.
- **Holdout + 20-second budget are preserved:** the audience field is one
  optional tap, identical in both arms, never reads `isHoldout`.

---

## 1. The conditional field model (the matrix)

The intent form has two axes that decide what renders: **intent mode** (self vs
gift, chosen at the hook) and **merchant vertical** (store type). A third axis,
the merchant's **`intent_form_fields` visibility toggle** (required / optional /
hidden), overrides on top — unchanged from spec "Widget configuration".

### 1.1 Fields by intent mode

| Field | Self | Gift | Notes |
|---|---|---|---|
| **relationship** | hidden | shown (required default) | Buying for yourself has no recipient. Already self-hidden in the widget today. |
| **occasion** | hidden | shown (required default) | Same reason. Gift-centric by nature. |
| **audience** (NEW) | shown* | shown* | *only when the vertical enables it (§1.2). Copy differs self vs gift (§2). |
| **budget** | shown (guided tiers, no suggestion) | shown (guided tiers + suggestion) | Unchanged from `budget-tiers.md`; self gets no pre-selected tier (§2.5 there). |
| **interests** | shown (reframed copy) | shown | The field the PO flagged. Copy adapts by mode (§2). |

- **Self is deliberately short** (audience? + budget + interests), gift is the
  fuller flow (relationship + occasion + audience? + budget + interests). This
  preserves the < 20 s target and the existing "progressive disclosure in the
  gift flow only" decision (2026-10-07).
- relationship and occasion stay **gift-only**. There is no sensible self
  equivalent, and inventing one ("occasion for yourself") adds friction for no
  ranking signal. Recommendation: do **not** add self-side relationship/occasion.

### 1.2 Audience-field visibility by vertical (default, merchant-overridable)

The audience field is the only field whose **default visibility depends on the
store type**. It is derived from the tenant's store signal, then overridable by
the merchant like any other `intent_form_fields` entry.

| Store signal | Default audience visibility | Why |
|---|---|---|
| `vertical_pack = fashion` | **optional, shown** | Apparel is gendered/aged; this is the core use case the PO described. |
| `vertical = kids` or `jewelry` | **optional, shown** | Kids stores need the age split; jewelry is often gendered. |
| `vertical = beauty` | hidden (merchant may enable) | Some beauty is gendered (fragrance), much is not. Off by default, one toggle to enable. |
| `vertical_pack = home_decor` | hidden | Home goods have no gender/age department. The PO's explicit "home-decor doesn't need it" case. |
| `vertical_pack = general` (and all other verticals) | hidden (merchant may enable) | No assumption; a general store can turn it on if its catalog is gendered. |

- **Mechanism:** the default is a code constant per vertical/pack (same pattern
  as `HookCopyDefaults` and the suggested-look tokens). The merchant's
  `intent_form_fields.audience` override, when present, wins — exactly the
  defaults-layer model used for packs ("an override existing is the edit").
- **Recommend tying the default to `vertical_pack` primarily** (3 clean values),
  with the 9-value `vertical` consulted only for the kids/jewelry/beauty nuance
  above. Rationale: `vertical_pack` is the signal already wired into copy
  resolution; `vertical` is finer but sparsely used. See Q-IF4.
- **Visibility values for audience:** `hidden | optional`. **Not `required`** by
  default — a required department question adds a mandatory tap to every session
  and risks the 20 s budget and a wrong forced choice. A merchant with a strictly
  single-department catalog could be allowed `required`; recommend deferring that
  until asked (Q-IF5).

### 1.3 Placement interaction

- The audience field lives in the **post-hook intent form**, so it appears on the
  discovery surfaces that lead with the hook (`home_hero`, `gift_finder`,
  `floating`, and `collection_header` when it ships) and in the opt-in gift flow.
- **PDP is already covered by the product, not the form.** Per
  `pdp-similars-redesign.md`, the PDP leads with similars and no longer leads with
  the intent question; the PDP similars are **already gender-filtered by the
  product's own `gender`** (similars doc §2.1). So the shopper does not need to
  pick an audience on a PDP — it is inherited from the product. The audience form
  field therefore does not render on `pdp_slot`.
- **Collection context can pre-fill audience.** On a gendered collection (e.g.
  "Women's dresses"), the inherited category constraint (hook-attractiveness
  Q-HA3) could pre-select the audience, so the shopper never restates it. Keep
  this as a **refinement, not MVP**: at MVP the audience defaults to Any on PLP
  too, and the category filter does the gendering. Flag as Q-IF6.

---

## 2. Reframed copy per field, self vs gift (exact microcopy)

EN is authoritative; ES/PT follow the existing pack translation pass (Q-P6).
These are the **pack/config-driven** labels — the widget must render them from
`config.intentForm[mode]`, not hardcode them (the fix in §4).

### 2.1 Interests — the field the PO flagged

The word "interests" reads as *gift research* ("what are they into?"). For a
shopper buying for themselves it is the wrong frame — they are describing **what
they want / their own taste**, not someone's hobbies.

| Pack | Gift label + helper | Self label + helper (REFRAMED) |
|---|---|---|
| General | **"What are they into?"** — e.g. cooking, hiking, coffee, board games | **"What are you looking for?"** — e.g. something for the weekend, a new hobby |
| Fashion | **"Anything about their style?"** — e.g. minimal, earthy colours, loves linen, sporty | **"What's your style?"** — e.g. a light jacket for spring, black ankle boots, minimal |
| Home & Decor | **"Any room or style in mind?"** — e.g. bedroom, Scandinavian, warm lighting | **"What are you looking for?"** — e.g. a reading lamp for the living room, linen bedding |

- The Fashion/self label moves from the generic "What are you looking for?" to
  **"What's your style?"** to match the self-shopping-for-clothes mental model.
  (General/Home stay "What are you looking for?" — "style" is apparel-specific.)
  This is a small copy delta on top of the pack labels already in
  `hooks-and-packs.md` §3.2; see Q-IF1.
- The chip input, 5-chip max, and privacy notice are unchanged. Only the **label
  + helper** adapt. The widget currently shows a fixed label "Interests
  (optional, up to 5)" and placeholder — both must come from config per mode.

### 2.2 Budget — already adaptive, no change

`"What's your budget?"` for both modes (budget-tiers.md §3.3). Self gets no
suggested tier; gift gets the relationship × occasion suggestion. No change here;
listed for completeness.

### 2.3 Relationship / occasion — gift only, no self variant

`"Who's it for?"` and `"What's the occasion?"` (gift). Not shown for self. No
reframe — they are inherently gift fields and the right call is to hide them, not
reword them, for self.

### 2.4 Audience (NEW) — copy differs by mode

The same product filter, framed differently depending on who it is for.

| Mode | Label | Options (shopper-facing) | Pre-selected |
|---|---|---|---|
| **Gift** | **"Who are you shopping for?"** | A woman · A man · Kids · Baby · Anyone | **Anyone** (= no filter) |
| **Self** | **"Which section?"** | Women's · Men's · Kids' · Baby · Any | **Any** (= no filter) |

- Gift framing is person-led ("a woman"); self framing is department-led
  ("Women's") because the shopper is the person. Both map to the same underlying
  filter (§3.3).
- **"Anyone" / "Any" is a real, selectable option and the default**, so the
  shopper can leave it untouched (no constraint) and the form still submits in
  one tap. It is not a blank — the neutral option is explicit so the control
  reads as answered.
- Kids and Baby are included per the PO's steer; see §3.1 for the option-set
  recommendation and §3.4 for the data that backs Kids/Baby.

---

## 3. The gender / audience filter

### 3.1 Option set and default — recommendation

**Recommended option set: Women · Men · Kids · Baby · Any (5 options).**

- **Include Kids and Baby** (the PO asked to consider them). They are a genuine,
  high-value split for apparel: a parent shopping for a child must not be shown
  adult clothing, and baby is a distinct, frequently-separate department.
- **Include an explicit "Any / Anyone"** as the neutral default. Do **not** rely
  on "no selection" as the neutral state — an explicit Any option is clearer and
  accessible (it is a real radio choice, not an empty group).
- **No standalone "Unisex" shopper option.** Unisex is a *product* attribute, not
  a shopper intent — a shopper does not shop "the unisex section". Unisex products
  surface under **both** Women and Men (the compatibility rule, §3.3), exactly as
  the similars doc handles unisex. Exposing "Unisex" as a shopper pick would be
  confusing and would under-serve (it would hide gendered items). See Q-IF2.
- **Default: Any (no filter).** Fail-open, matching the similars doc's
  "unknown source → no gender constraint" and the budget step's "Any price". We
  never silently constrain a shopper who did not ask to be constrained.
- **Display order:** Women, Men, Kids, Baby, Any. (Women/Men first as the largest
  apparel segments; Any last as the escape. The merchant does not reorder at MVP.)
- **Control:** a single-select pill / segmented group (native radio semantics),
  44 px targets, one tap, keyboard + SR labelled. It sits **above budget** in
  both forms (department before price is the natural narrowing order).

### 3.2 It is optional and one tap (20 s budget)

- The field is **optional** (default Any), so it never blocks submit and never
  adds a required step. A self shopper can submit with budget + audience Any +
  no interests in one or two taps, as today.
- One selection, no free text, no second screen. It costs at most one tap.

### 3.3 Mapping to the product signal + the recommendation filter

The shopper's Audience pick maps to a hard pre-ranking filter over the product's
normalized `gender` and `age_group` (§3.4), mirroring the similars §2.1
gender-compatibility table and extending it with age:

| Audience pick | Eligible product `gender` | Eligible product `age_group` |
|---|---|---|
| Women | women, unisex | adult |
| Men | men, unisex | adult |
| Kids | any gender (boys/girls/unisex all ok) | kids |
| Baby | any gender | baby |
| Any | no constraint | no constraint |

- **Hard filter, pre-LLM**, in the retrieval candidate `WHERE` — the same place
  the similars gender predicate lives (`NearestProductFinder` candidate filter).
  A closer-but-wrong-department product can never outrank a correct one.
- **Unisex products appear for both Women and Men** (compatibility), consistent
  with the similars doc.
- **Coverage / unknown handling (recommendations-specialist call, Q-IF3).** When
  a filter is set and the gender/age-correct pool is too thin to fill the slate:
  do **not** pad with the *opposite* gender, but it is reasonable to **relax to
  include `unknown`** products rather than return a near-empty slate (a store with
  poor tagging should still get results). Opposite gender stays excluded always.
  This differs from the PDP similars "return the short correct slate" rule because
  the live discovery slate has a stronger "fill the row" expectation; the ranker
  then re-orders. The exact relaxation threshold is the specialist's to set and
  measure (gender-coherence check, similars §6).
- **Cache signature:** the audience pick must enter the intent cache signature,
  or cached slates will leak across audiences (a Women request serving a cached
  Men slate) — the same class of bug as the budget-bucket leak
  (`budget-tiers.md` §5.2, Q-R3). Flag to `recommendations-specialist` +
  `backend-engineer`.

### 3.4 Does the gender column need extending for baby/kids? — YES

**Recommendation: keep `gender` as the similars doc defines it and add a
separate orthogonal `age_group` column. Do not overload `gender`.**

- `gender VARCHAR(10)` — `{men, women, unisex, unknown}`. Already proposed in
  `similars-gender-category-ranking.md` §1.3; this work **depends on and shares**
  that column (same signal filters PDP similars and the live path). Do not
  duplicate.
- **NEW: `age_group VARCHAR(10)`** — `{adult, kids, baby, unknown}`, default
  `'unknown'`, derived at ingest by an `AgeGroupClassifier` that mirrors the
  `GenderClassifier` pattern (tags → product_type synonym sets, else unknown):
  - baby ← `baby, newborn, infant, toddler, 0-3m, 6-12m, nb`
  - kids ← `kids, children, child, junior, youth, boys, girls, teen`
  - adult ← `adult, mens, womens, men, women` (or: default to `adult` when a
    gender is confidently men/women and no kid/baby signal — specialist's call)
  - unknown ← nothing matches
- **Why two columns, not one `audience`/`department` enum.** Gender and age are
  **orthogonal**: a "girls' dress" is *female* **and** *kids*; a "boys' coat" is
  *male* **and** *kids*. A single flat enum `{women, men, kids, baby, unisex}`
  cannot express "girls" (kids + female) and would force Kids to be gender-blind
  forever, blocking a future boys/girls split. Two columns keep the signal clean,
  reusable (the live ranker and the PDP similars both want it), and extensible.
  The flat single-column alternative is simpler to ship but loses that — I
  recommend against it; see Q-IF7.
- **Caveat (same as the gender column):** this becomes a **catalogue-data quality
  problem, surfaced cleanly instead of hidden**. A store that tags nothing gets
  `unknown` everywhere and the filter degrades to no-op (fail-open). That is the
  right failure mode and it is measurable (the similars §6 coherence/coverage
  checks extend to age). The catalogue checklist is the lever to fix it.
- **Derivation joins the content hash** (like gender) so a re-tag re-derives and
  re-triggers the relevant precompute; it is a *filter*, not part of the embedding
  vector.

---

## 4. Backend / widget / admin implications + change list

> Logical names. The backend owns the wire contract (`architecture.md` §9.2 and
> `hooks-and-packs-contract.md` §3.4). `recommendations-specialist` owns the
> filter semantics and the quality bars.

### 4.1 Product data + recommendations (backend-engineer + recommendations-specialist)

| # | Change | Where | Notes |
|---|---|---|---|
| D1 | Depend on / reuse the normalized `gender` column + `GenderClassifier` | `tenant_products`, catalog ingest | Shared with similars doc; do not duplicate. This work is **blocked on** that landing. |
| D2 | Add `age_group VARCHAR(10) NOT NULL DEFAULT 'unknown'` + `AgeGroupClassifier` at ingest; add to content hash | new Flyway migration, catalog/platform-integrations | §3.4 vocab |
| D3 | `/widget/v1/recommendations` intent accepts optional `audience` (canonical `women/men/kids/baby` or absent=Any); allow-list + canonicalise at the boundary, unknown → null (never 400) | recommendation API / intent DTO | Mirrors the Q-F3 occasion-canonicalisation pattern |
| D4 | Hard pre-ranking filter mapping `audience` → (`gender`, `age_group`) compatibility (§3.3), in the retrieval candidate `WHERE` | `NearestProductFinder` / retrieval | Same place the similars gender predicate lives |
| D5 | Coverage relaxation rule (thin pool → include `unknown`, never opposite gender) | retrieval | recommendations-specialist to set threshold + measure |
| D6 | Add `audience` to the intent **cache signature** | `SignatureComposer` | Prevents cross-audience cache leaks (budget-tiers §5.2 analog) |
| D7 | Gender/age-coherence + coverage checks for the live path | eval plan | Extend similars §6 bars; "unknown" share target |

### 4.2 Widget config contract (backend-engineer)

| # | Change | Where | Notes |
|---|---|---|---|
| C1 | `config.intentForm.{gift,self}` gains an **`audience`** block: `{ label, options[] (value+label, ordered), visibility }` | bootstrap `config.intentForm` | Same shape family as `relationship`/`occasion`. Options already mapped to the canonical values. |
| C2 | Per-intent **labels/helpers must be delivered and used** for `interests` (and all fields), self vs gift | bootstrap `config.intentForm` | Config already carries `self.interests.label`; the widget must stop hardcoding it (W1). |
| C3 | Audience **default visibility by vertical** (code constant, §1.2), overridable by `intent_form_fields.audience` | config resolution | Defaults-layer model, like packs |
| C4 | Form-schema version bumps when the audience field is added/changed | `intent_form_schemas` | So `intent_submitted` events segment correctly (events already carry `intentFormSchemaVersion`) |

### 4.3 Widget (frontend-engineer)

| # | Change | Where | Notes |
|---|---|---|---|
| W1 | **Render field labels/helpers from `config.intentForm[mode]`** instead of the hardcoded "Interests (optional, up to 5)" / placeholder | `SelfIntentForm.tsx`, `GiftIntentForm.tsx` | This is the core "copy adapts by intent" fix. Today both forms hardcode the interests label and only read budget copy from config. |
| W2 | New **Audience picker** (single-select pills, radio semantics, 44 px, SR-labelled), rendered when `config.intentForm[mode].audience.visibility !== "hidden"`, above budget | new picker component | Default = Any; one tap |
| W3 | Send `audience` in the intent payload on `/recommendations` (omit when Any/absent) | `types/intent.ts` `Intent`, request builder | Add optional `audience?: string` to `Intent` |
| W4 | Holdout symmetry: audience field identical in both arms; never read `isHoldout`; it is captured before `intent_submitted` | widget | Preserves lift validity (budget-tiers §2.7 pattern) |
| W5 | Do not render audience on `pdp_slot` (inherited from product per PDP redesign) | widget | §1.3 |

### 4.4 Admin (frontend-engineer + product-analyst)

| # | Change | Where | Notes |
|---|---|---|---|
| A1 | Add **"Gender / Audience"** as a 5th row to the **Intent form fields** card (Configuration), alongside relationship / occasion / budget / interests, with a visibility toggle (`Hidden` / `Shown (optional)`) | `configuration-screen.tsx` intent-fields card | Reuses the existing visibility-toggle pattern (spec "Widget configuration") |
| A2 | Default the toggle from the store type (§1.2); show a one-line helper: "Shown automatically for fashion stores. Turn on if your products are split by department." | admin | Makes the vertical-default legible |
| A3 | Show the audience options + the reframed per-intent labels **read-only** ("Set by your store type"), like the other intent questions | admin | Editing option labels is deferred (Q-IF8) |
| A4 | The intent-form-fields card is behaviour/content → stays on **Configuration**, not Appearance (per `admin-appearance-ia.md` split) | admin | Consistency with the IA |

---

## 5. Looks / complete-the-look — the deferred expansion

**Status: DEFERRED, confirmed. Not an intent-form field.** The PO wants it; it
stays the deferred expansion because it needs a **complementarity signal**
(co-purchase / curation / an LLM "style-a-look" pass), which is a different signal
from similarity and is scoped elsewhere (`hook-attractiveness-spec.md` §4.3,
`pdp-similars-redesign.md` §4, `recommender-eval-plan.md` §6).

Where it will slot, relative to this work (sketch only):

- **Not in the intent form.** Looks are a *results-side* capability (ranking over
  sets of complementary products), not a question we ask the shopper. Adding an
  audience filter does **not** bring looks closer — but it **feeds** it: a
  complete-the-look for a women's dress should stay within women's/unisex
  complements, so the same `gender` + `age_group` signal this work adds is a
  **precondition** that makes looks coherent when it lands.
- **Where it renders:** as a **second results module** ("Complete the look" /
  "Goes well with this") below the primary recommendations / similars — the exact
  slot already sketched in `pdp-similars-redesign.md` §4.2, and as a possible
  post-answer "styled looks" result mode on the discovery flows.
- **Admin:** reuses the already-drafted, present-but-disabled `complete_the_look`
  PDP mode control (`hook-attractiveness-spec.md` §4.3) — "coming soon" until the
  complementarity signal lands. No new admin concept here.
- **Dependency gate:** the complementarity signal + its own labelled evaluation
  set. Do not build the looks UI before that signal exists and is measured.

So: this spec **prepares the ground** for looks (the gender/age signal keeps a
look within the right department) but does **not** design or schedule it.

---

## 6. Interaction flow (the adapted forms)

```
Hook: "for you or for someone else?"
   │
   ├── For me  ──────────────► SELF form
   │                            [ Which section?  Women's · Men's · Kids' · Baby · (Any) ]   ← audience, if vertical enables; default Any
   │                            [ What's your budget?  guided tiers · Any price ]
   │                            [ What's your style? / What are you looking for?  chips ]     ← reframed interests
   │                            → Find recommendations
   │
   └── For someone else ─────► GIFT form
                                [ Who's it for?  relationship pills ]
                                [ What's the occasion?  occasion pills ]
                                [ Who are you shopping for?  A woman · A man · Kids · Baby · (Anyone) ]  ← audience, if vertical enables; default Anyone
                                [ What's your budget?  guided tiers (+ suggestion) ]
                                [ + Add more context → Anything about their style?  chips ]   ← reframed interests
                                → Find gifts

On submit: audience (if not Any) → hard (gender, age_group) filter on /recommendations,
           identical in both holdout arms. PDP slot does NOT show audience (inherited from product).
```

---

## 7. Open questions for the PO

| # | Question | Recommendation | Owner |
|---|---|---|---|
| **Q-IF1** | Approve the reframed self interests copy — General/Home "What are you looking for?", Fashion "What's your style?" — and the self vs gift split in §2? | Approve. It is the fix the PO asked for and sits on top of existing pack labels. | PO |
| **Q-IF2** | Audience option set **Women / Men / Kids / Baby / Any**, with no shopper-facing "Unisex" pick (unisex products surface under both Women and Men)? | Approve. Unisex is a product attribute, not a shopper intent. | PO |
| **Q-IF3** | Thin-pool behaviour when a gender/age filter is set: relax to include `unknown` products (never opposite gender) vs. return a short slate (PDP-style)? | Relax to include `unknown` for the live discovery slate; keep opposite gender excluded. Specialist sets the threshold. | PO + recommendations-specialist |
| **Q-IF4** | Drive the audience **default visibility** off `vertical_pack` (3 values) with `vertical` only for the kids/jewelry/beauty nuance? | Yes — `vertical_pack` is the wired signal; `vertical` refines. | PO + backend-engineer |
| **Q-IF5** | Allow `required` audience for strictly single-department catalogues, or `hidden`/`optional` only at MVP? | `hidden`/`optional` only at MVP; add `required` if a merchant asks. | PO |
| **Q-IF6** | On a gendered collection, pre-fill audience from the inherited category (hook-attractiveness Q-HA3) — MVP or refinement? | Refinement, not MVP. At MVP the category filter does the gendering; audience defaults Any on PLP. | PO + recommendations-specialist |
| **Q-IF7** | Data model: separate orthogonal `gender` + `age_group` columns (recommended) vs. a single flat `audience`/`department` enum? | Two columns. Single enum can't express "girls" (kids + female) and blocks a future boys/girls split. | PO + backend-engineer |
| **Q-IF8** | Let merchants edit the audience **option labels** (like a future configurable questionnaire), or keep them fixed at MVP? | Fixed at MVP; editing is part of the deferred configurable questionnaire. | PO |
| **Q-IF9** | This audience/gender filter is effectively the resolution of the deferred Fashion department question (hooks-and-packs Q-R1/Q-P4), via a normalized product attribute instead of merchant-selected collections. Confirm it supersedes the "merchant-selected Shopify collections" plan for the department split? | Supersede with the normalized attribute — it is lower-merchant-effort and reusable by the live ranker + PDP similars. Keep collection-based filtering for non-gender departments only. | PO + recommendations-specialist |

---

## 8. Relationship to existing specs

- **Extends** `spec.md` "Widget — Intent capture form (generic MVP)" and "Widget
  configuration" (adds the audience field + its toggle) and "Recommendation API"
  (adds the `audience` intent field + hard filter). Once the PO validates §7, the
  accepted ACs fold into those features and the data-model + filter changes go to
  `backend-engineer` and `recommendations-specialist`.
- **Depends on** `similars-gender-category-ranking.md` (the `gender` column +
  classifier) — this work shares that signal and adds the orthogonal `age_group`.
- **Reuses** `budget-tiers.md` §3 (the per-intent label + holdout-symmetry
  pattern) and `hooks-and-packs-contract.md` §3.4 (`config.intentForm` shape).
- **Resolves** the direction of hooks-and-packs Q-R1/Q-P4 (the Fashion department
  question) via a normalized product attribute (Q-IF9).
- **Prepares, does not design,** looks / complete-the-look (§5): the gender/age
  signal keeps a future look within the right department.
```
