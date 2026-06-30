# WiseGift — Workflow & Handoffs

This is the default playbook for how a feature moves through the team. It is a
**default, not a straitjacket** — small changes skip stages, and the product
owner can reorder. But when in doubt, follow this order. The goal is to catch
problems early (a privacy or auth issue caught at spec stage costs minutes;
caught after build, days).

## The human's role
You are Product Owner + Tech Lead. You sit at the top of every stage as the
validator. Agents propose and draft; you approve before anything moves forward,
and especially before anything irreversible runs.

## Standard feature flow

### 1. Frame — functional-analyst
Turns the product idea into a spec in `docs/spec.md`: user stories + acceptance
criteria. No implementation detail. Output reviewed by the human before moving on.

### 2. Safety review (on the spec, before any build)
- **security-expert** threat-models the feature: what could go wrong, what data
  and auth it touches. Notes in `docs/security.md`.
- **privacy-legal-advisor** flags privacy/consent/retention implications. Notes
  in `docs/legal.md`.
Doing this on the *spec* (not the finished code) is the whole point — it shapes
the design instead of forcing rework.

### 3. Core design — catalog-data-engineer + recommendations-specialist
*(for anything touching recommendations or catalog — i.e. most features)*
- **catalog-data-engineer** defines what catalog data/attributes the feature
  needs and ensures the data can support it (`docs/data.md`).
- **recommendations-specialist** defines the matching/ranking approach AND how
  its quality will be measured (`docs/recommendations.md`). Evaluation is
  defined *before* building, not after.
- **uiux-expert** designs the flow and components in parallel (`docs/uiux.md`).

### 4. Architecture sign-off — human (Tech Lead)
Confirm the API contract and data model cohere across repos. Record the agreed
contract in `docs/architecture.md`. This is where you keep the two repos from
drifting apart.

### 5. Build
- **backend-expert** implements the API/server logic against the contract in
  `../wisegift-backend`.
- **catalog-data-engineer** implements ingestion/data work as needed.
- **devops-expert** sets up/updates pipelines and environments to support it.
- **frontend-expert** builds the Flutter UI against the API contract and the
  UI/UX spec.
Backend contract should be settled before frontend builds against it.

### 6. Verify
- **qa-tester** tests against the acceptance criteria from step 1, both repos.
- **security-expert** reviews the built result for issues flagged in step 2.
- **recommendations-specialist** runs the evaluation defined in step 3 — does it
  actually meet the quality bar?
Failures go back to the relevant builder, not forward.

### 7. Release — devops-expert (human-gated)
Code moves through the branching chain in `docs/release-process.md`:
`feature/*` → `develop` (deploys to staging) → `main` (deploys to production).
The devops-expert prepares branches/PRs and the deploy plan but **does not merge
to `develop`/`main` or deploy on its own** — each merge triggers a deploy and is
an explicit human yes/no gate. (Early-days direct-to-`main` is allowed only when
the human invokes it.)

### 8. Grow — marketing-manager
Positioning and go-to-market in `docs/marketing.md`, based only on features that
actually shipped per `docs/spec.md`. No invented capabilities.

## Quick paths (skip stages)
- **Bug fix / small UI tweak:** analyst (light) → builder → QA → human. Skip
  core design and marketing.
- **Pure data/catalog work:** catalog-data-engineer → recommendations-specialist
  (re-evaluate) → QA → human.
- **Copy/marketing only:** marketing-manager → human.
- **Anything touching personal data:** never skip the privacy-legal-advisor.
- **Anything touching auth, secrets, or deploys:** never skip security-expert /
  the human approval gate.

## How to invoke the team
Name the sequence rather than saying "build feature X" — the orchestrator works
better with a path. Example:

> "functional-analyst: spec the 'recommend by occasion' feature. Then
> security-expert and privacy-legal-advisor review the spec. Then
> recommendations-specialist + catalog-data-engineer design the matching and
> the data it needs, with an evaluation plan. Show me everything before any
> code is written."

## Decision discipline
Every stage that makes a real choice appends to `docs/decisions.md`
(date · who · decision · why). That log is how all eleven agents — and future
sessions — stay aligned. It is the project's memory; keep it current.
