# QA Report — Garbati.OrchestratorApi PR #5 (T-036)

**Verdict: Approve** (no blockers, no should-fix items)

## 1. Scope — PASS

Exactly 3 files touched: `TransferService.cs`, `TransferServiceTests.cs`, `docs/DESIGN.md`. No changes to `ReversalService.cs`, `AccountOpeningService.cs`, or anything else.

## 2. Catch scoping and rethrow — verified correct

The new `try`/`catch` wraps only the `ledgerApiClient.TransferAsync` call. The earlier `AuthorizationDeniedException` catch (around the `RequestAuthorizationAsync` call) is a separate, non-overlapping try/catch — confirmed unaffected. The new catch uses bare `throw;`, re-raising the original exception with its original type and stack trace — not wrapped, not swallowed.

## 3. 422-with-body path — untouched, confirmed via diff

The existing `transfer.Status`/`FailureReason` handling after the try/catch is unchanged byte-for-byte. That path never reaches the new catch because LedgerApi returns it as a normal response value, not a thrown exception.

## 4. New test — genuinely proves what it claims

Makes the fake LedgerApi client throw a `DownstreamApiException(500, ...)`, asserts the *same* exception (`ServiceName`/`StatusCode`) reaches the caller, and separately queries `ctx.AuditLogs` directly via `SingleAsync` for the `TransferPosted`/`Unconfirmed` row — a real database read, not a call-count check. Each test resets the DB first, so the query can't be accidentally satisfied by a stale row.

## 5. No IdentityApi transfer-outcome endpoint — independently confirmed

Read `IIdentityApiClient.cs` directly: only `RecordReversalOutcomeAsync` exists. Read `Garbati.IdentityApi`'s actual `AuthorizationController.cs` directly: only `POST .../reversal-outcome` exists, gated by `identity.ops.reversal`. No transfer-outcome route exists anywhere. The builder's claim holds.

## 6. Shape comparison against `ReversalService.cs` — correct given the constraint above

Reversal's catch-all does two things transfer's can't (report an outcome to IdentityApi, then audit that report's own failure if it also fails) — since neither has an equivalent for transfers, the audit-write portion that *is* possible is equally thorough: same `Details` format, same `SaveChangesAsync` discipline, same shared `dbContext` transaction.

## 7. Audit transaction discipline — correct

`IAuditLogger.Record` stages the entity; `SaveChangesAsync` (called before the rethrow, same `dbContext` instance used throughout) commits it — satisfies "audit written in the same transaction as the change it describes," same as every other path in this file.

## 8. Build/tests — ran independently, confirmed

`dotnet build`: 0 warnings, 0 errors. `dotnet test`: **83/83 passed** (was 82/82, +1, 0 removed/weakened). CI (`test`/`docker`) green on the PR; `deploy` correctly skipped.

## 9. Sensitive-data check — clean

`ex.Message` in `Details` comes from `DownstreamApiException`, which deliberately excludes the downstream response body (could contain customer-identifying details) — same accepted pattern already used in `ReversalService`.

## Findings

1. **[Nit]** If `SaveChangesAsync` itself throws inside the new catch block (e.g., DB unavailable at that exact moment), that new exception — not the original LedgerApi failure — would propagate, masking the real cause. A pre-existing, already-accepted risk shape identical to `ReversalService`'s own catch-all (approved in PR #4), not introduced or worsened by this PR.
2. **[Nit]** A transfer's `AuthorizationDecision` still has no way to ever be marked resolved after an unconfirmed failure (no endpoint exists). Correctly disclosed and scoped out by the PR, not a defect in it — a residual platform-wide gap.

No Blockers, no Should-fix items.

## What QA could not check

- Behavior against real, deployed `IdentityApi`/`LedgerApi` instances (tests run against hand-rolled fakes, same pre-existing limitation as every prior PR on this service).
- Whether the deployed `IdentityApi` exactly matches the source read on disk.

## Interview Q&A

1. **Why does the new audit row use the decision id instead of a LedgerApi reference?** Because this catch only runs when the LedgerApi call throws — no reference is ever returned in that case. The decision id is the only identifier available, and it's the same one `TransferAuthorized`/`TransferDenied` rows already use.
2. **Why didn't this PR add an outcome-reporting call to IdentityApi, the way ReversalService has?** It would require inventing a brand-new cross-service endpoint that doesn't exist for transfers — out of scope for a task meant to close an audit-trail gap, not design a new contract.
3. **Does the new catch interfere with the existing AuthorizationDeniedException handling?** No — two separate, non-overlapping try/catch blocks, neither nested in the other.
4. **Why doesn't the 422-with-body case hit the new catch?** LedgerApi returns that as a normal response value (per its own documented contract), not a thrown exception — it never reaches a catch block.
5. **How did the test prove the audit row was actually written?** It queries the audit table directly and asserts the row's real content, not a call count on the fake client.
