# WiseGift — Command Center

This is the orchestration repo for the WiseGift project. The human is the
**Product Owner and Tech Lead**, and is the final validator on all decisions.
Agents recommend and draft; the human approves. Nothing is "done" until the
human validates it.

## What WiseGift is
A **B2B recommendation platform for e-commerce merchants**, sold as an
embeddable frontend widget that hooks shoppers with a gift-framed question
("for you or for someone else?"), captures intent, and calls WiseGift
micro-APIs for personalised recommendations drawn from the merchant's own
catalog. The engine handles both self and gift intents. MVP platform is
Shopify; follow-ons are VTEX and Salesforce Commerce Cloud. The **core
differentiator is the recommendation logic** (with gift-intent as its most
valuable use case) running over per-tenant catalogs — the catalog data quality
per merchant and the matching/ranking quality are the heart of the product.

Pre-2026-09-29 the project was a B2C consumer app; the strategic pivot is
recorded in `docs/decisions.md`. The Flutter consumer app is parked but not
deleted (potential future B2C storefront over participating merchants).

## Repositories
- `~/Workspace/wisegift/wisegift-backend` — backend (APIs, per-tenant data, order webhook receiver)
- `~/Workspace/wisegift/wisegift-flutter` — Flutter B2C client, **parked** as of 2026-09-29
- widget + merchant admin frontend repos — not yet created; scope defined in `docs/product/spec.md`
- this repo (`wisegift-hq`) — specs, decisions, the agent team, shared docs

> Launch Claude Code from this repo to get the full team, then grant access to
> the code repos as needed, e.g.:
> `claude --add-dir ../wisegift-backend`

## The team (subagents) — 7 active + 1 parked

**Product**
- `product-analyst` — specs (user stories + acceptance criteria) + interaction/UX design for widget and admin (`docs/product/spec.md`)

**Build**
- `backend-engineer` — multi-tenant APIs, per-tenant catalog ingestion, Shopify integration, tenant isolation, cost guardrails (`../wisegift-backend`)
- `frontend-engineer` — embeddable widget (Preact + Web Components, ≤ 50 KB) + merchant admin dashboard (React/Next.js)
- `recommendations-specialist` — ranking, cold-start-per-merchant, precomputed vs live serving, evaluation. **The differentiator — kept focused, not merged.**

**Trust & ops**
- `security-and-privacy` — defensive security (multi-tenant isolation, widget XSS, HMAC) + privacy/DPA/legal drafts. Day-1 critical.
- `devops-expert` — CI/CD, EU-region hosting, widget CDN, custom-app + App Store pipeline, cost telemetry infra

**Growth**
- `marketing-manager` — B2B SaaS positioning, Design Partner Program, App Store listing, case studies (`docs/growth/gtm-b2b.md`)

**Parked**
- `flutter-expert-parked` — owns the parked Flutter B2C app. Do not invoke unless the PO explicitly asks or the B2C revival roadmap item is activated.

*Test discipline is a coding standard folded into `backend-engineer` and `frontend-engineer` — no separate qa-tester at MVP scale.*

## Source of truth (read the relevant ones before acting)
- `docs/product/spec.md` — B2B product spec: merchant onboarding, widget, admin, APIs, attribution, cost guardrails
- `docs/decisions.md` — running decision log (append; never rewrite)
- `docs/growth/gtm-b2b.md` — Design Partner Program, pricing thesis, cost model & guardrails
- `docs/engineering/architecture.md` — system design & API contracts (pending B2B rewrite; currently describes the pre-pivot Firebase+Spring stack)
- `docs/engineering/release-process.md` — branching model & deployment chain
- `docs/legal/legal.md` — privacy/legal drafts including DPA template (to be consolidated with historical security posture from the archived security-privacy report)
- `docs/archive/` — pre-pivot docs preserved for context (B2C spec/data/recommendations/QA, giftability rules, one-off migrations, historical security-privacy report)

## Working rules
- Read `docs/product/spec.md` and `docs/decisions.md` before starting any work.
- Record meaningful decisions in `docs/decisions.md` (date, who, what, why). Append only — never rewrite history.
- The **backend defines the API contract**; the frontend implements against it.
- The **recommendations-specialist owns matching and must define how quality is measured** (holdout lift on CTR / add-to-cart / conversion / AOV / revenue-per-session) before building.
- **Multi-tenant isolation is a load-bearing invariant.** Every table with tenant data carries `tenant_id NOT NULL`; every query is tenant-scoped; cross-tenant leaks are P0 bugs.
- **Cost guardrails from `docs/growth/gtm-b2b.md` are engineering requirements, not v2** — per-tenant hard cap, response cache, precomputed cold recs, per-tenant kill switch, cost telemetry.
- **EU-only region for MVP.** All data (Neon Postgres, hosting, embeddings, logs) in EU. Adding US region requires a PO decision + coordinated DPA/sub-processor update.
- **All code changes target `develop` via a `feature/*` branch and PR. Never open a PR against `main` and never push to `main` directly.** Promotion of `develop` → `main` is a separate human-owned PR that ships production. See `docs/engineering/release-process.md`. Merging to either branch triggers a deploy and is human-gated.
- Never mark work "done" — recommend; the human validates.
- Flag ambiguity, risk, and tradeoffs to the human rather than guessing.
- **Side-effectful actions require explicit human approval** before running: deploys, infra changes, App Store submissions, anything that publishes, sends, deletes, or rotates keys. Propose and wait.
- `security-and-privacy` is defensive only — no exploit/offensive code. On the legal side it is informational only — not a lawyer; route binding questions to a qualified attorney. DPA sign-off before a first paying merchant requires attorney review.

## Commits
Code changes land in their own repos (`wisegift-backend`, plus the widget +
admin repos when they exist), reviewed in their respective IDEs. Specs and
decisions commit here in HQ — this repo is the single source of truth and the
project's real memory across sessions.
