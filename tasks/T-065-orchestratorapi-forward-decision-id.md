# T-065: OrchestratorApi forwards DecisionId to LedgerApi (decision #11, T-063)

Owner: builder. Implements `Garbati.OrchestratorApi/docs/DESIGN.md` decision #11 (`main` @ `1dd9b46`,
including its own scope correction — read the decision in full, it's short). This is the
`OrchestratorApi` half of T-064/decision #52; **depends on T-064 having merged to `LedgerApi`'s
`main` first** — `TransferRequest` has no `decision_id` field to send until then. Confirm T-064 is
merged before starting; if it isn't yet, stop and say so rather than building against a field that
doesn't exist.

## What to build

One change, in exactly one place: `TransferService.cs`'s call to `ledgerApiClient.TransferAsync`
(`src/Garbati.OrchestratorApi/Services/Implementations/TransferService.cs`, the
`new LedgerDto.TransferRequest(...)` construction). Add `authorization.Decision.DecisionId` as the
new `DecisionId` argument — a value this method already has in scope (it's used a few lines below in
the audit `Details`/`EntityId` fields) and already receives from `Garbati.IdentityApi` on every call.
You will need to:

1. Update `Clients/Contracts/Ledger/TransferRequest.cs` (the client-side mirror of LedgerApi's own
   contract) to add the new optional `DecisionId` field — read LedgerApi's real, just-built
   `TransferRequest` shape first (`LedgerApi/src/LedgerApi/Contracts/Requests/TransferRequest.cs`) to
   match it exactly, not guess.
2. Pass `authorization.Decision.DecisionId` through in the one call site named above.

## What this task does not include

- **`ReversalService` is explicitly out of scope** (decision #11's own correction, confirmed directly
  against the real code before this task was written): it calls LedgerApi's `ReversalRequest(
  OriginalTransactionReference, Reason)`, a narrower contract with no field for this and none being
  added — a reversal references an existing transaction, it doesn't post a new `TransferRequest`. Do
  not touch `ReversalService.cs` or `Clients/Contracts/Ledger/ReversalRequest.cs`.
- No chain/fee/tax integration — `OrchestratorApi` doesn't post `ChainRequest` anywhere today
  (confirmed: zero real usage, only a comment reference), and this task doesn't add any.
- No change to `value_date`/`branch_code`/`channel` (LedgerApi decisions #49-#51) — those have no
  real caller yet, by design, and this task doesn't become that caller. `DecisionId` is the one field
  with an immediate, real, already-available value to forward.
- No change to `LedgerApi` itself — that's T-064, already built.

## Acceptance

- Existing test suite still green, plus a test confirming `TransferAsync` is called with the real
  `DecisionId` from the authorization response (mirror whatever existing test already asserts on the
  call's other arguments — read it first rather than writing a new pattern).
- `DESIGN.md` decision #11 updated if anything above had to change from what's written there, with
  the deviation and why.
- No production-data or migration concern — this is a client-contract change, not a schema change, in
  a service with no database field of its own affected.
