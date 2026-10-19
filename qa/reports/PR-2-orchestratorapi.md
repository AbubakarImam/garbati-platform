# QA Report — Garbati.OrchestratorApi PR #2 (T-024)

**Verdict: Approve**

## 1. Task fidelity — PASS

`git diff main..fix/t-024-transfer-operation-literal --stat` shows exactly two files touched:
- `src/Garbati.OrchestratorApi/Services/Implementations/TransferService.cs` (+6/-1)
- `tests/Garbati.OrchestratorApi.Tests/Services/TransferServiceTests.cs` (+34/-0)

The fix is the one line the task specified: `"Transfer"` → `"Debit"` in the `IdentityDto.AuthorizationRequest` constructor call, plus an explanatory comment. One regression test was added, nothing else. No scope creep — confirmed by grepping the whole `src/` tree for `"Transfer"` as a literal (only remaining hits are `EntityType = "Transfer"` audit-log tags, unrelated) and for `RequestAuthorizationAsync` call sites (exactly one, in `TransferService.cs`). `AccountOpeningService`, `IdentityApiClient.cs`, and the exception-masking path in that client were all untouched, matching "out of scope."

## 2. Architecture/DESIGN.md — PASS, not built on "Proposed"

Independently read IdentityApi's `docs/DESIGN.md` Section 2.2: `MandatePermissions.Operation` (`View`, `Debit`, `Credit`) is documented as part of the **ratified** mandate model, and `MandateEligibility.CheckAsync` is called with `request.DebitAccountId` and the parsed `operation` — i.e., the real authorization logic checks "does this requester hold a Debit-permission mandate on the debit account," exactly matching the fix. Correct application of an already-decided contract, not a new decision disguised as a bug fix.

## 3. Independent verification against IdentityApi — PASS

Read `Validation/AuthorizationRequestValidator.cs` and `Entities/MandateOperation.cs` directly: enum is exactly `View, Debit, Credit`; `"Debit"` parses successfully, `"Transfer"` does not. Independently confirms the PR's claim.

## 4. Regression test quality — PASS, empirically proven to catch the bug

Traced `FakeIdentityApiClient` — the test's `capturedRequest` is the real argument `TransferService` constructed, not a copy. Assertion is `Assert.Equal("Debit", capturedRequest!.Operation)`, a value assertion.

QA reverted the fix locally (`"Debit"` → `"Transfer"`), reran: new test failed exactly as expected (`Expected: "Debit", Actual: "Transfer"`), all other tests still passed. Restored the file, reran: 60/60 clean.

## 5. Money safety / security

No code path touches amounts, rounding, balances, locking, or persistence beyond existing (unchanged) audit-log writes. No secrets, no new sensitive logging. No scope/auth changes. No AGENTS.md human-only gate crossed.

## 6. Build/tests

- `dotnet build -c Release`: 0 warnings, 0 errors.
- `dotnet test -c Release` (full suite, Testcontainers/Postgres): **60/60 passed**.
- `gh pr checks 2`: `test` pass, `docker` pass, `deploy` skipped (expected on a non-default branch).

## Findings

1. **[Nit]** `LEARNING_LOG.md` does not exist anywhere in the Garbati.OrchestratorApi repo (checked `main` and the PR branch), despite AGENTS.md's "Done means" checklist requiring one. The PR description contains a full learning-log entry as text but it was never committed as a real file. Pre-existing gap (PR #1/T-022 didn't create it either) — not introduced by this PR, but should be closed at some point.
2. **[Informational, correctly out of scope]** `IdentityApiClient.RequestAuthorizationAsync` still only special-cases HTTP 422; any other non-success status still falls through to a generic 500 with no detail. This fix removes the one 400 this task was about, but the opaque-500 pattern itself remains for other malformed requests.
3. **[Informational]** `docs/DESIGN.md` wasn't updated — reasonable, since the broken literal was never itself documented there; only the `MandateOperation` vocabulary it now correctly matches was.

No Blockers, no Should-fix items.

## What QA could not check

- Did not re-run this against a live deployed/local IdentityApi instance (source-level confirmation only, not a live HTTP round-trip with `"Debit"`).
- Did not check CI history for flaky prior runs beyond the current snapshot.

## Interview Q&A

1. **Why "Debit" instead of adding "Transfer" as a new accepted value in IdentityApi?** IdentityApi's mandate model already has a settled vocabulary for exactly this question ("can this requester take money out of this account?"). Adding "Transfer" would mean changing another service's ratified contract for something it already has a correct word for.
2. **How do you know the test would have caught the original bug?** QA reverted the fix locally and reran — the new test failed with the expected mismatch; restoring the fix brought the suite back to 60/60. Proven, not assumed.
3. **Does this change anything about the credit (destination) account's mandate check?** No — IdentityApi only checks a mandate for `DebitAccountId` against `Operation`; `CreditAccountId` is just the destination and isn't checked against any mandate. Unchanged by this fix.
4. **Is the 400→500 masking still a live risk?** In narrower form, yes — `"Debit"` always passes now, so this exact 400 can't recur, but other malformed requests to the same endpoint would still surface as an opaque 500. Correctly left out of this one-line fix.
5. **Why no DESIGN.md update?** The broken literal itself was never documented there — only the vocabulary it now correctly uses was already documented and unchanged.
