# QA Report — Garbati.IdentityApi PR #2 (T-017 mandates + grant issuance)

**Verdict: Changes needed** (one Blocker, one Should-fix; everything else clean)

## CI

`test` and `docker` green on CI; `deploy` skipping (unwired since T-016, not this PR's scope). Independently reproduced: `dotnet build` clean, `dotnet test` 190/190 passing locally with Testcontainers/Postgres via Docker.

## Scope boundary — PASS

Grepped for `PendingApprovalRequest`/`ApprovalRecord`/joint-account-approval-collection/`.../approve` endpoints/any `RequiredApprovals > 1` branch. The only conditional on `RequiredApprovals` is a defensive "is it exactly 1" guard in the authorization service — exactly the shape the task file asked for, nothing more. No multi-holder logic leaked in.

## The Production-signing-key condition — the main finding

The owner approved committing the dev-only ES256 signing key openly (`#TBD-15`), conditional on a startup guard that refuses to run with that key in a `Production` environment. **That guard does not exist.** `SigningKeyOptionsValidator` validates only structural key correctness (is it a well-formed key), not provenance. Nothing today would stop the committed dev key from being used if the app were ever deployed with `ASPNETCORE_ENVIRONMENT=Production` and someone forgot to override `SigningKeys` with a real key.

## Database constraint — verified directly, not just read

Confirmed the one-active-mandate-per-`(customer, account)` partial unique index is enforced by Postgres itself, not just application code — tested with a raw SQL insert that bypasses EF entirely and gets a real constraint violation back.

## The ten #TBD items (Appendix C)

Reviewed all ten. None of #TBD-6 through #TBD-14 look like a security/correctness problem dressed up as style — genuine implementation judgment calls (interface naming collision, 422-vs-200 for denial, required `CreditAccountId`, no idempotency table on a read-like decision endpoint, idempotent revoke, `SigningKeyOptions` requiring ≥1 key, extra `nbf` claim, duplicate `kid` placement, honestly-stubbed LedgerApi existence-check TODO). `#TBD-15` is the one real gap — not because committing the dev key was wrong (the owner approved that), but because the condition attached to that approval wasn't met yet.

## Numbered findings

1. **[Blocker]** No startup/runtime guard exists to refuse running with the dev/test ES256 signing key (`identity-dev-key-2026-10`) when `ASPNETCORE_ENVIRONMENT=Production`. This was the owner's explicit condition for approving #TBD-15 and it is not implemented. Must be added (and tested) before merge.
2. **[Should fix]** `GrantSigner` is registered as a singleton holding one long-lived `ECDsa` instance, and `Sign()` is called with no explicit synchronization under concurrent requests. .NET does not guarantee `ECDsa` instance methods are safe for concurrent use from multiple threads; unverified under real concurrent load (no concurrency test exists). Recommend confirming `Microsoft.IdentityModel`'s caching makes this safe, or adding a lock/pool, before relying on it under load.
3. **[Nit]** `#TBD-9` (no idempotency enforcement on `POST /v1/authorization/requests`) is reasonable per the task file's own instruction — worth the owner explicitly re-confirming before T-018/Orchestrator wiring, since a caller retry on timeout could in principle request two authorizations for the same intent (harmless today since nothing downstream consumes the grant yet).
4. **[Nit]** `#TBD-12`'s extra `nbf` claim pinned to `iat` is harmless, ordinary JWT hygiene; no action needed.

## What could not be checked

- No real LedgerApi grant verifier exists yet (correctly deferred per decision #27/D-6); the in-test ES256 verification is the explicitly-allowed substitute and is genuinely cryptographic, not a stub.
- Did not independently load-test `GrantSigner` under concurrency (finding 2) — flagged as unverified, not confirmed broken.
- Did not review the GitHub Actions workflow YAML itself (only observed pass/skip results via `gh pr checks`).
- Did not check production secret-management/deployment config outside this repo.

## Five interview questions for the owner

1. **Q: If someone accidentally deploys this app with `ASPNETCORE_ENVIRONMENT=Production` but forgets to override `SigningKeys`, what happens today?**
   A: The app boots and signs real-looking grants with the publicly-committed dev key — nothing stops it. This is the gap above; a safety check needs to compare the configured key against the known dev key and refuse to start in Production.
2. **Q: Why is the one-active-mandate-per-(customer, account) rule a database index rather than just a C# check?**
   A: Two requests arriving at the same instant could both pass an in-code check before either commits, creating two active mandates. A Postgres partial unique index rejects the second insert at the database level regardless of timing — confirmed with a raw-SQL test that bypasses EF and gets a real constraint violation.
3. **Q: The grant includes `kid` in two places — why, and which matters?**
   A: Token body (matching the design doc's example) and the JOSE header (the real mechanism a verifier uses, per RFC 8725). The header copy is what actually matters for verification.
4. **Q: How do we know a denied authorization can never accidentally become approved if something goes wrong internally?**
   A: No catch block anywhere turns a failure into a grant — any unhandled exception becomes a generic 500, which never carries a signed grant. A denial is always persisted before any exception is thrown.
5. **Q: This PR builds `MandateApprovalPolicies` with `RequiredApprovals` always 1 — does anything here need to change once T-018 allows 2 or 3?**
   A: No — the single `RequiredApprovals != 1` check is a defensive guard, not logic assuming 1 forever. It will correctly deny single-holder-style requests once real multi-holder accounts exist, forcing them through T-018's new flow instead.
