# QA Report — Garbati.OpsApi PR #3 (T-047: apply approved customer-edit request)

**Verdict: Approve**

## 1. Task fidelity — PASS

`git diff main..feature/apply-customer-edit --stat`: 16 files, +573/-48. Every change traces to the task
file:
- `Clients/Contracts/Identity/UpdateCustomerDetailsRequest.cs` (new record) + `IIdentityApiClient`/
  `IdentityApiClient` additions (`UpdateCustomerDetailsAsync`).
- `CustomerEditRequestService.ApproveAsync` rewired; constructor now takes `IIdentityApiClient`,
  `ILogger<CustomerEditRequestService>`.
- `OpsCustomerEditRequest.ApplicationError` (nullable) + one additive migration
  (`AddCustomerEditApplicationError`).
- `CustomerEditRequestResponse.ApplicationError` + controller mapping.
- `docs/DESIGN.md` updated (4.1, Section 6 item 2, new Appendix C, superseded-note on Appendix B item 8).
- Tests: new client test, new service/controller tests, two new migration tests.

Nothing extra. `Create`/`List`/`Get`/`Reject` paths, `Contracts/Requests/`, `Services/Interfaces/`,
`Program.cs` are all untouched (confirmed via targeted `git diff`, no output). No `Garbati.IdentityApi`
file was touched — confirmed directly: `git status` in that repo's checkout is clean, no local
modifications exist there.

## 2. ARCHITECTURE.md / DESIGN.md — PASS, resolves a documented gap, builds on nothing "Proposed"

`docs/DESIGN.md` Section 4.1 explicitly named this as the one remaining gap since T-034, contingent on
T-035 (`IdentityApi PATCH /v1/customers/{customerId}/details`) being merged and deployed — which the task
file states is confirmed, and which I independently verified by reading
`Garbati.IdentityApi/src/Garbati.IdentityApi/Controllers/CustomersController.cs` directly (route
`[HttpPatch("{customerId}/details")]`, `[Authorize(Policy = IdentityScopes.CustomerWrite)]` — live on
`main`, not a "Proposed" branch). The scope grant (`identity.customer.write` on `dev-ops`/
`compose-opsapi`) and the pre-existing `Applied`/`ApplicationFailed` enum values were both confirmed by
reading the files directly rather than trusting the task file's claim — both checked out.

## 3. `ApproveAsync` mirrors `ReversalRequestService.ApproveAsync` — PASS, read side by side

Compared both methods directly (not from the PR description):
- `Status`/`ApprovedByStaffId`/`DecidedAt` set unconditionally before the downstream call, in both.
  identical placement.
- Downstream call wrapped in `try { ... } catch (Exception ex)` — any exception, not a specific type — in
  both. Error message shape is identical: `ex is DownstreamApiException downstream ? "{Service} returned
  {code}: {msg}" : "{type}: {msg}"`.
- Final `SaveChangesAsync` wrapped in its own `try`/`catch (Exception saveEx)` →
  `logger.LogError(saveEx, "{Workflow} request {Id} reached {Status} but the result could not be saved.",
  request.Id, request.Status)`, not rethrown, in both. The comment above it explicitly states the
  in-memory `request` returned to the caller still reflects the true outcome even if the save failed — same
  wording, same reasoning, in both files.
- One structural difference, and it is the *correct* one: `ReversalRequestService` has a third outcome
  branch (`result is not null && result.Status != "Failed"`) because `OrchestratorApi.ReverseAsync` can
  return HTTP 200 with a business-rule failure body. `IdentityApi`'s PATCH has no such dual-outcome shape —
  `ReadOrThrowAsync` throws on any non-success status or an empty body, so a non-null `result` can only mean
  genuine success. `CustomerEditRequestService` correctly uses the simpler `result is not null` check. This
  is a deliberate, justified simplification, not a missed case.

## 4. Contract fidelity against `Garbati.IdentityApi` — PASS, read both sources directly

`Garbati.OpsApi`'s new `UpdateCustomerDetailsRequest(string? FullName, string? Email, string? PhoneNumber)`
and `Garbati.IdentityApi`'s own `Contracts/Requests/UpdateCustomerDetailsRequest.cs` are byte-for-byte
identical in field name, order, and nullability. Route and scope also verified directly against
`CustomersController.cs`: `PATCH v1/customers/{customerId}/details`, `IdentityScopes.CustomerWrite` —
matches what the OpsApi client calls and the scope the task file says is already granted.

## 5. `EnsureDecidable` / no-idempotency-key reasoning — PASS, genuinely verified, not just claimed

Read `EnsureDecidable` directly: `if (request.Status != OpsCustomerEditRequestStatus.PendingApproval)` —
unchanged since T-034, no code touched it in this PR (confirmed by diff). This rejects `Applied` and
`ApplicationFailed` the same way it already rejected `Approved`/`Rejected`, with no special-casing needed.

The two new regression tests are genuine, not decorative:
- `Approve_AlreadyAppliedRequest_Returns409` — creates a request, approves it for real (fake client
  succeeds, reaching `Applied`), then has a **second, different** Supervisor approve again → asserts
  `409`.
- `Approve_AlreadyApplicationFailedRequest_Returns409` — same shape, but the fake client is configured to
  throw on the first approve (reaching `ApplicationFailed`), then a second approve attempt → asserts `409`.

Both go through the real HTTP pipeline (`WebApplicationFactory`), not a unit-level call directly against
the service with a hand-constructed entity — each genuinely exercises its own distinct terminal state.
This is real empirical confirmation, not an inspection-only claim.

The no-idempotency-key reasoning itself holds: a `PATCH` setting the same field values twice has the same
end state either time (confirmed this is also `IdentityApi`'s own stated reasoning for not giving its
`UpdateCustomerDetailsRequest` an idempotency key), and `EnsureDecidable` already closes the only gap that
would matter (a second call reaching the downstream client at all). No idempotency mechanism was invented.

## 6. Failure path — PASS

`Approve_WhenIdentityApiCallThrows_StoresApplicationFailed_NotA500`: fake client throws
`DownstreamApiException("IdentityApi", 500, "boom")`. Asserts: HTTP response is `200 OK` (not a bubbled
500), `Status == "ApplicationFailed"`, `ApplicationError` contains both `"500"` and `"IdentityApi"` (a
specific message, not a generic one), and `ApprovedByStaffId`/`DecidedAt` are both still set — approval
itself recorded as having happened even though the apply failed. All confirmed by reading the test and the
service code together, not by trusting the PR description.

## 7. Migration — PASS, purely additive

`AddCustomerEditApplicationError`: `AddColumn<string> "application_error"` on
`ops_customer_edit_requests`, `type: "text", nullable: true`. `Down` drops it cleanly. New migration test
(`OpsCustomerEditRequestsTable_ApplicationErrorColumn_IsNullableAndDefaultsToNullOnExistingRows`) inserts a
row without setting `ApplicationError` and reads it back as `null` through a fresh context — a genuine
round-trip against the real Postgres container (Testcontainers), not an EF in-memory model assumption.

## 8. Scope fidelity — PASS

No change to `Garbati.IdentityApi` (confirmed: clean `git status` in that checkout). No change to
create/list/get/reject paths or to the controller's route/scope/role attributes (confirmed via targeted
diffs showing zero changes outside `ApproveAsync`, the response/entity/migration additions, and the
doc/test files). No new idempotency mechanism invented despite the asymmetry — explicitly reasoned about
and left out, exactly as the task file asked.

## 9. Security / money safety

No secrets, no new sensitive logging (the `logger.LogError` call logs the request id and status, not any
customer PII). `ApplicationError` is sourced from `DownstreamApiException.Message`/generic exception
message — worth a passing note that if `IdentityApi` ever put a customer's raw field value into an error
message, it would land in `ApplicationError` and be exposed in the `CustomerEditRequestResponse` to anyone
with `Supervisor`/`CustomerServiceOfficer` access to this workflow (both already-privileged roles that can
see the proposed `FullName`/`Email`/`PhoneNumber` anyway, so this is not a new disclosure surface — noted,
not a finding). No money is moved by this change at all; the "money safety" lens mostly doesn't apply here
beyond confirming the maker-checker and idempotency reasoning above, both of which check out.

## 10. Readability

Both new code and the three-paragraph DESIGN.md Appendix C addition are easy to follow without the PR
description — comments point at the exact sibling method being mirrored and explain *why* each divergence
exists (the `result is not null` simplification, no `ExecutedReference` equivalent). No cleverness beyond
what `ReversalRequestService` already established.

## 11. Performance

No loops, no pagination-sensitive query changes. `ListAsync`/`GetAsync` are unchanged. `ApproveAsync` makes
exactly one downstream HTTP call, as before — no N+1 introduced.

## 12. Tests — PASS, not weakened

The only test removed is `Approve_Succeeds_StopsAtApproved_AndNeverCallsIdentityApi`, replaced by
`Approve_Succeeds_CallsIdentityApi_AndReachesApplied` — correct, since the old test asserted behaviour
this PR deliberately changes (the whole point of the task). All of self-approval, non-Supervisor-approver,
role-downgraded-after-login, and expired-pending-row tests are untouched in substance and gained an
additional assertion each (`UpdateCustomerDetailsCallCount` unchanged) confirming the maker-checker/expiry
guards still block the downstream call before it's ever reached — a strengthening, not a weakening.

## Build & test run (this QA pass, not re-reporting CI secondhand)

```
dotnet test -c Release
Passed!  - Failed: 0, Passed: 125, Skipped: 0, Total: 125, Duration: 4 s
```

Ran the full suite myself, locally, via the real `WebApplicationFactory` + Testcontainers Postgres
pipeline — matches the PR's claimed 125/125 and completed in ~4 seconds (not slow; Docker/Testcontainers
was not a bottleneck here). GitHub Actions `test` and `docker` checks also show green on the PR
(independently confirmed via `gh pr view 3`), consistent with this local run. Did not rebuild the Docker
image myself, per the task instructions.

## Findings

No Blockers.

**Should fix:**
- None.

**Nits:**
1. `result is not null` in `CustomerEditRequestService.ApproveAsync` is logically unreachable as a false
   branch given `ReadOrThrowAsync`'s own guarantee (it throws on any non-success status *and* on an empty
   body) — a successful, non-thrown return can never be null. Harmless defensive style matching the
   existing codebase convention (same shape appears implicitly wherever `ReadOrThrowAsync`'s result is used
   elsewhere), not a bug. No action needed, just noting it's belt-and-suspenders rather than a real
   branch.
2. `docs/DESIGN.md` Appendix C's own file doesn't mention whether a `LEARNING_LOG.md` entry was actually
   committed anywhere findable inside this repo — the PR description contains a full learning-log entry as
   text, consistent with this platform's stated convention that `LEARNING_LOG.md` lives in
   `garbati-platform`, not per-repo. Did not independently verify an entry was actually added to
   `garbati-platform/LEARNING_LOG.md` as part of this task's "Done means" checklist — flagging as
   something the coordinator should confirm before closing T-047, not a defect in this PR itself (PR #3 is
   scoped to `Garbati.OpsApi` only).

## What I could not check

- Did not call a real, live-deployed `Garbati.IdentityApi` — all client-level tests run against a fake
  `HttpMessageHandler`/`FakeIdentityApiClient`, per this repo's established testing convention (documented
  limitation, consistent with every earlier appendix in `docs/DESIGN.md`). A real contract drift on
  `IdentityApi`'s deployed side would not be caught by this PR's own test suite.
- Did not independently confirm a `garbati-platform/LEARNING_LOG.md` entry exists for T-047 (see nit 2
  above) — out of this PR's own repo scope to check definitively.
- Did not rebuild the Docker image myself; relying on the already-green `docker` CI check per the task's
  own instruction.

## Interview Q&A

1. **Q: Why does the reversal flow need a deterministic idempotency key for its downstream call, but this
   one doesn't?**
   A: A reversal moves money — executing it twice would be a real, costly duplicate. The deterministic key
   (`ops:{request.Id}`) lets `OrchestratorApi` recognize a retried approve click as the same operation. A
   detail-update `PATCH` setting `FullName`/`Email`/`PhoneNumber` has the identical end state whether it
   runs once or twice — there's nothing for an idempotency key to protect against.

2. **Q: What actually stops a second `ApproveAsync` call from reaching `IdentityApi` a second time, given
   there's no idempotency key?**
   A: `EnsureDecidable`'s existing `Status != PendingApproval` check, unchanged since T-034. Once a request
   reaches `Applied` or `ApplicationFailed`, a second approve attempt is rejected with `409` before the
   downstream call is ever made — proven with two new tests that actually drive a request into each new
   terminal state and then retry.

3. **Q: If the `IdentityApi` call fails, does the Supervisor's approval get lost too?**
   A: No. `Status`/`ApprovedByStaffId`/`DecidedAt` are set unconditionally before the downstream call runs
   — the approval itself is a completed fact regardless of what happens next. Only the overall `Status`
   becomes `ApplicationFailed` instead of `Applied`, with the specific error recorded on the new
   `ApplicationError` field.

4. **Q: Why doesn't `ApplicationError` have an equivalent to `OpsReversalRequest.ExecutedReference`?**
   A: A reversal produces a transaction reference on success that's worth recording. A detail-update PATCH
   has no such artifact — on success there's nothing beyond "it worked," so `ApplicationError` stays
   `null` and nothing else is recorded.

5. **Q: How do you know the `UpdateCustomerDetailsRequest` record actually matches what `IdentityApi`
   expects, rather than just looking similar?**
   A: Both files were read directly, side by side —
   `Garbati.OpsApi/Clients/Contracts/Identity/UpdateCustomerDetailsRequest.cs` and
   `Garbati.IdentityApi/Contracts/Requests/UpdateCustomerDetailsRequest.cs` — field name, order, and
   nullability are identical (`string? FullName, string? Email, string? PhoneNumber`), and the route/scope
   the client calls (`PATCH v1/customers/{customerId}/details`, `identity.customer.write`) was confirmed
   against `Garbati.IdentityApi`'s own live controller, not assumed from the task file's description.
