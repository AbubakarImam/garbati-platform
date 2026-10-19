# T-025 — OrchestratorApi: demo account-opening KYC stub assigns a tier that can never transfer

## Found by

Live end-to-end verification against real deployed Azure infrastructure, 18 Oct 2026, immediately after
T-024 fixed the `"Transfer"` → `"Debit"` bug. With that fix in place, a real transfer call now gets a
correctly-formed response from IdentityApi instead of a masked 500 — but it is a denial:

```
422 {"title":"Authorization denied.","detail":"... OperationNotEligibleForKycTier.","reasonCode":"OperationNotEligibleForKycTier"}
```

## The cause

`AccountOpeningService.cs` line 41: `private const string DemoKycTier = "Tier1";` — every account opened
through the Orchestrator's demo flow is auto-KYC-approved at `"Tier1"`.

IdentityApi's `KycTierPolicies` placeholder seed data
(`Garbati.IdentityApi/src/Garbati.IdentityApi/Migrations/20261016222901_SeedPlaceholderKycTierPolicies.cs`):

```
('NG', 'Tier1', ..., '["View"]'),
('NG', 'Tier2', ..., '["View", "Credit", "Debit"]'),
('NG', 'Tier3', ..., '["View", "Credit", "Debit"]')
```

`Tier1` permits only `View`. `MandateEligibility.CheckAsync` in IdentityApi denies any `Debit`/`Credit`
request with `OperationNotEligibleForKycTier` when the customer's tier doesn't list that operation. Since
every Orchestrator-opened demo account is pinned to `Tier1`, **no account opened through this flow can
ever complete a transfer** (T-024's own fix included) — not because of a code bug in the authorization
call itself, but because the two repos' independently-chosen placeholder values don't agree on what the
demo is supposed to be able to do.

Neither side is "wrong" in isolation — IdentityApi's `Tier1 = View-only` placeholder correctly models "an
unverified/self-declared identity shouldn't be able to move money" (a real, sensible default). OrchestratorApi's
choice of `Tier1` for its own demo stub just didn't account for that. T-022's own task scope was explicitly
"account opening, **plain transfer**" — so the demo stub needs to land on a tier that can actually do what
T-022 was scoped to demonstrate.

No test anywhere in the Orchestrator repo currently references `Tier1`/`DemoKycTier` at all (checked by
grep) — same class of gap as T-024: a hardcoded value with real cross-service consequences, with zero
coverage.

## The fix (demo-stub value only — not a KYC policy change)

Change `DemoKycTier` from `"Tier1"` to `"Tier2"` in `AccountOpeningService.cs`. This is still explicitly
placeholder/demo data on both sides (IdentityApi's own migration comment: "PLACEHOLDER DATA - not real
Nigerian KYC tier limits... replace them with confirmed values before relying on this table for anything
real"; OrchestratorApi's own comment: "a hardcoded, clearly-labeled demo stub decision"). This is not a
real KYC-policy decision, not a money-rule change, not a new dependency, and not an auth-scope change — it
is picking a different already-existing, already-placeholder constant so the demo stub actually supports
the demo scenario it exists for. Update the comment above the constant to note why Tier2 (not Tier1) is
required for the account-opening-then-transfer demo path to ever succeed.

## Scope

- Change the one constant and its comment in `AccountOpeningService.cs`.
- Add a test that would have caught this — e.g. assert the `KycDecisionRequest` the demo stub sends
  carries a `KycTier` value that is NOT `"Tier1"` (or directly asserts `"Tier2"`), with a comment
  explaining this is guarding against exactly this cross-service mismatch, not an arbitrary string check.
- Do not touch IdentityApi, the `KycTierPolicies` seed data, or `MandateEligibility.cs` — those are correct
  as they are; this task only fixes which already-defined demo tier OrchestratorApi's own stub picks.
- Out of scope: real KYC-tier policy values (CBN compliance confirmation is still outstanding per
  `research/nigerian-regulatory-requirements.md`), jurisdiction-code handling for non-`"NG"` callers
  (already flagged separately in DESIGN.md note #5), anything about T-024's fix (already merged, leave
  `TransferService.cs` alone).

## Out of scope

Everything else. One constant, one comment, one test.
