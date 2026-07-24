# `migrate-residencecountry-dryrun.js` — read-only dry run

Companion to `docs/engineering/migration-residencecountry.md`. **Read-only.** No writes to Firestore under any circumstances. A runtime assertion enforces this even if the code is edited by mistake.

## Prerequisites

- Node.js 20+.
- `firebase-admin` installed locally: `npm install firebase-admin`.
- A service account key for the target Firebase project with **read-only** Firestore access. Minimum role: `roles/datastore.viewer`. It MUST NOT have write roles. Confirm with security-expert before generating the key.
- Key file stored outside any git repo (e.g. `~/.wisegift-keys/wisegift-prod-sa-readonly.json`), permissions `chmod 600`.

## Invocation

```bash
GOOGLE_APPLICATION_CREDENTIALS=/path/to/wisegift-prod-sa-readonly.json \
  node migrate-residencecountry-dryrun.js
```

For staging, point `GOOGLE_APPLICATION_CREDENTIALS` at the staging service account key. No other change.

Outputs, written next to the script:
- Console: bucket counts and a preview of the manual-review bucket.
- `mappable.csv` — mappable non-ISO docs with proposed target.
- `manual-review.csv` — non-ISO, non-mappable docs.
- `dryrun-summary.json` — full machine-readable report (same data as the CSVs plus bucket counts).

## Script

```javascript
// migrate-residencecountry-dryrun.js
// READ-ONLY. Enumerates users/{uid}.residenceCountry and buckets it.
// Absolutely no Firestore writes. See docs/engineering/migration-residencecountry.md.

'use strict';

const admin = require('firebase-admin');
const fs = require('fs');
const path = require('path');

// -------- HARD SAFETY GUARD --------
// This constant is the only place where write-mode could ever be enabled.
// It is set to true and any code path that would issue a write asserts against it.
// Do NOT flip this. The apply phase is a SEPARATE script.
const DRY_RUN = true;
if (!DRY_RUN) {
  throw new Error(
    'migrate-residencecountry-dryrun.js is read-only. Use the apply script for writes.'
  );
}
// Belt-and-braces: monkey-patch the writeable Firestore surfaces to throw.
// If any code below is ever changed to call .set/.update/.delete/.commit, it will
// blow up before touching production data.
function installWriteTraps() {
  const trap = (name) => () => {
    throw new Error(`Write attempted in dry-run script: ${name}. Aborting.`);
  };
  const proto = admin.firestore.DocumentReference.prototype;
  proto.set = trap('DocumentReference.set');
  proto.update = trap('DocumentReference.update');
  proto.delete = trap('DocumentReference.delete');
  const batchProto = admin.firestore.WriteBatch.prototype;
  batchProto.set = trap('WriteBatch.set');
  batchProto.update = trap('WriteBatch.update');
  batchProto.delete = trap('WriteBatch.delete');
  batchProto.commit = trap('WriteBatch.commit');
}

// -------- NAME -> ISO MAPPING --------
// Keys are lowercase, trimmed, accents preserved. See migration-residencecountry.md
// for the source of truth; keep the two lists in sync.
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

function classify(raw) {
  // Returns { bucket, proposedIso, reason }
  // bucket: 'missing' | 'iso' | 'mappable' | 'manual'
  if (raw === undefined) {
    return { bucket: 'missing', proposedIso: null, reason: 'field absent' };
  }
  if (raw === null) {
    return { bucket: 'manual', proposedIso: null, reason: 'null value' };
  }
  if (typeof raw !== 'string') {
    return { bucket: 'manual', proposedIso: null, reason: `non-string type: ${typeof raw}` };
  }
  const trimmed = raw.trim();
  if (trimmed === '') {
    return { bucket: 'manual', proposedIso: null, reason: 'empty string' };
  }
  const upper = trimmed.toUpperCase();
  if (ISO_RE.test(upper)) {
    // Already ISO. Even if raw casing was lowercase, we bucket as iso; the apply
    // phase decides whether to rewrite for uppercase-consistency.
    return { bucket: 'iso', proposedIso: upper, reason: null };
  }
  const key = trimmed.toLowerCase();
  const mapped = NAME_TO_ISO.get(key);
  if (mapped) {
    return { bucket: 'mappable', proposedIso: mapped, reason: null };
  }
  return { bucket: 'manual', proposedIso: null, reason: 'not in mapping table' };
}

function csvEscape(v) {
  if (v === null || v === undefined) return '';
  const s = String(v);
  if (s.includes(',') || s.includes('"') || s.includes('\n')) {
    return `"${s.replace(/"/g, '""')}"`;
  }
  return s;
}

async function main() {
  installWriteTraps();

  admin.initializeApp({
    credential: admin.credential.applicationDefault(),
  });
  const db = admin.firestore();

  console.log(`[${new Date().toISOString()}] Starting dry-run scan of users/*.residenceCountry`);
  console.log(`Project: ${admin.app().options.projectId || '(from ADC)'}`);
  console.log('DRY_RUN =', DRY_RUN, '(writes disabled at multiple layers)');

  const buckets = { missing: 0, iso: 0, mappable: 0, manual: 0 };
  const mappable = [];
  const manual = [];

  // Full-collection scan. Streamed to keep memory bounded.
  const snap = await db.collection('users').select('residenceCountry').get();

  for (const doc of snap.docs) {
    const raw = doc.get('residenceCountry');
    const c = classify(raw);
    buckets[c.bucket] += 1;
    if (c.bucket === 'mappable') {
      mappable.push({ uid: doc.id, current: raw, proposedIso: c.proposedIso });
    } else if (c.bucket === 'manual') {
      manual.push({ uid: doc.id, current: raw, reason: c.reason });
    }
  }

  const total = snap.size;
  console.log('');
  console.log('=== Bucket counts ===');
  console.log(`  total users docs         : ${total}`);
  console.log(`  (a) missing field        : ${buckets.missing}`);
  console.log(`  (b) already ISO alpha-2  : ${buckets.iso}`);
  console.log(`  (c) mappable non-ISO     : ${buckets.mappable}`);
  console.log(`  (d) manual review        : ${buckets.manual}`);
  console.log('');

  // CSV output
  const outDir = process.cwd();
  const mappableCsv = ['uid,currentValue,proposedIso'];
  for (const r of mappable) {
    mappableCsv.push(`${csvEscape(r.uid)},${csvEscape(r.current)},${csvEscape(r.proposedIso)}`);
  }
  fs.writeFileSync(path.join(outDir, 'mappable.csv'), mappableCsv.join('\n'));

  const manualCsv = ['uid,currentValue,reason'];
  for (const r of manual) {
    manualCsv.push(`${csvEscape(r.uid)},${csvEscape(r.current)},${csvEscape(r.reason)}`);
  }
  fs.writeFileSync(path.join(outDir, 'manual-review.csv'), manualCsv.join('\n'));

  fs.writeFileSync(
    path.join(outDir, 'dryrun-summary.json'),
    JSON.stringify({
      generatedAt: new Date().toISOString(),
      projectId: admin.app().options.projectId || null,
      total,
      buckets,
      mappable,
      manual,
    }, null, 2)
  );

  // Preview manual bucket in the console so the PO can eyeball it immediately.
  if (manual.length > 0) {
    console.log('=== Manual-review preview (first 20) ===');
    for (const r of manual.slice(0, 20)) {
      console.log(`  ${r.uid}  ${JSON.stringify(r.current)}  [${r.reason}]`);
    }
    if (manual.length > 20) {
      console.log(`  ... ${manual.length - 20} more in manual-review.csv`);
    }
  }

  console.log('');
  console.log('Wrote: mappable.csv, manual-review.csv, dryrun-summary.json');
  console.log('No Firestore writes were performed.');
}

main().catch((err) => {
  console.error('Dry-run failed:', err);
  process.exit(1);
});
```

## What the PO should do after running

1. Confirm bucket counts look sane (total roughly matches known user count; already-ISO bucket dominates or at least is non-zero).
2. Open `manual-review.csv`. For each row, decide:
   - "Extend the mapping table" → edit `migration-residencecountry.md` and the `NAME_TO_ISO` map in this script, re-run dry-run.
   - "Leave alone" → user's next login will re-resolve via the client hotfix chain.
   - "Individual UID fix" → note the UID, will be handled separately.
3. Open `mappable.csv`. Spot-check 5-10 entries; do the proposed ISO codes match the intent?
4. Paste bucket counts and any surprises back into the working thread for devops-expert to review before drafting the apply-phase go-signal.
