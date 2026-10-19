# T-030 — OrchestratorApi: add `POST /v1/reversals`

## Context

`garbati-platform` D-14, owner-approved 19 Oct 2026. Found while designing the ops console's backend-for-frontend (`Garbati.OpsApi`, T-028): `OpsApi`'s reversal maker-checker workflow can get a staff request all the way to "Approved," but nothing in the platform can actually execute it — `IdentityApi` already has the authorization side fully built (T-020), but `OrchestratorApi` has no reversal endpoint at all (its own v1 design explicitly deferred reversal, before T-020 existed). This task closes that gap, mirroring the existing `TransferService` shape exactly — this is additive, not a change to any existing endpoint or behavior.

## What to build

**New contract**, `Contracts/Requests/ReversalRequest.cs` (same file convention as the existing `TransferRequest.cs`):

```csharp
public record ReversalRequest(
    string IdempotencyKey,
    string RequestedBy,
    string OriginalTransactionReference,
    decimal Amount,
    string Currency,
    string ReasonCode) : IIdempotentRequest;
```

`Amount`/`Currency` are caller-supplied, not looked up — `LedgerApi` has no lookup-by-reference endpoint to derive them from (its `TransactionResponse` doesn't carry an amount at all; confirmed directly against its source before writing this task). This mirrors `TransferRequest` already requiring the caller to supply amount/currency rather than deriving it from anywhere — not a new pattern, just applied to a new operation.

**New response**, `Contracts/Responses/ReversalResultResponse.cs` (mirrors `TransferResultResponse.cs`):

```csharp
public record ReversalResultResponse(
    string DecisionId, string Reference, string Status, string? FailureReason);
```

**New endpoint**, `POST /v1/reversals`, new scope `orchestrator.reversals` (one scope per operation group — matches this service's existing convention of `orchestrator.account-openings`/`orchestrator.transfers`, one per operation group, not one shared scope; see `OrchestratorScopes.cs`'s own comment for why that convention was chosen).

**New service**, `ReversalService` (mirrors `TransferService.cs` exactly, same two-step shape, same exception handling, same audit-log pattern):

1. Call `IdentityApi POST /v1/authorization/requests` with:
   - `Operation: "Reversal"`
   - `DebitAccountId: request.OriginalTransactionReference` — per `IdentityApi`'s own documented convention (`AuthorizationRequest`'s own comment): "DebitAccountId itself carries that single reference for a Reversal request... there is no dedicated 'original transaction reference' field."
   - `CreditAccountId: null`
   - `Amount`, `Currency`, `IdempotencyKey`: passed straight through from `request`.
   - `ReasonCode: request.ReasonCode`
   - **This call needs a new scope grant on `OrchestratorApi`'s existing `IdentityApi` service key: `identity.ops.reversal`** (an App Service configuration change on the deployed credential, not a code change — same kind of operational step as any other scope grant on this platform). `OrchestratorApi` does not have this scope today; without it, `IdentityApi` will `403`.
   - On `AuthorizationDeniedException`: same handling `TransferService` already has (audit `ReversalDenied`, rethrow — middleware maps it to `422` with `decisionId`/`reasonCode`, unchanged pattern).
2. On authorization success, call `LedgerApi POST /api/reversals` with `{OriginalTransactionReference: request.OriginalTransactionReference, Reason: request.ReasonCode}` — this is `LedgerApi`'s existing `ReversalsController` (`LedgerScopes.Reverse` policy), already built, not new. **`OrchestratorApi` needs a new scope grant on its existing `LedgerApi` service key: `ledger.reverse`** (same kind of operational step as above) — it does not have this scope today.
3. Report the outcome back to `IdentityApi POST /v1/authorization/requests/{id}/reversal-outcome` (already built, T-020) — `RecordReversalOutcomeRequest { Outcome, RecordedBy, Notes }`:
   - `LedgerApi`'s reversal call succeeds → `Outcome: "Completed"` (money actually moved back).
   - `LedgerApi`'s reversal call fails (e.g. insufficient balance on the account being reversed from) → `Outcome: "WriteOff"`. `ReversalOutcomeKind` is a deliberately two-state enum (`Completed`/`WriteOff` only, decision #34, reconfirmed by T-021's research into real banking practice) — a failed reversal is exactly the "money not recovered" case `WriteOff` exists for, not a third state to invent.
   - `RecordedBy: "OrchestratorApi"`, `Notes: null` in both cases for v1 (nothing richer than the outcome itself is captured yet).
   `IdentityApi`'s own design (Section 8) anticipated this call coming from "the ops channel (or the Orchestrator)" — this task settles it as the Orchestrator, since it's the service that directly observes `LedgerApi`'s real result, not a relay of something another service saw.
4. Audit log entries mirroring `TransferService`'s existing three (`ReversalDenied`/`ReversalAuthorized`/`ReversalPosted` — new `AuditActions` constants, same naming convention as the existing `TransferDenied`/`TransferAuthorized`/`TransferPosted`).

## Scope

- `Contracts/Requests/ReversalRequest.cs`, `Contracts/Responses/ReversalResultResponse.cs`, `Services/Implementations/ReversalService.cs` (+ its interface), `Controllers` (new action or new controller — your call, consistent with this repo's existing controller-per-operation-group pattern), `Validation/ReversalRequestValidator.cs` (mirror `TransferRequestValidator.cs`'s shape), `OrchestratorScopes.cs` (add `Reversals`).
- Tests mirroring `TransferServiceTests.cs`'s existing shape: happy path, IdentityApi denial (no LedgerApi call made), LedgerApi failure after authorization succeeds, and a regression test for the `Operation` value sent to `IdentityApi` (the exact kind of gap T-024 found and fixed for transfers — don't let the same class of bug ship here uncaught).
- Deployment: grant `OrchestratorApi`'s existing `IdentityApi` and `LedgerApi` service keys the two new scopes named above. This is an Azure App Service configuration change, not a code change — note it clearly in the PR description so it isn't missed before this is live-tested.

## Out of scope

- `Garbati.OpsApi`'s own build (T-028's design, not yet a build task) — this task only builds what `OpsApi` will eventually call.
- Any change to `IdentityApi` or its existing reversal-authorization/outcome endpoints (T-020) — already correct, already built.
- Any change to `LedgerApi`'s existing `ReversalsController`/`ChainsController` — already correct, already built.
- Grant-token forwarding/local verification (decision #47/D-6) — still deliberately deferred everywhere on this platform; this task doesn't touch it either.
