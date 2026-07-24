# WiseGift — Branching & Release Process

This defines **where code goes and how it ships**. Every code-writing agent
(backend-expert, frontend-expert, catalog-data-engineer) and especially the
devops-expert must follow this. workflow.md says *who* does the work; this says
*how it gets delivered*. Both repos (`wisegift-backend` and `wisegift_flutter`)
follow this independently — each has its own `develop`, `main`, and pipeline.

## Environments

| Environment | URL | Branch | Firebase project |
|---|---|---|---|
| Production | https://wisegift.app | `main` | `wisegift` |
| Staging | https://wisegift-staging.web.app | `develop` | `wisegift-staging` |

---

## Branch model

- `main` — production. Always deployable. Only updated via promotion from
  `develop` (see below). Protected.
- `develop` — integration branch. **All feature work targets `develop` first.**
- `feature/<short-name>` — short-lived working branches off `develop`, merged
  back into `develop` via pull request.

## The delivery chain (target flow)

1. Work happens on a `feature/<name>` branch off `develop`.
2. Open a PR into `develop`. QA + the relevant reviewer (security for sensitive
   changes) sign off. The **human (Product Owner) approves the merge**.
3. Merge to `develop` → triggers the GitHub deploy process to the
   **staging/dev environment**.
4. Verify on staging.
5. Promote `develop` → `main` via PR. The **human approves** this promotion.
6. Merge to `main` → triggers the GitHub deploy process to **production**.

So: `feature/*` → `develop` (deploy to staging) → `main` (deploy to production).
**Code never lands directly on `main`. No exceptions.**

## Rules for agents

- **Every code change targets `develop`** via a `feature/*` branch and a PR
  whose base is `develop`. Never open a PR with base `main`. Never push to
  `main` directly. The `develop` → `main` promotion is a separate human-owned
  PR and is the only path code takes to production.
- Agents **do not merge PRs or push to `main`** on their own. They prepare the
  branch/PR and describe it; the **human approves and performs the merge**.
- Anything that **triggers a deploy** (merging to `develop` or `main`) is a
  human-gated action — propose, show what will happen, wait for approval. This
  is consistent with the side-effect rule in CLAUDE.md.
- The **devops-expert owns the GitHub deploy workflows** (Actions), the
  environment config, and keeps the two repos' pipelines consistent. Deploy
  changes are recorded in `docs/engineering/devops.md`.
- Never put secrets in workflow files or branches — reference GitHub
  Actions secrets / environment variables (coordinate with security-expert).

## Commit & PR hygiene
- One feature per `feature/*` branch; keep PRs focused and reviewable.
- PR description should reference the spec/acceptance criteria it satisfies.
- Record meaningful release/branching decisions in `docs/decisions.md`.
