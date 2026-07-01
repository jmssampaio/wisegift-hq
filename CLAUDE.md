# WiseGift — Command Center

This is the orchestration repo for the WiseGift project. The human is the
**Product Owner and Tech Lead**, and is the final validator on all decisions.
Agents recommend and draft; the human approves. Nothing is "done" until the
human validates it.

## What WiseGift is
A gift-recommendation product. The **core differentiator is the recommendation
logic running over a rich product catalog** — turning a gifting context
(recipient, occasion, budget, preferences) into relevant suggestions. The
catalog data and the matching/ranking quality are the heart of the product;
everything else supports them.

## Repositories
- `~/Workspace/wisegift-backend` — backend (API, server logic, data serving)
- `~/Workspace/wisegift_flutter` — Flutter mobile/web client
  (add via `--add-dir` when launching from here)
- this repo (`wisegift-hq`) — specs, decisions, the agent team, shared docs

> Launch Claude Code from this repo to get the full team, then grant access to
> the code repos, e.g.:
> `claude --add-dir ../wisegift-backend --add-dir ~/Workspace/wisegift_flutter`

## The team (subagents)
**Product & analysis**
- `functional-analyst` — specs & user stories with acceptance criteria (docs/spec.md)

**Core differentiator**
- `catalog-data-engineer` — catalog data model, taxonomy, ingestion, quality (docs/data.md)
- `recommendations-specialist` — matching/ranking logic + evaluation (docs/recommendations.md)

**Build**
- `backend-expert` — API, server logic in ../wisegift-backend
- `frontend-expert` — Flutter client in the flutter repo
- `uiux-expert` — flows, layouts, component specs (docs/design/uiux.md)

**Quality, safety & ops**
- `qa-tester` — tests & verification against acceptance criteria, both repos
- `security-expert` — defensive security review across both repos (docs/security.md)
- `privacy-legal-advisor` — privacy & legal drafts, informational only (docs/legal.md)
- `devops-expert` — CI/CD, deployment, env config across both repos (docs/devops.md)

**Growth**
- `marketing-manager` — positioning & go-to-market (docs/marketing.md)

## Source of truth (read the relevant ones before acting)
- `docs/product/spec.md` — what we're building + acceptance criteria
- `docs/engineering/architecture.md` — system design & API contracts
- `docs/decisions.md` — running decision log (append; never rewrite)
- `docs/product/workflow.md` — the standard handoff order for features
- `docs/engineering/release-process.md` — branching model & deployment chain
- `docs/core/data.md` — catalog structure & quality
- `docs/core/recommendations.md` — recommendation design & evaluation metrics
- `docs/design/uiux.md` — flows & component specs
- `docs/legal/security.md` — threat model & findings
- `docs/legal/legal.md` — privacy/legal drafts (NOT legal advice)
- `docs/engineering/devops.md` — pipelines, environments, release process
- `docs/growth/marketing.md` — positioning & copy

## Working rules
- Read `docs/product/spec.md` and `docs/decisions.md` before starting any work.
- Follow the handoff order in `docs/product/workflow.md`; flag when a feature needs a
  different path.
- Record meaningful decisions in `docs/decisions.md` (date, who, what, why).
  Append only — never rewrite history.
- The **backend defines the API contract**; the frontend implements against it.
- The **catalog-data-engineer owns data shape/quality**; backend serves it; the
  **recommendations-specialist owns matching and must define how quality is
  measured** before building.
- **Code changes target the `develop` branch first** via a `feature/*` branch
  and PR, never `main` directly — see `docs/engineering/release-process.md`. Merging to
  `develop` or `main` triggers a deploy and is human-gated. (Early-days direct
  pushes to `main` are allowed only when the human explicitly invokes them.)
- Never mark work "done" — recommend; the human validates.
- Flag ambiguity, risk, and tradeoffs to the human rather than guessing.
- **Side-effectful actions require explicit human approval** before running:
  deploys, infra changes, anything that publishes, sends, deletes, or rotates
  keys. Propose and wait. (Especially devops-expert.)
- `security-expert` is defensive only — no exploit/offensive code.
- `privacy-legal-advisor` is informational only — not a lawyer; route binding
  questions to a qualified attorney.

## Commits
Code changes land in their own repos (`wisegift-backend`, `wisegift_flutter`),
reviewed in their respective IDEs. Specs and decisions commit here in HQ — this
repo is the single source of truth and the project's real memory across
sessions.
