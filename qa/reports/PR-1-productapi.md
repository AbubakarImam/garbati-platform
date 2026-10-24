# QA Report — Garbati.ProductApi PR #1

**Branch:** `feature/productapi-foundation` → `main`
**Task:** T-045 (ProductApi foundation: project skeleton, 5 entities, 5 endpoints, migrations, tests)
**Reviewer:** QA agent, 2026-10-24 (second attempt — a first attempt stalled with no output and is discarded)

## Verdict: Approve

This is a clean, scope-disciplined foundation PR. It does what the task file asked, nothing more. The
single most important boundary (`FeeRule`/`ProfitSharingTerms` staying genuinely empty, with no code path
touching `ProfitSharingTerms` at all) is independently verified by grep, by reading every relevant file,
and by running the real test suite against real Testcontainers Postgres. Build and tests are clean
(58/58, matching the PR's own claim). No blockers found.

## 1. Task conformance

- All 5 entities, 2 migrations, 5 endpoints, 3 scopes, health checks, correlation id, audit trail match
  `docs/DESIGN.md` and the task file exactly.
- Scaffold (`global.json`, `dotnet-tools.json`, `.gitignore`, `.dockerignore`, `.sln`, `Dockerfile`,
  `ci.yml`) is byte-identical to `Garbati.StaffIdentityApi`'s, adjusted only for the project name —
  confirmed by diff.
- `ci.yml` has `test`+`docker` jobs only, no `deploy` job — confirmed, and matches the task's explicit
  instruction.
- Nothing extra found: no `OrchestratorApi` HTTP client, no real fee/profit-sharing content, no deploy
  wiring.

## 2. DESIGN.md / ARCHITECTURE.md fidelity

- No "Proposed" item was built on. `docs/DESIGN.md` for this service is itself ratified (Section 0,
  owner decision 23 Oct 2026), and this PR implements exactly what it specifies, including Appendix A's
  own judgment-call log, which matches the actual code.
- One tension worth flagging to the owner, not a defect in this PR: `ARCHITECTURE.md` Section 7's
  "Deployment guidance" suggests starting Product and Orchestrator as one deployable and splitting later
  "if there is a reason." This PR builds ProductApi as a fully separate repo/service with its own
  Dockerfile/CI — but that decision (separate repo) was made before this task, so it's not something this
  PR introduced or could have changed.

## 3. The single most important thing: FeeRule / ProfitSharingTerms

Verified directly, not taken on faith:

- `grep -rn "ProfitSharingTerms" src/ tests/` — every hit is either the `DbSet` declaration (required for
  EF to create the table), the entity definition, or a comment/test explaining it's untouched. **Zero
  reads or writes anywhere.**
- `grep -rn "FeeRule" src/ tests/` — `FeeRule` **is** queried by `FeeQuoteService.GetQuoteAsync` (the
  lookup), but never written to by any migration, any application code path, or any test.
- `FeeQuoteService.cs`: the `NoFeeConfigured` branch is the only reachable branch today; the
  `FeeApplied`/"real row found" branch is real code but provably unreachable (no seed, no test row) and
  is explicitly commented as such.
- `MigrationTests.FeeRulesTable_IsProvablyEmpty_AfterMigration` and
  `ProfitSharingTermsTable_IsProvablyEmpty_AfterMigration` run a real `SELECT count(*)` against a real
  migrated Postgres container (via Testcontainers) — run independently, both pass, both assert `0`.
- `FeeQuotesControllerTests.FeeQuote_AgainstARealSeededProduct_ReturnsNoFeeConfigured` hits the real HTTP
  endpoint against a real seeded product (`NG-SAV-STD`/`NG-CUR-STD` from the seed migration), through the
  real app pipeline (`WebApplicationFactory`), not a mock. The response shape
  (`feeResult: "NoFeeConfigured"`, `feeAmount: null`, `feeType: null`) is asserted directly, not just the
  200 status.

**This boundary is real and provably enforced, not just claimed in the PR description.**

## 4. The claimed idempotency-key-ordering bug fix

Verified by reading the actual code (`AccountProductBindingService.CreateAsync`) and the actual test:

- The catch block for `DbUpdateException` with Postgres SQL state `23505` checks for an existing
  `(CreatedByClientId, IdempotencyKey)` match **first**, before consulting `pgEx.ConstraintName`, exactly
  as claimed.
- `CreateAccountBinding_RetryWithTheSameKeyAndBody_ReturnsTheSameBinding` exercises exactly the scenario
  described (same key + same account number, which collides with both unique indexes simultaneously) and
  passes.
- Run independently (not just trusted the PR description) — passed, along with the sibling
  `CreateAccountBinding_SameKeyDifferentBody_Returns409` and `CreateAccountBinding_UnknownProductCode_Returns404`.
- Gap: the third branch this logic creates — a **different** idempotency key colliding on `AccountNumber`
  alone (→ `AccountAlreadyBoundException` → 409) — has no test anywhere. This is real, non-trivial logic
  (depends on the hardcoded constraint-name string `ix_account_product_bindings_account_number` matching
  what Postgres actually reports), and it's untested. See Should-fix #1.

## 5. jsonb pattern fidelity (ProductLimitPolicy vs KycTierPolicy)

Compared directly, side by side:
- `IdentityDbContext`'s `stringListConverter`/`stringListComparer` for
  `KycTierPolicy.RequiredEvidence`/`EligibleOperations` and `ProductDbContext`'s equivalent for
  `ProductLimitPolicy.AllowedOperations` are **character-for-character identical** (same
  `JsonSerializer.Serialize`/`Deserialize` calls, same `ValueComparer` hash/equality logic, same
  `HasColumnType("jsonb")`).
- This is a genuine copy, not a reinvention, as the task required.

## 6. Scope fidelity

- `.csproj` package list matches `Garbati.StaffIdentityApi`'s exactly (same 6 packages, same pinned
  versions) minus the one package StaffIdentityApi uses for password hashing (correctly omitted, since
  ProductApi has no password concept). No FluentValidation or other new NuGet dependency — validators are
  hand-rolled, matching platform convention.
- No `OrchestratorApi` wiring: grepped `src/` for `OrchestratorApi`/`LedgerApi`/`IdentityApi` — the only
  hit is a Swagger description string, not a client/HttpClient.
- No deploy CI job (confirmed by reading `ci.yml` directly — ends with a comment explaining why, matching
  sibling precedent).
- No real fee/profit-sharing content anywhere (Section 3 above).

## 7. Seed migration comment parity

Read both side by side:
- `Garbati.IdentityApi`'s `SeedPlaceholderKycTierPolicies.cs`: "PLACEHOLDER DATA — not real Nigerian KYC
  tier limits... replace them with confirmed values before relying on this table for anything real."
- `Garbati.ProductApi`'s `SeedPlaceholderProductLimitPolicies.cs`: "PLACEHOLDER DATA — not real, confirmed
  Nigerian minimum-balance/allowed-operations figures... Replace these with confirmed values before
  relying on this table for anything real," plus an extra paragraph explicitly contrasting itself with
  the deliberate absence of any `FeeRule`/`ProfitSharingTerms` seed.
- Same spirit, same plainness, slightly more thorough (the extra contrast paragraph is a good addition,
  not required but welcome).

## 8. Money safety / audit

- `MinimumBalance`, `FeeRule.Amount` use `HasPrecision(18,2)`; profit-share ratios use `HasPrecision(5,4)`
  — reasonable decimal precision choices (not exercised by any real money movement in this service, since
  it only computes, never posts, per `ARCHITECTURE.md` 4.4).
- Audit rows (`ProductAuditLog`) are added to the same `DbContext` and saved via the same
  `SaveChangesAsync` call as the entity change — same transaction, as required. Verified in both
  `ProductService.CreateAsync` and `AccountProductBindingService.CreateAsync`.
- `ProductService.CreateAsync`'s catch clause is `catch (DbUpdateException)` with no Postgres-error-code
  guard, unlike `AccountProductBindingService`'s more precise `when (ex.InnerException is
  PostgresException { SqlState: "23505" })`. See Should-fix #2.

## 9. Security

- API key comparison uses `CryptographicOperations.FixedTimeEquals` (timing-safe), hash-only
  configuration (SHA-256, never plaintext), matching every sibling service exactly.
- `appsettings.Development.json` contains only a hashed dev key and default local Postgres credentials —
  no real secret. It's correctly excluded from the Docker build context (`.dockerignore`) while still
  committed to git for local dev convenience, matching the task's explicit note that this is deliberate.
- Logs never include request bodies or identity numbers — the one custom log enrichment
  (`IdempotencyKeyLoggingFilter`) logs only the idempotency key, not the body, matching sibling pattern.
- Scope enforcement (401/403) is table-driven tested across every endpoint (`AuthorizationTests`), plus a
  reflection-based guard (`ControllerAuthorizationTests.EveryControllerAction_RequiresAKnownScopePolicy`)
  that fails if any future endpoint is added without a recognized scope policy.

## 10. Tests — build and run results

Run independently, bounded, with Docker/Testcontainers available:
- `dotnet restore` / `dotnet build -c Release --no-restore` — clean, 0 warnings, 0 errors.
- `dotnet test -c Release --no-build` — **58/58 passed**, ~5 seconds, against a real Testcontainers
  Postgres container and the real app via `WebApplicationFactory`. Matches the PR description's claimed
  count exactly.
- GitHub Actions status on the PR: both `test` and `docker` checks report `SUCCESS`.
- No evidence of any weakened or skipped test; no `[Skip]`/`Theory`-with-reduced-cases trickery found.

## Findings

**Blockers:** none.

**Should-fix:**

1. No test exercises the `AccountAlreadyBoundException` path (a *different* idempotency key attempting to
   bind an *already-bound* account to a different product). This is real, non-obvious logic — it depends
   on a hardcoded Postgres constraint-name string (`ix_account_product_bindings_account_number`) matching
   what Postgres actually reports — and currently has zero coverage. If that constant ever drifts from
   the real generated name (e.g., after a future migration regenerates the index), this would silently
   fall through to the bare `throw;` and surface as a 500 instead of 409, with nothing in the test suite
   to catch it.
2. `ProductService.CreateAsync`'s `catch (DbUpdateException)` has no `SqlState`/`PostgresException` guard,
   unlike `AccountProductBindingService`'s more precise equivalent. Any `DbUpdateException` that isn't
   actually the unique-constraint collision (e.g., a transient connection failure, or a future
   check-constraint violation) would be misreported as `DuplicateProductCodeException` → 409, rather than
   the generic 500 it should be. Low real-world likelihood today (inputs are pre-validated), but it's an
   inconsistency in defensive coding between the two services worth tightening.

**Nits:**

1. `ARCHITECTURE.md` Section 7's deployment guidance (start Product+Orchestrator as one deployable, split
   later "if there is a reason") was effectively already overridden by the repo being created standalone
   before this task started. Not a defect in this PR — flagging only so the owner is aware of the tension
   if it's ever revisited.
2. The README's "Appendix A" style documentation of judgment calls is thorough and a genuinely good
   practice; no action needed, calling it out as a positive.

## What I could not check

- Behavior against a real deployed Azure Postgres instance — only verified locally via Testcontainers,
  same limitation the PR author notes.
- Whether `OrchestratorApi` can actually integrate against this contract end-to-end — explicitly out of
  scope for this task, nothing to check yet.
- Long-term stability of the hardcoded Postgres constraint-name strings against a future EF Core/Npgsql
  version upgrade that might change index-naming behavior — theoretical, not something exercisable
  without actually triggering such an upgrade.

---

## Follow-up re-review — 2026-10-24 (after 3 fix commits: `9b1abd7`, `7ac7bc3`-adjacent docs, `7ebab28`)

**Scope:** Focused re-review only. Confirmed local `HEAD` (`7ebab28`) matches the PR's actual head commit
on GitHub before reviewing, so this reflects the real current state of PR #1, not a stale clone. Did not
re-review anything already approved above.

### Verdict: Approve

Both Should-fix items are genuinely fixed, with sound, honest tests — not tests that happen to pass for
the wrong reason. The unplanned flaky-test fix is a real root-cause fix, correctly scoped. Nothing
outside the stated scope changed. No new blockers.

### 1. Should-fix #1 (AccountAlreadyBoundException coverage) — verified fixed

Read `CreateAccountBinding_DifferentKey_SameAccountAlreadyBoundToAnotherProduct_Returns409` in
`AccountBindingsControllerTests.cs` directly:

- Binds account to product A under key 1, then attempts to bind the **same account number** to **product
  B** under a **brand-new key** (not a replay) — exactly the gap called out previously.
- Asserts the real `409`, `application/problem+json` content type, the exact title string ("This account
  is already bound to a product."), and that `detail` contains the account number.
- Also re-reads the binding row from the database afterward and asserts it's still bound to product A —
  confirming the rejected second call didn't mutate state. This is a stronger assertion than the original
  gap required and rules out a false pass (e.g., a bug that silently rebinds but still happens to return
  409 for an unrelated reason).
- Ran this test by itself against the real Testcontainers Postgres: passed, and the logged exception trace
  confirms it actually went through `AccountProductBindingService.CreateAsync`'s
  `pgEx.ConstraintName == AccountNumberConstraintName` branch → `AccountAlreadyBoundException` → the real
  exception-handling middleware → 409. Not a coincidental pass.

### 2. Should-fix #2 (DbUpdateException guard) — verified fixed

- `ProductService.CreateAsync`'s catch clause now reads
  `catch (DbUpdateException ex) when (ex.InnerException is PostgresException { SqlState: "23505" })` —
  character-for-character the same pattern as `AccountProductBindingService.CreateAsync`'s guard (modulo
  the latter also capturing `pgEx` by name, which `ProductService` doesn't need since it only reports a
  fixed message either way). Confirmed by direct read, not trusted from the diff summary.
- New `ProductServiceTests.cs` proves the negative case soundly: a `ThrowingOnSaveProductDbContext`
  subclass overrides the real (virtual) `SaveChangesAsync` to always throw a `DbUpdateException` whose
  `InnerException` is deliberately an `InvalidOperationException`, not a `PostgresException` — so it
  cannot possibly match the `SqlState: "23505"` guard. The test asserts the thrown exception is a
  `DbUpdateException` and explicitly `IsNotType<DuplicateProductCodeException>`. This is a real
  inheritance-based test double against the actual `ProductDbContext`/`SaveChangesAsync` (which is
  virtual on the EF Core base class), not a brittle mock of an interface that could drift from real
  behavior — a sound technique for a case that isn't practical to trigger against live Postgres on
  demand. Ran it standalone: passed.

### 3. Unplanned flaky-test fix (`TruncateToMicroseconds`) — verified as a real root-cause fix, correctly scoped

- The mismatch is real: Postgres `timestamp with time zone` stores microsecond precision (6 fractional
  digits) while a .NET `DateTimeOffset` tick is 100ns (7 fractional digits) — so an in-memory value
  created with `DateTimeOffset.UtcNow` and never re-read from the database can legitimately differ from
  the same value read back after a round trip, by up to a few hundred nanoseconds. This is a genuine
  precision-mismatch bug class, not a timing/ordering flake.
- The fix truncates `BoundAt` **once, at the point of construction**, before it's either saved or
  returned — so the in-memory value and the persisted value are identical from the start, rather than
  retrying the comparison or widening an assertion's tolerance. This is a correct fix at the actual root
  cause.
- Scope check: grepped all `DateTimeOffset` fields across every entity
  (`AccountProductBinding.BoundAt`, `ProductAuditLog.CreatedAt`, `Product.CreatedAt`/`EffectiveFrom`/
  `EffectiveTo`, `FeeRule`/`ProfitSharingTerms`/`ProductLimitPolicy`'s `EffectiveFrom`/`EffectiveTo`).
  `TruncateToMicroseconds` is a `private static` method inside `AccountProductBindingService` and is only
  ever called once, on `BoundAt`, at binding creation — it is not shared or applied to any other field or
  entity. Confirmed no other service/entity has an in-memory-vs-reread comparison test that this same
  class of bug could affect today (`ProductService.CreateAsync` constructs `CreatedAt`/`EffectiveFrom` in
  memory too, but no test currently round-trips and compares those values the way the binding replay test
  does) — so this is a correctly minimal fix for the one real, currently-exercised case, not a
  speculative fix applied everywhere. Worth a Nit for the owner's awareness, not a blocker.

### 4. Test run — verified directly, not just trusted

- `dotnet build -c Release`: clean, 0 warnings/errors.
- `dotnet test -c Release --no-build`, run **twice independently** against real Testcontainers Postgres:
  **60/60 passed both times** (~4s each), matching the claimed count exactly.
- Ran the three new/changed tests in isolation with verbose output
  (`CreateAccountBinding_DifferentKey_SameAccountAlreadyBoundToAnotherProduct_Returns409`,
  `ProductServiceTests.CreateAsync_ADbUpdateExceptionThatIsNotAUniqueViolation_...`,
  `CreateAccountBinding_RetryWithTheSameKeyAndBody_ReturnsTheSameBinding`) — all three passed, and the
  captured log for the first shows the real exception traversing the real service → middleware pipeline,
  not a stub.
- GitHub Actions on the live PR (head commit `7ebab28`, confirmed to match local `HEAD`): both `test` and
  `docker` jobs report `pass`.
- Did not attempt to force a genuinely flaky repeat-until-fails loop beyond this (e.g., hundreds of runs)
  — bounded by the task's own guidance to avoid open-ended retries. Two clean runs plus a green CI run is
  treated as sufficient corroboration, not proof the flake can never recur under different CI host timing;
  noted under "could not check."

### 5. Diff scope — confirmed clean

`git diff --stat` across all three new commits combined: exactly 6 files touched —
`docs/DESIGN.md`, `ProductService.cs`, `AccountProductBindingService.cs`,
`AccountBindingsControllerTests.cs`, `ApiFixture.cs` (a `CreateOptions()` accessor extracted so the new
service-level test can build a `ProductDbContext` subclass against the same real database — a reasonable,
minimal refactor, not scope creep), and the new `ProductServiceTests.cs`. Nothing else changed. No new
dependency, no new `OrchestratorApi` wiring, no unrelated refactors.

### Findings (follow-up)

**Blockers:** none.

**Should-fix:** none remaining — both original items are closed.

**Nits:**

1. `TruncateToMicroseconds` is currently private to `AccountProductBindingService` and only applied to
   `BoundAt`. If a future test ever round-trips and compares `Product.CreatedAt`/`EffectiveFrom` (or any
   other entity's `DateTimeOffset` field) the same way the binding replay test does, it could hit the
   identical latent precision mismatch. Not a defect today — no such test exists — but worth the owner
   knowing this is a per-field patch, not a platform-wide guard (e.g., a `SaveChanges` interceptor), in
   case the pattern needs repeating elsewhere later.

### What I could not check

- Whether the flaky test could still recur under different CI runner timing/load than what was observed —
  two local runs plus one green CI run is reasonable corroboration within this review's time budget, not
  an exhaustive stress test.
- Did not re-review anything outside the three new commits (by design, per the scope of this follow-up).
