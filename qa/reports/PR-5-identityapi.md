# QA Report — Garbati.IdentityApi PR #5 (T-031)

**Verdict: Approve with nits** (no blockers)

## 1. Scope fidelity — PASS

Diff touches exactly 10 files: `docs/DESIGN.md`, new `PagedResponse.cs`, new `InvalidPageTokenException.cs`, `AuthorizationController.cs` (new action only), `ExceptionHandlingMiddleware.cs` (one new mapping line), `JointApprovalService.cs`/`IJointApprovalService.cs` (new method only), plus three test files — 496 insertions, 0 deletions. `RequestAuthorization`, `Approve`, `RecordReversalOutcome`, `GetAuthorizationRequest` are byte-for-byte untouched. No new scope, no migration.

## 2. "Single-holder requests never appear" — verified, not just trusted

Directly read `ListPendingRequestsAsync_NeverIncludesASingleHolderRequest_WhichNeverCreatesARow`: creates a real single-holder account (grants immediately via `AuthorizationDecision`, confirmed `DebitRequestResult.Granted`), creates a genuine joint-approval request on a different account, asserts the joint request is present and the solo account's id is absent, plus an independent direct DB `COUNT` confirming zero `PendingApprovalRequest` rows for the solo account. Matches production logic: `RequestDebitAsync` only creates this row when `RequiredApprovals > 1`.

## 3. #TBD-34 (no lazy expiry on list reads) — real, disclosed, not money-unsafe; fixed

Confirmed: `GetPendingRequestAsync`/`ApproveAsync` both call `ExpireIfPastDueAsync` before answering; the original list implementation did not. **Not a money-safety bug** — `ApproveAsync` always re-checks expiry before issuing a grant, so no stale approval could ever wrongly succeed. It was a real UX/data-freshness gap on the exact screen this endpoint exists for: an ops queue could show a phantom "Pending" entry that would fail the moment someone acted on it.

**Fixed post-review** (coordinator, same PR): added a read-only `ExpiresAt > now` condition to the query whenever `status = Pending`, with a new regression test (`ListPendingRequestsAsync_ExcludesAPendingRowPastItsExpiry_WithoutTouchingItsStatus`) that proves the exclusion at the query level, independent of the lazy-expiry side effect the existing `StatusFilter` test relies on. This stays within the task's "no extra business logic beyond a WHERE filter" constraint — it's a read, not a write; the database's `Status` column can still legitimately say `Pending` for a stale row, the list just no longer surfaces it as actionable.

## 4. `PagedResponse<T>` and paging correctness — PASS, genuinely tested across pages

`ListPendingRequestsAsync_PagingActuallyPages_NoDuplicatesAndNoGaps` creates 5 real rows, walks with `pageSize: 2` across exactly 3 pages via `NextPageToken`, confirms all 5 ids returned exactly once in creation order. Keyset paging (`Id > cursor`, ascending) — no OFFSET drift, stable ordering, "ask for one extra row" determines `hasMore` without a second round trip. Malformed token → `InvalidPageTokenException` → 400, tested at both service and HTTP level.

## 5. Build and full test suite — ran independently, confirmed

`dotnet build -c Release`: 0 warnings, 0 errors. `dotnet test -c Release`: **287/287 passed** (286 at QA's review + 1 new regression test added in the follow-up fix; up from 266 on `main`).

## Findings

1. **[Nit]** No DB index on `Status`/`(Status, Id)` backs the new query. Fine at current scale (no migration was in scope for this task); worth a follow-up once `Approved`/`Expired`/`Cancelled` rows accumulate for real.
2. **[Nit / owner note, not a defect in this PR]** The scope this endpoint reuses (`identity.authorization.check`, per the task's explicit "no new scope" instruction) also grants the power to directly request/approve authorizations on anyone's behalf — more than a read-only queue screen needs. When `OpsApi` (T-028) is actually built and granted this scope to browse the queue, it will also be able to do much more than browse. Already anticipated in `Garbati.OpsApi/docs/DESIGN.md` (it needs `identity.authorization.check` anyway for its joint-account test-console pass-through) — not a new issue, just worth the owner keeping in mind.
3. No sensitive data (grant tokens) is returned by this endpoint — `ToResponse(r)` called with `grant: null` in the list path, consistent with decision #13.
4. Route collision check (`[HttpGet]` vs `[HttpGet("{id}")]`) — confirmed no ambiguity.

No Blockers, no Should-fix items remaining after the staleness-gap fix.

## What QA could not check

- No real `OpsApi` caller exists yet (T-028 design only, not built) to exercise this end-to-end.
- Did not load-test the unindexed query at scale.

## Interview Q&A

1. **If an ops agent's screen shows a request as "Pending" and they click approve, but it expired ten minutes ago, what happens?** The approve attempt re-checks expiry for real and rejects it — no money moves, no grant issued off a stale row. (And after the follow-up fix, the stale row wouldn't even show as Pending in the list anymore.)
2. **Why didn't the list endpoint originally filter out expired-but-untouched rows?** The task explicitly ruled out logic beyond a plain status filter; correcting every stale row on every page read would turn a read into a variable, data-mutating operation. Fixed afterward with a read-only query condition instead — no write, no violation of that constraint.
3. **Will a single-holder account's transfer ever show up in this list?** No — it resolves instantly via `AuthorizationDecision` and never creates the row type this endpoint reads. Specifically tested, not assumed.
4. **What scope does a caller need, and who has it today?** The existing `identity.authorization.check` scope — no new scope was created. Currently only `dev-orchestrator` holds it in dev config; `OpsApi` will need it granted when built, which also hands it more power than just browsing — worth a second look at that point.
5. **How does paging guarantee no duplicates or gaps?** Each page is ordered by an ever-increasing internal id, and the next page always asks for ids strictly greater than the last one seen — no offset to drift if rows are added mid-walk. Proven by actually creating five rows and walking through with a small page size.
