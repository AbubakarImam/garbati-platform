# QA Report — Garbati.IdentityApi PR #7 (T-044: BVN/NIN storage + narrow KYC-tier evidence enforcement)

**Verdict: Approve with nits** (no blockers; one Should-fix worth the owner's attention, one process item worth a clear answer either way)

Branch `feature/bvn-nin-storage-and-tier-enforcement` reviewed locally at `/Users/USER/Projects/Garbati.IdentityApi` (commit `426551b`). CI (`test`, `docker`) both green on the PR. I independently ran the full suite locally (see §6) and re-derived every claim in the PR description by reading the actual diff, not the summary.

## 1. Scope fidelity against the task file — PASS

`tasks/T-044-identityapi-bvn-nin-storage-and-tier-enforcement.md` asked for exactly three things; the diff (28 files, +2787/-19) delivers exactly those three and nothing else:

- `Bvn`/`Nin` nullable columns on `CustomerIdentityData` (`src/.../Entities/CustomerIdentityData.cs`), additive migration only (`20261023171514_AddBvnNinToCustomerIdentityData.cs` — pure `AddColumn`, no data touched).
- `PATCH /v1/customers/{customerId}/identity-numbers`, scope `identity.kyc.manage` (matches task exactly — not `identity.customer.write`), set-once semantics, 409 on conflicting resubmission, idempotent no-op on same-value resubmission, `CustomerResponse` unchanged shape.
- `AtLeastOneIdentityNumber`/`BothIdentityNumbers` evidence tokens wired into `RecordKycDecisionAsync` only when granting a tier, via a **second, separate** migration (`20261023171538_AddBvnNinCardinalityToKycTierPolicies.cs`) that `UPDATE`s the three existing placeholder `KycTierPolicy` rows rather than editing the old seed migration — exactly as instructed.

Nothing extra snuck in. `UpdateCustomerDetailsRequest`/`UpdateDetailsAsync` (T-035) is untouched (confirmed via `git diff` — zero lines changed in that method). No encryption/KMS code added. No external BVN/NIN verification call added. `close`/`mark-deceased` endpoints predate this PR (T-017) and are unrelated. The only "extra" items are a `docs/DESIGN.md` Appendix H write-up and updates to Sections 2.1/5.1/6.3/7.1/#TBD-4 — all documentation, exactly what "Done means" (`AGENTS.md`) requires.

## 2. Design/architecture conformance — PASS, with one process flag (see §4)

- `docs/DESIGN.md` Appendix H and the `#TBD-4` resolution read as an accurate record of what was actually built (cross-checked against the code, not just trusted).
- Nothing here is built on a "Proposed" item. `KycTierPolicy.RequiredEvidence` being "configurable, not hardcoded" is decision #11, Confirmed. The scope's own endpoint/scope choice (`identity.kyc.manage`, not a new scope) matches DESIGN.md Section 5.2's existing scope list — no new scope invented.
- `ARCHITECTURE.md` Section 5's error-code convention (400/401/403/404/409/422/500) is followed precisely: `IdentityNumberAlreadySetException` → 409, `KycTierEvidenceNotMetException` → 422 (`ExceptionHandlingMiddleware.cs`).

## 3. Set-once endpoint — verified directly, not trusted

Read `CustomerService.SubmitIdentityNumbersAsync` directly (not the PR summary):

```csharp
if (request.Bvn is not null)
{
    if (identityData.Bvn is null) { identityData.Bvn = request.Bvn; changes.Add("Bvn set"); }
    else if (identityData.Bvn != request.Bvn)
        throw new IdentityNumberAlreadySetException("Bvn is already set for this customer and cannot be changed via this endpoint.");
    // else: same value resubmitted - no-op
}
```

Confirmed behaviour matches spec: null→value sets it; same value is a no-op (no audit row, no `SaveChangesAsync` side effect beyond the no-op branch's early return); different value throws before anything is mutated on the tracked entity.

I read (not just trusted) `CustomerServiceTests.SubmitIdentityNumbersAsync_Throws409_WhenADifferentBvnIsSubmitted_AndLeavesTheStoredValueUnchanged`: it submits `"12345678901"` in one `DbContext`, then in a **second, independent** `DbContext` attempts `"99999999999"`, asserts the exception is thrown, then opens a **third, fresh** `DbContext` and queries `CustomerIdentityData` directly, asserting `Bvn == "12345678901"` — i.e. it genuinely re-reads from Postgres (via Testcontainers, `PostgresFixture` — real database, not EF InMemory) rather than re-using the same tracked entity. This is exactly the "query the database directly, don't just trust the response code" verification the task demanded, and I confirmed it actually does that, not merely claims to. Same pattern for the no-op case (`SubmitIdentityNumbersAsync_ResubmittingTheSameValue_IsAnIdempotentNoOp_AndWritesNoAuditRow` — asserts audit row count is exactly 1 after two submissions of the same value, queried fresh).

**Gap found (Should fix, not a blocker):** the set-once check is pure application-level read-then-write with no database-level guard (no unique/check constraint, no optimistic-concurrency token on `CustomerIdentityData`, no `SELECT ... FOR UPDATE`/serializable isolation). Two concurrent `PATCH` requests for the *same customer* with *different* BVN values can both read `Bvn == null`, both pass the "stored value is null" branch, and both successfully `SaveChangesAsync` — the second silently overwrites the first with **no 409** to either caller. This is exactly the failure mode the task file itself names as the motivation for building this endpoint ("a channel that lets someone silently swap a stored BVN/NIN after the fact is a real fraud vector"), and it is not covered by any test (all existing tests are sequential: submit, await, then submit again). This codebase already has precedent for guarding exactly this class of invariant with a real DB constraint rather than application logic alone — `DESIGN.md` 7.4 / decision #19 explicitly requires a database unique constraint for the one-active-mandate-per-pair rule "not application-level validation alone... under concurrent requests." The same discipline was not applied here. Recommend either a DB check constraint pattern (not straightforward for "set once" semantics) or, more practically, a short advisory lock / `SELECT ... FOR UPDATE` on the customer row inside this method, plus a concurrency test.

## 4. Logging/leakage — PASS, verified by direct grep, not by trusting the PR summary

I grepped the entire diff for the literal test BVN/NIN values (`12345678901`, `10987654321`, `99999999999`) outside `tests/` — zero hits in `src/`. I separately grepped all `Log*` calls in `src/` for BVN/NIN-adjacent terms — the only logging touching this code path is `ExceptionHandlingMiddleware`'s generic `logger.LogWarning(ex, ...)`/`LogError`, which logs `ex` (whose `.Message` only ever names the *field* — "Bvn is already set..." — never the value) plus `Method`/`Path`. `AuditEvent.Details` for `IdentityNumberSubmitted` is built from a `changes` list containing only the literal strings `"Bvn set"`/`"Nin set"` — confirmed by reading `SubmitIdentityNumbersAsync` line-by-line; the submitted value never touches that string. The response DTO (`CustomerResponse`) is unchanged — confirmed via `git diff` (zero lines changed in `ToResponse`/`CustomerResponse.cs`).

Three independent tests back this up and I read each to confirm it asserts what it claims:
- `SubmitIdentityNumbersAsync_RecordsAnAuditRow_ThatDoesNotContainTheSubmittedValue` — queries the real `AuditEvent` row and asserts `Assert.DoesNotContain(submittedBvn, audit.Details!)` — a direct content assertion, not "no exception thrown."
- `SubmitIdentityNumbers_ResponseJson_HasNoBvnOrNinKeys` (`IdentityNumbersHttpTests`) — a real HTTP round trip, parses the live JSON response, asserts no `bvn`/`nin` key (case-insensitive) — not relying on `CustomerResponse` having no such property as the only proof.
- `CustomerResponse` reflection test extended with `Assert.DoesNotContain("Bvn", propertyNames)` / `"Nin"` — compile-time-adjacent guarantee with a test that would actually fail if someone added the property back.

Pre-existing Serilog request-logging middleware (`Program.cs`) was not modified by this PR and does not log request bodies — confirmed unaffected.

## 5. Enforcement scope — confirmed exactly as narrow as required

`grep` for `GovernmentId`/`SourceOfFunds`/`SelfDeclaredIdentity` across `src/` (excluding migrations) returns **only comments** explaining why they're deliberately untouched — no code path reads or checks them. `EnsureIdentityNumberEvidenceIsMetAsync` checks `policy.RequiredEvidence.Contains(...)` for exactly the two new tokens and nothing else. The new migration's `UPDATE` statements only `||`-append the new token onto the existing jsonb array (`required_evidence || '[...]'::jsonb`), never replacing it — confirmed the three placeholder rows keep `SelfDeclaredIdentity`/`GovernmentId`/`SourceOfFunds` (tested directly in `MigrationTests.CardinalityMigration_AddsBvnNinEvidenceTokens_WithoutRemovingExistingOnes`, which I read and confirms exactly this with real `Assert.Contains` calls against both old and new tokens). Tier2/Tier3 remain grantable after this PR — the two newly-added tokens are the only new blockers, and they're satisfiable by this same PR's own new endpoint. No risk of making tiers permanently ungrantable.

A no-matching-policy-row case is treated as "nothing to check" (returns early, no exception) — matches the task file's explicit instruction and the pre-existing (absence of) behavior.

## 6. Full test suite — ran independently, not trusted

Ran `dotnet test -c Release` myself against the branch (real Postgres via Testcontainers, confirmed a local Docker daemon was available):

```
Passed!  - Failed: 0, Passed: 353, Skipped: 0, Total: 353, Duration: 8 s
```

This independently confirms the PR description's "353/353" claim — I did not just trust it. CI's `test` and `docker` jobs are also green on the PR itself (`gh pr view 7`).

Tests I additionally read to confirm they test behaviour, not just "doesn't throw":
- `MigrationTests.AddBvnNinColumns_ExistingRowsGetNull_ForBothNewColumns_NoDefaultInvented` — inserts a row via raw SQL that never mentions the new columns (modelling a genuinely pre-migration row), then asserts both are `NULL` post-migration — a real "no default invented" proof.
- `RecordKycDecisionAsync_Throws422_AndRecordsNothing_AndLeavesStatusAndTierUnchanged` — re-queries the DB after the 422 and asserts `KycStatus == Pending`, `KycTier == null`, and zero `KycVerification` rows — the "whole operation fails atomically" claim is actually checked against the database, not inferred from the exception type alone.
- `RecordKycDecisionAsync_ANonVerifiedDecision_NeverTriggersTheEvidenceCheck_RegardlessOfWhatIsOnFile` — explicitly uses a customer with **no** identity numbers on file and a `Rejected` decision, proving the guard is the pre-existing `newStatus == Verified` check and nothing stricter.

Four pre-existing test files' setup helpers (`AuthorizationServiceTests`, `JointApprovalServiceTests`, `ReversalAuthorizationServiceTests`, one test in `CustomerServiceTests` itself) were changed to call `SubmitIdentityNumbersAsync` with a throwaway value before granting a tier. I read every one of these diffs: none weaken or remove an assertion — they only add a prerequisite call inside an `if (kycTier is not null)` setup block, so the tests' actual assertions are unchanged. This is a legitimate, necessary consequence of the new enforcement working, not a weakened test (`AGENTS.md`: "Do not weaken or delete a test to make it pass" — this PR doesn't; it fixed the setup instead).

## 7. Security / authorization scope — PASS

New endpoint correctly uses the existing `identity.kyc.manage` scope (no new scope invented — would otherwise have been a human-only-gate trigger under `AGENTS.md`'s "changes authentication or authorization"). Scope-boundary coverage confirmed via a new row in `AuthorizationTests.Endpoints` (verified it actually exercises 401/403/reaches-the-handler for this specific route+scope pair, not a generic catch-all).

## 8. The data-migration human-only-gate question — independent assessment, not a repeat of the builder's own reasoning

`AGENTS.md` lists, as a **human-only gate** ("stop and ask the owner"): "adds or edits a database migration on data that exists." The builder flagged, in the PR description, that `AddBvnNinCardinalityToKycTierPolicies` — which runs `UPDATE kyc_tier_policies SET required_evidence = ...` against the three already-existing placeholder rows — crosses this gate, but treated it as pre-authorized because the task file spelled out the exact migration content, and self-proceeded rather than stopping to ask.

My independent read: **this is a defensible but not clearly safe reading, and I'd have preferred the builder stop and ask rather than self-authorize.** Two things cut in the builder's favour: the task file is unusually explicit here (names the exact three rows, the exact two token strings, and even mandates *how* to structure the migration — "via a new migration that updates them, not by editing the old migration file" — which reads like a decision the owner already made when writing the spec, not something left to the builder's judgment), and the actual SQL is low-risk (idempotent — guarded by `NOT (required_evidence @> ...)` — additive-only within the jsonb array, reversible via a symmetric `Down()`, and scoped to `jurisdiction_code = 'NG'` placeholder rows explicitly documented elsewhere in `DESIGN.md` as "not real Nigerian tier limits"). But the gate is listed in `AGENTS.md` as a category unto itself, independent of how precisely a task file describes the change — if "the task file was specific enough" were sufficient self-authorization, the gate would rarely ever fire, since task files are routinely this specific about *what* to build. The gate's evident purpose is to put a second, deliberate human checkpoint in front of exactly this class of action — editing rows that already exist, in any environment where this service has already run its seed migration (including, potentially, a live demo/staging database with real tier-grant history) — precisely because that's a different risk profile from a purely additive schema change, even when the eventual SQL is this narrow. `AGENTS.md`'s own "Before you start" guidance is also explicit: "If it is unclear, stop and write your questions at the bottom of the task file. Do not guess" — and a listed human-only gate is a stronger signal than mere unclarity. The PR description's transparent flagging of this (rather than silently proceeding) is the right instinct and makes this easy for the owner to catch before merging, but the decision to proceed rather than pause was still the builder's own call on a gate that says, in terms, not to make that call. Net: I'd ask the owner to explicitly confirm, before merging, that running this specific `UPDATE` against any environment's existing `kyc_tier_policies` rows is something he's reviewed and wants — not because the content looks wrong, but because the gate exists specifically so that confirmation happens before the fact, not after a PR description notices it.

## Findings summary

**Blockers:** none.

**Should fix:**
1. (§3) The set-once endpoint has no database-level guard against a race between two concurrent `PATCH` requests for the same customer with different values — the second can silently overwrite the first with no 409 to either caller. Not covered by any test. Low likelihood in practice, but it's exactly the fraud scenario this endpoint exists to close, and the codebase has an established pattern (real DB constraint, per decision #19) for this exact class of invariant that wasn't applied here.

**Process item (not a code defect, but worth a clear owner decision):**
2. (§8) The second migration edits existing `KycTierPolicy` rows and is a named `AGENTS.md` human-only gate. The builder self-authorized based on the task file's specificity rather than stopping to ask. The content is low-risk and well-documented, but the self-authorization itself is the thing I'd flag — the owner should explicitly confirm he's fine with this precedent (task-file specificity substituting for an explicit stop-and-ask) before it becomes the default interpretation for future PRs touching this same gate.

**Nits:** none beyond the above — comments are thorough and accurate against the code they describe (unusually so), DESIGN.md Appendix H matches the implementation exactly, and I did not find any discrepancy between what the PR description claims and what the code/tests actually do.

## What I could not check

- Did not deploy to Azure or exercise the GitHub Actions `docker` job's exact environment beyond what the CI run already shows green; relied on the CI status and my own local `dotnet test` run.
- Did not independently re-verify `research/nigerian-regulatory-requirements.md` Q1's Tier1/Tier2/Tier3 cardinality claims against a primary CBN source — that research document itself flags this as secondary-source-only and "needs compliance confirmation," which is outside this PR's and this QA pass's scope.
- Did not attempt to actually trigger the concurrency race in §3 against a live instance (would require a purpose-built two-thread repro); the finding is based on reading the code path for a transaction/lock boundary and confirming none exists, not an observed failure.

---

## Follow-up review — 2026-10-23 (commit d6acf2e, same branch/PR #7)

**Verdict: Approve.** The §3 Should-fix is resolved. No new issues found.

The owner asked the builder to fix the §3 concurrency gap before merge. Builder's commit `d6acf2e` reports: a `SELECT ... FOR UPDATE` lock added to `SubmitIdentityNumbersAsync`, a new genuine-concurrency test, and 354/354 passing. I re-verified each claim directly rather than trusting the summary.

### 1. The lock — real, correctly placed, matches the claimed precedent

Read the current `SubmitIdentityNumbersAsync` directly. The method now opens an explicit transaction (`dbContext.Database.BeginTransactionAsync`), reads the customer `AsNoTracking` (never locked — correct, since `Customer` itself isn't mutated here), then loads `CustomerIdentityData` via:

```csharp
var identityData = await dbContext.CustomerIdentityData
    .FromSqlInterpolated($"SELECT * FROM customer_identity_data WHERE customer_id = {customer.Id} FOR UPDATE")
    .FirstAsync(cancellationToken);
```

This happens **before** the `request.Bvn`/`request.Nin` null-check branches that read `identityData.Bvn`/`identityData.Nin` — the lock is acquired before the read that matters, not after. `identityData` comes back tracked (not `AsNoTracking`), so the in-place mutations (`identityData.Bvn = request.Bvn`, etc.) further down are picked up by the later `SaveChangesAsync`, and the transaction is committed on every return path (the no-op branch now explicitly commits too, rather than relying on dispose-rollback) — the row stays locked for the method's full duration, released only at commit.

I read `JointApprovalService.ApproveAsync` directly and compared line-by-line rather than trusting the commit message's claim of a match. It is the same pattern, applied correctly to a different row:
- Same shape: `BeginTransactionAsync` → `FromSqlInterpolated("... FOR UPDATE")` → business logic → `SaveChangesAsync` → `CommitAsync` on every return path (including early-return branches that still must commit a prior write, e.g. `ApproveAsync`'s `ExpireIfPastDueAsync` branch; `SubmitIdentityNumbersAsync`'s no-op branch, symmetrically).
- Same reason given for avoiding `Include()`: both avoid composing a `FOR UPDATE` query through a nullable-side join (Postgres rejects `FOR UPDATE` on the outer-joined side of a `LEFT JOIN`), and both separate the locked row query from any related-data load.
- Correctly-reasoned divergence, not a copy-paste mismatch: `ApproveAsync` locks a row to serialize a many-approvers-racing-one-threshold decision; this fix locks a row to serialize a null-to-value transition. The PR's own comments explain why a unique constraint (the mandate-uniqueness pattern, decision #19) doesn't fit here — there's no cross-row uniqueness invariant, only one row's state transition — and that reasoning is sound.

### 2. The new test — genuinely concurrent, confirmed by code reading and by behaviour

Read `SubmitIdentityNumbersAsync_TwoConcurrentRequestsWithDifferentBvnValues_OnlyOneWins` directly. Both attempts are constructed via a lazy `Select` into `Task.Run(...)`, and the resulting `IEnumerable<Task<...>>` is passed straight to `Task.WhenAll` without any intermediate `.ToList()`/await — `Task.WhenAll`'s enumeration triggers each `Task.Run` call (which starts executing immediately) back-to-back before awaiting either, so both backgrounded tasks are genuinely in flight concurrently, not sequential-but-disguised. Each runs against its own `fixture.CreateContext()` — confirmed by reading `PostgresFixture.CreateContext()`: every call builds a fresh `DbContextOptions`/`IdentityDbContext`, which (via Npgsql's connection pool) is a distinct logical connection, not a shared `DbContext` being reused unsafely across threads. A third, independent context verifies the persisted winner afterward.

This matches the shape of the existing `CreateCustomerAsync_ConcurrentRequestsWithOneKeyAndDifferentBodies_OneSucceedsOneConflicts` test referenced in the comment, which I spot-checked for the same shape.

### 3. Proved the test actually catches the regression — the decisive check

Per instructions, I did not just read the code and trust it. I reverted only `CustomerService.cs` to its pre-fix state (`git checkout d6acf2e^ -- src/Garbati.IdentityApi/Services/Implementations/CustomerService.cs`), leaving the new test, the DESIGN.md update, and everything else from the PR intact, then ran the new test in isolation 5 times:

```
Failed Garbati.IdentityApi.Tests.Services.CustomerServiceTests.SubmitIdentityNumbersAsync_TwoConcurrentRequestsWithDifferentBvnValues_OnlyOneWins
  Error Message:
   Assert.Single() Failure: The collection contained 2 matching items
   Collection: [Tuple ("11111111111", "ok"), Tuple ("22222222222", "ok")]
```

Failed 5/5 runs, every time with **both** calls reporting `"ok"` — the exact silent-overwrite bug the fix targets (both requests read `Bvn == null`, both won, no 409 to either caller). This also independently confirms the test is genuinely concurrent and not secretly sequential: if the two calls ran one-after-the-other, the second would always see the first's already-set value and correctly receive `IdentityNumberAlreadySetException` ("ok"/"conflict") even without the lock — a sequential test could never reproduce "both won." Getting "both won" 5/5 times is only possible if both transactions genuinely read the pre-write state concurrently.

I then restored the fix (`git checkout HEAD -- src/.../CustomerService.cs`), confirmed `git diff` was clean, and re-ran the test 5 times: passed 5/5. This is the strongest possible evidence the test is a real regression guard, not a vacuous one.

### 4. Full suite — ran independently

```
Passed!  - Failed: 0, Passed: 354, Skipped: 0, Total: 354, Duration: 8-9 s
```

Matches the PR's claim of 354/354 exactly (353 prior + 1 new). I ran this both before and after the revert/restore experiment above, both green.

### 5. Scope — confirmed nothing else changed

`git show d6acf2e --stat` touches exactly three files: `docs/DESIGN.md` (documentation only — the new "QA fix" paragraphs in Appendix H, read in full, accurately describe the change and nothing else), `CustomerService.cs` (the fix itself), and `CustomerServiceTests.cs` (the new test). `JointApprovalService.cs` is untouched by this commit (read-only comparison target, as instructed). The §8 migration-human-gate item from the first review is a different file/commit entirely and is unaffected by this follow-up commit.

### What I could not check

- Did not test behaviour under real network-level concurrency (e.g. two actual concurrent HTTP requests through Kestrel) — only the two unit-level paths (`Task.Run` racing a real Postgres instance via Testcontainers, as the test does, and the direct revert/restore experiment above). This is consistent with how `ApproveAsync`'s own existing concurrency coverage is tested and is the project's established pattern.
- Did not check behaviour under Postgres `deadlock_timeout`/statement_timeout configuration differences between this dev Testcontainers image and whatever production Postgres config is - the lock-wait semantics should be identical, but I didn't independently confirm production timeout settings wouldn't turn a lock-wait into a timeout error under higher real-world latency/connection-pool pressure.
- Did not re-run the rest of the first review's checks (§1, §2, §4-§7) — out of scope for this focused follow-up, per instructions.
