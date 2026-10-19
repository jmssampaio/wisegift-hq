# WiseGift Widget — EU Consent Position

> **Informational only. Not legal advice.** This paper is prepared by
> WiseGift's internal security-and-privacy function for onboarding
> conversations with pilot merchants and their advisors. It is not a legal
> opinion. A qualified attorney in the merchant's jurisdiction should sign
> off before a paying contract, and each merchant remains responsible for
> the consent posture on its own storefront.
>
> Basis: WiseGift's product specification (`docs/product/spec.md`), the
> system architecture (`docs/engineering/architecture.md`) and the data
> flows established from the backend code (Flyway V1–V15). If the data
> set or the region changes materially, this paper is superseded.

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
may disagree. We are building consent gating in which the shopper's own
analytics choice on your consent banner decides, and which fails closed
where no consent signal exists (see §6; widget PR #19, not yet shipped).

> *DRAFT — pending PO + attorney review.* The previous wording ("we ship
> an off-by-default fallback") described the consent fallback as shipped.
> It has not shipped. On Shopify the identifier also reaches the
> merchant's order records (see §2, §4).

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

**Order correlation (Shopify).**

> *DRAFT — pending PO + attorney review.* New paragraph.

To attribute orders, the widget also copies the identifier into the
visitor's Shopify cart as the attribute `_wg_session`. Shopify carries
it onto the resulting order, where it is visible to you (the merchant)
and to apps you authorise to read orders. If the visitor had no cart
yet, this creates one, which means Shopify sets its standard cart
cookie.

Once consent gating (§6) ships, neither the identifier nor the cart
attribute is created unless tracking is allowed. The cart write happens
when the widget is displayed. A follow-up proposal may also re-write the
attribute on page loads where a persisted session exists; this paper
will be updated if that is adopted.

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

> *DRAFT — pending PO + attorney review.* Caveat added.

**Caveat.** Writing the identifier into order records (§2, "Order
correlation") goes beyond pure audience measurement. It places the
identifier in a record the merchant can link to an identified customer,
and the copy in the cart may cause a cart cookie to be set. This weakens
the strictly-necessary reading above. For that reason the widget is
being changed to honour the shopper's analytics choice by default, and
to fail closed where no consent signal exists (§6). That change is in
widget PR #19 and has not shipped.

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
- We do not store or log the shopper's name, email, phone, address, IP
  address, or user-agent in our application data. The filtered
  `orders/create` webhook drops all customer PII at the receiver, before
  persistence. IP address and user-agent are carried on every HTTP
  request; our application never stores or logs them, though our EU
  hosting provider's request logs may hold them (see §6).
- WiseGift never joins session IDs to customer identities. Note that on
  Shopify the identifier is present on your order records (§2), so you
  can associate it with a customer. We recommend your privacy notice
  discloses this.
  > *DRAFT — pending PO + attorney review.* Replaces "We do not join
  > session IDs against the merchant's customer database", which read as
  > if no linkage were possible anywhere.
- We do not use session IDs for advertising or ad measurement.
- We do not sell or share event data with any third party for their own
  purposes. Sub-processors act only on WiseGift's documented instructions.

## 5. Retention and sub-processors

- **Retention.** Session identifier: 30-day sliding TTL in the browser.
  Server-side session and event rows: hard-deleted 120 days after the
  session was last seen / the event occurred, and in any event
  hard-purged 30 days after uninstall. Copies in Shopify carts follow
  Shopify's cart lifetime. Copies on orders follow your own order
  retention. WiseGift's order rows are deleted after 18 months.
  > *DRAFT — pending PO + attorney review.* Corrected to the decided
  > 120-day session/event retention (was "tenant's active lifetime"); the
  > pre-pivot `data.md` citation was removed (gap G11). The 120-day and
  > 18-month purges are not yet enforcing: the tenant purge runs in
  > dry-run (PR #79), so until it leaves dry-run nothing is deleted.
- **Region.** All storage and compute is in the **EU** at MVP (Neon EU,
  application hosting EU, telemetry EU). A US region is not deployed and
  would require a coordinated DPA and sub-processor update before use.
- **Sub-processors** relevant to shopper data: Neon (EU Postgres),
  OpenAI (intent embedding and LLM ranking — no session ID leaves our
  backend; the intent payload, including free-text interests, and
  product texts are sent to OpenAI's API, which is US-hosted), our EU
  hosting provider (Render), and the managed EU Redis cache. No
  monitoring/observability vendor is engaged at MVP (logs and cost
  telemetry sit in the EU Postgres database). The authoritative list
  ships with the DPA and is updated in-place.
  > *DRAFT — pending PO + attorney review.* Previously named Anthropic
  > and listed an EU monitoring provider that is not engaged at MVP
  > (gap G11; `sub-processors.md` row 8). The backend has used OpenAI for
  > ranking since PR 7d. On 2026-10-14 the PO decided to move to an
  > OpenAI EU data-residency project (`eu.api.openai.com`). That switch
  > has not been made: the backend still calls the US endpoint, so
  > "US-hosted" stays accurate until it is switched and verified.
- **Free-text interests.** The intent form has an optional free-text
  field, up to 5 interest chips. Its contents are **sent transiently to
  OpenAI** (above) on a live recommendation request and then reduced to
  a one-way SHA-256 hash that forms part of the 24-hour cache key. They
  are **not stored** with the session's events or on any other WiseGift
  table, and they are never written to the order or shared with the
  merchant. A visitor could type personal or sensitive information about
  themselves or the gift recipient, such as health or religion. We plan
  to show a short notice next to the field, e.g. "Don't include personal
  details". **This notice is planned and has not shipped.**
  > *DRAFT — pending PO + attorney review.* Corrected: free text is not
  > stored (gap G1). It is a transient transfer to OpenAI plus a one-way
  > hash. The Art. 9 locus is therefore the OpenAI transfer, not storage.
  >
  > **ATTORNEY QUESTION (Art. 9):** given that visitors may type
  > special-category data into the free-text field, and that this text is
  > sent transiently to a US-established processor (OpenAI), are a notice
  > and data minimisation enough? Or does Art. 9 GDPR require a further
  > measure, for example restricting free text to a fixed vocabulary or
  > filtering it before it is sent to OpenAI?

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

> *DRAFT — pending PO + attorney review.* The section below is aligned
> with widget PR #19 (`feature/g5-consent-gating`). That PR is open, not
> merged, and has **not shipped**. Do not hand this paper to a merchant
> DPO as describing current behaviour until it ships and is verified.
> This corrects the earlier draft, which said a per-tenant setting would
> "force this mode for every visitor". In PR #19 the setting only applies
> where no consent interface is available.

**Consent gating (in progress, PR #19, not yet shipped).** Consent
gating is the widget's default behaviour, not an opt-in for strict
jurisdictions. "Tracking" covers four things: the identifier in
`localStorage`, the `_wg_session` cart attribute, the `?wg_session=`
product-link marker, and event sending.

- **Where Shopify's Customer Privacy API is available, the visitor's own
  choice decides.**
  - Tracking runs only when `analyticsProcessingAllowed()` reports that
    analytics processing is allowed. That means the visitor consented, or
    no consent is required in their region.
  - If the visitor declines, or has not chosen yet where a choice is
    required, there is no tracking.
  - The merchant setting below cannot override the visitor's choice in
    either direction.
- **Where no consent interface is available** (a non-Shopify embed, or
  the interface failed to load), a per-tenant setting decides.
  - The setting, `consentModeRequired`, defaults to `true`.
  - With `true` (the default), nothing is minted, stored, written to the
    cart or sent as events.
  - Tracking runs only if the merchant has explicitly set it to `false`.
  - If the setting cannot be read, the widget treats it as `true`, so it
    fails closed.
- **While the signal is pending,** nothing is minted, stored, written or
  sent.
- **Without consent, earlier identifiers are cleared.** When tracking is
  definitively not allowed, the widget removes an identifier and a
  `_wg_session` cart attribute left by an earlier consented visit. This
  applies when the visitor declined, or when no API is available and the
  setting is `true`. Copies already carried onto past orders are not
  affected.
- **Recommendations always render.** Without tracking, holdout assignment
  falls back to a per-request coin flip on the backend. Recommendations
  still render, but measurement does not.

**Content request before any consent (in progress, widget
`feature/per-placement-hooks`, not yet shipped).**

> *DRAFT — pending PO + attorney review.* New paragraph. Accurate only
> once the conditions listed in the 2026-10-17 security-and-privacy
> review are met (explicit no-referrer / no-credentials on the request,
> backend behaviour verified on the shipping branch, hosting request-log
> retention confirmed). Do not hand to a merchant before then.

When an inline WiseGift block scrolls into view, the widget asks the
WiseGift API for your hook copy and appearance in the storefront's
language, so the block shows your copy instead of a placeholder. This
happens in every consent state, including before the visitor has chosen
and after they declined. Without tracking consent, this request:

- sends only the block's placement and the storefront language set by
  your theme, plus what every web request carries (IP address, browser
  user-agent, your storefront's origin);
- sends no WiseGift identifier and no cookie, and does not read or write
  the visitor's browser storage;
- returns, with the copy, a one-hour access token bound to a random
  identifier the server creates for that page. The widget keeps it in
  page memory only, to serve recommendations if the visitor asks for
  them. It is discarded when the page is left, and is never stored,
  written to the cart or used for events;
- creates no session or event record at WiseGift. WiseGift's
  application does not log the visitor's IP address or user-agent. Our
  EU hosting provider's request logs may hold them for
  **[RETENTION TO CONFIRM]**.

Our position is that this request is part of delivering the page content
the visitor asked to see, not tracking. It is therefore outside the
consent requirement or, if a supervisory authority treats it as access
to the terminal, strictly necessary.

> **ATTORNEY QUESTION (Art. 5(3) scope):** EDPB Guidelines 2/2023 read
> "storage" and "access" broadly, including script-triggered requests
> and data held in memory. Is a content request that also returns an
> in-memory, page-scoped token covered by the strictly-necessary
> exemption? Or must the pre-consent request return copy only, with the
> token minted when the visitor first interacts?

---

*Questions on this position should be routed to WiseGift's privacy
contact listed in the DPA. Merchants should treat this document as
input to their own legal review, not a substitute for it.*
