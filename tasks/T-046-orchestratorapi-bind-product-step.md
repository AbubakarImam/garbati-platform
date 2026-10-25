# T-046: OrchestratorApi — add the missing "bind product" step to account opening

Target repo: `Garbati.OrchestratorApi`. Branch: `feature/bind-product-step`.

## Context

`ARCHITECTURE.md`'s own account-opening sequence (Section 10) has always named 5 steps, not 4:
create customer → KYC verified → create ledger account → attach mandate → **bind product version** →
mark complete. `AccountOpeningService`'s real `AccountOpeningStep` enum only has 4 steps today
(`CreateCustomer`, `VerifyKyc`, `CreateLedgerAccount`, `AttachMandate`, `Complete`) — step 5 was never
built, because `Garbati.ProductApi` didn't exist until T-045 (24 Oct 2026, merged). It exists now, with a
real `POST /v1/account-bindings` endpoint built exactly for this. This task closes that gap.

**Scope, deliberately narrow (owner's decision, 25 Oct 2026):** this task wires account opening only.
It does **not** add a fee-quote lookup to `TransferService` — `ProductApi`'s `FeeRule` table is still
genuinely empty (no scholar sign-off yet, see `research/sharia-product-rules.md`), so a fee-quote call on
every transfer today would add real latency and a new failure mode for zero behavioral payoff. That
remains a deliberate "not yet" — revisit once a real fee rule actually exists, not as part of this task.

## What to build

### 1. New step in the sequence

Add `BindProduct` to `Entities/AccountOpeningStep.cs`, between `AttachMandate` and `Complete`:

```csharp
public enum AccountOpeningStep
{
    CreateCustomer,
    VerifyKyc,
    CreateLedgerAccount,
    AttachMandate,
    BindProduct,
    Complete,
}
```

Add a new `if (opening.CurrentStep == AccountOpeningStep.BindProduct)` block in
`AccountOpeningService.RunFromCurrentStepAsync`, directly after the existing `AttachMandate` block and
before the final `opening.Status = AccountOpeningStatus.Complete` line — same structure every existing
step already uses (call the client, advance `CurrentStep`, call `SaveStepProgressAsync`).

Add a `string? ProductCode` field to `Entities/AccountOpening.cs` (nullable, set once this step
succeeds) — same pattern `CustomerId`/`LedgerAccountId` already use for the steps before it. One
additive EF Core migration; every existing row gets `NULL`, no data migration needed, no human-only-gate
concern (purely additive schema change, same as every other "add a nullable column" migration already
done on this platform).

Expose it on `AccountOpeningResponse`/`ToResponse` too — add `ProductCode` as the last field, after
`LedgerAccountId`, so a caller can see what the account was actually bound to once the operation
completes (`null` until the step runs, same as `CustomerId`/`LedgerAccountId` are `null` before their own
steps run).

### 2. The ProductCode mapping — explicit, deliberately simple, same "demo stub" honesty as `DemoKycTier`

`StartAccountOpeningRequest.AccountType` is a free-text field (`"Savings"`/`"Current"` in every real
caller today, per `CreateCustomer`'s own test fixtures — not a fixed enum anywhere in this codebase).
`Garbati.ProductApi`'s seeded placeholder products are `NG-SAV-STD` and `NG-CUR-STD` (per its own
`SeedPlaceholderProductLimitPolicies` migration — read it directly, don't guess at the exact codes).

Derive the `ProductCode` to bind as `{JurisdictionCode}-{SAV|CUR}-STD`:
- `AccountType == "Savings"` → `SAV`
- `AccountType == "Current"` → `CUR`
- Anything else → **do not guess** — throw a clear, specific exception (e.g.
  `NoProductMappingException: $"No product mapping exists for AccountType '{opening.AccountType}'."`),
  caught by the method's own existing outer `catch (Exception ex)` block exactly like every other step
  failure, surfacing as `Status = Failed`, `FailureReason = ex.Message` — not a crash, not a silent
  success.

Document this plainly as a hardcoded placeholder mapping (mirroring the existing `DemoKycTier`/
`DemoKycVerificationType` comment style in `AccountOpeningService.cs` word-for-word in spirit — "a
hardcoded, clearly-labeled demo stub decision ... not a real product-selection choice"), not a real
product-catalog lookup. A real deployment would need a real way for a caller to choose or be assigned a
product; that's a future task, not this one. Note explicitly in the PR that this also means an
account-opening request for any `JurisdictionCode` other than `"NG"` will fail at this new step today
(since only `NG-SAV-STD`/`NG-CUR-STD` are seeded) — this is the *correct* failure mode (a clear,
recorded `Failed` status with a plain reason), not a bug to work around in this task.

### 3. `IProductApiClient` / `ProductApiClient`

Mirror `ILedgerApiClient`/`LedgerApiClient` exactly (read both files directly before writing this one) —
same `HttpClient`-based pattern, same `ReadOrThrowAsync`/`DownstreamResponseReader` helpers, registered
the same way in `Program.cs` (`builder.Services.AddHttpClient<IProductApiClient, ProductApiClient>(...)`,
reading a new `DownstreamServicesOptions.ProductApi` entry exactly like `IdentityApi`/`LedgerApi` already
do). One method for this task:

```csharp
Task<AccountProductBindingResponse> BindAccountToProductAsync(
    BindAccountToProductRequest request, CancellationToken cancellationToken);
```

calling `POST v1/account-bindings` on `Garbati.ProductApi` with body `{ IdempotencyKey, AccountNumber,
ProductCode }` (matching `Garbati.ProductApi`'s own `CreateAccountProductBindingRequest` shape exactly —
read that file directly). Idempotency key for this call: `orch:{opening.OperationId}:bind-product`, same
derived-key convention every other downstream call in this method already uses.

**No special exception handling needed for a retry** — unlike `AttachMandateAsync` (which has no
idempotency key and needs a `MandateAlreadyAttachedException` catch), `POST /v1/account-bindings` **does**
take an idempotency key, so a retry with the same key and the same account/product just returns the same
binding (200), the ordinary idempotent-create behavior every other step in this method already relies
on. Confirm this by reading `Garbati.ProductApi`'s own `AccountProductBindingService.CreateAsync` and its
test for this exact scenario before assuming otherwise.

Add `DownstreamServiceOptions ProductApi` to `Configuration/DownstreamServicesOptions.cs` and the matching
`Check(options.ProductApi, "DownstreamServices:ProductApi", errors)` line in the validator — same shape as
the two existing entries, nothing new invented.

### 4. Dev configuration — both sides, additive only

On `Garbati.ProductApi`'s own `appsettings.Development.json`: add a new `dev-orchestrator` client
(scope `product.catalog.write` only — this integration only ever creates bindings, never reads one back
or asks for a fee quote) to the existing `ApiKeys.Clients` list, **additive only** — do not touch the
existing `dev-ops-api` entry. Compute the `KeyHash` the same way every other cross-service dev client on
this platform has been computed this session: `openssl dgst -sha256 -binary | base64` on a literal
placeholder string of your choosing (e.g. `dev-orchestrator-product-key`), matching the naming convention
`DownstreamServices:IdentityApi:ApiKey`/`DownstreamServices:LedgerApi:ApiKey` already use for their own
placeholder strings in `Garbati.OrchestratorApi`'s own `appsettings.Development.json`.

On `Garbati.OrchestratorApi`'s own `appsettings.Development.json`: add a `ProductApi` entry under
`DownstreamServices` (`BaseUrl: "http://localhost:5240"` — confirmed directly in
`Garbati.ProductApi/src/Garbati.ProductApi/Properties/launchSettings.json`'s own `applicationUrl`), `ApiKey`
set to the exact same placeholder string you hashed on the `Garbati.ProductApi` side above.

## Explicitly out of scope

- Any fee-quote lookup in `TransferService` — see Context above.
- Any change to `Garbati.ProductApi` itself.
- A real (non-hardcoded) way for a caller to choose a product — the `{JurisdictionCode}-{SAV|CUR}-STD`
  mapping is a deliberate, documented placeholder, not a design to improve on in this task.
- Wiring this up on Azure (no deploy changes) — this task is local-dev/code only, matching how
  `Garbati.ProductApi` itself isn't deployed yet either.

## Tests to write

- A full account-opening happy path now runs 5 steps, not 4 — `ProductCode` is set on the completed
  operation's response, and a fake/mock `IProductApiClient` recorded exactly one call with the expected
  `AccountNumber`/`ProductCode`/idempotency key.
- `BindProduct` step failure (e.g. the fake client throws) leaves `CurrentStep == BindProduct`,
  `Status == Failed`, a clear `FailureReason` — and a subsequent retry resumes from `BindProduct`, not
  from `CreateCustomer` (mirror the existing step-failure/retry tests already in this test file for the
  other three steps exactly).
- An `AccountType` with no mapping (anything other than `"Savings"`/`"Current"`) fails clearly at this
  step with the specific exception message, not a generic/confusing error.
- A retry of an already-`Complete` operation remains a no-op (unchanged from today — just confirm the new
  step doesn't break this existing guarantee).
- `DownstreamServicesOptionsValidator` rejects a missing/invalid `ProductApi` entry, mirroring the
  existing tests for `IdentityApi`/`LedgerApi`.

## PR

Open a pull request from `feature/bind-product-step` against `main` on
`AbubakarImam/Garbati.OrchestratorApi` (branch-protected — get `test`/`docker` CI green first). Be
explicit in the description about the hardcoded `ProductCode` mapping and why it's scoped this narrowly.
Do not merge — only the owner merges.
