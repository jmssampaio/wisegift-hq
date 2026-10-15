# Runbook — Merchant admin staging deploy (Render, Frankfurt)

**Purpose.** First deployment of the merchant admin (`wisegift-admin`, Next.js
App Router) to a hosted staging URL. Then point the backend's invite redirect
at it (`WISEGIFT_ADMIN_SIGNUP_URL`) and land backend PR #73.

**Owner.** `devops-expert` drafts; the **Product Owner executes**. Every step
below creates infrastructure, deploys, or changes live config. Nothing here
runs without explicit PO approval.

**Blocks.** Backend PR #73 (`fix/admin-invite-signup-flow`). That PR makes
the Shopify OAuth callback 302 to `WISEGIFT_ADMIN_SIGNUP_URL`, and the backend
boot-fails if the variable is unset, relative or http.

**Cross-refs.**
- `docs/decisions.md`:
  - 2026-09-29 "Frozen MVP scope": EU-only hosting.
  - 2026-10-14 "Admin ↔ backend transport": same-origin Next.js proxy; the embedded-admin auth model is still PENDING.
  - 2026-10-14 "Admin repo uses `main` as its integration branch".
  - 2026-10-14 "Staging deploy pipeline findings": backend staging is deployed manually from the Render dashboard.
- `wisegift-admin/render.yaml`: the service definition this runbook imports.
- `render-service-provisioning.md`: the sibling runbook for the backend service.

## What gets created

| Item | Value |
|---|---|
| Render service | `wisegift-admin-staging`, Web Service, **Node** runtime |
| Region | **Frankfurt** (same as `wisegift-backend-staging`) |
| Plan | **Free** (USD 0) while there are no pilots (PO decision 2026-10-15). It sleeps after ~15 min idle, so the first request takes 30–60 s. Switch to Starter (USD 7/month) before the first pilot merchant installs. |
| Branch / deploys | `main`, auto-deploy on merge (merging is the human gate) |
| Build / start | `npm ci --include=dev && npm run build` / `npm start` |
| Node | from `.nvmrc` (20.20.2) |
| Health check | `GET /healthz` (server liveness only; never calls the backend) |
| Public URL | `https://<admin-host>`. Render assigns it, normally `wisegift-admin-staging.onrender.com`. Copy the real value from the dashboard; the rest of this runbook calls it `<admin-host>`. |

The browser only talks to `<admin-host>`. `/admin/v1/*` is proxied
server-side to the backend, so the backend's session and `XSRF-TOKEN` cookies
are first-party to `<admin-host>`. No backend CORS change is needed.

---

## Prerequisites

1. **The admin PR `chore/admin-staging-deploy` → `main` is merged.** The
   Blueprint reads `render.yaml` from `main`. Merging deploys nothing yet,
   because the service does not exist.
2. Access to the Render workspace that hosts `wisegift-backend-staging`.
   The admin must be in the same workspace.
3. The Render GitHub app can see `jmssampaio/wisegift-admin`. Grant
   repo-level access, not org-wide, if Render asks.
4. A staging admin login you already use locally (email + password) for the
   login smoke check.

---

## 1. Create the service

**Preferred: Blueprint.** It keeps the service config in git, and avoids the
`render.yaml` drift the backend has.

1. Render Dashboard → **New** → **Blueprint**.
2. Select the repo `jmssampaio/wisegift-admin` and branch `main`.
3. Name the Blueprint `wisegift-admin-staging`.
4. Review the plan Render shows. It must list **exactly one** resource:
   - a web service `wisegift-admin-staging`
   - region **Frankfurt**
   - plan **Free**

   If any other region appears, **stop**.
5. Click **Apply**. This creates the service and immediately starts the first
   deploy (step 3).

**Fallback: manual Web Service.** Use this if the Blueprint flow is
unavailable.
- Path: New → Web Service → `jmssampaio/wisegift-admin`.
- Settings:
  - Name `wisegift-admin-staging`
  - Region **Frankfurt (EU Central)**
  - Branch `main`
  - Runtime **Node**
  - Build command `npm ci --include=dev && npm run build`
  - Start command `npm start`
  - Instance type **Free**
  - Advanced → Health Check Path `/healthz`
  - Auto-Deploy **On commit**
- Add the step-2 env vars **before** clicking Create, because Create triggers
  the first deploy.

**Verify.**
- The service page shows region Frankfurt.
- The service URL is shown. Record it as `<admin-host>`.

**Rollback.** Settings → **Delete Web Service**. This is irreversible but
stateless: the admin stores nothing. Delete the Blueprint too if you used one.

> Blueprint-managed: change service settings in `render.yaml` through a PR,
> not in the dashboard. Dashboard edits can be overwritten on the next
> Blueprint sync.

## 2. Environment variables

The Blueprint sets these from `render.yaml`. Check them under
**Environment**, or enter them by hand on the manual path. None is a secret.

| Key | Value | Notes |
|---|---|---|
| `WISEGIFT_BACKEND_URL` | `https://wisegift-backend-staging.onrender.com` | Backend origin for the `/admin/v1/*` proxy. https, no trailing slash. On Render the build **fails** if it is unset or not https. |
| `NEXT_PUBLIC_SHOPIFY_API_KEY` | the staging app `client_id` from `wisegift-widget/shopify.app.toml` | Public App Bridge identifier, **not** the Shopify app secret. Never put the client secret here. |
| `NEXT_TELEMETRY_DISABLED` | `1` | Next.js telemetry reports outside the EU. |

Do **not** set `NODE_ENV`, `PORT` or `NODE_VERSION`. Render injects `PORT`,
`.nvmrc` pins Node, and Next sets `NODE_ENV` itself.

Both `WISEGIFT_BACKEND_URL` and `NEXT_PUBLIC_SHOPIFY_API_KEY` are **compiled
in at build time**. After changing either, use **Save, rebuild, and deploy**.
A plain restart keeps the old value.

**Verify.** The three keys are present with the values above.

**Rollback.** Edit the values and Save, rebuild, and deploy.

## 3. First deploy

The deploy starts on Apply (Blueprint) or Create (manual). Watch
**Events → Deploy → Logs**.

**Verify.**
1. The build log shows Node **20.20.2** (picked up from `.nvmrc`).
2. `next build` prints the route table, including `ƒ /healthz`.
3. The deploy ends **Live** and the health check passes.
4. From a terminal, run `curl -i https://<admin-host>/healthz`. Expect `200`,
   `cache-control: no-store`, and body `{"status":"ok"}`.

**If it fails.**
- Build error `WISEGIFT_BACKEND_URL must be set to an absolute https:// URL`:
  fix the env var (step 2), then Manual Deploy → Clear build cache & deploy.
- Out of memory during the build: report it to devops-expert before you
  change the plan.

**Rollback.** None needed. A failed first deploy leaves nothing serving.

## 4. Smoke checks

Use a desktop browser **top-level** (not inside the Shopify admin), with
DevTools open on Network + Application → Cookies.

1. **`/login` loads.** Open `https://<admin-host>/login`.
   - The login form renders.
   - In Network, `GET /admin/v1/auth/me` returns **401**. It is answered via
     the proxy and is not a CORS error.
   - Cookies for `<admin-host>` include `XSRF-TOKEN`.
2. **Login works against the staging backend.**
   - Sign in with your staging admin credentials. The dashboard shell
     renders.
   - Check the `WG-ADMIN-SESSION` cookie: domain `<admin-host>`,
     **HttpOnly**, **Secure**, **SameSite=Lax**.
   - Open Widget → Configuration. It loads your tenant's config, so reads
     work through the proxy.
   - Click **Log out** and confirm `POST /admin/v1/auth/logout` returns
     **204**. This proves CSRF double-submit works through the proxy. A **403**
     here means CSRF is broken: stop and report.
3. **`/signup?token=bad` shows the invalid-invite state.**
   - Open `https://<admin-host>/signup?token=bad`.
   - The form "Create your WiseGift account" renders, and `?token=bad`
     disappears from the address bar.
   - The response headers include `Referrer-Policy: no-referrer`.
   - Submit with a throwaway email (e.g. `smoke-test@example.com`) and a
     password that meets the on-screen rules.
   - Expected: **"This invite link can't be used"** (`POST
     /admin/v1/auth/signup` → 403 `invalid_invite_token`). No account is
     created.
   - If you see a generic error instead, the call failed for another reason
     (a CSRF 403 without a code, or a proxy error). Stop and report.
4. **`/signup` with no token** shows "Invite link needed".

**Rollback.** None. These checks are read-only, apart from your own
login/logout.

## 5. Set `WISEGIFT_ADMIN_SIGNUP_URL` on the backend

1. Render → `wisegift-backend-staging` → **Environment** → Add environment
   variable.
2. Key: `WISEGIFT_ADMIN_SIGNUP_URL`. Value: `https://<admin-host>/signup`.
   - Absolute and https.
   - Path exactly `/signup`, no trailing slash, no `?` and no `#`. The backend
     rejects anything else at boot.
3. Choose **Save only** if offered. The current `develop` build ignores the
   variable, and step 6 deploys the code that reads it. "Save and deploy" is
   also harmless: it redeploys the current `develop`.

Set it in the dashboard even though PR #73 declares the slot in the backend
`render.yaml` (`sync: false`). The live backend service is configured from
the dashboard; see the 2026-10-14 drift finding.

**Verify.**
- The variable is listed with the exact value.
- The backend is still Live and `GET https://wisegift-backend-staging.onrender.com/actuator/health`
  returns `UP`.

**Rollback.** Delete the variable. This is safe **only while PR #73 is not
deployed**. Once #73 is live, removing it makes the next boot fail.

## 6. Merge backend PR #73 and deploy the backend

1. Re-check the PR on GitHub:
   - CI is green, including the integration tests.
   - It is still based on `develop`.
   - Remove the "Do not merge yet" note once steps 1–5 are done.
2. **Merge PR #73 into `develop`.** This is the human gate.
3. Render → `wisegift-backend-staging` → **Manual Deploy → Deploy latest
   commit**. Backend staging is deployed manually; the CI hook is
   non-functional (2026-10-14 finding).

**Verify.**
1. The deploy's commit SHA is the #73 merge commit, and the deploy goes
   **Live**.
2. The boot logs contain **no** error mentioning `WISEGIFT_ADMIN_SIGNUP_URL`.
3. `/actuator/health` returns `UP`.
4. Repeat smoke check 4.2 (login/logout through the admin). The backend
   restart must not break the proxy.

**Rollback.** See step 8B.

## 7. Verify a fresh Shopify install redirects to the admin signup

Use a **new development store**, not `wisegift-dev.myshopify.com`. That
tenant already has an admin, and re-installing there changes its state.

1. Partner Dashboard → Stores → **Add store** → development store, e.g.
   `wisegift-smoke-<date>`.
2. Install the **WiseGift Staging** app on it from the Partner Dashboard
   ("Test on development store" / select store). Approve the scopes.
3. Expected chain:
   - Shopify → `wisegift-backend-staging…/oauth/shopify/install`
   - → consent
   - → `/oauth/shopify/callback`
   - → **302 to `https://<admin-host>/signup?token=…`**
4. The admin signup form renders and the token is stripped from the address
   bar.
5. Complete the signup with a real test inbox and a valid password. Expect
   "Account created", then the dashboard for the new tenant.

**Record one observation for the PO decision log.** Did the signup page load
**top-level** (the address bar shows `<admin-host>`) or **inside the Shopify
admin frame** (the address bar stays on `admin.shopify.com`)?
- Inside the frame, browsers withhold the admin's `SameSite=Lax` cookies.
  Signup or login will then fail with a CSRF/401 error.
- That is the **pending embedded-admin auth decision** (decisions.md
  2026-10-14), not a deploy defect. Record it and stop. Do not work around it
  by loosening cookie settings.

**Side effects.**
- The new store becomes a new staging tenant.
- Its starter catalog is synced and embedded through OpenAI, costing cents.
- To clean up afterwards, uninstall the app from the store. This fires
  `app/uninstalled` and deactivates the tenant.

**Rollback.** Uninstall the app from the smoke-test store.

## 8. Rollback

There are no database migrations in either change, so nothing to roll back
in Neon.

**A. Bad admin deploy** (the backend is fine):
1. `wisegift-admin-staging` → **Events** → the last good deploy →
   **Rollback**.
2. Then revert the offending admin PR on `main` through a PR. A plain
   rollback is undone by the next auto-deploy from `main`.
3. Check Settings → Auto-Deploy after a rollback. Make sure it is in the
   state you want before the fix merges.

**B. PR #73 misbehaves on staging:**
1. `wisegift-backend-staging` → **Events** → the last pre-#73 deploy →
   **Rollback**.
2. Open a revert PR for #73 on `develop` (human-merged), so the next manual
   deploy does not bring it back.
3. **Leave `WISEGIFT_ADMIN_SIGNUP_URL` set.** Pre-#73 code ignores it.
4. Invite links minted while #73 was live point at `<admin-host>/signup`.
   They keep working while the admin is up, as long as the invite token
   format is unchanged (backend-engineer to confirm).

**C. Take the admin offline entirely:**
1. First do **B**. While #73 is live, the backend sends new installs to the
   admin, and a dead admin means no new merchant can sign up.
2. Then `wisegift-admin-staging` → Settings → **Suspend** (reversible).
3. Delete the service only if you are abandoning this host.

**D. Wrong `WISEGIFT_ADMIN_SIGNUP_URL` value** (boot failure after #73):
1. Correct the value (step 5 rules).
2. Manual Deploy → Deploy latest commit.

---

## Later changes (not part of this runbook)

- **Custom domain** `admin-staging.wisegift.app` (architecture.md
  §Environments).
  1. Add it in Render plus DNS.
  2. Update `WISEGIFT_ADMIN_SIGNUP_URL` and redeploy the backend.
  3. Keep the `onrender.com` host up for 24 h, so invites already minted still
     resolve.
- **Production admin service.** It needs a promotion model, because `main`
  already auto-deploys staging. That is a separate decision.
