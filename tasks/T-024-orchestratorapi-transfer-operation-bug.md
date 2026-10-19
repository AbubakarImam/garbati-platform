# T-024 — OrchestratorApi: fix invalid `Operation` value sent to IdentityApi on transfer

## Found by

Live end-to-end verification against real deployed Azure infrastructure (not a mock, not local dev),
18 Oct 2026, immediately after T-022/PR #1 was deployed for the first time. This is why it was never
caught by `TransferServiceTests.cs`: those tests mock `IIdentityApiClient` directly and never send the
`Operation` string through IdentityApi's real `AuthorizationRequestValidator`.

## The bug

`src/Garbati.OrchestratorApi/Services/Implementations/TransferService.cs`, line ~35:

```csharp
authorization = await identityApiClient.RequestAuthorizationAsync(
    new IdentityDto.AuthorizationRequest(
        request.RequestedBy, "Transfer", request.DebitAccountId, request.CreditAccountId,
        request.Amount, request.Currency, request.IdempotencyKey),
    cancellationToken);
```

`"Transfer"` is passed as `Operation`. IdentityApi's `AuthorizationRequestValidator`
(`Garbati.IdentityApi/src/Garbati.IdentityApi/Validation/AuthorizationRequestValidator.cs`) only accepts
values that parse as the `MandateOperation` enum (`View`, `Debit`, `Credit`) — or the literal string
`"Reversal"`, handled separately. `"Transfer"` is none of these, so IdentityApi correctly rejects the
request with `400 Bad Request: "'Transfer' is not a recognized Operation."`

`IdentityApiClient.RequestAuthorizationAsync` only has a specific branch for IdentityApi's `422` (denial);
every other non-success status — including this `400` — falls through to the generic
`DownstreamApiException`, which `ExceptionHandlingMiddleware.Map` maps to a bare `500 "An unexpected error
occurred."` with no detail. **Every live transfer call fails with a 500**, with nothing in the response
pointing at the real cause.

Confirmed live by replicating the exact call IdentityApi receives:

```
curl -X POST https://garbati-identityapi.azurewebsites.net/v1/authorization/requests \
  -H "X-Api-Key: $IDENTITY_KEY" -H "Content-Type: application/json" \
  -d '{"requestedBy":"CUS-...","operation":"Transfer","debitAccountId":"...","creditAccountId":"...","amount":1500,"currency":"NGN","idempotencyKey":"..."}'
→ 400 ["'Transfer' is not a recognized Operation."]
```

## The fix (not a design question — the contract already settles this)

Change the literal to `"Debit"`. A transfer is, from IdentityApi's point of view, exactly the question
"may this requester debit this account for this amount?" — `MandateOperation.Debit` is what that check
already means (DESIGN.md's own mandate-permission vocabulary, Section 2.2). No new value is being
invented, no contract is changing, and this doesn't touch money rules, auth scopes, or anything on the
human-only-gate list in `AGENTS.md` — it's a one-line implementation bug where the code didn't match the
existing, already-ratified contract.

```csharp
request.RequestedBy, "Debit", request.DebitAccountId, request.CreditAccountId,
```

## Scope

- Fix the one-line bug above.
- Add a regression test in `TransferServiceTests.cs` (or a new test) that asserts the `Operation` value
  sent to `IIdentityApiClient.RequestAuthorizationAsync` is `"Debit"`, not `"Transfer"` — this is exactly
  the kind of mismatch the existing mock-based tests silently let through, so the test must actually
  assert on the captured argument, not just that the mock was called.
- Do not touch `AccountOpeningService` or anything else — this file only.
- Out of scope: re-verifying/forwarding the grant token (decision #47/D-6, already deliberately deferred),
  any change to IdentityApi, any change to the `MandateOperation` enum.

## Out of scope

Everything else. This is a one-line fix plus one regression test.
