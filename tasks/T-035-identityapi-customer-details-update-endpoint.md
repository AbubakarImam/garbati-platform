# T-035 — IdentityApi: add `PATCH /v1/customers/{customerId}/details`

## Context

`garbati-platform` D-19, owner-approved 20 Oct 2026. Found while scoping `Garbati.OpsApi`'s second build task (T-034): `docs/DESIGN.md` Section 4.1 describes a customer-detail-edit maker-checker workflow (a `CustomerServiceOfficer` stages a change, a `Supervisor` approves it, `OpsApi` then applies it via `IdentityApi`'s `identity.customer.write` scope) — but `IdentityApi`'s `CustomersController` has no endpoint to apply such a change at all. It has `POST /v1/customers` (create), `PATCH .../kyc`, `POST .../suspend`/`.../reactivate`/`.../close`/`.../mark-deceased`, and the two mandate actions — nothing that updates a customer's name, email, or phone number after creation. Confirmed directly against the controller's source before writing this task.

This task closes that gap. It is purely additive — no existing endpoint, scope, or behavior changes.

## What to build

**New endpoint**, `PATCH /v1/customers/{customerId}/details`:

- Scope: `identity.customer.write` — the existing scope (`[Authorize(Policy = IdentityScopes.CustomerWrite)]`, same policy `POST /v1/customers` already uses). **No new scope.**
- Request contract (new, `Contracts/Requests/UpdateCustomerDetailsRequest.cs`), a **partial** update — every field optional, only supplied fields change:
  ```csharp
  public record UpdateCustomerDetailsRequest(string? FullName, string? Email, string? PhoneNumber);
  ```
  These three fields live on `CustomerIdentityData` (`Entities/CustomerIdentityData.cs`), not `Customer` itself — update that entity, not `Customer`. `CustomerReference` (on `Customer`) is **not** part of this endpoint; nothing in the OpsApi workflow this task supports needs it, and extending scope beyond the three fields `CustomerIdentityData` already defines is not this task's call to make.
- Validation: reject a request where every field is `null` (nothing to update) with `400`. Reuse this platform's existing validators' style for field-level checks (e.g. a non-empty `FullName` if supplied, a plausible email shape if supplied) — mirror whatever validation rigor `CreateCustomerRequestValidator` already applies to these same three fields for creation, don't invent a stricter or looser standard for the same data.
- Returns the updated `CustomerResponse` (existing shape — this endpoint doesn't need a new response type).
- `404` if the customer doesn't exist (same convention as `GET`/`PATCH .../kyc`).
- No idempotency key required — this is a plain field-level update (setting the same value twice has no different effect than setting it once), the same reasoning that already applies to `PATCH .../kyc` and the status-change actions, none of which take one either.
- Audit: a new `AuditActions` entry (check the exact existing constant-naming convention in this codebase, e.g. `CustomerDetailsUpdated`) recording which fields changed (old → new values for whichever fields were actually supplied) — staged on the same `DbContext`/committed by the same `SaveChangesAsync` as the update itself, same convention as every other mutating action in this service. Do **not** log or audit a raw BVN/NIN-adjacent value beyond what's already stored here (this endpoint only ever touches `FullName`/`Email`/`PhoneNumber` — no BVN/NIN column exists on `CustomerIdentityData` at all, confirmed by reading the entity directly, so there is nothing of that sensitivity to protect against here beyond this service's already-existing logging discipline).

## Scope

- `Contracts/Requests/UpdateCustomerDetailsRequest.cs`, a validator (`UpdateCustomerDetailsRequestValidator.cs`, mirroring the existing validator pattern), one new controller action on `CustomersController`, one new method on `ICustomerService`/`CustomerService` (e.g. `UpdateDetailsAsync`), one new `AuditActions` constant.
- Tests: updating one field only (others unchanged), updating all three, updating none (400), a non-existent customer (404), the audit row's content, and scope enforcement (missing/wrong scope → 401/403, matching the existing endpoints' coverage style).
- Do not touch `CreateCustomer`, `RecordKycDecision`, `Suspend`/`Reactivate`/`Close`/`MarkDeceased`, or the mandate endpoints — purely additive.

## Out of scope

- Any change to `CustomerReference`, `JurisdictionCode`, `CustomerStatus`, or `KycStatus`/`KycTier` — none of those belong to this endpoint.
- Any UI/workflow logic (`OpsApi`'s staging/approval maker-checker, `OpsCustomerEditRequest`) — that's `garbati-platform` T-034's job, built against this endpoint once it exists.
- Encryption-at-rest or a BVN/NIN column — out of scope here as it was for every prior task touching `CustomerIdentityData` (T-016's own flagged follow-up, still not this task's job).
