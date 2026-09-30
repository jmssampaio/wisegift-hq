# Runbook — Neon EU staging project provisioning

**Purpose.** One-time provisioning of the Neon Postgres project that backs the
`wisegift-backend` staging environment, including enabling the `pgvector`
extension so Flyway's `V1__b2b_initial_schema.sql` can run on first Spring
Boot startup.

**Blocks.** `wisegift-backend` PR 2 (`docs/decisions.md` 2026-09-30 "Backend
PR 2 plan"). PR 2 assumes an up-and-running Neon staging DB with the extension
already loaded; the current follow-up ("Backend PR 1 merged" — Outstanding
follow-ups) leaves this task assigned to `devops-expert` and unexecuted.

**Owner.** `devops-expert`. Requires product-owner approval to execute — this
runbook is informational.

**Cross-refs.**
- `docs/engineering/architecture.md` §Environments — staging URLs and the
  EU-only region posture.
- `docs/engineering/architecture.md` §11 Deployment topology — "All backend,
  DB, cache, and observability infra in EU regions."
- `docs/decisions.md` 2026-09-29 "Frozen MVP scope" — EU-only hosting is a
  frozen invariant; adding a US region requires a separate PO decision + DPA
  update.
- `docs/decisions.md` 2026-09-29 "Backend PR 1 plan" — "Fresh Neon EU project.
  No B2C data to migrate. Enable `pgvector` extension in the Neon UI once per
  environment before first Spring Boot run."
- `docs/decisions.md` 2026-09-30 "Backend PR 1 merged" — Outstanding
  follow-ups: Neon EU staging + `CREATE EXTENSION vector`.
- `docs/product/spec.md` "Feature: Multi-tenancy — Isolation model" — every
  tenant-scoped table carries `tenant_id UUID NOT NULL`; V1 lays this floor.

---

## Prerequisites

1. **Neon account** (`console.neon.tech`) with billing enabled and permission
   to create projects. If unsure who owns the WiseGift Neon org, escalate to
   PO before creating a personal-org project.
2. **`neonctl` CLI** installed locally (`npm i -g neonctl`) and authenticated
   via `neonctl auth`. CLI is preferred over UI clicks for step reproducibility
   — every step below shows the CLI command; UI is documented only as a
   fallback.
3. **`psql`** installed locally (Postgres 16 client — matches the
   `pgvector/pgvector:pg16` image in `wisegift-backend/docker-compose.yml`).
4. **Confirmed empty state**: no pre-existing `wisegift-staging` project on
   the Neon org. If one exists from earlier experimentation, coordinate a
   drop-and-recreate with PO first (per `docs/decisions.md` 2026-09-29
   "Backend PR 1 plan" — Consequences: "Existing Neon project (if any) on
   staging/production must be replaced or dropped-and-recreated. No user data
   to preserve.").
5. **This runbook is not authorised to touch production.** Only the staging
   project. Production Neon provisioning is a separate runbook to be written
   when the first paying merchant is in sight.

---

## Steps

### 1. Create the Neon project

Neon EU region choice: **Frankfurt (`aws-eu-central-1`)**. Rationale is not
documented in a decision entry (the frozen scope pins "EU-only" but does not
pick between Frankfurt and Ireland); Frankfurt is picked here for latency
proximity to the ES/PT + eventually DE pilot pool per `docs/decisions.md`
2026-09-29 "Frozen MVP scope". If PO prefers Dublin (`aws-eu-west-1`) for
Ireland-DPA reasons, swap the region flag; do not run both.

```
neonctl projects create \
  --name wisegift-staging \
  --region-id aws-eu-central-1 \
  --org-id <wisegift-org-id>
```

**Expect on success.** JSON response containing `project.id`,
`project.region_id: "aws-eu-central-1"`, and an initial branch (`main`) with
a compute endpoint. Note the `project.id` — every subsequent `neonctl` call
below is scoped to it.

If `--org-id` is omitted the project lands in the caller's personal org,
which is wrong for a shared staging DB. Confirm the org ID with PO before
running the command.

### 2. Confirm the region

```
neonctl projects get <project-id>
```

**Expect on success.** `region_id: "aws-eu-central-1"`. If the value is
anything else (e.g. `aws-us-east-2`), the project is in the wrong region —
stop, delete the project (step "Rollback / recovery" below), and re-run
step 1 with the correct flag. Do not proceed with a non-EU project; that
breaks the frozen EU-only invariant.

### 3. Enable the `pgvector` extension on the primary branch

Neon inherits Postgres's extension mechanism. `CREATE EXTENSION` runs against
a branch, not the project. The primary branch is `main` (Neon-side; unrelated
to the git `main` branch). Every subsequent branch created off `main`
inherits the extension because it forks the underlying storage state, so
enabling once on `main` covers all future preview branches without repeating
this step per-branch.

Obtain the direct (non-pooled) connection string for the `main` branch first
(pooled connections don't allow `CREATE EXTENSION`):

```
neonctl connection-string \
  --project-id <project-id> \
  --branch main \
  --role-name neondb_owner \
  --database-name neondb
```

**Expect on success.** A URI of the form
`postgresql://neondb_owner:<password>@ep-<compute-id>.eu-central-1.aws.neon.tech/neondb?sslmode=require`.
Copy it into a shell variable — do not paste into any tracked file.

```
export NEON_DIRECT_URL="postgresql://…"
psql "$NEON_DIRECT_URL" -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

**Expect on success.** `CREATE EXTENSION` printed to stdout. If the DB
already had it enabled (unlikely on a fresh project), `NOTICE: extension
"vector" already exists, skipping` and the command still exits 0 — safe.

If the command fails with `permission denied to create extension "vector"`,
the role does not carry the required privilege. On Neon, `pgvector` is on
the allowlist for the `neondb_owner` role — verify the role in the connection
string is `neondb_owner`, not a role you may have created manually.

### 4. Verify the extension is loaded

```
psql "$NEON_DIRECT_URL" -c "SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';"
```

**Expect on success.**

```
 extname | extversion
---------+------------
 vector  | 0.7.0
(1 row)
```

Version number will vary — anything ≥ `0.5.0` is fine for the 1536-dimension
column in `V1__b2b_initial_schema.sql`. If the row is missing, step 3 did not
land; re-run step 3 before proceeding.

### 5. Optional pre-check — dry-run V1 locally against Neon

Not required. Useful if the operator wants to catch schema-migration errors
before Spring Boot's first boot writes anything durable. From the
`wisegift-backend` repo root:

```
SPRING_PROFILES_ACTIVE=staging \
SPRING_DATASOURCE_URL="jdbc:postgresql://ep-<compute-id>.eu-central-1.aws.neon.tech/neondb?sslmode=require" \
SPRING_DATASOURCE_USERNAME="neondb_owner" \
SPRING_DATASOURCE_PASSWORD="<password>" \
./mvnw -pl app -am spring-boot:run
```

**Expect on success.** Spring Boot startup log shows Flyway applying
`V1__b2b_initial_schema.sql`, then Hibernate schema validation passing,
then the embedded Tomcat starts on port 8080. Kill the process (`Ctrl-C`)
once you see `Started WisegiftApplication`. This exercises exactly the same
migration path Render staging will run on next deploy, so any Neon-side
surprise (missing extension, permission problem, region mis-config) shows
up here without a Render deploy attempt.

If this step is skipped, step 6's Render deploy is the first place the
migration runs — that is still fine; the migration is idempotent enough
(`CREATE EXTENSION IF NOT EXISTS`, plus fresh empty tables).

### 6. Record the connection string for Render

Spring Data JPA / HikariCP wants the **direct** connection string (long-lived
JDBC pool, not per-request). Neon's pooled endpoint (`-pooler` suffix in the
hostname) uses PgBouncer transaction-mode pooling, which is incompatible with
some JPA features (server-side prepared statements, `LISTEN/NOTIFY`, session
GUCs). Direct connections are the right choice for the Spring Boot app.

Two forms are needed for Render:

1. **JDBC URL** — for `SPRING_DATASOURCE_URL`. Prefix with `jdbc:` and drop
   the `postgresql://user:pass@` credentials (Spring takes them from
   separate env vars):

   ```
   jdbc:postgresql://ep-<compute-id>.eu-central-1.aws.neon.tech/neondb?sslmode=require
   ```

2. **Username + password** — for `SPRING_DATASOURCE_USERNAME` and
   `SPRING_DATASOURCE_PASSWORD`. From the URI in step 3 (the substring
   between `://` and `@` split on `:`).

Feed all three into the corresponding fields in the Render staging service
per `render-staging-env-vars.md`. **Never paste these values into any file
committed to git.**

Suggested pool size: `spring.datasource.hikari.maximum-pool-size=10` on
staging (default is 10, so no override needed unless tuning). Neon's free
tier compute has ~200 max_connections; 10 gives headroom for parallel
Flyway + integration test runs without exhausting the compute.

### 7. Backup / snapshot policy

**Defer.** No merchant data exists yet; nothing to protect. Neon's
point-in-time-restore (PITR) is on by default on paid tiers with a 7-day
window, which is sufficient for staging accidental-drop recovery during the
pre-pilot phase.

Set explicit retention when the first paying merchant lands (production
project, not this one). Not a task for this runbook.

### 8. Branch strategy

**MVP:** single Neon branch (`main`) for staging. No per-PR preview
branches. Rationale — the `wisegift-backend` CI does not currently spawn
ephemeral DBs per PR (Testcontainers covers integration tests locally), so
a preview-branch pipeline has no consumer.

**Post-MVP consideration:** Neon's branch-per-PR pattern is trivial to add
later via a GitHub Action that runs `neonctl branches create` on PR open
and `neonctl branches delete` on PR close. Out of scope for this runbook.

---

## Verification (end-to-end)

The operator confirms staging Neon is ready when **all** of the following
are true:

1. `neonctl projects get <project-id>` returns `region_id: aws-eu-central-1`.
2. `psql "$NEON_DIRECT_URL" -c "SELECT extname FROM pg_extension WHERE extname='vector';"`
   returns one row.
3. The JDBC URL, username, and password have been recorded (out-of-band, not
   in git) and are ready to paste into Render per `render-staging-env-vars.md`.
4. (Optional) A local `SPRING_PROFILES_ACTIVE=staging` run of the backend
   against the Neon URL boots to `Started WisegiftApplication` without Flyway
   errors.

Once all four are true, the Neon side of PR 2's unblock is complete. Proceed
to `render-staging-env-vars.md`.

---

## Rollback / recovery

### The project was created in the wrong region

Delete and recreate. Neon does not support in-place region migration.

```
neonctl projects delete <project-id>
```

Confirm the deletion via `neonctl projects list`. Then re-run step 1 with
the correct `--region-id`. Because no data exists yet, deletion is
recoverable in the strict sense (nothing to restore) but irreversible in
the destructive sense (project ID is gone).

### `CREATE EXTENSION` failed

Re-run step 3. `CREATE EXTENSION IF NOT EXISTS vector` is idempotent — a
retry is safe. If the error is permission-related and the role is confirmed
correct, check the Neon dashboard's Settings → Allowed extensions and
confirm `vector` is on the allowlist for the project (it is on every Neon
project by default at time of writing; escalate to Neon support if not).

### The extension was enabled on a non-`main` branch

Neon branches inherit extensions from their parent branch at fork time.
If the operator accidentally enabled `vector` on a preview branch instead
of `main`:

1. Enable it on `main` per step 3 (using `main`'s connection string).
2. The preview branch already has it — no action needed there.
3. Verify per step 4 against `main`'s connection string.

No data corruption risk; extension state is per-branch.

### The Spring Boot first-boot fails on Flyway V1

If step 5 (or the first Render staging deploy) fails with a Flyway error
mentioning `vector`, `CREATE EXTENSION`, or `type "vector" does not exist`,
step 3 did not land or landed on the wrong branch. Re-run step 3 and step 4
against the branch the staging JDBC URL points at, then retry the boot.

Because `V1__b2b_initial_schema.sql` itself starts with
`CREATE EXTENSION IF NOT EXISTS vector;` (line 6 of the migration, confirmed
on `wisegift-backend` `develop`), the Neon-side extension enable in step 3
is strictly a permission-hedge — if the JDBC role has `CREATE` privilege on
the DB, Flyway will attempt it too. Enabling it explicitly in step 3 removes
one class of first-boot failure mode.

### The whole staging DB got corrupted / needs a clean slate

Delete the project (as above), recreate, re-enable pgvector, update the
Render env vars with the new connection string, and let Flyway re-run V1
on next deploy. No merchant data exists to preserve at MVP.

---

## Follow-ups after this runbook completes

- Hand off the connection-string values to whoever executes
  `render-staging-env-vars.md`.
- Track "Neon staging provisioned" in the `docs/decisions.md` follow-up on
  "Backend PR 1 merged" as closed (append a short note to the follow-ups
  bullet, do not rewrite history).
- Schedule the production-Neon variant of this runbook when the first paying
  merchant is confirmed. Production picks up a Grafana Cloud EU export path
  for `cost_telemetry` per `docs/decisions.md` 2026-09-29 "Architecture open
  questions closed" — that's a separate downstream task, not this runbook.
