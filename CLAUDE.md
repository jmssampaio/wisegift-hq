# WiseGift — Command Center

This is the orchestration repo for the WiseGift project. The human is the
Product Owner and Tech Lead, and is the final validator on all decisions.

## Repositories
- ../wisegift-backend — backend (API, data, business logic)
- ../wisegift-flutter — Flutter mobile/web client
- this repo (wisegift-hq) — specs, decisions, agent team, shared docs

## The team (subagents)
- functional-analyst — specs & user stories (docs/spec.md)
- backend-expert — works in ../wisegift-backend
- frontend-expert — works in ../wisegift-flutter
- uiux-expert — design specs (docs/uiux.md)
- qa-tester — tests & verification across both repos
- marketing-manager — go-to-market (docs/marketing.md)

## Source of truth (read before acting)
- docs/spec.md — what we're building + acceptance criteria
- docs/architecture.md — system design & API contracts
- docs/decisions.md — running log of decisions (append, never rewrite)
- docs/uiux.md — flows & component specs
- docs/marketing.md — positioning & copy

## Working rules
- Always read docs/spec.md and docs/decisions.md before starting work.
- Record meaningful decisions in docs/decisions.md (date, who, what, why).
- Never mark work "done" — recommend; the human validates.
- Flag ambiguity and risk to the human rather than guessing.
- Backend defines the API contract; frontend implements against it.
