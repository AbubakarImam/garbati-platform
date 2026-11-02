# T-062: Ops console frontend foundation (T-029's design, build)

Owner: builder. Builds the whole v1 (minimal) design from `garbati-ops-console/docs/DESIGN.md`
(T-029) in one task — the same "foundation task builds the full ratified v1 design" pattern already
used for `Garbati.StaffIdentityApi` (T-027 → T-032) and `Garbati.ProductApi` (T-009 → T-045). Read
that design doc in full before starting — it is the spec, and this task does not reopen any of its
choices. `Garbati.OpsApi/docs/DESIGN.md` Section 6 is the backend contract this app calls against;
read it too, and read the actual controllers/contracts in `Garbati.OpsApi/src/Garbati.OpsApi/` rather
than trusting any paraphrase here, including this one.

## Why this is buildable now

T-029's design was blocked on one thing: "`OpsApi` itself being built" (Section 9/10 item 1). `OpsApi`
is now built, tested, deployed, and live on Azure (T-033/T-034, T-048) — this is the one concrete
build gap left open in the whole ops-console initiative. Nothing else in the platform depends on
this shipping; it's the last piece of D-13.

## What to build (per DESIGN.md Section 9, "what done looks like")

1. **Vite + React 18 + TypeScript project skeleton** (DESIGN.md Section 3). React Router v6 for
   routing, TanStack Query for every call to `OpsApi`, Ant Design for components (a recommendation
   per the design doc, not locked — swap it if you have a good reason, but don't reinvent form/table
   primitives from scratch without one).
2. **`AuthContext` + the shared `401` handler** (Section 4): `POST /v1/ops/login`, immediately
   followed by `GET /v1/ops/me` → `{staffId, fullName, role}` stored in a top-level React context —
   this is the *only* place `role` lives client-side. Every full page load calls `GET /v1/ops/me`
   before rendering anything role-gated (the session cookie is `HttpOnly`, invisible to this app's
   own JS — that's the whole reason this call exists on every load, not just at login). Any `401`
   from any call, anywhere in the app, routes to Login through one shared handler (a React Query
   global error handler or a fetch/axios interceptor) — not reimplemented per screen. `POST
   /v1/ops/logout` on explicit logout, clears context, routes to Login.
3. **Layout shell + role-gated navigation** (Section 5): a sidebar/top-bar layout (Ant Design's
   `Layout`/`Menu` if using it) showing the eight screens below, filtered by `AuthContext.role`
   against the same role matrix `Garbati.OpsApi/docs/DESIGN.md` Section 4 defines — keep this as one
   shared constant, not duplicated/hand-copied, so the two can't silently drift apart. **This is UX,
   not security** (design doc Section 1, stated explicitly, repeat it in a code comment where the
   matrix lives): hiding a nav item does not stop a direct URL visit or a raw API call — `OpsApi`
   re-checks the real session's role server-side on every request regardless. A route visited without
   permission shows the real `403` `OpsApi` returns as a plain "not permitted" state, not a
   client-side guess that pre-empts the call.
4. **The eight screens**, each calling exactly the real `OpsApi` route group it maps to (read the
   actual controller before building the screen that calls it — route paths below are a pointer, not
   the contract):
   - **Login** — `SessionController` (`POST /v1/ops/login`, `GET /v1/ops/me`, `POST
     /v1/ops/logout`). Everyone, unauthenticated.
   - **Customer & account lookup** — `CustomerLookupController` (`GET /v1/ops/customers/{customerId}`).
     All seven roles.
   - **Customer detail edit — stage a change** — `CustomerEditRequestsController` (`POST
     /v1/ops/customer-edits`). `CustomerServiceOfficer` (maker).
   - **Customer detail edit — approve/reject queue** — `CustomerEditRequestsController` (`GET
     /v1/ops/customer-edits`, `GET .../{id}`, `POST .../{id}/approve`, `POST .../{id}/reject`).
     `Supervisor` (checker).
   - **Reversal requests — create** — `ReversalRequestsController` (`POST /v1/ops/reversals`).
     `Teller`, `CustomerServiceOfficer`.
   - **Reversal requests — approve/reject queue** — same controller (`GET /v1/ops/reversals`, `GET
     .../{id}`, `POST .../{id}/approve`, `POST .../{id}/reject`). `Supervisor` acts;
     `ComplianceOfficer`/`RiskOfficer`/`Auditor` see the same list/detail views read-only (no
     approve/reject buttons rendered for them — Section 2's table lists this as a separate row
     precisely because the allowed actions differ by role on the same data).
   - **Transfer/transaction testing console** — `TestingConsoleController` (`POST
     /v1/ops/console/account-openings`, `POST .../transfers`, `POST .../funding`). `Teller`,
     `Supervisor`.
   - **Staff & role management** — `StaffManagementController` (`POST /v1/ops/staff`, `GET
     .../{staffId}`, `GET /v1/ops/staff`, `PATCH .../{staffId}/role`, `PATCH .../{staffId}/status`,
     `PATCH .../{staffId}/supervisor`, `POST .../{staffId}/password`). `Administrator`-only.
   - **Joint-account approval test console** — `JointApprovalConsoleController` (`GET
     /v1/ops/joint-approvals`, `GET .../{requestId}`, `POST .../{requestId}/approve`). `Supervisor`
     acts, `Auditor` read-only. Label this screen in-UI as a stand-in for a real customer channel
     that doesn't exist yet (design doc Section 2, OpsApi design doc item 6) — don't let it look like
     a real customer-facing flow.

   No screen outside this list. No screen invents its own data shape where `OpsApi`'s response
   already defines one — use the real response records in `Garbati.OpsApi/src/Garbati.OpsApi/Contracts/Responses/`
   directly (generate or hand-write matching TypeScript types from them, your call, but they must
   match field-for-field).
5. **Local development**: Vite's `server.proxy` forwarding `/v1/ops/*` to a locally-running `OpsApi`
   (design doc Section 6) — same-origin in dev exactly as in production, no CORS path needed anywhere.
   Confirm this actually works against a real local `OpsApi` instance (started via
   `garbati-platform`'s own `docker-compose.yml`, T-038) before calling this done, not just against a
   mock.
6. **Testing** (design doc Section 8): Vitest + React Testing Library. Cover: the role-gated
   nav/route logic actually shows/hides the right items per role (test at least two different roles
   against the matrix, not just one), the "any `401` redirects to Login" behavior, and
   `AuthContext`'s handling of a `GET /v1/ops/me` failure on initial load. Full browser e2e
   (Playwright/Cypress) is explicitly out of scope for v1 per the design doc — don't add it.

## What this task does not include

- No change to `OpsApi` or any other backend service. If building this surfaces a real contract gap
  (the same way T-029's own design process found two — session introspection/logout, and a
  same-origin cookie conflict, both already fixed in `OpsApi` before this task), stop and flag it the
  same way those were flagged, don't silently patch around it or invent a workaround in the frontend.
- No MFA for privileged actions, no accessibility audit, no write access for
  `ComplianceOfficer`/`RiskOfficer` beyond read-only — all explicitly deferred in the design doc
  (Section 10), not this task's job to add.
- No deployment wiring in this task. The design doc (Section 7) describes deploying the built
  `dist/` bundle into `OpsApi`'s existing Azure Web App via this repo's own scoped credential — that
  Azure-side wiring (the credential, the deploy workflow actually running against production) is
  real infrastructure work, scope it as a separate step once this PR is reviewed, the same way
  T-048 was a separate task from each service's own build task.

## Acceptance

- `npm run build` produces a working production bundle; `npm run dev` runs locally against a real
  `docker-compose`-started `OpsApi` and the full login → lookup → logout path works end to end by
  hand, for at least two different staff roles (confirm actual role-based nav differences, not just
  that *a* nav renders).
- Vitest suite green, covering the three behaviors named in item 6 above.
- `docs/DESIGN.md` updated if anything above had to change from what's written there, with the
  deviation and why — same discipline as every other decision log on this platform.
- PR description states plainly what was tested against a real `OpsApi` instance vs. a stub/mock, if
  any mocking was used anywhere during development.
