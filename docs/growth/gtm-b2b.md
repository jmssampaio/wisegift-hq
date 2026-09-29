# WiseGift — B2B Go-to-Market

> Living document. Scope: how WiseGift acquires its first paying merchants and evolves its pricing. Companion to `docs/product/spec.md` (product) and `docs/core/recommendations.md` (engine).

## Product context

WiseGift sells **AI gift-recommendation infrastructure** to e-commerce merchants. Delivery is an embeddable frontend widget on the merchant's storefront (home / PLP / PDP) that hooks the shopper — "for you or someone else?" — captures gifting intent, and calls WiseGift micro-APIs for personalised recommendations drawn from the merchant's own catalog.

- MVP platform: **Shopify**. Follow-ons: VTEX, SFCC, custom via REST.
- Catalog is **per-tenant** (each merchant's own products). No affiliates.
- The existing Flutter B2C app is parked; kept for a possible future consumer play with merchant-authorised catalog resale.

## Ideal Customer Profile (ICP) — pilot phase

Pick merchants big enough to produce meaningful case studies, small enough to protect our API budget while we tune.

| Attribute | Target |
|---|---|
| Monthly sessions | 50k – 500k |
| SKU count | 1k – 50k |
| Platform | Shopify Plus or serious Shopify |
| Vertical | Gift-heavy: jewelry, beauty, home goods, kids, specialty food, fashion accessories, books |
| Region | EU (start ES/PT — where the team is), then UK/DE |
| Traits | Runs promos, has a marketing team, cares about conversion rate, willing to give weekly feedback |

**Explicitly out of scope for pilots:** enterprise merchants (Mango, Zalando tier) — too expensive on API and too slow in procurement. Revisit after 3 case studies exist.

## Design Partner Program (the MVP go-to-market)

The first ~5 merchants pay nothing. We use them to build the product, generate lift data, and produce case studies that sell the next 20 deals. Free is the **cost of learning**, not a growth channel.

### The offer

- **Free access** to the widget + APIs for a fixed **90-day pilot window**.
- Full onboarding support from the WiseGift team.
- A dedicated success metric report at day 30, 60, and 90.
- A conversion conversation on day 90 with a pilot-partner discount (see Pricing thesis).

### What we ask for in exchange

- Weekly 30-minute feedback call for the first 6 weeks; bi-weekly thereafter.
- Rights to publish a **case study** with the merchant's logo and pilot lift numbers.
- **Two warm intros** to peer merchants in their network.
- Commitment to enter a paid-conversion conversation on day 90 if success criteria are met.

### Selection criteria (target 5, sign 3)

Pick merchants that match the ICP AND:
- Have a decision-maker on the pilot call (owner, head of e-com, head of growth).
- Can install a Shopify app themselves (no procurement/legal delays > 2 weeks).
- Are willing to run the exposed-vs-holdout split from day one.
- Have a real gifting use case in their assortment (not pure functional/replenishment).

### 90-day pilot playbook

| Phase | Days | Milestones |
|---|---|---|
| Onboard | 1 – 14 | Install app, catalog sync verified, widget placed on 1 PDP + 1 dedicated gift-finder page, exposed/holdout split live |
| Learn | 15 – 45 | First lift report at day 30. Adjust intent form, widget placement, ranking weights |
| Prove | 46 – 75 | Second lift report at day 60. Add a second placement (e.g. home hero). Case study data draft |
| Convert | 76 – 90 | Third lift report. Commercial conversation. Case study sign-off |

### Success criteria (what we're measuring)

Report per merchant, all measured against the holdout:

- **CTR** on the widget (widget-shown → widget-engaged)
- **Add-to-cart rate** on recommended products
- **Conversion rate** for sessions that engaged the widget
- **AOV** on those sessions
- **Revenue per session** (the composite metric that matters most)

We need **statistical significance** (p < 0.05, min ~5,000 exposed sessions per placement) before a lift number goes in a case study.

### Conversion path

At day 90, one of three outcomes:

1. **Convert to paid** at the pilot-partner discount (see Pricing thesis).
2. **Extend the pilot 30 days** if we're close on significance but not yet cleared.
3. **Graceful exit**: they keep the case study rights (both sides), we keep the learnings, no bad blood.

## Pricing thesis (post-pilot)

The market has converged on **tiered subscription + usage overage** (Nosto, Rebuy, Klevu, Bloomreach). We follow the same shape.

### Base structure

- **Monthly base fee by tier**, with a bundle of recommendations included.
- **Overage per 1,000 recommendations** above the tier.
- **Annual commit discount** (~15%) once merchants trust the product.

### Draft tiers (revisit after 3 pilots convert)

| Tier | Monthly base | Recommendations included | Target merchant size |
|---|---|---|---|
| Starter | €X | 50k / mo | < 100k sessions |
| Growth | €X | 250k / mo | 100k – 500k sessions |
| Scale | €X | 1M / mo | 500k – 2M sessions |
| Enterprise | Custom | Custom | 2M+ sessions, custom SLA, premium features |

*(Fill in prices after we have 3 paying merchants and know the per-recommendation cost more precisely. Anchor: land the Growth tier at 3–5× our fully loaded cost per recommendation.)*

### Pilot-partner discount

Design partners get **50% off the tier price for the first 12 months** as a thank-you and to soften the "free → paid" transition. Locks them in through year 1; they renew at list.

### Premium features (upsell levers)

- **Verticalised intent forms & style parameter packs** (see Roadmap, below). Included in Growth and above.
- **Purchase-outcome learning loop** (see Roadmap). Included in Scale and above; drives higher lift over time.
- **Custom placements & A/B testing dashboard**. Enterprise.
- **SLA & dedicated support**. Enterprise.

### What we explicitly do NOT do at MVP

- **Rev-share on attributed sales.** Magnetic for merchants, nightmare to enforce and collect. Requires order-level webhook integration and a legal audit right. Revisit at Enterprise scale only.
- **Per-seat pricing.** This isn't a users tool.
- **Free forever tier.** Freemium burns AI budget on tire-kickers with no upgrade path. Revisit after we have a solid paying base and want a PLG channel.

## Cost model & guardrails (MVP requirements)

Since pilots are free, protecting the AI budget is the difference between shipping and running out of runway. These are **non-negotiable engineering requirements** for the MVP, not v2 nice-to-haves.

### Cost math (planning baseline)

Assuming a mid-market pilot (200k monthly sessions, ~30–40% widget engagement, cheap model default):

| Component | Approach | Est. cost |
|---|---|---|
| Catalog embeddings | One-time per SKU + on updates; pgvector on Neon | ~€2–10 / merchant, one-time |
| Retrieval | Vector nearest-neighbour, no LLM | ~€0 |
| Re-ranking / intent parse | Claude Haiku 4.5 on top-K candidates | ~€0.0005–0.001 / call |
| Rec calls | ~150k / mo / merchant | ~€75–150 / mo / merchant |
| Infra (Neon, hosting, CDN) | Starter tiers | ~€50–150 / mo total |

**Planning total for 3 pilots:** €275–600 / month worst case. With the guardrails below, floor is **€100–200 / month**.

### Required guardrails (MVP scope)

1. **Hard per-tenant usage cap.** Default 50k rec calls/month during pilot. Above cap → serve cached/precomputed recs, no LLM. Configurable per merchant.
2. **Response caching.** Same product + same intent signature → same rec for 24h. Expect 60–80% cache hit rate at steady state.
3. **Precomputed cold recs per SKU.** Nightly job produces top-N per SKU without LLM. Widget serves these when the shopper provides no intent signal (passive PDP view). LLM is called only when we have real gifting context.
4. **Cheap model default.** Haiku 4.5 for re-ranking and intent parsing. Escalate to Sonnet only for the full gifting-flow experience where the intent is rich and the value justifies the cost.
5. **Session-level rate limiting.** No merchant's bot traffic drains our budget. Hard limit: 20 rec calls per session.
6. **Kill switch per tenant.** Daily spend threshold triggers alert + graceful degradation to precomputed recs. We are never surprised by a bill.
7. **Cost telemetry from day one.** Every rec call tagged with tenant, placement, model, tokens in/out, cache hit/miss. Dashboard shows cost per merchant per day.

**These requirements must be echoed in `docs/product/spec.md` when the spec is rewritten for the B2B pivot.**

## Roadmap levers that affect pricing

Two directions confirmed by the PO on 2026-09-29 that shape future tier packaging:

### Purchase-outcome feedback loop (v2)

Track what shoppers **actually bought** in the sessions where they engaged the widget — both recommended and non-recommended products. Feed this back into ranking (via contextual bandit / preference learning) to improve per-tenant recommendation quality over time.

Why it matters for pricing:
- **Merchant lift compounds** the longer they stay on — reinforces retention.
- **Data moat**: newer competitors start cold; our engine gets smarter per tenant month over month.
- **Enterprise unlock**: the outcome dataset is what an Enterprise merchant will want visibility into.

Deferred to v2 because it requires order-level webhook integration and an evaluation harness. But architecturally, the MVP must emit the events (widget shown, widget engaged, product clicked, tenant order webhook received) so we don't have to backfill.

### Verticalised intent parameter packs

Ship starter packs of intent-capture questions and style parameters **per merchant vertical**, extensible by the merchant. Examples confirmed by PO:

- **Fashion**: style archetypes (retro, minimal, streetwear, classic…) with merchant-defined reference photos of models in their own clothes. Ask for recipient sizing hints only if merchant offers easy returns.
- **Tech / Gaming**: platforms owned (console, PC, mobile), preferred genres, recent games they liked.
- **Books**: currently reading, favourite author, preferred genre, format (hardback / paperback / ebook).
- **Beauty**: skin type, tone preferences, fragrance family (fresh / floral / woody).
- **Home**: room being furnished, style, existing colour palette.

This is a **major differentiator vs. generic reco engines** (Nosto/Rebuy do behavioural, not intent-driven gifting). It also justifies the Growth-tier price step.

Deferred to a follow-on release because it requires:
- A vertical taxonomy on merchant onboarding.
- A merchant admin UI for uploading style reference imagery.
- An intent-form schema per vertical (versioned).

MVP ships a **single generic intent form**. Verticalised packs land in the release right after MVP.

## Open questions (to close before we start selling)

- Legal / tax vehicle for taking payment from EU merchants? (SPV, sole trader, Stripe Connect setup)
- Data processor DPA template — needed on day 1 for any merchant conversation.
- Sub-processor list (Neon, Anthropic, hosting, monitoring) — merchants will ask.
- Region: are pilot merchants OK with EU-hosted data only, or do we need US region for future pilots?
- Do we register as a Shopify Partner immediately (needed to publish an app), or start with a custom app per pilot merchant?

*Route to `privacy-legal-advisor` and `devops-expert` respectively.*

---

*Change history in `docs/decisions.md` — append there, don't rewrite this file's history.*
