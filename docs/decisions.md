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

## 2026-06-30 — Added Flutter Integration Notes to architecture.md
- Context: Flutter dev needs a concise reference for auth, the Firestore-to-Spring recommendation handoff, URL namespace rules, country filtering, and the availability-check pattern.
- Decision: Appended `## 8. Flutter Integration Notes` to docs/architecture.md covering those five topics. Also flagged that `POST /api/v1/gifts/recommendations` is auth-free despite carrying personal PII — backend expert should assess adding Firebase JWT requirement.
- Made by: frontend-expert
