# `migrate-residencecountry-apply.js` — write phase

> **DO NOT RUN — REQUIRES DRY-RUN REVIEW AND EXPLICIT PO APPROVAL.**
>
> This script writes to production Firestore. It must not be executed until:
> 1. The Flutter `CountryService.resolve()` hotfix is deployed to prod (`main` in `wisegift_flutter`).
> 2. `migrate-residencecountry-dryrun.js` has been run against the target project and the report reviewed by the PO.
> 3. The PO has given explicit, written approval in the working thread for this specific project (`wisegift` prod first, then separately for `wisegift-staging`).
>
> If any of those is not true, stop.

Companion to `docs/engineering/migration-residencecountry.md`.

## Prerequisites

- Node.js 20+, `firebase-admin` installed.
- A **distinct** service account key from the dry-run one, with write scope: `roles/datastore.user` (or a custom role restricted to `datastore.entities.get`, `datastore.entities.update`, `datastore.entities.list` on the `users` collection). Confirm with security-expert. Key file `chmod 600`, stored outside any git repo.
- The `mappable.csv` from the most recent dry-run available for cross-check (the script does its own classification but the CSV is used to sanity-check the write count matches what was reviewed).

## Invocation

```bash
GOOGLE_APPLICATION_CREDENTIALS=/path/to/wisegift-prod-sa-writer.json \
  node migrate-residencecountry-apply.js
```

Add `--staging` after the script name to acknowledge you are targeting staging (the script cross-checks the resolved project ID against the flag and refuses if there is a mismatch; prevents pointing the prod key at a script invocation labelled staging or vice versa).

Outputs, written next to the script:
- Console: batch-by-batch progress, then a final summary.
- `apply-results.json` — per-UID success/failure + counts + list of failed UIDs for retry.

## Script

```javascript
// migrate-residencecountry-apply.js
// WRITE PHASE. Do not run without PO approval. See migration-residencecountry.md.

'use strict';

const admin = require('firebase-admin');
const fs = require('fs');
const path = require('path');

// -------- SAFETY CONFIG --------
// Batches are capped at 400 ops each. Firestore's hard limit is 500, and each
// updated doc uses ~1 op (set with merge or update), so 400 leaves headroom for
// future field additions and keeps commits well under the 10 MiB batch payload.
const BATCH_SIZE = 400;

// Require an explicit env acknowledgement flag before doing any writes. Prevents
// "I meant to run the dry-run" accidents.
if (process.env.WISEGIFT_MIGRATION_ACK !== 'apply-residencecountry') {
  console.error('Refusing to run: set WISEGIFT_MIGRATION_ACK=apply-residencecountry to proceed.');
  console.error('This should only be set after PO written approval.');
  process.exit(2);
}

// Environment guard: --staging must match the project ID.
const args = new Set(process.argv.slice(2));
const targetIsStaging = args.has('--staging');

// -------- NAME -> ISO MAPPING (must match dry-run) --------
const NAME_TO_ISO = new Map(Object.entries({
  'spain': 'ES',
  'españa': 'ES',
  'espana': 'ES',
  'portugal': 'PT',
  'united kingdom': 'GB',
  'uk': 'GB',
  'great britain': 'GB',
  'reino unido': 'GB',
  'inglaterra': 'GB',
  'united states': 'US',
  'united states of america': 'US',
  'usa': 'US',
  'u.s.a.': 'US',
  'estados unidos': 'US',
  'estados unidos de américa': 'US',
  'estados unidos de america': 'US',
  'france': 'FR',
  'francia': 'FR',
  'germany': 'DE',
  'alemania': 'DE',
  'deutschland': 'DE',
  'italy': 'IT',
  'italia': 'IT',
  'colombia': 'CO',
  'mexico': 'MX',
  'méxico': 'MX',
  'brazil': 'BR',
  'brasil': 'BR',
  'argentina': 'AR',
  'chile': 'CL',
  'netherlands': 'NL',
  'holanda': 'NL',
  'países bajos': 'NL',
  'paises bajos': 'NL',
  'belgium': 'BE',
  'bélgica': 'BE',
  'belgica': 'BE',
  'ireland': 'IE',
  'irlanda': 'IE',
  'canada': 'CA',
  'canadá': 'CA',
}));

const ISO_RE = /^[A-Z]{2}$/;

// Idempotency + safety: decide whether this doc should be touched, and what to write.
// Returns null if the doc should be skipped, or { targetIso, previousValue } if it
// should be written. previousValue captures the raw pre-migration value for backup.
function decide(docData) {
  if (docData === undefined) return null;
  const raw = docData.residenceCountry;

  // Idempotency guard: already migrated in a previous run.
  if (Object.prototype.hasOwnProperty.call(docData, 'residenceCountry_previous')) {
    return null;
  }

  // Missing field: skip. The client resolve chain handles new users.
  if (raw === undefined) return null;

  // Non-string types are manual review, not automated apply. Skip.
  if (raw === null || typeof raw !== 'string') return null;

  const trimmed = raw.trim();
  if (trimmed === '') return null;

  const upper = trimmed.toUpperCase();
  if (ISO_RE.test(upper)) {
    // Already an ISO alpha-2 code. Skip when raw is already uppercase (fully
    // idempotent); rewrite for uppercase-consistency when it's lowercase.
    if (raw === upper) return null;
    return { targetIso: upper, previousValue: raw };
  }

  const key = trimmed.toLowerCase();
  const mapped = NAME_TO_ISO.get(key);
  if (!mapped) return null; // manual review; skip in automated apply

  return { targetIso: mapped, previousValue: raw };
}

async function main() {
  admin.initializeApp({
    credential: admin.credential.applicationDefault(),
  });
  const db = admin.firestore();
  const projectId = admin.app().options.projectId || process.env.GCLOUD_PROJECT || null;

  // Guard the --staging flag against the actual project ID.
  if (projectId) {
    const looksLikeStaging = projectId.includes('staging');
    if (targetIsStaging && !looksLikeStaging) {
      console.error(`Refusing: --staging flag set but project ID is "${projectId}" (does not look like staging).`);
      process.exit(2);
    }
    if (!targetIsStaging && looksLikeStaging) {
      console.error(`Refusing: no --staging flag but project ID is "${projectId}" (looks like staging).`);
      console.error('Add --staging to acknowledge, or point GOOGLE_APPLICATION_CREDENTIALS at prod.');
      process.exit(2);
    }
  }

  console.log(`[${new Date().toISOString()}] Starting APPLY phase.`);
  console.log(`Project: ${projectId || '(unknown)'}`);
  console.log(`Target: ${targetIsStaging ? 'STAGING' : 'PRODUCTION'}`);
  console.log(`Batch size: ${BATCH_SIZE}`);
  console.log('');

  const snap = await db.collection('users').select('residenceCountry', 'residenceCountry_previous').get();

  const toWrite = []; // [{ uid, targetIso, previousValue }]
  let skipped = 0;

  for (const doc of snap.docs) {
    const decision = decide(doc.data());
    if (decision === null) {
      skipped += 1;
      continue;
    }
    toWrite.push({ uid: doc.id, ...decision });
  }

  console.log(`Scanned ${snap.size} docs. To write: ${toWrite.length}. Skipped (idempotent/manual): ${skipped}.`);

  const results = {
    startedAt: new Date().toISOString(),
    projectId,
    target: targetIsStaging ? 'staging' : 'production',
    scanned: snap.size,
    toWrite: toWrite.length,
    skipped,
    successCount: 0,
    failureCount: 0,
    failures: [], // [{ uid, error }]
    successes: [], // [{ uid, from, to }]
  };

  // Chunk into batches of BATCH_SIZE and commit each independently. On batch failure,
  // log and continue — do not abort the run. The idempotency guard means a retry run
  // will pick up whatever this run missed.
  for (let i = 0; i < toWrite.length; i += BATCH_SIZE) {
    const chunk = toWrite.slice(i, i + BATCH_SIZE);
    const batchNo = Math.floor(i / BATCH_SIZE) + 1;
    const totalBatches = Math.ceil(toWrite.length / BATCH_SIZE);
    const batch = db.batch();
    for (const item of chunk) {
      const ref = db.collection('users').doc(item.uid);
      // update() + FieldValue.serverTimestamp() is intentionally not used for the
      // migration timestamp field — we do not add one, keeping the change surface
      // minimal. residenceCountry_previous is the sole new field.
      batch.update(ref, {
        residenceCountry: item.targetIso,
        residenceCountry_previous: item.previousValue,
      });
    }
    try {
      await batch.commit();
      for (const item of chunk) {
        results.successes.push({ uid: item.uid, from: item.previousValue, to: item.targetIso });
        results.successCount += 1;
      }
      console.log(`  [${batchNo}/${totalBatches}] committed ${chunk.length} updates`);
    } catch (err) {
      // Batch-level failure: mark every UID in the chunk as failed and continue.
      // Firestore batches are atomic, so if commit throws, none of these were written.
      for (const item of chunk) {
        results.failures.push({ uid: item.uid, error: err.message });
        results.failureCount += 1;
      }
      console.error(`  [${batchNo}/${totalBatches}] FAILED: ${err.message}`);
    }
  }

  results.finishedAt = new Date().toISOString();

  console.log('');
  console.log('=== Apply summary ===');
  console.log(`  successes: ${results.successCount}`);
  console.log(`  failures : ${results.failureCount}`);
  if (results.failureCount > 0) {
    console.log('  failed UIDs (retry candidates):');
    for (const f of results.failures.slice(0, 50)) {
      console.log(`    ${f.uid}  ${f.error}`);
    }
    if (results.failures.length > 50) {
      console.log(`    ... ${results.failures.length - 50} more in apply-results.json`);
    }
  }

  fs.writeFileSync(
    path.join(process.cwd(), 'apply-results.json'),
    JSON.stringify(results, null, 2)
  );

  // -------- VERIFICATION BLOCK --------
  // Re-scan and count non-ISO docs. Expect zero remaining (excluding known manual-
  // review docs the PO chose to leave alone).
  console.log('');
  console.log('=== Verification pass ===');
  const verifySnap = await db.collection('users').select('residenceCountry').get();
  let stillNonIso = 0;
  const stillNonIsoSample = [];
  for (const doc of verifySnap.docs) {
    const raw = doc.get('residenceCountry');
    if (raw === undefined || raw === null) continue;
    if (typeof raw !== 'string') { stillNonIso += 1; if (stillNonIsoSample.length < 20) stillNonIsoSample.push({ uid: doc.id, raw }); continue; }
    const trimmed = raw.trim();
    if (trimmed === '') continue;
    if (!ISO_RE.test(trimmed.toUpperCase())) {
      stillNonIso += 1;
      if (stillNonIsoSample.length < 20) stillNonIsoSample.push({ uid: doc.id, raw });
    }
  }
  console.log(`  docs still non-ISO: ${stillNonIso} (expected: manual-review bucket size only)`);
  if (stillNonIso > 0) {
    console.log('  sample:');
    for (const s of stillNonIsoSample) console.log(`    ${s.uid}  ${JSON.stringify(s.raw)}`);
  }

  console.log('');
  console.log('Wrote: apply-results.json');
  console.log('Post-run action: run the dry-run script again as an independent verification and paste its bucket counts back to the working thread.');
}

main().catch((err) => {
  console.error('Apply run failed:', err);
  process.exit(1);
});
```

## Post-run actions

1. Paste the apply summary (successes, failures, verification count) into the working thread.
2. If failures > 0, review failed UIDs and re-run the script — the idempotency guard will skip the successes and retry the failures.
3. Run `migrate-residencecountry-dryrun.js` again as an independent verification. Mappable bucket should be empty. Manual-review bucket should match the "knowingly left alone" set from step 3 of the migration plan.
4. Monitor prod for ~24h. Confirm `/api/v1/products/search` is returning results for affected users. Backend-expert to confirm no Redis cache invalidation is needed (see coordination notes in `migration-residencecountry.md`).
5. Only after prod is validated, repeat the whole flow (dry-run → PO review → approval → apply → verification) against `wisegift-staging`.

## Rollback (separate script, not included here)

If a rollback is needed, a separate `migrate-residencecountry-rollback.js` will read all docs with `residenceCountry_previous`, restore that value into `residenceCountry`, and delete the `_previous` field. Same PO-approval gate applies. Draft on request.
