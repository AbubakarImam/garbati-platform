# T-047: OpsApi — actually apply an approved customer-edit request

Target repo: `Garbati.OpsApi`. Branch: `feature/apply-customer-edit`.

## Context

`OpsCustomerEditRequest`'s maker-checker workflow (T-034) was deliberately built to stop at `Approved` —
at the time, `Garbati.IdentityApi`'s `PATCH /v1/customers/{customerId}/details` endpoint (the thing an
approval would need to call) didn't exist yet. It exists now (T-035, merged and deployed). This task
closes that gap the same way `ReversalRequestService` already closed the identical gap for
`OpsReversalRequest` (T-034) — read that class directly before writing anything; this task mirrors its
shape, not a new pattern.

Confirmed directly, so you don't have to re-check: `identity.customer.write` is already granted to
`Garbati.OpsApi`'s own IdentityApi client (`dev-ops`/`compose-opsapi` in
`Garbati.IdentityApi/src/Garbati.IdentityApi/appsettings.Development.json` both already list it) — this
task needs no new scope grant. `OpsCustomerEditRequestStatus.Applied`/`ApplicationFailed` already exist
on the enum (added in T-034, deliberately unused until now) — you are not adding new statuses, only
finally reaching them.

## What to build

### 1. `IIdentityApiClient.UpdateCustomerDetailsAsync`

Add `Task<CustomerResponse> UpdateCustomerDetailsAsync(string customerId, UpdateCustomerDetailsRequest
request, CancellationToken cancellationToken)` to `Clients/Interfaces/IIdentityApiClient.cs` and its
implementation in `Clients/Implementations/IdentityApiClient.cs` — calling `PATCH
v1/customers/{customerId}/details` on `Garbati.IdentityApi`, mirroring `GetCustomerAsync`'s existing
`ReadOrThrowAsync` pattern in that same file exactly.

Add `Clients/Contracts/Identity/UpdateCustomerDetailsRequest.cs`:

```csharp
public record UpdateCustomerDetailsRequest(string? FullName, string? Email, string? PhoneNumber);
```

matching `Garbati.IdentityApi`'s own `UpdateCustomerDetailsRequest` field-for-field exactly (read that
file directly — `Garbati.IdentityApi/src/Garbati.IdentityApi/Contracts/Requests/
UpdateCustomerDetailsRequest.cs`) — same reasoning `OpsCustomerEditRequest`'s own fields already used
this exact shape for (its own entity comment says so).

### 2. Wire `CustomerEditRequestService.ApproveAsync`

Change the constructor to take `IIdentityApiClient identityApiClient` and
`ILogger<CustomerEditRequestService> logger`, same as `ReversalRequestService`'s constructor shape.

After `EnsureMakerChecker` passes (same guard, unchanged), **do not stop at `Approved`** — call
`identityApiClient.UpdateCustomerDetailsAsync(request.TargetCustomerId, new
UpdateCustomerDetailsRequest(request.ProposedFullName, request.ProposedEmail,
request.ProposedPhoneNumber), cancellationToken)`, wrapped in the exact same `try`/`catch (Exception ex)`
shape `ReversalRequestService.ApproveAsync` already uses for its own downstream call — any exception
(not just a specific type) is a failure to apply, recorded, never left to bubble as an opaque 500.

- On success: `Status = OpsCustomerEditRequestStatus.Applied`.
- On any failure: `Status = OpsCustomerEditRequestStatus.ApplicationFailed`, with the error recorded on a
  new field (see below) — same "record what actually happened, don't swallow it" principle.
- `ApprovedByStaffId`/`DecidedAt` are set exactly as they already are today, regardless of which of the
  two outcomes above happens — approval itself still happened; what's new is whether applying it also
  succeeded.
- Wrap the final `SaveChangesAsync` in the same `try`/`catch`-log-don't-rethrow pattern
  `ReversalRequestService.ApproveAsync` uses (`logger.LogError` on failure, in-memory `request` still
  reflects the true outcome to the caller even if persisting it failed) — mirror it exactly, including
  the log message shape.

**No idempotency key needed for the `PATCH` call itself**, unlike the reversal flow's deterministic
`ops:{request.Id}` key for `OrchestratorApi.ReverseAsync`. Explain this reasoning in the PR rather than
inventing an idempotency mechanism: a reversal moves money and must never execute twice; a detail-update
`PATCH` setting the same field values twice has the same end state either time — it's a plain setter, not
an operation that compounds. `EnsureDecidable` already rejects a second `ApproveAsync` call once the
request has left `PendingApproval` (the same guard that already protects `ReversalRequestService` from a
double-approve), so this is not a gap this task needs to additionally close.

### 3. New field: `ApplicationError`

Add `string? ApplicationError` to `Entities/OpsCustomerEditRequest.cs` (nullable, set only on
`ApplicationFailed`, mirroring `OpsReversalRequest.ExecutionDetail`'s role exactly — there is no
equivalent to `ExecutedReference` needed here, since a detail-update has no reference/transaction id to
record on success). One additive EF Core migration; every existing row gets `NULL`.

Add `ApplicationError` to `Contracts/Responses/CustomerEditRequestResponse.cs` so a caller can actually
see why an application failed, the same way `OpsReversalRequest`'s own response already exposes
`ExecutionDetail`.

## Explicitly out of scope

- Any change to `Garbati.IdentityApi` itself.
- Any change to the create/list/get/reject paths — only `ApproveAsync` changes.
- A retry-the-apply-only endpoint for a request stuck in `ApplicationFailed` — not asked for, not built;
  if this is ever needed, that's a separate, future task.
- Any change to the controller's route/scope/role requirements — unchanged.

## Tests to write

- `ApproveAsync` happy path: a fake `IIdentityApiClient` records the exact `customerId`/
  `FullName`/`Email`/`PhoneNumber` it was called with; `Status` ends `Applied`; `ApplicationError` stays
  `null`.
- `ApproveAsync` when the fake client throws: `Status` ends `ApplicationFailed`; `ApplicationError`
  contains a clear, specific message (not a generic "an error occurred"); `ApprovedByStaffId`/`DecidedAt`
  are still set (approval itself still happened).
- Maker-checker and expiry guards still work exactly as before (mirror the existing tests for these,
  confirming they weren't accidentally weakened by this change).
- A second `ApproveAsync` call against an already-`Applied` (or `ApplicationFailed`) request still throws
  `OpsRequestNotPendingException` — confirm the existing `EnsureDecidable` guard covers both new terminal
  states without any code change needed there (if it doesn't, that's a real finding to report, not to
  silently patch around).
- Migration applies cleanly; existing rows get `NULL` for `ApplicationError`.

## PR

Open a pull request from `feature/apply-customer-edit` against `main` on `AbubakarImam/Garbati.OpsApi`
(branch-protected — get `test`/`docker` CI green first). Be explicit in the description about the
no-idempotency-key reasoning for the `PATCH` call, since it's a deliberate asymmetry from the reversal
flow's pattern, not an oversight. Do not merge — only the owner merges.
