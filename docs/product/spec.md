# WiseGift — Specifications

> B2B rewrite. Supersedes the pre-2026-09-29 B2C spec, which described a consumer gift-discovery app on a shared affiliate catalog. Historical context is in `docs/decisions.md` (entries dated 2026-09-29).

## Vision

WiseGift sells **AI recommendation infrastructure to e-commerce merchants**, delivered as an embeddable frontend widget on the merchant's storefront. The widget hooks the shopper with a gift-framed question ("Are you looking for something for you or for someone else?"), captures intent, and calls WiseGift micro-APIs for personalised recommendations drawn from the merchant's own catalog. General recommendations for self-purchase are equally supported — the gift use case is the differentiator, not the whole surface.

**MVP platform: Shopify.** Follow-ons: VTEX, Salesforce Commerce Cloud, custom via REST.

---

## Product surface (MVP at a glance)

Three surfaces, one shared multi-tenant backend:

1. **Widget** — embedded JavaScript on the merchant's storefront (home / PDP / dedicated gift-finder page). Preact + Web Components, ≤ 50 KB gzipped, async, lazy.
2. **Merchant admin dashboard** — web app where the merchant installs, configures placements, edits intent-form copy, and sees the analytics dashboard. Hosted by WiseGift.
3. **Recommendation & event APIs** — the micro-APIs the widget calls to fetch recs and emit events. Same APIs power any future platform integration.

---

## Personas

### Merchant admin
The person at the merchant who installs the Shopify app, configures the widget, and reads the analytics. Usually the e-commerce manager or the growth/marketing lead. Not a developer.

### Widget shopper
Any visitor to the merchant's storefront who sees or interacts with a WiseGift widget. Anonymous by default — WiseGift never receives the shopper's identity.

---

## Platform scope

| Platform | MVP | Post-MVP |
|---|---|---|
| Shopify (App Store public app + Theme App Extension) | ✓ | — |
| Shopify (custom app install for pilots) | ✓ | — |
| VTEX | — | ✓ |
| Salesforce Commerce Cloud | — | ✓ |
| Custom / headless via REST | — | ✓ |

---

## Feature: Merchant onboarding — Shopify app install

### User Story
As a merchant admin, I want to install WiseGift from the Shopify App Store (or as a custom app during pilot) so that I can start showing gift recommendations on my storefront without engineering work.

### Acceptance Criteria
- [ ] Public App Store listing includes app icon, screenshots, pricing tiers, and a data-processing summary.
- [ ] Install flow uses standard Shopify OAuth; the merchant approves the requested scopes (`read_products`, `read_orders` for attribution, `write_theme_extensions` for widget injection).
- [ ] On successful OAuth, a `tenant` record is created in the shared WiseGift database with `tenant_id`, shop domain, Shopify shop ID, install timestamp, region (EU), and a placeholder vertical (defaults to `generic`).
- [ ] The merchant is redirected to the WiseGift admin dashboard's onboarding wizard.
- [ ] During the pilot phase, a merchant can install as a **custom app** created in their Shopify partner dashboard using the same OAuth flow — no App Store listing required.
- [ ] Uninstall via Shopify triggers WiseGift's `app/uninstalled` webhook; tenant is soft-deleted (retained 30 days for reactivation, then hard-purged with all shopper events).
- [ ] All data written during install is stored in the EU region.

### Installation gate and error screens: copy and support contact (ADM-1)

> **DRAFT — pending PO review (2026-10-16).** Copy for the screens in `embedded-admin-auth-phase1-plan.md` §4 (installation gate, not-embedded page) and §3.6 (error contract). Values in `[BRACKETS]` are placeholders **the PO fills**. Engineering must not invent them.

**Support contact (one value, used everywhere):**
- `[SUPPORT EMAIL]`: a WiseGift role address, for example `support@<wisegift domain>`. Not a personal inbox. **PO to fill.**
- `[SUPPORT RESPONSE TIME]`: for example "within 1 business day". **PO to fill.** If the PO prefers not to promise a time, the sentence that uses it is dropped.
- Optional `[SUPPORT URL]` (help page or form). If there is none at MVP, the copy works with the email alone.
- The same value is used on every admin surface that says "contact support": these screens, Catalog sync ("Contact support to run more often"), the data-export request (Resolved questions, "Merchant data export SLA"), and Account → Contact email.
- The address is shown as **visible, selectable text** as well as a `mailto:` link. Inside the Shopify admin iframe a `mailto:` link may not open a mail app, so the merchant must be able to copy the address.
- *Flag (security-and-privacy):* the mailbox provider behind `[SUPPORT EMAIL]` receives merchant messages, which may include personal data. It should be EU-hosted, and the merchant-contact privacy notice must name it.

| Screen (trigger) | Title | Body | Actions |
|---|---|---|---|
| **Setting up** (`409 installation_required`, then `POST /admin/v1/installation` in flight) | Setting up WiseGift… | This only takes a few seconds. | None. Non-interactive (App Store 2.3.2). A spinner with `role="status"` and `aria-busy="true"` |
| **Still trying** (`503 platform_unavailable`, automatic retries in progress) | Setting up WiseGift… | Shopify is taking longer than usual. We're still trying. | None |
| **Shopify unreachable** (`503` after the automatic retries are used up) | We couldn't reach Shopify | Shopify didn't respond in time. This is usually temporary. Try again in a minute. If it keeps happening, contact us at `[SUPPORT EMAIL]`. | **Try again** (re-runs the gate) |
| **Setup failed** (`502 installation_failed`) | We couldn't finish setting up WiseGift | Something went wrong on our side, and we've been alerted. Contact us at `[SUPPORT EMAIL]` and mention the error code below. We'll get back to you `[SUPPORT RESPONSE TIME]`. Error code: `installation_failed` | **Try again**; **Copy error code** |
| **Not embedded** (`window.shopify` missing, or `idToken()` rejects) | Open WiseGift from your Shopify admin | WiseGift runs inside your Shopify admin. Go to **Apps → WiseGift** in Shopify. Already there and still seeing this? An ad blocker or privacy extension may be blocking Shopify. Allow `admin.shopify.com` and reload. Need help? `[SUPPORT EMAIL]` | None. No data, no login form |
| **Unexpected error** (any other failure while the gate runs, including `403`) | Something went wrong | We couldn't load WiseGift. Reload the page to try again. If it keeps happening, contact us at `[SUPPORT EMAIL]`. Error code: `<code>` | **Reload** |

- [ ] The error code shown is the `code` from the backend `ErrorResponse`, or `network_error` when there is no response. No stack traces, IDs, tokens or URLs (the URL can carry `id_token`, plan S1).
- [ ] Title as the page's `<h1>`. On every error screen, focus moves to the title, and the body is announced once (`role="alert"` on the error screens, `role="status"` on the progress screens).
- [ ] "Try again" and "Reload" are real buttons, at least 44 × 44 px. "Copy error code" confirms with a polite status message ("Copied").
- [ ] Copy is EN at MVP; the admin is EN-only unless the PO decides otherwise (open question Q-C9).

---

## Feature: Merchant onboarding — Onboarding wizard

### User Story
As a merchant admin, I want a guided setup after install so that the widget is live on my storefront within 15 minutes without me needing to read documentation.

### Acceptance Criteria
- [ ] Step 1 — **Vertical selection**: the merchant picks their primary vertical from a fixed list (Fashion, Beauty, Home, Tech, Books, Food, Kids, Jewelry, Other). MVP treats all verticals identically at the intent-form level, but the choice is stored on the tenant record for future verticalised packs.
- [ ] Step 2 — **Catalog sync**: WiseGift kicks off the initial catalog import from the Shopify Admin API. A progress indicator shows product count ingested. First-time sync of up to 10k SKUs must complete in under 15 minutes; larger catalogs continue in the background and the wizard proceeds.
- [ ] Step 3 — **Widget placement**: the merchant picks at least one placement (Home hero, PDP recommendation slot, dedicated gift-finder page). Each placement is a Theme App Extension block the merchant enables in the Shopify theme editor via a "Configure in Shopify" deep link.
- [ ] Step 4 — **Preview**: an inline preview of the widget rendered against the merchant's own theme colours and typography. The merchant can adjust the intent-hook copy from a preset library (5 options in EN + ES + PT) or write custom copy.
- [ ] Step 5 — **Go live**: enabling the tenant flips `tenant.active = true`. From this moment the widget renders live for shoppers and events start being recorded.
- [ ] The onboarding wizard is resumable — closing the browser and returning drops the merchant back at the incomplete step.

> **DRAFT, pending PO review (2026-10-15):** Step 1 is proposed to become "What do you sell?" (Fashion / Home & Decor / General niche packs that pre-fill existing configuration). See [`hooks-and-packs.md`](hooks-and-packs.md) §3.4.
>
> **DRAFT, pending PO review (2026-10-16):** a new **Step 2 — Contact email** is inserted, and a confirmed contact email gates Go live. See "Contact email step and go-live gate" below.

### Contact email step and go-live gate

> **DRAFT — Q-C1..Q-C9 decided by PO 2026-10-16 (see decisions.md); copy and layout pending PO review (2026-10-16).** Written by `product-analyst` for **BE-3** (backend) and **ADM-3** (admin onboarding).
> **Sources:**
> - `decisions.md` 2026-10-15 "Embedded admin auth…", decision 5: an explicit contact email collected in onboarding, pre-filled from the shop email, for DPA breach notices and billing. The attorney words the notice.
> - `decisions.md` 2026-10-15 "…Phase 1 plan accepted…", PO-4: a **confirmed** contact email is required before go-live.
> - `embedded-admin-auth-phase1-plan.md` §3.4, §4 and §7 (BE-3, ADM-3).
> - `dpa-template.md` clause 13: WiseGift notifies the merchant of a personal-data breach within 72 hours. The DPA does not yet say *where* that notice goes; this address is the answer.
>
> The API and data-model blocks below are **proposals**. `backend-engineer` owns the contract and the schema.

#### User stories
- As a **merchant admin**, I want to tell WiseGift which address should receive important notices about my account and my shoppers' data, so that a security or data-protection notice reaches someone who will act on it.
- As a **merchant admin**, I want the address pre-filled from my Shopify store, so that this step takes seconds.
- As **WiseGift** (the merchant's processor), I want a confirmed contact for every live tenant, so that we can meet the DPA clause 13 breach-notice duty and reach the merchant about billing.

#### Where it sits in onboarding
Proposed wizard order (with the 2026-10-15 "What do you sell?" draft):

| # | Step | Note |
|---|---|---|
| 1 | What do you sell? | `hooks-and-packs.md` §3.4 |
| 2 | **Contact email** | New |
| 3 | Catalog sync | The sync already starts at install (plan §3.3 step 6), so it keeps running during steps 1–2 |
| 4 | Widget placement | |
| 5 | Preview | |
| 6 | Go live | Gated on a confirmed contact email |

Why second:
- **Not first:** "What do you sell?" is the step that shows the merchant their store in the product. Opening with an admin chore delays that.
- **Not last:** at Go live the merchant wants to launch. An unexpected form at that moment feels like a blocker. Early in the flow it takes about 10 seconds and is forgotten.
- **Covered early:** catalog data flows from install onwards. Having the contact before most of setup means WiseGift can reach the merchant about any incident during setup too.

- [ ] Step 2 is required to continue the wizard. There is **no "Skip for now"**. One fewer state to handle, and with the pre-fill it costs one tick and one click.
- [ ] The go-live gate (below) still applies as defence in depth. It covers tenants that reach Go live without the step: tenants created before ADM-3 ships (for example the staging tenant `wisegift-dev`), reinstalls, and direct navigation.
- [ ] The wizard resumes at Step 2 while `/me` → `tenant.contactEmailConfirmed` is `false` and Step 1 is complete (plan §3.4).
- [ ] The step is framed as a **setting**, not an account sign-up. No password, no "create account" wording. Built for Shopify 3.1.3 forbids an extra sign-up prompt (plan S5).

#### Step layout and copy (EN)

| Element | Copy / behaviour |
|---|---|
| Step heading (`<h1>` of the step) | Where should we send important notices? |
| Intro | We use this address for security and data-protection notices about your store, and for billing. We won't use it for marketing. *(The "no marketing" clause depends on Q-C3.)* |
| Field label (visible, always) | Contact email |
| Helper text, while the value equals the Shopify suggestion | Suggested from your Shopify store email. Change it if someone else should get these notices. |
| Helper text, always | Use an inbox someone checks. A shared address, like privacy@yourstore.com, keeps working when people change roles. |
| Confirmation checkbox | This address is monitored and can receive important notices about this store. |
| Privacy notice | **PENDING attorney wording.** Placeholder: `[ATTORNEY: short notice on how WiseGift uses and keeps this address]` plus a link "How we use your contact email" → `[MERCHANT-CONTACT PRIVACY NOTICE URL — PENDING]`. It sits directly under the checkbox, before the button. |
| Primary button | Confirm and continue |
| Success | The wizard moves to Step 3 and announces "Contact email confirmed". |

#### Field rules
**Pre-fill (suggestion only)**
- [ ] On entering the step, the admin requests the live Shopify shop email (`shop-contact-suggestion`, below) and pre-fills the field with it.
- [ ] The suggestion is **never stored by WiseGift** until the merchant confirms. It is not saved as a draft, not cached server-side, and not logged. If the merchant leaves the step without confirming, nothing is stored. On return the suggestion is fetched live again.
- [ ] If a confirmed address already exists (resume, reinstall, revisiting the step), the field shows the **stored** address, not the suggestion.
- [ ] If the suggestion is empty, invalid, or the request fails, the field starts empty with no error. Only the "always" helper text shows. The merchant types an address.
- [ ] The "Suggested from your Shopify store email" helper disappears as soon as the value differs from the suggestion.

**Validation**
- [ ] Leading and trailing whitespace is trimmed. The address is otherwise stored exactly as entered: no lower-casing, because the local part is technically case-sensitive.
- [ ] Valid means **one** address: exactly one `@`, a non-empty local part of at most 64 characters, a domain containing at least one dot, no spaces or control characters, and at most **254** characters in total. Display-name forms (`Jane <jane@shop.com>`) and lists (`a@x.com, b@y.com`) are rejected.
- [ ] The backend is authoritative and applies the same rule. If the client accepts and the server rejects, the server message is shown on the field. *Note for backend:* Bean Validation `@Email` alone accepts `jane@localhost`; the "domain contains a dot" rule needs adding.
- [ ] Validation runs on blur and on submit, not on every keystroke. Once a field error is shown, it re-checks on each change, so it clears as soon as the value is fixed.
- [ ] The checkbox must be ticked to submit.
- [ ] Error messages:

| Condition | Message (shown under the field or checkbox) |
|---|---|
| Empty | Enter a contact email. |
| Invalid format | Enter an email address like name@yourstore.com. |
| Too long | This address is too long. Use one with 254 characters or fewer. |
| More than one address | Enter one address only. To reach several people, use a shared inbox. |
| Checkbox not ticked | Tick the box to confirm this address is monitored. |

#### What "confirmed" means at MVP: recommendation

**Recommendation: an explicit in-app confirmation** (the ticked checkbox plus the "Confirm and continue" click). No emailed verification link at MVP.

| | In-app confirmation (recommended) | Emailed verification link |
|---|---|---|
| Proves | The merchant deliberately chose and vouched for the address | The address exists and someone can read it |
| New vendor | None | An email-sending provider (for example Postmark or Amazon SES EU). It must be EU-hosted (CLAUDE.md EU-only rule). It processes merchant contact data, and every notice later sent through it |
| Legal and doc work | None beyond the privacy notice | The vendor must be assessed and listed. Whether it belongs on `sub-processors.md` (which covers shopper data under the DPA) or only in WiseGift's own merchant-contact privacy notice is a **security-and-privacy / attorney** call. DPA clause 9.3 requires 30 days' notice to merchants for new sub-processors |
| Engineering | One timestamp | Tokens, expiry, resend, bounce handling, SPF/DKIM/DMARC on a WiseGift domain, a "pending verification" state in the wizard and the gate |
| Onboarding time | About 10 s | Leaves the app to open an inbox. Puts the 15-minute target at risk and adds a drop-off point |
| Main risk | A typo or an abandoned inbox goes unnoticed until a notice bounces | Lower |

Why in-app wins at MVP:
- Pilots are onboarded hands-on through the Design Partner Program, so the PO can check the address on the kickoff call.
- The pre-fill comes from an address Shopify already holds, which limits typos.
- The checkbox stops a reflexive click-through.
- Above all, it adds no vendor and no DPA or sub-processor change before the first pilot. M3·4 attorney review is already the longest lead-time item.

- [ ] **Definition:** a tenant's contact email is *confirmed* when `contact_email` is non-null **and** `contact_email_confirmed_at` is non-null. Both are written together, in one request, only when the merchant submits with the checkbox ticked.
- [ ] There is no stored "unconfirmed" address. The only states are "none" and "confirmed".
- [ ] **Revisit trigger:** before the public App Store launch (self-serve installs with no kickoff call), or as soon as WiseGift adopts an email-sending provider for any other reason. If verification is added then, it gets its **own** column (`contact_email_verified_at`). `confirmed_at` keeps its meaning.

#### Go-live gate
- [ ] A tenant cannot be set live (`tenants.active = true`) unless its contact email is confirmed. This applies to **every** activation path: wizard Step 6, any later "Go live" or resume toggle, and re-going-live after a reinstall (PO-3).
- [ ] **Enforced in the backend**, not only hidden in the UI. *Proposal:* the go-live command (not built yet, `architecture.md` §9.3) returns `409` with `code: contact_email_required` when the contact is not confirmed.
- [ ] On the Go live step, a short checklist shows the prerequisites. The contact row reads either "Contact email: jane@shop.com ✓ Change" or "Contact email: not confirmed".
- [ ] When the contact is not confirmed, an inline warning sits above the Go live button:
  - **Title:** Add a contact email before going live
  - **Body:** We need an address for important notices about your store, such as security and data-protection notices. It takes a few seconds.
  - **Link:** Add contact email → Step 2, with focus on the field.
  - After confirming there, the merchant is returned to **Step 6** (not Step 3), with focus on the Go live button and the status message "Contact email confirmed. You can go live now."
- [ ] The Go live button uses `aria-disabled="true"` plus `aria-describedby` pointing to the warning, rather than the `disabled` attribute, so keyboard and screen-reader users can reach it and hear why. Activating it moves focus to the warning title.
- [ ] If the backend returns `409 contact_email_required` anyway (for example a stale page), the same warning is shown.

#### Editing later (Account → Contact email)
- [ ] The Account screen (Merchant admin — Account & billing) has a **Contact email** card showing the address and "Confirmed on 16 Oct 2026", in the merchant's browser locale, plus a **Change** button.
- [ ] **Change** turns the card into the same field, checkbox, helper text and privacy notice as Step 2. The field is pre-filled with the **stored** address (no Shopify suggestion here). The checkbox starts **unticked**, so each change is a fresh confirmation. Buttons: **Save** and **Cancel**. Cancel restores the stored value and returns focus to Change.
- [ ] Saving writes the new address and a new `contact_email_confirmed_at` atomically. Toast: "Contact email updated".
- [ ] Saving the **same** address again with the box ticked is allowed. It refreshes `confirmed_at` ("still correct").
- [ ] The address **cannot be removed**, only replaced. A live tenant must always have one. The card states: "To remove this address, uninstall WiseGift. It is deleted with your store's data 30 days after uninstall."
- [ ] Changing the address never takes a live widget offline.
- [ ] **Audit:** every confirmation writes an audit record: tenant, timestamp, and action (`set` for the first confirmation, `changed` for a different address, `reconfirmed` for the same one).
  - The record does **not** contain the address, old or new (data minimisation). It also has no staff identity, because decision 4 forbids storing Shopify user IDs.
  - Whether the previous address must be kept as evidence for a breach notice is attorney question Q-C5.
  - At MVP, WiseGift sends no "your contact email changed" mail to the old address, because there is no email-sending provider.

#### Reinstall
- [ ] On reinstall within 30 days (PO-3), the stored address and `confirmed_at` are kept. Step 2 shows the stored address with its "Confirmed on…" date and a **Keep this address** button, alongside Change. *Keep this address* counts as a re-confirmation (`reconfirmed`). *Proposal; see Q-C6.*
- [ ] After a hard purge (> 30 days), the merchant starts as a new tenant and Step 2 behaves as a first install.

#### Privacy and retention
- [ ] The address is used only for: personal-data breach notices (DPA clause 13), other notices the DPA requires (pending Q-C3: sub-processor changes under clause 9.3, region changes under 10.3), essential service and security notices, and billing. It is **not** used for marketing.
- [ ] It is never logged, never included in error bodies or analytics, never part of CSV exports, and never sent to OpenAI or any other sub-processor.
- [ ] It is not returned by `/me`. Only the boolean `contactEmailConfirmed` is.
- [ ] Retention: it lives on the `tenants` row, so it is kept while the tenant exists, including the 30-day soft-delete window (a breach affecting retained data must still be notifiable). The tenant hard purge (M3·1) deletes the `tenants` row, and with it the address. The audit records are tenant-scoped and are purged with it. Backups follow DPA clause 14.4.
- [ ] *Flag (security-and-privacy / attorney):* WiseGift is the **controller** of this address, not the merchant's processor: it is a merchant staff or role contact, not shopper data. It therefore needs WiseGift's own privacy notice for merchant contacts (purpose, legal basis, retention, recipients). It is an exception to decision 4 ("no merchant-staff personal data") explicitly made by decision 5. The DPA should name this contact as the clause 13 notice address. The shared-inbox helper text reduces how often the address is personal data at all.

#### States and errors (Step 2 and the Account card)

| State | Behaviour |
|---|---|
| Loading the suggestion or the stored value | Field disabled with a skeleton, under 1 s target. If it takes longer than 3 s, the field is enabled empty and the late response is ignored if the merchant has started typing |
| Suggestion unavailable | Empty field, no error (above) |
| Submitting | Button shows a spinner after 300 ms and sets `aria-busy="true"`; the field and checkbox are read-only. No double submit |
| `400` validation from server | Server message shown on the field (or on the checkbox for `confirmation_required`); focus moves to the field |
| `503` / network failure | Inline banner above the button: "We couldn't save your contact email. Check your connection and try again." The entered value and checkbox are kept; nothing is stored |
| `401` | Handled by the admin client (one retry with a fresh token, plan §4). If it still fails, the "Unexpected error" screen (install feature above) |
| `409 installation_required` | Back to the installation gate (plan §4) |
| Success | Step 2: advance to Step 3 (or back to Step 6 when coming from the gate). Account: card returns to read mode with the new date and a toast |

#### Accessibility
- [ ] Native `<input type="email" autocomplete="email">` with a visible `<label>`. Helper text, privacy notice and error are linked with `aria-describedby`. On error: `aria-invalid="true"` and the message as text, never colour alone.
- [ ] Native checkbox with a visible, clickable label. Its error is linked with `aria-describedby`.
- [ ] On entering the step, focus goes to the step heading. On a submit error, focus moves to the first invalid control. Success messages are announced through a `role="status"` region.
- [ ] All controls are keyboard-operable, at least 44 × 44 px, with a visible focus indicator; they follow the admin's existing component styles. Text contrast at least 4.5:1, and control borders at least 3:1.
- [ ] Works at 320 px width and at 200% zoom without loss of content.

#### Data model: proposal for `architecture.md` §8 (`tenants`)

> **PROPOSAL — DRAFT, pending PO review; `backend-engineer` owns the schema.** Not yet copied into `architecture.md` §8. Once accepted, it lands there before BE-3 merges (plan §3.4 prerequisite).

| Column | Type | Notes |
|---|---|---|
| contact_email | VARCHAR(254) | Nullable. The merchant-confirmed contact for DPA notices and billing. Trimmed, otherwise stored as entered. Written only together with `contact_email_confirmed_at`. Never logged. Controller data of WiseGift (not shopper data). Deleted with the `tenants` row at hard purge (M3·1) |
| contact_email_confirmed_at | TIMESTAMPTZ | Nullable. Set to `now()` on every merchant confirmation (first set, change, re-confirm). "Confirmed" = both columns non-null. Gates go-live |

- **`contact_email_source`: recommended NOT to add at MVP.** Its values would be `shop_suggestion` or `merchant_entered`. It would tell us how often merchants keep the Shopify address, but nothing in the product or the DPA duty depends on that. To fill it reliably the server would need to fetch the shop email again at save time, an extra Shopify call. Easy to add later. PO may overrule (Q-C7).
- **Audit table (proposed name `tenant_contact_email_events`):** `id`, `tenant_id UUID NOT NULL`, `action VARCHAR(16)` (`set` / `changed` / `reconfirmed`), `occurred_at TIMESTAMPTZ`. No address column. Tenant-scoped, purged with the tenant. This follows the same pattern as the D4 holdout-split audit.
- **Retention row** for §8 "Data retention": "`tenants.contact_email`: kept for the tenant's lifetime, including the 30-day soft-delete window; deleted by the tenant hard purge (M3·1)."
- The plan's migration name `V7__tenants_contact_email.sql` may need renumbering: decisions.md 2026-10-15 already lists a "V7 consent backfill". `backend-engineer` to check.

#### API contract sketch for BE-3

> **PROPOSAL — DRAFT, pending PO review. The backend defines the contract.** This sketch starts from plan §3.4, with three deviations, all marked. The tenant comes only from the verified principal (plan §3.1).

- `GET /admin/v1/tenant/contact` → `200 { contactEmail: string | null, confirmedAt: string (ISO-8601 instant) | null }`.
- `PUT /admin/v1/tenant/contact`, body `{ contactEmail: string, confirmed: true }`.
  - Sets `contact_email` and `contact_email_confirmed_at = now()` atomically, and writes the audit record.
  - Errors: `400 invalid_contact_email` (format or length) and `400 confirmation_required` (`confirmed` missing or not `true`).
  - **Deviation 1:** the required `confirmed: true` literal. Plan §3.4 has `{ contactEmail }` only. The literal makes the merchant's act explicit in the contract, so no future client can silently save the Shopify suggestion. That would break the "never stored until confirmed" rule.
  - **Deviation 2:** respond `200` with the GET body instead of `204`, so the UI can show the new "Confirmed on" date without a second call. This matches the widget-config pattern, where mutations return the full resource. Either is fine for the UX.
- `GET /admin/v1/platform/shop-contact-suggestion` → `200 { suggestedEmail: string | null }`.
  - Read live from Shopify (`shop.email`), never persisted, never logged.
  - **Deviation 3:** if Shopify fails or the email is absent, return `200 { suggestedEmail: null }` rather than an error, because the UI treats both cases the same (empty field). The failure is logged without the address.
- `GET /admin/v1/auth/me` → `tenant.contactEmailConfirmed: boolean` (plan §3.6). The address itself is not in `/me`.
- Go-live command (not built): `409 contact_email_required` when not confirmed (gate above).
- All responses: `Cache-Control: no-store` (plan F-F). Tenant-isolation tests: tenant A can neither read nor write B's contact (plan §3.7 `TenantContactIT`), plus a test that the suggestion endpoint persists nothing.

#### Open questions (contact email)

| # | Question | Owner |
|---|---|---|
| Q-C1 | Accept the in-app confirmation (checkbox + click) for MVP, with the revisit trigger "before public App Store launch"? | PO |
| Q-C2 | Accept Step 2 placement and "no skip"? | PO |
| Q-C3 | Allowed uses: breach notices + billing (decided). May it also carry other DPA notices (sub-processor changes, clause 9.3; region change, 10.3) and essential service notices? Recommendation: yes; never marketing without a separate opt-in. | PO + attorney |
| Q-C4 | Privacy notice for merchant contacts (WiseGift as controller), its URL, and the short in-app wording. The DPA should name this address as the clause 13 notice point. | attorney (via security-and-privacy) |
| Q-C5 | Must WiseGift keep the previous address (and when it changed) as evidence of where a notice was sent? The audit proposal stores timestamps only. | attorney |
| Q-C6 | On reinstall, keep the stored address with a one-click "Keep this address", or force a fresh entry? | PO |
| Q-C7 | Add `contact_email_source`? Recommendation: no. | PO + backend-engineer |
| Q-C8 | Does the widget serve (and record events) while `tenants.active = false`? If it does, shopper-data processing starts before Go live, and the gate does not protect what it is meant to. | backend-engineer + frontend-engineer |
| Q-C9 | Admin language: EN-only at MVP, or EN/ES/PT like the widget? Affects all copy above. | PO |
| Q-C10 | Billing runs through Shopify App Store billing (pilots are free). What billing communication does WiseGift itself send to this address? Only the Stripe safety net? | PO + marketing-manager |

---

## Feature: Merchant onboarding — Catalog sync

### User Story
As a merchant admin, I want my product catalog to stay in sync with WiseGift automatically so that recommendations always reflect what's actually available in my store.

### Acceptance Criteria
- [ ] Initial full-catalog import at install using Shopify Admin API `products.json` (paginated, respecting Shopify's rate limits).
- [ ] For each product WiseGift stores: `tenant_id`, `platform_product_id`, `slug`, `title`, `description`, `price`, `currency`, `image_url`, `availability`, `product_type`, `tags`, `variants[]` (id, price, availability), `last_seen_at`.
- [ ] Product embeddings are computed at ingest for each product (title + description + product_type + tags). Vector column on Neon pgvector.
- [ ] The following Shopify webhooks are subscribed and processed within 60 seconds: `products/create`, `products/update`, `products/delete`, `inventory_levels/update` (for availability).
- [ ] On `products/update` or `products/delete` the affected product row is updated; embedding is recomputed only when the description or title changes.
- [ ] On `inventory_levels/update` the `availability` field is updated but embedding is untouched.
- [ ] Unavailable products (all variants out of stock or archived) are excluded from recommendation candidates but retained in the DB.
- [ ] Nightly reconciliation job re-fetches the catalog and diffs against the DB to catch any webhook that was missed.
- [ ] Catalog size at any time is visible to the merchant on the admin dashboard.
- [ ] Merchant-facing sync controls and visibility (manual resync triggers, sync-status panel, webhook health) live in the **Merchant admin — Catalog sync controls** feature.

---

## Feature: Widget — Embed and lifecycle

### User Story
As a widget shopper, I want the widget to load without slowing down the merchant's page so that my browsing experience is not disrupted.

### Acceptance Criteria
- [ ] Widget bundle is served from a WiseGift-controlled CDN with cache headers that permit long TTL + hash-based cache-busting.
- [ ] The Theme App Extension block includes a single `<script async src="…/widget.js">` tag (or Shopify's block-native equivalent). No render-blocking assets.
- [ ] Total gzipped payload of `widget.js` is ≤ **50 KB**.
- [ ] The widget renders inside a `<wisegift-widget>` **Web Component with Shadow DOM** — all styles are isolated from the merchant's theme.
- [ ] Time-to-interactive of the widget (from script fetch complete → widget accepts user input) is < **100 ms** at p95.
- [ ] Recommendation content is **lazy** — no `/recommendations` API call is made until either (a) the widget scrolls into the viewport (IntersectionObserver) or (b) the shopper interacts with the intent hook.
- [ ] The widget passes a Lighthouse Performance audit on a representative merchant page with score ≥ 90 with the widget enabled vs. without.

---

## Feature: Widget — Gift intent hook

### User Story
As a widget shopper, I want a friendly question that helps the widget understand whether I'm shopping for myself or a gift so that the recommendations match my intent.

### Acceptance Criteria
- [ ] The widget renders a hook prompt (default: "Are you looking for something for you, or for someone else?") with two primary buttons: **For me** and **For someone else**.
- [ ] Merchant admins can override the prompt copy from a preset library or with custom text (Merchant admin — Widget configuration feature).
- [ ] Selecting **For me** routes the shopper to the general-recommendations intent flow (budget + interests optional; can also submit with no additional input on a PDP-context placement).
- [ ] Selecting **For someone else** routes to the gift-intent flow (recipient relationship, occasion, budget, optional interests).
- [ ] Both flows share the same downstream API — the intent object carries an `intent_mode: "self" | "gift"` field.
- [ ] The hook is skippable on PDP placements: if the shopper does nothing, the widget still shows recommendations based on the current product context after 3 seconds (fires a widget-shown event but no intent capture).
- [ ] Widget copy is available in EN, ES, PT at MVP.

> **DRAFT, pending PO review (2026-10-15):** per-placement hook layouts (`home_hero`, `collection_header`, `pdp_slot`, `gift_finder`, `floating`), with eyebrow / headline / subline copy, states, accessibility, theming and measurement constraints, are specified in [`hooks-and-packs.md`](hooks-and-packs.md) §2.

---

## Feature: Widget — Intent capture form (generic MVP)

### User Story
As a widget shopper, I want a short form to describe what I'm looking for so that recommendations are relevant to me or my gift recipient.

### Acceptance Criteria
- [ ] **Self-intent form** fields (all optional): budget (min/max), interests (free-text chips, max 5).
- [ ] **Gift-intent form** fields: recipient relationship (Partner / Family / Friend / Colleague / Other — required), occasion (Birthday / Anniversary / Wedding / Baby / Christmas / Housewarming / Just because / Other — required), budget min/max (required), interests (free-text chips, max 5, optional).
- [ ] The form is a single scrollable card, not a multi-step wizard — form completion in < 20 seconds is the design target.
- [ ] The form's field schema is stored per tenant and versioned (`intent_form_schema_version` on the tenant record) so verticalised packs (post-MVP) can be introduced without breaking existing widgets.
- [ ] Submit button is disabled until required fields are filled; validation is inline.
- [ ] Submitting fires the recommendation call and shows a loading state; recommendations appear in < 1 second at p95.

---

## Feature: Widget — Recommendation display

### User Story
As a widget shopper, I want to see a small set of relevant product recommendations so that I can quickly decide what to buy.

### Acceptance Criteria
- [ ] Recommendations render as a horizontal scrollable strip of product cards on desktop; vertical stack on mobile.
- [ ] Each card shows product image, title and price. The whole card is the link to the merchant's own product page (`/products/{slug}` on the shop's storefront domain), and it opens in a **new tab**, so the widget drawer stays open and the shopper can keep comparing (`decisions.md` 2026-10-07, item 3; matches the widget code).
- [ ] Because the link opens a new tab, the card's accessible name tells screen-reader users so (for example "Product title, opens in a new tab"). A small visual new-tab indicator is optional and must not crowd the card.
- [ ] Default set size is 5 products. Configurable per placement (3–10) by the merchant.
- [ ] If fewer than the requested number of candidates pass the ranking threshold, the widget shows the products it has (never pads with irrelevant items).
- [ ] Clicking a product card fires a `product_clicked` event. Because the current page stays open, the event does not depend on an unload beacon.
- [ ] If the recommendation call fails or returns zero results, the widget renders a graceful fallback (top-N merchant bestsellers or precomputed cold recs for the current PDP), never an error message visible to the shopper.

---

## Feature: Widget — Attribution & session tracking

### User Story
As a merchant admin, I want the widget's impact on my revenue to be measurable so that I can trust WiseGift's lift claims.

### Acceptance Criteria
- [ ] **Consent gating (pilot gate, `decisions.md` 2026-10-14 "`_wg_session` order join … G5").** Each tenant has a `consentModeRequired` flag (`tenants.consent_mode_required`), **default `true`** (fail closed). When it is `true`, the widget reads the Shopify Customer Privacy API (`analyticsProcessingAllowed()`) before any measurement. Without consent: no session ID, no localStorage write, no cart attribute, no events. The widget still renders for the shopper in degraded mode, with a per-request holdout coin flip (2026-09-29 "Widget consent for EU"). Degraded shoppers are outside the measured population. G5 must ship (widget + backend) before the widget runs on any pilot storefront.
- [ ] **Consent ordering.** The widget learns `consentModeRequired` only from the session bootstrap response (`POST /widget/v1/session`, architecture §9.2), and that response already contains a `sessionId`. The widget therefore must not persist that `sessionId`, write the `_wg_session` cart attribute, append the `wg_session` URL marker, or emit events until it has read the flag and, when the flag is `true`, confirmed consent. Without consent, the ID is kept in memory for the current page only.
  - *Open (frontend-engineer + security-and-privacy):* may a previously stored `wg_session` be read and sent on the bootstrap call before the flag is known? See architecture §7.
- [ ] **`consentModeRequired = false` (PENDING PO + attorney confirmation).** This is the legitimate-interest mode: the widget measures without a consent signal, under the 2026-09-29 reasoning (first-party UUID, no PII, 30-day TTL, EU-only, GDPR Art. 6(1)(f)). The 2026-10-14 security review found that the cart attribute creates a join key to the merchant's identified customer, and that weakens this reasoning. So no tenant is switched to `false` until the PO and an attorney confirm the mode is still defensible.
- [ ] If measurement is allowed (see consent gating), the widget stores an **anonymous session ID** (UUID v4, 30-day sliding TTL) in localStorage under the key `wg_session` and re-sends it on every bootstrap. The ID is the one the session bootstrap returned: the backend honours a valid stored ID, or mints one. No merchant customer identity is attached. If localStorage is blocked, no cart attribute is written (2026-10-14, R4).
- [ ] The session is deterministically assigned to **exposed** or **holdout** by the backend from the session ID, and returned as `isHoldout` at bootstrap. The widget never computes its own split. The default split is **90% exposed / 10% holdout** for every tenant. A design partner may opt in to **80/20** as an explicit, recorded merchant choice, and is told that 20% of triggered traffic gets the placebo (`decisions.md` 2026-10-14, M1·2 D4). Assignment is stable across placements and page loads for the session lifetime. A split change applies only to sessions first shown after the change. The server-side `sessions.is_holdout` value is authoritative for measurement.
- [ ] Holdout sessions see the widget in a **placebo state**: the hook and form render identically, but after intent submission the recommendation slot **collapses neutrally**. It shows no "no matches" copy and no WiseGift-generated recs (`decisions.md` 2026-10-14, M1·2 D6, replacing the earlier "hidden or merchant static fallback" wording). Placebo is invisible to the shopper. Required before the first pilot readout.
- [ ] Every widget interaction fires a typed event to the WiseGift event API: `widget_shown`, `widget_engaged`, `intent_submitted`, `product_clicked`, tagged with `tenant_id`, `session_id`, `placement`, `intent_mode`, `is_holdout`, timestamp.
- [ ] The `orders/create` Shopify webhook is received by the WiseGift backend for every merchant order. WiseGift extracts only `order_id`, `line_items[].platform_product_id`, `line_items[].quantity`, `line_items[].price`, `total_price`, `currency`, `created_at`, and correlates the order to a session through a widget-planted marker. Sources are checked in this order (`decisions.md` 2026-10-14, R1):
  1. the **`_wg_session` cart attribute**, which reaches the order's `note_attributes`;
  2. the legacy `wg_session` cart attribute;
  3. the `wg_session=<id>` URL marker appended to rec-card links (`landing_site` / `referring_site`).

  The widget writes the cart attribute **identically for both arms**, so holdout orders can be attributed. The URL marker exists only for exposed clicks. Markers must be strict v4 UUIDs (R2). Customer PII fields are dropped at ingest and never persisted. *Note:* the M1·2 spec (§3.1, gap G2) excludes URL-marker orders from lift, because the URL marker is exposed-only.
- [ ] Attribution report joins events + orders on `session_id` (tenant-scoped) within a **7-day attribution window** anchored on the session's **last symmetric widget event** before the order (`widget_shown`, `widget_engaged` or `intent_submitted`). The order must also come **after the session's first trigger event** in the range. The window does **not** anchor on `product_clicked`, because holdout sessions cannot produce clicks (`decisions.md` 2026-10-14, M1·2 D3). Full rule: `docs/engineering/m1-2-lift-measurement.md` §3.2.
- [ ] Merchant dashboard renders KPIs against the holdout (see Analytics dashboard feature).

---

## Feature: Widget — Fallbacks and offline behaviour

### User Story
As a widget shopper, I should never see a broken widget so that my confidence in the merchant is not undermined.

### Acceptance Criteria
- [ ] If the `widget.js` script fails to load, the Theme App Extension block renders nothing (empty container). The merchant's page layout is unaffected.
- [ ] If the recommendation API times out (> 2 seconds), the widget serves precomputed cold recs from local widget cache (populated on previous session) or hides the recommendation slot.
- [ ] If the tenant's kill switch is triggered (see Cost guardrails), the widget renders precomputed cold recs only, with no live LLM calls. Shoppers cannot tell.

---

## Feature: Merchant admin — Widget configuration

### User Story
As a merchant admin, I want to control what the widget says and where it appears so that it matches my brand.

### Acceptance Criteria
- [ ] Configuration lives in the WiseGift admin dashboard under Widget → Configuration.
- [ ] The merchant can edit the **hook prompt copy** (up to 200 characters, or pick from 5 language-localised presets).
- [ ] The merchant can toggle the **intent form fields**: relationship, occasion, budget, interests are individually toggleable (required-vs-optional-vs-hidden), with defaults matching the MVP form.
- [ ] The merchant can set the **number of recommendations** shown per widget instance (3–10, default 5).
- [ ] The merchant can pick brand accent colour and border radius; other styling matches the storefront theme via inherited CSS variables.
- [ ] Save persists to the tenant's widget config; changes propagate to live widgets within 5 minutes (config is fetched with the widget bundle and cached).
- [ ] All fields are inline-validated.
- [ ] Deep visual look-and-feel (colours, fonts, spacing, shadow) is handled in the **Merchant admin — Widget theming** feature; this screen links out to Theming rather than duplicating those controls.

> **DRAFT, pending PO review (2026-10-15):** a "Store type" (niche pack) card, per-placement hook copy tabs (eyebrow / headline / subline) with live preview, and pack-driven intent-question labels are specified in [`hooks-and-packs.md`](hooks-and-packs.md) §3.

---

## Feature: Merchant admin — Widget theming

### User Story
As a merchant admin, I want the widget to look like it belongs on my storefront without me writing CSS so that shoppers see a native experience and I can override specific tokens when the auto-inheritance is not enough.

### Acceptance Criteria
- [ ] A **Theming** screen lives in the admin dashboard under Widget → Theming, distinct from Widget → Configuration (copy / placements / rec count).
- [ ] On Shopify tenants, an **auto-inherit info banner** at the top reads "Colours and fonts inherit from your Shopify theme when your theme provides them. The live preview below shows the actual result." No "N of X" claim, no per-token source detection — the live preview is the authoritative signal. The banner is hidden entirely for non-Shopify tenants (VTEX / SFCC post-MVP) where auto-inheritance does not apply. See `docs/decisions.md` 2026-09-29 "Widget theming UX questions closed".
- [ ] Each of the 8 design tokens (`accent_color`, `text_color`, `background_color`, `font_family`, `heading_font_family`, `border_radius_px`, `spacing_scale`, `card_shadow`) is exposed as an individually editable control on the Theming screen, with the control matched to the token type: colour picker with hex input for the three colour tokens, free-text input for the two font-family tokens, numeric stepper (0–24) for `border_radius_px`, preset dropdown (`compact` / `cozy` / `roomy`) for `spacing_scale`, preset dropdown (`none` / `subtle` / `medium`) for `card_shadow`.
- [ ] Every token control shows its current effective value and its source next to the control: `Inherited from your theme` (Shopify only), `Custom override`, or `WiseGift default`.
- [ ] A **live preview panel** renders the widget with the current settings applied and updates within 500 ms of any control change, without requiring a save; the preview mirrors the widget's real Shadow DOM isolation so what the merchant sees is what shoppers will see.
- [ ] Each token has a unified **Reset** control that clears the merchant's override for that single token; the widget's fallback chain then applies (theme variable → WiseGift default). Copy is uniform across tenant types; a hover tooltip explains "On Shopify, resets to your theme's value if provided, otherwise to the WiseGift default." The live preview updates immediately.
- [ ] Two **presets** at the top of the screen give the merchant a one-click starting point: **Auto-inherit from theme** (clears every override in one click, with an inline confirmation) and **WiseGift neutrals** (fills all 8 tokens with WiseGift's neutral defaults, which pins the widget to a WiseGift look regardless of theme — for merchants whose theme variables produce a poor result). Undo is available until the merchant navigates away; no destructive dialog on either preset.
- [ ] If `text_color` on `background_color` fails WCAG AA contrast (< 4.5:1 for body text), a non-blocking warning appears next to those controls ("Contrast ratio 3.2:1 — shoppers with low vision may struggle to read recommendations") with a "Suggest a compliant colour" affordance; the merchant can still save.
- [ ] If a merchant enters a `font_family` value that is not one of a known-safe list of system fonts (system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, sans-serif, serif, Georgia, Times New Roman, monospace), a non-blocking warning reads "We cannot verify this font is loaded on your storefront — if it is not, shoppers will see a fallback font"; the merchant can still save.
- [ ] `heading_font_family` left blank falls back to `font_family` at render time; the control shows placeholder text "Uses body font" to make the fallback visible.
- [ ] Save persists all token overrides to the tenant's widget config; changes propagate to live widgets within 5 minutes (same cache behaviour as Widget → Configuration).
- [ ] The Theming screen is keyboard-navigable end-to-end (tab order, focus rings inside the live preview, colour picker reachable without a mouse) and screen-reader-labels every control (colour pickers, font pickers, preset dropdowns, per-token Reset buttons, the two top-level preset buttons).

---

## Feature: Merchant admin — Placements

### User Story
As a merchant admin, I want to see which placements are live and manage them so that I can experiment with where the widget appears.

### Acceptance Criteria
- [ ] The admin dashboard's Placements screen lists each configured placement (Home hero / PDP slot / Gift-finder page) with status (enabled / disabled), the theme location, and links to the Shopify theme editor for that block.
- [ ] The merchant can enable or disable a placement; disabled placements do not render the widget on the storefront.
- [ ] Each placement carries an independent set of KPIs on the analytics dashboard.
- [ ] Adding a new placement follows the same "Configure in Shopify" deep-link flow used in onboarding.

---

## Feature: Merchant admin — Catalog sync controls

### User Story
As a merchant admin, I want visibility into how my catalog is syncing and the ability to trigger a resync when I need one so that I trust the widget is recommending what is actually in my store.

### Acceptance Criteria
- [ ] A **Catalog sync** panel in the admin dashboard shows: total product count in WiseGift, timestamp of last full sync, timestamp of last webhook received per topic (`products/update`, `inventory_levels/update`, etc.), and a webhook-health indicator (OK / degraded / not receiving) derived from time-since-last-webhook thresholds.
- [ ] A **Refresh prices & stock** button re-pulls variant price and availability from Shopify Admin API without recomputing embeddings. Runs asynchronously with a progress indicator. Rate-limited to one run per tenant per hour.
- [ ] A **Full resync** button re-fetches the full catalog and re-embeds only products whose `content_hash` changed (title / description / tags). Runs asynchronously with a progress indicator. Rate-limited to one run per tenant per 24 hours, with a "Contact support to run more often" affordance.
- [ ] Both buttons disable while a sync is in progress and re-enable on completion; failures show a short error and a "Retry" affordance.
- [ ] Manual syncs are counted against the same Shopify per-app rate-limit budget as automatic syncs — a large-catalog manual run cannot starve other tenants (per `data.md` §3 rate-limit posture).
- [ ] Per-tenant sync cadence (nightly reconciliation frequency) is NOT configurable at MVP; it is fixed for all tenants. Configurable cadence is post-MVP.
- [ ] Per-product force-refresh is NOT exposed in the merchant UI at MVP; it is available support-only via a backend admin path.

---

## Feature: Merchant admin — Analytics dashboard (minimal MVP)

### User Story
As a merchant admin, I want to see the widget's impact on my store in one screen so that I know whether it's worth keeping.

### Acceptance Criteria
> Measurement definitions (populations, attribution join, statistics, API contract) live in `docs/engineering/m1-2-lift-measurement.md`. PO decisions D1–D9 are in `decisions.md` 2026-10-14.

- [ ] Default date range: last 30 days. Configurable ranges: 7d, 14d, 30d, 90d. Ranges and daily trend points use **UTC day boundaries**, and the UI labels them "UTC" (M1·2 D9).
- [ ] **Population** (M1·2 D2): lift KPIs default to **triggered sessions**, meaning sessions that submitted the intent form. A secondary **store-level view** (all sessions that saw the widget, `shown`) is available through a population switch on the same screen. The active population is named in the screen header and in the CSV.
- [ ] **Lift tiles** (three), each shown as `exposed vs holdout` with lift percentage and a significance flag. **Green** = significant positive (p < 0.05). **Red** = significant negative (M1·2 D7). **Grey** = not significant, or not enough data yet:
  - **Conversion rate**: sessions with ≥ 1 order inside the 7-day attribution window / sessions in the active population.
  - **AOV** for attributed orders, exposed vs holdout.
  - **Revenue per session**: attributed order revenue / sessions in the active population.
- [ ] **Data checks (A/A integrity, no lift claim)** (`decisions.md` 2026-10-15 "M1·2 P1–P3", P2; 2026-10-16 C5). The hook is identical in both arms, so these rates should match between exposed and holdout. A difference means the measurement is broken, not that WiseGift works:
  - **Widget CTR**: engagement at or before `t₀`, in `shown` sessions. It is **not** a lift tile.
  - **Trigger rate** (formerly `intent_submit_rate`).
  - **Sample-ratio check (SRM)** on `shown` and on the active population.
  - They are shown in a "Data checks" panel with a pass / failed status per check, never with green/red lift colouring or a lift percentage.
  - If any check fails (p < 0.01), every lift tile turns grey and the panel explains: "A data check failed, so lift results are hidden until we've looked into it." Definitions: `m1-2-lift-measurement.md` §2.1.
  - *DRAFT — pending PO review (2026-10-16):* the panel layout and copy are new. The P2 rule itself is decided.
- [ ] **Revenue** is Shopify `total_price`. That is gross: shipping and tax included, before refunds. Tiles and the CSV label it "Gross order value (incl. shipping & tax, before refunds)" (M1·2 D8).
- [ ] **Recommendation diagnostics (exposed only, no lift claim)** replace the former add-to-cart tile (M1·2 D1). They are shown as absolute rates for the exposed arm, with no holdout column, no lift percentage and no significance colour:
  - **Rec click-through**: triggered exposed sessions with ≥ 1 `product_clicked` after the trigger.
  - **Rec purchase rate**: triggered exposed sessions with ≥ 1 attributed order containing a product the session clicked.
  - Add-to-cart rate is **not measured at MVP**. There is no `carts/update` ingestion, and holdout has no recommended products to compare against.
- [ ] A small line chart per KPI shows daily trend over the selected range.
- [ ] A minimum-data callout displays if any KPI has fewer than 5,000 exposed sessions (in the active population) in the selected range ("Not enough data yet — need N more exposed sessions"). Each lift tile also shows **"readout at N sessions"** and the **minimum detectable lift at the current sample size** ("a lift smaller than X% would not be detectable yet"), so pilots know what the data can and cannot show (M1·2 D4).
- [ ] The live dashboard is not the source of case-study claims. Those come only from the pre-registered readout run by `recommendations-specialist` (M1·2 D4, spec §5.5).
- [ ] Data is refreshed hourly; a timestamp shows last refresh.
- [ ] All numbers are export-to-CSV.
- [ ] No cohort analysis, funnel drill-down, or per-recommendation breakdown at MVP — those are post-MVP.

---

## Feature: Merchant admin — Account & billing

### User Story
As a merchant admin, I want to see my plan and usage so that I know when I'm approaching my limits.

### Acceptance Criteria
- [ ] The Account screen shows: plan name (Pilot / Starter / Growth / Scale), monthly recommendation budget included in the plan, current-month recommendations served, days until reset.
- [ ] A visual progress bar shows current usage vs. plan cap. Colour changes to warning at 80% and to error at 100%.
- [ ] Billing is handled via Shopify's built-in App Store billing API for public-app installs (recurring subscription with usage-based overage). Pilot tenants show "Pilot — free through YYYY-MM-DD" with no billing hooks.
- [ ] If the merchant hits their monthly cap, the widget serves precomputed cold recs only (no live LLM) and the admin dashboard shows a persistent banner with an upgrade CTA.
- [ ] Merchant can cancel or downgrade from within Shopify (App Store billing is authoritative).

> **DRAFT, pending PO review (2026-10-16):** the Account screen gains a **Contact email** card (view, change with re-confirmation, no removal). See "Merchant onboarding — Onboarding wizard" → "Contact email step and go-live gate" → "Editing later".

---

## Feature: Recommendation API — `POST /widget/v1/recommendations`

### User Story
As the widget, I want to fetch personalised recommendations for the current shopper's intent so that I can display relevant products.

### Acceptance Criteria
> Exact wire shapes (camelCase) are in `architecture.md` §9.2. The field names below are logical names.

- [ ] Endpoint: `POST /widget/v1/recommendations`. Auth: the short-lived (1 h) widget JWT issued by `POST /widget/v1/session`. That bootstrap call resolves the tenant from the storefront's `Origin` against the merchant's shop domain or custom domain; there is no public key. The request's session ID must match the token's, otherwise the call is rejected (403).
- [ ] Request body: session ID, `placement`, intent mode, `intent` (relationship, occasion, budget_min, budget_max, interests[]), `context` (current `platform_product_id` if on a PDP), `limit` (default 5, max 10).
- [ ] Response body: `recommendations[]` (each: platform product ID, `slug`, `title`, `price`, image URL, rank score), cache hit (boolean), served-from (`live_llm` | `cache` | `precomputed` | `fallback`).
- [ ] The endpoint enforces the **per-tenant hard usage cap** — over-cap requests are served from precomputed cold recs, cache hit is set to true, `served_from = precomputed`.
- [ ] The endpoint enforces the **per-session rate limit** (max 20 live-LLM calls per session per hour); over-limit requests are served from cache/precomputed.
- [ ] p95 latency budget: **300 ms** end-to-end.
- [ ] Every response emits cost telemetry (tenant, placement, model, tokens in/out, cache hit/miss, latency) to the ops metrics stream.
- [ ] Errors return a fallback payload with `served_from = fallback` and 200 OK — the widget must never render an error state to the shopper.

---

## Feature: Event API — `POST /widget/v1/events`

### User Story
As the widget, I want to record shopper interactions so that attribution and analytics are accurate.

### Acceptance Criteria
> Exact wire shape (camelCase; for example the product ID travels as `shopifyProductId` and the holdout flag as `holdout`) is in `architecture.md` §9.2.

- [ ] Endpoint: `POST /widget/v1/events`. Same auth as the recommendation API. Each event's session ID must match the token's (otherwise 403).
- [ ] Accepts a batch of 1–10 events (`events[]`), each with: event type (`widget_shown` | `widget_engaged` | `intent_submitted` | `product_clicked`), session ID, `placement`, intent mode, holdout flag, `timestamp`, optional platform product ID, optional rank score and served-from (echoed from the rec response), and optional intent-form schema version.
- [ ] The client holdout flag is never trusted. The server uses `sessions.is_holdout`.
- [ ] Success returns **202 Accepted** with no body.
- [ ] Events are validated against a strict schema; unknown fields are rejected (fail fast on client bugs).
- [ ] Events are enqueued and processed asynchronously into the analytics store. p95 write latency < 100 ms.
- [ ] The endpoint is fire-and-forget from the widget's perspective — no retry storms if a request fails.
- [ ] Batching: widget can send up to 10 events per request; widget batches events with a 1-second flush interval.

---

## Feature: Webhook receiver — Shopify order attribution

### User Story
As the WiseGift backend, I want to record order events to attribute revenue lift so that merchant dashboards show accurate KPIs.

### Acceptance Criteria
- [ ] HMAC signature verification against Shopify's shared secret; invalid signatures rejected with 401.
- [ ] Idempotency: repeated webhooks for the same `order_id` are deduplicated.
- [ ] From the order payload we retain only: `order_id`, `line_items[].platform_product_id`, `line_items[].quantity`, `line_items[].price`, `total_price`, `currency`, `created_at`, `tenant_id` (derived from shop domain).
- [ ] All customer PII fields (`customer`, `billing_address`, `shipping_address`, `email`, `phone`, `client_details`) are dropped at ingest and never persisted, logged, or forwarded.
- [ ] Correlation to a widget session is read in this order: the `_wg_session` cart attribute (`note_attributes`), then the legacy `wg_session` cart attribute, then the `wg_session` URL marker (`landing_site` / `referring_site`). Values that are not strict v4 UUIDs are ignored (`decisions.md` 2026-10-14, R1/R2). Unattributed orders are still stored (to compute merchant-wide baselines) but do not contribute to per-session lift.
- [ ] Webhook processing p95 < 500 ms.

---

## Feature: Multi-tenancy — Isolation model

### User Story
As the platform operator, I want strict tenant isolation so that no cross-tenant data leak is possible even in the presence of a bug in application code.

### Acceptance Criteria
- [ ] Every table containing tenant data carries a `tenant_id UUID NOT NULL` column; a composite index `(tenant_id, primary_key)` is present.
- [ ] Every repository method takes a `tenant_id` argument or reads it from a request-scoped context; a lint rule flags any raw SQL that does not include `WHERE tenant_id = ?`.
- [ ] Integration tests seed two tenants, run queries as tenant A, and assert zero rows from tenant B are returned. Rerun in CI on every PR.
- [ ] Every recommendation call and event write is tenant-scoped from the API layer down through embeddings retrieval to the analytics store.
- [ ] Uninstall soft-deletes for 30 days, then a scheduled job hard-purges all tenant-scoped rows (products, embeddings, events, orders, config).

---

## Feature: Cost guardrails (engineering requirements)

### User Story
As the founder, I want the AI cost of running the product to be bounded regardless of merchant traffic so that a pilot can never generate a bill I cannot pay.

### Acceptance Criteria
- [ ] **Per-tenant hard usage cap** — configurable, default 50k live-LLM recommendation calls per calendar month during pilot. Over-cap requests are served from precomputed cold recs. Enforced at the recommendation API layer with a Redis-backed counter.
- [ ] **Response cache** — 24-hour TTL on live recommendations, keyed by `(tenant_id, intent_signature, context_signature)`. Expected hit rate > 60% at steady state.
- [ ] **Nightly precomputed cold recs** — a per-tenant nightly job produces top-N recommendations per SKU without LLM calls (vector retrieval + heuristic ranking only). These serve as PDP-context recommendations when the shopper provides no intent signal, and as the fallback when the live path is unavailable or over-cap.
- [ ] **Model selection** — OpenAI GPT-4o-mini is the default for re-ranking and intent parsing. Escalation to GPT-4o is behind a per-tenant feature flag (`feature_flags.escalate_ranker`) and used only for full-form gift-intent flows with a rich intent object. (Vendor-neutral `RankingClient` port — one adapter class to swap. Was Claude Haiku 4.5 / Sonnet; swapped to reuse the existing OpenAI billing + drop a sub-processor. See `docs/decisions.md` 2026-10-06 "PR 7d".)
- [ ] **Session-level rate limit** — max 20 live-LLM calls per widget session per hour.
- [ ] **Per-tenant kill switch** — daily-spend threshold triggers an ops alert and flips the tenant to precomputed-only mode until manually reset. Configurable per plan tier.
- [ ] **Cost telemetry** — every rec call is tagged with `tenant_id`, `placement`, `model`, `input_tokens`, `output_tokens`, `cache_hit`, `served_from`, `latency_ms`. Dashboard shows cost per merchant per day.

---

## Data & privacy

### What we store
- **Per tenant:** identity (shop domain, Shopify shop ID, region, plan), catalog snapshot with embeddings, widget config, placement config, feature flags.
  - *DRAFT, pending PO review (2026-10-16):* plus the merchant-confirmed **contact email** and its confirmation timestamp. WiseGift is the controller of that address; it is not shopper data. The Shopify shop email is read live as a suggestion and never stored. See "Contact email step and go-live gate".
- **Per session:** anonymous session ID, exposed/holdout assignment, events (widget_shown, widget_engaged, intent_submitted, product_clicked), intent submissions (recipient relationship, occasion, budget, interests — no free-text PII).
- **Per order:** filtered order payload — `order_id`, line items (product IDs, quantities, prices), total, currency, order `created_at`, receipt time, tenant, plus the widget `session_id` when the order carries a marker. No customer identifiers. The `session_id` is pseudonymous, and through the `_wg_session` cart attribute it can be joined to the merchant's identified customer (`decisions.md` 2026-10-14). That is why it is nulled at 120 days (below).

### Retention (`decisions.md` 2026-10-14, M1·2 D5 and "Order data … 18-month retention")
- [ ] Raw widget `events` are kept **120 days**. `sessions` rows are kept **120 days** after the session was last seen.
- [ ] `orders_attributed.session_id` is **nulled at 120 days**. This breaks the customer ↔ widget-history link while keeping the order for baselines.
- [ ] Order rows (attributed and unattributed) are kept **18 months** from the order's `created_at`, then hard-deleted.
- [ ] The anonymous daily aggregate `analytics_kpi_daily` (no session IDs) is kept **18 months**. It serves dashboard trend lines older than the raw-event window.
- [ ] Enforcement is the **M3·1 scheduled purge job, which is not built yet**. Until it ships, nothing is purged. Tenant uninstall still overrides all of the above: everything for the tenant is hard-purged at day 30, and that job is also M3·1.

### What we never store
- Merchant customer names, emails, phone numbers, addresses, IPs.
- Full order payloads with PII (filtered at ingest, before persistence).
- Cross-merchant browsing behaviour or profiles.
- Anything gathered from a source other than the widget interaction, the merchant's Shopify APIs (catalog + orders), or the merchant admin's own inputs.

### Region
All storage and compute in **EU regions** at MVP (Neon EU, application hosting in EU). US region considered only when a US pilot is signed.

**Known exception, being closed:** OpenAI embedding and ranking calls (shopper intent incl. free-text interests, plus candidate products) currently go to `api.openai.com`, which is US processing. Moving to an OpenAI EU data-residency project is decided but not yet switched. It waits on PO preconditions on the OpenAI account (`decisions.md` 2026-10-14 "LLM processing moves to an OpenAI EU data-residency project"; `architecture.md` §1).

### Data-processing role
WiseGift is a **data processor** for the merchant's shopper data (pseudonymous session events + filtered order records). A **DPA is required** for every merchant, including pilots. Drafts are owned by `security-and-privacy` and are informational only, pending PO and attorney review: `docs/legal/dpa-template.md`, `docs/legal/consent-position.md`, `docs/legal/sub-processors.md`.

---

## Roadmap — post-MVP (in likely priority order)

1. **Verticalised intent parameter packs** — per-vertical intent forms (Fashion / Tech / Books / Beauty / Home) with merchant-editable style archetypes and reference imagery. Justifies the Growth-tier price step. See `docs/decisions.md` 2026-09-29 entry. *Precursor (DRAFT, pending PO review):* niche packs as starter presets over existing configuration only, in [`hooks-and-packs.md`](hooks-and-packs.md) §3; the configurable questionnaire and looks stay here (§6 Deferred there).
2. **Purchase-outcome learning loop** — the event schema is MVP-ready; the learner (contextual bandit / preference model) lands here to compound per-tenant lift over time.
3. **Analytics — deeper cuts** — per-placement funnel, per-intent-mode conversion, cohort analysis, exportable reports.
4. **VTEX and SFCC integrations** — same core APIs, new adapter modules for catalog sync and order webhooks.
5. **Custom / headless integration** — publish a REST integration guide and a signed-request SDK for merchants on non-supported platforms.
6. **Admin — A/B experiments UI** — let merchants A/B test alternate hook copy, form field sets, and rec-slot sizes with attribution baked in.
7. **Behavioural signals (opt-in)** — if a merchant opts in and has consent-banner infrastructure, ingest cross-page browsing signals for stronger cold-start recs. Not before a legal review.
8. **B2C storefront revival (parked Flutter app)** — resurrect the Flutter consumer app to browse participating merchants' catalogs on a WiseGift-branded surface. Requires per-merchant commercial deal. Not before at least 20 paying merchants.

---

## Out of MVP scope (explicit)

- iOS / Android native mobile apps for merchants.
- Multi-store roll-ups for merchants running several Shopify stores under one brand.
- Free forever tier or self-serve trial without a pilot conversation.
- Rev-share pricing model.
- SSO / SAML for merchant admin login (email + password + Google OAuth are enough at MVP).
- Multi-user seats per merchant tenant (single-user admin at MVP).
- Public API for merchants to hit recommendation endpoints from server-side code (widget-only at MVP).
- Any behavioural tracking beyond widget interactions and the single order-confirmation webhook.
- Shopify `carts/update` ingestion and an add-to-cart KPI (`decisions.md` 2026-10-14, M1·2 D1).

---

## Open questions

*(No MVP-scope questions currently open. New items land here; resolved items move to the section below.)*

---

## Resolved questions

- **Language coverage** (2026-09-29): MVP ships widget copy in EN, ES, PT. No DE / FR / IT preemptively. Rationale: pilot pool is EU merchants (ES/PT initially, UK/DE eventually); EN covers the UK case; DE / FR / IT translations are cheap to add per-pilot on demand (one signed merchant = one translation pass), not preemptively. See `docs/decisions.md` 2026-09-29 "Language coverage".
- **Custom-app pilot billing** (2026-09-29): Public App Store billing throughout, with a Stripe safety net if App Store review is not approved by the first pilot's day-90 conversion. Rationale: single billing system through the lifecycle is cleaner than dual-track; Stripe engineering is deferred until we actually need it (i.e. review runs long OR a specific pilot converts before public app is live). Submission to App Store review is planned for day 60 of the first pilot so approval lands before day 90. See `docs/decisions.md` 2026-09-29 "Custom-app pilot billing".
- **Merchant data export SLA** (2026-09-29): Support-only export at MVP — merchant emails support, we run a manual query and hand back a signed URL to a JSONL file within 5 business days. Self-serve export becomes an Enterprise-tier feature post-MVP once a real merchant asks. Rationale: mid-market pilots don't demand this; Enterprise negotiates it in the contract; premature to build self-serve before any demand signal. See `docs/decisions.md` 2026-09-29 "Merchant data export SLA".
- **Widget consent for EU** (2026-10-14, supersedes 2026-09-29). **Current position:**
  - Consent gating through the Shopify Customer Privacy API (`analyticsProcessingAllowed()`) is a **hard gate before any pilot storefront**.
  - Each tenant carries `consentModeRequired`, **default `true`** (fail closed). Without consent there is no session ID, no storage, no cart attribute and no events, and recs are served in degraded mode.
  - `consentModeRequired = false` (legitimate-interest mode, measure without a consent signal) is **PENDING PO + attorney confirmation**.
  - Why it changed: the `_wg_session` cart attribute (attribution gap G1) creates a join key between the merchant's identified customer and the widget history, which the 2026-09-29 reasoning did not cover.
  - Detail: Widget — Attribution & session tracking; `decisions.md` 2026-10-14 "`_wg_session` order join … G5"; drafts in `docs/legal/consent-position.md`.

  *History:* on 2026-09-29 the decision was "no Consent Mode integration at MVP; ship the escape hatch", with `consent_mode_required` defaulting to `false`. It relied on the CNIL-line "strictly-necessary measurement" reading of ePrivacy Art. 5(3), with GDPR Art. 6(1)(f) as belt-and-braces (`decisions.md` 2026-09-29 "Widget consent for EU").
- **App Store category** (2026-09-29): Primary category **Marketing → Upselling & Cross-selling**; secondary (if App Store rules permit) **Store Design → Product Discovery**. Rationale: mid-market ICP searches Marketing → Upselling for revenue-per-session lift; Product Discovery is the second organic entry for the "gift finder" mental model. Positioning inside the listing leads with "AI recommendations that also do gifts" so we don't cap addressable use to gifting seasons. Listing artefacts at `docs/growth/app-store-listing.md`. See `docs/decisions.md` 2026-09-29 "App Store category".

*The seven pivot-era scope decisions are captured in `docs/decisions.md` entry dated 2026-09-29 "Frozen MVP scope".*
