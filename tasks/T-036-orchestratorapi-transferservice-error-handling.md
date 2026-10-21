# T-036: OrchestratorApi — give `TransferService` the same defensive error handling `ReversalService` already has

Target repo: `Garbati.OrchestratorApi`. Branch: `fix/transferservice-ledgerapi-failure-handling`.

## Context

Found by an external QA pass (opencode, 21 Oct 2026) and independently confirmed against the real code: `TransferService.TransferAsync` (`src/Garbati.OrchestratorApi/Services/Implementations/TransferService.cs`) has **no error handling at all** around its call to `ledgerApiClient.TransferAsync` — unlike `ReversalService.ReverseAsync`, which was specifically hardened during T-030/T-034 (this platform's own real QA process, not a hypothetical) to handle exactly this class of failure.

Today, if `ledgerApiClient.TransferAsync` throws (a `DownstreamApiException` on a `404`/`409`/`500`, a timeout, any unexpected exception) **after** `IdentityApi` has already authorized the transfer:
- The exception propagates unhandled to `ExceptionHandlingMiddleware`, which maps it to a bare `500` with no decision id, no reference, nothing to trace or retry against.
- Only the earlier `TransferAuthorized` audit row exists — no `TransferPosted` row records that the transfer was attempted and what happened, unlike every other path in this method and unlike `ReversalService`'s equivalent catch-all.
- The caller has no way to tell "this definitely failed, safe to retry" apart from "this might have actually succeeded, don't blindly retry with a new key."

This is real (confirmed by reading the actual, current `TransferService.cs` — there is no `try`/`catch` around the `ledgerApiClient.TransferAsync` call at all) and is an asymmetry worth closing, not a hypothetical.

## What to build

Read `ReversalService.cs`'s current shape first (`src/Garbati.OrchestratorApi/Services/Implementations/ReversalService.cs`) — this task is explicitly "bring `TransferService` up to the same standard," not "invent a new pattern." In particular:

- `LedgerApi`'s `TransferAsync` endpoint (unlike its `ReverseAsync` endpoint) **returns** a body with `Status: "Failed"` on a business-rule failure rather than throwing (confirmed: `LedgerApiClient.TransferAsync`'s own comment says "LedgerApi's TransfersController maps its own TransactionStatus.Failed result to HTTP 422, but the body is still a normal TransferResponse... read the same way as a 200"). `TransferService` already handles that case correctly today — **this task is not about the 422-with-body case**, which is already fine.
- This task is about what happens when `ledgerApiClient.TransferAsync` **throws** instead of returning a body — a `404`/`409`/`500`/timeout/any other `DownstreamApiException`, or a non-`DownstreamApiException` transport-level failure. Today, nothing catches this at all.

**Add a catch-all around the `ledgerApiClient.TransferAsync` call** (mirroring `ReversalService`'s catch-all shape, adapted — there is no outcome to report to `IdentityApi` for a transfer the way there is for a reversal, since `IdentityApi`'s grant/decision model for a transfer doesn't have an equivalent "record what happened" endpoint; check `IdentityApiClient`/`IdentityApi`'s real contract before assuming one exists, and if none does, that's fine — the fix here is about **auditing the failure clearly and giving the caller something to act on**, not inventing a new cross-service call):

1. On any exception from `ledgerApiClient.TransferAsync`: write a `TransferPosted` audit row (same `Action` constant already used for the success case) with `Status = "Unconfirmed"` and `Details` describing the failure (same shape `ReversalService`'s catch-all uses: `$"DecisionId {authorization.Decision.DecisionId}; LedgerApi failure: {ex.Message}"`), then `SaveChangesAsync`, then rethrow the original exception unchanged (do not swallow it, do not fabricate a response — the caller still needs the real error).
2. This gives the same two things `ReversalService` already provides for its own downstream call: a durable audit trail even on an unexpected failure, and a clear signal (via the `decisionId` already present in the audit row, traceable by an operator) that this transfer's outcome is unconfirmed rather than silently lost.

**Do not** add a reporting-outcome call to `IdentityApi` unless you confirm a real endpoint exists for recording a transfer's outcome the way `RecordReversalOutcomeAsync` does for reversals — check `IdentityApiClient`'s actual interface and `IdentityApi`'s actual `AuthorizationController` before assuming. If none exists (likely, since transfers don't have the same deferred-grant-verification machinery reversals do — decision #47/D-6 is still deferred for both, but reversal outcome-reporting was a deliberate, separate addition in T-020), say so plainly in the PR and don't invent one; this task is scoped to the audit-trail gap only.

## Scope

- `TransferService.cs` only — wrap the existing `ledgerApiClient.TransferAsync` call, add the catch-all audit-and-rethrow block described above.
- Tests: mirror `ReversalServiceTests.cs`'s own test for this exact scenario (`Approve_WhenOrchestratorApiCallThrows...` or whichever test proves the catch-all works for reversals) — create a test where `FakeLedgerApiClient.TransferHandler` throws, confirm the original exception still propagates (not masked), and confirm a `TransferPosted` audit row with `Status = "Unconfirmed"` was actually written (query the audit table directly, the same way this platform's own QA process has verified this exact class of fix before — a call-count assertion alone is not enough).
- Do not touch `ReversalService.cs`, `AccountOpeningService.cs`, or anything else.

## Out of scope

- The 422-with-body case (`LedgerApi` returning `Status: "Failed"`) — already correctly handled, not part of this task.
- Any new call to `IdentityApi` to report a transfer's outcome — only add this if a real, already-existing endpoint for it is confirmed; do not design a new cross-service contract as part of this fix.
- D-16 (the separate, already-tracked idempotency-dedup gap on `IdentityApi`'s authorization endpoints) — unrelated, not this task's job.

## PR

Open a pull request from `fix/transferservice-ledgerapi-failure-handling` against `main` on `AbubakarImam/Garbati.OrchestratorApi` (branch-protected — get `test`/`docker` CI green first). Include what changed, how it was tested, and confirm explicitly whether an `IdentityApi` outcome-reporting endpoint for transfers exists or not (and why that did or didn't change the fix). Do not merge — only the owner merges.
