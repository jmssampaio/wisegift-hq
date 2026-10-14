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

---

## 0. TL;DR

1. **The randomisation unit is the widget session.** That is the anonymous `wg_session` UUID, sticky for up to 30 days. Every KPI is computed per session and compared across the arms. No event-level ratios.
2. **The headline population is triggered sessions**: sessions with at least one `intent_submitted`. Exposed and holdout behave identically up to and including that event, and they diverge right after it (recs vs placebo). This is the most powerful comparison that is still apples-to-apples. The store-level view (all `widget_shown` sessions) is returned alongside it.
3. **Lift is real only for conversion, AOV and revenue-per-session.** Widget CTR cannot move by design, because the hook is identical in both arms. It becomes an **integrity check** (an A/A guardrail), not a lift tile. Add-to-cart on recommended products **cannot be measured at MVP**: there is no cart signal, and holdout has no recommended products. It is replaced by an exposed-only diagnostic. **PO decision needed.**
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
| (divergence point) | recs rendered | placebo = "no matches" empty state, no `/recommendations` call | — | |
| `product_clicked` | yes | **never** | **no** | Only exists when recs exist. |

**Consequence:** any metric or window anchor that involves `product_clicked` is **exposed-only by construction**.

### 1.3 Populations (per range, per placement filter)
- **`shown`**: sessions with ≥ 1 `widget_shown` in range. This is the store-level / intent-to-treat view and matches `spec.md`'s literal denominator.
- **`triggered`** (headline): sessions with ≥ 1 `intent_submitted` in range. This is the standard *triggered analysis*. No session can be treated without first submitting intent, and nothing before submission differs between arms. So membership in this set is unaffected by treatment, and the comparison is unbiased. It also discards sessions that could never have been affected (pure dilution).
  - Mapping to the PO's wording: `decisions.md` says "conversion on engaged sessions" and `spec.md` says "/ widget_shown sessions". Both are valid but diluted versions of `triggered`. I propose `triggered` as the headline and `shown` as the secondary view. **PO decision D2.**
- **Trigger time** `t₀(s)` = the first trigger event of the session inside the range (first `widget_shown` for `shown`, first `intent_submitted` for `triggered`).
- **Placement filter:** a session belongs to placement *p*'s population if its trigger event happened on *p*. Outcomes are still **all** attributed orders of that session, so per-placement numbers are **not additive** across placements. `placement=all` is the deduplicated, session-level view and is the default.

---

## 2. KPI definitions

All rates are session-level. "Attributed order" is defined in §3.

| # | KPI | Per-session value `yᵢ` | Population / denominator | Comparison | Role |
|---|---|---|---|---|---|
| 1 | **Widget CTR** | 1 if session has ≥ 1 `widget_engaged` | `shown` sessions | **integrity check** (expect no difference) | guardrail |
| 2a | **Rec click-through** (replaces add-to-cart at MVP, see below) | 1 if ≥ 1 `product_clicked` after `t₀` | `triggered` **exposed** sessions only | **exposed-only**, no lift | diagnostic |
| 2b | **Rec purchase rate** (replaces add-to-cart at MVP) | 1 if ≥ 1 attributed order contains a `platform_product_id` the session clicked | `triggered` **exposed** sessions only | **exposed-only**, no lift | diagnostic |
| 3 | **Conversion rate** | 1 if ≥ 1 attributed order | `triggered` (headline) and `shown` (secondary) | **lift** | supporting |
| 4 | **AOV** | ratio Σ revenue / Σ orders over converting sessions | converting sessions of the population | **lift, with mix-shift caveat** | supporting |
| 5 | **Revenue per session** | Σ attributed order revenue (winsorised, §5.3) | `triggered` (headline) and `shown` (secondary) | **lift** | **primary** |

### 2.1 Why CTR is an integrity check, not a lift
Treatment starts *after* `intent_submitted`, which itself requires `widget_engaged`. So at the session level "ever engaged" can never be caused by recommendations. Its expected lift is exactly 0. If exposed and holdout CTR differ significantly (p < 0.01), the most likely cause is an instrumentation bug: holdout events being dropped, the placebo being visible, or assignment skew. The dashboard should then say so, not show it as a "win". The same logic applies to the intent-submit rate (`triggered`/`shown`), which is also returned as an integrity metric.

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
2. `o.session_id IS NOT NULL` **and** the marker came from a **symmetric channel** (see G2): the `note_attributes` cart attribute written by the widget for both arms. URL-marker orders (`landing_site` / `referring_site`) can only exist for exposed sessions that clicked. They are **excluded from lift** but counted in the exposed-only diagnostics 2b. This requires persisting the marker source (G2). Until then, lift is computed with all marker sources and the response flags `attributionSourceMixed: true`.
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
For each population: `z = (h − 0.10·n) / sqrt(n·0.10·0.90)`, where `n` = sessions in both arms and `h` = holdout sessions. If two-sided **p < 0.001**, the response sets `integrity.srm.ok = false`. The dashboard then shows a data-quality warning, and **every significance flag degrades to `insufficient_data`**. SRM is the cheapest, most reliable detector of the gaps in §7. The CTR/intent-submit A/A checks (§2.1) are the second line.

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
         e.platform_product_id, e.occurred_at AS ts          -- received_at after G4
  FROM events e
  JOIN sessions s ON s.tenant_id = e.tenant_id AND s.session_id = e.session_id
  WHERE e.tenant_id = :tenant AND e.occurred_at >= :from AND e.occurred_at < :to
),
sess AS (
  SELECT session_id, bool_or(is_holdout) AS is_holdout,
         min(ts) FILTER (WHERE event_type = 'widget_shown'
                         AND (:placement IS NULL OR placement = :placement))     AS t0_shown,
         min(ts) FILTER (WHERE event_type = 'intent_submitted'
                         AND (:placement IS NULL OR placement = :placement))     AS t0_triggered,
         bool_or(event_type = 'widget_engaged')                                  AS engaged,
         bool_or(event_type = 'product_clicked')                                 AS clicked
  FROM ev GROUP BY session_id
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
    { "key": "intent_submit_rate", "role": "integrity", "comparison": "integrity_check", "...": "..." },
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
    "attributionSourceMixed": true,
    "orphanAttributedOrders": 3,
    "currencyMismatchOrders": 0,
    "degradedRequestShare": null,
    "servedFromShare": { "live_llm": 0.41, "cache": 0.37, "precomputed": 0.20, "fallback": 0.02 }
  }
}
```
Field rules:
- `significance` ∈ `significant_positive | significant_negative | not_significant | insufficient_data | not_applicable`. It is computed as: `insufficient_data` if `!sufficientData || !integrity.srm.ok`; else by `pValue < 0.05` and the sign of `lift.absolute`.
- `lift` is `null` when `holdout.value == 0` or `holdout.n == 0`.
- `degradedRequestShare` is `null` until G5 lands. The UI renders "not measured".
- `servedFromShare` comes from `product_clicked.served_from` or `cost_telemetry`. It is optional in M1·2.
- **Stub compatibility:** the current `AnalyticsQueryPort.KpiMetrics` (five nullable doubles) cannot carry arms, CI, p, n or flags. Replace it with a `LiftReport` read model. The port is unconsumed today (M2·6 not built), so this is a free change.

---

## 8. Gaps and risks found in the current code

Ordered by impact on measurement. **G1–G3 block meaningful lift numbers**. M1·2 can be built in parallel, but its output is not trustworthy until they land.

| # | Gap | Where | Effect on measurement | Proposed fix / owner |
|---|---|---|---|---|
| **G1** | **No cart attribute is ever written.** The widget only appends `?wg_session=` to rec-card hrefs (`src/attribution.ts`, `RecommendationCard.tsx`). Nothing calls `/cart/update.js`. | widget `main` | **Holdout orders can never be attributed** (no recs → no clicks → no marker). Conversion/RPS lift becomes "exposed vs ~0", which is meaningless and maximally flattering. | Widget: on first `widget_shown` per page, if the cart's `wg_session` attribute ≠ the current session id, `POST /cart/update.js {attributes:{wg_session:id}}`. Do it **identically for both arms**. The theme block can render `{{ cart.attributes.wg_session }}` into a data-attribute so steady state costs zero requests. Re-assert after checkout (the cart resets). `frontend-engineer`; privacy check by `security-and-privacy`, because the id becomes visible to the merchant in order "Additional details". |
| **G1b** | **The URL marker very likely never reaches the order.** Shopify `landing_site` is the first page of the storefront *visit*. Rec cards open a **new tab on the same origin** (same visit cookies), so the landing page stays whatever page the shopper entered on. `referring_site` is the external referrer, and the card also sets `rel="noreferrer"`. | backend extractor + widget | Exposed attribution is also near zero. That is consistent with the one staging order landing with `session_id = NULL`. | Verify on staging (order placed after a rec click, mid-visit). Expect G1 to be the only reliable channel. |
| **G2** | **The marker source is not persisted.** `extractWgSession` checks note_attributes → landing_site → referring_site, but `orders_attributed` stores only `session_id`. | backend M1·1 | URL-marker attribution is exposed-only by construction. Mixing it in biases lift upward. | Add `attribution_source VARCHAR(16)` (`note_attribute|landing_site|referring_site`). Lift uses `note_attribute` only. `backend-engineer` (small Flyway migration, fits in M1·2). |
| **G3** | **The placebo is a "no matches" empty state**, not hidden or neutral (`widget.tsx`: `flow.view === "placebo"` → `RecommendationEmpty`). | widget | Measured effect = "recs vs a visible failure message". If the empty state frustrates shoppers or nudges them away, holdout conversion is depressed and **lift is overstated**. The spec allows "hidden or merchant static fallback". | PO/product-analyst: decide the placebo. Recommendation: a neutral collapse of the rec slot (no "no matches" copy), or the merchant's static fallback. Must be decided before the first pilot readout (**D6**). |
| **G4** | **Events carry only the client clock.** `occurred_at` = widget `Date.now()`, and there is no server receive time. | backend events | Clock skew moves anchors relative to Shopify `created_at`, which drops or adds orders at the window edges and at `t₀`. | Add `events.received_at TIMESTAMPTZ NOT NULL DEFAULT now()` (backfill = `occurred_at`). The attribution join uses it. `backend-engineer`, in M1·2. |
| **G5** | **The consent escape hatch is not implemented.** The widget never reads `Shopify.customerPrivacy`. The backend records no `attribution_mode`. `tenants.consent_mode_required` exists but nothing on the widget path consumes it. | widget + backend | Today every shopper is measured, including ones who denied consent (a compliance risk, not a bias). Once it ships, there is no way to report the degraded share. | Widget: implement the 2026-09-29 decision. Backend: count rec requests without a client session id per tenant/day (cost telemetry already has the hook). `security-and-privacy` to confirm. Not an M1·2 blocker. |
| **G6** | **Test orders are not filtered.** The order mapper ignores Shopify `test: true`. Cancellations and refunds are not ingested. | backend M1·1 | Dev-store and test checkouts pollute pilot numbers. RPS is gross of refunds (symmetric noise, not bias, unless the widget drives returns). | Drop `test: true` at ingest, or persist `is_test` and filter. Refunds: accept at MVP and document as "gross revenue". |
| **G7** | **`total_price` includes shipping and tax.** The spec says "revenue". | backend | Symmetric noise. It inflates AOV/RPS levels in absolute terms. | Accept at MVP, label "gross order value". Optionally persist `subtotal_price` (not PII) later. |
| **G8** | **`/widget/v1/recommendations` does not enforce holdout server-side.** The placebo is enforced only in the widget. | backend recommendation | A stale CDN-cached widget build, a theme override or a bug could serve recs to holdout sessions. That is contamination: lift is biased toward 0. | Defence in depth: if `HoldoutAssigner.isHoldout(principal.sessionId)` is true, return an empty slate with `servedFrom="holdout"` (or 204), and count it as an integrity metric. `backend-engineer`, small. |
| **G9** | **`sessions` PK is `session_id` alone**, not `(tenant_id, session_id)`. | backend schema | A UUID collision across tenants (practically impossible with v4; possible with a forged id) would make the second tenant's session insert fail and drop that tenant's events silently. | Low priority. Make the PK composite at the next schema touch. |
| **G10** | **The widget `holdout` flag is `false` for events queued before bootstrap.** | widget | **No effect.** The server correctly ignores it and uses `sessions.is_holdout`. Noted so nobody "fixes" the backend to trust the client. | None. |
| **G11** | **Docs disagree:** `spec.md` (anchor on `product_clicked`, conversion "/ widget_shown", add-to-cart via `carts/update`), `decisions.md` ("engaged sessions"), `architecture.md` §9.3 (stale path). | docs | Builders will implement whichever doc they read first. | Fold the outcomes of D1–D3 into spec + architecture after PO approval. |
| **G12** | **`HoldoutAssigner` is unsalted**: the same UUID gets the same bucket in every future experiment. | shared | Fine for the single MVP holdout. A v2 A/B layer on the exposed arm must hash with a per-experiment salt, or it will correlate with the holdout split. | Note for the v2 bandit/experiments design. |

---

## 9. Open questions for the PO

| # | Decision | My recommendation |
|---|---|---|
| **D1** | Add-to-cart tile: not measurable at MVP (no cart ingestion; no holdout counterfactual). | Replace with exposed-only **Rec click-through** + **Rec purchase rate**, without a lift claim. Spec edit. |
| **D2** | Headline population for conversion and RPS: `triggered` (intent submitted) vs `shown` (spec literal) vs `engaged` (decisions wording). | `triggered` as headline, `shown` as secondary store-level view in the same response. |
| **D3** | 7-day window anchor: spec says last `product_clicked`. | Last **symmetric** widget event (`shown`/`engaged`/`intent_submitted`), plus "order after first trigger". Spec + architecture edit. |
| **D4** | 90/10 split power: 5k exposed sessions detects only ≈ +40–90% relative lift. | Keep 90/10 as the default but tell pilots honestly ("readout at N sessions"). Offer a pilot opt-in 80/20 for faster proof. Pre-registered readout for case studies. |
| **D5** | Raw `events` retention + daily aggregate. | 120 days raw `events`/`sessions`; 18 months anonymous `analytics_kpi_daily`; consider nulling `orders_attributed.session_id` after 120 days. |
| **D6** | Placebo UX (currently a "no matches" message). | A neutral collapsed slot or the merchant's static fallback before any pilot readout. |
| **D7** | Show significant **negative** lift as red? (Spec only has green/grey.) | Yes. Honesty is the product. |
| **D8** | Revenue definition: `total_price` (gross, incl. shipping/tax, pre-refund). | Accept for MVP and label it. Revisit with `subtotal_price` if merchants ask. |
| **D9** | Day boundaries: UTC vs shop timezone. | UTC at MVP (the shop tz is not stored). Add `iana_timezone` from `shop/update` later. |

---

## 10. How M1·2 is verified (definition of done for the measurement, before merge)

1. **`LiftCalculator` unit tests** with hand-computed fixtures: proportions (matching a reference two-proportion z within 1e-6), the RPS mean with zeros, the AOV ratio with multi-order sessions, winsorisation at pooled P99, `m_H = 0` → null lift, and the SRM trigger.
2. **Synthetic A/A integration test**: seed 20k sessions with an identical outcome distribution in both arms (seeded RNG). Over 200 seeds, the false-green rate should be ≈ 5% (within 3–7%). Run nightly, not per-PR, if it is slow.
3. **Synthetic known-lift test**: seed +20% conversion in exposed at n large enough. Assert `significant_positive`, and assert the CI covers 0.20.
4. **Bias regression tests:** (a) a holdout order whose only anchor is earlier than 7 days → not attributed; (b) an exposed order anchored only by `product_clicked` → **not** attributed (anchor symmetry); (c) a URL-sourced marker → excluded from lift once G2 lands; (d) an order before `t₀` → excluded.
5. **Two-tenant isolation test** per §6.3.
6. **Every response logs** `metricVersion`. Any change to these definitions bumps it (`lift-v2`, …), so a past readout can be reproduced.
