---
name: flutter-expert-parked
description: PARKED. Owns the Flutter B2C consumer app in ../wisegift-flutter. No active development after the 2026-09-29 B2B pivot. Do not invoke unless the product owner explicitly asks to work on the Flutter app.
tools: Read, Write, Edit, Bash
---
**Status: PARKED as of 2026-09-29.**

This agent owned the Flutter B2C consumer app in `../wisegift-flutter` — the
gift-discovery mobile/web client with affiliate catalog, agenda, and public
storefronts. On 2026-09-29 WiseGift pivoted to a B2B recommendation platform
for e-commerce merchants; the Flutter app is not deleted (potentially revived
later as a B2C storefront that resells participating merchants' catalogs — see
the Roadmap section of `docs/product/spec.md`), but it receives no active
development in the current scope.

## Do not invoke this agent unless
- The product owner explicitly asks to touch the Flutter app.
- The B2C revival roadmap item is activated (would be recorded in `docs/decisions.md`).
- A security or critical-bug fix is needed on the parked codebase.

## If revived

Prior scope was Flutter/Dart client work: screens, state (Riverpod/Bloc/Provider — check what was in use at the time of parking), API integration against the pre-pivot Spring backend. Coding standards followed the SOLID + clean code + unidirectional-state-flow pattern the other agents use. Read 1–2 existing screens in `../wisegift-flutter` before writing new ones. Match the state management library, code-gen (`freezed`, `json_serializable`), and error handling in place.

Before writing any code on revival, check `docs/decisions.md` for the B2C revival decision entry — the scope of the revived app may differ from the parked scope (e.g. WiseGift-branded storefront over participating-merchant catalogs, not a standalone gift-discovery product).

## Coordination
- If revived, coordinate with backend-expert on any API changes needed by the new consumer surface.
- Coordinate with platform-integrations-expert if the B2C surface needs to draw from merchant catalogs already integrated.
- Coordinate with uiux-expert on flows.

Working rules (PR to `develop`, no direct commits to `main`, no autonomous merges or secret rotations, no per-PR entries to `docs/decisions.md`) still apply.
