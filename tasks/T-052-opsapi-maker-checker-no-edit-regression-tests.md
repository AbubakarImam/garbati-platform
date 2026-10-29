# T-052: OpsApi — regression tests pinning "no edit path" for pending maker-checker requests

Target repo: `Garbati.OpsApi`. Branch: `feature/maker-checker-no-edit-regression-tests`.

## Context

External feedback (D-29, `garbati-platform`): "for the maker-checker flow, test a transaction edited while the checker has it open. the approval should be tied to the exact version they reviewed." This names a real, well-known class of bug: a time-of-check-to-time-of-use race, where a checker reviews a pending request, something mutates it before they approve, and the checker unknowingly approves different content than what they actually reviewed.

The coordinator investigated directly against the real code (not assumed) and confirmed: **no such race exists today, for `OpsReversalRequest` or `OpsCustomerEditRequest`, because neither has any edit/update endpoint or service method that can mutate the entity's own substantive fields after creation.** The only state transitions available are `ApproveAsync` (by someone other than the maker, enforced by `EnsureMakerChecker`), `RejectAsync`, or lazy expiry — confirmed by reading `ReversalRequestsController.cs`/`CustomerEditRequestsController.cs` (no `[HttpPut]`/`[HttpPatch]` action anywhere), `IReversalRequestService.cs`/`ICustomerEditRequestService.cs` (exactly 5 methods each: `CreateAsync`, `ListAsync`, `GetAsync`, `ApproveAsync`, `RejectAsync` — no `UpdateAsync`), and both service implementations (only `Status`/`ApprovedByStaffId`/`RejectionReason`/`DecidedAt`/execution-outcome fields are ever written outside `CreateAsync` — never `OriginalTransactionReference`/`Amount`/`Currency`/`ReasonCode`/`ProposedFullName`/`ProposedEmail`/`ProposedPhoneNumber`).

This task does **not** change any production behavior. It writes regression tests that pin this invariant in code, so if anyone ever adds an edit endpoint later, a test fails and forces a deliberate decision about the TOCTOU risk at that moment, rather than the safety property silently eroding.

## What to build

Two new test classes (or one, your call, grouped sensibly), asserting the *closed set* of what exists today — not just asserting the absence of something literally named "Edit":

### 1. Controller-level: no PUT/PATCH action on either controller

Using reflection over `ReversalRequestsController` and `CustomerEditRequestsController` (`Controllers/`), assert that **no public action method on either class carries an `HttpPutAttribute` or `HttpPatchAttribute`**. Iterate `typeof(ReversalRequestsController).GetMethods(BindingFlags.Public | BindingFlags.Instance | BindingFlags.DeclaredOnly)` (and the same for `CustomerEditRequestsController`), check `GetCustomAttributes(true)` for either attribute type, fail with a clear message naming the offending method if found.

### 2. Interface-level: the exact, closed method set

For `IReversalRequestService` and `ICustomerEditRequestService`, assert via reflection that the full set of method names equals exactly:
- `IReversalRequestService`: `{"CreateAsync", "ListAsync", "GetAsync", "ApproveAsync", "RejectAsync"}`
- `ICustomerEditRequestService`: `{"CreateAsync", "ListAsync", "GetAsync", "ApproveAsync", "RejectAsync"}`

(Confirm the exact real method names directly against the current interface files before writing the assertion — don't copy this list blindly if the interface has since changed.) This is deliberately a closed-set assertion, not a "doesn't contain Update" substring check — it catches *any* new method added to either interface, forcing whoever adds one to update this test and think about whether it reopens the TOCTOU risk.

### 3. One clear top-of-file comment on both new test classes

State explicitly what invariant is being pinned and why — reference D-29 and the reviewer's original comment — so a future reader understands this isn't an arbitrary reflection test but a deliberate safety-property regression test. Example shape (adapt, don't copy verbatim):

```csharp
// D-29 (garbati-platform): external feedback flagged a TOCTOU risk - a checker could approve
// a pending request that was silently edited after they reviewed it. Investigated directly:
// neither OpsReversalRequest nor OpsCustomerEditRequest has any edit path today - the only
// writes after CreateAsync are the terminal Approve/Reject transitions. These tests pin that
// invariant: if either controller ever gains a PUT/PATCH action, or either service interface
// ever grows a new method, this test fails and forces a deliberate decision about whether that
// addition reopens the race, rather than letting the safety property erode silently.
```

## Explicitly out of scope

- No production code changes. This is tests only.
- Do not add a "cancel my own pending request" endpoint — a related but separate gap the investigation also found (no `Cancelled` status is ever reachable for either entity today). That's a real, minor product gap, not a security issue, and is a separate decision for the owner if they want it built — not this task.
- Do not touch `Garbati.IdentityApi` — the joint-approval flow's equivalent regression test is a separate task (T-053) in that repo.

## Tests to write

Exactly the two reflection-based tests described above (controller attribute check ×2 controllers, interface method-set check ×2 interfaces) — 4 assertions total, however you choose to organize them into test methods/classes. Run the full existing suite too and confirm nothing else broke (expected: nothing should, since no production code changes).

## PR

Open a pull request from `feature/maker-checker-no-edit-regression-tests` against `main` on `AbubakarImam/Garbati.OpsApi` (get `test`/`docker` CI green first). Reference D-29 explicitly in the PR description. Do not merge — only the owner merges.
