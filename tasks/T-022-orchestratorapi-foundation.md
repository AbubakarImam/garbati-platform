# T-022: OrchestratorApi foundation — project skeleton, account opening, transfer

Target repo: `Garbati.OrchestratorApi` (currently design-only, one commit). Branch: `feat/foundation`. Spec: `docs/DESIGN.md` in full — it's short, read the whole thing. This is the **first** build task for this service; there is no existing code to extend.

---

This is OrchestratorApi's entire v1 scope in one task — unlike IdentityApi, which needed several build tasks, `docs/DESIGN.md`'s whole point is that this version is deliberately minimal (account opening + a plain transfer only, no fee, no reversal, no joint-account multi-approval awareness). Read Section 1.1 for exactly why those are excluded before starting — do not add them back in because they seem like natural extensions.

## Match LedgerApi's and IdentityApi's conventions exactly

`Garbati.OrchestratorApi` is a third sibling service. Per `ARCHITECTURE.md`'s shared conventions and `AGENTS.md` ("match the existing code"), mirror the actual implementations already built in `/Users/USER/Projects/LedgerApi` and `/Users/USER/Projects/Garbati.IdentityApi` — read their real source, not just their docs, for every cross-cutting concern: project layout (`src/Garbati.OrchestratorApi/` with `Controllers/`, `Services/Implementations/`, `Entities/`, `Data/`, `Migrations/`, `Validation/`, `Middleware/`, `Auditing/`, `Authorization/`, `Configuration/`, `Contracts/Requests`, `Contracts/Responses`, `Exceptions/`; `tests/Garbati.OrchestratorApi.Tests/` mirroring the same subfolder structure), `global.json`, API-key auth with hash-only configuration and scopes, correlation-id + exception-handling middleware (`application/problem+json`, the same 400/401/403/404/409/422/500 mapping), an audit logger staged on the same `DbContext` as the change it describes, `/health/live` + `/health/ready`, a two-target Dockerfile, `garbati-platform/templates/ci-dotnet.yml` as the CI starting point, and Testcontainers-based integration tests. Use Postgres port `5435` for local dev (LedgerApi uses `5433`, IdentityApi uses `5434`).

**This service is also a client of both LedgerApi and IdentityApi.** You will need an HTTP client setup (named `HttpClient`s via `IHttpClientFactory`, or a thin typed client per downstream service) configured with this service's own API key for each. Look at how LedgerApi's and IdentityApi's own `ApiKeyAuthenticationHandler`/`ApiKeyOptions` expect the key to be sent (the header name and format) so your outgoing calls match what they expect incoming.

## What to build

### 1. `AccountOpenings` (`docs/DESIGN.md` 1.3/1.4, decisions #1/#2/#6)

Entity: `Id`, `IdempotencyKey`, `CustomerId` (nullable, set once step 1 succeeds), `LedgerAccountId` (nullable, set once step 3 succeeds), `CurrentStep` (`CreateCustomer`/`VerifyKyc`/`CreateLedgerAccount`/`AttachMandate`/`Complete`), `Status` (`InProgress`/`Complete`/`Failed`), `FailureReason` (nullable), `CreatedAt`, `CompletedAt` (nullable).

`POST /v1/account-openings` — request: an `IdempotencyKey` plus whatever customer-creation fields IdentityApi's `CreateCustomerRequest` needs (`FullName`, `JurisdictionCode`, optional `CustomerReference`/`Email`/`PhoneNumber`), plus the new account's `AccountName`, `AccountType`, `CurrencyCode` for LedgerApi's `CreateAccountRequest`. Runs the sequence synchronously in one call:

1. `IdentityApi: POST /v1/customers`, idempotency key = `orch:{operationId}:create-customer`. Store the returned `CustomerId`.
2. `IdentityApi: PATCH /v1/customers/{customerId}/kyc` with a hardcoded, clearly-labeled demo stub decision (`KycStatus: "Verified"`, a placeholder `VerificationType`/`Provider` — **this is explicitly a demo stub, not real verification**; say so in a code comment, matching the spirit of every other "no real X exists yet" comment in this platform).
3. `LedgerApi: POST /api/accounts`, idempotency key = `orch:{operationId}:create-account`. Store the returned `AccountNumber` as `LedgerAccountId`.
4. `IdentityApi: POST /v1/customers/{customerId}/mandates` with `LedgerAccountId` (the account number from step 3) and `Operations: ["View", "Debit", "Credit"]`. **IdentityApi generates its own idempotency key internally for mandate attachment per its own design** — check whether `AttachMandateRequest` actually needs/accepts one from the caller (read the real contract and controller, don't assume); if it does, use `orch:{operationId}:attach-mandate`.
5. Mark `Complete`.

If any step throws, catch it, set `Status = Failed`, `FailureReason`, leave `CurrentStep` at whichever step failed, and return a response indicating the operation's id and failure — do not let an unhandled exception escape as a bare 500 with no operation record.

`GET /v1/account-openings/{id}` — current state.

`POST /v1/account-openings/{id}/retry` — re-runs from `CurrentStep` onward, reusing the exact same derived idempotency keys (so already-completed steps are safe no-ops per downstream idempotency, not re-executed). If `Status` is already `Complete`, this is a no-op that just returns the current state (don't error — matches the project's established "repeat of an already-successful thing is idempotent" convention from `Customer.Suspend`/`Mandate.Revoke`, as opposed to "repeat of a terminal failure" conventions — an account opening retry is the former, since finishing it is still possible).

### 2. Transfer (`docs/DESIGN.md` 1.5/1.6/1.7)

`POST /v1/transfers` — request: `IdempotencyKey`, `RequestedBy` (a `CustomerId` — **this is the caller-supplied, unauthenticated-behind-it value Section 1.6 flags as a known gap; do not build any real authentication here, and say so plainly in the PR, the same way the design doc does**), `DebitAccountId`, `CreditAccountId`, `Amount`, `Currency`, `Narration?`.

1. `IdentityApi: POST /v1/authorization/requests` with `Operation: "Transfer"` (check the real `AuthorizationRequest`/`AuthorizationResponse` contracts directly — `RequestedBy`, `DebitAccountId`, `CreditAccountId`, `Amount`, `Currency`, `IdempotencyKey`). A denial (IdentityApi returns 422) means: stop, return the denial reason, do not call LedgerApi.
2. On approval, IdentityApi returns a signed grant (a JWS string) and a decision id. You **may** verify the grant's signature yourself as defence in depth (`docs/DESIGN.md` 1.7/3.4 pattern) — if you do, you'll need IdentityApi's ES256 public key available in config; check how IdentityApi's own tests verify a grant (`GrantSignerTests` or similar) for the exact claim names/algorithm to mirror. If you decide this is out of scope for a first build, say so explicitly as a judgment call rather than silently skipping it.
3. `LedgerApi: POST /api/transfers` (ordinary single transfer — **not** `/api/chains`, since there's no fee/tax leg). The grant is **not** forwarded into this call (`docs/DESIGN.md` 1.7 — LedgerApi has no field for it yet, decision #47 is still deferred).
4. Return the transfer result (LedgerApi's `TransactionResponse`) plus the authorization decision id — never the grant token itself (same "never log/return the raw grant" rule every other service already follows).

No new persistent entity for a transfer (`docs/DESIGN.md` 1.5 — both downstream calls are already independently idempotent).

### 3. Funding — not a new endpoint

`docs/DESIGN.md` 1.8: funding goes straight to `LedgerApi: POST /api/deposits`, called directly by whoever is driving the demo — no Orchestrator endpoint, no code to write here. Do not build one.

## Service identities and scopes (`docs/DESIGN.md` 1.10)

Need, as a configured client of each downstream service: from IdentityApi — `identity.customer.write`, `identity.kyc.manage`, `identity.mandate.write`, `identity.authorization.check`; from LedgerApi — `ledger.accounts`, `ledger.transfer`. This service's **own** API, in turn, needs its own scope(s) for whoever calls *it* (the demo driver / Swagger) — define a sensible scope name (e.g. `orchestrator.operations`) covering both new endpoints; there's no existing convention to mirror here since this is the first caller-facing surface this service has, so use your judgment and say what you chose in the PR.

## Explicitly out of scope

Fee/tax (no ProductApi, `docs/DESIGN.md` decision #7). Reversal (decision #8, T-015/T-020 exist now but Orchestrator-side reversal wiring was never part of this design — if you think it should be added now that IdentityApi supports it, don't; that's a separate future task, not this one). Joint-account/multi-holder-approval awareness in the transfer flow (IdentityApi's `/v1/authorization/requests` already handles the `RequiredApprovals > 1` branch server-side if it ever applies — this service doesn't need its own logic for it, since account-opening in this task only ever attaches a single mandate). Real Channel/customer authentication (decision #3, a known, stated gap). Any key-rotation/compromise handling beyond whatever minimal config is needed to hold IdentityApi's public key, if you build grant verification at all.

## Tests

- Account opening: the full happy path (all 4 steps, ends `Complete`, `CustomerId`/`LedgerAccountId` populated). A failure at each step (mock/stub the downstream call to fail) leaves `Status = Failed` with the right `CurrentStep` and `FailureReason`, and nothing downstream of the failure was attempted.
- Retry: after a step-3 failure, retry resumes from step 3, not step 1 (steps 1-2 are not re-called — verify this with a call-count assertion on your HTTP client mock/stub, not just the end state). Retrying an already-`Complete` operation is a no-op, not an error.
- Transfer: a denial from IdentityApi means LedgerApi is never called (verify with a call-count assertion). An approval calls LedgerApi and returns its result plus the decision id, never the grant string.
- Idempotency: retrying `POST /v1/account-openings` with the same key and same body returns the same operation, not a second one (check what convention T-016/T-017 established for this exact pattern and mirror it, including whether a different body with the same key is 409).
- API-key auth: missing/wrong scope on both endpoints.
- Health checks respond.
- Migration test for `AccountOpenings`' schema.

Since LedgerApi and IdentityApi are real, running services' *code* (not live network services) available in this environment, decide and say in the PR whether you're testing against them via real HTTP calls to locally-run instances, or via mocked/stubbed HTTP clients — either is reasonable for a first build, but be explicit and consistent, and say what you could not verify end-to-end as a result (e.g., if you stub both downstream services, you have not actually proven the request/response shapes match the real deployed contracts — say so).

Run build and tests yourself (Docker should be available for Testcontainers — if not, say so explicitly, matching every prior task's honesty standard). Open a pull request from `feat/foundation` against `main` on `AbubakarImam/Garbati.OrchestratorApi` (this repo has only force-push/deletion protection so far, no CI yet — add the CI workflow as part of this task, per the "match conventions" instruction above, and make sure it's green on your branch). Include what changed, how it was tested, what you could not verify, every judgment call and why, and the five-question/plain-language LEARNING_LOG draft, per `AGENTS.md`'s "Done means" checklist. Do not merge.
