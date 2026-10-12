# WiseGift — MVP Build Plan

> Created 2026-10-12. Sequences the remaining MVP work into shippable PRs,
> ordered by the critical path to a **first design-partner pilot**. Derived from
> a full gap analysis across `wisegift-backend` + `wisegift-widget` against
> `docs/product/spec.md`. See `docs/decisions.md` 2026-10-12 for the logging entry.

## Where we are (2026-10-12)
The staging stack already runs the widget → recommendation flow end-to-end (real
Shopify catalog, embeddings on pgvector, live GPT-4o-mini ranking, cost
telemetry, 24h cache, manual kill switch). The gaps cluster in three places:

1. **Attribution spine** — half-built on both sides; without it there is no
   revenue-lift proof, which is the whole merchant value proposition.
2. **Merchant admin dashboard** — repo does not exist yet; 100% of the admin
   surface is unbuilt (merchants currently hit "iframe refused to connect").
3. **Compliance / data-lifecycle** — 30-day hard-purge, DPA lawyer review, and
   (for the public App Store path only) GDPR webhooks + listing.

### Already done (not re-listed as PRs)
Backend: OAuth install + tenant creation, catalog ingest (paginated,
rate-limited), 4 catalog webhooks, embeddings w/ recompute-on-change, nightly
reconciliation, `POST .../recommendations` (budget filter + LLM re-rank +
`served_from`), `POST .../events`, cost guardrails (hard cap, session rate
limit, 24h cache, telemetry, **manual** kill switch), **server-side holdout
assignment**, per-placement hook copy, admin auth (Google OAuth + password),
catalog-sync controls, multi-tenant isolation w/ two-tenant leak tests.
Widget: Web Component + Shadow DOM at **15.85 KB gzipped** (budget 50 KB), self +
gift intent forms w/ validation, 8-token theming, session + recommendations
wiring, PDP + floating theme blocks.

## Conventions
- Every code PR: `feature/*` → PR into **`develop`**. Never PR against `main`;
  never push to `main`. Promotion `develop` → `main` is a separate human-owned PR.
- Merging either branch triggers a deploy and is **human-gated**.
- The **backend defines the API contract; the widget/admin implement against it.**
- Sizes: S ≈ ≤1 day, M ≈ 1–2 days, L ≈ 3–5 days.

---

## Decision to close first (non-PR)
**Ranker spec reconciliation.** Implementation is GPT-4o-mini default / GPT-4o
escalation (deliberate PR 7d vendor swap, ~7× cheaper); `spec.md` still names
Haiku 4.5. One-line doc fix in `spec.md` + `decisions.md`. Do before M1.

---

## Milestone 1 — Attribution spine (the value proof)
*A pilot merchant can see real exposed-vs-holdout lift.*

| PR | Repo | Scope | Depends on | Size |
|---|---|---|---|---|
| **M1·1** | backend | `orders/create` webhook receiver — register topic at install, HMAC verify, **PII-drop at ingest**, idempotency on `order_id`, persist filtered payload to `orders_attributed`, capture `wg_session` correlation key | — | M |
| **M1·2** | backend | 7-day attribution **join + analytics KPI service** — replace the zeros-stub `AnalyticsQueryService`; compute CTR / add-to-cart / conversion / AOV / rev-per-session as exposed-vs-holdout + significance; expose `GET /admin/v1/analytics` | M1·1 | L |
| **M1·3** | widget | Session identity — generate + persist anonymous UUID in `localStorage` (30-day sliding TTL); consume `is_holdout` from the session response (backend already assigns it server-side); render **placebo state** when holdout | — (parallel w/ M1·1) | M |
| **M1·4** | widget | Event pipeline — wire the `/widget/v1/events` client (endpoint already live), fire typed `widget_shown`/`widget_engaged`/`intent_submitted`/`product_clicked`, 1s batched flush, fire-and-forget | M1·3 | M |
| **M1·5** | widget | `wg_session` marker on product-click URLs + `product_clicked` before navigate + **IntersectionObserver** lazy-load | M1·4 | S |

**Parallel lanes:** backend M1·1→M1·2 ‖ widget M1·3→M1·4→M1·5. Converge → attribution verifiable end-to-end on staging.

---

## Milestone 2 — Merchant admin dashboard (self-service)
*A merchant can install, configure, and watch spend without us touching the DB.*

| PR | Repo | Scope | Depends on | Size |
|---|---|---|---|---|
| **M2·1** | backend | Admin config APIs — widget-config CRUD, theming-tokens write, placements write | — (parallel w/ M1) | M |
| **M2·2** | **admin (new repo `wisegift-admin`)** | Scaffold — Next.js App Router, Shopify App Bridge (embedded-app iframe fix), Google-OAuth against backend, app shell | — (parallel w/ M1) | M |
| **M2·3** | admin | Onboarding wizard (vertical → catalog-sync progress → placement picker w/ "Configure in Shopify" deep-links → theme-aware preview → go-live) + wire **`tenant.active` activation flow** | M2·2 | L |
| **M2·4** | admin | Widget **Configuration** + **Theming** screens — 8-token editor, live Shadow-DOM preview, presets, WCAG-AA contrast + font-safety warnings, per-token reset | M2·1, M2·2 | L |
| **M2·5** | admin | **Placements** screen + **Catalog-sync** panel (backend already serves these) | M2·2 | M |
| **M2·6** | admin | **Analytics dashboard** — 5 KPI tiles exposed-vs-holdout, trend lines, min-data callout, CSV export, hourly-refresh stamp | M1·2, M2·2 | M |
| **M2·7** | backend + admin | **Billing** — Shopify Billing API integration (currently absent) + Account & usage screen (plan, budget, usage bar w/ 80%/100% states, over-cap banner) | M2·2 | L |

M2·1 + M2·2 start immediately (parallel to M1). M2·3–M2·6 fan out once the shell exists; M2·6 gated on M1·2. M2·7 only required before a *paying* conversion.

---

## Milestone 3 — Compliance & go-live gates
*Legally clear + operationally safe to onboard a real merchant (even a free pilot).*

| PR | Repo | Scope | Depends on | Size |
|---|---|---|---|---|
| **M3·1** | backend | **30-day hard-purge** scheduled job (soft-delete already works) — DPA commitment | — | S |
| **M3·2** | backend | GDPR mandatory webhooks: `customers/data_request`, `customers/redact`, `shop/redact` | — | S |
| **M3·3** | backend | Ops hardening — webhook `"address already taken"` idempotency fix + kill-switch **auto-trip-on-spend** | — | S |
| **M3·4** | — (non-code) | **DPA lawyer review** (security-and-privacy drafts → qualified attorney). External dependency — start early | — | — |

M3·2 + App Store listing gate only the **public** App Store path; a custom-app pilot doesn't need them. M3·1 and M3·4 gate *any* merchant.

---

## Milestone 4 — Widget completeness & polish
*Close MVP-scope spec gaps that aren't pilot-blocking.*

| PR | Repo | Scope | Depends on | Size |
|---|---|---|---|---|
| **M4·1** | widget + backend | **EN/ES/PT i18n** — widget copy, theme locale files, preset copy library (currently EN-only) | — | M |
| **M4·2** | widget | **Home-hero** + **gift-finder-page** theme blocks (only PDP + floating exist) | — | S |
| **M4·3** | backend | Nightly **precomputed cold-recs** job (writes `precomputed_recs` — table exists, no writer) | — | M |
| **M4·4** | widget | Graceful fallback consumption — serve cold/cached recs on timeout or kill-switch instead of the error panel | M4·3 | S |
| **M4·5** | widget + admin | Configurable rec set-size per placement (3–10, default 5) | M2·1 | S |

---

## Wave schedule (2 engineers)
- **Wave 1 (now):** M1·1/M1·2 (backend) ‖ M1·3→M1·5 (widget) ‖ kick off M3·4 (lawyer) ‖ M2·2 scaffold when backend has a spare cycle.
- **Wave 2:** M2·1 (backend) ‖ M2·3–M2·6 admin screens fan out; M2·6 after M1·2.
- **Wave 3:** M3·1–M3·3 hardening ‖ M2·7 billing ‖ M4 polish.

**First-pilot-ready = M1 complete + M2·3/4/5/6 + M3·1 + M3·4.** Billing (M2·7), GDPR webhooks (M3·2), App Store listing, and M4 trail into the pilot window.

## Per-PR plans
Each milestone PR gets its own detailed plan entry in `docs/decisions.md` (matching
the existing "Backend PR N plan" format) at the point it is picked up for build.
