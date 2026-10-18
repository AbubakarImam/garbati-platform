# QA Report — Garbati.OrchestratorApi PR #1 (T-022 foundation)

**Verdict: Approve** (no blockers; one nit sent back for a cheap pre-merge fix)

## Contract fidelity — the main focus of this review, PASS

Diffed every one of the six client DTOs this PR constructs (`CreateCustomerRequest`, `KycDecisionRequest`, `AttachMandateRequest`, `AuthorizationRequest`/`AuthorizationResponse`/`AuthorizationDecisionResponse` on the IdentityApi side; `CreateAccountRequest`/`AccountResponse`, `TransferRequest`/`TransferResponse` on the LedgerApi side) against the sibling repos' actual `Contracts/Requests`/`Contracts/Responses` source. All fields, types, nullability and order matched exactly. Route paths, HTTP verbs, header name (`X-Api-Key`), and status-code/`ProblemDetails` shapes were also confirmed against the siblings' actual controllers/middleware. The builder's claim that its `Clients/` test suite uses "real response bodies copied from their actual controllers/middleware" checks out — not stale or invented.

## Task fidelity

All four task-file deliverables built (account-opening 4-step sequence + GET + retry, transfer with authorization-then-transfer, no funding endpoint, scopes) — nothing extra. Grep for `deposit|fee|reversal|jointapproval|requiredapproval` across `src`/`tests` returned zero hits.

## Account opening

4 steps, derived idempotency keys (`orch:{operationId}:create-customer` etc.), a reserved-prefix validator stopping a caller from forging those keys, failure stops exactly at the failing step with nothing downstream attempted (verified via call-count assertions), retry resumes from `CurrentStep` with call-count proof steps 1–2 aren't re-invoked, retrying a `Complete` operation is a no-op. The mandate-attach step's lack of an `IdempotencyKey` was independently confirmed against the real contract, and the 409→"treat as success" retry path is sound and tested.

## Transfer

A 422 denial from IdentityApi is proven (call-count assertion) to prevent any call to LedgerApi. The grant string is never logged, persisted, or returned — confirmed by grep and by a reflection-based test asserting the response type has no such property.

## Judgment calls — both held up

- Skipping grant self-verification: correctly tied to the "adds a new dependency" human-only gate rather than decided unilaterally; LedgerApi doesn't verify the grant either yet, so no load-bearing safety property is actually being skipped today.
- Testing against mocked clients: honestly scoped and disclosed, with its claimed mitigation (fidelity-checked `Clients/` tests) confirmed real.

## Scopes/auth, security, build

Two scopes (`orchestrator.account-openings`, `orchestrator.transfers`) with a concrete rationale; API keys hash-only with fixed-time comparison, matching both siblings; 401/403 tested per-endpoint; `.dockerignore` present and correct, `bin`/`obj` properly git-ignored. `dotnet build -c Release`: clean. `dotnet test -c Release`: **59/59 passing** against real Postgres Testcontainers. `gh pr checks 1`: `test` and `docker` both green.

## Findings

1. **[Nit — sent back for a pre-merge fix]** `appsettings.Development.json`'s comment claims placeholder outbound keys (`dev-orchestrator-identity-key`, `dev-orchestrator-ledger-key`) are "committed openly the same way" as the siblings' dev keys. QA computed `SHA256("dev-orchestrator-identity-key")` directly and confirmed it does **not** match IdentityApi's configured `dev-orchestrator` hash, and LedgerApi's dev config has no orchestrator-caller entry at all — these are non-functional placeholders, unlike the siblings' which work out-of-the-box. Not a functional bug (the README correctly says to replace them), but the comment overstates readiness. Sent back for a reword.
2. **[Nit — process note, no code action]** `docs/DESIGN.md`'s status line reads "ratified and built" as part of the PR itself, ahead of QA approval/owner ratification — other services in this platform update status only after that step. Harmless here; flagged for the coordinator to keep the convention consistent going forward.
3. **[Informational, not a defect]** `OperationIdGenerator` uses `Random.Shared`, not a CSPRNG — correct for a public, non-secret display identifier backed by a real DB unique-index retry loop, exactly mirroring the sibling pattern. Noted only because an operation id can be mistaken for something that needs to be unguessable; it isn't used that way here.

## What could not be checked

- No real local IdentityApi/LedgerApi instances were spun up for an actual end-to-end HTTP smoke test — exactly the gap the builder disclosed, not something it claimed to have closed. The contract-fidelity check (a direct source diff) is strong evidence but not the same as a live call.
- Did not read every validator/test file line-by-line against the full diff (4432 additions) — verified the files most relevant to money safety, contracts, and auth, plus ran the real build/tests.
- Did not trace every exception branch's audit-log coverage (unexpected 500s outside the explicit `AuthorizationDeniedException` catch fall through to a generic 500 with no audit row) — acceptable for a demo-stage build, not exhaustively traced.

## Five interview questions for the owner

1. **Q: If you started all three services locally right now using only their committed defaults, would a transfer actually succeed?**
   A: No — the placeholder key's hash doesn't match what IdentityApi's dev config expects, and LedgerApi's dev config has no orchestrator entry yet. The README does say to edit these first, so it's not broken as shipped, but worth knowing before trying a real smoke test.
2. **Q: If the service crashes right after creating the ledger account but before attaching the mandate, and you hit retry, how do you know a second ledger account won't get created?**
   A: The request to LedgerApi always carries the same derived key, and LedgerApi's own idempotency rule means a repeat of that exact key just hands back the account already made — and the Orchestrator's own code never re-enters that step once `CurrentStep` has moved past it.
3. **Q: The mandate-attach step has no idempotency key at all. What happens if it's retried twice in a row?**
   A: The first retry succeeds and the operation completes. If retried again anyway, IdentityApi rejects it with "already has an active mandate" (409), and the Orchestrator specifically catches that and treats it as already-done, not a failure.
4. **Q: Why two separate scopes instead of one, when the design doc didn't require either?**
   A: So a caller wired up only for transfers doesn't automatically also get account-opening rights — the design doc only specified what the Orchestrator presents outward, not what it demands of its own callers, so this was a reasonable call left open.
5. **Q: What's the actual risk of skipping grant self-verification today?**
   A: Low — building it needs a new crypto dependency, which correctly needed the owner's sign-off rather than a unilateral add, and LedgerApi doesn't check the signature either yet, so nothing downstream would behave differently even if the Orchestrator verified it. The one safety property that matters today — never calling LedgerApi after a denial — is already built and tested without needing it.
