# Hooks and packs: backend contract

> **DRAFT: backend contract.** Written 2026-10-16 by `backend-engineer` for `docs/product/hooks-and-packs.md` (itself a draft pending PO review). Nothing here is decided until the PO validates it. The widget (`frontend-engineer`) and the admin implement against this document once it is accepted.
>
> **Inputs:**
> - `hooks-and-packs.md` §2, §3 and §5
> - `decisions.md`:
>   - 2026-10-07 "Per-placement hook copy"
>   - 2026-10-15 "Merchant admin is the single source of truth…"
>   - 2026-10-15 "Hooks per placement and niche packs"
>   - 2026-10-16 M2·3 follow-ups (`tenants.vertical_pack`, V11)
> - `architecture.md` §9.2 and §10.5
> - `wisegift-backend` `origin/develop` at `0fa42bd`
> - `wisegift-widget` `origin/main` at `92e1d46`
>
> **Delivery.** Two backend PRs. PR 1 is storage, copy resolution, bootstrap additions and the copy and trigger admin endpoints (§1, §2, §3.1–3.3, §4.1–4.3, §5, §6). PR 2 is the pack switch preview and apply, the pack intent form and form schema versioning (§3.4, §4.4). PR 2 is blocked on Q-F3 (§7).

---

## 0. Defaults adopted for open PO questions

These follow the spec's recommendation or the orchestrator's instruction. The PO can still overrule them.

| # | Adopted | Effect on this contract |
|---|---|---|
| Q-P1 | The side tab stays the default trigger. The mockup pill is the `pill` mode. | `trigger.mode` defaults to `side_tab` (§2.2). |
| Q-P3 | `gift_finder` reuses the `home_hero` layout. | `gift_finder` has eyebrow, headline and subline (§1.2). |
| Q-P5 | The global `hook_copy` is demoted **below** the pack per-placement headlines. | Headline chain in §1.3. Because every known placement has a pack headline, `hook_copy` no longer reaches any of the five placements. It still feeds the legacy `copy` field, and unknown placements. See §7, Q-C1. |
| Q-P7 | No collapsed state for the inline home card. | Nothing to store. |
| Q-B6 | A new intent form schema version on each pack switch. | §3.4. |

Backend answers to its own questions:

| # | Answer |
|---|---|
| Q-B1 | Evolve `widget_config.placement_hooks` JSONB from `placement → string` to `placement → language → field → string`. V12 converts the old shape. No new copy table (§1.1). |
| Q-B2 | Bootstrap `config` gains `language`, `hook`, `trigger` and `strings` (PR 1), then `intentForm` (PR 2). About 1 KB in PR 1 and under 3 KB with `intentForm` (§3). |
| Q-B3 | Already answered by V11: `tenants.vertical_pack`, not `widget_config`. Pack defaults are code constants (§2). |
| Q-B4 | `collection_header` and `floating` become accepted copy keys. `floating` gets pack defaults. Adding them to `placements_enabled` is **not** in PR 1 (§7, Q-C4). |
| Q-B5 | New append-only table `widget_config_events` (§6.1). |
| Q-B6 | A new `intent_form_schemas` version when a pack switch changes form content (§3.4). |

---

## 1. Storage model

### 1.1 `widget_config.placement_hooks` (evolved, V12)

The shape is three levels deep: placement, then language, then field.

```json
{
  "home_hero": {
    "en": { "eyebrow": "Our picks", "headline": "Shopping for someone?" },
    "es": { "subline": "" }
  },
  "floating": {
    "en": { "trigger_title": "Need a hand?" }
  }
}
```

- **Only merchant edits are stored.** An absent key means "no override", so the field resolves through the chain in §1.3. This is the defaults-layer model in spec §3.1: "an override existing is the edit."
- **The empty string is an override.** For fields that can be hidden, it means "hide this part". Fields that cannot be hidden reject the empty string at write time (§5).
- Languages are `en`, `es` and `pt`. `pt` is European Portuguese, informal "tu" (Q-P6).
- Field keys are snake_case in storage (like `theme_tokens`) and camelCase on the wire.

**V12 conversion of the existing shape.**
- `{"pdp_slot": "Thinking of this as a gift?"}` becomes `{"pdp_slot": {"en": {"headline": "Thinking of this as a gift?"}}}`.
- Blank strings are dropped.
- Entries that are already objects are kept as they are, so the statement is safe to re-run.
- Existing values longer than 90 characters are kept. The old limit was 200, and the spec forbids truncation. The admin shows them over the limit, and the next save has to shorten them.
- Old values were language-less. They are migrated as **EN** only, because the widget has rendered EN only so far. See §7, Q-C2.

### 1.2 Fields per placement

| Placement | Copy fields | Limit (code points) | Can be hidden? |
|---|---|---|---|
| `home_hero` | `eyebrow` · `headline` · `subline` | 24 · 90 · 120 | yes · no · yes |
| `collection_header` | `headline` · `subline` | 90 · 120 | no · yes |
| `pdp_slot` | `headline` · `subline` | 90 · 120 | no · yes |
| `gift_finder` | `eyebrow` · `headline` · `subline` | 24 · 90 · 120 | yes · no · yes |
| `floating` | `eyebrow` · `headline` · `subline` (drawer) | 24 · 90 · 120 | yes · no · yes |
| `floating` | `trigger_title` · `trigger_subtitle` (pill) | 24 · 32 | no · yes |
| `floating` | `side_tab_label` (side tab) | 32 | no |

- Writing a field that a placement does not show is rejected with 422 `unknown_copy_field`. For example, `collection_header` has no eyebrow (spec §2.3).
- **`side_tab_label` is new in this contract.** The spec keeps the side tab's title-only label, "Gift finder" (≤ 32), separate from the pill title (≤ 24). It is pack-independent, with the default "Gift finder", and it replaces the theme-block `trigger_copy` setting.

### 1.3 Resolution chain (per placement × language × field)

| Field | Chain (first non-null wins) |
|---|---|
| `headline` | 1. override `[placement][lang]` → 2. pack default `(pack, placement, lang)` → 3. `widget_config.hook_copy` (non-blank) → 4. generic fallback |
| every other field | 1. override `[placement][lang]` → 2. pack default `(pack, placement, lang)` → 3. none (`null`) |

- **Pack default by language:**
  - EN is final.
  - ES and PT pack copy does not exist yet (spec §3.2 is EN only), so ES and PT fall back to the EN pack default.
  - When the ES and PT translations land, they slot in with no contract change.
- **The pack** is `tenants.vertical_pack` (V11). A tenant that never answered Step 1 has `general`.
- **The General pack replaces the 2026-10-07 code constants** for every tenant (spec §2.2). No pilots exist, so no merchant sees a silent change.
- **Overrides are never shared across languages.** An EN edit never shows on an ES storefront (spec §2.2: "Edits are never auto-translated").
- **An override of `""` resolves to `null`** (hidden). It does not fall through to the pack default.
- **Unknown placement slugs** have no pack default. They resolve the headline through `hook_copy`, then the generic fallback, as today. All other fields are `null`.
- **Generic fallback:** "Looking for a gift, or something for yourself?" (unchanged, EN).

### 1.4 Floating trigger settings (Q-P2: they move to the admin)

Today `floating_hook.liquid` exposes:
- `trigger_copy` (≤ 32, default "Gift finder");
- `trigger_mode`: `side_tab | pill | icon`;
- `trigger_position`: `bottom_right | bottom_left`;
- `trigger_background_color` (default `#111111`);
- `trigger_text_color` (default `#ffffff`).

New typed `widget_config` columns (V12):

| Column | Type | Default | Values |
|---|---|---|---|
| `trigger_mode` | `VARCHAR(16) NOT NULL` | `'side_tab'` | `side_tab`, `pill` (CHECK) |
| `trigger_position` | `VARCHAR(16) NOT NULL` | `'bottom_right'` | `bottom_right`, `bottom_left` (CHECK). The side tab uses only the side: right or left. |
| `trigger_background_color` | `VARCHAR(7) NULL` | `NULL` | `#rgb` / `#rrggbb`. `NULL` means use the accent (spec M3). |
| `trigger_text_color` | `VARCHAR(7) NULL` | `NULL` | `#rgb` / `#rrggbb`. `NULL` means the widget picks white or near-black by contrast against the background (spec §2.5). |

- **`icon` is dropped as a stored mode.** It was a back-compat option for earlier installs, and none exist. The 56 px icon-only button stays as the ≤ 400 px rendering of both modes (spec M10).
- The trigger copy (title, subtitle, side tab label) is per-language copy under `placement_hooks.floating` (§1.2), not trigger settings.
- **After this ships, the theme block drops all five settings** (a widget PR). The Theme App Extension only decides that the embed is on.

---

## 2. Pack defaults (code constants)

`HookCopyDefaults` in `tenant/domain/model`. The values are the spec §3.2 tables verbatim (EN).

| Placement / field | General | Fashion | Home & Decor |
|---|---|---|---|
| `home_hero` eyebrow | Shopping guide | Style guide | Home guide |
| `home_hero` headline | Not sure what to pick? Tell us who it's for. | Find the piece that fits. Tell us who it's for. | Make a space feel finished. Tell us who it's for. |
| `home_hero` subline | A few quick questions, no account needed. | A few quick questions about style and budget. | A few quick questions about style and budget. |
| `collection_header` headline | Too many options? Let's narrow them down. | So many styles. Let's find the right one. | Lots to choose from? Let's find what fits. |
| `collection_header` subline | Tell us who it's for and we'll pick the ones worth a look. | Tell us who you're shopping for and we'll pick the pieces worth a look. | Tell us who it's for and we'll pick the pieces worth a look. |
| `pdp_slot` headline | Buying this for someone? | Buying this for someone? | Buying this for someone's home? |
| `pdp_slot` subline | Tell us a little about them and we'll suggest a few they'd love. | Tell us a little about them and we'll suggest pieces that suit their style. | Tell us a little about them and we'll suggest a few pieces they'd love. |
| `gift_finder` eyebrow | Gift finder | Gift guide | Gift guide |
| `gift_finder` headline | Find a gift they'll actually love. | Gifts they'll actually wear. | Gifts for the home they'll keep for years. |
| `gift_finder` subline | Answer a few quick questions and we'll pick from the whole store. | Tell us about them and we'll pick from our collection. | Housewarming, wedding or just because. Tell us about them and we'll pick from our range. |
| `floating` trigger_title | Need help choosing? | Need a style hand? | Need a hand choosing? |
| `floating` trigger_subtitle | Tell us who it's for | Tell us who it's for | Tell us who it's for |
| `floating` side_tab_label | Gift finder | Gift finder | Gift finder |
| `floating` eyebrow | Shopping guide | Style guide | Home guide |
| `floating` headline | Who are we shopping for today? | Who are we dressing today? | Who are we shopping for today? |
| `floating` subline | A few quick questions, no account needed. | A few quick questions, no account needed. | A few quick questions, no account needed. |

Every default fits the §1.2 limits; a unit test asserts this.

**Fixed strings** (spec §2.1): `FixedHookStrings`, with EN final and ES and PT as **drafts** for the PO. They are delivered in the bootstrap (§3) because spec §2.8 keeps per-locale string tables out of the bundle.
- The keys: `choice.self`, `choice.self.sub`, `choice.gift`, `choice.gift.sub`, `pdp.gift`, `pdp.self`, `reassure`, `error.retry`, `drawer.close`.
- The widget still ships the EN `error.retry` and the EN choice labels, because they are needed when the bootstrap fails.

---

## 3. Session bootstrap: `POST /widget/v1/session`

### 3.1 Request

```json
{ "sessionId": "…", "placement": "home_hero", "locale": "es-ES" }
```

- **`locale`** (new, optional) is the **storefront** locale. In Liquid it is `{{ request.locale.iso_code }}`, passed by the widget as `data-locale`.
- The backend takes the primary subtag, lower-cased: `es-ES` gives `es` and `pt-BR` gives `pt`. If the result is `en`, `es` or `pt`, that is the language. Anything else, or no `locale` at all, gives `en`.
- **`Accept-Language` is deliberately not used.** The copy must match the language the rest of the storefront page is in, not the shopper's browser. An EN-only store must not mix in Spanish hooks for a Spanish browser. One value per page also keeps the behaviour deterministic.
- Older bundles send no `locale` and get `en`, which is today's behaviour. The field is ignored by older backends: Spring's default `FAIL_ON_UNKNOWN_PROPERTIES` is false outside the events API.
- `pt-BR` storefronts receive European Portuguese (Q-P6). See §7, Q-C3.

### 3.2 Response `config` (additions in bold; nothing removed or renamed)

```json
{
  "copy": "…or null",
  "copyForPlacement": "Find a gift they'll actually love.",
  "placementsEnabled": ["home_hero", "pdp_slot", "gift_finder"],
  "themeTokens": "{…}",
  "borderRadiusPx": 12,
  "brandAccentColor": "#6c47ff",
  "language": "en",
  "hook": {
    "eyebrow": "Gift finder",
    "headline": "Find a gift they'll actually love.",
    "subline": "Answer a few quick questions and we'll pick from the whole store."
  },
  "trigger": null,
  "strings": { "choice.self": "For me", "choice.self.sub": "Find what suits you", "…": "…" },
  "intentForm": "PR 2, see §3.4"
}
```

- **`copy`** is unchanged: the raw `hook_copy`, or `null`.
- **`copyForPlacement`** is now always equal to `hook.headline`, in the resolved language. Older bundles therefore render the pack headline: same field, same meaning ("the resolved headline", spec §2.2).
- **`language`** is the resolved language. The widget uses it for `lang` attributes and for its own EN fallback decision.
- **`hook`**:
  - It is always present.
  - `headline` is never null or blank.
  - `eyebrow` and `subline` are `null` when hidden or not applicable. The widget hides the part when the value is `null`, and must never substitute its own default for a `null`.
- **`trigger`** is **only present for `placement = "floating"`**. For every other placement it is `null`. Shape:

  ```json
  { "mode": "side_tab", "position": "bottom_right",
    "backgroundColor": null, "textColor": null,
    "title": "Need help choosing?", "subtitle": "Tell us who it's for",
    "sideTabLabel": "Gift finder" }
  ```

  - `backgroundColor: null` means use the accent chain.
  - `textColor: null` means on-accent text chosen automatically by contrast.
  - `subtitle: null` means hidden.
- **`strings`** holds the fixed strings for `language` (§2).
- All values are untrusted plain text, so the widget renders them as text nodes (§5).

### 3.3 Timing caveat (widget-owned, flagged)

The bootstrap runs on first interaction today. Two problems follow:
- The hook and the floating trigger render **before** any bootstrap, so they would flash the bundle defaults until the first click.
- The trigger is clicked before a bootstrap exists at all.

See §7, Q-C5 for the options. This contract does not change when the bootstrap happens.

### 3.4 `config.intentForm` (PR 2; blocked on Q-F3)

```json
{
  "schemaVersion": 3,
  "gift": {
    "relationship": { "label": "Who's it for?", "visibility": "required",
      "options": [ { "value": "partner", "label": "My partner" }, { "value": "friend", "label": "A friend" } ] },
    "occasion": { "label": "What's the occasion?", "visibility": "required",
      "options": [ { "value": "birthday", "label": "Birthday" } ] },
    "budget": { "label": "Budget", "visibility": "required",
      "bands": [ { "min": 0, "max": 30 }, { "min": 30, "max": 60 }, { "min": 60, "max": 120 }, { "min": 120, "max": null } ] },
    "interests": { "label": "Anything about their style?", "helperText": "e.g. minimal, earthy colours, loves linen, sporty", "visibility": "optional" }
  },
  "self": {
    "budget": { "label": "What's your budget?", "visibility": "required", "bands": [ "…" ] },
    "interests": { "label": "What are you looking for?", "helperText": "e.g. …", "visibility": "optional" }
  }
}
```

- **`options` is ordered** (display order) and contains **only visible options.** For example, Fashion omits `housewarming`.
- **Wire values do not change per pack.** The `value` set must be the one the widget sends today and the ranker sees (Q-R2). That set is exactly what Q-F3 must settle:
  - the live widget sends `holiday` and `client`;
  - the spec lists `christmas`, `wedding` and `other`, and has no `client`.
- **`bands`:**
  - They carry numbers only, in shop currency.
  - `max: null` means "and over". The widget sends `budget_max` absent for an open band, as today.
  - The widget formats the labels with the storefront currency. The backend does not know the currency symbol.
- **`visibility`** is the merchant's `intent_form_fields` toggle. Packs never change it (spec §3.3).
- **`schemaVersion`** is the active `intent_form_schemas.version`. The widget **must** send it as `intentFormSchemaVersion` on `intent_submitted`. The field already exists in the events API, and the widget does not send it yet.
- **Versioning rule (Q-B6):**
  - The first bootstrap that needs a schema writes version 1 from the tenant's current pack.
  - Any pack change whose resolved form content differs from the active version inserts `max(version) + 1` with `active = true` and deactivates the previous one, in the same transaction.
  - `schema_json` stores the resolved form plus `{ "pack": "fashion" }`, so analysis can map a version to the pack and labels shown.
  - A pack change that only touches hook copy does not bump the version. Today every pack pair differs in form content, so in practice every switch bumps it.
- Labels are EN for now. ES and PT come with the translation pass.

---

## 4. Admin endpoints

All endpoints are under `/admin/v1`, authenticated, and scoped to the tenant from `AdminTenantResolver` (cookie or bearer principal) only. They never take a tenant from the path, body or query. Errors use `{ code, message }`. Validation failures return **422**, as the existing widget-config controller does. Every response sends `Cache-Control: no-store`.

### 4.1 Hook copy (PR 1)

**`GET /admin/v1/widget/copy?language=en`**

`language` is optional, defaults to `en`, and accepts `en | es | pt`. Anything else is 422 `invalid_language`.

```json
{
  "pack": "fashion",
  "language": "en",
  "globalHookCopy": null,
  "placements": [
    {
      "placement": "home_hero",
      "fields": [
        { "field": "eyebrow", "value": "Style guide", "source": "pack_default",
          "edited": false, "packDefault": "Style guide", "maxLength": 24, "hideable": true },
        { "field": "headline", "value": "Shopping for someone?", "source": "override",
          "edited": true, "packDefault": "Find the piece that fits. Tell us who it's for.", "maxLength": 90, "hideable": false }
      ]
    }
  ],
  "updatedAt": "2026-10-16T10:00:00Z"
}
```

- All five placements are always listed, in this order: `home_hero`, `collection_header`, `pdp_slot`, `gift_finder`, `floating`. Filtering by enabled placements is admin-side.
- **`value`** is what the storefront shows for that language. `null` means hidden.
- **`source`** is one of `override | pack_default | global_hook_copy | generic_fallback`. The last two can only appear for `headline`, and only if a pack default is missing.
- **`edited`** is `source == "override"` and drives the "Edited" badge. When it is `false`, the admin shows the muted "[Pack] default" label.
- **`packDefault`** is the value "Reset to [Pack] default" would produce.

**`PUT /admin/v1/widget/copy/{language}`**: set overrides, batched across placements.

```json
{ "placements": { "home_hero": { "headline": "Shopping for someone?", "subline": "" },
                  "floating":  { "triggerTitle": "Need a hand?" } } }
```

- The wire field names are `eyebrow`, `headline`, `subline`, `triggerTitle`, `triggerSubtitle` and `sideTabLabel`.
- An absent or `null` field leaves that field unchanged. A string sets the override; `""` hides the field, where it can be hidden (§1.2).
- At least one field is required.
- The response is the `GET` body for `{language}`. Fields not sent are untouched.
- A batch can also apply "the same headline everywhere": send `headline` for every placement.

**`PUT /admin/v1/widget/copy/{language}/reset`**: per-field "Reset to [Pack] default".

```json
{ "fields": [ { "placement": "home_hero", "field": "headline" } ] }
```

- Clears those overrides in `{language}` only.
- Clearing a field that has no override is a no-op.
- The response is the `GET` body.

**`PUT /admin/v1/widget/copy/reset-all`**: "Reset all copy to [Pack] defaults".
- It clears **every** override in **every** language.
- It does not touch `hook_copy`; "Clear default hook copy" stays `PUT /admin/v1/widget/config { "hookCopy": "" }`.
- It does not touch the trigger settings or the theme.
- The confirmation's list of fields comes from the `GET` bodies, where `edited: true`.
- The response is the `GET` body for `en`.

**Legacy surface (kept for the current admin build):**
- `WidgetConfigResponse.placementHooks` stays a `placement → string` map. It now holds the **EN headline overrides** only.
- `PUT /admin/v1/widget/config { placementHooks: { … } }` still takes strings. Each key sets the **EN headline** override; a blank value clears it.
- **Semantics change:** the update is now a merge, where it used to be a full-map replace. Keys not sent are left alone, so the call cannot wipe eyebrows, sublines or other languages.
- The current admin always sends all three placements, so its behaviour is unchanged.
- The per-value limit drops from 200 to 90 (spec §2.1). The admin's counter must follow.
- New code should use §4.1. The legacy fields can be removed once the admin moves.

### 4.2 Trigger settings (PR 1)

**`GET /admin/v1/widget/trigger`**

```json
{ "mode": "side_tab", "position": "bottom_right", "backgroundColor": null, "textColor": null }
```

**`PUT /admin/v1/widget/trigger`**: a partial update.
- An absent or `null` field leaves that setting unchanged.
- For colours, `""` clears the colour back to `null` (inherit).
- `mode` must be `side_tab | pill`, `position` must be `bottom_right | bottom_left`, and colours must be `#rgb | #rrggbb`. Anything else is 422 `invalid_trigger_setting`.
- The response is the `GET` body.

The trigger copy lives in §4.1, under `floating`.

### 4.3 Store type (exists, #86) and its interaction

- `PUT /admin/v1/tenant/store-type { pack, vertical? }` stays the **single write path** for `vertical_pack`.
- From PR 1, unedited copy resolves from the new pack immediately: it is the defaults layer.
- PR 1 also appends a `pack` row to `widget_config_events` when the pack actually changes (§6.1).
- Onboarding Step 1 keeps using this endpoint. Once PR 2 exists, onboarding should call `/pack/apply` with `applySuggestedLook: true` instead, so the suggested look is applied (spec §3.4).

### 4.4 Pack switch preview and apply (PR 2)

**`POST /admin/v1/widget/pack/preview`** with `{ "pack": "fashion" }` is side-effect free (CQRS query over POST because it has a body):

```json
{
  "from": "general", "to": "fashion",
  "willChange": [
    { "placement": "home_hero", "language": "en", "field": "eyebrow", "before": "Shopping guide", "after": "Style guide" }
  ],
  "editsKept": [
    { "placement": "home_hero", "language": "en", "field": "headline", "value": "Shopping for someone?", "newPackDefault": "Find the piece that fits. Tell us who it's for." }
  ],
  "formChanges": [
    { "path": "gift.budget.bands", "before": "…", "after": "…" }
  ],
  "suggestedLook": { "border_radius_px": "2", "spacing_scale": "roomy", "card_shadow": "none" }
}
```

- **`willChange`** lists fields with no override whose resolved value differs between the two packs, for every language. `before == after` entries are omitted.
- **`editsKept`** lists every override, each with the new pack's default. The admin renders the "Replace with [Pack] default" checkbox per row, unchecked by default.
- **`formChanges`** lists labels, options and bands that differ. `visibility` is never listed, because a pack never changes it.
- **`suggestedLook`** carries shape tokens only. It is `{}` for General ("no suggestion", spec §3.2).

**`POST /admin/v1/widget/pack/apply`**

```json
{ "pack": "fashion", "vertical": null,
  "replaceEdits": [ { "placement": "home_hero", "language": "en", "field": "headline" } ],
  "applySuggestedLook": false }
```

One transaction does all of the following:
1. Sets the pack through the same command as `PUT /tenant/store-type` (same validation: `vertical` is only allowed with `general`).
2. Clears exactly the listed overrides.
3. If `applySuggestedLook` is set, writes the shape tokens as `theme_tokens` overrides, exactly like a Theming preset.
4. Bumps the form schema version (§3.4).
5. Appends a `pack` audit row.

The response:

```json
{ "pack": "fashion", "vertical": null, "intentFormSchemaVersion": 4,
  "copy": "GET /widget/copy?language=en body",
  "undo": { "pack": "general", "vertical": "other",
            "restoreOverrides": [ { "placement": "home_hero", "language": "en", "field": "headline", "value": "…" } ],
            "restoreThemeTokens": { "border_radius_px": "8", "spacing_scale": null, "card_shadow": null } } }
```

- **Undo** (the toast) replays `undo`:
  1. `POST /pack/apply` with the previous pack, `replaceEdits: []` and `applySuggestedLook: false`;
  2. `PUT /widget/copy/{lang}` with `restoreOverrides`;
  3. `PUT /widget/theme-tokens` with `restoreThemeTokens`, where `null` clears a token.
- The replay goes through the normal validated write paths: there is no trusted snapshot endpoint.
- Undo creates another form schema version and another audit row. That is acceptable, and the readouts see both.

---

## 5. Validation and limits

### 5.1 Per field

| Field | Max (Unicode code points) | Empty `""` |
|---|---|---|
| `eyebrow` | 24 | allowed: hides it |
| `headline` | 90 | rejected (422) |
| `subline` | 120 | allowed: hides it |
| `triggerTitle` | 24 | rejected (422) |
| `triggerSubtitle` | 32 | allowed: hides it |
| `sideTabLabel` | 32 | rejected (422) |
| global `hookCopy` (unchanged) | 200, measured in UTF-16 units as today; the admin warns above 90 | allowed: clears it |

### 5.2 Rules
- **Length is counted in code points.** JS `[...s].length` gives the same count, so the admin counter and the backend agree. Over the limit is 422 `copy_too_long`. **Nothing is ever truncated**, on write or on read.
- **Leading and trailing whitespace is trimmed** before the length check and before storing. A value that is only whitespace counts as `""`.
- **Plain text only.** Rejected with 422 `invalid_copy_characters`:
  - control characters (U+0000–U+001F, U+007F–U+009F), which includes line breaks, because every field is single-line;
  - bidi override and isolate characters (U+202A–U+202E, U+2066–U+2069), which allow text spoofing.

  `<`, `>` and `&` are allowed literally. They are **never** HTML-escaped on storage, because that would double-escape. They are **never** rendered as HTML anywhere: the widget uses text nodes, and the admin must not use `dangerouslySetInnerHTML`.
- An unknown placement, an unknown field for the placement, or an unknown language gives 422 with code `unknown_placement`, `unknown_copy_field` or `invalid_language`.

---

## 6. Events and measurement

- **No new event types and no new event fields** (spec §2.7). Confirmed against `WidgetEventController`: `placement` already accepts every slug, and `intentFormSchemaVersion` already exists.
- The widget must start sending `intentFormSchemaVersion` on `intent_submitted` from `config.intentForm.schemaVersion` (PR 2). The field is accepted today and simply not sent.
- The form schema version bumps on a pack change that changes form content (§3.4, Q-B6).
- **Holdout symmetry:** the bootstrap returns identical `hook`, `trigger`, `strings` and `intentForm` for exposed and holdout sessions. `isHoldout` is computed independently and never read by the copy resolution.

### 6.1 Change timestamps (Q-B5)

New append-only table `widget_config_events` (V12):

| Column | Type | Notes |
|---|---|---|
| `id` | `BIGSERIAL PK` | |
| `tenant_id` | `UUID NOT NULL REFERENCES tenants` | |
| `kind` | `VARCHAR(16) NOT NULL` | `copy`, `copy_reset`, `trigger` or `pack` (CHECK) |
| `placement` | `VARCHAR(32) NULL` | set for `copy` and per-field `copy_reset`; null for reset-all, `trigger` and `pack` |
| `language` | `VARCHAR(2) NULL` | set for `copy` and per-field `copy_reset` |
| `occurred_at` | `TIMESTAMPTZ NOT NULL` | |

- The index is `(tenant_id, occurred_at)`. The table is purged with the tenant: it is added to `TenantPurgeRepository.TENANT_SCOPED_TABLES`.
- **Copy text is not stored.** Readouts need the change dates per placement, not the content. Content history, if wanted, is a later decision.
- **Rows written:**
  - one `copy` row per placement and language touched by a PUT;
  - one `copy_reset` row per placement and language a per-field reset actually changed, and one row with no placement or language for reset-all;
  - a write that changes nothing (same value again, resetting an unedited field) writes no row;
  - one `trigger` row per trigger PUT;
  - one `pack` row when `vertical_pack` changes.
- The legacy `PUT /widget/config` writes a `copy` row per placement whose EN headline changed.
- **The readout surfacing is not built here.** That is Q-R4, with recommendations-specialist.

### 6.2 Propagation
- The backend has **no config cache.** Every bootstrap reads `widget_config` and `tenants`, so changes are visible on the next bootstrap.
- The spec's "within 5 minutes (existing config cache)" depends on the widget's own caching, if any.

---

## 7. Open questions

Blocking items are marked.

| # | Question | Blocks | Owner |
|---|---|---|---|
| **Q-F3** | **Occasion and relationship wire values.** The live widget sends occasion `birthday, anniversary, just_because, holiday, baby, housewarming` and relationship `…, client, …`. Spec §3.2 lists Christmas, Wedding and Other, and no Client. The backend accepts any string, so nothing fails, but the pack option tables need one canonical value set. **Recommendation:** adopt the spec's 8 occasions as the canonical slugs (`christmas`, `wedding`, `other` replace `holiday`), and keep `client` for General only. `recommendations-specialist` must confirm the ranker and the cache signature are fine with that one-time value change (Q-R2). | **PR 2** (intent form in bootstrap) | frontend-engineer + backend-engineer, then PO |
| **Q-C1** | **Global `hook_copy` under Q-P5 "demoted below pack headlines".** Every known placement has a pack headline, so `hook_copy` can no longer reach a storefront. It only feeds the legacy `copy` field and unknown placements. The staging tenant's "Wise Assistant" stops showing after deploy. **Recommendation:** the admin replaces "Default hook copy" with a "Use the same headline everywhere" action that writes per-placement overrides through the batched PUT (§4.1), and drops the global field later. Alternative: keep the spec §3.3 order (global above pack). That is a one-line change in the resolver. | PO confirmation before admin work | PO |
| Q-C2 | V12 migrates legacy per-placement strings as **EN** overrides only. ES and PT storefronts get pack defaults for those placements. Accept? No pilots exist; staging only. | No | PO |
| Q-C3 | `pt-BR` storefronts get European Portuguese copy. Accept for MVP? | No | PO |
| **Q-C5** | **The hook and the trigger render before the bootstrap.** **Recommendation:** for the floating trigger, the backend mirrors the trigger settings and per-language trigger copy into the existing app-data metafield set (`wisegift.*`, like `wisegift.live`) so the Liquid block renders them with no request (PR 3, Shopify-only, behind the platform port). For inline hooks, the widget bootstraps on viewport entry, which is consent-safe without a `sessionId` (degraded mode). This needs a decision with `frontend-engineer` (relates to Q-F1, because the bootstrap moment must not change when `widget_shown` fires). | Widget rendering quality, not PR 1 | frontend-engineer + backend-engineer |
| Q-C4 | `placements_enabled` and the Placements screen still list 3 placements. Adding `collection_header` (needs Q-F4's new app block) and `floating` (an app embed, whose on/off state the backend cannot see) is deferred. The Configuration screen can show all five copy tabs regardless. | No | PO + frontend-engineer |
| Q-C6 | The `side_tab_label` field (§1.2) is new: the spec keeps the side tab's own label but lists no admin field for it. Confirm it is merchant-editable and pack-independent ("Gift finder"). | No | product-analyst |
| Q-C7 | Dropping `trigger_mode = icon` (§1.4). | No | PO |
| Q-P6 | The ES and PT translations of the pack copy (§2) and the fixed strings (drafts). | No (falls back to EN) | PO |
| Q-R2 / Q-R3 | The ranker and the cache signature use wire values; the effect of pack budget bands on the cache. | PR 2 | recommendations-specialist |
