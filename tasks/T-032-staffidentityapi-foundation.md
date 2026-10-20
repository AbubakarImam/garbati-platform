# T-032: StaffIdentityApi foundation — the entire v1 design in one task

Target repo: `Garbati.StaffIdentityApi` (currently design-only, one commit plus one revision — `docs/DESIGN.md`). Branch: `feat/foundation`. Spec: `docs/DESIGN.md` — read it in full first; it is short and this task builds all of it, not a slice.

This is a **minimal, already-fully-designed** service — unlike `Garbati.IdentityApi` (split across T-016/T-017 because its design is large), this task builds the whole of `docs/DESIGN.md` in one pull request, the same way `Garbati.OrchestratorApi`'s v1 foundation (T-022) built its entire (also deliberately minimal) design in one go.

## Match the sibling services' conventions exactly

Per `ARCHITECTURE.md`'s shared conventions (Section 5) and `AGENTS.md` ("match the existing code"), mirror `LedgerApi`/`Garbati.IdentityApi`/`Garbati.OrchestratorApi`'s actual implementation, not just their docs, for every cross-cutting concern — check whichever of those three repos this environment has checked out alongside this one (`/Users/USER/Projects/LedgerApi`, `/Users/USER/Projects/Garbati.IdentityApi`, `/Users/USER/Projects/Garbati.OrchestratorApi`):

- **Project shape**: `src/Garbati.StaffIdentityApi/` with `Controllers/`, `Services/Implementations/` + `Services/Interfaces/`, `Entities/`, `Data/` (DbContext), `Migrations/`, `Validation/`, `Middleware/`, `Auditing/`, `Authorization/`, `Configuration/`, `Contracts/Requests`/`Contracts/Responses`, `Exceptions/`. `tests/Garbati.StaffIdentityApi.Tests/` mirroring the same structure.
- **`global.json`**: same .NET SDK version/rollForward the sibling services pin.
- **API-key authentication + scopes**: copy the existing `ApiKeyAuthenticationHandler`/`ApiKeyOptions`/`IValidateOptions` pattern exactly (hash-only configuration, never the raw key — `ARCHITECTURE.md` Section 5). This service's scopes (`docs/DESIGN.md` Section 3, decision #3): `staffidentity.authenticate`, `staffidentity.admin` — exactly two, nothing more.
- **Correlation ID middleware, exception-handling middleware** (`application/problem+json`, the same status-code mapping convention — 400/401/403/404/409/422/500), Serilog request logging — copy the existing pattern, adapted to this service.
- **Audit logging**: copy the existing `IAuditLogger`/`AuditLogger` pattern (staged on the same `DbContext`, committed by the same `SaveChangesAsync` as the change it describes). This service's own `StaffAuditLog` table and action names are in `docs/DESIGN.md` Section 2.5 — use that table, do not reuse another service's `AuditLogs` shape verbatim if the field names differ.
- **Health checks**: `/health/live`, `/health/ready` (checks the database).
- **Dockerfile + CI workflow**: copy `garbati-platform/templates/ci-dotnet.yml` as `.github/workflows/ci.yml` and a sibling's `Dockerfile` shape (API image + migrator image targets). Leave the `deploy` job's Azure specifics as a close copy of a sibling's for now (placeholder resource/app names) — getting `test`/`docker` green is what matters for this task; the owner wires up real deploy values separately, the same way IdentityApi's and OrchestratorApi's deploy jobs were initially placeholders later filled in.
- **Postgres via Testcontainers** for integration tests, no local database needed to run `dotnet test`.
- **Idempotency key**: only `POST /v1/staff` takes one (`docs/DESIGN.md` Section 3) — same convention (client-supplied key, hash of the request body, same key + different body → `409`) as every other create-something-new endpoint on this platform. No other endpoint in this task needs one.

## What to build

**Entities** (`docs/DESIGN.md` Section 2):

- `StaffUser` (2.1): `Id`/`StaffId` (public, `STF-XXXXXXXXXX`), `FullName`, `Username` (unique, case-insensitive — a unique index, not just an application-level check), `PasswordHash`, `Role` (enum, 2.2), `Status` (`Active`/`Suspended`/`Terminated`), `MustChangePassword`, `SupervisorStaffId` (nullable, self-referencing FK to `StaffUser.StaffId` — not a foreign key to the internal `Id`, to stay consistent with how every other service exposes only its public id externally), `CreatedAt`, `CreatedByStaffId` (nullable), `UpdatedAt`.
- `Role` enum (2.2): `Teller`, `CustomerServiceOfficer`, `Supervisor`, `ComplianceOfficer`, `RiskOfficer`, `Auditor`, `Administrator`. Exactly these seven, in this order if an ordinal matters anywhere (prefer string storage for the database column regardless, matching how sibling services store their own enums, so a future reordering can't silently corrupt stored data).
- `StaffAuditLog` (2.5): `Id`, `Action` (`StaffCreated`/`RoleChanged`/`StatusChanged`/`SupervisorChanged`/`PasswordReset`/`AuthenticationSucceeded`/`AuthenticationFailed`), `TargetStaffId`, `ActingStaffId` (nullable only for `AuthenticationFailed` and the seeded Administrator's own `StaffCreated` row), `Details`, `CreatedAt`.

**The one-time Administrator seed (decision #2):** a migration (or startup check — your call, say which and why in the PR) creates exactly one `StaffUser` with `Role = Administrator`, `MustChangePassword = true`, and a password that is **not** committed to source control and **not** logged anywhere — generate a strong random one at migration-write time is wrong (it would be baked into the migration file, which IS committed); instead, either (a) seed with a password that must be set via an out-of-band mechanism before the account is usable (e.g. a known-invalid placeholder hash that fails every real login attempt until an operator runs a one-time "set initial admin password" script/endpoint against the deployed instance), or (b) generate it from an environment variable read only at migration-apply time, never from a literal in the migration file. Pick one, implement it, and explain the choice clearly in the PR — this is the same class of judgment call `Garbati.IdentityApi`'s production signing-key handling and `LedgerApi`'s system-account seeding both had to make, and the task file deliberately leaves the exact mechanism to you rather than prescribing one that might not fit how this gets deployed.

**Password hashing (`docs/DESIGN.md` Section 4):** `Microsoft.AspNetCore.Identity`'s `PasswordHasher<StaffUser>` — **the one new package dependency this task introduces**, used only for its hasher (no cookie middleware, no `UserManager`, no other part of that package's surface). Minimum 12 characters, no composition rules. `PasswordHasher<T>.VerifyHashedPassword`'s `SuccessRehashNeeded` result (its hasher upgrades its own algorithm/iteration count over time) should trigger a silent re-hash-and-save on next successful login — a real, standard behavior of this exact API, not extra scope you're inventing.

**Endpoints** (`docs/DESIGN.md` Section 3 — the table there is the literal spec; implement every row):

1. `POST /v1/staff/authenticate` (scope `staffidentity.authenticate`) — `{username, password}` → `200 {staffId, fullName, role}` on success, generic `401` on any failure (wrong username, wrong password, `Status != Active` — all three must be indistinguishable to the caller, to avoid username enumeration). Audits `AuthenticationSucceeded`/`AuthenticationFailed` either way (failure case: `Details` holds the attempted username, never the password).
2. `POST /v1/staff` (scope `staffidentity.admin`) — `{idempotencyKey, actingStaffId, fullName, username, role}`, no password in the request. Generates a one-time temporary credential server-side, returned once in the response body, never persisted in plain text or logged. Sets `MustChangePassword = true`. Idempotency key required, same convention as elsewhere.
3. `GET /v1/staff/{staffId}` (scope `staffidentity.admin`) — never returns `PasswordHash`; includes `supervisorStaffId`.
4. `GET /v1/staff?supervisorStaffId=...` (scope `staffidentity.admin`) — paged list; the `supervisorStaffId` query parameter filters, when present.
5. `PATCH /v1/staff/{staffId}/role` (scope `staffidentity.admin`) — `{actingStaffId, role}`. Does **not** touch `SupervisorStaffId` even if the new role is no longer `Teller`/`CustomerServiceOfficer` — that's a separate, independently-audited call (endpoint 6).
6. `PATCH /v1/staff/{staffId}/status` (scope `staffidentity.admin`) — `{actingStaffId, status}`. Rejects (400) any transition out of `Terminated`.
7. `PATCH /v1/staff/{staffId}/supervisor` (scope `staffidentity.admin`) — `{actingStaffId, supervisorStaffId}` (nullable, to unassign). Rejects (400) unless `supervisorStaffId` is `null` or references an existing, `Active`, `Role = Supervisor` `StaffUser`; rejects self-reference.
8. `POST /v1/staff/{staffId}/password` — two distinct behaviors on one route, distinguished by which scope the caller's credential holds (`docs/DESIGN.md` Section 3's own note on this — mirrors how `Garbati.IdentityApi`'s `AuthorizationController.RequestAuthorization` already branches one endpoint on caller scope, T-020): `staffidentity.admin` → admin-driven reset, sets `MustChangePassword = true`, no current-password check; `staffidentity.authenticate` → self-service change, caller must supply their own current password, clears `MustChangePassword`. Implement the branching explicitly (an `if` on which scope claim is present), not as two separate routes.

**Validation rules to get right, not just endpoint shapes:**
- `SupervisorStaffId` assignment (`docs/DESIGN.md` 2.4): must reference an `Active` `Supervisor`, never an `Administrator` or any other role; self-reference forbidden; this check runs on *every* write of the field (endpoint 7 and, if you ever let `POST /v1/staff` set it directly at creation — the design doesn't call for that, so don't add it).
- `Terminated` is a true dead end: no test, no code path, no admin override should move a `Terminated` account back to `Active`/`Suspended`.
- Username uniqueness is case-insensitive at the database level (a citext column, or a normalized-lowercase shadow column with a unique index — your call, consistent with whatever the sibling services already do for a similar uniqueness constraint, if any exist to check).

## Explicitly out of scope for this task

Everything `docs/DESIGN.md` Section 1.2 and Section 6 already list as deferred: more than one role per staff member, MFA, branch/jurisdiction scoping, JIT/break-glass access, SSO/external directory integration, a formal role-change-approval workflow, breached-password checking. Also out of scope: anything about `OpsApi` itself (not built yet, not this repo) — this service never calls out to anything, and nothing in this task should assume `OpsApi` exists yet.

## Tests

- Authenticate: success; wrong password; wrong username; `Suspended`/`Terminated` account — confirm all four failure shapes are byte-for-byte identical (`401`, same generic message) so no test accidentally locks in a distinguishing signal. Audit row written for both success and failure.
- Create staff: happy path; idempotent retry with the same key+body; same key with a different body → `409`; the one-time credential is never returned on a second (idempotent-replay) response the same way it was on the first — decide and document which: re-returning a secret on every retry is a real exposure risk worth avoiding deliberately, not accidentally.
- Role/status/supervisor PATCHes: each audited independently; `Terminated` is a dead end (attempt every other status as a source, assert 400 trying to leave it); supervisor validation rejects a non-`Supervisor` target, a `Suspended` supervisor, and self-reference.
- Password: admin reset sets `MustChangePassword`; self-service change requires and verifies the current password, clears the flag; wrong current password on self-service → clear failure, not a silent no-op.
- `GET` list with and without the `supervisorStaffId` filter; paging.
- API-key auth and scopes: no key → 401; wrong scope for the endpoint → 403; confirm `staffidentity.authenticate` alone cannot call any `staffidentity.admin`-only endpoint and vice versa (except endpoint 8's deliberate dual-scope branching).
- Health checks respond.
- Migration test confirming the schema matches the entities above, mirroring the sibling services' migration-test style.
- The Administrator seed exists exactly once after migrations run, with `MustChangePassword = true` and whatever password-bootstrap mechanism you chose actually working end-to-end (not just that a row exists).

Run build and tests yourself (Docker should be available for Testcontainers — if it is not in your sandbox, say so explicitly and report exactly which suites you could and couldn't run).

## PR

Open a pull request from `feat/foundation` against `main` on `AbubakarImam/Garbati.StaffIdentityApi` (branch-protected — get `test`/`docker` CI green on your own branch before opening it). Use `garbati-platform/templates/pull_request_template.md` if it fits, or a sibling repo's PR style otherwise. Include what changed, how it was tested, what you could not verify, every judgment call this task file left open (the Administrator-seed password-bootstrap mechanism, above all), and a draft LEARNING_LOG-style plain-language explanation plus five interview questions with answers, per `AGENTS.md`'s "Done means" checklist. Do not merge — only the owner merges.
