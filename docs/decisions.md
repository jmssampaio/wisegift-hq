# Decision Log
(append entries; never rewrite history)
## YYYY-MM-DD — <decision>
- Context:
- Decision:
- Made by:

## 2026-06-30 — Added Flutter Integration Notes to architecture.md
- Context: Flutter dev needs a concise reference for auth, the Firestore-to-Spring recommendation handoff, URL namespace rules, country filtering, and the availability-check pattern.
- Decision: Appended `## 8. Flutter Integration Notes` to docs/architecture.md covering those five topics. Also flagged that `POST /api/v1/gifts/recommendations` is auth-free despite carrying personal PII — backend expert should assess adding Firebase JWT requirement.
- Made by: frontend-expert
