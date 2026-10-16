# M1·2 — Lift measurement spec (exposed vs holdout)

> **Status:** DRAFT for PO validation. Owner: `recommendations-specialist`.
> Input contract for `backend-engineer` (M1·2: attribution join + KPI service +
> `GET /admin/v1/analytics`) and for admin M2·6 (analytics dashboard).
> Nothing here is settled policy until the PO signs off; `decisions.md` is not
> edited by this draft.
>
> Sources read: `mvp-build-plan.md` (M1·2), `architecture.md` §7–§9,
> `spec.md` (Attribution, Analytics dashboard, Event API, Order webhook),
> `decisions.md` (2026-09-29 Frozen MVP scope, Widget consent for EU,
> Architecture open questions #6; 2026-10-04 PR 6 plan; 2026-10-14 entries),
> `docs/core/recommendations.md` §2. Code read-only:
> `wisegift-backend@origin/develop` (ff0b2a7) and
> `wisegift-widget@origin/main` (b0a4c00). The staging DB was **not** queried;
> the Neon tools were not available in this session.
>
> **Revision 2026-10-14 (after D1–D9 were accepted):** a product-analyst review
> found that pre-intent PDP recs can bias D2's `triggered` population. The
> resolution is in §1.4, along with two follow-up proposals (§2.1 CTR, §3.1
> URL markers). All three are marked **ACCEPTED by PO 2026-10-15** and listed in
> §9 as P1–P3. The code was re-read at `wisegift-widget@origin/main` (0abf63d,
> after PR #17) and `wisegift-backend@origin/develop` (cf2a959).

---

## 0. TL;DR

1. **The randomisation unit is the widget session.** That is the anonymous `wg_session` UUID, sticky for up to 30 days. Every KPI is computed per session and compared across the arms. No event-level ratios.
2. **The headline population is triggered sessions**: sessions with at least one *trigger*, meaning the first moment the two arms can differ. Today that moment is `intent_submitted`, which is D2 as accepted. Exposed and holdout behave identically up to and including the trigger, and they diverge right after it (recs vs placebo). This is the most powerful comparison that is still apples-to-apples. The store-level view (all `widget_shown` sessions) is returned alongside it. **ACCEPTED by PO 2026-10-15 (P1, §1.4):** once a placement shows recs *before* any intent (the spec'd 3-second PDP auto-render), a `widget_shown` on that mount also counts as a trigger. Otherwise the triggered set would be selected by the treatment itself.
3. **Lift is real only for conversion, AOV and revenue-per-session.** Widget CTR cannot move before the arms diverge, because the hook is identical in both arms. It becomes an **integrity check** (an A/A guardrail), not a lift tile (**ACCEPTED by PO 2026-10-15, P2**). Add-to-cart on recommended products **cannot be measured at MVP**: there is no cart signal, and holdout has no recommended products. It is replaced by an exposed-only diagnostic. **PO decision needed.**
4. **The 7-day window is anchored on the session's last *symmetric* widget event** (`widget_shown` / `widget_engaged` / `intent_submitted`), **not** on `product_clicked`. Holdout can never fire `product_clicked`, so anchoring on it would hand the exposed arm attribution that holdout cannot get.
5. **One statistical engine for every KPI.** Each KPI is a per-session mean (binary KPIs are means of 0/1). We use Welch's z on the difference, and a delta-method CI on relative lift. AOV is a ratio metric with a delta-method variance at the session level. Revenue is **winsorised at the pooled P99 order value**.
6. **5,000 exposed sessions is a floor, not a power guarantee.** At that size we can only detect lifts of roughly +40% to +90% relative (§5.4). Realistic +10% lifts need about 165k–280k triggered sessions at 90/10. Pilots will mostly show grey tiles. **PO tradeoff needed.**
7. **Blocking gap:** today, **neither arm can be attributed in practice.** The widget writes no cart attribute. Holdout produces no product clicks, so it never gets a URL marker. The URL marker on exposed clicks most likely never reaches `landing_site` either (§7, G1). Unless that is fixed, M1·2 will compute lift on (near-)zero attributed orders, or worse, on an exposed-only sample.

---

## 1. Units, arms, populations

### 1.1 Unit of analysis
- **Session** = one `sessions` row = one anonymous `wg_session` UUID per browser per merchant origin. It is randomised by `HoldoutAssigner` (SHA-256 of the UUID, mod 100 < 10 → holdout), sticky, and never re-rolled.
- A "session" here is a **browser over up to 30 sliding days, not a visit**. All per-session outcomes accumulate across visits.
- Arm membership is read from `sessions.is_holdout` (server-authoritative). It is never read from the client `holdout` flag.

### 1.2 Event symmetry (what each arm actually emits today)

| Event | Exposed | Holdout | Symmetric? | Notes (widget `main`) |
|---|---|---|---|---|
| `widget_shown` | yes | yes | **yes** | Once per mount, on IntersectionObserver viewport-enter. Several placements on one page → several events. |
| `widget_engaged` | yes | yes | **yes** | First hook click per mount. Hook is identical in both arms. |
| `intent_submitted` | yes | yes | **yes** | Fired before `flow.submit`, i.e. before the arms diverge. |
| (divergence point) | recs rendered | placebo = "no matches" empty state, no `/recommendations` call (D6 will replace it with a neutral collapsed slot) | — | |
| `product_clicked` | yes | **never** | **no** | Only exists when recs exist. |

**Consequence:** any metric or window anchor that involves `product_clicked` is **exposed-only by construction**.

**Re-checked at `origin/main` 0abf63d:** this table still holds on **every** placement, PDP included. The widget calls `/recommendations` only from `flow.submit`, which runs after `intent_submitted`. There is no 3-second timer and no viewport-triggered fetch. The theme block does not pass a PDP product id (`contextPlatformProductId` is never set from Liquid). The spec and `architecture.md` §5 / §10.2 describe a pre-intent PDP path that the code does not have yet (see §1.4).

### 1.3 Populations (per range, per placement filter)
- **`shown`**: sessions with ≥ 1 `widget_shown` in range. This is the store-level / intent-to-treat view and matches `spec.md`'s literal denominator.
- **`triggered`** (headline): sessions with ≥ 1 **trigger event** in range. This is the standard *triggered analysis*. A trigger event is the first point at which the arms can differ, and it is emitted identically in both arms:
  - `intent_submitted`, on any placement (D2 as accepted); and
  - **ACCEPTED by PO 2026-10-15 (P1, §1.4):** `widget_shown` on a mount that renders recs before any intent. The event carries `pre_intent_recs = true` for these mounts.

  No session can be treated before its first trigger, and nothing before it differs between arms. So membership in this set is unaffected by treatment, and the comparison is unbiased. It also discards sessions that could never have been affected (pure dilution). Until a pre-intent placement ships, no `widget_shown` carries the flag, and the rule is exactly D2.
  - Mapping to the PO's wording: `decisions.md` says "conversion on engaged sessions" and `spec.md` says "/ widget_shown sessions". Both are valid but diluted versions of `triggered`. **D2 (accepted):** `triggered` is the headline and `shown` is the secondary view.
- **Trigger time** `t₀(s)` = the first trigger event of the session inside the range, **across all placements**. For `shown` it is the first `widget_shown`. For `triggered` it is the first trigger event as defined above.
- **Placement filter (ACCEPTED by PO 2026-10-15, part of P1):** a session belongs to placement *p*'s population if its **first** trigger event (the one at `t₀`) happened on *p*.
  - The previous rule used the first trigger *on p*. That rule breaks once pre-intent recs exist. Example: a shopper sees PDP recs on Monday and submits the home-page form on Thursday. Under the old rule, the session joins `home_hero` at Thursday's trigger, but it was already treated on Monday. Whether it ever reaches `home_hero` may then depend on Monday's treatment.
  - First-trigger attribution is decided before any treatment, so it stays unbiased. A side benefit: placements now **partition** the sessions, so per-placement session counts add up to `placement=all`.
  - Outcomes are still **all** attributed orders of the session. The placement is "where the session entered the experiment", not "which placement caused the order". Per-placement views remain labelled *exploratory* (§5.5).
  - `placement=all` is the default.

### 1.4 Pre-intent recommendations (PDP) and the trigger rule — PROPOSED, pending PO (P1)

**Issue raised by product-analyst (2026-10-14).**
- `spec.md` ("Widget — Gift intent hook") says PDP placements show context recs after about 3 s with no intent capture. `architecture.md` §5 (serving-policy step 3) and §10.2 (step 3, fetch on viewport entry) say the same.
- On that path the exposed arm is treated **before** any `intent_submitted`. Under D2 as written, whether a session ever submits intent can then depend on the arm:
  - auto-recs may satisfy a shopper who would otherwise have used the form, or
  - they may prime a shopper to engage.
- Either way, filtering on "submitted intent" compares two differently selected groups. This is post-treatment conditioning, and its direction is unknown. It also leaks into `placement=all`, because one session spans placements.

**Verdict.**
- **Today: no bias.** At `wisegift-widget@origin/main` 0abf63d, no placement renders recs before `intent_submitted` (§1.2). The backend's no-intent branch (`RealIntentCheck` → `served_from="precomputed"`) exists, but the widget never reaches it without a submitted form. D2 is unbiased for the code as it stands.
- **Latent and real.** The pre-intent PDP path is an accepted acceptance criterion. The day `frontend-engineer` implements it, D2's `triggered` population becomes biased, on PDP and also on every other placement those sessions later touch. This is why it has to be fixed in the definition now, not after a pilot readout.

**Rule (P1).**
1. **A trigger is the first moment the arms can differ, logged identically in both arms.** For placements without pre-intent recs (`home_hero`, `gift_finder`, `floating`, and `pdp_slot` today), that moment is `intent_submitted`. For a mount that renders recs before intent (`pdp_slot` once the 3-second path ships), it is that mount's `widget_shown`.
2. **The widget marks such mounts explicitly.** `widget_shown` carries `pre_intent_recs: true`. The flag is computed from placement and config only, **before the holdout bucket is known**, so it is identical across arms. The backend events schema must accept the field (the schema is strict). Reading the flag is safer than hard-coding "`pdp_slot` means pre-intent": it stays correct across widget versions and if the behaviour later becomes merchant-configurable.
3. **Session level:** `t₀` = the earliest trigger of either kind, on any placement (§1.3). The placement filter uses the placement of that first trigger (§1.3).
4. **Sequencing gate:** the 3-second PDP auto-render must not reach a pilot storefront without the flag and the matching backend change. Shipping one without the other silently biases every readout from that day on.

**What the holdout sees on the pre-intent path (consistent with D6).**
- The holdout widget runs **the same code path and timer** as exposed: same mount, same `widget_shown` (with the same flag), same 3-second wait.
- When exposed renders the context recs, the holdout rec slot **stays collapsed**: zero height, no copy, no "no matches", no spinner, and **no `/recommendations` call**.
- **The hook stays exactly where and how it is in the exposed arm.** If the exposed layout moves or shrinks the hook when recs appear, the difference is part of the treatment. The holdout must not imitate it.
- A holdout shopper who later submits intent gets the D6 neutral collapsed slot, as today.
- Server-side enforcement (G8) must cover the no-intent request as well: a holdout session gets an empty slate whether or not intent is present.

**Cost of this rule (stated plainly).**
- On PDP-heavy stores, `triggered` will be close to `shown`, because most PDP views become triggers. It includes shoppers who scrolled away before the 3 seconds passed and never saw a rec.
- That is **dilution, not bias**. The lift estimate stays honest, but the per-session effect is smaller, so readouts need more sessions (§5.4).
- A tighter trigger (a dedicated symmetric event fired at the 3-second mark in both arms) would cut the dilution. It needs a new event type and more widget logic. I don't recommend it for MVP. Revisit only if PDP readouts turn out too slow.

**Alternatives considered and rejected.**
- *Report PDP on `shown` and other placements on `triggered`, keeping the intent-only trigger.* This fixes the PDP tile but not `placement=all`: a session treated on a PDP and later triggered on the home page is still selected by treatment. The proposed rule is the general form of this idea.
- *Show the precomputed PDP recs to the holdout too, and test only the intent-driven recs.* This keeps `intent_submitted` as a clean trigger, but it measures "intent recs vs WiseGift's own cold recs", not "WiseGift vs no WiseGift". It contradicts D6 and undersells the product to the merchant.
- *Drop the 3-second auto-render from the spec.* That is a product call, not a measurement one. The rule above makes it unnecessary.

---

## 2. KPI definitions

All rates are session-level. "Attributed order" is defined in §3.

| # | KPI | Per-session value `yᵢ` | Population / denominator | Comparison | Role |
|---|---|---|---|---|---|
| 1 | **Widget CTR** | 1 if session has ≥ 1 `widget_engaged` **at or before `t₀`** (or at any time, if the session has no trigger) | `shown` sessions | **integrity check** (expect no difference); **ACCEPTED by PO 2026-10-15 (P2)** | guardrail |
| 2a | **Rec click-through** (replaces add-to-cart at MVP, see below) | 1 if ≥ 1 `product_clicked` after `t₀` | `triggered` **exposed** sessions only | **exposed-only**, no lift | diagnostic |
| 2b | **Rec purchase rate** (replaces add-to-cart at MVP) | 1 if ≥ 1 attributed order contains a `platform_product_id` the session clicked | `triggered` **exposed** sessions only | **exposed-only**, no lift | diagnostic |
| 3 | **Conversion rate** | 1 if ≥ 1 attributed order | `triggered` (headline) and `shown` (secondary) | **lift** | supporting |
| 4 | **AOV** | ratio Σ revenue / Σ orders over converting sessions | converting sessions of the population | **lift, with mix-shift caveat** | supporting |
| 5 | **Revenue per session** | Σ attributed order revenue (winsorised, §5.3) | `triggered` (headline) and `shown` (secondary) | **lift** | **primary** |

### 2.1 Why CTR is an integrity check, not a lift — PROPOSED, pending PO (P2)
**Recommendation: accept.** Widget CTR is not shown as a lift tile. It is returned as an integrity check, and `spec.md`'s CTR lift tile is replaced by this check. The `spec.md` edit is for product-analyst after PO approval.

Why:
- Treatment cannot start before the session's first trigger `t₀` (§1.3). Engagement *before* `t₀` happens while both arms see exactly the same widget, so the recommendations cannot cause it. Its expected difference is exactly 0.
- So if exposed and holdout CTR differ significantly (p < 0.01), the most likely cause is an instrumentation bug: holdout events being dropped, the placebo being visible, or assignment skew. The dashboard should say so, not show it as a "win".
- A CTR "lift" would be at best zero and at worst a bug presented as success. Neither belongs in front of a merchant.

**Why "at or before `t₀`" and not "ever engaged":** today the two are the same, because `intent_submitted` requires `widget_engaged` first. Once pre-intent PDP recs exist (§1.4), an exposed shopper can see recs and *then* engage, so "ever engaged" would partly measure the treatment. Cutting the count at `t₀` keeps the check valid without a later redefinition.

The same logic applies to the **trigger rate** (`triggered` / `shown`), which is also returned as an integrity metric. Whether a session ever reaches its first trigger is decided before any treatment.

*(Event-level CTR, i.e. engaged events / shown events, could drift through repeat-visit behaviour. But it violates the independence assumption of any test, because events cluster within sessions. Do not use it.)*

### 2.2 Why add-to-cart is not measurable at MVP
- `spec.md` defines it through a Shopify `carts/update` webhook. The Frozen MVP scope (2026-09-29) **explicitly excludes cart events**. Nothing ingests `carts/update`.
- Even with cart data, "add-to-cart **on recommended products**" has no holdout counterpart. Holdout sessions get no recommendations, and the rec API is not called for them, so there is no counterfactual slate either.
- **Proposal:** at MVP, replace tile 2 with exposed-only diagnostics 2a/2b, shown as absolute rates without a lift claim. Real add-to-cart comes back only if the PO re-opens cart ingestion (privacy + scope cost). A holdout counterpart would need us to log the counterfactual slate for holdout sessions, which means calling the ranker for holdout without showing the result. That costs LLM spend for no shopper value, so I don't recommend it at MVP. **PO decision D1.**

### 2.3 AOV caveat
AOV is conditioned on a post-treatment outcome (having converted). Suppose the widget converts marginal shoppers with small baskets. AOV can then drop even while revenue per session rises. AOV is shown with its test, but it is labelled *descriptive / mix-sensitive*. Case-study claims use RPS and conversion, never AOV alone.

---

## 3. Attribution join

### 3.1 Which orders are eligible
An order `o` in `orders_attributed` is **eligible for lift** if all of the following hold:
1. `o.tenant_id = :tenant` (always, in every subquery).
2. `o.session_id IS NOT NULL` **and** the marker came from a **symmetric channel** (see G2): the `note_attributes` cart attribute (`_wg_session`, or the legacy `wg_session`) written by the widget for both arms. **ACCEPTED by PO 2026-10-15 (P3). Recommendation: accept.**
   - Orders matched **only** through a URL marker (`landing_site` / `referring_site`) are **excluded from lift**.
   - They still count in the exposed-only diagnostic 2b and in `integrity.urlOnlyAttributedOrders`.
   - Why: a URL marker can only exist for an exposed shopper who clicked a rec, because holdout has nothing to click.
   - Example: a shopper uses "Buy it now" or an express wallet. That checkout skips the cart, so the cart attribute is missing. If the shopper is exposed, the URL fallback can still catch the order. If the shopper is in holdout, the order is lost.
   - Counting those orders would credit WiseGift with sales that the holdout arm is structurally unable to record. Excluding them drops the same kind of order from both arms, so the comparison stays fair. The only cost is a slightly smaller sample.
   - **Mechanism:** the marker source must be persisted (G2, a small migration, in M1·2 scope). If G2 has not landed, the response sets `attributionSourceMixed: true` and every lift tile degrades to `insufficient_data`. A mixed-source lift is never shown as significant. (This replaces the earlier interim rule, which computed lift with all sources and only flagged it.)
3. `o.currency` = the tenant's reporting currency (the modal currency of the tenant's orders in range). Other currencies are counted in `integrity.currencyMismatchOrders` and excluded. No FX conversion at MVP.
4. `o.created_at` is not null. If it is null, fall back to `received_at`.
5. Not a test order (G6). This needs the `test` flag to be dropped or persisted at ingest.

### 3.2 Window rule (7 days)
An eligible order `o` with `session_id = s` is **attributed** to `s` for population P and range [from, to) if:

```
s ∈ P(range)                                      -- s has a trigger event in range
AND o.created_at >= t₀(s)                         -- outcome after (first in-range) trigger
AND EXISTS anchor event a of s with
      a.event_type IN ('widget_shown','widget_engaged','intent_submitted')   -- symmetric only
      AND a.ts ∈ [from, to)
      AND a.ts <= o.created_at
      AND a.ts >  o.created_at - INTERVAL '7 days'
```

Equivalently, the order lands within 7 days of the session's **last symmetric widget event before the order**. Points to note:
- **Anchor ≠ `product_clicked`** (deviates from `spec.md` and `architecture.md` §7). Those docs anchor on the last `product_clicked`, which holdout cannot produce. Anchoring on it would make every holdout order unattributable and inflate lift without bound. The symmetric anchor keeps the "within 7 days of widget contact" intent. **PO decision D3** (doc fix in spec + architecture).
- **Orders before `t₀`** are pre-treatment for that session. They are excluded, not counted as outcomes. This is symmetric across arms and only removes noise.
- **`a.ts`** = the event's **server receive time** once G4 lands (`events.received_at`). Until then it is `occurred_at`, which is the widget's client clock. Shopify `created_at` is server time, so a skewed client clock can push an anchor to *after* the order. Interim mitigation: treat `occurred_at > o.created_at` by ≤ 10 minutes as `<= o.created_at`.
- **Orders after `to`** still count if they satisfy the rule, up to `min(now, last anchor + 7d)`. The cohort is defined by trigger time, and outcomes are allowed to mature.

### 3.3 Multiple orders per session
- **Conversion**: binary. Any number of attributed orders → 1.
- **RPS**: sum of **all** attributed orders of the session (winsorised per order).
- **AOV**: Σ revenue / Σ orders across converting sessions. This is a ratio metric with the session as the unit, so multiple orders per session are handled in the variance (§5.2).
- **All orders vs orders containing a recommended product:** lift KPIs (3, 4, 5) count **all** attributed orders of the session. The holdout measures the *total* causal effect, including halo, substitution and cannibalisation. "Orders containing a rec" has no holdout counterpart, and filtering on it would bias toward exposed. The rec-containing view exists only as exposed-only diagnostic 2b.

### 3.4 Late data and maturation
- **Events arrive late** (1 s flush, buffered until session bootstrap, unload beacons). **Orders arrive** about 4 s after checkout (observed on staging). Neither matters for an hourly recompute, except that the last hour is partial.
- **Right-censoring:** a session triggered *k* < 7 days before `now` has had only *k* days to convert. Both arms are censored equally, so lift is unbiased, but absolute rates are depressed. The API marks `provisionalFrom = now − 7d`. Trend points on or after that date are `provisional: true`. Range-level numbers include them, and the UI footnotes them.
- **Recompute horizon:** any aggregate covering the trailing **8 days** must be recomputed on each refresh (7-day window + 1 day of late events). Older days are immutable, apart from order retractions, which are not ingested (G7).

### 3.5 Kill switch / precomputed / fallback in the exposed arm
Exposed sessions served from `precomputed` / `fallback` / `cache` stay in the exposed arm (intent-to-treat; `recommendations.md` §7.2). The response carries a `servedFrom` mix as a diagnostic so a bad lift can be explained ("40% of exposed rec requests were precomputed").

---

## 4. Degraded (consent-denied) sessions, unattributed and orphan orders

### 4.1 `attribution_mode = degraded`
Per `decisions.md` 2026-09-29 "Widget consent for EU", a consent-denied shopper has no stored UUID, emits no events, carries no `wg_session` marker, and gets a per-request holdout coin flip on recs.
- **Measurement treatment: excluded from both arms by construction**, because there are no sessions or events to count. This is unbiased *for the consenting population*. Consent is decided before assignment and cannot be affected by treatment. **External-validity caveat:** the reported lift is "among shoppers who allow measurement". It is not the store-wide lift. The dashboard must say this when the degraded share is material (> 10% of rec requests).
- **Coverage metric:** `integrity.degradedRequestShare` = rec requests without a client session id / all rec requests in range. This needs the backend to count them (G5). Today nothing records `attribution_mode`, and the widget does not implement the consent check at all (G5).
- Per-request coin-flip placebos for degraded shoppers are unmeasured noise. They neither help nor hurt the measured comparison.

### 4.2 Unattributed orders (`session_id IS NULL`)
- **Excluded from all lift KPIs** (per `spec.md`).
- They are used only for **context and coverage**: `integrity.attributionCoverage` = attributed orders / all orders in range, plus total store orders and revenue. This tells the merchant how much of the store the measurement sees. It is **not** a lift denominator. Unattributed orders contain organic shoppers who never saw the widget, so a "store-wide conversion vs holdout" comparison would be invalid.
- A **sudden drop in coverage** (for example after a theme change that breaks the cart-attribute write) is an alertable integrity signal.

### 4.3 Orphan attributed orders
These are orders with a valid `session_id` but **no `sessions` row** for this tenant, or with no symmetric event in range: events dropped, ad-blocked events endpoint, marker from > 30 days ago, or a forged marker. They are excluded and counted in `integrity.orphanAttributedOrders`. Do not re-derive the arm via `HoldoutAssigner` for them. A session that never reached our event store has no trigger time, and including it would be asymmetric if event loss differs between arms.

### 4.4 Sample-ratio-mismatch (SRM) guard
For each population: `z = (h − 0.10·n) / sqrt(n·0.10·0.90)`, where `n` = sessions in both arms and `h` = holdout sessions. If two-sided **p < 0.001**, the response sets `integrity.srm.ok = false`. The dashboard then shows a data-quality warning, and **every significance flag degrades to `insufficient_data`**. SRM is the cheapest, most reliable detector of the gaps in §7. The CTR and trigger-rate A/A checks (§2.1) are the second line. (SRM uses the fixed 10%; under the D4 80/20 opt-in, it uses the tenant's split per split period.)

---

## 5. Statistics

### 5.1 One engine for mean KPIs (CTR, conversion, RPS, and diagnostics)
For each arm *a* ∈ {E, H}, from per-session values `yᵢ`: `nₐ`, `mₐ = Σy/n`, `sₐ² = (Σy² − n·m²)/(n−1)`.
- **Absolute difference**: `Δ = m_E − m_H`, `SE = sqrt(s_E²/n_E + s_H²/n_H)`.
- **p-value**: two-sided Welch z, `z = Δ / SE`. The normal approximation is fine at the minimum-data floor. For 0/1 data this *is* the unpooled two-proportion z-test, so the family is consistent with `recommendations.md` §2.2.
- **Relative lift**: `L = m_E/m_H − 1`. **95% CI** by the delta method on the log ratio: `se_log = sqrt(s_E²/(n_E·m_E²) + s_H²/(n_H·m_H²))`, `CI = exp(ln(m_E/m_H) ± 1.96·se_log) − 1`. `L` is null when `m_H = 0`.
- The **significance flag uses the p-value only**. At the margin the ratio CI can straddle 0 while p < 0.05, or the reverse. The UI shows the CI as context, not as the decision rule.
- Aggregation needs only `(n, Σy, Σy²)` per arm. That makes daily rollups and CSV reproduction trivial.

### 5.2 AOV (ratio metric)
Over converting sessions *i* in arm *a*: `rᵢ` = winsorised revenue and `oᵢ` = attributed order count. `R = Σr / Σo`. Delta-method variance with the session as the unit:
`Var(R) ≈ (1/(n·ō²)) · (s_r² − 2R·s_ro + R²·s_o²)`.
Welch z on `R_E − R_H`, and the log-ratio CI as in §5.1. Needed sums: `n, Σr, Σo, Σr², Σo², Σro`. Do **not** treat orders as independent units, because they cluster within sessions.

### 5.3 Heavy-tailed revenue: why winsorise and not log-transform
- `recommendations.md` §2.2 proposed Welch's t on log-revenue. I'm **changing that proposal**. RPS is mostly zeros, and `log(0)` forces `log1p`, which changes the estimand to a geometric-mean-like quantity. Merchants cannot reconcile that with their revenue, and it can disagree in sign with the arithmetic lift.
- **Rule:** winsorise each attributed **order** total at the **pooled (both arms) P99 of eligible attributed order totals** in the range, computed at query time. Pooling keeps the cap treatment-blind. Then compute RPS and AOV on the capped values. Below 200 eligible orders in range, cap at the pooled max (no-op) and set `sufficientData=false` anyway.
- The headline lift, CI and p all come from **capped** values, for consistency. **Uncapped** arm means are returned in the response (`valueUncapped`) and in the CSV for reconciliation with Shopify.
- Bootstrap is an offline audit tool. The recommendations-specialist uses it in case-study readouts to confirm the delta-method CI. It is not computed on the hourly path.

### 5.4 Minimum-data rule and power (honesty section)
- **`sufficientData`** for a lift KPI requires all of:
  `n_E ≥ 5,000` **in the KPI's population**, **and** `n_H ≥ 500`, **and**, for conversion/RPS/AOV, `≥ 30 converting sessions in each arm`, **and** `integrity.srm.ok`.
- **Callout:** `neededExposedSessions = max(0, 5000 − n_E)` for the headline population ("need N more exposed sessions").
- **MDE at current n** (80% power, α = 0.05): `mdeRelative ≈ 2.8 · SE / m_H`. This is returned per KPI so the UI can say "a lift smaller than X% would not be detectable yet".
- **What 5,000 buys** (90/10 split, so about 556 holdout sessions):

  | Baseline conversion (triggered) | Detectable relative lift at 5,000 exposed |
  |---|---|
  | 2% | ≈ +88% |
  | 5% | ≈ +55% |
  | 10% | ≈ +38% |

  Detecting a **+10% relative** conversion lift needs about **165k** (5% baseline) to **280k** (3% baseline) triggered sessions in total at 90/10. At **50/50** the same test needs about **60k–100k**. The 10% holdout, not the exposed count, dominates the variance. Most pilots will show grey tiles for weeks or months. **PO decision D4:** keep 90/10 and accept long readouts, temporarily raise the holdout for pilots that opt in (e.g. 20%), or extend readout windows. Changing the split later is safe only for *new* sessions, and assignment must stay sticky.

### 5.5 Peeking and multiple comparisons
- **Peeking:** with hourly refresh and four range presets, a merchant who watches the tile until it turns green has a false-positive rate far above 5%. Two mitigations at MVP:
  1. Significance is evaluated only once `sufficientData` is true. Before that, the flag is `insufficient_data`.
  2. **Case-study / renewal claims are never read off the live dashboard.** They come from a **pre-registered readout**: a fixed analysis window (pilot start → the date the headline population reaches the planned n, or day 60, whichever comes first), one **primary KPI** (RPS, triggered), and conversion as secondary with Holm correction. The recommendations-specialist runs it offline from the same SQL/aggregates, with the bootstrap audit.
  - **v1.1 recommendation:** always-valid p-values (mSPRT with a normal mixing prior) on the cumulative series since pilot start. This makes the live flag safe to watch continuously. It is not built in M1·2.
- **Multiple comparisons:** 3 lift KPIs × placements × ranges. With 5 independent tests at α = 0.05 the family-wise error is about 23%. The KPIs are correlated, so the real figure is lower, but still material. The response marks `role: primary|supporting`. The UI emphasises the primary tile, and per-placement views carry an explicit "exploratory" label.
- **Negative results are shown.** `significance` has four values: `significant_positive`, `significant_negative`, `not_significant`, `insufficient_data` (plus `not_applicable` for exposed-only and integrity tiles). Hiding a significant negative would break the "merchant is told the truth" principle (`recommendations.md` §7.1). Rendering for M2·6: green / red / grey / grey.

---

## 6. Computation model and retention

### 6.1 Proposal
- **M1·2 ships on-the-fly SQL** over raw `events` + `sessions` + `orders_attributed`, behind a `LiftReadModel` out-port in the `events` module. A single tenant-scoped query builds a per-session CTE and reduces it to per-arm sufficient statistics `(n, Σy, Σy², …)`. The math in §5 runs in Java on those sums (a pure, unit-testable `LiftCalculator`). At pilot scale (thousands of sessions per tenant per month) this costs milliseconds.
- **Hourly cache**: Redis key `wg:analytics:{tenant_id}:{metric_version}:{range}:{placement}`, TTL 1 h, tenant-first like the existing cache keys. Compute happens on read-miss. The response includes `computedAt`. There is no scheduled job in M1·2. The hourly freshness promise is met by the TTL.
- **Daily anonymous aggregate `analytics_kpi_daily`** (proposed for M3·1, alongside retention): one row per `(tenant_id, day, placement, arm, population, metric_key)` with sufficient statistics only (`n, sum_y, sum_y2`, plus the ratio sums for AOV). It holds **no session ids**, so it is not pseudonymous personal data and can outlive raw events. It feeds trend lines beyond raw retention and long-horizon / YoY views. Days older than 8 are immutable once written (§3.4). Daily rows are **session-day cohorts by trigger date**, so summing days approximates but does **not** equal the distinct-session range KPI. Range KPIs inside raw retention always come from raw.
- **Why not daily aggregates as the only source:** distinct-session semantics across a 30- or 90-day range cannot be rebuilt from additive daily rows when a session spans days, and they often do (30-day sticky UUID). A session-level rollup table would solve that, but it is more machinery than pilots need. Revisit when raw events per tenant exceed about 10M rows.

### 6.2 Retention (proposal; PO decision D5)
| Data | Proposed retention | Why |
|---|---|---|
| `events` (raw) | **120 days** from `occurred_at` | 90-day max range + 7-day window + reprocessing/backfill buffer. Pseudonymous (`session_id`). |
| `sessions` | 120 days after `last_seen_at` | Must outlive the latest event that references it. |
| `orders_attributed` | 18 months (decided 2026-10-14) | Unchanged. Beyond 120 days, orders keep `session_id` but can no longer be joined to events. That's fine, because lift for such old ranges comes from `analytics_kpi_daily`. Consider nulling `session_id` at 120 days to minimise pseudonymous data (security-and-privacy to weigh in). |
| `analytics_kpi_daily` | 18 months | Anonymous aggregates. Aligns with order retention for YoY. |

Purge enforcement goes into the M3·1 scheduled-purge job.

### 6.3 Tenant isolation (load-bearing)
- `tenant_id` comes **only** from the `AdminPrincipal` (same `requireTenant` pattern as `CatalogAdminController`). It is never a query parameter.
- **Every** CTE / subquery carries `tenant_id = :tenant`, including the `sessions` join (`s.tenant_id = e.tenant_id`) and the orders subquery. A join on `session_id` alone is not acceptable, even though UUIDs are random.
- Required test: a two-tenant fixture where tenant B has an order whose `session_id` equals a tenant-A session. Tenant A's analytics must not count it, and tenant B's must treat it as an orphan.
- New indexes: `events (tenant_id, session_id, occurred_at)` and `orders_attributed (tenant_id, session_id, created_at)`.

### 6.4 SQL sketch (non-normative; backend owns the final query)
```sql
WITH ev AS (           -- in-range events, arm from the session row
  SELECT e.session_id, s.is_holdout, e.event_type, e.placement,
         e.platform_product_id, e.occurred_at AS ts,          -- received_at after G4
         (e.event_type = 'intent_submitted'
          OR (e.event_type = 'widget_shown' AND e.pre_intent_recs)) AS is_trigger   -- §1.4 (P1)
  FROM events e
  JOIN sessions s ON s.tenant_id = e.tenant_id AND s.session_id = e.session_id
  WHERE e.tenant_id = :tenant AND e.occurred_at >= :from AND e.occurred_at < :to
),
first_ev AS (          -- first shown / first trigger per session, ACROSS placements (§1.3)
  SELECT DISTINCT ON (session_id, kind) session_id, kind, ts, placement
  FROM (SELECT session_id, 'shown' AS kind, ts, placement FROM ev WHERE event_type = 'widget_shown'
        UNION ALL
        SELECT session_id, 'triggered', ts, placement FROM ev WHERE is_trigger) x
  ORDER BY session_id, kind, ts
),
sess AS (
  SELECT ev.session_id, bool_or(ev.is_holdout) AS is_holdout,
         fs.ts AS t0_shown,     fs.placement AS placement_shown,
         ft.ts AS t0_triggered, ft.placement AS placement_triggered,   -- placement filter applies to these
         bool_or(ev.event_type = 'widget_engaged'
                 AND (ft.ts IS NULL OR ev.ts <= ft.ts))                AS engaged_pre_t0,   -- KPI 1 (P2)
         bool_or(ev.event_type = 'product_clicked')                    AS clicked
  FROM ev
  LEFT JOIN first_ev fs ON fs.session_id = ev.session_id AND fs.kind = 'shown'
  LEFT JOIN first_ev ft ON ft.session_id = ev.session_id AND ft.kind = 'triggered'
  GROUP BY ev.session_id, fs.ts, fs.placement, ft.ts, ft.placement
  -- population P keeps sessions whose t0 is non-null and, if :placement is set,
  -- whose first-trigger placement = :placement (never "first trigger on :placement")
),
anchor AS (
  SELECT session_id, ts FROM ev
  WHERE event_type IN ('widget_shown','widget_engaged','intent_submitted')
),
eligible_orders AS (
  SELECT o.platform_order_id, o.session_id, COALESCE(o.created_at, o.received_at) AS ots,
         o.total_price, o.line_items
  FROM orders_attributed o
  WHERE o.tenant_id = :tenant AND o.session_id IS NOT NULL
    AND COALESCE(o.created_at, o.received_at) >= :from
    AND COALESCE(o.created_at, o.received_at) <  :to + INTERVAL '7 days'
    AND o.currency = :reporting_currency
    -- AND o.attribution_source = 'note_attribute'   (after G2)
    -- AND NOT o.is_test                             (after G6)
),
attributed AS (        -- one row per (population, order); :pop picks t0 column
  SELECT s.session_id, s.is_holdout, eo.platform_order_id, eo.total_price
  FROM eligible_orders eo
  JOIN sess s ON s.session_id = eo.session_id
  WHERE eo.ots >= s.t0_triggered                       -- or t0_shown for 'shown'
    AND EXISTS (SELECT 1 FROM anchor a
                WHERE a.session_id = eo.session_id
                  AND a.ts <= eo.ots AND a.ts > eo.ots - INTERVAL '7 days')
)
-- then: winsorise total_price at pooled P99, roll up per session,
-- reduce to (n, Σy, Σy², Σo, Σo², Σro) per arm in Java.
```

---

## 7. API contract — `GET /admin/v1/analytics`

Path follows the live admin convention (`/admin/v1/catalog`, `/admin/v1/widget`). `architecture.md` §9.3's `/api/v1/admin/analytics/kpis` is stale and should be fixed in the same doc pass. The JSON uses camelCase records, matching `CatalogAdminController`.

**Request:** `GET /admin/v1/analytics?range=30d&placement=all&population=triggered`
- `range`: `7d | 14d | 30d | 90d` (default `30d`). `to` = start of the current hour (UTC). `from = to − range`.
- `placement`: `all` (default) or a placement key.
- `population`: `triggered` (default) or `shown`.
- Auth: admin session. Tenant comes from the principal. `401`/`403` follow the existing admin pattern.

**CSV:** `GET /admin/v1/analytics/export?range=…&placement=…&population=…`, which returns `text/csv`. It has one row per `(date, kpi, arm)` with `n, value, valueUncapped, provisional`, plus a summary block per KPI with `lift, ciLow, ciHigh, pValue, mdeRelative, significance`. Every number on screen must be reproducible from the CSV.

**Response 200:**
```json
{
  "metricVersion": "lift-v1",
  "computedAt": "2026-10-14T13:00:07Z",
  "range": { "key": "30d", "from": "2026-09-14T13:00:00Z", "to": "2026-10-14T13:00:00Z",
             "provisionalFrom": "2026-10-07T13:00:00Z", "timezone": "UTC" },
  "placement": "all",
  "population": "triggered",
  "reportingCurrency": "EUR",
  "sessions": {
    "exposed": { "shown": 41210, "engaged": 6020, "triggered": 3110 },
    "holdout": { "shown": 4580,  "engaged": 671,  "triggered": 344 }
  },
  "minimumData": { "requiredExposedSessions": 5000, "exposedSessions": 3110,
                   "neededExposedSessions": 1890, "sufficient": false },
  "kpis": [
    {
      "key": "revenue_per_session", "role": "primary", "comparison": "lift",
      "unit": "currency",
      "exposed": { "n": 3110, "value": 4.12, "valueUncapped": 4.31 },
      "holdout": { "n": 344,  "value": 3.70, "valueUncapped": 3.70 },
      "lift": { "relative": 0.114, "ciLow": -0.21, "ciHigh": 0.57, "absolute": 0.42 },
      "pValue": 0.48, "mdeRelative": 0.61,
      "sufficientData": false, "significance": "insufficient_data"
    },
    { "key": "conversion_rate", "role": "supporting", "comparison": "lift", "unit": "ratio", "...": "..." },
    { "key": "aov", "role": "supporting", "comparison": "lift", "unit": "currency",
      "caveat": "mix_sensitive", "...": "..." },
    { "key": "widget_ctr", "role": "integrity", "comparison": "integrity_check", "unit": "ratio",
      "exposed": { "n": 41210, "value": 0.146 }, "holdout": { "n": 4580, "value": 0.147 },
      "pValue": 0.86, "integrityOk": true, "significance": "not_applicable" },
    { "key": "trigger_rate", "role": "integrity", "comparison": "integrity_check", "...": "..." },
    { "key": "rec_click_through", "role": "diagnostic", "comparison": "exposed_only", "unit": "ratio",
      "exposed": { "n": 3110, "value": 0.38 }, "holdout": null, "significance": "not_applicable" },
    { "key": "rec_purchase_rate", "role": "diagnostic", "comparison": "exposed_only", "...": "..." }
  ],
  "trend": [
    { "date": "2026-10-13", "provisional": true,
      "values": { "revenue_per_session": { "exposed": 3.9, "holdout": 3.1 },
                  "conversion_rate": { "exposed": 0.031, "holdout": 0.027 } } }
  ],
  "integrity": {
    "srm": { "expectedHoldoutShare": 0.10, "observedHoldoutShare": 0.0996, "pValue": 0.81, "ok": true },
    "attributionCoverage": { "storeOrders": 1840, "attributedOrders": 212, "share": 0.115 },
    "attributionSourceMixed": false,
    "urlOnlyAttributedOrders": 4,
    "orphanAttributedOrders": 3,
    "currencyMismatchOrders": 0,
    "degradedRequestShare": null,
    "servedFromShare": { "live_llm": 0.41, "cache": 0.37, "precomputed": 0.20, "fallback": 0.02 }
  }
}
```
Field rules:
- `significance` ∈ `significant_positive | significant_negative | not_significant | insufficient_data | not_applicable`. It is computed as: `insufficient_data` if `!sufficientData || !integrity.srm.ok || integrity.attributionSourceMixed` (the last condition is P3); else by `pValue < 0.05` and the sign of `lift.absolute`.
- `urlOnlyAttributedOrders` (P3): orders whose only marker was a URL marker. They are excluded from lift and counted in diagnostic 2b. A large or rising value relative to `attributedOrders` means the cart-attribute write is failing: a theme change, or checkouts that skip the cart.
- `sessions.*.triggered` uses the §1.3 trigger definition. Per-placement session counts add up to `placement=all` (first-trigger attribution, P1).
- `lift` is `null` when `holdout.value == 0` or `holdout.n == 0`.
- `degradedRequestShare` is `null` until G5 lands. The UI renders "not measured".
- `servedFromShare` comes from `product_clicked.served_from` or `cost_telemetry`. It is optional in M1·2.
- **Stub compatibility:** the current `AnalyticsQueryPort.KpiMetrics` (five nullable doubles) cannot carry arms, CI, p, n or flags. Replace it with a `LiftReport` read model. The port is unconsumed today (M2·6 not built), so this is a free change.

---

## 8. Gaps and risks found in the current code

Ordered by impact on measurement. **G1–G3 block meaningful lift numbers**. M1·2 can be built in parallel, but its output is not trustworthy until they land.

| # | Gap | Where | Effect on measurement | Proposed fix / owner |
|---|---|---|---|---|
| **G1** | **No cart attribute is ever written.** The widget only appends `?wg_session=` to rec-card hrefs (`src/attribution.ts`, `RecommendationCard.tsx`). Nothing calls `/cart/update.js`. **Status at 0abf63d:** widget PR #17 is merged. It writes the attribute on the first `widget_shown`, before the bucket is known, for both arms. The R1 rename to `_wg_session` and the R2–R4 hardening (`decisions.md` 2026-10-14) are not yet on `main`: the code still uses `wg_session`. | widget `main` | **Holdout orders can never be attributed** (no recs → no clicks → no marker). Conversion/RPS lift becomes "exposed vs ~0", which is meaningless and maximally flattering. | Widget: on first `widget_shown` per page, if the cart's `wg_session` attribute ≠ the current session id, `POST /cart/update.js {attributes:{wg_session:id}}`. Do it **identically for both arms**. The theme block can render `{{ cart.attributes.wg_session }}` into a data-attribute so steady state costs zero requests. Re-assert after checkout (the cart resets). `frontend-engineer`; privacy check by `security-and-privacy`, because the id becomes visible to the merchant in order "Additional details". |
| **G1b** | **The URL marker very likely never reaches the order.** Shopify `landing_site` is the first page of the storefront *visit*. Rec cards open a **new tab on the same origin** (same visit cookies), so the landing page stays whatever page the shopper entered on. `referring_site` is the external referrer, and the card also sets `rel="noreferrer"`. | backend extractor + widget | Exposed attribution is also near zero. That is consistent with the one staging order landing with `session_id = NULL`. | Verify on staging (order placed after a rec click, mid-visit). Expect G1 to be the only reliable channel. |
| **G2** | **The marker source is not persisted.** `extractWgSession` checks note_attributes → landing_site → referring_site, but `orders_attributed` stores only `session_id`. | backend M1·1 | URL-marker attribution is exposed-only by construction. Mixing it in biases lift upward. | Add `attribution_source VARCHAR(16)` (`note_attribute|landing_site|referring_site`). Lift uses `note_attribute` only. `backend-engineer` (small Flyway migration, fits in M1·2). Under P3, lift tiles stay `insufficient_data` until this lands. |
| **G3** | **The placebo is a "no matches" empty state**, not hidden or neutral (`widget.tsx`: `flow.view === "placebo"` → `RecommendationEmpty`). | widget | Measured effect = "recs vs a visible failure message". If the empty state frustrates shoppers or nudges them away, holdout conversion is depressed and **lift is overstated**. The spec allows "hidden or merchant static fallback". | PO/product-analyst: decide the placebo. Recommendation: a neutral collapse of the rec slot (no "no matches" copy), or the merchant's static fallback. Must be decided before the first pilot readout (**D6**). |
| **G4** | **Events carry only the client clock.** `occurred_at` = widget `Date.now()`, and there is no server receive time. | backend events | Clock skew moves anchors relative to Shopify `created_at`, which drops or adds orders at the window edges and at `t₀`. | Add `events.received_at TIMESTAMPTZ NOT NULL DEFAULT now()` (backfill = `occurred_at`). The attribution join uses it. `backend-engineer`, in M1·2. |
| **G5** | **The consent escape hatch is not implemented.** The widget never reads `Shopify.customerPrivacy`. The backend records no `attribution_mode`. `tenants.consent_mode_required` exists but nothing on the widget path consumes it. | widget + backend | Today every shopper is measured, including ones who denied consent (a compliance risk, not a bias). Once it ships, there is no way to report the degraded share. | Widget: implement the 2026-09-29 decision. Backend: count rec requests without a client session id per tenant/day (cost telemetry already has the hook). `security-and-privacy` to confirm. Not an M1·2 blocker. |
| **G6** | **Test orders are not filtered.** The order mapper ignores Shopify `test: true`. Cancellations and refunds are not ingested. | backend M1·1 | Dev-store and test checkouts pollute pilot numbers. RPS is gross of refunds (symmetric noise, not bias, unless the widget drives returns). | Drop `test: true` at ingest, or persist `is_test` and filter. Refunds: accept at MVP and document as "gross revenue". |
| **G7** | **`total_price` includes shipping and tax.** The spec says "revenue". | backend | Symmetric noise. It inflates AOV/RPS levels in absolute terms. | Accept at MVP, label "gross order value". Optionally persist `subtotal_price` (not PII) later. |
| **G8** | **`/widget/v1/recommendations` does not enforce holdout server-side.** The placebo is enforced only in the widget. | backend recommendation | A stale CDN-cached widget build, a theme override or a bug could serve recs to holdout sessions. That is contamination: lift is biased toward 0. | Defence in depth: if `HoldoutAssigner.isHoldout(principal.sessionId)` is true, return an empty slate with `servedFrom="holdout"` (or 204), and count it as an integrity metric. This applies **with or without intent**, so it also covers the future pre-intent PDP request (§1.4). `backend-engineer`, small. |
| **G9** | **`sessions` PK is `session_id` alone**, not `(tenant_id, session_id)`. | backend schema | A UUID collision across tenants (practically impossible with v4; possible with a forged id) would make the second tenant's session insert fail and drop that tenant's events silently. | Low priority. Make the PK composite at the next schema touch. |
| **G10** | **The widget `holdout` flag is `false` for events queued before bootstrap.** | widget | **No effect.** The server correctly ignores it and uses `sessions.is_holdout`. Noted so nobody "fixes" the backend to trust the client. | None. |
| **G11** | **Docs disagree:** `spec.md` (anchor on `product_clicked`, conversion "/ widget_shown", add-to-cart via `carts/update`, CTR as a lift tile), `decisions.md` ("engaged sessions"), `architecture.md` §9.3 (stale path). `architecture.md` §10.2 step 3 says recs are fetched on viewport entry for every placement. The widget fetches only on intent submit, and the spec limits pre-intent recs to PDP. | docs | Builders will implement whichever doc they read first. | Fold the outcomes of D1–D3, and of P1–P3 once approved, into spec + architecture. §10.2 step 3 should say: pre-intent fetch only on mounts flagged `pre_intent_recs`, and holdout never fetches. Owner: product-analyst. |
| **G12** | **`HoldoutAssigner` is unsalted**: the same UUID gets the same bucket in every future experiment. | shared | Fine for the single MVP holdout. A v2 A/B layer on the exposed arm must hash with a per-experiment salt, or it will correlate with the holdout split. | Note for the v2 bandit/experiments design. |
| **G13** | **The pre-intent PDP path has no measurement hook yet.** (a) `widget_shown` has no `pre_intent_recs` flag, and the strict events schema would reject one. (b) `product_clicked` is sent only when `flow.lastPayload?.intent_mode` exists (`widget.tsx` `onCardClick`), so clicks on intent-less PDP recs would be dropped silently. | widget + backend events | No effect today, because the path does not exist. Shipping the 3-second auto-render without (a) biases `triggered` (§1.4). Without (b), diagnostics 2a/2b undercount. | Must ship **in the same release** as the auto-render (§1.4 sequencing gate). (a) Widget sets the flag from placement/config, before the bucket is known. Backend accepts the field and persists it as `events.pre_intent_recs BOOLEAN NOT NULL DEFAULT false`. (b) Send `product_clicked` with `intent_mode = null`, or a `context` value, for intent-less recs. Owners: `frontend-engineer` + `backend-engineer`. |

---

## 9. Open questions for the PO

| # | Decision | My recommendation |
|---|---|---|
| **D1** ✅ decided 2026-10-14 (see decisions.md) | Add-to-cart tile: not measurable at MVP (no cart ingestion; no holdout counterfactual). | Replace with exposed-only **Rec click-through** + **Rec purchase rate**, without a lift claim. Spec edit. |
| **D2** ✅ decided 2026-10-14 (see decisions.md) | Headline population for conversion and RPS: `triggered` (intent submitted) vs `shown` (spec literal) vs `engaged` (decisions wording). | `triggered` as headline, `shown` as secondary store-level view in the same response. |
| **D3** ✅ decided 2026-10-14 (see decisions.md) | 7-day window anchor: spec says last `product_clicked`. | Last **symmetric** widget event (`shown`/`engaged`/`intent_submitted`), plus "order after first trigger". Spec + architecture edit. |
| **D4** ✅ decided 2026-10-14 (see decisions.md) | 90/10 split power: 5k exposed sessions detects only ≈ +40–90% relative lift. | Keep 90/10 as the default but tell pilots honestly ("readout at N sessions"). Offer a pilot opt-in 80/20 for faster proof. Pre-registered readout for case studies. |
| **D5** ✅ decided 2026-10-14 (see decisions.md) | Raw `events` retention + daily aggregate. | 120 days raw `events`/`sessions`; 18 months anonymous `analytics_kpi_daily`; consider nulling `orders_attributed.session_id` after 120 days. |
| **D6** ✅ decided 2026-10-14 (see decisions.md) | Placebo UX (currently a "no matches" message). | A neutral collapsed slot or the merchant's static fallback before any pilot readout. |
| **D7** ✅ decided 2026-10-14 (see decisions.md) | Show significant **negative** lift as red? (Spec only has green/grey.) | Yes. Honesty is the product. |
| **D8** ✅ decided 2026-10-14 (see decisions.md) | Revenue definition: `total_price` (gross, incl. shipping/tax, pre-refund). | Accept for MVP and label it. Revisit with `subtotal_price` if merchants ask. |
| **D9** ✅ decided 2026-10-14 (see decisions.md) | Day boundaries: UTC vs shop timezone. | UTC at MVP (the shop tz is not stored). Add `iana_timezone` from `shop/update` later. |
| **P1** ACCEPTED by PO 2026-10-15 | Pre-intent PDP recs would make D2's `triggered` population depend on the treatment (§1.4). This is not happening in today's code, but it will once the spec'd 3-second auto-render ships. | **Accept.** A trigger is the first moment the arms can differ: `intent_submitted`, or `widget_shown` on a mount flagged `pre_intent_recs`. It is computed across placements, and the placement filter uses the session's first trigger. On that path the holdout keeps the identical hook and a collapsed, empty rec slot, with no rec call (D6). The auto-render must not ship without the flag (G13). This refines D2; it does not reverse it. |
| **P2** ACCEPTED by PO 2026-10-15 | Widget CTR: lift tile (spec) vs integrity check (§2.1). | **Accept as integrity check.** Count engagement up to the first trigger only. A significant difference means broken instrumentation, not success. Remove the CTR lift tile from `spec.md`. |
| **P3** ACCEPTED by PO 2026-10-15 | Orders matched only through the URL marker (§3.1, G2). | **Accept exclusion from lift.** Holdout can never produce such orders, so counting them flatters WiseGift. Keep them in diagnostic 2b and `urlOnlyAttributedOrders`. Lift tiles stay `insufficient_data` until G2 persists the marker source. |

---

## 10. How M1·2 is verified (definition of done for the measurement, before merge)

1. **`LiftCalculator` unit tests** with hand-computed fixtures: proportions (matching a reference two-proportion z within 1e-6), the RPS mean with zeros, the AOV ratio with multi-order sessions, winsorisation at pooled P99, `m_H = 0` → null lift, and the SRM trigger.
2. **Synthetic A/A integration test**: seed 20k sessions with an identical outcome distribution in both arms (seeded RNG). Over 200 seeds, the false-green rate should be ≈ 5% (within 3–7%). Run nightly, not per-PR, if it is slow.
3. **Synthetic known-lift test**: seed +20% conversion in exposed at n large enough. Assert `significant_positive`, and assert the CI covers 0.20.
4. **Bias regression tests:** (a) a holdout order whose only anchor is earlier than 7 days → not attributed; (b) an exposed order anchored only by `product_clicked` → **not** attributed (anchor symmetry); (c) a URL-sourced marker → excluded from lift, and lift tiles are `insufficient_data` while `attributionSourceMixed` (P3); (d) an order before `t₀` → excluded; (e) **P1:** a holdout session whose only event is a `widget_shown` with `pre_intent_recs = true` is in `triggered`, with `t₀` = that event; (f) **P1:** a session with a flagged PDP `widget_shown` on day 1 and a `home_hero` `intent_submitted` on day 3 has `t₀` = day 1, is counted under `pdp_slot` only, and an order on day 2 is attributed; (g) **P2:** a `widget_engaged` after `t₀` does not count toward the CTR integrity check.
5. **Two-tenant isolation test** per §6.3.
6. **Every response logs** `metricVersion`. Any change to these definitions bumps it (`lift-v2`, …), so a past readout can be reproduced.

---

## Clarifications 2026-10-16 — acknowledged by PO 2026-10-16 (C1–C9; P4 accepted)

> Owner: `recommendations-specialist`. These answer the seven questions
> `backend-engineer` raised while implementing PR 1 of M1·2
> (`wisegift-backend@feature/m1-2-lift-measurement`, 6959e8b), plus findings from
> reading that implementation. This section is append-only. Where it conflicts
> with an earlier section, this section wins **once the PO acknowledges it**.
> Until then the earlier text stands. C1–C7 are clarifications. C8 is a new
> rule and needs an explicit PO decision (**P4**).
> `metricVersion` stays `lift-v1`: nothing has been published under it yet.

### C1. Variance convention: keep n−1 (§5.1 wins over §10.1)
- The engine uses the sample variance `s² = (Σy² − n·m²)/(n−1)` for every mean KPI, binary ones included.
- §5.1's sentence "for 0/1 data this *is* the unpooled two-proportion z-test" should read "**equals it up to a factor of sqrt((n−1)/n)**". The n−1 version gives a z that is very slightly smaller, so it is slightly conservative.
- At the 500-holdout floor the gap is 0.1% of z. Example: a textbook z of 1.962 becomes 1.960. That cannot change a decision in practice.
- §10.1 is corrected. The reference for "proportions within 1e-6" is the unpooled z computed with `p̂(1−p̂)/(n−1)` per arm. Agreement with the textbook `p̂(1−p̂)/n` z is asserted only up to that factor. The existing pinned test is correct.
- Why: one variance formula for 0/1 and revenue data keeps a single engine, and the CSV reproduces it in one line of SQL (`var_samp`).

### C2. Orders ingested before V7 (unknown marker source)
- **Accept the current behaviour. Do not backfill.** The marker source of a pre-V7 row cannot be recovered honestly, because the raw webhook payloads are not kept. Guessing the source by date would be invented data.
- **Narrow the trigger.** `attributionSourceMixed` is set only when an UNKNOWN-source order is **attributed to a session of the selected population**. It is no longer set for any marked order loaded for the range. An unknown order that cannot affect the numbers should not grey out the tiles.
- Effect: only tenants with orders from before V7 are affected, for at most one 90-day range plus the 7-day tail after V7 deploys. Today that is staging only. If a production tenant already has marked orders from before V7, the same rule applies to it, and the PO should be told before its first readout.

### C3. Clock tolerance also applies at t₀
- The rule `o.created_at >= t₀(s)` becomes `o.created_at >= t₀(s) − 10 min`, using the same interim tolerance as the anchor (§3.2). It is removed or revisited when G4 lands.
- Why: a client clock that runs fast moves `t₀` later than the real order time. That mostly drops orders placed within minutes of the trigger. Fast "click a rec, then buy" orders happen mostly in the exposed arm, so without the tolerance lift is **understated**. The tolerance does admit genuinely pre-trigger orders up to 10 minutes early, but those happen equally in both arms. They add noise, not bias.

### C4. The under-200-orders rule covers revenue KPIs only
- Confirmed. Below 200 eligible attributed orders (pooled), `sufficientData=false` applies to **revenue per session and AOV**, not to conversion.
- Why: the rule protects the P99 revenue cap. Conversion does not use revenue, and the "≥ 30 converting sessions per arm" floor already protects it.

### C5. Scope of SRM and the A/A checks
- "For each population" (§4.4) means **compute SRM on both `shown` and the selected population**, with the same placement filter. If either fails (p < 0.001), every lift flag degrades to `insufficient_data`.
- API: `integrity.srm` keeps describing the selected population. A new `integrity.srmShown` has the same shape. When `population=shown`, the two are identical.
- Also, a failed A/A integrity check (`widget_ctr` or `trigger_rate`, p < 0.01) degrades every lift flag to `insufficient_data`, exactly like SRM.
- Why: SRM on `shown` is the most direct detector of holdout `widget_shown` events being lost, and a uniform loss leaves the CTR and trigger-rate *ratios* unchanged. A broken-instrumentation signal that does not stop a green tile is not a guardrail. The cost is a false-alarm rate of about 2–3% per readout, which is acceptable.

### C6. Diagnostic definitions: both confirmed
- **Rec purchase rate (2b).** The denominator is triggered exposed sessions. The numerator counts a session when an order attributed under the 7-day rule (any marker source including URL and UNKNOWN, any currency) contains a product the session clicked **no later than the order time + 10 min**. A click after the order cannot have caused it. The 10 minutes are the same clock tolerance as C3.
  - Make sure product-id formats match: `events.platform_product_id` (from the widget) and the line-item product id (from Shopify REST) must both be the numeric id string, not a GID. Otherwise 2b reads zero without any error.
- **Coverage.** `attributedOrders` = orders placed in `[from, to)` that carry a session marker from any source, including orphans and other currencies. It describes how much of the store the marker reaches; it is not a lift denominator. `urlOnlyAttributedOrders` shows separately how much of that coverage lift cannot use.

### C7. Line-item JSON key
- `architecture.md` §8 (`orders_attributed.line_items`) should say **`platformProductId`** (camelCase), which is what is actually stored: `[{platformProductId, quantity, price}, …]`.
- The read-model SQL may keep accepting `platform_product_id` as a harmless fallback.
- Why: the stored rows are the real contract. Renaming the key would need a JSONB rewrite migration and gives no measurement value.
- Related point on the event wire: the field is `preIntentRecs` (camelCase, like every other event field), stored as `events.pre_intent_recs`. Where §1.4/G13 write `pre_intent_recs: true` on the event, the widget must send `preIntentRecs`.

### C8 (P4, PROPOSED). The cohort is the session's *first-ever* entry, not its first entry inside the range
**Issue.** §1.3 defines `t₀` as the first trigger *inside the range*. A session that was triggered, and therefore treated, *before* `from` and returns during the range joins the range's cohort as if it were new. Whether it returns, and whether it triggers again, can depend on what it was shown the first time. Example: an exposed shopper who liked the recs comes back. A holdout shopper who saw an empty slot either retries the form or leaves. That selects the two arms differently after treatment, the same kind of problem P1 fixed across placements. The bias grows with the share of returning sessions, and it is largest on 7d/14d ranges. `wg_session` lasts 30 sliding days, so many sessions do return. The direction is unknown.

**Rule (proposed).**
- A session belongs to population P for range `[from, to)` only if its **first-ever** entry into P falls inside the range. That is the first `widget_shown` for `shown`, and the first trigger for `triggered`.
- "First-ever" is checked over a fixed **30-day lookback** `[from − 30 d, from)`. The lookback matches the sliding UUID lifetime and fits D5's 120-day retention at the 90-day maximum range.
- A session that entered earlier belongs to the earlier range's cohort. It is not re-counted later.
- A session that was only *shown* before `from` (never triggered) can still enter `triggered` in range. It was never treated.

**Why it is unbiased.** The first-ever entry happens before any treatment, so membership is decided before the arms can differ.

**Cost.** Smaller n per range, because returning sessions from earlier ranges are dropped. Readouts get slower, mostly on short ranges.

**Side benefit.** Every session belongs to exactly one cohort day. The trend points and the future `analytics_kpi_daily` rows then add up to session counts for a range.

**Residual.** Sessions active continuously for more than 30 days before `from` can slip through. That is rare and is accepted.

**Recommendation: accept and implement in PR 1**, so `lift-v1` is never published with the flaw. Add a §10.4 regression test: a session triggered on day −3 and again on day 2 is **not** in the cohort for a range that starts on day 0.

### C9. Spec items not yet implemented in PR 1 (to track, not new rules)
- **§3.1.5 / G6, test orders.** They are not filtered: the Shopify `test` flag is ignored at ingest. This is not just noise. A merchant QA-ing the widget on a live store is exposed 90% of the time and tends to buy the recommended product, so test orders bias lift **upward** on small pilots. **This must land before the first pilot readout.**
- **G4, `events.received_at`.** It is not in V7. The 10-minute tolerance (C3) stays until it lands. Confirm that it is planned for PR 2 of M1·2.
