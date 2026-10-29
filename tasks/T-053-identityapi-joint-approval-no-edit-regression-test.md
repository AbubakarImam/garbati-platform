# T-053: IdentityApi — regression test pinning "no edit path" for pending joint-approval requests

Target repo: `Garbati.IdentityApi`. Branch: `feature/joint-approval-no-edit-regression-test`.

## Context

External feedback (D-29, `garbati-platform`): "for the maker-checker flow, test a transaction edited while the checker has it open. the approval should be tied to the exact version they reviewed." A real, well-known class of bug: a checker reviews a pending request, something mutates it before they approve, and the checker unknowingly approves different content than what they actually reviewed.

The coordinator investigated directly against the real code (not assumed) and confirmed: **no such race exists today for `PendingApprovalRequest`** (the joint-account M-of-N approval entity, T-018/`JointApprovalService`), because no code path mutates its substantive fields (`Operation`, `DebitAccountId`, `CreditAccountId`, `Amount`, `Currency`, `RequestedRequiredApprovals`) after creation. `ApproveAsync` only ever appends a new `ApprovalRecord` and, at threshold, writes `Status`/`DecisionId` on the request itself (or `RequiredApprovals`/`LastChangedAt`/`LastChangedBy` on the separate `MandateApprovalPolicy` row for a threshold-change request) — never the canonical fields. `ExpireIfPastDueAsync` only writes `Status`. Confirmed via `AuthorizationController.cs`/`AccountsController.cs` (no `[HttpPut]`/`[HttpPatch]` action anywhere) and `IJointApprovalService.cs`.

Notably, `PendingApprovalRequest.cs`'s own existing doc comment already states this is deliberate: these fields "mirror the fields that will eventually appear in the signed grant, fixed at creation time so a later approval cannot be tricked into approving a changed transaction" — a documented design decision from when this flow was originally built, not an accident this task is discovering for the first time. This task pins that existing invariant in a test, so it can't silently erode later.

One related, separate gap the investigation also found: `PendingApprovalRequestStatus` defines a `Cancelled` value in its enum, but nothing anywhere ever writes it — there is no "withdraw my own pending request" endpoint. Not a security issue, not in scope here — a real but minor product gap for the owner to decide on separately, if at all.

## What to build

A regression test (or two, your call) asserting the *closed set* of what exists today:

### 1. Controller-level: no PUT/PATCH action on the relevant controllers

Using reflection, assert that **no public action method on `AuthorizationController` or `AccountsController` carries an `HttpPutAttribute` or `HttpPatchAttribute`** anywhere. `typeof(AuthorizationController).GetMethods(BindingFlags.Public | BindingFlags.Instance | BindingFlags.DeclaredOnly)` (and the same for `AccountsController`), check each for either attribute, fail with a clear message naming the offending method if found. (Confirm directly whether either controller has other, unrelated actions beyond the joint-approval ones before writing this — the assertion should cover the whole controller class, which is fine since the investigation already confirmed neither has any PUT/PATCH action at all today.)

### 2. Interface-level: the exact, closed method set on `IJointApprovalService`

Assert via reflection that the full set of public method names on `IJointApprovalService` equals exactly the current real list — confirm the exact names directly against `Services/Interfaces/IJointApprovalService.cs` before writing the assertion (the investigation found: `RequestDebitAsync`, `ApproveAsync`, `GetPendingRequestAsync`, `GetApprovalPolicyAsync`, `RequestThresholdChangeAsync`, `ListPendingRequestsAsync` — verify this is still accurate, don't copy blindly). This is a closed-set assertion specifically so that *any* new method added later forces a deliberate look at whether it reopens the TOCTOU risk, not just one literally named "Edit" or "Update".

### 3. One clear top-of-file comment

State explicitly what invariant is being pinned and why, referencing D-29 and the reviewer's original comment, and note that `PendingApprovalRequest.cs`'s own existing doc comment already documents this as a deliberate design choice this test is now pinning in code. Example shape (adapt, don't copy verbatim):

```csharp
// D-29 (garbati-platform): external feedback flagged a TOCTOU risk - a checker could approve
// a joint-approval request that was silently edited after they reviewed it. Investigated
// directly: PendingApprovalRequest has no edit path today - its own entity comment already
// states its canonical fields are "fixed at creation time so a later approval cannot be
// tricked into approving a changed transaction" (T-018). This test pins that invariant in
// code: if AuthorizationController/AccountsController ever gains a PUT/PATCH action, or
// IJointApprovalService ever grows a new method, this test fails and forces a deliberate
// decision about whether that addition reopens the race.
```

## Explicitly out of scope

- No production code changes. This is a test only.
- Do not add a "withdraw my own pending request" endpoint for the unused `Cancelled` status — a separate, minor product gap for the owner, not this task.
- Do not touch `Garbati.OpsApi` — its equivalent regression tests are a separate task (T-052) in that repo.

## Tests to write

Exactly the two reflection-based checks described above (controller attribute check ×2 controllers, interface method-set check ×1 interface). Run the full existing suite too and confirm nothing else broke (expected: nothing should, since no production code changes).

## PR

Open a pull request from `feature/joint-approval-no-edit-regression-test` against `main` on `AbubakarImam/Garbati.IdentityApi` (get `test`/`docker` CI green first). Reference D-29 explicitly in the PR description. Do not merge — only the owner merges.
