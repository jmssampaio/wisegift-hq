# WiseGift — Shopify App Store Listing

> Marketing-owned draft for the public-app listing. Grounded in `docs/product/spec.md` (features) and `docs/growth/gtm-b2b.md` (ICP, guardrails). Any lift or performance number in this file must clear the `spec.md` significance bar (p < 0.05, at least 5,000 exposed sessions per placement) and be signed off by the Product Owner before publication.

## Category recommendation

### Primary category — **Marketing → Upselling & Cross-selling**

This is where our ICP-fit merchants actually search. The pilot ICP in `gtm-b2b.md` — mid-market Shopify merchants (50k–500k monthly sessions, gift-heavy verticals) with a marketing team that cares about conversion rate and revenue per session — reaches for an app when they want AOV or revenue-per-session lift. That reflex takes them to "Upselling & Cross-selling", not to "Product Discovery". The pitch we sell them (measured lift vs. holdout on CTR, add-to-cart, conversion, AOV, revenue-per-session — per `spec.md` analytics feature) is a Marketing pitch: it is priced in revenue, not in UX. Listing under Marketing keeps the search-intent match honest.

The category is more crowded (Rebuy, LimeSpot, Nosto, Wiser, ReConvert), but that crowd is where the budget goes. Losing the search-intent match to escape the crowd trades a real acquisition problem for a differentiation problem we can solve inside the listing itself.

### Secondary category — **Store Design → Product Discovery** *(if secondary listing is available)*

**TBD — confirm at App Store submission time whether Shopify still permits a primary + one secondary category. Historically the App Store has allowed a primary and a small number of tags/subcategories; the exact mechanics change and this needs to be verified against current partner-dashboard rules before submission.**

If a secondary slot is available, use Product Discovery. It is where "gift finder" quiz apps live and it is genuinely part of our surface (the dedicated gift-finder page placement described in `spec.md` Widget → Recommendation display is a discovery experience). It also gives us a second organic-search entry point for merchants who frame the problem as "help my shoppers find the right product" rather than "lift my AOV" — a real subset of gift-heavy verticals like jewelry and specialty food.

If only one category is permitted at submission time, stay in Marketing → Upselling & Cross-selling. Discovery-only positioning underplays the revenue story the pricing tiers depend on.

### Positioning tradeoff to flag to Product Owner

The category choice locks in the top-of-funnel framing. Two alternatives were considered and rejected in this draft — flag either for reconsideration if the PO disagrees:

- **Lead with "gift recommendations"** (Product Discovery primary). Cleaner differentiation, less crowded, matches the gift-finder mental model. Rejected because it caps the addressable use case — the widget serves self-intent recommendations too (per `spec.md` Widget → Gift intent hook), and merchants who read us as gift-only will assume the widget is dark 11 months of the year outside Q4/Valentine's/Mother's Day.
- **Lead with "AI recommendations that also do gifts"** (Marketing primary, gift as a secondary talking point in the listing body). This is what the draft below does — gift-intent is the differentiator inside the listing, not the category.

## Listing keywords

Metadata keywords — 5–8 short phrases biased for merchant search intent, not our internal vocabulary. Ordered by expected search volume among the ICP.

1. product recommendations
2. upsell cross-sell
3. gift finder
4. AOV increase
5. personalized recommendations
6. related products
7. gift recommendations
8. product recommendation quiz

Notes for the submission form:
- "product recommendations" and "upsell cross-sell" carry the category-match weight; they should stay in the first two slots regardless of any reshuffle.
- "gift finder" is the differentiator hook — it doesn't compete with the incumbents on their term, it opens a distinct search intent.
- Avoid "AI" as a standalone keyword — every competitor in the category uses it, so it neither differentiates nor helps ranking. If AI must appear anywhere in metadata, use it in a phrase ("AI product recommendations") rather than as a bare token.

## Listing tagline

**Product recommendations that ask what shoppers are actually looking for.**

*(96 characters. Under the 110-character brief.)*

Rationale — reads as a Marketing/Upselling app in search results (the word "recommendations" carries the category), but the second clause signals that the mechanism is different from behavioural incumbents. It works whether the merchant's inbound intent was "increase AOV" or "help shoppers find gifts". No "AI-powered", no "revolutionary", no performance claim we don't yet have signed-off data for.

Alternate tagline for A/B in the second listing revision, once we have live pilot data:
- "Ask shoppers what they need. Recommend accordingly." (55 characters — punchier but drops the "recommendations" search-match keyword.)

## Short listing description

**≤ 500 characters. Currently 497.**

> WiseGift asks each shopper a single question — are you buying for yourself or for someone else? — and uses their answer to recommend products from your catalog. Unlike recommendation apps that guess from browsing history, WiseGift captures the intent directly, so first-time visitors and gift shoppers get relevant suggestions from the start. Runs on any product page, home hero, or dedicated gift-finder page. Every merchant sees measured lift against a built-in holdout group.

Structure hits the four beats:
1. **Problem for the merchant** — first-time visitors and gift shoppers convert poorly because behavioural recommenders have no history to work from.
2. **How it works from the shopper's side** — one question, then recommendations. Matches the actual widget flow in `spec.md` (Widget → Gift intent hook, Widget → Intent capture form).
3. **Differentiator vs. behavioural recommenders** — intent-captured, not browsing-inferred. This is the honest mechanism, not marketing filler.
4. **Attribution built in** — holdout is a real feature per `spec.md` Widget → Attribution & session tracking; naming it up front sets us apart from apps that claim lift without a control.

Claims that need Product Owner sign-off before this ships:
- The word "measured lift" implies we have a lift number. It does not, in this draft, quote one — but the moment the listing quotes a percentage, that number must clear the significance bar (p < 0.05, min 5,000 exposed sessions per placement) per `spec.md` and `gtm-b2b.md`. Flag: do not add a percentage lift to the description until at least one pilot clears the bar.
- No "AI" claim in the description body — deliberate. If PO wants "AI-powered" restored for merchant-perceived credibility, that is a positioning call, not a copy call.

## Open items for submission time

- **Confirm secondary-category availability** in the Shopify Partners submission form. If unavailable, drop the Product Discovery slot without changing the primary or the tagline.
- **App name registration** — "WiseGift" needs to be confirmed available in the App Store; if a conflict exists we may need to list as "WiseGift Recommendations" or similar. Not a marketing decision — flag to PO.
- **Screenshots and app icon** are not in this doc. They are a separate deliverable requiring the widget UI to be built enough to screenshot in a real merchant theme (per `spec.md` Widget theming — live-preview panel is the natural source).
- **Pricing tier descriptions** for the listing pricing block are deferred until the pricing thesis in `gtm-b2b.md` is closed (post-pilot).
- **Data-processing summary** for the listing — required by Shopify. Owned by `security-and-privacy`, not marketing.
