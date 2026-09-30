# Runbook — Render staging environment variables

**Purpose.** Provision every environment variable the `wisegift-backend`
staging Spring Boot app needs to boot. PR 2 (Shopify OAuth install flow)
adds a hard contract: missing any of five Shopify-related vars causes
Spring Boot to fail loudly at startup per `docs/decisions.md` 2026-09-30
"Backend PR 2 plan" — Decision 4. This runbook enumerates the full set,
documents where each value comes from, and describes setting them on
Render.

**Blocks.** `wisegift-backend` PR 2 merge. Cannot merge until every
variable in the inventory below is present on the Render staging service.

**Owner.** `devops-expert`. Requires product-owner approval to execute —
this runbook is informational.

**Cross-refs.**
- `docs/engineering/architecture.md` §Environments — staging URLs.
- `docs/engineering/architecture.md` §11 Deployment topology — "Render EU
  hosting", secrets-manager posture.
- `docs/decisions.md` 2026-09-30 "Backend PR 2 plan" — the five Shopify env
  vars, the `WISEGIFT_ENCRYPTION_KEY` contract, and the Shopify Partner
  dashboard config requirement.
- `docs/decisions.md` 2026-09-30 "Backend PR 1 merged" — outstanding
  follow-up: 5 new Shopify env vars provisioned on Render staging.
- `docs/decisions.md` 2026-09-29 "Frozen MVP scope" — EU-only region.
- `docs/decisions.md` 2026-09-29 "Architecture open questions closed" —
  cost telemetry lands in Postgres at MVP (no separate observability
  connection string required), Redis provider TBD.
- `neon-staging-provisioning.md` (sibling runbook) — source of the
  Postgres URL / user / password values.

---

## Prerequisites

1. **Render account** with access to the WiseGift org and permission to
   edit the staging service's environment. If the staging service does
   not yet exist, this runbook cannot complete — see gap #1 in
   `README.md` (Render staging service existence).
2. **Neon staging DB is provisioned and `vector` extension enabled.**
   See `neon-staging-provisioning.md`. This runbook consumes the JDBC
   URL, username, and password produced by that runbook. Do not run this
   runbook first.
3. **Shopify Partner dashboard access** with an app already registered
   as the WiseGift dev / staging app. Client ID, client secret, and
   webhook shared secret come from that app's config. If the app has
   not been registered yet, register it first (Partner dashboard →
   Apps → Create app → Custom app; set `App URL` to
   `https://api-staging.wisegift.app/oauth/shopify/install` and
   `Allowed redirection URLs` to
   `https://api-staging.wisegift.app/oauth/shopify/callback`).
4. **`render` CLI** installed and authenticated (`brew install render`,
   `render login`). CLI is preferred over dashboard clicks for diffable
   steps.
5. **`openssl`** installed locally (macOS + most Linux distros ship it).
   Used once to generate `WISEGIFT_ENCRYPTION_KEY`.
6. **`api-staging.wisegift.app` DNS + TLS wired to the Render service.**
   Required for the Shopify OAuth callback URL to actually resolve when
   PR 2's post-merge smoke test runs. If not wired, flag as gap #2 in
   `README.md` — this runbook cannot fix it, but it can proceed setting
   env vars (Spring Boot will boot happily; only the Shopify callback
   fails at smoke-test time).

---

## Environment variable inventory

Every variable the staging Spring Boot app expects at boot. Sourced from
`wisegift-backend/app/src/main/resources/application.yml` +
`application-staging.yml` + the PR 2 plan's `wisegift.shopify.*` config
addition per `docs/decisions.md` 2026-09-30 "Backend PR 2 plan" —
Decision 4.

| Variable | Value source | Required at boot | Secret |
|---|---|---|---|
| `SPRING_PROFILES_ACTIVE` | Literal string `staging` | Yes — selects `application-staging.yml` | No |
| `SPRING_DATASOURCE_URL` | `neon-staging-provisioning.md` step 6 (JDBC form) | Yes — no default in `application-staging.yml` | No (URL is not secret; password is separate) |
| `SPRING_DATASOURCE_USERNAME` | `neon-staging-provisioning.md` step 6 (`neondb_owner`) | Yes | No |
| `SPRING_DATASOURCE_PASSWORD` | `neon-staging-provisioning.md` step 6 | Yes | **Yes** |
| `SPRING_DATA_REDIS_URL` | Managed Redis EU vendor (TBD — see `README.md` gap #3). Staging fallback: Render-managed Redis add-on connection string | Yes for PR 2+ (OAuth CSRF nonce + webhook dedupe). Not consumed by PR 1's tenant module | **Yes** (contains password) |
| `WISEGIFT_ENCRYPTION_KEY` | Generated locally via `openssl` (step 1 below); 32 bytes, base64-encoded, AES-256-GCM key for the `OAuthTokenCipher` | Yes for PR 2+. Boot fails loud if unset per `docs/decisions.md` 2026-09-30 "Backend PR 2 plan" | **Yes** — never regenerate without a re-encrypt-all migration |
| `WISEGIFT_SHOPIFY_CLIENT_ID` | Shopify Partner dashboard → Apps → WiseGift staging app → App setup → Client ID | Yes for PR 2+ | No (client ID is public in OAuth URLs) |
| `WISEGIFT_SHOPIFY_CLIENT_SECRET` | Shopify Partner dashboard → App setup → Client secret | Yes for PR 2+ | **Yes** |
| `WISEGIFT_SHOPIFY_WEBHOOK_SECRET` | Shopify Partner dashboard → App setup → Client secret (**same value** as `CLIENT_SECRET` for Shopify custom apps at time of writing — Shopify uses the client secret as the webhook HMAC shared secret. Public apps that register webhooks via the API may receive a separate `webhook_secret` in the response; document explicitly if that differs). | Yes for PR 2+ | **Yes** |
| `WISEGIFT_SHOPIFY_REDIRECT_URI` | Literal string `https://api-staging.wisegift.app/oauth/shopify/callback` — must exactly match the value registered under Allowed redirection URLs in the Shopify Partner dashboard for this app | Yes for PR 2+ | No |
| `JAVA_TOOL_OPTIONS` | Optional. `-XX:MaxRAMPercentage=75.0` for tuning JVM heap to Render's dyno memory. Leave unset unless the Render instance runs into OOM. | No | No |

**Java runtime.** Java 21 per `wisegift-backend/pom.xml` (`<java.version>21</java.version>`).
Configured on the Render service itself (Dashboard → Settings → Runtime →
Docker or native Java runtime with `JAVA_VERSION=21`), not as an env var
consumed by Spring. Keep this in mind when creating the Render service if
gap #1 in `README.md` is being resolved in parallel.

---

## Steps

### 1. Generate `WISEGIFT_ENCRYPTION_KEY`

Run **once**, locally, on the operator's machine. The output is the value
paste into Render.

```
openssl rand -base64 32
```

**Expect on success.** A 44-character base64 string (32 raw bytes encoded
as base64 with padding), for example:

```
9NwK7fVBGYqRLMt5+Ux3PZaJ2sE4hDIcFV0nOWpTkQU=
```

Do not commit this value anywhere. Do not paste it into Slack or email.
Paste it directly into Render in step 4.

**Rotation contract (per `docs/decisions.md` 2026-09-30 "Backend PR 2 plan"
— Consequences):** this key must NEVER be regenerated once the app has
written any `platform_credentials.oauth_access_token_encrypted` row. The
`OAuthTokenCipher` needs the original key to decrypt existing tokens; a
regen breaks every merchant install. A future rotation requires a
re-encrypt-all migration (documented in the `OAuthTokenCipher` Javadoc per
the PR 2 plan). If rotation becomes necessary, coordinate with
`security-and-privacy` — never do it unilaterally.

### 2. Collect the Neon connection string values

From `neon-staging-provisioning.md` step 6. Three values:

- JDBC URL: `jdbc:postgresql://ep-<compute-id>.eu-central-1.aws.neon.tech/neondb?sslmode=require`
- Username: `neondb_owner`
- Password: `<from Neon connection string>`

If the sibling runbook has not been executed yet, stop here — this
runbook cannot complete without those values. Do not synthesise
placeholder values just to unblock step 4; a boot failure with a
credentials error costs more debugging time than pausing here.

### 3. Collect the five Shopify values

From the Shopify Partner dashboard:

1. **Client ID** — Apps → WiseGift staging app → App setup → Client
   credentials → Client ID. Copy verbatim.
2. **Client secret** — same page, Client secret. **Reveal** requires a
   dashboard permission; if the current logged-in Partner account
   cannot reveal it, escalate to whoever registered the app. Do not
   regenerate — regenerating rotates it and invalidates every existing
   install. Copy verbatim.
3. **Webhook shared secret** — for Shopify custom apps registered from
   the Partner dashboard, this is the **same value** as the Client
   secret; Shopify uses the client secret to HMAC-sign webhook bodies.
   Confirmation lives in Shopify's own docs; PR 2's `ShopifyHmacVerifier`
   assumes this equality per `docs/decisions.md` 2026-09-30 "Backend PR
   2 plan" — Decision 7. If a future migration to a public App Store
   app introduces a distinct webhook secret (returned in the
   webhook-registration API response), this runbook must be updated to
   split the two values.
4. **Redirect URI** — literal string
   `https://api-staging.wisegift.app/oauth/shopify/callback`. **Also
   register this in the Shopify Partner dashboard** under App setup →
   URLs → Allowed redirection URLs. Shopify validates the callback URL
   exactly; a mismatch (missing trailing slash, http vs https, wrong
   subdomain) fails the OAuth flow with an opaque error at Shopify's
   end.
5. **App URL** — while in the Partner dashboard, also set App URL to
   `https://api-staging.wisegift.app/oauth/shopify/install`. Not
   exposed to Spring Boot as an env var (PR 2 does not read it), but
   required for Shopify to launch the install flow correctly.

### 4. Set the variables on Render

CLI form (preferred):

```
render env set \
  --service <staging-service-id> \
  SPRING_PROFILES_ACTIVE=staging \
  SPRING_DATASOURCE_URL='jdbc:postgresql://ep-<compute-id>.eu-central-1.aws.neon.tech/neondb?sslmode=require' \
  SPRING_DATASOURCE_USERNAME='neondb_owner' \
  SPRING_DATASOURCE_PASSWORD='<neon-password>' \
  SPRING_DATA_REDIS_URL='<redis-vendor-url-or-render-managed>' \
  WISEGIFT_ENCRYPTION_KEY='<base64-key-from-step-1>' \
  WISEGIFT_SHOPIFY_CLIENT_ID='<shopify-client-id>' \
  WISEGIFT_SHOPIFY_CLIENT_SECRET='<shopify-client-secret>' \
  WISEGIFT_SHOPIFY_WEBHOOK_SECRET='<shopify-client-secret>' \
  WISEGIFT_SHOPIFY_REDIRECT_URI='https://api-staging.wisegift.app/oauth/shopify/callback'
```

**Expect on success.** Render prints one confirmation line per variable
set (`env var SPRING_PROFILES_ACTIVE set on service …`) and queues a
deploy. Depending on Render account settings, the deploy is auto or
manual — do not let it auto-deploy before you have verified step 5.

Dashboard form (fallback if CLI is unavailable): Render Dashboard →
staging service → Environment → Add Environment Variable → repeat for
each. Save. Render prompts to deploy.

**Do NOT paste any of these values into a shell history file, a chat, or
a git-tracked file.** Use the CLI's `--from-file` flag with a `.env`
file kept out of git if the shell-history exposure is a concern:

```
render env set --service <staging-service-id> --from-file staging.env
```

Delete `staging.env` after the command completes.

### 5. Verify the variables are set (before deploying)

```
render env list --service <staging-service-id>
```

**Expect on success.** All ten variables from step 4 appear in the
output, with **values redacted** for the ones marked as secret. If any
are missing, re-run step 4 for the missing ones only. If any secret
values print in plaintext, escalate to security-and-privacy immediately
— that indicates a Render config bug or a wrong service ID.

### 6. Deploy the staging service

Trigger a manual deploy (do not force-deploy if the last commit on
`develop` is not the one you want to verify against):

```
render deploys create --service <staging-service-id> --wait
```

**Expect on success.** Build succeeds, container starts, Spring Boot
startup log shows:

- `The following 1 profile is active: "staging"`
- Flyway applying `V1__b2b_initial_schema.sql` (if this is the first
  deploy against the fresh Neon DB)
- Hibernate schema validation passing (`spring.jpa.hibernate.ddl-auto`
  is `validate` — any drift fails startup loudly)
- `Started WisegiftApplication in <N> seconds`

If the log shows any of the following, one of the env vars is wrong:

- `Failed to configure a DataSource: 'url' attribute is not specified` — `SPRING_DATASOURCE_URL` unset or blank.
- `The server requested password-based authentication, but no password was provided` — `SPRING_DATASOURCE_PASSWORD` unset or blank.
- `type "vector" does not exist` — Neon `pgvector` extension not enabled; return to `neon-staging-provisioning.md` step 3.
- `Missing required property: wisegift.shopify.client-id` (or similar for any of the 5 vars) — the corresponding `WISEGIFT_SHOPIFY_*` env var is unset. This is the "missing any = boot failure" contract from the PR 2 plan.
- `Missing required property: wisegift.encryption-key` — `WISEGIFT_ENCRYPTION_KEY` unset.
- `Could not connect to Redis` — `SPRING_DATA_REDIS_URL` unset, wrong, or pointing at a Redis that is not reachable from the Render EU region.

### 7. Post-deploy verification

Hit the internal health endpoint from a machine that can reach the
staging service:

```
curl -sf https://api-staging.wisegift.app/internal/health
```

**Expect on success.** HTTP 200 with a JSON body listing Postgres,
Redis, and (for PR 6+) Anthropic reachability. PR 2 alone will not
have wired Anthropic yet; Postgres + Redis are the two that matter now.

If this endpoint 404s, either the Render service does not have PR 2
merged yet (which is fine — this runbook must complete before the merge),
or the DNS is not wired (see gap #2 in `README.md`).

For PR 2 specifically, the OAuth install path should be reachable:

```
curl -sI 'https://api-staging.wisegift.app/oauth/shopify/install?shop=test-shop.myshopify.com'
```

**Expect on success.** HTTP 302 redirect to a `shopify.com` URL. If it
returns 500 or the HMAC-verification-required error page, one of the
Shopify env vars is set to the wrong value.

---

## Verification (end-to-end)

The operator confirms staging Render env is ready when **all** of the
following are true:

1. `render env list --service <staging-service-id>` shows all 10
   variables from step 4 with correct names.
2. A manual deploy completes without error and the log shows
   `Started WisegiftApplication`.
3. `/internal/health` returns 200 with Postgres + Redis reachable.
4. (PR 2 merged only) `/oauth/shopify/install?shop=…` returns 302.

Once all four are true, PR 2 is unblocked from an infrastructure
standpoint. The remaining follow-up is a manual OAuth smoke test
against a Shopify dev store per `docs/decisions.md` 2026-09-30
"Backend PR 2 plan" — Follow-ups.

---

## Rollback / recovery

### An env var was set to the wrong value

Overwrite it via the same command:

```
render env set --service <staging-service-id> <NAME>='<correct-value>'
```

Render queues a redeploy; the old value is gone. No historical trail on
Render's side for the old value — if the wrong value contained a secret
that should not have been there (e.g. a production secret pasted into a
staging service), also rotate the underlying secret at the source (Neon
password rotation, Shopify client-secret rotation via Partner dashboard,
etc.). Coordinate rotation with `security-and-privacy`.

### `WISEGIFT_ENCRYPTION_KEY` was regenerated by mistake

**High-severity contract violation.** If any
`platform_credentials.oauth_access_token_encrypted` rows exist in the
staging DB when this happens, those tokens are permanently unreadable
under the new key. Options in order of preference:

1. **If no rows exist** (Neon DB is fresh, PR 2 hasn't been used for
   any Shopify installs yet): revert the env var to any consistent
   value going forward. Nothing was encrypted with the old key, so
   nothing is lost. Preferred outcome — this is why staging exists.
2. **If rows exist** (a Shopify dev-store install has landed): drop
   the `platform_credentials` rows, force every dev store to
   reinstall. Acceptable on staging where the installed shops are
   your own dev stores. Would be catastrophic on production.
3. **If neither is acceptable**: manual re-encrypt-all migration.
   Not built. Escalate.

Escalate to PO and `security-and-privacy` on any accidental regen of
this key regardless of which option applies. Log the incident in
`docs/decisions.md` as a lessons-learned entry.

### The wrong environment got updated (production instead of staging)

Render's CLI takes an explicit `--service <id>` — a wrong service ID
targets the wrong env. If any of the values in step 4 landed on the
production service:

1. Immediately overwrite the affected variables on production with
   their known-good production values.
2. If the wrong values included secrets that leaked from staging into
   production, treat those secrets as **compromised on staging** and
   rotate them: regenerate the Neon password (via Neon dashboard →
   role rotation), rotate the Shopify client secret (Partner
   dashboard), regenerate `WISEGIFT_ENCRYPTION_KEY` on staging (subject
   to the contract above — usually safe on staging as of PR 2 launch).
3. Escalate to PO and `security-and-privacy`.

Prevention: run `render services list` before every `env set` and
confirm the service ID matches `wisegift-backend-staging` (or whatever
the operator's known service name is). Better: alias the staging
service ID in the operator's shell profile so it can never be
mistyped:

```
export WG_STAGING_SVC="srv-abc123"
render env set --service "$WG_STAGING_SVC" …
```

### A deploy failed and the service is now down

Render keeps the last-successful container running until the new
deploy passes health checks. If the failed deploy took the service
down (health-check timeout misconfigured, or a startup crash was
misread as ready), roll back:

```
render deploys list --service <staging-service-id>
render rollback --service <staging-service-id> --to <previous-deploy-id>
```

Then diagnose the env-var mistake from the failed deploy's log and
re-run step 4 for the fix.

---

## Follow-ups after this runbook completes

- Hand off to `backend-engineer` for the PR 2 post-merge smoke test
  per `docs/decisions.md` 2026-09-30 "Backend PR 2 plan" —
  Follow-ups.
- Track "Render staging env provisioned" as closed on the same
  decision entry (append a short follow-up note, do not rewrite
  history).
- When the Managed Redis EU vendor pick lands (`README.md` gap #3),
  update the `SPRING_DATA_REDIS_URL` source column in the inventory
  above and re-set the env var via step 4.
- Production variant of this runbook lands when the first paying
  merchant is confirmed. Production will consume a distinct
  `WISEGIFT_ENCRYPTION_KEY`, a distinct Shopify Partner app (or the
  same app promoted from staging — decision pending), and a distinct
  Neon project.
