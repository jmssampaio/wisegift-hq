# QA — Open Questions Validation
**Date:** 2026-06-30

## Decision 1 — Follow/Unfollow removed

| Check | Status | Evidence |
|---|---|---|
| No `newFollower` in `notifications_page.dart` switch statements | PASS | File has no `newFollower` case; switch only handles `eventInvite` and `default`. Line 114–119 and 123–129 and 132–137 confirmed. |
| No "Social Graph" feature section in `spec.md` | PASS | No `Social Graph` heading exists in the document. |
| No follower count on User Profile spec | PASS | The "User Profile (Viewing Others)" section (lines 316–328) contains no mention of follower counts. |
| No `newFollower` in Notifications spec acceptance criteria | FAIL | `spec.md` line 334 — the Notifications feature **User Story** still reads "so that I know when **someone follows me** or invites me to an event". This is spec copy, not code, but it asserts a follower notification is expected behaviour. The acceptance criteria body (lines 337–343) does not list `newFollower` as a supported type, which is correct. |
| No `users/{uid}/following` or `users/{uid}/followers` paths in `architecture.md` | PASS | Firestore Collections table (lines 159–166) lists no follower or following sub-collections. Confirmed clean. |
| Decision recorded in `decisions.md` | PASS | `decisions.md` lines 8–12: "2026-06-30 — Remove follow/unfollow social graph feature". Entry is present with context, decision, and maker. |

**Verdict:** INCOMPLETE

**Remaining issues:**

1. `spec.md` line 334 — Notifications feature User Story still contains "so that I know when **someone follows me** or invites me to an event." This sentence must be updated to remove the follower premise (e.g. "so that I know when I am invited to an event").

2. `spec.md` lines 364 and 372 (Open Questions section) — Open Question 1 still describes `newFollower` as implemented and visible on profiles, and Open Question 5 still refers to the "follower graph" as part of account deletion. These must be removed or rewritten now that the decision is final.

3. `lib/domain/enums/notification_type.dart` line 2 — `newFollower` remains as an enum value. The decision log allows domain-layer remnants to remain if not surfaced in the UI. Confirmed: `newFollower` is not referenced in any presentation or service file. This is **acceptable per the stated criteria** but is noted for future clean-up.

4. `lib/l10n/app_localizations_en.dart` line 838, `app_localizations_es.dart` line 844, `app_localizations.dart` lines 1619–1623 — `notificationFollowerBody` and `followers` l10n keys remain defined but are not used in any presentation screen (grep confirmed). Acceptable per criteria, but orphaned strings add maintenance debt.

---

## Decision 2 — AI Curator country

| Check | Status | Evidence |
|---|---|---|
| No `_GeoLimitationBanner` class in `gift_agent_page.dart` | PASS | File contains no `_GeoLimitationBanner` definition or reference. Confirmed by full file read. |
| No hardcoded `'ES'` country string in `gift_agent_page.dart` | PASS | `grep` found zero `'ES'` occurrences in `gift_agent_page.dart`. |
| Country read from Firestore profile via `_countryService.resolve(storedCountry: _buyerCountry)` | PASS | `gift_agent_page.dart` lines 48–56: `_resolveCountry()` reads `AppUser.fieldResidenceCountry` from Firestore via `_usersService.getUserProfile`, stores it in `_buyerCountry`, then calls `_countryService.resolve(storedCountry: stored)`. Line 111 calls `resolve(storedCountry: _buyerCountry)` again at recommendation time. Pattern matches requirement. |
| `CountryService` fallback chain correct (profile → IP → locale → US) | PASS | `country_service.dart` lines 9–28: docstring and implementation confirm priority order. Returns `storedCountry` if non-null, then calls `https://api.country.is/` for IP geolocation, then falls back to `platformDispatcher.locale.countryCode ?? 'US'`. Correct. |
| Spec criterion added for country resolution | PASS | `spec.md` line 168: "The recommendation request uses the logged-in user's `residenceCountry` from their Firestore profile. For guests, the country is resolved via IP geolocation, falling back to device locale, then `US`." Criterion is present in the AI Curator feature section. |
| Decision recorded in `decisions.md` | PASS | `decisions.md` lines 14–18: "2026-06-30 — AI Curator country uses user's stored residenceCountry; expand beyond Spain". Entry is present with context, decision, and maker. |

**Verdict:** INCOMPLETE — `gift_agent_page.dart` itself is clean, but three other files retain the Spain-only hardcoding that the decision was supposed to eliminate.

**Remaining issues:**

1. `lib/data/datasources/remote/recommendation_engine_micro_api.dart` line 28 — fallback is `appUser.residenceCountry ?? 'ES'`. If country resolution fails upstream and `residenceCountry` is null, the API call silently defaults to Spain. This should be `?? 'US'` to match the documented fallback chain, or the null case should be handled before reaching this layer.

2. `lib/presentation/widgets/home/curated_collections_view.dart` lines 250–251 and 713–745 — `_GeoLimitationBanner` **still exists** in this widget file and is conditionally rendered when `_resolvedCountry != 'ES'`. The banner was removed from `gift_agent_page.dart` but was not removed from the home screen curated collections view. This directly contradicts "geo-limitation banner must be gone."

3. `lib/presentation/widgets/agenda/calendar_view.dart` line 75 — `residenceCountry: 'ES'` is hardcoded when constructing a temporary `AppUser` for birthday recommendation generation. This bypasses the `CountryService` entirely for recommendations triggered from the Agenda screen.

4. `spec.md` line 366 — Open Question 2 still describes the old state: "The Gift Agent page passes `residenceCountry: 'ES'` unconditionally." This question is now partially resolved (gift_agent_page.dart is fixed) but should be removed or marked resolved, especially since the hardcoding persists in two other locations listed above.
