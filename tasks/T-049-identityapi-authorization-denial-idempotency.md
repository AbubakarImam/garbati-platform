# T-049: IdentityApi — dedupe a retried authorization *denial* (D-16)

Target repo: `Garbati.IdentityApi`. Branch: `fix/authorization-denial-idempotency`.

## Context

D-16 (`TASKS.md`): `AuthorizationService.RequestAuthorizationAsync` mints a fresh `Guid.NewGuid()`
`DecisionId` and writes a brand-new `AuthorizationDecision` row on *every* call, with no lookup by
`IdempotencyKey` anywhere. A client retry (lost response, timeout) on the ordinary single-holder transfer
path currently produces a second, independent decision + a second audit event for what was really one
logical request.

**Owner's decision on scope** (read this before changing anything): a retry of an *approved* request
should keep getting a fresh grant — no behavior change there. The signed grant token is deliberately
never persisted (decision #13 — see `AuthorizationDecision.cs`'s header comment), so unlike every other
idempotent-create endpoint on this platform, a retried approval has nothing stored to "replay." Money
safety on a re-granted retry is backstopped by `LedgerApi`'s own
`ix_transactions_initiated_by_idempotency_key` constraint regardless of how many grants IdentityApi
issues — that's the real guard, not this endpoint.

That leaves exactly one real gap to close: **a retried *denial* writes a duplicate row.** This is pure
waste (an extra row, an extra `AuthorizationDenied` audit event) with no money-safety angle either way,
but it's still worth fixing cleanly since the data is otherwise misleading (looks like two denied
attempts when it was one request retried once).

**`ReversalAuthorizationService.RequestReversalAsync` needs NO change.** Read it directly before assuming
otherwise — it has no denial branch at all. Per decision #31, the ops channel's own maker-checker already
gates a reversal request before this method is ever called; every call here unconditionally sets
`Approved = true`. There is nothing to dedupe on that path under the owner's decision above. Do not touch
`ReversalAuthorizationService.cs` in this task.

## The exact fix

Everything changes inside `AuthorizationService.PersistDecisionAsync` (`Services/Implementations/
AuthorizationService.cs`) — the single place both `RequestAuthorizationAsync`'s denial call and
`RecordDenialAsync`'s call (used by `JointApprovalService`, T-018) already go through. Fixing it here
covers both denial call sites with no separate change needed in `JointApprovalService.cs`.

### 1. Migration: partial unique index

Add, in `OnModelCreating`'s existing `modelBuilder.Entity<AuthorizationDecision>(entity => { ... })`
block (`Data/IdentityDbContext.cs`, right where the existing `entity.HasIndex(d => d.DecisionId)
.IsUnique();` lives):

```csharp
entity.HasIndex(d => new { d.RequestedBy, d.IdempotencyKey })
    .IsUnique()
    .HasFilter("approved = false")
    .HasDatabaseName("ix_authorization_decisions_requested_by_idempotency_key_denied");
```

**Why partial, not a plain unique index**: a plain unique index on `(RequestedBy, IdempotencyKey)` would
also block the second row a retried *approval* is supposed to get under the owner's decision above. The
`WHERE approved = false` filter means Postgres only enforces uniqueness among denied decisions — exactly
the one case being fixed, nothing else.

One additive EF Core migration. It only creates an index; no existing row's values change. If any
`RequestedBy`+`IdempotencyKey` pair somehow already has two or more denied rows in production today, the
migration will fail cleanly at deploy time (index creation refuses, deploy step aborts) rather than
silently corrupting anything — if that happens, stop and report it; don't work around it by weakening the
index.

### 2. Catch-and-resolve in `PersistDecisionAsync`, mirroring `CustomerService.CreateCustomerAsync`'s exact pattern

Read `Services/Implementations/CustomerService.cs`'s `CreateCustomerAsync` first — this fix is the same
shape, not a new pattern: attempt the insert directly, catch the unique-violation, resolve by querying,
compare request fields, 409 on a real mismatch. Do **not** add a separate "check first" read before the
insert — that would reintroduce the exact race a database constraint exists to close.

In `PersistDecisionAsync`, wrap the existing `await dbContext.SaveChangesAsync(cancellationToken);` (only
reached when `approved == false` for this new catch — an approved insert never hits the new index's
filter, so let that case fall through unchanged):

```csharp
try
{
    await dbContext.SaveChangesAsync(cancellationToken);
    return decision;
}
catch (DbUpdateException ex) when (!approved
    && ex.InnerException is PostgresException { SqlState: "23505", ConstraintName:
        "ix_authorization_decisions_requested_by_idempotency_key_denied" })
{
    dbContext.Entry(decision).State = EntityState.Detached;
    // (detach the audit entry too, the same way CreateCustomerAsync detaches everything it staged
    // before the retry/resolve — check what auditLogger.Record returns and detach that too if it's
    // tracked the same way)

    var existing = await dbContext.AuthorizationDecisions.AsNoTracking().FirstOrDefaultAsync(
        d => d.RequestedBy == request.RequestedBy && d.IdempotencyKey == request.IdempotencyKey
            && !d.Approved, cancellationToken);

    if (existing is null)
        throw new InvalidOperationException("Error retrieving the result of the existing idempotency key.");

    // Same key, different request: refuse rather than report the old denial as this one's (409) —
    // compare the request's own canonical fields, not computed outputs like DenialReason.
    if (existing.Operation != request.Operation
        || existing.DebitAccountId != request.DebitAccountId
        || existing.CreditAccountId != request.CreditAccountId
        || existing.Amount != amount
        || existing.Currency != request.Currency)
    {
        throw new DuplicateIdempotencyKeyException(
            $"Idempotency key '{request.IdempotencyKey}' was already used for a different request.");
    }

    return existing;
}
```

Confirm whether `auditLogger.Record(...)`'s return value is itself tracked by `dbContext` (check how
`CreateCustomerAsync` detaches `audit` before its own retry) — if so, detach it the same way before
returning `existing`, so a second `AuthorizationDenied` audit event isn't written for a request that
didn't actually happen again.

### 3. Propagate the resolved decision's real identity back to the caller

`RequestAuthorizationAsync` currently throws `AuthorizationDeniedException` using the `decisionId` it
generated locally *before* calling `PersistDecisionAsync` — on a resolved duplicate, that local id is
wrong (a fresh guid that was never actually persisted). Change:

```csharp
if (reasonCode is not null)
{
    await PersistDecisionAsync(decisionId, request, amount, now, approved: false, reasonCode, mandateId: null, cancellationToken);
    throw new AuthorizationDeniedException(
        decisionId, reasonCode, $"Authorization denied for decision '{decisionId}': {reasonCode}.");
}
```

to use the entity `PersistDecisionAsync` actually returns:

```csharp
if (reasonCode is not null)
{
    var decision = await PersistDecisionAsync(decisionId, request, amount, now, approved: false, reasonCode, mandateId: null, cancellationToken);
    throw new AuthorizationDeniedException(
        decision.DecisionId, decision.DenialReason!,
        $"Authorization denied for decision '{decision.DecisionId}': {decision.DenialReason}.");
}
```

So a retried denial surfaces the *original* decision's id and reason, not a phantom new one. (
`RecordDenialAsync`'s own caller, `JointApprovalService`, already ignores the returned value — no change
needed there, but confirm this directly rather than assuming it.)

## Explicitly out of scope

- `ReversalAuthorizationService.cs` — no change, see above. State this explicitly in the PR description
  so a reviewer doesn't go looking for a missing fix there.
- Any change to the *approved*-retry behavior — a retry of an approved request still gets a fresh
  `DecisionId` and a fresh grant, exactly as today. That is a deliberate decision, not a gap.
- A `RequestHash` column (unlike `CustomerService`'s pattern) — every field needed to detect a "same key,
  different body" conflict already exists as a plain column on `AuthorizationDecision`. Don't add one.
- `JointApprovalService.cs` itself — unchanged; it benefits from this fix automatically because it calls
  through `RecordDenialAsync` → `PersistDecisionAsync`.

## Tests to write

- Retry of an identical denied request (same `RequestedBy` + `IdempotencyKey` + same Operation/
  DebitAccountId/CreditAccountId/Amount/Currency): second call returns/throws the **same** `DecisionId`
  and `DenialReason` as the first; confirm via a direct query that only **one** row exists in
  `AuthorizationDecisions` for that key, and only one `AuthorizationDenied` audit event was recorded.
- Same `RequestedBy` + `IdempotencyKey` but a different `Amount` (or any other canonical field): second
  call throws `DuplicateIdempotencyKeyException` (409), original row untouched.
- A denial followed by a *different* idempotency key for the same `RequestedBy` and otherwise-identical
  request: both rows persist independently — confirms the fix is scoped to the key, not just the
  requester.
- An approved request, retried with the same key: still gets a second row with a fresh `DecisionId` (the
  owner's decided behavior, unchanged) — a regression test proving this fix did *not* accidentally start
  blocking approval retries, since that's the easiest way to get this wrong.
- `JointApprovalService`'s existing denial-path tests still pass unchanged (confirms `RecordDenialAsync`'s
  callers aren't affected in any way other than the dedup itself).
- Migration applies cleanly against a fresh test database; confirm the new index actually exists and is
  filtered (`approved = false`) via `information_schema` or by attempting to insert two denied rows
  directly and confirming violation, then two approved rows and confirming no violation.

## PR

Open a pull request from `fix/authorization-denial-idempotency` against `main` on
`AbubakarImam/Garbati.IdentityApi` (branch-protected — get `test`/`docker` CI green first). Be explicit in
the description about: why `ReversalAuthorizationService` needed no change, why the index is partial, and
why approved-retries are deliberately left producing a fresh grant. Do not merge — only the owner merges.
