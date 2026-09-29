# Migration — Normalize `users/{uid}.residenceCountry` to ISO 3166-1 alpha-2

Status: **draft — dry-run only, no writes yet.**

## Problem

Legacy user documents in the `wisegift` prod Firebase project hold `residenceCountry` as a display name (e.g. `"Spain"`, `"España"`) rather than the ISO 3166-1 alpha-2 code the backend expects. The Flutter `CountryService.resolve()` returns the stored value verbatim, so the display name flows into `/api/v1/products/search` and matches zero ISO-coded rows.

Root-cause and full context: `docs/decisions.md` entry **2026-07-10 — Diagnosed prod "product search returns empty"; root cause is legacy Firestore residenceCountry display names**.

Fix has two parts. Part 1 (Flutter client hotfix, `CountryService.resolve()` defensive normalization) is owned by frontend-expert and MUST ship first so new writes are ISO-clean. Part 2 (this document) normalizes the legacy Firestore data.

## Scope

- Field: `users/{uid}.residenceCountry`.
- Project: `wisegift` (prod) first. `wisegift-staging` gets the same treatment after prod is validated.
- No other fields, no other collections, no other projects.
- API contract stays ISO 3166-1 alpha-2. We are not making the backend tolerant of display names — see the 2026-07-10 decision.

## Detection strategy

A doc is a candidate if `residenceCountry` exists and its raw value, uppercased and trimmed, does not match `^[A-Z]{2}$`. That means:

- `null`, `""`, whitespace → candidate (bucketed as manual review; the resolve chain will fall through to IP → locale → `"US"` anyway, so we do not overwrite these).
- Any non-string type (number, boolean, map, array) → candidate, manual review.
- 3+ letter strings (`"ESP"`, `"Spain"`, `"España"`) → candidate, mapping attempted.
- 2-letter strings that are not both alpha (`"E1"`, `"1S"`) → candidate, manual review.
- 2-letter alpha strings in any case (`"es"`, `"Es"`, `"ES"`) → normalize to uppercase in the apply phase (still counts as candidate for the dry-run report so we see them).

Firestore has no server-side regex filter cheap enough at this scale, so the dry-run reads all `users/{uid}` docs and filters client-side. At current user counts this is a full-collection scan; acceptable one-off cost.

## Name → ISO mapping table

Case-insensitive lookup on the trimmed value. Left-hand side is normalized to lowercase before comparison; accents are preserved (both `españa` and `espana` are keys).

| Input value (case-insensitive) | ISO alpha-2 |
|---|---|
| `spain`, `españa`, `espana` | `ES` |
| `portugal` | `PT` |
| `united kingdom`, `uk`, `great britain`, `reino unido`, `inglaterra` | `GB` |
| `united states`, `united states of america`, `usa`, `u.s.a.`, `estados unidos`, `estados unidos de américa`, `estados unidos de america` | `US` |
| `france`, `francia` | `FR` |
| `germany`, `alemania`, `deutschland` | `DE` |
| `italy`, `italia` | `IT` |
| `colombia` | `CO` |
| `mexico`, `méxico` | `MX` |
| `brazil`, `brasil` | `BR` |
| `argentina` | `AR` |
| `chile` | `CL` |
| `netherlands`, `holanda`, `países bajos`, `paises bajos` | `NL` |
| `belgium`, `bélgica`, `belgica` | `BE` |
| `ireland`, `irlanda` | `IE` |
| `canada`, `canadá` | `CA` |

Additions beyond the minimum requested list cover countries the app has explicitly launched into (US, ES, CO — see 2026-06-30 decision) and common Iberian/EU/LATAM neighbours where a Spanish or Portuguese display name is plausible. Anything not on this table lands in the manual-review bucket.

Two-letter alpha values that are already valid ISO codes but in lowercase (e.g. `es`) are treated as "already ISO" for bucketing purposes but the apply phase still rewrites them uppercase — this is not a name mapping.

## Buckets (dry-run output)

For each doc with `residenceCountry` present:

- **(a) missing field** — key not set; excluded from the mapping report entirely. Counted only.
- **(b) already ISO** — value matches `^[A-Z]{2}$` after `trim().toUpperCase()`. Excluded from mapping report; the apply phase still touches these if the raw value is not already uppercase, but that is an internal detail.
- **(c) mappable non-ISO** — value maps via the table above. Emitted to CSV/JSON with proposed target.
- **(d) unmappable / manual review** — non-null, non-empty, non-ISO, and not in the mapping table. Emitted to CSV/JSON with `proposedIso = null`. PO decides case-by-case: extend the table, mark for individual UID fix, or leave alone and let the client resolve chain handle it on next login.

## Rollback plan

On write day, every updated doc gets a new field `residenceCountry_previous` in the same batch write that updates `residenceCountry`. Value is the exact pre-migration value (including type — string, null, number, whatever). A separate rollback script reads all docs with `residenceCountry_previous`, writes the previous value back into `residenceCountry`, and deletes the `_previous` field. Rollback is itself idempotent and read-only-until-explicit-approval, same gating as the apply phase.

`residenceCountry_previous` is retained in production for at least 30 days after apply-phase completion before any cleanup script is even drafted. Cleanup is a separate, later, human-gated decision.

## Idempotency

The apply script MUST skip a doc when either:

1. `residenceCountry` already matches `^[A-Z]{2}$` (already ISO, correct case), OR
2. `residenceCountry_previous` already exists (already migrated in a previous run).

This makes the apply phase safe to re-run after partial failure, network interruption, or PO-directed retry of a subset.

## Phases (human-gated)

1. **Frontend hotfix ships first.** `CountryService.resolve()` defensive normalization is live on `main` in `wisegift_flutter`. New writes are ISO-clean. This is a prerequisite — we do not touch legacy data until new writes have stopped adding to the problem.
2. **Dry-run.** PO runs `migrate-residencecountry-dryrun.js` against prod with a read-only service account. Script produces bucket counts + `mappable.csv` + `manual-review.csv`.
3. **PO reviews report.** Manual-review bucket is triaged. Mapping table is extended if needed (edit this doc, re-run dry-run). Repeat until the manual-review bucket is either empty or explicitly acknowledged as "leave as-is".
4. **Explicit approval.** PO says "run the apply phase on prod." Written approval in the working thread; no verbal-only sign-off.
5. **Apply.** PO runs `migrate-residencecountry-apply.js` against prod with a write-scoped service account. Script writes in batches of 400 with `residenceCountry_previous` backup.
6. **Verification.** PO re-runs the dry-run script. Expect the mappable bucket to be empty and the already-ISO bucket to have absorbed everything that was previously mappable. Any residual manual-review docs are the ones knowingly left alone in step 3.
7. **Staging.** Repeat steps 2–6 against `wisegift-staging`. Same scripts, different `GOOGLE_APPLICATION_CREDENTIALS`.

Each step is a hard stop. Devops-expert does not proceed to the next step without explicit PO approval.

## Coordination

- **frontend-expert** — `CountryService.resolve()` client hotfix must be deployed to prod (`main` in `wisegift_flutter`) before step 2. Confirm with the PO that this shipped.
- **backend-expert** — Confirm whether any prod cache holds `residenceCountry`-derived state that must be invalidated post-apply. Two known candidates:
  - Redis cache on `gift-recommendation` (`RecipientProfileHasher` includes `residenceCountry` in the SHA-256 key — see 2026-07-01 decision). New keys will be generated automatically once user profiles are corrected, so a stale entry only lingers until its TTL expires. No manual invalidation needed unless TTL is very long; confirm TTL value.
  - No known search-backend cache; `/api/v1/products/search` reads live from PostgreSQL. Confirm.
- **catalog-data-engineer** — Not on the critical path for this migration, but note the parallel `amazon-products.yml` `country: ES` re-poisoning issue flagged in the 2026-07-10 decision follow-ups. That is a separate fix.
- **security-expert** — Confirm the service account used for dry-run has read-only Firestore scope and the write-phase service account is a distinct principal, not the same key with elevated roles.
- **privacy-legal-advisor** — `residenceCountry` is user PII. Writing a `_previous` backup field is retention of user data for rollback purposes; this is proportionate and time-limited (30 days minimum, cleanup pending). No user-facing notification required for a data-quality correction that does not change the semantic value of the field.

## Ambiguity / open questions

- Field type in the raw docs: `residenceCountry` is untyped in Firestore. The dry-run must sample a real prod doc distribution to confirm we are not missing edge cases (null, empty string, number, map). The bucketing logic in the script handles unknown types by routing them to manual review — but PO should eyeball the first dry-run output before we commit to the mapping table.
- Whether to overwrite lowercase-but-otherwise-valid ISO codes (`"es"` → `"ES"`) in the apply phase. Current plan: yes, silently, because the client hotfix will produce uppercase from now on and mixed case in prod data is a bug regardless. Flag for PO confirmation.
- Whether to write `residenceCountry_previous` when the only change is uppercasing a 2-letter code. Current plan: yes, for consistency and rollback symmetry. Minor storage cost.

## Files

- `docs/engineering/scripts/migrate-residencecountry-dryrun.md` — draft read-only dry-run script.
- `docs/engineering/scripts/migrate-residencecountry-apply.md` — draft write script, DO NOT RUN until steps 1–4 above are complete.
