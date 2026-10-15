# Embedded merchant admin: auth model proposal

> **ACCEPTED by PO 2026-10-15 (Option A; see decisions.md). Shopify facts S1–S6 still to be verified before build.** Prepared by `security-and-privacy`
> on 2026-10-15. This is a recommendation, not a decision. It closes the
> "PENDING decision — embedded-admin auth model" item in `decisions.md`
> (2026-10-14, "Admin ↔ backend transport") once the PO decides.
>
> **Not legal advice.** The privacy notes below are informational. Route
> the questions marked **ATTORNEY** to a qualified attorney.
>
> **Evidence caveat.** The backend code was read from the local
> `wisegift-backend` working tree, not from `origin/develop`. No shell
> was available to check out the remote branch. The working tree looks
> older than `develop`: the OAuth callback still redirects to the
> backend-relative `/admin/signup` (pre-#73), webhook registration still
> uses a single try/catch (pre-#64), and `AdminSecurityConfig` has no 401
> entry point. The local `wisegift-widget/shopify.app.toml` also lacks
> `read_inventory`, which widget PR #16 added. Every code-level statement
> below must be re-checked against `origin/develop` by `backend-engineer`
> (open question B1). The recommendation does not depend on those details.

---

## 1. Summary in plain language

Inside the Shopify admin, our dashboard runs in an iframe on
`admin.shopify.com`. To the browser, that iframe is a different site.
Our login uses `SameSite=Lax` cookies, which the browser neither sets nor
sends in that iframe. So inside Shopify, signup and login cannot work,
and no browser setting we control fixes this reliably.

Shopify's answer for embedded apps is **session tokens**. App Bridge,
the Shopify script we already load, gives the dashboard a short-lived
signed token (about 1 minute) that says "this request comes from a staff
member of shop X". The dashboard sends the token with every API call.
The backend checks the signature with our Shopify app secret and finds
the tenant from the shop in the token.

**Recommendation: Option A, session tokens only.** There are no WiseGift
passwords, no Google sign-in, no invite link and no signup page. A
merchant who can open the app in their Shopify admin is signed in. The
install lands directly on the onboarding wizard. We would also stop
storing merchant staff emails and password hashes. The standalone admin
(outside Shopify) is not needed for the Shopify-only MVP; we revisit it
when VTEX/SFCC arrives.

---

## 2. Where we are today

| Piece | Current state | Effect inside the Shopify iframe |
|---|---|---|
| Admin identity | `admin_users` (email + BCrypt hash, or Google `sub`), one per tenant | Requires a separate WiseGift account |
| Session | Spring Session Redis cookie `WG-ADMIN-SESSION`, `HttpOnly`, `Secure`, `SameSite=Lax`; 24 h idle, 7 d absolute (`SessionConfig`) | Not set or sent in a cross-site frame, so every call is 401 |
| CSRF | `CookieCsrfTokenRepository` double-submit `XSRF-TOKEN` (`AdminSecurityConfig`); `client.ts` echoes it as `X-XSRF-TOKEN` | The cookie is never set in the frame, so every POST fails CSRF |
| Transport | Next.js `rewrites()` proxies `/admin/v1/*` to the backend (`next.config.ts`), so cookies are first-party to the admin host | Same-origin within the frame, but the frame itself is cross-site to `admin.shopify.com` |
| Install | `application_url` = backend `/oauth/shopify/install` (`shopify.app.toml`, `embedded = true`, `use_legacy_install_flow = false`) → authorization-code grant → callback mints a 24 h single-use invite JWS → 302 to `{admin}/signup?token=` | Every app open in Shopify admin re-runs the install handshake. This matches the "install flow ran 3× concurrently" finding (2026-10-14) |
| App Bridge | CDN `app-bridge.js` + `shopify-api-key` meta tag (`app-bridge-provider.tsx`); being fixed in admin PR #6 | Required for session tokens anyway |
| Framing | CSP `frame-ancestors https://admin.shopify.com https://*.myshopify.com` | Allows framing; see §6.3 for the per-shop form |

The fresh-install check in `runbooks/admin-staging-deploy.md` step 7 asks
whether signup opens top-level or in the frame. From the code path, it
will almost certainly open **in the frame**. `application_url` is loaded
in the iframe. With Shopify-managed install the scopes are already
granted, so the authorize step should pass straight through to our
callback. The callback's 302 then stays in the frame. Expected result:
the signup form renders, then the POST fails with a CSRF or 401 error.

---

## 3. Options compared

### 3.1 The options

- **A. Shopify session tokens only.** App Bridge `shopify.idToken()` is
  sent as `Authorization: Bearer` on every `/admin/v1/*` call. The
  backend verifies the JWT and derives the tenant from the `dest` shop.
  The admin is stateless: no cookies, no Spring Session, no CSRF token.
  Install uses Shopify Token Exchange (§5.2) instead of the redirect and
  invite.
- **B. Keep cookies, make them iframe-compatible.** Set `WG-ADMIN-SESSION`
  and `XSRF-TOKEN` to `SameSite=None; Secure; Partitioned` (CHIPS). Keep
  email/password, Google, invite and signup.
- **C. Hybrid.** Session tokens when embedded. The current cookie login
  stays for a standalone admin (outside Shopify).

### 3.2 Evaluation

| Criterion | A. Session tokens only | B. Iframe-compatible cookies | C. Hybrid |
|---|---|---|---|
| **Works in the iframe** | Yes, in every browser. No cookies involved | **Unreliable.** Chrome: works today (third-party cookies still on by default; CHIPS supported), **fails in Incognito** and when the user blocks third-party cookies. Firefox: Total Cookie Protection partitions per top-level site, so it works in standard mode; strict mode or settings may block. **Safari: blocks third-party cookies by default (ITP).** CHIPS support has been inconsistent, so test before relying on it. The Storage Access API needs a user gesture and a prompt | Embedded: as A. Standalone: as today |
| **CSRF** | Not applicable. The browser never attaches an `Authorization` header by itself, so a forged cross-site request carries no credential. CSRF protection can be removed **only because** the chain accepts no cookie credential | **Worse than today.** `SameSite=None` means the browser attaches the session cookie to requests initiated by **any** site. CSRF defence then rests entirely on the double-submit token plus Origin checks. Login CSRF must also be handled | Two chains with different CSRF rules on the same `/admin/v1` surface. A request carrying both a cookie and a bearer must be resolved one way only. Easy to get wrong |
| **Token replay** | The bearer is valid for about 60 s (plus clock leeway). App Bridge reuses one token for several calls, so single-use `jti` enforcement would break it; rely on the short expiry instead. Never log the header. HTTPS only | Session cookie valid for up to 24 h idle / 7 d absolute. Theft via XSS is blocked by `HttpOnly`, but replay works for longer | Mixed |
| **Tenant binding** | From the verified `dest` claim only (`*.myshopify.com` host) → `platform_credentials` → `tenant_id`. Never from `?shop=`, body or path | From the `admin_users.tenant_id` row via the session principal. Sound | Two derivations to keep consistent |
| **Session fixation** | Not applicable (no server session) | Present today: login and signup do not call `changeSessionId()` (in the pending hardening list). Must be fixed | Must be fixed on the cookie path |
| **Logout / revocation** | No WiseGift logout; the Shopify admin session is the session. Staff removed or app permission withdrawn in Shopify: no new tokens within about 1 min. Uninstall: the backend rejects tokens for soft-deleted tenants (§5.1) | Server-side invalidate (works). A user removed from the store keeps WiseGift access until we delete the row: **we never learn about Shopify staff changes** | Both paths |
| **Multi-tenant isolation** | Strong. The tenant comes from a claim signed with our app secret; the client cannot choose it | Strong once logged in, but the account's link to the shop is set once at signup and never re-checked against Shopify | Strong per path; more surface |
| **Privacy (merchant staff data)** | We persist **nothing** about staff if we skip `sub` storage (§6.4). In memory: Shopify user ID (`sub`), pseudonymous. No password hashes, no emails, no Google as an extra identity party. **Removes** the `admin_users` PII | Email, BCrypt hash, optional Google `sub`, `created_at`. Password-reset flows needed later | Same as B for standalone users |
| **Shopify App Store / Built for Shopify** | Matches Shopify's documented pattern for embedded apps (session tokens, no reliance on third-party cookies, merchant lands in the app UI right after install). **Verify current wording** (§8, S5) | **Likely review risk.** Shopify guidance says embedded apps must not rely on third-party cookies, and a separate email/password signup inside the admin is poor UX that reviewers flag. Treat as likely rejection for Built for Shopify | Acceptable if the embedded path is A, but the second login is dead weight for Shopify merchants |
| **Merchant UX** | Install → approve scopes → onboarding wizard, in the Shopify admin. No signup, no password | Install → "Create your WiseGift account" form inside the Shopify admin → and Safari users are stuck | As A inside Shopify |
| **Engineering effort** | Backend ~5–7 dev-days; admin ~2–3; widget repo TOML ~0.5; devops ~0.5–1 (§7) | Smaller code change (~1–2 days), but cross-browser testing, CHIPS support in Spring Session's cookie serializer to verify, plus fixation and CSRF hardening. Still broken in Safari | A **plus** keeping and hardening everything in B's cookie path |
| **Impact on built work** | Retires invite-signup (#73), password/Google login, CSRF config, Spring Session for admin. Reshapes `AdminPrincipal` | Keeps everything | Keeps everything and adds A |

### 3.3 Answers to the brief's questions

**Option A**

- *Do we still need WiseGift admin users and passwords?* **No.** "Logged
  into Shopify with access to this store's app" becomes the identity. The
  verified token proves shop membership. Shopify staff permissions
  control which staff (and collaborators) can open the app. MVP is
  single-user per tenant anyway (spec "Out of MVP scope"), so
  "everyone the merchant lets open the app" is consistent with it.
- *Invite token, signup, Google SSO, standalone admin?* All four go.
  - The invite exists only to bridge the OAuth callback to a WiseGift
    account; with no account, there is nothing to bridge.
  - Google SSO goes with the password login.
  - Standalone admin: not needed while every tenant is a Shopify shop.
    Outside the Shopify admin, the dashboard shows "Open WiseGift from
    your Shopify admin" (§5.3). When VTEX/SFCC arrives, design that login
    then, probably on a managed EU identity provider rather than
    reviving our own passwords (open question P5).
- *Is CSRF still needed?* **Not on a chain that accepts only bearer
  tokens and never reads a cookie.** If any cookie credential stays on
  the same chain (Option C, or the Phase 1 transition), CSRF must stay
  for those requests.
- *Session lifetime and logout?* Token lifetime is about 1 minute; App
  Bridge refreshes it transparently. There is no WiseGift session and no
  logout button. Leaving the Shopify admin is the logout.
- *Does Token Exchange matter?*
  - **Yes, for install, not for per-request auth.** With
    `use_legacy_install_flow = false`, Shopify installs the app and grants
    scopes before our app loads. The documented way to get our **offline**
    access token (used for catalog sync and webhooks) is to exchange a
    session token for it, without redirecting. That lets the backend
    provision the tenant on the first authenticated admin call. It
    removes the redirect chain, the invite, and the repeated-install
    problem.
  - **Online tokens are not needed.** We never call Shopify on behalf of
    a specific staff member. Online tokens also return the staff
    member's name and email (`associated_user`), which we would then
    hold.

**Option B**

- *Safari/ITP and Firefox:* see the table. Safari is the deciding
  failure. It blocks third-party cookies in frames by default, and many
  merchants run the Shopify admin in Safari on Mac and iPad.
- *CSRF implications:* `SameSite=None` removes the browser's built-in
  cross-site protection, so the double-submit token becomes the only
  layer. It also needs login-CSRF protection and fixation fixes.
- *Shopify review:* likely problematic (see the table). Verify (S5).

---

## 4. Recommendation

**Adopt Option A: Shopify session tokens only, with Token Exchange
install. No standalone admin at MVP.**

Why:
1. **It is the only option that works in every browser** the merchant
   might use inside the Shopify admin. B fails in Safari by default and
   in Chrome Incognito.
2. **It is the stronger security posture.**
   - The tenant comes from a claim signed with our app secret.
   - There is no CSRF surface, no session fixation, and no long-lived
     credential to steal.
   - Shopify staff changes take effect within about a minute without
     any action from us.
3. **It is the smaller privacy footprint.** We stop collecting staff
   emails and password hashes. Google drops out as an identity party.
4. **It removes the separate signup**, which is the main merchant
   friction and a likely App Store review problem.
5. **It fixes the repeated-install behaviour.** `application_url` moves
   to the admin, so opening the app no longer re-runs OAuth.
6. **The pending pre-pilot hardening items become unnecessary** under A:
   CSRF cookie flags, `changeSessionId()`, and anonymous `/me` creating
   Redis sessions. Don't spend effort on them unless the PO picks B or C.

Rejected:
- **B**, as above.
- **C**: two authentication models on one API surface, to serve a
  standalone user who does not exist at MVP. That is extra attack
  surface and test burden with no current user.

The cost of A is real but bounded (§7). It sits on the pilot's critical
path, because the current model cannot work inside the Shopify admin.

---

## 5. Design of Option A

### 5.1 Backend: verifying the session token

Verification lives **only in the backend**. The client secret must
never reach the Next.js app; it holds only the public `client_id`.

A new `ShopifySessionTokenVerifier` goes in `platform-integrations`
(`infrastructure/security`, next to `ShopifyHmacVerifier`). It
**fails closed**: any failure is a 401 with a reason code in the logs,
and the token itself is never logged.

1. **Format and algorithm.**
   - Three base64url parts.
   - Header `alg` must equal `HS256`. Reject `none` and any other value.
     **Never** pick the algorithm from the token.
2. **Signature.**
   - HMAC-SHA256 over `header.payload` with the app client secret,
     compared in constant time (`MessageDigest.isEqual`).
   - During a Shopify secret rotation, accept the current or the previous
     secret (two configured keys, previous one optional).
3. **Parse claims** with a real JSON parser (Jackson). This is
   third-party input. **Do not reuse** `InviteTokenService`'s `indexOf`
   parser: it is acceptable only for our own token format.
4. **Time.**
   - `exp > now − leeway` and `nbf ≤ now + leeway`.
   - Leeway of about 5–10 s (S2).
   - Sanity check: `exp − iat` ≤ a few minutes. Anything longer is not a
     Shopify session token.
5. **Audience.** `aud` equals our `client_id` exactly.
6. **Shop.**
   - `dest` must be `https://<host>`, with `<host>` matching the existing
     `SHOP_DOMAIN` regex (`^[a-z0-9][a-z0-9-]*\.myshopify\.com$`).
   - `iss` must equal `https://<same host>/admin`.
7. **Subject.** `sub` is present (the Shopify staff user ID). It is used
   only in memory (§6.4).
8. **Tenant.**
   - Look up `platform_credentials` by `(shopify, host)`, giving
     `tenant_id`.
   - Tenant soft-deleted, or no credential yet: go to the install path
     (§5.2). Never serve tenant data.
   - `tenants.active = false` is **not** a rejection, because the
     onboarding wizard runs before go-live.

Wiring in `app/`, following the `WidgetAuthFilter` pattern:

- A dedicated `SecurityFilterChain` for `/admin/v1/**`:
  - `SessionCreationPolicy.STATELESS`;
  - CSRF disabled (justified only because this chain reads no cookie);
  - `HttpStatusEntryPoint(401)`;
  - a `ShopifySessionTokenFilter` that populates the security context.
- **Principal.** Reshape `AdminPrincipal` to
  `(TenantId tenantId, String shopDomain, String shopifyUserId)`.
  - Drop `AdminUserId` and `email`.
  - It no longer needs to be `Serializable`, because no session store
    holds it.
  - Admin controllers keep taking the tenant **only** from the principal
    (architecture.md §9.3). That invariant is unchanged.
- **Expired-token retry.** On a 401 caused by an expired or invalid
  token, add the header Shopify's App Bridge uses to retry with a fresh
  token (`X-Shopify-Retry-Invalid-Session-Request: 1`; verify, S3). This
  matters while Render cold starts can exceed the token lifetime.
- **Tests** (test-first; two tenants per the backend's testing rule):
  - Every negative claim case: bad signature, `alg` swap, wrong `aud`,
    expired, `nbf` in the future, `dest`/`iss` mismatch, non-myshopify
    host, missing `sub`.
  - A token for shop A with `?shop=B` in the query and `shop: B` in the
    body resolves to tenant A.
  - Soft-deleted tenant: no data returned.
  - No `Set-Cookie` is ever emitted on `/admin/v1/**`.

### 5.2 Backend: install via Token Exchange

`application_url` moves from the backend install endpoint to the admin
root (§5.4). On an authenticated admin call (in practice the first
`GET /admin/v1/auth/me`), if the verified shop has no usable
`platform_credentials` row:

1. **Take a per-shop lock.** Redis `SET NX` with a short TTL, so
   concurrent first calls run one exchange. This also closes the
   "3× concurrent install" finding.
2. **Exchange.** POST the session token to Shopify's token endpoint with
   the token-exchange grant, asking for an **offline** access token
   (S4).
3. **Provision.** Reuse the existing `ShopifyInstallService` logic:
   - provision-or-reactivate the tenant;
   - store the encrypted credential (`OAuthTokenCipher`);
   - compare granted and expected scopes (the 2026-10-14 follow-up);
   - register webhooks per topic (#64);
   - publish `TenantInstalledEvent`.
   - **Do not** mint an invite.
4. **Respond.** Return `/me` with `tenant.active = false`, so the admin
   routes to onboarding.

While Phase 1 is in transition, keep the existing `/oauth/shopify/*`
endpoints. Retire them in Phase 2 once Token Exchange is proven (§6).
Check Shopify's current position on **expiring offline tokens** (refresh
tokens) before building the credential model (S4).

### 5.3 Admin (Next.js)

- **`client.ts`**
  - Before each request, `await shopify.idToken()` and set
    `Authorization: Bearer <token>`. Do this explicitly, rather than
    relying on App Bridge's automatic fetch header, so it is visible and
    testable.
  - Remove the XSRF read/echo.
  - Use `credentials: "omit"`.
  - Honour the retry header once.
- **Session state.** `SessionProvider` keeps probing `/me`, now with the
  bearer token. Remove `login` and `signup`; logout is not offered.
- **Pages.** Remove `/login` and `/signup`.
- **Outside Shopify.** Add a minimal page for "not embedded" (no App
  Bridge context): "Open WiseGift from your Shopify admin". It shows no
  data and offers no login form.
- **Proxy.** Keep the same-origin `rewrites()`. Confirm it forwards the
  `Authorization` header unchanged (F2) and that nothing in the Next or
  Render request logs records it.
- **Referrer.** Apply `Referrer-Policy: no-referrer` (or
  `strict-origin`) site-wide. The initial app URL Shopify loads carries
  `shop`, `host`, `hmac` and a short-lived `id_token` query parameter
  (S1).

### 5.4 Shopify app config (`wisegift-widget/shopify.app.toml`)

- `application_url` → `https://<admin-host>/`, the staging admin.
- Keep `embedded = true` and `use_legacy_install_flow = false`.
- `[auth] redirect_urls`: keep the callback until Phase 2, then remove.
- Releasing this needs `shopify app deploy`, which is **PO-gated**, after
  a widget build (2026-10-14 finding 4).

---

## 6. Phased migration plan

**Phase 0: decide (now, docs only).** The PO decides §9.

**Fresh-install test (runbook step 7): defer it.** Its result no longer
changes the decision, because A is needed either way. Running it now
creates a staging tenant and syncs a catalog through OpenAI for an
expected failure. Run it once, against the Phase 1 build (step 1e). If
the PO runs it anyway, an iframe CSRF/401 failure is the expected
result, not a deploy defect, and needs no workaround. Do **not** loosen
cookie settings.

**Phase 1: pilot blocker.** Separate PRs, one concern each. Backend PRs
target `develop`; admin and widget PRs target `main`.

| Step | Repo | Content |
|---|---|---|
| 1a | backend | Verifier + stateless `/admin/v1/**` bearer chain + reshaped `AdminPrincipal` + tests (§5.1). **Transition:** the bearer chain matches only requests carrying `Authorization: Bearer`. Other requests fall through to the existing cookie chain, so staging keeps working until the admin switches. That is temporarily Option C, on staging only. |
| 1b | backend | Token Exchange install path with a per-shop lock (§5.2) |
| 1c | admin | Bearer client, remove login/signup UI, not-embedded page, per-shop `frame-ancestors` (§5.3, F3) |
| 1d | widget (config) | `application_url` → admin; `shopify app deploy` (PO-gated), after 1a–1c are on staging |
| 1e | verification (PO) | Fresh install on a **new** dev store: install → scope approval → onboarding wizard inside the Shopify admin, with no signup. Repeat the app open in Chrome, Chrome Incognito, Safari and Firefox. Reopen the app and confirm no new install, catalog sync or webhook registration. Then uninstall. |

**Phase 2: before the first pilot install.** Remove the old path:

- Backend:
  - Drop the cookie chain for `/admin/v1/**`.
  - Remove the password/Google login and signup endpoints, invite
    issuance, `InviteTokenService`, and the backend `/admin/signup`
    placeholder.
  - Remove `/oauth/shopify/install` and `/callback`, plus the TOML
    `redirect_urls`.
  - Remove the Spring Session cookie config for admin.
- Config: remove `WISEGIFT_ADMIN_SIGNUP_URL` and its boot check,
  `WISEGIFT_ADMIN_INVITE_TOKEN_SECRET`, and `WISEGIFT_GOOGLE_CLIENT_ID`.
- Data: delete the staging `admin_users` rows. Production has none.

**Phase 3: later.**
- Flyway migration dropping `admin_users`.
- Built for Shopify / App Store review checks (S5).
- Decide the non-Shopify login when VTEX/SFCC is scheduled (P5).

**Migrating the existing staging admin user.** No data migration is
needed. The `wisegift-dev` tenant becomes reachable through its shop in
the Shopify admin. The email/password row is deleted in Phase 2.

### 6.1 Keep or remove

| Keep | Remove (phase) |
|---|---|
| Same-origin proxy `/admin/v1/*` (no CORS on admin) | `/login`, `/signup` pages and the `/signup` Referrer-Policy rule, superseded by the site-wide rule (1c) |
| App Bridge provider (PR #6) | XSRF logic and `credentials: "include"` in `client.ts` (1c) |
| `frame-ancestors` CSP, tightened per shop (F3) | `CookieCsrfTokenRepository` on the admin chain (2) |
| 401 entry point for unauthenticated admin calls | Password + Google login/signup endpoints, `GoogleIdTokenVerifier`, `PasswordEncoder` bean (2) |
| `AdminPrincipal` as the **only** tenant source, reshaped (1a) | `InviteTokenService`, `ConsumedTokenStore` usage for invites, `WISEGIFT_ADMIN_SIGNUP_URL` (2) |
| `ShopifyHmacVerifier` (webhooks), `OAuthTokenCipher`, tenant provisioning and widget-config seeding | `SessionConfig` + `AbsoluteSessionExpiryFilter`, if nothing else uses HTTP sessions (2; B4) |
| Redis (cache, caps, kill switch, install lock) | `/oauth/shopify/install`, `/callback`, `/success`, backend `/admin/signup` (2) |
| `/me` endpoint, with a reshaped response (B3) | `admin_users` table (3) |

### 6.2 Side findings (not blocking this decision)

Noted while reading; all low severity.

1. **`/oauth/shopify/install` HMAC is optional.** It is checked only if
   the `hmac` parameter is present. Low risk, because the step only
   redirects to Shopify with a state nonce, and the endpoint is removed
   in Phase 2. If Phase 2 slips, require the HMAC.
2. **Widget CORS sets `allowCredentials = true`** with an any-origin
   pattern on `/widget/v1/**`. The widget uses bearer tokens and no
   cookies, so set it to `false` (least privilege).
3. **`frame-ancestors https://*.myshopify.com`** allows any shop to frame
   the admin. Under A, a framing shop still only sees its own tenant, so
   the impact is low. Shopify expects the per-shop form (F3).

### 6.3 Security notes specific to A

- **The app client secret becomes the master key.** It already signs
  webhooks and OAuth. Under A it also authenticates every admin request
  for every tenant. Keep it backend-only, in Render secrets, never in
  logs. Rotation is PO-gated. Support a previous-key window during
  rotation (D1).
- **Clock skew.** Render hosts use NTP. Keep the leeway small (5–10 s),
  not minutes.
- **Never log** the `Authorization` header, the `id_token` query
  parameter, or decoded claims beyond `dest` host and reason code.
  Logging `sub` is a privacy decision (P3).

### 6.4 Privacy notes (informational, not legal advice)

- **Today:** for each merchant admin we hold an email, a BCrypt password
  hash, optionally a Google `sub`, and `created_at`. This is merchant
  **staff** data, outside the shopper-focused DPA (DPA clause 5 covers
  storefront visitors only). WiseGift is probably the controller for
  this account data. **ATTORNEY:** confirm the role, and what the
  merchant-admin privacy notice must say.
- **Under A with no `sub` storage:** we persist no staff personal data.
  In memory we process the Shopify user ID (`sub`) and the shop domain
  per request. This is a clear data-minimisation gain: no password-hash
  breach exposure, no email list, no Google identity flow.
- **Online access tokens would add data.** They return the staff
  member's name and email. Recommendation: don't request them.
- **Gap this creates: a merchant contact.** The DPA requires WiseGift to
  notify the Controller of a personal-data breach, and billing and
  pilot operations need a contact. Today the admin email is that
  contact by accident. Under A we need an explicit one. See P4.
- **Sub-processors.** No change: Shopify is upstream, not a
  sub-processor, and no new vendor is added. Removing Google sign-in
  removes Google from the merchant-admin flow. Update the merchant-admin
  privacy notice draft accordingly when it is written.
- **EU-only** is unaffected. Verification and Token Exchange run on the
  EU backend. The Token Exchange call goes to Shopify, the merchant's own
  platform.

---

## 7. Effort estimate

Rough, single engineer, including tests. Re-estimate after B1.

| Repo / owner | Phase 1 | Phase 2–3 | Notes |
|---|---|---|---|
| `wisegift-backend` (backend-engineer) | 4–5 days: verifier + chain + principal + tests ≈ 2–3; Token Exchange + lock + reactivation ≈ 2 | 1–2 days removal + Flyway drop | Biggest risk: Token Exchange and offline-token details (S4) |
| `wisegift-admin` (frontend-engineer) | 2–3 days | < 0.5 day | Depends on admin PR #6 (App Bridge loading) |
| `wisegift-widget` TOML (frontend-engineer, PO deploys) | < 0.5 day | < 0.5 day (`redirect_urls`) | `shopify app deploy` is PO-gated and needs a widget build first |
| Infra (devops-expert) | 0.5–1 day | 0.5 day env cleanup | Starter plan before pilot (already decided). Under A a cold start can also outlast the 1-minute token, which the retry header covers |
| **Total** | **≈ 7–9 dev-days** | **≈ 2–3 dev-days** | |

---

## 8. Open questions

### Backend-engineer
- **B1.** Re-check everything in §2 and §5 against `origin/develop`.
  This proposal read a stale working tree (see the caveat at the top).
- **B2.** Nimbus JOSE (already reachable through Spring Security's JOSE
  module, if on the classpath) or a hand-rolled verifier with a pinned
  header? Either is acceptable if it pins `HS256`, compares in constant
  time and parses claims with Jackson.
- **B3.** `/me` shape. The "locked, additive-only" `admin { id, email,
  createdAt }` no longer has an email. Propose the new shape: for
  example, drop `admin` and add `shop { domain }`. Admin and backend
  ship together, so no external consumer breaks.
- **B4.** Does anything besides admin auth use HTTP sessions or Spring
  Session? If not, remove `@EnableRedisHttpSession` in Phase 2.
- **B5.** Reinstall within 30 days: confirm the Token Exchange path
  reactivates the soft-deleted tenant and reuses `widget_config`, as
  `refreshExisting` does today.

### Frontend-engineer
- **F1.** Get the token explicitly with `shopify.idToken()`, or rely on
  App Bridge's automatic `Authorization` on same-origin fetch? The
  recommendation is explicit; confirm with current App Bridge docs.
- **F2.** Confirm that the Next `rewrites()` proxy forwards
  `Authorization` and that no middleware or logging captures it.
- **F3.** Per-shop `frame-ancestors` (`https://{shop}.myshopify.com
  https://admin.shopify.com`) from the validated `shop` query parameter,
  set in Next middleware. Does Shopify still require this form?
- **F4.** UX for the not-embedded page, and for the "installing…" state
  while the first `/me` performs Token Exchange (it can take seconds).

### Devops-expert
- **O1.** Change `application_url` and `redirect_urls`, then
  `shopify app deploy` (PO-gated). Add it to the runbook, after the
  widget build step.
- **O2.** Do Render or Next request logs record query strings or
  `Authorization` headers? Ensure `id_token` and bearer tokens are not
  retained.
- **O3.** Phase 2 env cleanup and the client-secret rotation procedure,
  including a previous-key window.
- **O4.** Update `runbooks/admin-staging-deploy.md`. Smoke checks 4.2–4.4
  and step 7 describe the cookie/signup flow.

### PO
- **P1.** Approve Option A (session tokens only, no standalone admin at
  MVP)? See §9.
- **P2.** Approve Token Exchange install, which retires the invite and
  signup flow from #73?
- **P3.** Staff identity: persist nothing (recommended), or log the
  Shopify `sub` on config changes for an audit trail? Logging it is
  pseudonymous personal data, and the retention period would need to be
  stated.
- **P4.** Merchant contact for DPA breach notices and billing. Options:
  (a) an explicit "notification email" field in onboarding, pre-filled
  from the Shopify shop email, which the merchant confirms
  (recommended); (b) use the Shopify shop email silently. **ATTORNEY**
  for the notice wording.
- **P5.** Confirm that the standalone (non-Shopify) admin is deferred
  until VTEX/SFCC is scheduled, and is then designed fresh.
- **P6.** Defer the fresh-install test (runbook step 7) to Phase 1e?
- **P7.** Product doc follow-up (`product-analyst`): spec.md (onboarding
  "redirected to the onboarding wizard"; Out of scope "email + password
  + Google OAuth are enough") and architecture.md §1, §3, §4 (Admin API
  "session → user → tenant") and §9.1/§9.3. Update them once the
  decision is logged.

### Shopify facts to verify against current docs before building
These are stated from knowledge, not verified in this session.
- **S1.** The initial embedded load carries `id_token` (plus `shop`,
  `host`, `hmac`) in the query string.
- **S2.** Session-token claims (`iss`, `dest`, `aud`, `sub`, `exp`,
  `nbf`, `iat`, `jti`, `sid`), a lifetime of about 60 s, and the leeway
  Shopify's own libraries use.
- **S3.** The App Bridge retry header name and behaviour on 401.
- **S4.** Token Exchange endpoint and parameters for an offline token,
  and Shopify's current policy on **expiring offline access tokens and
  refresh tokens** (whether and when it applies to new apps).
- **S5.** The current App Store requirement and Built for Shopify
  criterion wording on session tokens, third-party cookies, and separate
  signup/login for embedded apps.
- **S6.** Safari's current CHIPS / third-party cookie behaviour in
  cross-site iframes. This matters only if B is reconsidered.

---

## 9. Decisions requested from the PO

1. **Auth model: Option A**, Shopify session tokens only (recommended).
2. **Install: Token Exchange**, `application_url` → admin; the invite
   and signup flow is retired (recommended).
3. **No standalone admin at MVP**; revisit with VTEX/SFCC (recommended).
4. **Staff data: persist nothing**; no Shopify `sub` audit log at MVP
   (recommended).
5. **Merchant contact email:** explicit field in onboarding, pre-filled
   from the shop email (recommended; attorney for notice wording).
6. **Fresh-install test:** defer to Phase 1e (recommended).
7. **Skip the pending cookie-hardening items** (CSRF cookie flags,
   `changeSessionId()`, anonymous `/me` sessions) under A, since Phase 2
   removes that code (recommended).

*Informational draft by `security-and-privacy`. Not a formal audit or
penetration test, and not legal advice. Before an Enterprise tenant or
App Store submission, commission a professional security assessment of
the admin auth path.*
