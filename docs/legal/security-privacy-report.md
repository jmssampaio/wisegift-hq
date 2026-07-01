# WiseGift — Security & Privacy Report

**Last updated:** 2026-07-01 (H-01, H-02, H-03 resolved)
**Scope:** Flutter client (`wisegift_flutter`), Spring backend (`wisegift-backend`), Firestore rules
**Authors:** security-expert, privacy-legal-advisor

Status legend: 🔴 OPEN · 🟡 IN PROGRESS · ✅ RESOLVED

---

## CRITICAL

| ID | Finding | Area | Status | Resolution |
|---|---|---|---|---|
| C-01 | `POST /api/v1/gifts/recommendations` was fully unauthenticated despite receiving PII (recipient name, birthday, gender, interests). Any actor on the internet could call it. | Backend | ✅ RESOLVED 2026-07-01 | Firebase JWT required via `FirebaseTokenFilter` + `SecurityConfig` mirroring the catalog module pattern. |
| C-02 | `deleteAccount()` only deleted the Firebase Auth credential. All Firestore sub-collections, Firebase Storage files, and Redis cache remained. Violates GDPR Art. 17 (right to erasure) and App Store/Play Store account deletion requirements. | Backend + Flutter | ✅ RESOLVED 2026-07-01 | `DELETE /api/v1/users/me` endpoint added to catalog module. `UserDeletionService` cascades deletion across all Firestore collections and Storage. Flutter `deleteAccount()` now calls backend first. |
| C-03 | No privacy policy link, no terms of service, no consent checkbox at any point in the registration flow. Violates GDPR Art. 13, CCPA, and App Store/Play Store mandatory requirements. | Flutter | ✅ RESOLVED 2026-07-01 | Consent checkbox added to registration step 3. Submit button disabled until accepted. Links to `https://wisegift.app/privacy` and `https://wisegift.app/terms` (placeholder URLs — real documents still needed). |

---

## HIGH

| ID | Finding | Area | Status | Resolution |
|---|---|---|---|---|
| H-01 | Firestore rule `allow read: if true` on `users/{userId}` exposes name, email, birthday, gender, interests, and country to any unauthenticated caller. | Firestore | ✅ RESOLVED 2026-07-01 | Changed to `allow read: if request.auth != null`. Sub-collection rules are independent. Public catalog rules (giftCollections, collection_stats) intentionally remain permitAll. |
| H-02 | Recommendation cache key is `name + eventType + maxBudget + country` (no user UID). Two users with the same name can receive each other's cached recommendations, computed from one user's private profile. | Backend | ✅ RESOLVED 2026-07-01 | Replaced raw concatenation with SHA-256 hash of all 8 RecipientProfile fields via `RecipientProfileHasher`. Interests sorted before hashing for order-independence. |
| H-03 | Third-party contact data (device contacts: name + birthday) stored in Firestore without those individuals' consent. GDPR household exemption does not apply to a commercial app syncing contacts to cloud servers. | Flutter + Firestore | ✅ RESOLVED 2026-07-01 | Disclosure notice added above import button (EN + ES). Bulk-delete already covered by `UserDeletionService` (contacts and private_birthdays sub-collections). |
| H-04 | Recipient PII (name, birthday, gender, interests) sent to OpenAI in an LLM prompt without disclosure to users. No DPA with OpenAI confirmed. No GDPR Art. 46 transfer mechanism documented. | Backend | 🟡 IN PROGRESS | **Legal action required by Product Owner:** sign OpenAI DPA and document Art. 46 transfer mechanism. Add sub-processor disclosure to privacy policy. Code improvement (log redaction) tracked as L-05. |

---

## MEDIUM

| ID | Finding | Area | Status | Resolution |
|---|---|---|---|---|
| M-01 | No age gate at registration. COPPA (under-13) and GDPR Art. 8 (under-16) protections not enforced. Any person of any age can create an account. | Flutter | 🔴 OPEN | Add minimum birth year validation at registration step 2. Show minimum age requirement on screen. |
| M-02 | `InternalRequestGuard` fails open: if `internal.token` property is unset, all admin/sync endpoints (`/api/v1/admin/**`, `/api/v1/collections/generate`, `/api/v1/products/imports/sync`) are fully accessible. No startup warning. | Backend | 🔴 OPEN | Replace `if (expectedToken.isBlank()) return;` with `@PostConstruct` assertion that throws on blank token in non-local profiles. |
| M-03 | Prompt injection: user name and interests inserted into LLM system message without sanitisation or length limits. Impact bounded by `LLMResponseParser` but pattern is fragile. | Backend | 🔴 OPEN | Move user-supplied data to the user turn with an "untrusted" label. Add length/character-set validation on `name` and `interests` before reaching `RecipientProfileFormatter`. |
| M-04 | IP geolocation call to `api.country.is` transmits user IP to a third-party service with no published DPA, before any registration or consent is shown. | Flutter | 🔴 OPEN | Replace with device-locale fallback first, or proxy through backend to avoid third-party IP disclosure. Disclose in privacy policy if retained. |
| M-05 | Firestore `followers`/`following` sub-collection rules still allow any authenticated user to write to any other user's social graph, even though the follow feature was removed from the app. | Firestore | 🔴 OPEN | Lock down these rules or remove the sub-collections entirely from `firestore.rules`. |
| M-06 | `collection_stats` Firestore collection allows write by any authenticated user with no ownership check. Stats can be manipulated to inflate popularity rankings. | Firestore | 🔴 OPEN | Add `allow write: if request.auth.uid == resource.data.creatorId` or equivalent ownership check. |
| M-07 | Occasion guests can update any field in the occasion document (not just their RSVP status). No field-level restriction in Firestore rules. | Firestore | 🔴 OPEN | Restrict guest writes to RSVP-related fields only using Firestore field-mask rules. |
| M-08 | `phoneNumber` collected at registration but no feature uses it (no SMS, 2FA, or phone-based matching found). Superfluous under GDPR Art. 5(1)(c) data minimisation principle. | Flutter | 🔴 OPEN | Remove phone number collection from registration, or document its planned use and disclose in privacy policy. |
| M-09 | Consent checkbox is UI-only. No `consentAcceptedAt` timestamp or policy version written to Firestore. GDPR Art. 7 requires the data controller to demonstrate consent was given. | Flutter + Backend | 🔴 OPEN | Write `consentAcceptedAt: Timestamp` and `consentVersion: String` to `users/{uid}` on account creation. |

---

## LOW

| ID | Finding | Area | Status | Resolution |
|---|---|---|---|---|
| L-01 | `FirebaseTokenFilter` (both modules) does not return 401 when `Authorization` header is absent — delegates to Spring Security's `.authenticated()` rule. Functionally equivalent but fragile if filter is reused outside a secured chain. | Backend | 🔴 OPEN | Add explicit 401 return in the filter when the header is missing. |
| L-02 | `UserDeletionService` only deletes `profile_photo/{uid}.jpg` and `wishlist_banner/{uid}.jpg`. Any future Storage paths tied to the user would create a silent GDPR erasure gap. | Backend | 🔴 OPEN | Maintain a list of Storage path patterns in `UserDeletionService`. Enforce a policy that any new Storage write must register its path pattern there. |
| L-03 | RevenueCat `logOut()` silently skipped on web with no log entry during account deletion. No deletion audit trail for web users. | Flutter | 🔴 OPEN | Log the skip explicitly. If RevenueCat web is ever initialised, update the guard. |
| L-04 | `noGiftsYet` l10n key duplicated in both `app_en.arb` and `app_es.arb` (lines 357 and 373). Duplicate JSON keys cause silent overwrites. | Flutter | 🔴 OPEN | Remove the duplicate key from both files. |
| L-05 | Backend error logs include `recipient.name()` in the failure message (`"Failed to generate gift ideas for " + recipient.name()`). If logs are retained or shipped to a logging service, they constitute an undisclosed PII store. | Backend | 🔴 OPEN | Replace with a redacted log: `"Failed to generate gift ideas for [REDACTED]"` or log only the event type and error code. |
| L-06 | Swagger UI (`/swagger-ui/**`, `/v3/api-docs/**`) is `permitAll` in production. API schema is publicly browsable. | Backend | 🔴 OPEN | Gate Swagger behind an environment flag (`spring.profiles.active != prod`) or require authentication. |
| L-07 | `WebConfig.java` CORS bean in gift-recommendation module is dead code — superseded by `SecurityConfig`'s `CorsFilter`. Confusing to have two CORS configurations. | Backend | 🔴 OPEN | Delete `WebConfig.java` from the gift-recommendation module. |

---

## UNKNOWNS — Require Direct Investigation

| ID | Item | Owner |
|---|---|---|
| U-01 | Firebase / Google DPA and Standard Contractual Clauses (SCCs) for EU-to-US transfers — has this been signed in Firebase console? | Product Owner / Legal |
| U-02 | OpenAI Data Processing Agreement — does one exist? | Product Owner / Legal |
| U-03 | RevenueCat DPA — does one exist? | Product Owner / Legal |
| U-04 | `favouriteBrands` field exists in `AppUser` entity but no collection UI was found. Is it actively collected anywhere? | Frontend |
| U-05 | Redis access controls — is authentication and encryption at rest enabled on the Redis instance? | DevOps |
| U-06 | Backend application logs — are they retained, rotated, shipped to a third-party logging service? If so, they constitute an undisclosed PII store (see L-05). | DevOps |
| U-07 | `internal.token` property — is it set in the production deployment environment? | DevOps |
