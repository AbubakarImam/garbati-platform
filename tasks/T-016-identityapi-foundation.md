# T-016: IdentityApi foundation — project skeleton, Customer, KYC

Target repo: `Garbati.IdentityApi` (currently design-only, two commits: v1 draft, v2 ratified). Branch: `feat/foundation`. Spec: `docs/DESIGN.md` — read it in full first, especially Sections 1, 2.1, 2.3, 4, 6, 7, and the decisions log (Section 9).

---

**This is the first of two build tasks.** This task builds the project skeleton plus the `Customer` and KYC pieces. Mandates and authorization-grant issuance are a separate follow-up task (T-017, not started yet) — **do not build `Mandates`, `MandatePermissions`, the joint-account entities (`MandateApprovalPolicies`/`PendingApprovalRequests`/`ApprovalRecords`), or any `/v1/authorization/*` or `/v1/mandates/*` endpoint in this task.** Stopping at a clean boundary matters more than getting ahead — T-017 will build directly on what you leave here.

## Why this split

`docs/DESIGN.md` is a big document (joint accounts, signed tokens, multi-holder approval). Building all of it in one pull request would be too large to review properly. This task is scoped to exactly what's needed to create a customer and move them through KYC — the foundation everything else sits on.

## Match LedgerApi's conventions exactly

`Garbati.IdentityApi` is a sibling service to `LedgerApi` (cloned alongside it — check `/Users/USER/Projects/LedgerApi` or wherever this environment has it checked out). Per `ARCHITECTURE.md`'s shared conventions (Section 5) and `AGENTS.md` ("match the existing code"), mirror LedgerApi's actual implementation, not just its docs, for every cross-cutting concern:

- **Project shape**: `src/Garbati.IdentityApi/` with `Controllers/`, `Services/` (an `Implementations/` subfolder if LedgerApi uses one), `Entities/`, `Data/` (DbContext), `Migrations/`, `Validation/`, `Middleware/`, `Auditing/`, `Authorization/`, `Configuration/`, `Contracts/Requests` and `Contracts/Responses`, `Exceptions/`. `tests/Garbati.IdentityApi.Tests/` mirroring LedgerApi's test folder structure (`Services/`, `Controllers/`, `Validation/`, `Configuration/`).
- **`global.json`**: same .NET SDK version/rollForward LedgerApi pins.
- **API-key authentication + scopes**: copy LedgerApi's `ApiKeys:Clients` pattern (hash-only configuration, never the raw key) exactly, including its `IValidateOptions` startup validation. Use this service's own scopes from `docs/DESIGN.md` Section 5.2 (only the ones relevant to this task: `identity.customer.read`, `identity.customer.write`, `identity.kyc.manage` — leave `identity.mandate.*`, `identity.authorization.check` and `identity.ops.freeze` for T-017, but it's fine to define the full scope list now if LedgerApi's own `LedgerScopes`-equivalent class is simplest to write once).
- **Correlation ID middleware, exception-handling middleware (`application/problem+json`, matching LedgerApi's status-code mapping conventions from `ARCHITECTURE.md` Section 5: 400/401/403/404/409/422/500), Serilog request logging** — copy LedgerApi's middleware, adapted to this service.
- **Audit logging**: copy LedgerApi's `IAuditLogger`/`AuditLogger` pattern (staged on the same `DbContext`, committed by the same `SaveChangesAsync` as the change it describes — `docs/DESIGN.md` Section 6.3). Audit at least: customer creation, KYC verification outcomes/tier changes, customer suspension/reactivation (including the `Closed`/`Deceased` cascade — see below).
- **Health checks**: `/health/live`, `/health/ready` (checks the database), matching LedgerApi's.
- **Dockerfile + CI workflow**: copy `garbati-platform/templates/ci-dotnet.yml` as `.github/workflows/ci.yml` (per `TASKS.md` T-000's established pattern) and LedgerApi's `Dockerfile` shape (API image + migrator image targets). Leave the `deploy` job's Azure specifics as a close copy of LedgerApi's for now — note in the PR if anything there needs owner-specific values (resource group names, etc.) before it can actually deploy; getting the workflow syntactically correct and the `test`/`docker` jobs actually passing is what matters for this task.
- **Postgres via Testcontainers** for integration tests, no local database needed to run `dotnet test` — same as LedgerApi.
- **Idempotency key validation**: `docs/DESIGN.md` doesn't require idempotency on the endpoints this task builds (`POST /v1/customers` isn't listed with an idempotency key in the design — check Section 5.1 carefully; if you believe it should have one by analogy with LedgerApi's `POST /api/accounts`, say so in the PR as a question rather than silently adding it, since that would be a scope decision beyond this task).

## What to build

**Entities** (`docs/DESIGN.md` Section 2.1, 2.3, 7.1):
- `Customer`: `Id`/`CustomerId`, `CustomerReference`, `CustomerStatus` (`Active`/`Suspended`/`Deceased`/`Closed`), `KycStatus` (`Pending`/`Verified`/`Rejected`/`Expired`), `KycTier`, `JurisdictionCode`, `CreatedAt`, `UpdatedAt`.
- `CustomerIdentityData`: sensitive fields separated from `Customer` itself (Section 2.1's "sensitive identity fields should be separated from ordinary operational fields"). Keep this minimal for now — a small set of fields (e.g. name, contact info) is enough; this task is not a full KYC-data-model build, just the separation itself. Do not design a BVN/NIN column without re-reading Section 6.2's "never log raw BVN/NIN" rule and applying the same care to storage (encrypt at rest if LedgerApi's stack gives you an easy, standard way to do this; otherwise flag it as a follow-up rather than guessing at crypto).
- `KycVerifications`: `Id`, `CustomerId`, `VerificationType`, `Provider`, `ProviderReference`, `Status`, `ResultCode`, `VerifiedAt`, `ExpiresAt`.
- `KycTierPolicies`: `JurisdictionCode`, `TierCode`, `EffectiveFrom`, `EffectiveTo`, required-evidence/eligible-operations fields (decision #11: configurable policy data, not hardcoded — keep the policy *shape* flexible, but you do not need real Nigerian tier values; a placeholder seed is fine, clearly marked as a placeholder, since the real values are still pending compliance confirmation per `garbati-platform/research/nigerian-regulatory-requirements.md`).
- `AuditEvents`, `OutboxMessages` (Section 7.1) — build these now since customer-status-change auditing and the freeze outbox both need them; `OutboxMessages` rows can be written in this task with no consumer yet (the Orchestrator, which would consume them, has no code yet either — that's expected and fine, say so in the PR).

**Endpoints** (`docs/DESIGN.md` Section 5.1, only these four):
- `POST /v1/customers` — create a customer record (`CustomerStatus = Active`, `KycStatus = Pending`).
- `GET /v1/customers/{customerId}` — redacted view for an authorized caller (never raw `CustomerIdentityData` fields like BVN/NIN — Section 6.2).
- `PATCH /v1/customers/{customerId}/kyc` — record a KYC status/tier decision.
- `POST /v1/customers/{customerId}/suspend` and `POST /v1/customers/{customerId}/reactivate` — build these now too, since they're simple status transitions with no mandate dependency, and the `Closed`/`Deceased` cascade behavior (below) needs somewhere to hang. If the design implies a separate `close`/`mark-deceased` endpoint rather than overloading `suspend`, follow the design's intent, not this task file's phrasing — check Section 2.1's status table and pick sensible endpoint names if the design doesn't spell them out; say what you chose and why in the PR.

**Closed/Deceased cascade (`docs/DESIGN.md` Section 2.1, decision #14):** when a customer's status becomes `Closed` or `Deceased`, every mandate they hold should be revoked and a freeze should be queued (via `OutboxMessages`) for every account where it was their only active mandate. **Since `Mandates` doesn't exist until T-017, you cannot fully implement the mandate-revocation half of this cascade yet.** Build what you can: the status transition itself, its audit event, and the *shape* of the outbox message this cascade will eventually enqueue (even if, for now, there are no mandates to look up, so no outbox row is actually created on this path yet). Leave a clear, explicit note — in a comment and in the PR description — that T-017 must come back to this exact spot and wire in the real mandate lookup. Do not invent a fake mandate check to make this look more finished than it is.

**Explicitly out of scope for this task** (confirmed, not ambiguous): `Mandates`, `MandatePermissions`, `MandateApprovalPolicies`, `PendingApprovalRequests`, `ApprovalRecords`, any `/v1/mandates/*`, `/v1/accounts/*`, or `/v1/authorization/*` endpoint, JWT/grant signing of any kind, and the key-rotation/compromise configuration (Section 4.3) — none of that is needed to create a customer and move them through KYC. Reversal authority is not designed anywhere yet (T-015) — nothing in this task should reference it.

## Tests

- Customer creation, read (confirm no sensitive `CustomerIdentityData` fields leak into the response), KYC status/tier update.
- Suspend: no new grants would be issuable afterward — there's no grant-issuance code yet to test this against directly, so instead assert the status transition and its audit event; note in the PR that the "no new grants" guarantee itself is untestable until T-017 exists.
- Closed/Deceased: the status transition, its audit event, and that attempting it twice (already-Closed customer) behaves sensibly (your choice — 409 or a no-op; say which and why).
- API-key auth and scopes: a request with no key is 401; a key without the right scope is 403.
- Health checks respond.
- Migration test: confirms the schema matches the entities above (mirror LedgerApi's migration-test style).

Run build and tests yourself (Docker should be available for Testcontainers — if it is not in your sandbox, say so explicitly, exactly as the LedgerApi chains builder did, and report exactly which suites you could and couldn't run).

## PR

Open a pull request from `feat/foundation` against `main` on `AbubakarImam/Garbati.IdentityApi` (branch-protected: the `test` and `docker` CI jobs you add in this same task must pass before it can merge — so get CI green on your own branch before opening it, not after). Use `garbati-platform/templates/pull_request_template.md` if it fits this repo, or LedgerApi's own PR style otherwise. Include what changed, how it was tested, what you could not verify, every place you made a judgment call the task file left open, and the explicit note about the Closed/Deceased cascade's unfinished mandate half. Draft a LEARNING_LOG-style plain-language explanation plus five interview questions with answers, per `AGENTS.md`'s "Done means" checklist. Do not merge — only the owner merges.
