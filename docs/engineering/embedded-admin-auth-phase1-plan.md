# Embedded admin auth: Phase 1 implementation plan

> **ACCEPTED by PO 2026-10-15 (PO-1..PO-5 + rotation guard; see decisions.md).** Prepared by `backend-engineer` on 2026-10-15.
> Implements the PO decision logged in `decisions.md` (2026-10-15, "Embedded
> admin auth: Shopify session tokens (Option A); platform identity as a
> separate layer") on top of `embedded-admin-auth-proposal.md`.
> This is a plan only. No code, no commits, no deploys.
>
> **Evidence base.**
> - Backend: a detached worktree of `origin/develop` at `5c0cdbf` (includes #71–#76).
> - Admin: `origin/main` at `6b8acae` (includes admin #6).
> - Widget: `origin/main` at `f2d20d3`.
> - Shopify facts: fetched from shopify.dev on 2026-10-15 (sources in §2). Where
>   the docs are silent, Shopify's own `shopify-app-js` library source is used and
>   labelled as such.

---

## 0. Spec features this plan serves

- **Feature: Merchant onboarding — Shopify app install** (spec.md).
  - "Install flow uses standard Shopify OAuth": satisfied by Shopify-managed
    install plus Token Exchange, which is Shopify's documented OAuth grant for
    embedded apps.
  - "On successful OAuth, a `tenant` record is created…".
  - "The merchant is redirected to the WiseGift admin dashboard's onboarding wizard."
  - "Uninstall … tenant is soft-deleted (retained 30 days for reactivation…)".
- **Feature: Merchant onboarding — Onboarding wizard.** The wizard must
  be resumable. It gains the contact-email step (decision 5). A spec AC is still
  missing for that step (§9, PO-4).
- **Multi-tenant isolation invariant.** The tenant comes only from the verified
  `dest` claim (decision 1).

The spec text still says "scopes `write_theme_extensions`" (not a valid Shopify
scope) and "redirected". `product-analyst` should update it (proposal P7).

---

## 1. Verification of the proposal's code claims against `origin/develop`

| # | Claim in proposal | Result on `origin/develop` | Impact on the plan |
|---|---|---|---|
| C1 | `admin_users` holds email + BCrypt hash or Google `sub`, one admin per tenant | **Confirmed.** V1 schema; `AdminAuthenticationService` rejects a second admin per tenant (`findByTenantId`) | None |
| C2 | Spring Session Redis cookie `WG-ADMIN-SESSION`, HttpOnly, Secure, SameSite=Lax, 24 h idle / 7 d absolute | **Confirmed.** `SessionConfig` (`@EnableRedisHttpSession(redisNamespace="wg:spring:session")`), `server.servlet.session.timeout: 24h`, `AbsoluteSessionExpiryFilter`. Note: `AbsoluteSessionExpiryFilter` is a `@Component` servlet filter that runs on **every** request, not only `/admin/**` | The bearer chain must stay cookie-blind even though this filter calls `getSession(false)` (§3.2) |
| C3 | `CookieCsrfTokenRepository` double-submit; `client.ts` echoes `X-XSRF-TOKEN` | **Confirmed.** Plus `SpaCsrfTokenRequestHandler`, `AdminCsrfCookieFilter`, `CsrfTokenRotationStrategy` (#71, #76) | Removed in Phase 2 |
| C4 | `AdminSecurityConfig` has **no** 401 entry point (stale-tree remark) | **Wrong for develop.** `HttpStatusEntryPoint(401)` exists. `NullRequestCache` (#76) is also present | None |
| C5 | A single filter chain | **Confirmed.** One `SecurityFilterChain` covers everything, with `WidgetAuthFilter` added before `UsernamePasswordAuthenticationFilter` | Phase 1 adds a second chain at `@Order(1)` (§3.2) |
| C6 | `AdminPrincipal(AdminUserId, TenantId, email)`, `Serializable` | **Confirmed** (#72). It lives in `tenant/infrastructure/in/web`, is consumed by `AdminWidgetConfigController` and `catalog`'s `CatalogAdminController`, and is guarded by `AdminSessionSerializationTest` | Live Redis sessions hold this class. **Do not rename or move it in Phase 1**: deserialising old sessions would fail with a 500. A new principal is added alongside it (§3.1) |
| C7 | OAuth callback redirects to backend-relative `/admin/signup` (stale tree) | **Wrong for develop** (#73). The callback 302s to the absolute `WISEGIFT_ADMIN_SIGNUP_URL` + invite token. `AdminSignupPageController` still forwards legacy `/admin/signup?token=` (TODO remove after 2026-11-01) | Phase 2 removes both |
| C8 | `InviteTokenService` is a hand-rolled HS256 JWS with an `indexOf` parser | **Confirmed.** Header pinned, `MessageDigest.isEqual`, verify/consume/release split (#73), Redis `ConsumedTokenStore`. The `indexOf` parser must not be reused for Shopify tokens | Not reused; Phase 2 deletes it |
| C9 | Offline token stored encrypted with `OAuthTokenCipher` | **Confirmed.** AES-256-GCM, `Base64(iv‖ct‖tag)`, key `WISEGIFT_ENCRYPTION_KEY`, **no key rotation support**. One decrypt site only: `ShopifyPlatformCatalogSource` | One choke point for a future token provider (expiring tokens, §8) |
| C10 | Token type | Code exchange uses `/admin/oauth/access_token` with JSON `{client_id, client_secret, code}`. No `expiring` parameter, so Shopify issues a **non-expiring** offline token. `TokenExchangeResponse` maps only `access_token` and `scope` | See §2 S4 and PO-1 |
| C11 | Webhook HMAC | **Correct.** Raw body, HMAC-SHA256 with `wisegift.shopify.webhook-secret`, base64, `MessageDigest.isEqual`, and an `X-Shopify-Webhook-Id` idempotency claim. It accepts **one** secret only | Rotation needs dual-secret acceptance (§3.6) |
| C12 | Webhooks registered per topic with one try/catch each (#64) | **Confirmed.** REST `webhooks.json` registration on install and re-auth. The webhook base URL is **derived from `WISEGIFT_SHOPIFY_REDIRECT_URI`** | Phase 2 cannot delete the redirect URI without first adding a `webhook-base-url` property (§7) |
| C13 | Install path provisions or reactivates the tenant, and `refreshExisting` reuses `widget_config` (B5) | **Partly wrong.** `refreshExisting` re-encrypts the token and keeps `widget_config`, but it **never clears `soft_deleted_at` and never reactivates the tenant**. A reinstall within 30 days today leaves the tenant soft-deleted. `completeInstall` is also not `@Transactional`: a failure after `provisionTenant` leaves an orphan tenant + `widget_config`, and two concurrent first installs create two tenants (the loser fails on the unique key) | Phase 1 fixes this in the new install path (§3.3) |
| C14 | §6.2-1: `/oauth/shopify/install` HMAC is optional | **Confirmed.** It is checked only if `hmac` is present. The callback always requires it | Low. The endpoint is deleted in Phase 2; require HMAC only if Phase 2 slips |
| C15 | §6.2-2: widget CORS `allowCredentials=true` with any origin | **Confirmed.** `allowedOriginPatterns("*")` + `setAllowCredentials(true)` on `/widget/v1/**`. The comment says it is "to keep the door open" | Optional 1-line PR (BE-6) |
| C16 | §6.2-3: `frame-ancestors https://admin.shopify.com https://*.myshopify.com` | **Confirmed**, set in the **admin** `next.config.ts` `headers()`, not in the backend. Shopify requires the per-shop form (S5) | ADM-2 |
| C17 | `application_url` = backend `/oauth/shopify/install`, `embedded = true`, `use_legacy_install_flow = false` | **Confirmed** on widget `origin/main`. `redirect_urls` = backend callback. Scopes = `read_inventory,read_orders,read_products` (the stale tree lacked `read_inventory`). `[webhooks] api_version = "2026-10"`. **No compliance webhooks** (`customers/data_request`, `customers/redact`, `shop/redact`). `name = " WiseGift Staging"` has a leading space | WID-1 |
| C18 | Anything else using HTTP sessions (B4)? | **No.** Only admin auth (`AdminAuthController`, `SessionConfig`). `catalog/pom.xml` has `spring-session-data-redis` in **test** scope only. `app/pom.xml` also carries `spring-boot-starter-oauth2-client`, which no code uses | Phase 2 removes `@EnableRedisHttpSession`, the dependency and the unused oauth2-client starter |
| C19 | Next `rewrites()` same-origin proxy; `client.ts` uses `credentials: "include"` | **Confirmed** | ADM-1 |

**Additional findings (not in the proposal):**

- **F-A. `custom_domain` is always null at install.** `ShopifyInstallService`
  reads `shop.getPrimary_domain()`, but REST `shop.json` has no `primary_domain`
  field (shopify.dev REST Shop resource; the primary domain is `domain`).
  Storefronts on a custom domain fail the widget `Origin` check until the first
  `shop/update` webhook arrives, which falls back to `domain`. **Medium for
  pilots on custom domains.** Phase 1 fixes it in the new install path (read
  `domain`), test-first.
- **F-B. Admin API version `2024-10`** is past support (already logged
  2026-10-14). New public apps must use **GraphQL only** (App Store requirement
  2.2.4). This does not block Phase 1, but it blocks App Store submission and
  interacts with expiring tokens (§2 S4).
- **F-C. Hexagonal violations already exist.** `ShopifyInstallService`
  (application) imports `infrastructure` classes (`ShopifyAdminApiClient`,
  `ShopifyProperties`, `OAuthTokenCipher`, `OAuthStateStore`), and so does
  `InviteTokenService` (`AdminProperties`). New Phase 1 code goes through ports.
  The existing violations are not "fixed" silently here.
- **F-D. Redis is `allkeys-lru`** (backend `render.yaml`), so any key can be
  evicted. A Redis `SET NX` install lock can vanish under memory pressure. The
  plan uses a Postgres advisory lock instead (§3.3).
- **F-E. The outbound `RestClient` has no timeouts** (`JdkClientHttpRequestFactory`
  defaults). The new token-exchange call sets explicit connect and read timeouts.
- **F-F. `/me` is not cacheable by contract today.** The bearer chain adds
  `Cache-Control: no-store` on `/admin/v1/**` (Spring Security default
  headers already do this; a test pins it).

---

## 2. Shopify facts S1–S6, checked against official sources (2026-10-15)

| ID | Proposal statement | Status | Source and what it says |
|---|---|---|---|
| **S1** | The initial embedded load carries `id_token` (plus `shop`, `host`, `hmac`) in the query string | **VERIFIED** (via Shopify's official library; the docs prose does not spell it out) | `github.com/Shopify/shopify-app-js` → `packages/apps/shopify-app-react-router/src/server/authenticate/admin/helpers/ensure-session-token-search-param-if-required.ts`: `SESSION_TOKEN_PARAM = 'id_token'`, read with `shop` and `embedded=1`. `validate-shop-and-host-params.ts` reads `shop` and `host`. Consequence: keep `Referrer-Policy` site-wide and never log admin page URLs |
| **S2** | Claims `iss, dest, aud, sub, exp, nbf, iat, jti, sid`; ~60 s lifetime; library leeway | **VERIFIED, with two corrections** | https://shopify.dev/docs/apps/build/authentication-authorization/id-tokens. Session tokens are now called **"ID tokens"**. They are HS256 with the client secret and "expire one minute after they're issued". Validate `exp` (future), `nbf` (past), `aud` = client ID, and `iss`/`dest` **hostnames must match**. `sub`, `sid`, `jti` and `iat` are read-only: "You don't validate these". App Bridge "may return a cached token with less than the full minute remaining". Leeway: `shopify-app-js` `packages/apps/shopify-api/lib/session/decode-session-token.ts` uses `JWT_PERMITTED_CLOCK_TOLERANCE = 10` with `algorithms: ['HS256']`. **Corrections:** (a) do **not** require `sub` (the proposal did); (b) leeway = 10 s. **Doc inconsistency:** the App Bridge ID Token API page's example token begins `eyJhbGciOiJSUzI1NiIs…` (RS256), which contradicts the HS256 statement on the id-tokens page and in the official library. Pin HS256, and confirm with the header of one real staging token before BE-1 merges (risk R3) |
| **S3** | Retry header name and behaviour | **VERIFIED** | https://shopify.dev/docs/apps/build/authentication-authorization/implement-token-exchange: "When you reject a request with a `401`, set the `X-Shopify-Retry-Invalid-Session-Request` header if the request came from your frontend over XHR or `fetch`. App Bridge intercepts the response, fetches a fresh ID token, and retries the request once." The same page maps a Token Exchange **400** (stale ID token) to `401` + this header. Also https://shopify.dev/docs/api/app-home/apis/authentication-and-data/resource-fetching-api: App Bridge's fetch interceptor adds the ID token in `Authorization` **for requests to your app's domain and subdomains**, which means the `application_url` host |
| **S4** | Token Exchange endpoint and parameters for offline; expiring offline tokens | **CHANGED (material)** | https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens#token-exchange-grant: `POST https://{shop}.myshopify.com/admin/oauth/access_token`, form-urlencoded, with `client_id`, `client_secret`, `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`, `subject_token=<id token>`, `subject_token_type=urn:ietf:params:oauth:token-type:id_token`, `requested_token_type=urn:shopify:params:oauth:token-type:offline-access-token` (default), and `expiring` = `1` for expiring or `0` (default) for non-expiring. A stale ID token gives `400`. **Expiring offline tokens:** "New public apps already can't use [non-expiring offline tokens] for GraphQL Admin API requests, and existing public apps can't after January 1, 2027. … This doesn't apply to custom apps." An expiring token lasts 1 h with a 90-day rotating refresh token. Each exchange **retires** the store's other expiring tokens and refresh tokens, and "Don't [exchange and refresh] at the same time for the same store". Sources: …/migrate-to-expiring-offline-access-tokens and …/implement-token-exchange#refresh-an-expiring-offline-token. Online tokens return `associated_user` with email, so not requesting them is confirmed (decision 4). **Plan impact:** pilots (custom apps) can keep non-expiring tokens. The public App Store app must use expiring tokens **and** GraphQL. See PO-1 |
| **S5** | App Store / Built for Shopify wording on session tokens, third-party cookies and separate signup | **VERIFIED, stronger than stated** | https://shopify.dev/docs/apps/launch/shopify-app-store/app-store-requirements: **1.1.1 "Use session tokens for authentication.** Your embedded app must function properly without relying on third-party cookies **or local storage**, including when accessed in incognito mode on Chrome." **2.3.2** "Your app must immediately authenticate using OAuth before any other steps occur. Merchants should not be able to interact with the UI before OAuth." **2.3.4** is the same after reinstall. **2.2.4** says GraphQL only for new public apps since 2025-04-01. The id-tokens page says "Apps embedded in the Shopify admin **must** authenticate with ID tokens". https://shopify.dev/docs/apps/launch/built-for-shopify/requirements: **3.1.3** "Enable seamless sign up based on Shopify credentials … without requiring an additional login or sign-up prompt". **2.1.1** requires LCP ≤ 2.5 s at p75. https://shopify.dev/docs/apps/build/security/set-up-iframe-protection says `frame-ancestors https://{shop}.myshopify.com https://admin.shopify.com`, "must be different for every shop" and present on every HTML route (answers F3) |
| **S6** | Safari blocks third-party cookies; CHIPS inconsistent | **CHANGED (does not affect the decision)** | MDN browser-compat-data `http/headers/Set-Cookie.json` → `Partitioned`: Chrome 114, Firefox 141, **Safari 26.2** (briefly 18.4, removed in 18.5). https://webkit.org/tracking-prevention/: third-party cookies are still blocked by default. So "B fails in Safari by default" is now true only for Safari < 26.2. Option B stays rejected on firmer ground: App Store 1.1.1 and the id-tokens page require ID-token auth for embedded apps outright |

**Facts that change the plan:**
1. Expiring offline tokens plus GraphQL are mandatory for the public app. PO-1
   decides whether Phase 1 builds refresh (+2–3 dev-days) or defers it to the
   GraphQL migration.
2. App Store 2.3.2/2.3.4: the admin must not render interactive UI before the
   offline token exists. The admin blocks on the installation call (§4).
3. The App Bridge fetch interceptor only attaches tokens for the
   `application_url` host. Until WID-1 ships, the admin must attach the token
   explicitly. The plan attaches it explicitly anyway (F1 answered: explicit).
4. Webhook signing during secret rotation uses the **oldest unrevoked** secret
   (https://shopify.dev/docs/apps/build/authentication-authorization/manage-credentials).
   The docs do not say which secret signs ID tokens during rotation
   (**UNVERIFIED**). Both verifiers therefore accept current and previous.
5. 1.1.1 forbids relying on **local storage** too. The admin has no
   `localStorage` usage in `src/` today; keep it that way.

---

## 3. Backend design (`wisegift-backend`, PRs target `develop`)

### 3.1 Platform-identity layer (decision 8)

**Goal:** turn a platform credential into the platform-agnostic `AdminPrincipal`.
Admin controllers, tenant isolation and business rules never see Shopify types.

**Module placement.** These follow the existing module graph
(`platform-integrations → tenant → shared`; `catalog → platform-integrations`;
`app → all`) and the `WidgetPrincipal` precedent.

| Type | Module / package | Role |
|---|---|---|
| `AdminPrincipal(TenantId tenantId, String platform, String storeDomain)` | `shared` → `com.wisegift.shared.admin` | The only identity admin controllers see. A plain record, **not** `Serializable` (no session store holds it). `platform` is a string so `shared` stays free of the `Platform` enum |
| `VerifiedPlatformIdentity(Platform platform, String storeDomain)` | `platform-integrations` → `domain/model` | The output of a verifier. Carries no tenant |
| `PlatformIdentityVerifier` (interface) | `platform-integrations` → `application/port/out` | `Platform platform()`; `VerifiedPlatformIdentity verify(String credential)` throws `InvalidPlatformCredentialException(reason)` |
| `ShopifyIdTokenVerifier` | `platform-integrations` → `infrastructure/security` (next to `ShopifyHmacVerifier`) | The first implementation (§3.1.1) |
| `AdminAuthenticationQuery` (port in) + `PlatformAdminAuthenticator` (service) | `platform-integrations` → `application/port/in`, `application/service` | **Query, side-effect free.** Verifies, then binds the tenant. Returns the sealed result `Authenticated(AdminPrincipal)` or `InstallationRequired(VerifiedPlatformIdentity)` |
| `InstallAppCommand` (port in) + `EmbeddedInstallService` | same | **Command.** Token Exchange install/reinstall (§3.3) |
| `AdminBearerAuthFilter`, `AdminBearerSecurityConfig`, `AdminAuthErrorWriter` | `app` → `com.wisegift.security` | Wiring only, following the `WidgetAuthFilter` pattern |

`PlatformAdminAuthenticator` holds a `List<PlatformIdentityVerifier>`. At MVP the
list has exactly one bean, and startup fails if there are none or more than one.
The selection rule for a second verifier (VTEX/SFCC) is decided when that
verifier lands: for example a distinct admin build per platform, or a header
set by the platform's shell. Building it now would be speculative.

**Tenant binding (the isolation invariant):**

```
credential ──verify──▶ VerifiedPlatformIdentity(platform, storeDomain)   // from the dest claim only
          ──PlatformCredentialRepository.findByPlatformAndShopDomain──▶ tenant_id
          ──TenantQueryPort.findById──▶ tenant
   live  = credential present AND tenant.softDeletedAt == null   → Authenticated(AdminPrincipal)
   else                                                           → InstallationRequired
```

- **No other input is consulted:** no `?shop=`, body field, path segment,
  `X-Shopify-Shop-Domain` header or cookie. `tenants.active = false` is **not**
  a rejection, because the onboarding wizard runs before go-live.
- Controllers keep taking the tenant only from the principal (architecture.md §9.3).

**Phase 1 coexistence with the legacy principal.** During Phase 1 the cookie
chain still produces the legacy `tenant…AdminPrincipal`, while the bearer chain
produces `shared.admin.AdminPrincipal`.

- A small `AdminTenantResolver` in `tenant/infrastructure/in/web` returns the
  `TenantId` from either type, or empty.
- `AdminWidgetConfigController` and `CatalogAdminController` switch to it.
  They currently duplicate an `instanceof` block; that becomes one call.
- Phase 2 deletes the legacy branch, and the resolver collapses to
  `@AuthenticationPrincipal AdminPrincipal`.

#### 3.1.1 `ShopifyIdTokenVerifier`: rules (fail closed)

The verifier uses **Nimbus JOSE** (B2 answer), declared explicitly in
`platform-integrations/pom.xml`. Today it arrives only transitively through
`tenant`'s `oauth2-resource-server`, which Phase 2 may remove. It is not
hand-rolled.

1. Parse as a JWS. `alg` must be exactly `HS256`, checked against a pinned
   `JWSAlgorithm.HS256` and never taken from the token. Reject `none`, any
   `RS*`/`ES*`, and any `crit` header.
2. **Signature.** Use `MACVerifier` with the current secret, then the previous
   secret if one is configured (§3.5). Nimbus compares in constant time.
   Nimbus requires a key ≥ 256 bits. Shopify client secrets (`shpss_…`) are
   38 bytes, and a boot check asserts ≥ 32 bytes.
3. **Claims**, parsed with Nimbus' JSON parser:
   - `exp > now − 10 s` and `nbf ≤ now + 10 s` (S2);
   - `exp − iat ≤ 5 min` as a sanity bound;
   - `aud` equals `wisegift.shopify.client-id` exactly;
   - the `dest` host matches `^[a-z0-9][a-z0-9-]*\.myshopify\.com$` (the
     existing `SHOP_DOMAIN` regex, moved to one shared constant);
   - `dest` is exactly `https://{host}` and `iss` is exactly `https://{host}/admin`
     (stricter than Shopify's "hostnames match").
   - `sub`, `sid` and `jti` are **not** required (S2) and are never stored or
     logged (decision 4).
4. **Logging.** Log only the reason code and the `dest` host. Never log the
   token, the `Authorization` header, or other claims. (Shopify's own library
   puts the raw token in its exception message; we must not copy that.)
5. Every failure throws `InvalidPlatformCredentialException` with a reason
   enum (`MALFORMED`, `ALG`, `SIGNATURE`, `EXPIRED`, `NOT_YET_VALID`,
   `AUDIENCE`, `DEST`, `ISSUER`, `LIFETIME`). The filter turns all of them
   into the same 401.

### 3.2 Stateless admin security chain (bearer, CSRF off)

New `AdminBearerSecurityConfig`, `@Order(1)`. The existing chain becomes
explicitly `@Order(2)`, unchanged.

- `securityMatcher`: `/admin/v1/**` **and** an `Authorization` header is
  present. Any scheme other than `Bearer` returns 401 on this chain; it does
  not fall through. Requests without an `Authorization` header go to the
  legacy chain, so the PO's standalone staging login keeps working until
  Phase 2. Phase 2 drops the header condition.
- `csrf(disable)`, `cors(disable)`, `sessionManagement(STATELESS)`,
  `securityContext(RequestAttributeSecurityContextRepository)` (never
  `HttpSession`), `requestCache(disable)`, `formLogin`/`httpBasic`/`logout`
  disabled.
- `AdminBearerAuthFilter` (before `AnonymousAuthenticationFilter`) calls
  `AdminAuthenticationQuery`. It sets one of two authentications:
  - `Authenticated` → `AdminPrincipal` with authority `ADMIN`;
  - `InstallationRequired` → a `PendingInstallation` principal (identity only,
    no tenant) with authority `INSTALLATION_PENDING`. The raw ID token is kept
    as a request attribute only for the install command.
- Authorization rules:
  - `POST /admin/v1/installation` → `ADMIN` or `INSTALLATION_PENDING`
    (idempotent for `ADMIN`);
  - `GET /admin/v1/auth/me` → `ADMIN` (pending is handled by the denied handler);
  - `/admin/v1/auth/{login,signup,logout}/**` → `denyAll` on this chain;
  - `/admin/v1/**` → `ADMIN`.
- **Error writer (contract in §3.6):**
  - entry point → `401` + `X-Shopify-Retry-Invalid-Session-Request: 1` + body;
  - access denied for a `PENDING` principal → `409 installation_required`;
  - other access denied → `403`.

**Why CSRF off is safe on this chain:**

1. The credential travels in the `Authorization` header. Browsers never attach
   it automatically, unlike cookies, so a forged cross-site request carries no
   credential.
2. A cross-origin script cannot set `Authorization` without a CORS preflight.
   Neither the admin host nor the backend's `/admin/v1/**` grants CORS: backend
   CORS is scoped to `/widget/v1/**`.
3. The chain is **cookie-blind**. With STATELESS plus the request-attribute
   context repository, a `WG-ADMIN-SESSION` cookie riding along is never read
   for authentication.
4. The token is bound to our app (`aud`) and to one shop (`dest`), and lives
   about 60 s.

Tests pin all four points (§3.7).

### 3.3 Install / reinstall via Token Exchange (decision 2)

**Endpoint:** `POST /admin/v1/installation`. It is a command; no body is needed.
The proposal had the first `GET /me` trigger the exchange. This plan uses an
explicit POST instead, because a GET with side effects breaks CQRS and HTTP
semantics. App Store 2.3.2 is still met: the admin blocks all UI until the call
succeeds (§4).

`EmbeddedInstallService.install(identity, idToken)`:

1. **Per-shop lock.** Use a transaction-scoped Postgres advisory lock,
   `pg_advisory_xact_lock(hashtextextended('shopify:' || shop, 0))`, behind an
   `InstallationLock` out-port. Do not use Redis: it is `allkeys-lru` (F-D).
   The same lock must later guard token refresh, because Shopify says an
   exchange and a refresh must not run concurrently for one store (S4).
2. **Re-check inside the lock.** If the credential is live (§3.1 rule), return
   `ALREADY_INSTALLED` with no exchange. This makes concurrent and repeated
   calls idempotent: the second caller waits on the lock, then no-ops.
3. **Exchange** through the `PlatformAccessTokenGrant` out-port, implemented by
   `ShopifyTokenExchangeClient`:
   - form-urlencoded POST with the S4 parameters,
     `requested_token_type=…offline-access-token` and `expiring=0` (sent
     explicitly; PO-1 may change it to `1`);
   - connect timeout 5 s, read timeout 10 s (F-E);
   - if the response unexpectedly carries `expires_in`, **fail loudly**
     (`installation_failed`, error log) rather than store a token that dies in
     one hour without refresh support.
4. **Fetch shop** (`id`, `domain`, `email`). Read `domain` as the custom domain
   (fixes F-A). Use the existing REST client at the existing version.
   `email` is **not** persisted (§3.4).
5. **Provision, inside the same transaction:**
   - **New shop:** `provisionTenant` + seed `widget_config` + insert the
     credential (scopes from the response).
   - **Existing credential, tenant soft-deleted (reinstall ≤ 30 days):**
     - re-encrypt the token and update scopes, shop id and custom domain;
     - **reactivate** the tenant: `soft_deleted_at = null` and `active = false`.
       Shopify removes theme app extension blocks on uninstall, so the merchant
       must redo placement and go live again (PO-3);
     - keep `widget_config` and the catalog rows.
   - **No credential, tenant hard-purged (> 30 days):** treat as a new shop.
     The purge job (M3·1) does not exist yet.
   - The provisioning logic shared with the legacy callback is extracted into
     a `ShopifyTenantProvisioner`. The legacy callback keeps using it until
     Phase 2 deletes the callback.
6. **After commit**, outside the lock (`@TransactionalEventListener(AFTER_COMMIT)`
   or explicit sequencing):
   - register webhooks per topic, reusing #64's per-topic try/catch;
   - compare granted and expected scopes; on a mismatch, log a warning and
     expose `installation.missingScopes` in `/me` rather than block the install;
   - publish `TenantInstalledEvent` (async initial sync). On reinstall this
     resyncs a catalog that may be up to 30 days stale.
7. **Respond** `200` with the `/me` body.

The exchange runs inside the DB transaction, so the transaction holds a pooled
connection for up to about 15 s. That is acceptable for a once-per-shop
operation. It is not acceptable in any per-request path, and none uses it.

**Uninstall:** `ShopifyUninstallService` is unchanged. The "live" rule keys off
`soft_deleted_at`, so a token minted in the ≤ 60 s before an uninstall resolves
to `installation_required`. The exchange then fails at Shopify (the app is no
longer installed) and the admin shows the error state. No tenant data is served.

**Legacy endpoints in Phase 1:** `/oauth/shopify/install|callback|success` stay
live but are no longer reached once WID-1 changes `application_url`. Phase 2
deletes them.

### 3.4 Merchant contact email (decision 5)

- **Prerequisite:** architecture.md §8 must list the columns first (backend rule:
  "Every column type comes from architecture.md §8, no invention"). The spec
  needs an onboarding AC (PO-4). Owners: `product-analyst`, and the PO for the
  go-live gating rule.
- Flyway **`V7__tenants_contact_email.sql`** on `tenants` (the tenant row itself,
  PK `tenant_id`):
  - `contact_email VARCHAR(254) NULL`;
  - `contact_email_confirmed_at TIMESTAMPTZ NULL`.
  - The entity uses `Instant`, following the tenant module.
- `tenant` module:
  - `GET /admin/v1/tenant/contact` → `{ contactEmail, confirmedAt }` (query);
  - `PUT /admin/v1/tenant/contact` `{ contactEmail }` → `204` (command).
    Validation: `@Email`, ≤ 254 chars. It sets `confirmed_at = now`.
  - The tenant comes from the principal only.
- `platform-integrations`: `GET /admin/v1/platform/shop-contact-suggestion`
  → `{ suggestedEmail }`. It is read live from Shopify `shop.json` `email` and
  **never persisted** (data minimisation: we store only what the merchant
  confirms).
- `/me` adds `tenant.contactEmailConfirmed: boolean`, which the wizard uses to
  resume at the right step.
- Notice wording: **ATTORNEY**. The field stores the address; the notice text is
  not engineering's call.

### 3.5 Secret handling and rotation with a grace window

- New optional property `wisegift.shopify.client-secret-previous`
  (`WISEGIFT_SHOPIFY_CLIENT_SECRET_PREVIOUS`, default empty), held in a small
  `ShopifySigningKeys` component:
  - `verificationKeys()` = [current, previous?];
  - `oauthSecret()` = current.
- **ID-token verification** accepts either key (S2 note 4: which secret signs
  ID tokens during rotation is UNVERIFIED).
- **Webhook HMAC** accepts either key (BE-5). Shopify signs webhooks with the
  **oldest unrevoked** secret during rotation (VERIFIED). The separate
  `webhook-secret` property stays in Phase 1, to avoid a silent config change.
  Shopify's docs confirm it equals the client secret, so Phase 2 can fold it in.
- **Outbound OAuth** (exchange, and refresh later) uses only the current secret
  (manage-credentials step 3).
- **Rotation procedure (devops, PO-gated, runbook O3):**
  1. Generate a new secret in the Dev Dashboard.
  2. Set `…_PREVIOUS` = old and `…_CLIENT_SECRET` = new, then redeploy.
  3. **Re-pin stored tokens.** Shopify's step 4: access tokens stay pinned to the
     secret that minted them.
     - For non-expiring tokens, that means the Dev Dashboard "rotation refresh
       token" re-pin call per store, within a 1 h window.
     - Our install path only exchanges when no live credential exists, so
       "coverage follows merchants opening your app" does **not** apply to us.
     - A small PO-run re-pin task is needed before the first rotation. It is
       not part of Phase 1 code (§8, risk R6).
  4. Revoke the old secret.
  5. Clear `…_PREVIOUS`.
- Boot check: the current secret is non-blank and ≥ 32 bytes; the previous
  secret is blank or ≥ 32 bytes.
- Never logged; backend-only; it never reaches the Next.js app. The admin keeps
  only the public `client_id`.

### 3.6 Error contract for the admin client

All bodies use the existing `ErrorResponse { code, message }`.

| Situation | Status | Headers | `code` |
|---|---|---|---|
| Missing, malformed, expired or invalid bearer; any verifier failure | 401 | `X-Shopify-Retry-Invalid-Session-Request: 1` | `invalid_session_token` |
| Token Exchange returned 400 (stale ID token) | 401 | same | `invalid_session_token` |
| Valid token, no live credential (first install, or reinstall) | 409 | | `installation_required` |
| Exchange 5xx, timeout or 429 | 503 | `Retry-After: 5` | `platform_unavailable` |
| Exchange other 4xx (client credentials wrong), or unexpected `expires_in` | 502 | | `installation_failed` (error log, alerts devops) |
| Authenticated, but forbidden | 403 | | `forbidden` |

Plus `Cache-Control: no-store` on every `/admin/v1/**` response, and never a
`Set-Cookie` on the bearer chain.

**`/me` shape (B3).** It stays additive during Phase 1:

```jsonc
{
  "admin": null,                         // legacy cookie path only; removed in Phase 2
  "shop": { "platform": "SHOPIFY", "domain": "x.myshopify.com" },
  "tenant": { "id", "region", "vertical", "plan", "active", "consentModeRequired",
              "contactEmailConfirmed" },
  "installation": { "missingScopes": [] }
}
```

`shop.domain` also closes the "contract gap" noted in the admin's
`use-shop-domain.ts`.

### 3.7 Backend tests (test-first; at least two tenants on every read path)

- **`ShopifyIdTokenVerifierTest`** (unit, fixed `Clock`). Uses tokens minted in
  the test with a known secret, plus **one recorded real staging token header**
  as a contract fixture (R3). Cases:
  - valid;
  - expired, `nbf` future, inside and outside the leeway;
  - `alg` `none`/`RS256`/`HS512`;
  - bad signature;
  - previous key accepted and unknown key rejected;
  - wrong `aud`;
  - `dest` not myshopify, `dest` with a path, `iss`/`dest` mismatch;
  - lifetime > 5 min;
  - missing `sub` accepted.
- **`AdminBearerChainIT`** (MockMvc, Testcontainers, tenants A and B):
  - a token for A with `?shop=B`, body `shop:B` and `X-Shopify-Shop-Domain: B`
    resolves to A on **every** admin endpoint (widget config GET/PUT, theme
    tokens, placements, catalog sync-status, sync triggers, `/me`, contact);
  - a token for C (unknown) → 409 on data endpoints, and `POST /installation`
    is allowed;
  - soft-deleted tenant → 409, with no data in the body;
  - invalid token → 401 + retry header, and the body has no detail;
  - a valid `WG-ADMIN-SESSION` cookie for B + an invalid bearer → 401 (never
    falls back to the cookie);
  - a valid cookie for B + a bearer for A → A;
  - no `Set-Cookie` on any bearer response;
  - non-GET without an XSRF header succeeds (CSRF off);
  - `Authorization: Basic …` → 401;
  - legacy auth endpoints with a bearer → 403/401;
  - no-bearer requests still hit the legacy chain (existing ITs stay green).
- **`EmbeddedInstallServiceIT`** (WireMock Shopify, Testcontainers):
  - new install creates the tenant, credential (encrypted), widget_config and
    the event after commit;
  - **two concurrent installs → exactly one tenant, one exchange call**;
  - repeat call → no exchange;
  - reinstall after soft delete reactivates with `active=false` and keeps
    widget_config;
  - exchange 400 → 401 + header;
  - 5xx → 503;
  - `expires_in` present → 502 and nothing persisted;
  - partial failure after the exchange → rollback with no orphan tenant;
  - webhook registration failure does not roll back the install;
  - `custom_domain` comes from `domain` (F-A).
- **`ShopifyTokenExchangeClientContractTest`**: request shape (form fields,
  content type) against a recorded fixture of Shopify's documented exchange
  response.
- **`ShopifyHmacVerifierTest`**: accepts the previous secret, and rejects when
  no previous secret is set (BE-5).
- **`TenantContactIT`**: A cannot read or write B's contact; validation;
  `confirmed_at` is set.

---

## 4. Admin changes (`wisegift-admin`, PRs target `main`)

- **`client.ts`:**
  - Before each request: `const token = await window.shopify.idToken()`, then
    set `Authorization: Bearer ${token}`. This is explicit because the App Bridge
    interceptor only covers the `application_url` host, which is still the
    backend until WID-1 (S3). The Shopify token-exchange tutorial uses this
    same explicit pattern.
  - `credentials: "omit"`.
  - Remove the XSRF read/echo.
  - On `401` with `X-Shopify-Retry-Invalid-Session-Request`, get a fresh token
    and retry **once**. If App Bridge's interceptor already retried, the second
    attempt is still bounded.
  - Map `409 installation_required` to an `InstallationRequiredError`.
- **Installation gate** (App Store 2.3.2/2.3.4):
  1. Probe `GET /admin/v1/auth/me`.
  2. On 409, render a non-interactive "Setting up WiseGift…" state and call
     `POST /admin/v1/installation`. Retry with backoff on 503; on 502 show an
     error with a support contact.
  3. Then re-probe `/me`.
  4. No navigation or interactive UI renders before step 3 succeeds.
  - Onboarding routing uses `tenant.active` and `tenant.contactEmailConfirmed`.
- **Not embedded:** if `window.shopify` is absent or `idToken()` rejects (for
  example an ad blocker; Shopify's docs flag ad blockers), show "Open WiseGift
  from your Shopify admin". No data, no login form.
- **Session provider:** remove `login`/`signup`/`logout` from the context. The
  `/login` and `/signup` route files are deleted in Phase 2, but are unreachable
  from Phase 1 routing.
- **Headers (ADM-2), via Next middleware on every HTML route:**
  - `Content-Security-Policy: frame-ancestors https://{shop} https://admin.shopify.com;`
    - `shop` comes from the query, validated against the myshopify regex;
    - if `shop` is missing or invalid, use `frame-ancestors https://admin.shopify.com`;
    - this covers S5 / F3.
  - `Referrer-Policy: no-referrer` site-wide (S1: `id_token` sits in the URL).
- **Contact step (ADM-3):** an onboarding step pre-filled from
  `shop-contact-suggestion`, editable, saved with PUT. The UX is
  `product-analyst`'s; the AC is pending (PO-4).
- **No `localStorage`/`sessionStorage`** for anything auth- or state-critical
  (App Store 1.1.1).
- **Proxy (F2):** confirm on staging, in devtools and backend logs, that the
  Next `rewrites()` forwards `Authorization` unchanged and that nothing logs it.

## 5. Widget repo TOML (`wisegift-widget/shopify.app.toml`, PR targets `main`)

- `application_url = "https://<admin-host>/"`, the staging admin (normally
  `https://wisegift-admin-staging.onrender.com/`; copy the real value from Render).
- Keep `embedded = true` and `use_legacy_install_flow = false` (Shopify-managed
  install).
- `[auth] redirect_urls`: keep the backend callback until Phase 2.
- Optional cosmetic fix: `name = "WiseGift Staging"` (the leading space).
- **Not in Phase 1, but listed so it isn't lost:**
  - compliance webhooks (`[webhooks.subscriptions] compliance_topics`), owned by M3·2;
  - moving topic subscriptions from per-shop REST registration into the TOML
    (app-specific subscriptions). That would remove install-time registration
    and the redirect-URI coupling (C12). Separate decision, separate PR.
- **Release:** a widget build first (2026-10-14 finding 4), then
  `shopify app deploy`. Both are **PO-gated**. Rollback: redeploy the previous
  app version from the Dev Dashboard.

## 6. DevOps (`devops-expert`)

- **Backend Render env:** add `WISEGIFT_SHOPIFY_CLIENT_SECRET_PREVIOUS`
  (`sync: false`, empty). Nothing else is new in Phase 1, and all existing
  variables stay until Phase 2.
- **Admin Render:** no new env (`NEXT_PUBLIC_SHOPIFY_API_KEY` already exists).
  Switch it from **free to starter before the first pilot** (already decided).
  The 30–60 s cold start is fine for the E2E test, because App Bridge mints
  fresh tokens per request, but it fails BFS LCP and is poor merchant UX.
- **Logs (O2):**
  - confirm the backend never logs `Authorization`;
  - confirm Render's admin service logs no request URLs (`id_token` query);
  - confirm Next has no request logging.
- **Runbooks:**
  - update `admin-staging-deploy.md` (smoke 4.2–4.4 and step 7 describe
    cookie/signup) and `render-staging-env-vars.md` (new optional var);
  - add the client-secret rotation runbook (§3.5), including the re-pin task.

---

## 7. PR breakdown (dependency order)

Each PR carries its own tests. The backend merges to `develop`; staging deploys
from `develop` manually (`autoDeploy: false`). Admin and widget merge to
`main`, and admin `main` auto-deploys to staging. **[DO NOT MERGE]** applies
while a deploy prerequisite is open (decisions.md 2026-10-15).

| # | Repo → base | Branch | Content | Depends on | Title prefix |
|---|---|---|---|---|---|
| **BE-1** | backend → `develop` | `feature/admin-platform-identity` | `shared.admin.AdminPrincipal`; `PlatformIdentityVerifier` port + `ShopifyIdTokenVerifier` (Nimbus, dual key); `PlatformAdminAuthenticator`; `@Order(1)` bearer chain + filter + error writer; `AdminTenantResolver` and the two controllers switched to it; `/me` additive `shop` field; `ShopifySigningKeys`; tests from §3.7 (verifier, bearer chain). No install yet: an unknown shop → 409 | — | none (no env needed; `_PREVIOUS` defaults to empty) |
| **BE-2** | backend → `develop` | `feature/admin-token-exchange-install` | `POST /admin/v1/installation`; `ShopifyTokenExchangeClient` (timeouts, `expiring=0`); `InstallationLock` (Postgres advisory); `ShopifyTenantProvisioner` extraction; reactivation fix (C13); `custom_domain` from `domain` (F-A); `/me` `installation`; ITs | BE-1 | none |
| **BE-3** | backend → `develop` | `feature/tenant-contact-email` | V7 migration; contact GET/PUT; shop-contact suggestion; `/me` `contactEmailConfirmed`; ITs | BE-1; architecture.md §8 + spec AC (PO-4) | none |
| **BE-5** | backend → `develop` | `feature/webhook-hmac-dual-secret` | Webhook HMAC accepts the previous secret; tests | BE-1 (`ShopifySigningKeys`) | **[DO NOT MERGE]** until `WISEGIFT_SHOPIFY_CLIENT_SECRET_PREVIOUS` is declared on Render (empty is fine) |
| **BE-6** *(optional)* | backend → `develop` | `fix/widget-cors-no-credentials` | `allowCredentials=false` on `/widget/v1/**` (C15); test | — | none |
| **ADM-1** | admin → `main` | `feature/app-bridge-bearer-auth` | Bearer client + single retry; installation gate; not-embedded page; remove XSRF and `credentials:"include"`; session provider without login/signup; types for the new `/me`; Vitest | BE-1 + BE-2 **deployed to backend staging** | **[DO NOT MERGE]** until BE-1 and BE-2 are live on staging (admin `main` auto-deploys and would otherwise strand the staging admin) |
| **ADM-2** | admin → `main` | `feature/per-shop-frame-ancestors` | Next middleware: per-shop `frame-ancestors`; site-wide `Referrer-Policy: no-referrer` (replaces the `/signup` rule); tests | — | none |
| **ADM-3** | admin → `main` | `feature/onboarding-contact-email` | Contact step in the wizard | BE-3 deployed; ADM-1 | **[DO NOT MERGE]** until BE-3 is live on staging |
| **WID-1** | widget → `main` | `chore/application-url-admin` | `application_url` → admin host; (name trim) | BE-1, BE-2, ADM-1, ADM-2 live on staging | **[DO NOT MERGE]** until those are live; then widget build + `shopify app deploy` (**PO**) |
| **OPS-1** | hq (docs; PO commits) | — | Runbook updates (§6), rotation runbook | WID-1 | n/a |
| **E2E** | PO + backend-engineer | — | §10 script | everything above | n/a |

BE-3 and ADM-3 are not needed for the E2E install test, but they are needed
before the first pilot (DPA breach-notice contact).

---

## 8. Phase 2 cleanup list (before the first pilot install)

**Remove:**

- **Backend chain:**
  - the legacy cookie chain's `/admin/**` rules;
  - the bearer chain's `Authorization`-present condition, so the path alone matches;
  - `AdminCsrfCookieFilter`, `SpaCsrfTokenRequestHandler`,
    `CsrfTokenRotationStrategy`, the `adminCsrfTokenRepository` bean, and the
    `adminSessionAuthenticationStrategy` bean.
- **Backend auth:**
  - `AdminAuthController` login/signup/google/logout endpoints and DTOs, and
    the `admin` field of `/me`;
  - legacy `tenant…AdminPrincipal`, `AdminSessionSerializationTest`, and the
    `AdminTenantResolver` legacy branch;
  - `AdminAuthenticationService`, `AdminAuthPort`, `AdminUserRepository` and its
    adapter, `AdminUser`, `AdminUserId`, `AuthResult`, `AdminPasswordPolicy`,
    `MaxUtf8Bytes`, and the auth exceptions;
  - the `PasswordEncoder` bean (`TenantConfig`).
- **Google pair:** `GoogleIdTokenVerifier`, its adapter,
  `NullGoogleIdTokenVerifier` and `GoogleOAuthEnabledCondition`, plus
  `tenant`'s `spring-boot-starter-oauth2-resource-server`. Nimbus is declared
  directly in BE-1, so this removal is safe. Also remove the unused
  `spring-boot-starter-oauth2-client` from `app`.
- **Invite:** `InviteTokenService`, `InviteToken`, `ConsumedTokenStore`,
  `ConsumedInviteTokenStore`, `AdminSignupLinkService`, `AdminSignupUrl`,
  `AdminSignupUrlResolver`, the signup URL in `AdminProperties`, and
  `AdminSignupPageController` (plus its `permitAll`).
- **Legacy OAuth:**
  - `ShopifyOAuthController` (`/install`, `/callback`, `/success`),
    `OAuthStateStore`, `ShopifyInstallService.buildAuthorizationUrl`/`completeInstall`,
    and `exchangeCodeForToken`;
  - the TOML `[auth] redirect_urls` (WID-2, PO deploy).
  - **First** add `wisegift.shopify.webhook-base-url` and switch
    `deriveWebhookBase` to it (C12). Only then remove `WISEGIFT_SHOPIFY_REDIRECT_URI`.
- **Sessions:** `SessionConfig` (`@EnableRedisHttpSession`, cookie serializers,
  `AbsoluteSessionExpiryFilter`), `spring-session-data-redis` (`app` and
  `catalog` test scope), and `server.servlet.session.timeout`. Nothing else
  uses HTTP sessions (C18).
- **Env (Render, PO):** `WISEGIFT_ADMIN_INVITE_TOKEN_SECRET`,
  `WISEGIFT_ADMIN_SIGNUP_URL`, `WISEGIFT_GOOGLE_CLIENT_ID`,
  `WISEGIFT_GOOGLE_CLIENT_SECRET`, `WISEGIFT_SHOPIFY_REDIRECT_URI` (after the
  webhook base property exists). Optionally fold `WISEGIFT_SHOPIFY_WEBHOOK_SECRET`
  into the client secret.
- **Data:** delete the staging `admin_users` rows (Phase 2) and flush
  `wg:spring:session:*`, `wg:oauth:state:*` and `wg:admin:invite:consumed:*`.
  A Flyway `DROP TABLE admin_users` follows in Phase 3.
- **Admin:** the `/login` and `/signup` routes, the `components/signup/*`
  components, and the `lib/auth` invite/login/signup modules and tests.

**Keep:**

- the bearer chain and the platform-identity layer;
- `ShopifyHmacVerifier` (dual-key) and `OAuthTokenCipher`;
- `ShopifyTenantProvisioner` and widget-config seeding;
- the webhook idempotency store;
- Redis, which still serves the response cache, caps, kill switch, rate
  limiter, webhook dedupe and widget session rate limits. It just no longer
  holds sessions;
- the `/admin/v1` same-origin proxy;
- the App Bridge provider;
- the 401 entry point;
- per-shop `frame-ancestors`.

**Pre-App-Store (not Phase 2, listed for visibility):**
- GraphQL migration (F-B);
- expiring offline tokens plus refresh under the same advisory lock, using the
  documented no-session migration for existing tokens (PO-1);
- compliance webhooks;
- the re-pin task for secret rotation.

---

## 9. Decisions the PO must take or revisit

- **PO-1. Expiring offline tokens: now or with the GraphQL migration?**
  - Facts: custom apps (pilots) are exempt. The public app must use expiring
    tokens for GraphQL and must use GraphQL only. Moving later is a documented
    no-user-session call per store.
  - **Recommendation: defer.** Phase 1 sends `expiring=0` and fails loudly if
    Shopify returns an expiring token. Refresh is built with the GraphQL
    migration before App Store submission.
  - Building refresh now costs +2–3 dev-days.
  - **The PO should also confirm in the Dev Dashboard which distribution the
    staging app has** (custom or public). The answer decides whether `expiring=0`
    is accepted at all.
- **PO-2. `/me` contract break (B3).** The locked "additive-only"
  `admin { id, email, createdAt }` disappears in Phase 2. The shape in §3.6 is
  recommended. The admin and backend ship together, so no external consumer
  breaks.
- **PO-3. Reinstall semantics.** A reinstall within 30 days reactivates the
  tenant with `active = false` and keeps the configuration. The merchant
  re-places the blocks and goes live again. The recommendation is this option,
  because Shopify removes theme-extension blocks on uninstall.
- **PO-4. Contact email AC.** Is a confirmed contact email required before
  "Go live"? The recommendation is yes, because the DPA breach-notice duty
  needs it.
  - `product-analyst` adds the AC and architecture.md §8 columns.
  - **ATTORNEY** words the notice.
- **PO-5. Decision 1 rationale wording.** Safari now supports CHIPS (26.2+).
  This changes nothing in the decision, because App Store 1.1.1 and Shopify's
  docs require ID-token auth for embedded apps. The PO may want the log's
  rationale to cite that requirement rather than Safari (append-only note).
- **PO-6.** The install lock uses Postgres, not Redis as the proposal said
  (F-D). No decision is needed; this is for awareness.

---

## 10. Fresh-install end-to-end test (end of Phase 1; PO runs with backend-engineer)

**Preconditions:**
- BE-1, BE-2, BE-5, ADM-1, ADM-2 and WID-1 are live on staging, and the PO has
  run `shopify app deploy`;
- a **new** dev store with no WiseGift data;
- the backend log tail is open;
- catalog fixture: a few products only (OpenAI cost).

1. **Install.** From the Dev Dashboard / Partner "Test on development store",
   install on the new store.
   - Expect: the scope approval screen lists exactly `read_inventory`,
     `read_orders`, `read_products`.
   - After approval, the app opens **inside** the Shopify admin iframe and shows
     "Setting up WiseGift…". There is **no** signup or login screen.
2. **Backend.**
   - Exactly one `POST /admin/v1/installation` performs one token exchange (one
     log line).
   - One `tenants` row exists (`active=false`, `soft_deleted_at` null) and one
     `platform_credentials` row (token encrypted, scopes match, `custom_domain`
     = the store's domain).
   - One `widget_config` row exists.
   - 7 webhook registrations, logged per topic.
   - The initial sync starts.
   - No `admin_users` row. No `Set-Cookie` on any `/admin/v1` response
     (devtools).
3. **Onboarding.** The wizard renders. The contact step (if ADM-3 is live) is
   pre-filled with the shop email. Save it and confirm `contact_email_confirmed_at`
   is set.
4. **Reopen** the app 3× (close the tab, reopen from Apps). Expect: no further
   token exchange, no new tenant, no webhook re-registration, no new sync.
   `/me` returns 200 directly.
5. **Browser matrix.** Open the app in Chrome, **Chrome Incognito**, Safari
   (record the version) and Firefox. Each one loads with no login and no
   cookie/CSRF errors. Make one config save (PUT widget config) per browser:
   200, no `X-XSRF-TOKEN` sent, and the `Authorization` header is present (devtools).
6. **Token expiry.** Leave the app idle for at least 2 min, then save the config.
   Expect: success. Either the token was fresh, or there was one
   `401 + X-Shopify-Retry-Invalid-Session-Request` followed by a successful retry.
7. **Isolation spot-check.** With store A open, run devtools
   `fetch('/admin/v1/widget/config?shop=<other-store>.myshopify.com')`.
   Expect: store A's config only.
8. **Not embedded.** Open `https://<admin-host>/` directly in a new tab. Expect
   the "Open WiseGift from your Shopify admin" page, with no data.
9. **Framing.** The response header on the app HTML is
   `frame-ancestors https://<store>.myshopify.com https://admin.shopify.com`.
10. **Uninstall.** Expect the `app/uninstalled` webhook: the tenant is
    soft-deleted and inactive.
11. **Reinstall (same store, same day).** Expect: one exchange; the **same**
    `tenant_id` is reactivated (`soft_deleted_at` null, `active=false`);
    `widget_config` is preserved; webhooks are re-registered; a resync runs; the
    wizard is shown.
12. **Cleanup.** The PO decides whether to keep the store. Record the results
    in decisions.md (PO).

**Pass** = every expectation above holds. Any iframe CSRF/401 failure is a
defect at this point, not an expected result.

---

## 11. Effort estimate (single engineer, including tests)

| Area | Phase 1 | Notes |
|---|---|---|
| Backend: BE-1 identity layer + chain + resolver | 2.5–3 d | Test matrix across every admin endpoint |
| Backend: BE-2 install via Token Exchange | 2–2.5 d | Lock, transactional provisioning, reactivation fix, F-A, WireMock ITs |
| Backend: BE-3 contact email | 1 d | Blocked on the spec AC + architecture.md §8 |
| Backend: BE-5 (+ optional BE-6) | 0.5 d | |
| **Backend subtotal** | **6–7 d** | Proposal said 4–5. The increase covers the reactivation and transaction fixes, the coexistence resolver and the contact email, which the proposal did not count |
| Admin: ADM-1, ADM-2, ADM-3 | 3–4 d | Proposal said 2–3; the installation gate and contact step are added |
| Widget TOML: WID-1 | 0.25 d | Plus the PO deploy |
| DevOps: env, logs check, runbooks, rotation runbook | 1 d | |
| E2E test (§10) | 0.5 d | PO + backend-engineer |
| **Phase 1 total** | **≈ 11–13 dev-days** | Proposal said 7–9. **+2–3 d** if PO-1 chooses expiring tokens now |
| Phase 2 (removal across repos, env, data) | 2.5–3 d | Includes the webhook-base-url property |
| Phase 3 (Flyway drop `admin_users`) | 0.25 d | |

## 12. Risks

- **R1. Expiring tokens / distribution type (PO-1).** If the staging app is
  public-distribution and Shopify refuses `expiring=0`, BE-2 fails loudly at
  install. Refresh would then become Phase 1 scope (+2–3 d).
- **R2. REST `2024-10`** is past support and is served by the oldest supported
  version. Response shapes may drift. This is not a Phase 1 change; the bump
  is already a logged follow-up.
- **R3. ID token `alg`.** The docs are inconsistent: an RS256-looking example
  token on the App Bridge ID Token API page. Pinning HS256 is correct per the
  id-tokens page and the official library. Confirm with one real staging token
  before BE-1 merges. If it ever differs, the verifier fails closed: every
  admin call returns 401, which is safe but an outage.
- **R4. Admin on the Render free plan.** Cold starts take 30–60 s. App Bridge
  survives this (fresh tokens per request), but the merchant UX and BFS LCP do
  not. Move to starter before the first pilot (already decided).
- **R5. Ad blockers** can stop App Bridge from issuing a token (Shopify docs).
  The not-embedded page must be explicit, and support docs should mention it.
- **R6. Secret rotation is not a one-step operation.** Stored non-expiring
  tokens must be re-pinned per store within 1 h windows, before the old secret
  is revoked. Until the re-pin task exists, **an emergency rotation means
  merchants lose catalog sync until they reopen the app**. Even then, the
  current "live credential" rule would not re-exchange. Mitigation: build the
  re-pin task before the first pilot, or add a "credential invalid on Shopify
  401" flag that forces a re-exchange on next open (small; can ride with BE-2
  if the PO wants it).
- **R7. Exchange inside a DB transaction** holds a pooled Neon connection for
  up to about 15 s per install. This is fine at pilot scale; revisit if bulk
  installs happen.
- **R8. Phase 1 is temporarily Option C** on staging: a cookie chain and a
  bearer chain on one surface. The mitigation is the strict matcher and the
  "bearer never falls through" tests. Phase 2 should follow Phase 1 closely.
- **R9. The legacy callback stays live in Phase 1** (optional HMAC on
  `/install`, C14). After WID-1 nothing routes to it. Delete it in Phase 2 on
  schedule.
- **R10. Missing compliance webhooks** block App Store submission (M3·2). They
  are not a pilot blocker.
