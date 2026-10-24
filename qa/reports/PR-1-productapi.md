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
