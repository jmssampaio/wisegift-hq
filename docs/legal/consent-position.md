# WiseGift Widget — EU Consent Position

> **Informational only. Not legal advice.** This paper is prepared by
> WiseGift's internal security-and-privacy function for onboarding
> conversations with pilot merchants and their advisors. It is not a legal
> opinion. A qualified attorney in the merchant's jurisdiction should sign
> off before a paying contract, and each merchant remains responsible for
> the consent posture on its own storefront.
>
> Basis: WiseGift's product specification (`docs/product/spec.md`) and data
> model (`docs/core/data.md`) as of 2026-09-29. If either changes
> materially — in particular the data set or the region — this paper is
> superseded.

---

## 1. TL;DR

WiseGift's widget stores a single first-party anonymous UUID in the
shopper's browser to run holdout-vs-exposed attribution for the merchant.
It is not used for cross-site tracking, is never joined against personal
data on our side, and lives only on the merchant's own origin. Our
position is that this storage is defensible under ePrivacy Article 5(3)
as strictly necessary for a measurement service the merchant has
requested, with GDPR Article 6(1)(f) legitimate interests as the lawful
basis for the resulting processing; we recognise stricter national DPAs
may disagree and we ship an off-by-default fallback that respects a
merchant's Consent Mode signal.

## 2. What we set, and why

On the shopper's first widget render on your storefront, the widget
generates a **UUID v4 client-side** and writes it to `localStorage` under a
WiseGift-scoped key (`wg_session`). The identifier:

- is used only inside your storefront's origin — it is a first-party
  identifier for your site;
- is used to correlate `widget_shown → widget_engaged → intent_submitted
  → product_clicked` into a single funnel so we can compute the widget's
  lift for you against a 10 % holdout;
- deterministically decides whether this browser is exposed to WiseGift
  recommendations (90 %) or shown a placebo slot (10 %). Without a stable
  identifier the holdout assignment would re-roll on every page load and
  every lift number we report to you would be meaningless.

It has a **30-day sliding TTL** and is regenerated after that window.

## 3. Legal basis (belt-and-braces)

### 3.1 ePrivacy Directive Article 5(3) — strictly-necessary limb

Article 5(3) permits storage on a user's terminal equipment without prior
consent where it is "strictly necessary in order for the provider of an
information society service explicitly requested by the subscriber or
user to provide the service." The merchant explicitly requests WiseGift's
service by installing our app and enabling the widget; the identifier is
the mechanism by which the service (measurable, holdout-controlled
recommendations) is delivered. Removing the identifier does not degrade
the widget — it disables the measurement the merchant purchased.

CNIL (France) has published guidance since 2020 tolerating first-party,
non-cross-site aggregate measurement cookies/identifiers under a
strictly-necessary reading when the analytics are for the site operator
only and are not fed to a third party for their own purposes. Our data
flow matches those preconditions: session IDs stay per-merchant, are
never sold, and are never enriched into a cross-site profile.

### 3.2 GDPR Article 6(1)(f) — legitimate interests

Even where ePrivacy is satisfied, the resulting processing needs a GDPR
lawful basis. We rely on the merchant's **legitimate interest** in
measuring the effectiveness of a paid service on its own storefront,
balanced against a low-intrusion identifier that carries no directly
identifying data, no cross-site tracking, and a bounded 30-day retention.
The balancing test lands in favour of the merchant given: no profile is
built, no data is shared, no automated decision has legal or similarly
significant effect on the shopper, and shoppers retain the ordinary
browser controls over `localStorage`.

## 4. What we do NOT do

Explicit non-practices, so they are on the record:

- We do not set third-party cookies from the widget.
- We do not track shoppers across merchants. Two merchants using
  WiseGift produce two independent, unlinked identifiers.
- We do not receive, request, or store the shopper's name, email, phone,
  address, IP address, or user-agent. The filtered `orders/create`
  webhook drops all customer PII at the receiver, before persistence.
- We do not join session IDs against the merchant's customer database.
- We do not use session IDs for advertising or ad measurement.
- We do not sell or share event data with any third party for their own
  purposes. Sub-processors act only on WiseGift's documented instructions.

## 5. Retention and sub-processors

- **Retention.** Session identifier: 30-day sliding TTL in the browser.
  Server-side session and event rows: kept for the tenant's active
  lifetime; hard-purged 30 days after uninstall (see `data.md` §7).
- **Region.** All storage and compute is in the **EU** at MVP (Neon EU,
  application hosting EU, telemetry EU). A US region is not deployed and
  would require a coordinated DPA and sub-processor update before use.
- **Sub-processors** relevant to shopper data: Neon (EU Postgres),
  Anthropic (LLM ranking — no session ID leaves our backend; only the
  intent payload and product texts are sent), our EU hosting provider,
  our EU monitoring provider. The authoritative list ships with the DPA
  and is updated in-place.

## 6. Where a stricter DPA may disagree, and our fallback

We do not overclaim uniformity across the EU. In particular:

- **Germany** (LfDI Baden-Württemberg and the DSK) has taken a stricter
  reading of ePrivacy Article 5(3), treating most analytics identifiers
  as consent-required regardless of first-party scope.
- **Italy** (Garante) has issued decisions requiring consent for
  first-party analytics identifiers where the operator could reasonably
  configure the analytics to be less intrusive.
- **Spain** (AEPD) largely tracks CNIL's more permissive line but has
  not formally endorsed a strictly-necessary reading for measurement.

If your DPO concludes that your jurisdiction requires consent for the
WiseGift identifier, the widget supports a **Consent Mode fallback**: it
reads the merchant's consent signal (on Shopify, `window.Shopify.
customerPrivacy`) and, when consent is denied, does not write the
identifier, does not emit events, and falls back to a per-request coin
flip for holdout assignment. Recommendations still render — measurement
does not. The fallback is configurable per tenant so a stricter merchant
can enable it without a code change on our side.

---

*Questions on this position should be routed to WiseGift's privacy
contact listed in the DPA. Merchants should treat this document as
input to their own legal review, not a substitute for it.*
