# QA Report — Garbati.OpsApi PR #2 (T-034: reversal and customer-edit maker-checker workflows)

**Verdict: Approve with nits** (no blockers)

## 1. Scope fidelity — PASS

Touches exactly 31 files, 1452 insertions / 3 deletions. Every non-test, non-doc file is a pure addition except `Program.cs` (+2 DI lines) and `ExceptionHandlingMiddleware.cs` (+5 lines mapping the three new domain exceptions). No existing endpoint, session, or cascade-revocation code from T-033 is touched.

## 2. Reversal execution — all three claims verified directly against code and tests

- **Client-supplied idempotency key at creation**: validated by the existing `IdempotencyKeyRule`, not server-generated. `CreateAsync` has no downstream dependency at all, so a malformed create has no path to reach `OrchestratorApi`.
- **Deterministic idempotency key on approval**: code uses `$"ops:{request.Id}"`. A real test captures the actual key sent and asserts it matches this derivation, not a fresh random one.
- **`RequestedBy` is the original requester, not the approver**: verified by a real test asserting both `requesterId == last.RequestedBy` and `approverId != last.RequestedBy`.

## 3. Hard maker-checker rule — verified live, not cached, for both workflows

`SessionAuthenticationHandler` re-resolves role from the `OpsSession` DB row on every request — no cached claim anywhere. A dedicated test mutates a session's role directly in the database mid-flow (the exact row the handler re-reads) and confirms the approve action then correctly 403s — real proof the check isn't reading anything stale from login time. Self-approval and non-Supervisor-approver cases both tested with call-count assertions confirming `OrchestratorApi` was never reached on rejection.

## 4. Role-scoped visibility — verified, one real test gap found and closed

Reversals: `Teller`/`CustomerServiceOfficer` see only their own; full-oversight roles see everything; `Administrator` gets 403 on list. Customer-edits: only `CustomerServiceOfficer` (own) and `Supervisor` (everything) have any access; every other role gets 403.

**Gap found:** no test proved the Supervisor-specific narrowing the task spec actually calls for — "every `PendingApproval` request plus every request *they've personally decided*," not "decided by anyone." The code was correct by inspection; nothing proved the negative case. **Closed by the coordinator**: added a test where a second Supervisor correctly cannot see a first Supervisor's already-decided request, in either the list or a direct `GET` by id (confirmed this 404s, not 200 — a real role-visibility boundary, not the expiry-related "GET still works" exception used elsewhere in this same file). 119/119 tests passing (was 118/118, +1).

## 5. `OpsCustomerEditRequest` never calls `IdentityApi` on approval — confirmed two ways

The service class has no `IIdentityApiClient` dependency at all — it cannot call it even by mistake. A test captures all four fake-client call counters across a full create→approve cycle and confirms every one stays at zero.

## 6. Lazy-expiry-aware listing — verified as read-only for both workflows

The expiry cutoff is a `.Where(...)` query condition only; `Expired` is a defined enum value that is never actually written anywhere. `GET` by id still returns the real, stale status; an approve attempt against a stale row still correctly fails live (409). Full-oversight roles are confirmed to still see an expired row in list results (not filtered for them, matching their mandate).

## 7. Scope fidelity re: IdentityApi/OrchestratorApi — confirmed

No code path to the new `PATCH /v1/customers/{customerId}/details` exists anywhere in this PR — grepped and confirmed. `orchestrator.reversals` is correctly wired end to end. T-035's branch was still unmerged at PR time, matching this PR's own disclosure and the task's explicit "doesn't matter either way" framing.

## 8. Downstream contract fidelity — exact match, verified against real source

`Clients/Contracts/Orchestrator/{ReversalRequest,ReversalResultResponse}.cs` are field-for-field, order-for-order identical to `OrchestratorApi`'s real contracts, read directly.

## 9. Build and tests — ran independently, confirmed

`dotnet build -c Release`: 0 warnings, 0 errors. `dotnet test -c Release`: **119/119 passed** (after the coordinator's added test). CI: `test`/`docker` pass, `deploy` correctly skipping on a non-default branch.

## Findings

1. **[Should fix → Decided]** No row-level lock guards two concurrent `approve` calls on the same `OpsReversalRequest` — correctness depends entirely on `OrchestratorApi`'s own idempotency dedup (a real, already-proven backstop from T-030) rather than anything OpsApi enforces itself. Owner asked explicitly; decided as **D-20: documented v1 limitation**, written into `docs/DESIGN.md` directly.
2. **[Should fix → Fixed]** Missing test for Supervisor cross-visibility (see §4) — closed with a new regression test, 119/119 passing.
3. **[Nit]** `IIdentityApiClient` has no structural "exposes exactly these N methods" test, unlike the other two clients — currently correct, just less future-proof against an accidental addition.
4. **[Nit]** `Currency` validation only checks length == 3, not alphabetic/uppercase — pre-existing pattern elsewhere on this platform, not a regression introduced here.
5. No secrets/sensitive data logged anywhere in the new code. `OpsRequestIdGenerator` correctly uses non-cryptographic `Random.Shared` for public, non-secret identifiers only.

No Blockers.

## What QA could not check

- No real, live `OrchestratorApi` instance was exercised — this suite tests against a hand-rolled fake client, same disclosed limitation as T-033's own foundation.
- Did not test the deployed Azure environment.

## Interview Q&A

1. **If two tabs both click "Approve" at nearly the same instant, what stops the money from reversing twice?** Nothing at OpsApi's own database layer — both calls carry the identical deterministic idempotency key, and `OrchestratorApi`'s own dedup treats the second as a replay. A real, disclosed design choice (D-20), not an oversight.
2. **Why does a Supervisor see every pending reversal but only their own decided ones, while for customer-edits a Supervisor sees literally everything?** Matches the design table exactly — reversals: "pending + personally decided"; customer-edits: "everything." Now both directions are actually tested, not just the pending-queue case.
3. **Does the database ever literally say "Expired"?** No — it's a purely lazy, read-time filter on list results; a `GET` by id and an approve attempt both still see the real status.
4. **Why can't `CustomerEditRequestService` call IdentityApi even by mistake?** Its constructor simply never takes an `IIdentityApiClient` — there's no reference to call.
5. **Does it matter that T-035 wasn't merged yet when this was built?** No — the task required this workflow to stop at `Approved` regardless of whether T-035 exists, and it correctly never calls it either way.
