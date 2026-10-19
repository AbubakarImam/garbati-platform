# QA Report — Garbati.OrchestratorApi PR #3 (T-025)

**Verdict: Approve**

## 1. Task fidelity — PASS

Exactly three files touched, matching scope: `AccountOpeningService.cs` (+11/-5), `AccountOpeningServiceTests.cs` (+29/-0), `docs/DESIGN.md` (+18/-1, new Appendix B). `DemoKycTier` changed `"Tier1"` → `"Tier2"`, comment rewritten. Grepped the whole repo for `Tier1`/`Tier2`/`Tier3`/`KycTier` — this was the only hardcoded tier value anywhere; nothing else missed.

## 2. Independent verification against IdentityApi — PASS

Read IdentityApi's real seed migration and `MandateEligibility.cs` directly (not trusting the PR's narrative): `Tier1 → ["View"]`, `Tier2`/`Tier3` → `["View","Credit","Debit"]` on `"NG"`. Also checked the *second* gate `MandateEligibility` applies (mandate permissions, not just tier) — `AttachMandateAsync` already requests `["View","Debit","Credit"]` unconditionally, so the tier-eligibility gate was the sole blocker, and `"Tier2"` genuinely clears it.

## 3. Scope — PASS, no creep

No edits to IdentityApi, its seed migration, or `MandateEligibility.cs`. Decision #6 updated in place rather than assigned a new number, matching the task's framing (placeholder-value pick, not a new policy decision).

## 4. Other Tier1-style assumptions — none found

Only non-"NG" risk (arbitrary jurisdiction codes) was already pre-existing/flagged in DESIGN.md note #5, correctly out of scope here.

## 5. Test quality — PASS, empirically proven

`StartAsync_RecordsKycDecision_WithATierThatAllowsDebitAndCredit` captures the real `KycDecisionRequest` via the existing fixture hook and asserts `KycTier == "Tier2"` (value assertion). QA reverted the constant to `"Tier1"` locally and reran: test failed as expected (`Expected: Tier2, Actual: Tier1`). Restored, reran: 61/61 passed.

## 6. Build/tests/CI

- `dotnet build -c Release`: 0 warnings, 0 errors.
- `dotnet test -c Release`: 61/61 passed.
- `gh pr checks 3`: `test` pass, `docker` pass, `deploy` skipping (expected, non-default branch).

## 7. Money safety / security

No ledger/money code, no auth-scope change, no AGENTS.md human-only gate crossed. Only selects a different already-existing placeholder value in the Orchestrator's own self-contained demo stub.

## Findings

1. **[Nit]** `LEARNING_LOG.md` still doesn't exist in this repo — pre-existing gap, flagged again (also noted in PR #2's QA report).
2. **[Nit]** Live re-verification against real deployed Azure infra hasn't happened yet at merge time (can only happen post-deploy) — correctly disclosed in the PR description as the next step.
3. **[Informational]** The fix only needed to touch the tier-eligibility gate because the mandate-permission gate (`AttachMandateAsync`) already grants `Debit`/`Credit` unconditionally. If that attach step is ever narrowed, this class of bug could resurface from the other gate — not a defect now, just a note for future vigilance.

No Blockers, no Should-fix items.

## What QA could not check

- Live round-trip verification against the real deployed Azure IdentityApi instance post-merge.
- CI history for flakiness beyond this snapshot.

## Interview Q&A

1. **Why Tier2 instead of adding Debit/Credit to Tier1 in IdentityApi?** Tier1=View-only is a deliberate, reasonable placeholder; changing it would mean editing another service's policy data outside this task. Simpler and correct to fix which tier Orchestrator's own stub picks.
2. **How do we know Tier2 actually unlocks Debit, not just tier eligibility?** Two gates exist (tier eligibility + mandate permission); the mandate-permission gate already grants Debit/Credit unconditionally via `AttachMandateAsync`, so only the tier gate was blocking. Confirmed by reading both repos' source directly.
3. **How was the test proven to catch the original bug?** Reverted to `"Tier1"` locally, reran — failed with the expected mismatch; restored, reran — 61/61 clean.
4. **Does this need owner sign-off per AGENTS.md's human-only gates?** No — no fee/limit/reversal/migration/contract/auth-scope change; only a placeholder constant in a self-contained demo stub.
5. **Does this require any IdentityApi change?** No — IdentityApi's seed data and `MandateEligibility.cs` were already correct and independently re-confirmed; the mismatch was entirely in Orchestrator's own stub choice.
