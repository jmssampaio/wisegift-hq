# Operational runbooks — inventory

Step-by-step operational procedures for provisioning, configuring, and
recovering WiseGift infrastructure. Runbooks are informational: they document
what a human operator should do. Nothing here executes automatically. Match the
tone/format of `docs/engineering/release-process.md`.

Cross-references to rationale go into `docs/decisions.md` — runbooks link, they
do not restate.

---

## Current runbooks

| Runbook | Purpose | Status | Blocking |
|---|---|---|---|
| [`neon-staging-provisioning.md`](neon-staging-provisioning.md) | One-time Neon EU staging project provisioning + `pgvector` extension enable + connection-string retrieval | Pending execution | Blocks Flyway V1 running on staging; blocks `wisegift-backend` PR 2 merge |
| [`render-staging-env-vars.md`](render-staging-env-vars.md) | Inventory + provisioning of every environment variable the staging Spring Boot app requires | Pending execution | Blocks `wisegift-backend` PR 2 merge (missing any of the 5 Shopify vars = boot failure) |

Both runbooks share the same 2026-09-30 unblock target: PR 2 (Shopify OAuth
install flow, `platform-integrations` module) cannot merge until each has been
executed end-to-end against staging.

---

## Prerequisites & gaps (project-wide)

The following are **not owned by any of the two runbooks below** and must be
resolved separately. Runbooks assume the operator has already handled these,
or explicitly flag when a step depends on one.

1. **Render staging service existence.** It is not confirmed at the time of
   writing that a Render Web Service for `wisegift-backend` staging exists. If
   the service does not exist yet, the `render-staging-env-vars.md` runbook
   cannot complete its "set the env vars" steps — the service must be created
   first (Render Dashboard → New → Web Service → connect the
   `wisegift-backend` GitHub repo → `develop` branch → EU region — Frankfurt).
   This is a separate one-time task and warrants its own runbook once
   scheduled. Flagged here so the gap is visible; not solved here.

2. **`api-staging.wisegift.app` DNS + TLS.** The staging backend URL used by
   the Shopify Partner dashboard's redirect URI configuration
   (`https://api-staging.wisegift.app/oauth/shopify/callback`, per
   `docs/decisions.md` 2026-09-30 "Backend PR 2 plan") must resolve to the
   Render service and terminate TLS via a valid certificate. If the custom
   domain is not yet mapped in Render + DNS (Cloudflare or wherever
   `wisegift.app` is administered), Shopify's OAuth callback will fail. Not
   solved by either runbook; must land before PR 2's post-merge smoke test.

3. **Managed Redis EU vendor pick.** Per `docs/decisions.md` 2026-09-29
   "Architecture open questions closed" and the follow-up in
   "Docs housekeeping: stale references cleaned", the Redis provider is
   still TBD. PR 2 uses Redis for the OAuth CSRF nonce and the webhook-id
   dedupe key. Staging can defer with a Render-managed Redis add-on (available
   in EU regions) or a docker-hosted Redis on the Render service — neither is
   production-grade but both unblock PR 2 for staging smoke tests. Do not use
   the Render-hosted-in-service Redis for production; that requires the vendor
   pick to close first. The Render env-vars runbook documents `SPRING_DATA_REDIS_URL`
   as required and leaves the value source pending until the pick lands.

4. **Widget CDN vendor pick.** Not blocking PR 2 (PR 2 is backend-only). Called
   out in the same follow-up. Will be its own runbook when the pick lands.

5. **`postman/` API smoke-test collection.** Deleted in PR 1 (per
   `docs/decisions.md` 2026-09-30 "Backend PR 1 merged" — closure notes). A
   fresh staging smoke-test collection targeting PR 2's OAuth endpoints does
   not yet exist. Not blocking PR 2 merge; blocks a repeatable post-merge
   smoke test. `backend-engineer` follow-up.

6. **Secrets manager choice.** Not blocking for staging (Render's own env-var
   store is acceptable at MVP scale and encrypted at rest by Render). For
   production, a dedicated secrets manager (Doppler, 1Password Secrets
   Automation, HashiCorp Vault, etc.) should land before the first paying
   merchant. Coordinate with `security-and-privacy`.

---

## Conventions

- **Never execute anything from a runbook without explicit product-owner
  approval.** Runbook execution is a side-effectful action — deploys, infra
  provisioning, key generation, DNS changes — and is human-gated per the
  `devops-expert` charter and `CLAUDE.md`.
- **Every step has a verification.** If a step cannot be verified, the
  operator stops and escalates before continuing.
- **Every step has a rollback.** If a step lands in the wrong environment,
  the rollback section says how to unstick without cascading damage.
- **Runbooks reference decisions by date + short title** rather than
  restating rationale. If a step's rationale is unclear, follow the
  cross-reference into `docs/decisions.md`.
- **Secrets never appear in runbook text.** Values are generated during
  execution, pasted into the target system (Render, Shopify Partner
  dashboard, Neon UI), and never committed anywhere.
