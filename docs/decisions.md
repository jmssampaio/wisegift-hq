# Decision Log
(append entries; never rewrite history)
## YYYY-MM-DD — <decision>
- Context:
- Decision:
- Made by:

## 2026-06-30 — Remove follow/unfollow social graph feature
- Context: A follow/unfollow system was partially developed (follower counts on profiles, newFollower notification type, Firestore sub-collections). Product owner confirmed users do not have followers in the app.
- Decision: Remove all follower-related UI and spec references. The block/report moderation flow is retained. Firestore follower sub-collections may still exist on the backend but are no longer written to by the app.
- Made by: Product Owner

## 2026-06-30 — AI Curator country uses user's stored residenceCountry; expand beyond Spain
- Context: The app was Spain-only. The Gift Agent showed a geo-limitation banner for users outside Spain. The app is rolling out to USA and Colombia.
- Decision: Remove the geo-limitation banner. The Gift Agent already reads residenceCountry from the user's Firestore profile and falls back to IP geolocation → device locale → 'US'. No country is hardcoded. Supported markets: ES, US, CO (and any country with affiliate catalog coverage going forward).
- Made by: Product Owner

## 2026-06-30 — AI gift recommendations triggered only from Agenda; persisted per occasion
- Context: Two entry points existed for AI recommendations — Occasion Detail ("GET AI RECOMMENDATIONS" button) and Calendar view (star icon on event card). Results from the detail page were already persisted to the occasion wishlist; results from the calendar were ephemeral. Product owner wants a single, simplified flow and cost control (one AI call per occasion).
- Decision: Remove the generate button from Occasion Detail (keep read-only display of persisted AI picks). Calendar is the sole trigger. On first request, results are persisted as WishlistItem objects (source: ai) on the occasion. Once persisted, the calendar button is permanently disabled for that occasion. Registered-user country from Firestore profile and budget from UserSettings are used for the call; guests fall back to IP geolocation.
- Made by: Product Owner

## 2026-06-30 — Rejected DDD rename of "Occasion" to "Event"
- Context: Product owner evaluated renaming the core aggregate "Occasion" to "Event" for ubiquitous language alignment. Backend expert assessed domain fit, naming conflicts, and blast radius.
- Decision: Do not rename. "Occasion" is more domain-precise than "Event" for a gift-giving milestone. "Event" is maximally overloaded in the Java/Spring/Spring Modulith stack (ApplicationEvent, ApplicationEventPublisher, Modulith transactional events, Kafka). The production Firestore collection `events/{uid}/events/{eventId}` would require a live data migration with no tooling. Counterproposal: if consistency is desired, rename the Firestore path from `events/` to `occasions/` in a future backend cleanup, not the other way around. "Occasion" remains the canonical ubiquitous language term.
- Made by: Product Owner (advised by backend-expert, frontend-expert, functional-analyst)

## 2026-06-30 — Added Flutter Integration Notes to architecture.md
- Context: Flutter dev needs a concise reference for auth, the Firestore-to-Spring recommendation handoff, URL namespace rules, country filtering, and the availability-check pattern.
- Decision: Appended `## 8. Flutter Integration Notes` to docs/engineering/architecture.md covering those five topics. Also flagged that `POST /api/v1/gifts/recommendations` is auth-free despite carrying personal PII — backend expert should assess adding Firebase JWT requirement.
- Made by: frontend-expert

## 2026-07-01 — Cascading user account deletion endpoint added to catalog module
- Context: GDPR and App Store guidelines require a mechanism for users to delete all their personal data. The catalog module already owns Firebase Admin SDK integration (FirebaseAuth, Firestore, Storage).
- Decision: Added `DELETE /api/v1/users/me` (204 No Content) to the catalog module. `rr` deletes data in a strictly ordered sequence: (1) Firestore sub-collections under `users/{uid}` (contacts, private_birthdays, savedCollections, blocked_users, notifications, followers, following) in batches of 400; (2) `occasions` docs where `creatorId == uid`; (3) `giftCollections` docs where `userId == uid` with their `products` sub-collection deleted first; (4) `wishlists` docs where `userId == uid`; (5) `users/{uid}` document; (6) Storage files `profile_photo/{uid}.jpg` and `wishlist_banner/{uid}.jpg` (not-found errors swallowed); (7) Firebase Auth record last (token invalidation). `FirebaseConfig` updated to set `storageBucket` from new `firebase.storage-bucket` property (default `wisegift.appspot.com`). `SecurityConfig` has an explicit rule for the endpoint; it was already covered by `anyRequest().authenticated()` but added explicitly for visibility. Compiled: 52 source files, BUILD SUCCESS.
- Made by: backend-expert

## 2026-07-01 — Fixed cache-key collision in gift-recommendation module (HIGH security)
- Context: The `@Cacheable` key on `GiftRecommendationRagEngine.getRecommendations` was `name + eventType + maxBudget + residenceCountry`. Two users with the same name, country, budget, and event type but different ages, genders, or interests shared one cache entry — the first user's personalised results were returned to the second.
- Decision: Replaced raw field concatenation with a SHA-256 hex digest of all eight profile fields (`name`, `eventType`, `maxBudget`, `residenceCountry`, `relationship`, `gender`, `birthday`, `interests`). Interests list is sorted before joining so element order does not produce different keys for identical profiles. The hasher lives in new class `RecipientProfileHasher` (`com.wisegift.recommendation.application.service`). Cache TTL and Redis config are unchanged. Two users gifting an identical recipient profile still share a cache entry as intended — the key is profile-based, not requestor-based. No PII is stored in the cache key. Compiled: 22 source files, BUILD SUCCESS.
- Made by: backend-expert

## 2026-07-01 — Extracted `user` module; promoted Firebase infrastructure to `shared` module
- Context: `UserController` and `UserDeletionService` were living inside the `catalog` module, which owns the product/collection domain — a clear domain boundary violation. `FirebaseConfig` and `FirebaseTokenFilter` were copy-pasted verbatim across catalog and gift-recommendation, with a minor behavioural difference (catalog returned HTTP 503 when Firebase was uninitialised; gift-recommendation silently skipped validation). Three separate Spring Boot services each carrying their own copy of auth infrastructure created maintenance risk.
- Decision: Created `shared` module (`com.wisegift:shared:1.0.0`, packaging=jar, no spring-boot-maven-plugin repackaging) containing `FirebaseConfig` and `FirebaseTokenFilter` under `com.wisegift.shared.infrastructure.security`. The canonical behaviour adopted is catalog's (return HTTP 503 if Firebase SDK is not initialised, rather than silently passing the request). Deleted catalog's `FirebaseConfig.java` and `FirebaseTokenFilter.java`; deleted gift-recommendation's copies. Both modules now declare `shared` as a compile dependency and their `SecurityConfig` classes import from `com.wisegift.shared.infrastructure.security`. Created new `user` module (`com.wisegift:user:1.0.0`, port 8082) with `@SpringBootApplication` class `UserApp`, `UserDeletionService` under `com.wisegift.user.application.services`, `UserController` under `com.wisegift.user.infrastructure.in.web`, and a dedicated `SecurityConfig` that permits only Swagger (permitAll) and `DELETE /api/v1/users/me` (authenticated). Removed the now-irrelevant `/api/v1/users/me` rule from catalog's SecurityConfig. Root pom.xml `<modules>` reordered so `shared` builds before its consumers. All four modules compile: `mvn compile` from root, BUILD SUCCESS (5 artifacts in 2.8 s).
- Made by: backend-expert

## 2026-06-30 — Firebase JWT authentication added to gift-recommendation module
- Context: `POST /api/v1/gifts/recommendations` was fully unauthenticated despite receiving PII (recipient profile). The Flutter client already sends `Authorization: Bearer <firebase-id-token>` on every call. The catalog module had a working `FirebaseTokenFilter` + `SecurityConfig` + `FirebaseConfig` pattern.
- Decision: Mirrored the catalog security pattern exactly into the gift-recommendation module. Added `spring-boot-starter-security` and `firebase-admin:9.4.1` to gift-recommendation pom.xml. Created `FirebaseConfig`, `FirebaseTokenFilter`, and `SecurityConfig` under `com.wisegift.recommendation.infrastructure.{config,security}`. Authorization rules: `POST /api/v1/gifts/recommendations` requires authentication; `POST /web/v1/gifts/collection-products` (internal service-to-service) and Swagger remain permitAll. The existing `WebConfig.java` CORS bean is superseded by SecurityConfig's `CorsFilter` — it is now dead code but was not deleted to minimise blast radius; recommend removing it in a follow-up cleanup. Compiled successfully: `mvn compile -pl gift-recommendation` from backend root, 21 source files, BUILD SUCCESS.
- Made by: backend-expert

## 2026-07-01 — Refactored `user` module to hexagonal architecture
- Context: The `user` module had a single `UserDeletionService` that mixed Firebase SDK calls (Firestore, Storage, Auth) directly inside the application layer, violating the dependency rule. Infrastructure concerns were leaking into application code with no port abstraction.
- Decision: Introduced full ports-and-adapters structure. Domain layer (`domain/model/UserId.java`, `domain/port/out/{UserDataRepository,UserStoragePort,UserAuthPort}.java`) contains zero framework or Firebase imports. Application layer (`application/service/DeleteUserAccountService.java`) orchestrates the three ports with Lombok `@RequiredArgsConstructor`; deletion order is enforced by code (Firestore data → Storage files → Auth record last). Three `@Component` adapters in `infrastructure/out/firebase/` carry all Firebase SDK calls: `FirestoreUserDataAdapter` implements all Firestore batching logic extracted verbatim from the old service; `FirebaseStorageAdapter` swallows not-found errors silently; `FirebaseAuthAdapter` calls `FirebaseAuth.getInstance().deleteUser(uid)`. `UserController` updated to inject `DeleteUserAccountService` and convert `Authentication.getPrincipal()` String to `UserId` before calling the service; HTTP contract unchanged (`DELETE /api/v1/users/me` → 204). Old `UserDeletionService.java` and its `application/services/` package deleted. Compiled: 11 source files, BUILD SUCCESS.
- Made by: backend-expert
