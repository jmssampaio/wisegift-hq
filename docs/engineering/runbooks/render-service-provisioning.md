# Runbook — Render service provisioning (staging)

**Purpose.** One-time provisioning of the single `wisegift-backend` Web Service
on Render EU that runs the post-pivot Spring Modulith deployable
(`WisegiftApp` from the `app/` module). Replaces the three pre-pivot Render
services (`catalog`, `gift-recommendation`, `user`) whose source modules were
deleted in PR 1. Covers decommissioning the dead services, creating the new
one, hooking it to a new deploy workflow, and smoke-testing the first deploy.

Scope: **staging only** in this pass. A production variant lands as a separate
pass once the first paying merchant is in sight; this runbook's "repeat for
production" notes flag the deltas inline so the production variant is a diff,
not a rewrite.

**Blocks.** The current absence of a working deploy pipeline for the merged
PR 1 code (`feature/shopify-oauth` branch deleted the pre-pivot
`deploy-staging.yml` / `deploy-production.yml` workflows that targeted the
three dead services). PR 2 merge is also blocked on this — no live staging
means no post-merge OAuth smoke test.

**Owner.** `devops-expert`. Requires product-owner approval to execute — this
runbook is informational.

**Cross-refs.**
- `docs/engineering/architecture.md` §Environments — staging/production URLs,
  EU-only region posture.
- `docs/engineering/architecture.md` §1 Stack — Java 21 Spring Modulith backend.
- `docs/engineering/architecture.md` §11 Deployment topology — "All backend,
  DB, cache, and observability infra in EU regions"; CI/CD convention
  (merge to `develop` → auto-deploy staging; merge to `main` → auto-deploy
  production, both human-gated).
- `docs/decisions.md` 2026-09-29 "Frozen MVP scope" — EU-only hosting frozen
  invariant.
- `docs/decisions.md` 2026-09-29 "Architecture open questions closed" —
  Render EU as the hosting choice.
- `docs/decisions.md` 2026-09-30 "Backend PR 1 merged" — pre-pivot modules
  (`catalog`, `gift-recommendation`, `user`) deleted; single `app/` module is
  the only Spring Boot deployable now.
- `docs/decisions.md` 2026-09-30 "Backend PR 2 plan" — the OAuth install flow
  this service must serve on first useful deploy.
- `neon-staging-provisioning.md` (sibling runbook) — Neon DB this service
  connects to.
- `render-staging-env-vars.md` (sibling runbook) — env-var inventory this
  runbook assumes is already executed against the newly-created service.

---

## Prerequisites

1. **Render account** with access to the operator's existing WiseGift
   organisation and permission to create / suspend / delete Web Services.
2. **`render` CLI** installed (`brew install render`) and authenticated
   (`render login`). CLI preferred over dashboard clicks for diffability; UI
   paths are documented as fallback. Some steps (branch selection on service
   creation, deploy-hook generation) are UI-only at the time of writing.
3. **GitHub repo access.** `jmssampaio/wisegift-backend` must be reachable
   from the operator's Render org — on first use Render prompts an OAuth
   consent to the GitHub app. Grant repo-level access, not org-wide, if
   given the choice.
4. **Neon staging provisioning runbook completed.** The service created here
   is useless without a reachable DB; execute `neon-staging-provisioning.md`
   first and hold the three connection values ready to paste into step 4 of
   `render-staging-env-vars.md`.
5. **Render staging env-vars runbook on standby.** Env vars must be populated
   **before** the first deploy runs (step 5 below), otherwise the first
   build will boot-fail on the encryption-key validator and other required
   properties per `docs/decisions.md` 2026-09-30 "Backend PR 2 plan" —
   Decision 4.
6. **GitHub CLI** (`gh`) installed + authenticated against `jmssampaio` with
   `admin:repo_hook` and `repo` scopes. Used only in step 6 to add deploy-hook
   secrets.

### Current-state gap flags

The following prerequisites are **not resolved** at time of writing. Each is
either a hard blocker on a specific step below or a soft blocker on the
end-to-end verification. Flagged here so the operator can decide whether to
start now and resolve mid-flight, or wait.

1. **`api-staging.wisegift.app` DNS + TLS not wired.** The custom domain
   does not currently resolve to any Render service. The runbook can
   complete through step 7 (service created, env vars set, first deploy
   succeeds, health check passes **against the `onrender.com`-provided
   hostname**). The custom-domain binding is called out explicitly in
   step 7's smoke test and `docs/engineering/runbooks/README.md` gap #2.
   Not fixed by this runbook.
2. **Spring Boot Actuator is not on the `app/` classpath.** Confirmed
   against `wisegift-backend/app/pom.xml` on `develop`: the module declares
   `spring-boot-starter-web`, `-data-jpa`, `-validation`, `-data-redis`,
   plus Flyway and Postgres driver — no `spring-boot-starter-actuator`.
   Consequence: `/actuator/health` returns 404; Render's native "Health
   Check Path" option cannot point there. Options in step 3's health-check
   subsection: (a) point Render at the existing `/internal/health`
   endpoint if PR 2 ships one (per `render-staging-env-vars.md` step 7 it
   does, but PR 2 is not merged at time of writing); (b) add
   `spring-boot-starter-actuator` to `app/pom.xml` in a small follow-up PR
   before this runbook is executed; (c) leave Render's health check field
   blank and rely on container-port-open as the readiness signal (Render's
   default). Lean (b) — tiny PR, standard dependency, puts `/actuator/health`
   and `/actuator/info` on the standard Spring surface for free. Flagged
   explicitly in step 3.
3. **Managed Redis EU vendor pick still TBD.** `render-staging-env-vars.md`
   `SPRING_DATA_REDIS_URL` has no value source. Staging can fall back to a
   Render-managed Redis add-on (available in Frankfurt) for pre-pilot
   smoke tests. Not resolved here. Called out in step 4.

---

## Steps

### 1. Suspend the three pre-pivot Render services

Three services exist on the operator's Render dashboard from the pre-pivot
B2C stack:

- `catalog`
- `gift-recommendation`
- `user`

All three now point at source directories that no longer exist on `develop`
(deleted in PR 1 per `docs/decisions.md` 2026-09-30 "Backend PR 1 merged").
Any auto-deploy on `develop` would already be failing — but any **manual**
redeploy from an older commit on their configured branch would still boot
the pre-pivot code against the new Neon DB and corrupt the schema. Suspend
first to kill both failure modes without irreversibly losing the service
configuration.

Order of operations is: suspend all three → wait a short grace period
(suggested: 24 hours; no production tenant data at stake, but gives time to
realise if an unknown integration still depends on one of these URLs) →
delete in step 8 at the end.

Per service, Render Dashboard → Services → click the service name → Settings
(top-right) → scroll to "Suspend Service" → confirm.

**Expect on success.** Service badge changes to "Suspended". Deploy history,
env vars, and the deploy hook URL remain accessible. No HTTP traffic is
served; auto-deploy is paused.

CLI alternative (if available in your `render` CLI version):

```
render services suspend srv-<catalog-service-id>
render services suspend srv-<gift-recommendation-service-id>
render services suspend srv-<user-service-id>
```

**Preserve for audit?** Pre-pivot services carry no production tenant data
(the B2C project never served real merchants per
`docs/decisions.md` 2026-09-29 "Strategic pivot"). Env vars contained only
Firebase service-account JSON + pre-pivot DB credentials, both of which are
either rotated or dead already. Deletion in step 8 is safe without an
export. If the operator wants a belt-and-braces snapshot of the env-var
lists before deletion for an audit trail, Render Dashboard → service →
Environment → copy to a password-manager vault entry; do not commit
anywhere.

### 2. Create the new Web Service

Render dashboard UI path (CLI does not support branch selection + runtime
pick in one shot at time of writing):

1. Render Dashboard → **New** (top-right) → **Web Service**.
2. **Connect a repository** → GitHub tab → select
   `jmssampaio/wisegift-backend`. Grant repo access if prompted (first time
   only).
3. **Name.** `wisegift-backend-staging`.
   - Repeat-for-production delta: `wisegift-backend-production`.
4. **Region.** `Frankfurt (EU Central)`. Matches the Neon project region
   picked in `neon-staging-provisioning.md` step 1 (`aws-eu-central-1`).
   Keeping backend + DB in the same cloud region cuts round-trip latency
   on every query and keeps all data on one AWS region's network. If the
   PO switched Neon to Dublin (`aws-eu-west-1`) mid-execution of the Neon
   runbook, also switch here — do not split the pair.
5. **Branch.** `develop`.
   - Repeat-for-production delta: `main`.
   - Rationale: per `docs/engineering/architecture.md` §11 "Merge to
     `develop` → auto-deploy to staging; promotion to `main` → auto-deploy
     to production." One service per branch, one branch per environment.
6. **Root directory.** Leave blank (defaults to repo root). The build
   command below targets the `app/` module explicitly from the repo root
   via Maven's `-pl` + `-am` flags.
7. **Runtime.** Pick **"Node"** `NO` — pick **native Java** via
   Render's "Language" selector → **Java**. Render supports Java 21
   natively (Temurin distribution). Native is simpler than Docker here:
   the repo currently ships no Dockerfile (confirmed against
   `wisegift-backend/` listing on `develop`), writing one adds a surface
   area to maintain, and Render's native Java runtime handles the
   `mvn package` → `java -jar` flow without a container boundary. Switch
   to Docker only if we later need base-image reproducibility beyond what
   Render's Temurin image offers.
   - Set **Java version** = `21`. Matches
     `wisegift-backend/pom.xml` `<java.version>21</java.version>` and
     `.github/workflows/ci.yml` step "Set up JDK 21".
8. **Build Command.**
   ```
   mvn -pl app -am clean package -DskipTests
   ```
   - `-pl app` builds just the `app` module.
   - `-am` ("also-make") pulls its reactor dependencies (`shared`,
     `tenant`, `platform-integrations`) in the same reactor pass.
   - `-DskipTests` — CI already runs the full `mvn clean verify` on every
     PR per `.github/workflows/ci.yml`. Running tests again at deploy
     time doubles the build minute cost and adds no safety: a PR that
     passed CI is the only thing that ever gets to `develop`.
9. **Start Command.**
   ```
   java -jar app/target/app-1.0.0-exec.jar
   ```
   - **The `-exec` classifier is critical.** `wisegift-backend/app/pom.xml`
     configures `spring-boot-maven-plugin` with `<classifier>exec</classifier>`
     (lines 90-97, with a code comment explaining why — the plain jar is
     kept as the primary artifact so Failsafe can walk the classloader
     normally during `mvn verify`; the executable fat jar lands as the
     classified `-exec` jar).
   - Consequence: `java -jar app/target/app-1.0.0.jar` **will not boot**
     — the plain jar has no `Main-Class` manifest entry pointing at the
     Spring Boot launcher, and will crash with
     `no main manifest attribute`. Only `app-1.0.0-exec.jar` is
     bootable. This is non-obvious; the only hint is the pom comment.
10. **Instance type.** `Starter` (~$7/month). Sufficient for pre-pilot
    staging load (operator + a handful of dev-store webhook callbacks).
    Resize later if the per-tenant kill-switch telemetry work (post-PR 6)
    needs more memory. Repeat-for-production: start with `Standard`
    (~$25/month, 2 GB RAM) at first paying-merchant launch — leaves JVM
    headroom for the response cache + the recommendation engine's hot
    path.
11. **Health Check Path.** **Leave blank for now.** See gap #2 above —
    `/actuator/health` does not exist yet because
    `spring-boot-starter-actuator` is not a dependency. Render falls back
    to "container port open" as the readiness signal, which is acceptable
    for the first deploy. Follow-up gap flagged in step 10 and in
    "Adjacent gaps" at the bottom.
12. **Auto-Deploy.** **Set to "No" for now** — we will enable it in step 7
    only after step 5 confirms env vars are populated. If auto-deploy is
    "Yes" on service creation, Render immediately kicks a deploy against
    the current `develop` commit with zero env vars configured, which
    boot-fails on the first required-property validator. Avoid the noise.
13. Click **Create Web Service**.

**Expect on success.** Render provisions the service, assigns an
`onrender.com` hostname (format: `wisegift-backend-staging-<hash>.onrender.com`
or similar — note the exact value for step 7's smoke test), and lands on
the service dashboard with the badge "Deploy not yet triggered" (because
auto-deploy was disabled in step 12).

If Render fires a first deploy anyway (defaults have drifted since this
runbook was written), that deploy will fail at Spring Boot startup on
missing properties. Not catastrophic — the service stays provisioned, the
deploy history just carries one red entry. Proceed to step 3.

### 3. Decide and apply the health-check configuration

Three options from gap #2. Pick one before enabling auto-deploy in step 7.

- **(a) `/internal/health`** — PR 2 (not yet merged) wires a Spring MVC
  endpoint at `/internal/health` per `render-staging-env-vars.md` step 7.
  Will work once PR 2 lands. Does not work for the first deploy of PR 1-only
  code (there is no `/internal/health` on PR 1's merged state).
- **(b) Add `spring-boot-starter-actuator` + enable `/actuator/health`.**
  Smallest viable change. Add the dependency to `wisegift-backend/app/pom.xml`
  (follow-up PR, out of scope for this runbook — flag to `backend-engineer`),
  set `management.endpoints.web.exposure.include=health,info` in
  `application.yml`, set `management.endpoint.health.probes.enabled=true` so
  `/actuator/health/liveness` + `/actuator/health/readiness` are also
  available (standard Kubernetes-style split that Render understands too).
  **Lean (b).** This is the long-term correct answer and the followup is
  10 minutes of work.
- **(c) Leave blank, rely on port-open.** Acceptable for the pre-pilot
  phase. Downside: Render will mark the container "live" the moment the
  JVM binds port 8080, which happens before Flyway migration completes.
  A failed Flyway migration mid-startup can cause the deploy to be
  reported as succeeded then immediately start 500ing on all requests.

Recommendation: ship (b) as a follow-up PR before this runbook is
executed. If the operator wants to execute this runbook today and defer
the actuator PR to tomorrow, pick (c) now and switch to (b) later by
editing Render Dashboard → service → Settings → Health Check Path.

### 4. Populate env vars

Switch to `render-staging-env-vars.md` and execute every step. All 10
environment variables listed in that runbook's inventory must be set on
`wisegift-backend-staging` **before** step 7 enables auto-deploy. Return
here when that runbook's step 5 ("Verify the variables are set") shows
all 10 names present.

Reminder: `SPRING_DATA_REDIS_URL` is blocked on gap #3 (Redis vendor
pick). Staging workaround — add a Render-managed Redis add-on via
Render Dashboard → New → Redis → Frankfurt region → Starter plan
(~$10/month). Render auto-populates the connection string as an env var
on the staging service if you link it; verify the var name matches
`SPRING_DATA_REDIS_URL` (Render's auto-population defaults vary — it may
set `REDIS_URL` instead; rename or alias).

### 5. Trigger the first deploy manually

With env vars populated and auto-deploy still disabled, trigger one
deploy by hand and watch the log before switching auto-deploy on.

```
render deploys create --service <staging-service-id> --wait
```

Dashboard fallback: service page → **Manual Deploy** → **Deploy latest
commit from `develop`**.

**Expect on success** (full checklist from `render-staging-env-vars.md`
step 6, re-stated here because it is the single most important
verification in the runbook):

- Build phase: `mvn -pl app -am clean package -DskipTests` completes.
  Watch for the line `Building App 1.0.0` and the final
  `BUILD SUCCESS`.
- Boot phase: `java -jar app/target/app-1.0.0-exec.jar` starts, log
  shows `The following 1 profile is active: "staging"`, Flyway applies
  `V1__b2b_initial_schema.sql` against the Neon DB (first deploy only;
  subsequent deploys see V1 already applied and skip), Hibernate
  `ddl-auto=validate` passes, embedded Tomcat binds port 8080, final
  line `Started WisegiftApplication in <N> seconds`.

Common failure modes with pointers:
- `no main manifest attribute, in app/target/app-1.0.0.jar` — start
  command missing the `-exec` suffix. See step 2.9.
- `Failed to configure a DataSource` / `password-based authentication` —
  Neon env vars wrong or missing. Back to `render-staging-env-vars.md`
  step 4.
- `type "vector" does not exist` — pgvector not enabled on Neon. Back to
  `neon-staging-provisioning.md` step 3.
- `Missing required property: wisegift.encryption-key` — see
  `render-staging-env-vars.md` step 1.

### 6. Grab the deploy hook and wire GitHub Actions

With the first deploy green, generate a deploy hook for use by the new
GitHub Actions workflow.

Render Dashboard → `wisegift-backend-staging` → **Settings** → scroll to
**Deploy Hook** → **Generate Hook**. Copy the URL (format:
`https://api.render.com/deploy/srv-<id>?key=<token>`). Treat as secret —
anyone with the URL can trigger a staging deploy.

Add to GitHub repository secrets on `jmssampaio/wisegift-backend`:

```
gh secret set RENDER_APP_STAGING_DEPLOY_HOOK \
  --repo jmssampaio/wisegift-backend \
  --body '<hook-url>'
```

Dashboard fallback: GitHub repo → Settings → Secrets and variables →
Actions → New repository secret → Name `RENDER_APP_STAGING_DEPLOY_HOOK`,
Value the hook URL.

Repeat-for-production delta: `RENDER_APP_PROD_DEPLOY_HOOK` from the
production service's own hook.

**Delete the pre-pivot deploy-hook secrets** at the same time (they are
dead — the services they pointed at are suspended, and will be deleted
in step 8). Explicit list of secrets to delete, derived from the
deleted `deploy-staging.yml` / `deploy-production.yml` workflows:

- `RENDER_CATALOG_STAGING_DEPLOY_HOOK`
- `RENDER_CATALOG_PROD_DEPLOY_HOOK`
- `RENDER_GIFT_RECOMMENDATION_STAGING_DEPLOY_HOOK`
- `RENDER_GIFT_RECOMMENDATION_PROD_DEPLOY_HOOK`
- `RENDER_USER_STAGING_DEPLOY_HOOK`
- `RENDER_USER_PROD_DEPLOY_HOOK`

```
gh secret delete RENDER_CATALOG_STAGING_DEPLOY_HOOK --repo jmssampaio/wisegift-backend
gh secret delete RENDER_CATALOG_PROD_DEPLOY_HOOK --repo jmssampaio/wisegift-backend
gh secret delete RENDER_GIFT_RECOMMENDATION_STAGING_DEPLOY_HOOK --repo jmssampaio/wisegift-backend
gh secret delete RENDER_GIFT_RECOMMENDATION_PROD_DEPLOY_HOOK --repo jmssampaio/wisegift-backend
gh secret delete RENDER_USER_STAGING_DEPLOY_HOOK --repo jmssampaio/wisegift-backend
gh secret delete RENDER_USER_PROD_DEPLOY_HOOK --repo jmssampaio/wisegift-backend
```

Verify none remain under the pre-pivot naming:

```
gh secret list --repo jmssampaio/wisegift-backend | grep -E 'CATALOG|GIFT_RECOMMENDATION|_USER_'
```

**Expect on success.** Grep returns no matches; only `RENDER_APP_*`
secrets remain for Render deploy hooks.

### 7. Propose the new deploy workflows

**Do not commit these files from this runbook.** The runbook proposes
the exact content; a separate PR by `backend-engineer` or `devops-expert`
with explicit PO approval commits them to
`wisegift-backend/.github/workflows/`.

Proposed `.github/workflows/deploy-staging.yml`:

```yaml
name: Deploy — Staging

on:
  push:
    branches: [develop]

jobs:
  deploy:
    name: Trigger Render staging deploy
    runs-on: ubuntu-latest
    steps:
      - name: Call Render deploy hook
        run: curl -fsS -X POST "${{ secrets.RENDER_APP_STAGING_DEPLOY_HOOK }}"
```

Proposed `.github/workflows/deploy-production.yml`:

```yaml
name: Deploy — Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    name: Trigger Render production deploy
    runs-on: ubuntu-latest
    steps:
      - name: Call Render production deploy hook
        run: curl -fsS -X POST "${{ secrets.RENDER_APP_PROD_DEPLOY_HOOK }}"
```

Design notes:
- One file per environment. Matches the existing CI workflow's
  "one file, one concern" convention (`.github/workflows/ci.yml` is
  only CI; it does not deploy).
- `curl -fsS` — `-f` fails on HTTP 4xx/5xx so a Render-side error
  surfaces as a red CI step, `-s` silences progress, `-S` keeps error
  messages visible on failure. Matches the convention from
  `.github/workflows/ci.yml` (short, explicit flags, no magic).
- No `actions/checkout@` — the workflow does not need the repo content;
  it only hits an external URL. Keep it minimal.
- No pinning to a hash because no third-party actions are invoked; the
  `curl` binary ships on the `ubuntu-latest` image. If a future
  workflow adds an action (e.g. a Slack-notify step), pin it to a hash
  per the devops-expert coding standard.
- Trigger: `push` not `pull_request`. These workflows run **after**
  merge to `develop` / `main`, which is the human-gated step per
  `docs/engineering/release-process.md`. The human approves the merge;
  the deploy is a mechanical follow-on.

Once these workflows are committed and the next push to `develop` goes
green, flip Render's auto-deploy flag (Dashboard → service → Settings
→ Auto-Deploy → **Yes**). Both mechanisms (push → GitHub Action →
deploy hook, and Render's own auto-deploy on branch push) will run on
the same commit, which is **intentionally redundant** — if the GitHub
Action is suspended for billing or quota, Render's native auto-deploy
still ships. Deploys are idempotent (same commit, same build cache), so
the double-fire costs one extra build minute and does not corrupt
state. Pick one if the double-fire becomes a cost concern post-pilot.

### 8. Smoke-test the live staging service

After the first manual deploy from step 5 is green, run these from the
operator's machine. Replace `<render-hostname>` with the
`onrender.com` URL assigned by Render on service creation. Switch to
`api-staging.wisegift.app` once gap #1 (DNS wiring) is resolved.

Base reachability:

```
curl -sfI https://<render-hostname>.onrender.com/
```

**Expect.** HTTP 404 or 401 from Spring Security's default
"no mapping for `/`" response — this is the correct signal that the
container is up and Spring Boot's dispatcher is responding. HTTP 502 or
connection timeout means the deploy did not actually come up — go
back to Render's deploy log.

Health endpoint (depends on step 3 choice):

```
# If step 3 picked (b) actuator:
curl -sf https://<render-hostname>.onrender.com/actuator/health
# Expect: {"status":"UP"} or {"status":"UP","components":{...}}

# If step 3 picked (c) blank + waiting for PR 2:
# No endpoint to hit yet — rely on Render's "Live" badge in the dashboard.
```

PR 2 OAuth install placeholder (only valid once PR 2 is merged — this
runbook executes before PR 2 merge, so this check is for the next
operator pass):

```
curl -sfI 'https://<render-hostname>.onrender.com/oauth/shopify/install?shop=test-shop.myshopify.com'
```

**Expect.** HTTP 302 redirect to a `shopify.com` authorise URL.

PR 2 success-page placeholder (also post-PR-2-merge only):

```
curl -sf https://<render-hostname>.onrender.com/oauth/shopify/success
```

**Expect.** HTTP 200 with the static success-page HTML body per the
PR 2 plan (`docs/decisions.md` 2026-09-30 "Backend PR 2 plan" —
Decision 6).

### 9. Delete the pre-pivot services

After the grace period from step 1 (suggested 24 hours), delete the
three suspended services. This is irreversible — the service IDs are
gone, deploy history is wiped, and any lingering references to the
service URLs (bookmarks, old env-var pastes) will 404 permanently.

Per service, Render Dashboard → suspended service → Settings → scroll
to bottom → **Delete Service** → type the service name to confirm.

**Order.**
1. `user` (least risk, no inbound traffic since pivot).
2. `gift-recommendation` (same).
3. `catalog` (last — if any pre-pivot ingestion job somewhere in the
   operator's local environment is still pointed at it, it will fail
   loud on deletion, which is useful signal).

Also delete any associated **private databases** on Render if the
pre-pivot services provisioned Render-managed Postgres instances
(unlikely — the pre-pivot stack used Neon per `docs/decisions.md`
2026-09-29 — but check Dashboard → Databases list). Pre-pivot
Render-side Redis add-ons: delete if present.

### 10. Enable auto-deploy

With the new service verified live and the GitHub Actions workflows
committed via a separate PR (step 7), enable Render's auto-deploy:

Render Dashboard → `wisegift-backend-staging` → Settings → **Auto-Deploy**
→ **Yes** → **Save Changes**.

**Expect on success.** Next push to `develop` triggers both the GitHub
Action and Render's native auto-deploy. Deploy history shows the merge
commit SHA. See step 7 for the intentional-redundancy note.

Follow-up tracked: the actuator-starter PR referenced in step 3. Flag
to `backend-engineer` on PR 2 scope creep check or as its own PR.

---

## Verification (end-to-end)

The operator confirms the staging Render service is ready when **all**
of the following are true:

1. Render Dashboard shows one Web Service named `wisegift-backend-staging`
   in the Frankfurt region, branch `develop`, Live status.
2. The three pre-pivot services (`catalog`, `gift-recommendation`,
   `user`) are deleted (step 9 complete).
3. The first manual deploy (step 5) is green and the log shows
   `Started WisegiftApplication`.
4. The six pre-pivot GitHub deploy-hook secrets are deleted; the one
   new `RENDER_APP_STAGING_DEPLOY_HOOK` secret exists on the repo.
5. The staging URL responds to the smoke-test probes in step 8 as
   described.
6. Auto-deploy is enabled on the service.

Once all six are true, the staging deploy pipeline is restored. PR 2
merge is unblocked from a deploy-pipeline standpoint (still needs
`neon-staging-provisioning.md` and `render-staging-env-vars.md`
executed first, which this runbook assumes are done).

---

## Rollback / recovery

### A deploy failed and the service is now broken

Render keeps the last-successful container running until the new deploy
passes its post-build phase, so a failed deploy usually does not take
the service down. If the broken deploy did go live (misconfigured
health check, startup crash reported as success), roll back via Render's
built-in button:

Render Dashboard → `wisegift-backend-staging` → **Deploys** → find the
previous green deploy → **Rollback to this deploy** → confirm.

CLI alternative:

```
render deploys list --service <staging-service-id>
render rollback --service <staging-service-id> --to <previous-deploy-id>
```

**Expect on success.** The previous build's container image boots back
up and takes traffic. No DB state is rolled back (Flyway migrations are
forward-only — the schema at commit N+1 remains, running against the
code from commit N). This is usually safe for schema additions
(additive columns, new tables); it is **not safe** if commit N+1 ran a
destructive migration (dropped a column, renamed a table). Flag any
such migration in the PR description so the rollback decision at deploy
time is informed.

### A broken workflow commit is live on `develop`

If the committed `deploy-staging.yml` itself is broken (bad YAML, wrong
secret name, wrong curl flags), the deploy fails but the service is not
harmed. Roll back the workflow commit:

```
git revert <broken-workflow-commit-sha>
git push origin develop
```

Render's native auto-deploy still runs on the revert commit, so the
service does not need separate attention. Fix the workflow in a
follow-up PR.

### The wrong service was targeted (production instead of staging)

Deploy hooks are per-service URLs; mis-pasting the production hook URL
into `RENDER_APP_STAGING_DEPLOY_HOOK` would mean every push to `develop`
deploys to production. This is high-severity and matches the
`render-staging-env-vars.md` "wrong environment got updated" rollback
pattern:

1. Immediately overwrite the secret with the correct staging hook URL
   via step 6's `gh secret set` command.
2. Audit the last few deploys on both services (Render Dashboard →
   each service → Deploys) to confirm which commits landed where.
3. If any `develop`-commit shipped to production unintentionally, roll
   production back to the last known-good production deploy per the
   section above.
4. Escalate to PO and `security-and-privacy`.

Prevention: when generating the hook in step 6, immediately verify the
service name shown in the Render dashboard URL bar matches the secret
name you are about to set. The hook URL contains the service ID
(`srv-<id>`) which can be cross-checked against
`render services list`.

### The new service was created in the wrong region

Render does not support in-place region migration. Delete the service
(Dashboard → Settings → Delete Service) and re-run step 2 with the
correct region. No data at stake if this is caught before the first
successful deploy; after the first successful deploy, the Neon DB is
in Frankfurt and a Render service in another region adds a trans-EU
hop (not catastrophic but violates the "co-locate backend + DB"
rationale from step 2.4).

---

## Cost expectations

Pre-pivot steady state:
- `catalog` Starter: ~$7/month
- `gift-recommendation` Starter: ~$7/month
- `user` Starter: ~$7/month
- Total: **~$21/month**

Post-provisioning steady state (staging only):
- `wisegift-backend-staging` Starter: ~$7/month
- Render-managed Redis (staging fallback per gap #3): ~$10/month
- Total: **~$17/month** for staging

Net change at this runbook's conclusion: **~$4/month saving on compute**
(three dead $7 services replaced by one), **~$10/month new spend on
managed Redis** (only if the Redis vendor pick stays with Render for
staging). Net: roughly flat.

Repeat-for-production delta (future pass): `wisegift-backend-production`
Standard instance ~$25/month + production Redis (vendor TBD, likely in
the $15-30/month band for a small managed EU Redis). Expected
production add: ~$40-55/month. Not within this runbook's scope — flagged
so the operator sees the cost shape before approving the production
pass.

---

## Adjacent gaps (surfaced during this runbook's drafting)

Everything flagged in the "Current-state gap flags" section of
Prerequisites, re-stated here as a consolidated TODO list that outlives
this runbook's execution:

1. **`api-staging.wisegift.app` DNS not wired.** Needs a CNAME to the
   Render-assigned `onrender.com` hostname, plus Render Dashboard →
   service → Settings → Custom Domains → add → verify ownership. Owner
   candidate: `devops-expert` once Cloudflare (or wherever
   `wisegift.app` DNS lives) credentials are confirmed with PO.
2. **Spring Boot Actuator missing from `app/pom.xml`.** Small follow-up
   PR by `backend-engineer`: add `spring-boot-starter-actuator`
   dependency, set `management.endpoints.web.exposure.include=health,info`
   in `application.yml`, add `management.endpoint.health.probes.enabled=true`.
   Enables Render's "Health Check Path" to point at `/actuator/health`
   per step 3 option (b).
3. **Managed Redis EU vendor pick still TBD.** Documented in
   `docs/engineering/runbooks/README.md` gap #3. This runbook falls back
   to Render-managed Redis for staging; production needs a real pick
   before first-paying-merchant launch. Candidates: Upstash EU, Aiven
   Redis EU, Render-managed Redis Standard tier.
4. **Production variant of this runbook.** Repeat-for-production notes
   are inlined throughout but not consolidated into a separate runbook.
   Write when the first paying merchant is confirmed.
5. **`postman/` smoke-test collection for the staging API.** Flagged
   already in `docs/engineering/runbooks/README.md` gap #5. The step 8
   `curl` block is a minimal substitute; a proper Postman collection
   would be more operator-friendly for repeated smoke tests.

---

## Follow-ups after this runbook completes

- Hand off to `backend-engineer` for the actuator-starter PR (gap #2).
- Hand off to `devops-expert` (self) for the `api-staging.wisegift.app`
  DNS pass (gap #1).
- Hand off to `devops-expert` for the production variant of this
  runbook once PO confirms first-paying-merchant timing.
- Append a short closure note to `docs/decisions.md` 2026-09-30
  "Backend PR 1 merged" — Outstanding follow-ups, marking "Render
  staging service created + pre-pivot services decommissioned" as
  closed. Do not rewrite the entry; append.
- Coordinate with `security-and-privacy` on the eventual secrets
  manager pick (per `docs/engineering/runbooks/README.md` gap #6) —
  until then, Render's env-var store is the staging secret store.
