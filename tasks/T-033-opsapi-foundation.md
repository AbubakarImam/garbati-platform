# T-033: OpsApi foundation — sessions, downstream clients, lookup/testing-console/staff-management/joint-approval endpoints

Target repo: `Garbati.OpsApi` (currently design-only, three commits — `docs/DESIGN.md`). Branch: `feat/foundation`. Spec: `docs/DESIGN.md` — read it in full first, especially Sections 1–4, 6, and 2.1a/2.1b.

**This is the first of two build tasks.** `docs/DESIGN.md` describes six endpoint groups (Section 6, numbered 0–6). This task builds everything except the two stateful maker-checker workflows — **do not build `OpsReversalRequest`, `OpsCustomerEditRequest`, or any endpoint under Section 6 groups 2 (account/customer detail updates) or 3 (reversal requests)** in this task. A follow-up task (T-034, not started yet) builds those directly on top of what you leave here, the same way `Garbati.IdentityApi`'s mandates/grants (T-017) built on its foundation (T-016). Stopping at a clean boundary matters more than getting ahead.

## Why this split

This service integrates with **four** downstream services and will eventually hold **two** new stateful workflow entities on top of session management — building all of it in one pull request would be too large to review properly, the same reasoning that split `Garbati.IdentityApi`'s build into two tasks. This task is scoped to exactly what every other feature needs to sit on: a working login/session mechanism, correctly-scoped clients for all four downstream services, the frontend's static-hosting setup, and the three endpoint groups that are thin pass-throughs with no new workflow state of their own (lookup, the testing console, staff management, and the joint-account test console).

## Match the sibling services' conventions exactly

Check whichever of `LedgerApi`, `Garbati.IdentityApi`, `Garbati.OrchestratorApi`, or the now-built `Garbati.StaffIdentityApi` this environment has checked out alongside this repo, and mirror the actual implementation (not just docs) for every cross-cutting concern:

- **Project shape**: `src/Garbati.OpsApi/` with `Controllers/`, `Services/Implementations/` + `Services/Interfaces/`, `Clients/Contracts/{StaffIdentity,Identity,Orchestrator,Ledger}/` + `Clients/Implementations/` (mirroring `Garbati.OrchestratorApi`'s own `Clients/` shape exactly, since this is the second service on this platform to be a client of multiple others), `Entities/`, `Data/` (DbContext), `Migrations/`, `Validation/`, `Middleware/`, `Auditing/`, `Authorization/`, `Configuration/`, `Contracts/Requests`/`Contracts/Responses`, `Exceptions/`. `tests/Garbati.OpsApi.Tests/` mirroring the same structure, plus `Tests/Fixtures/` with hand-rolled fakes for all four downstream clients (`FakeStaffIdentityApiClient`, `FakeIdentityApiClient`, `FakeOrchestratorApiClient`, `FakeLedgerApiClient`) — same hand-rolled-fake convention `Garbati.OrchestratorApi` already uses, not a mocking library.
- **`global.json`**: same .NET SDK version/rollForward the sibling services pin.
- **`IHttpClientFactory`-managed `HttpClient`s** for all four downstream services, carrying `OpsApi`'s own `X-Api-Key` per service and propagating `X-Correlation-ID` — exactly `Garbati.OrchestratorApi`'s proven pattern (`IIdentityApiClient`/`ILedgerApiClient`), extended to four clients instead of two. Scopes per `docs/DESIGN.md` Section 3's table — read it carefully, the set granted to each service is deliberately narrow and explained there (in particular: **no** `identity.ops.reversal` on the `IdentityApi` client, **no** `ledger.accounts`/`.transfer`/`.reverse`/`.admin` on the `LedgerApi` client — only `ledger.read` and `ledger.funding`).
- **This service's own inbound authentication**: staff sessions (below), not API keys — `OpsApi` has no API-key-authenticated callers of its own in this version (its only caller is the browser, via a session cookie). Do not copy the `ApiKeyAuthenticationHandler` pattern for *inbound* requests to this service; that pattern is only used for this service's own *outbound* calls as a client.
- **Correlation ID middleware, exception-handling middleware** (`application/problem+json`, the same status-code mapping convention), Serilog request logging — copy the existing pattern.
- **Audit logging**: this task does not need a new audit table — session create/destroy events can log via the existing correlation-id-tagged request logging; there is no `docs/DESIGN.md`-specified audit table for sessions themselves (unlike `StaffAuditLog` on the identity side). If you believe session lifecycle events need a durable audit row beyond request logging, say so as a question in the PR rather than inventing a new entity silently.
- **Health checks**: `/health/live`, `/health/ready` (checks the database).
- **Dockerfile + CI workflow**: copy `garbati-platform/templates/ci-dotnet.yml` and a sibling's `Dockerfile` shape. Deploy job: close copy of a sibling's with placeholder values, same as every other service's first foundation task — getting `test`/`docker` green is what matters here.
- **Postgres via Testcontainers** for integration tests.

## What to build

### 1. Session management (`docs/DESIGN.md` Section 2, 2.1a, 2.3)

- `OpsSession` entity (2.2): `SessionId`, `SessionTokenHash` (SHA-256 of the raw token — the raw token is never persisted, same hash-only convention this platform already uses for API keys), `StaffId`, `FullName`, `Role`, `CreatedAt`, `LastActivityAt`, `ExpiresAt`.
- `POST /v1/ops/login` (2.1): `{username, password}` → calls `StaffIdentityApi POST /v1/staff/authenticate`. On success, create the `OpsSession` row and set a cookie containing only the opaque raw token, `HttpOnly`/`Secure`/`SameSite=Strict`. On failure, propagate `StaffIdentityApi`'s generic `401` unchanged.
- `GET /v1/ops/me` (2.1a): reads the current session's cached `{staffId, fullName, role}` — no downstream call. `401` if the cookie is missing, the token doesn't match any session, or the session is expired (idle or absolute, decision #1: 30-minute idle, 8-hour absolute — make both configurable, not hardcoded literals, so they can be tuned later without a code change).
- `POST /v1/ops/logout` (2.1a): deletes the current session's `OpsSession` row, clears the cookie.
- **Cascade revocation (decision #2, 2.3):** whenever this service calls `StaffIdentityApi PATCH /v1/staff/{staffId}/role` or `.../status` (as part of the staff-management endpoints, item 4 below), delete every `OpsSession` row for that `staffId` in the same request, before returning success. This is the actual "instant revocation" mechanism — get it right, and test it directly (change a logged-in staff member's role or status through this service, then confirm their session is gone).
- Session-validating middleware/filter: every authenticated endpoint in this task (everything except `login`) must run through one shared mechanism that resolves the session from the cookie, checks idle/absolute expiry, and makes `{staffId, fullName, role}` available to the controller — not duplicated per-action.

### 2. Downstream clients (`docs/DESIGN.md` Section 3)

Four clients, scoped exactly as the table in Section 3 specifies. Build the client interfaces/implementations for every operation this task's endpoints need (below) — you do not need every operation the full design eventually calls for (e.g. nothing related to `OpsReversalRequest`/`OpsCustomerEditRequest` yet), only what groups 1, 4, 5, 6 require:
- `IStaffIdentityApiClient`: authenticate, create staff, get staff, list staff, patch role, patch status, patch supervisor, password reset/change (mirrors `StaffIdentityApi`'s own 8 endpoints from T-032 — all of them are needed for group 5 below).
- `IIdentityApiClient`: customer read (for the lookup screen).
- `IOrchestratorApiClient`: account-opening, transfer (for the testing console). **Not** reversal — that's T-034's job, and this service doesn't call `OrchestratorApi`'s reversal endpoint from this task at all.
- `ILedgerApiClient`: account/balance read, funding (deposit).

### 3. Same-origin static hosting (`docs/DESIGN.md` 2.1b, decision #6)

`app.UseStaticFiles()` serving a dedicated folder (e.g. `wwwroot-console/` — not the implicit `wwwroot` root, to avoid any future collision), plus `MapFallbackToFile` for that folder's `index.html` so a client-side route resolves on a full page load instead of `404`. The folder can be empty for now (the frontend isn't built yet, T-029 is design-only) — just get the hosting plumbing correct and tested (a request to an arbitrary unmatched path returns the fallback file's content-type/status correctly, even if the file itself is a placeholder).

### 4. Endpoints — Section 6 groups 1, 4, 5, 6 only

- **Group 1 (customer & account lookup):** a single endpoint (your choice of exact route, e.g. `GET /v1/ops/customers/{customerId}`) combining an `IdentityApi` customer read and a `LedgerApi` account/balance read into one response — the frontend shouldn't have to make two calls itself (`docs/DESIGN.md` Section 6 item 1's own framing). Role check: all seven roles per Section 4's matrix.
- **Group 4 (transfer/transaction testing console):** endpoints wrapping `OrchestratorApi`'s account-opening and transfer operations, plus `LedgerApi`'s funding operation directly (3.4's deliberate exception — not through `OrchestratorApi`). Role check: `Teller`/`Supervisor` only (Section 4).
- **Group 5 (staff & role management):** thin pass-through endpoints to every one of `StaffIdentityApi`'s own 8 endpoints (create/get/list/role/status/supervisor/password) — this is where the cascade-revocation logic (above) actually gets exercised, on the role/status PATCHes specifically. Role check: `Administrator` only.
- **Group 6 (joint-account approval test console):** thin pass-through to `IdentityApi`'s existing single-request lookup/approve endpoints and its pending-requests list endpoint (`GET /v1/authorization/requests?status=Pending`, built by T-031). Role check: `Supervisor` can act, `Auditor` is read-only (Section 4) — enforce this distinction (an `Auditor` calling the approve action must be rejected, not just have it hidden from them in some UI that doesn't exist yet).

**Authorization matrix enforcement (Section 4):** every endpoint in this task checks the current session's `Role` against Section 4's table as real enforcement, not just as documentation — write this as one shared, reusable check (e.g. an attribute or a filter taking the allowed roles for that endpoint), not copy-pasted per action.

## Explicitly out of scope for this task

`OpsReversalRequest`, `OpsCustomerEditRequest`, and any endpoint under Section 6 groups 2 or 3 (customer-detail-update staging/approval, reversal request creation/approval). `OrchestratorApi`'s reversal endpoint is never called from this task. Nothing in `docs/DESIGN.md` Section 5 (reversal requests) is built here — that is entirely T-034's job.

## Tests

- Login: success sets a working session cookie; failure propagates `StaffIdentityApi`'s generic 401 unchanged (including the suspended/terminated case).
- `GET /v1/ops/me`: correct session → correct `{staffId, fullName, role}`; missing cookie → 401; a session artificially pushed past idle timeout → 401; past absolute timeout → 401.
- Logout: session actually gone afterward (a subsequent `/me` call with the same cookie is 401).
- **Cascade revocation**: log in as a staff member (create a real session), then — through this service's own staff-management endpoints — change that staff member's role or status; confirm their session is deleted and a subsequent `/me` call with their old cookie is 401. This is the single most important behavioral test in this task — it's the actual mechanism `docs/DESIGN.md` 2.3 depends on, not just a documented intention.
- Static hosting: a request to an unmatched client-side-looking route returns the fallback file (even if it's a placeholder), not a bare 404.
- Each group-1/4/5/6 endpoint: happy path, and a role check that actually rejects a role not in Section 4's table for that action (not just that an allowed role succeeds — prove the gate works both ways).
- Scope-correctness of the four downstream clients: assert (via the fakes) that each client sends exactly the scopes/fields `docs/DESIGN.md` Section 3 specifies — the same kind of regression test T-024 showed is necessary (a captured-argument assertion, not just a call-count check).
- Health checks respond.
- Migration test confirming the `OpsSession` schema matches the design.

Run build and tests yourself (Docker should be available for Testcontainers — if not, say so explicitly and report which suites you could and couldn't run).

## PR

Open a pull request from `feat/foundation` against `main` on `AbubakarImam/Garbati.OpsApi` (branch-protected — get `test`/`docker` CI green on your own branch before opening it). Use `garbati-platform/templates/pull_request_template.md` if it fits, or a sibling repo's PR style otherwise. Include what changed, how it was tested, what you could not verify (there is no real `StaffIdentityApi`/`IdentityApi`/`OrchestratorApi`/`LedgerApi` instance to test against live — note this explicitly, the same way `Garbati.OrchestratorApi`'s own foundation task flagged testing against mocked clients, not real locally-run siblings), every judgment call this task file left open, and a draft LEARNING_LOG-style plain-language explanation plus five interview questions with answers. Do not merge — only the owner merges.
