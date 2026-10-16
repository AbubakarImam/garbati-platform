# QA Report — Garbati.IdentityApi PR #1 (T-016 foundation)

**Verdict: Approve** (with Should-fix items the owner should note, none blocking)

## Scope-boundary check (most important item) — PASS

Grepped `src/` and `tests/` for `mandate`, `authorization/`, `jwt`, `jws`, `es256`, `signing`, `grant`, `privatekey`. Every hit is either:
- a comment explaining the deliberate no-op / forward reference for T-017, or
- scope constant names (`IdentityScopes.MandateRead/Write`) that exist but are **not wired to any endpoint or policy check** (confirmed — only `CustomerRead`, `CustomerWrite`, `KycManage` are used by `[Authorize]` in `CustomersController.cs`).

No `Mandates`, `MandatePermissions`, `MandateApprovalPolicies`, `PendingApprovalRequests`, `ApprovalRecords` entities exist. No `/v1/mandates/*` or `/v1/authorization/*` controller/route exists. No JWT/JWS/signing code exists anywhere. The scope boundary is genuinely respected.

## Findings

1. **(Nit)** `ICurrentClient`/`HttpCurrentClient` (`Authorization/CurrentClient.cs`) is registered in `Program.cs` but never consumed anywhere in the codebase. Harmless dead code, but worth removing or using before T-017 builds on top of it.

2. **(Should fix)** `CustomerStatusChangeRequest.cs`'s comment claims "Reason is required for suspend/close/mark-deceased" but no validator is ever invoked for this request type (unlike `CreateCustomerRequest`/`KycDecisionRequest`, which go through dedicated validators in the controller). Confirmed via grep that `request.Reason` is only ever read, never null-checked, and no test exercises a missing reason on a privileged status change. The comment overstates what the code does — DESIGN.md Section 6.3 expects privileged overrides to record "actor and reason," and today a `close`/`suspend`/`mark-deceased` call with no `Reason` silently succeeds and audits `"{previous} -> {new}"` with no reason captured. Low severity (no money/security impact at this stage, since nothing consumes the reason yet), but the owner should know this enforcement doesn't actually exist yet, contrary to what the comment implies.

3. **(Nit)** No standalone `LEARNING_LOG.md` file exists in this repo; the learning-log content (five Q&A) is embedded only in the PR description. Resolved by the coordinator: the authoritative `LEARNING_LOG.md` for this project lives in `garbati-platform`, not per-service — the coordinator adds the entry there after merge, matching every prior merged feature (chains, fee/tax accounts). No action needed in this repo.

4. **(Nit)** `appsettings.Development.json` commits two API key hashes (`dev-orchestrator`, `dev-ops`) for local dev. These are SHA-256 hashes, not raw keys, so this matches the "hash-only configuration" rule and isn't a secret leak — matches LedgerApi's existing pattern, not a new risk.

5. **(Informational, not a defect)** `#TBD-3` (no idempotency key on `POST /v1/customers`) was correctly surfaced as an open question rather than silently decided. Resolved separately by the owner: add it (see follow-up below).

## Point-by-point against the task file

- **Entities**: `Customer`, `CustomerIdentityData`, `KycVerification`, `KycTierPolicy`, `AuditEvent`, `OutboxMessage` — all present, match the task file's field lists. `KycTierPolicy`'s evidence/operations lists are jsonb-backed string lists (flexible shape, not hardcoded), seeded with three clearly-commented placeholder Nigerian rows via a dedicated, reversible migration (`SeedPlaceholderKycTierPolicies`). Matches decision #11/#24 exactly.
- **Endpoints**: the four from the task file are present, plus `close`/`mark-deceased`. Checked this addition against DESIGN.md Section 2.1's status table directly — `Closed` and `Deceased` are explicitly distinct, terminal, irreversible states with their own automatic cascade, clearly different from reversible `Suspended`. Overloading `suspend` for them would have been the worse design choice; adding separate endpoints is the right call, honestly flagged as `#TBD-1`.
- **Closed/Deceased cascade**: `CustomerService.CascadeMandateRevocationAndFreezeAsync` is a literal `Task.CompletedTask` no-op, preceded by a seven-line comment spelling out exactly what T-017 must implement there. Honest, non-faked gap, exactly as the task required.
- **Regression test for the no-op**: `CustomerServiceTests.CloseCustomerAsync_WritesNoOutboxMessage_BecauseMandatesDoNotExistYet` — read and ran it. Creates a real customer against real Postgres (Testcontainers), calls `CloseCustomerAsync`, asserts `OutboxMessages.CountAsync() == 0` in a fresh `DbContext`. Genuine behavioural assertion, will fail the day T-017 starts writing outbox rows on this path — forcing a deliberate update rather than silent staleness.
- **GET redaction**: `CustomerResponse` has no `FullName`/`Email`/`PhoneNumber`/BVN/NIN property at all. Leakage is structurally impossible, not just filtered.
- **No BVN/NIN column**: confirmed deliberate, pending an encryption-at-rest decision.
- **Auditing**: `AuditLogger.Record` stages the `AuditEvent` on the same `IdentityDbContext` that `CustomerService` later calls `SaveChangesAsync` on — one database transaction, matching Section 6.3. Holds even in the `CreateCustomerAsync` retry-on-collision loop.
- **API-key auth/scopes**: near-identical to LedgerApi's real source (timing-safe `FixedTimeEquals`, hash-only config, `IValidateOptions` with `ValidateOnStart()`). Scopes wired: exactly the three the task file said were relevant; `identity.mandate.*`/`identity.authorization.check`/`identity.ops.freeze` defined but unused, as explicitly permitted. A reflection-based test (`ControllerAuthorizationTests`) guards that every controller action carries a recognized `[Authorize(Policy=...)]`.

## Tests and build — ran independently

- `dotnet build`: clean, 0 warnings, 0 errors.
- `dotnet test` (real Testcontainers Postgres): **90/90 passed**, matching the PR's claim exactly.
- Read (not just ran) the validators: `CreateCustomerRequestValidator` rejects empty/whitespace `FullName`, out-of-range `JurisdictionCode`, malformed email (weak check, `@`-only, but low-risk and consistent with "keep this minimal" scope). `KycDecisionRequestValidator` correctly requires `KycTier` only when `KycStatus == Verified`.

## CI — `gh pr checks 1`, actual state

```
deploy   skipping   (push-to-main only, correctly not a required check)
docker   pass       1m37s
test     pass       48s
```

Both required checks (`test`, `docker`) are green. The CI `docker` job runs `docker/build-push-action` with `target: final` and `target: migrator` — the exact two targets in the committed `Dockerfile`, on GitHub's hosted runners. The builder's choice to rely on CI's `docker` job (after a local build hung on an `mcr.microsoft.com` pull in their sandbox) is sound — CI exercises the real artifact.

## Appendix B `#TBD` items — sanity-checked individually

1. `close`/`mark-deceased` as separate endpoints — reasonable, matches DESIGN.md's status table.
2. 409 on repeat `Close`/`MarkDeceased` — reasonable, matches LedgerApi's own `AlreadyReversedException → 409` convention (confirmed this exists in LedgerApi's `ExceptionHandlingMiddleware.cs`).
3. No idempotency key on `POST /v1/customers` — correctly raised as an open question. (Owner has since decided: add it.)
4. No BVN/NIN column yet — the right call given no encryption-at-rest decision exists; avoids inventing a plaintext column for sensitive data.
5. The cascade no-op — honest, commented, tested.

None of the five weaken a ratified decision or an audit/security rule.

## What could not be checked

- Could not verify the Azure `deploy` job actually deploys anything — intentionally unwired, correctly never runs on a PR, doesn't gate merge.
- Did not independently build the Docker image locally (same `mcr.microsoft.com` pull friction present in this environment too); relied on CI's `docker` job, verified it exercises the same Dockerfile/targets.
- Did not review LedgerApi's own QA history/PRs beyond spot-diffing the specific files this PR claims to mirror.
- Did not test concurrent/parallel `CreateCustomerAsync` calls beyond reading the retry-on-`23505` logic.

## Five interview questions for the owner

1. **Q: If someone calls `close` twice on the same customer with two different reasons, what happens, and why?**
   A: The second call gets a 409 Conflict, not success. Suspend/reactivate are reversible, so a repeat is harmless. Close and mark-deceased are one-way doors, so a second call — possibly carrying different information — is never silently dropped.
2. **Q: The design says closing a customer should automatically revoke their account access. Does this PR do that?**
   A: No, on purpose. "Mandate" (the thing linking a customer to a bank account) doesn't exist in the code yet — it's built next. This PR only changes status and audits the change; the method where revocation belongs is an empty, commented stub, and a test proves nothing is faked.
3. **Q: How do we know `GET /v1/customers/{id}` can never leak a name, email, or BVN/NIN?**
   A: The response object has no field to put that information in — it's structurally impossible, not a filter that could have a bug. There's also no BVN/NIN column in the database at all yet, since nobody has decided how to encrypt one safely.
4. **Q: The code defines scopes like `identity.mandate.read`/`write` even though nothing uses them. Isn't that premature?**
   A: Deliberately allowed — the task said it was fine to write the full scope list once, as long as only the three scopes actually relevant to this task are wired to real endpoints. Confirmed the unused scopes aren't attached to any authorization policy, so they can't grant access to anything yet.
5. **Q: Why add a database table for outbox messages when nothing consumes them yet?**
   A: So the next task can write to it without a schema change. A test proves the table has the right shape, and a separate test proves today's `close`/`mark-deceased` calls write zero rows into it — the gap is visible and tested, not assumed away.
