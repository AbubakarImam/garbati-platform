# QA Report — Garbati.StaffIdentityApi PR #1 (T-032 foundation)

**Verdict: Approve** (no blockers; one Should-fix item for the owner to consciously accept or close, plus nits)

## 1. Administrator-seed bootstrap (highest priority) — VERIFIED SAFE

The seed migration inserts exactly one row (`STF-0000000001`, username `admin`) with a password hash that is 32 bytes of `0xFF` — not a real hash of anything, ever. QA independently built a throwaway test harness against the real `PasswordHasher<T>.VerifyHashedPassword` and confirmed this placeholder fails against every attempted password (including edge cases), always `Failed`, never throws — because `PasswordHasher<T>` reads the first byte as a format marker, and `0xFF` matches neither recognized format. `StaffService` also wraps the call defensively, though QA's test shows that's not even strictly needed for this placeholder.

The bootstrap path (one call to the existing admin password-reset endpoint against the seeded account) is documented end-to-end in the README with a working `curl` example, and proven by a dedicated test that QA ran itself against a real Postgres container. No secret ever appears in the migration file, PR description, or commit history.

## 2. SupervisorStaffId validation — correct, with one real, undisclosed gap

Self-reference, non-existent staff, non-`Active` staff, and non-`Supervisor` role (including `Administrator` specifically) are all correctly rejected, enforced on every write of the field. `CreateAsync` never sets it directly, matching the task's instruction.

**Real gap found:** changing a `Supervisor`'s role to anything else does not touch or flag any `Teller`/`CustomerServiceOfficer` still pointing at them via `SupervisorStaffId` — they silently keep "reporting to" someone no longer eligible. Consistent with the ratified design's "assignment-time-only" choice, but nowhere documented as an accepted consequence, and untested either way.

## 3. Generic-401 non-enumeration — VERIFIED with real byte-for-byte comparison

A test parses all four failure-case response bodies (wrong username, wrong password, suspended, terminated) and asserts they're identical after stripping only legitimately-varying fields (correlation id). The underlying code structurally collapses all three causes into one boolean before responding — there's no separate branch that could drift apart later.

## 4. Idempotent staff creation — VERIFIED

A replay with the same idempotency key returns the same staff record with the one-time credential explicitly `null` the second time — and since the plaintext is never persisted anywhere, a replay has no way to reconstruct it regardless.

## 5. Scope separation — VERIFIED, including the deliberate exception

Full matrix tested across every endpoint (no key → 401, wrong scope → 403, correct scope → succeeds). A reflection-based structural test fails if any future controller action lacks a recognized scope policy, with exactly one named, intentional exception (the dual-scope password endpoint) — itself confirmed not anonymous.

## 6. Scope/task boundaries — clean

No out-of-scope items (MFA, multi-role, SSO, etc.) implemented anywhere — all confirmed still just prose in the design doc. Single new dependency (`Microsoft.Extensions.Identity.Core`), used only for the hasher.

## 7. Audit/security

- Audit rows always staged in the same transaction as the change they describe, across every mutating path.
- `Terminated → Terminated` is a true no-op — no audit row written for a repeated attempt, unlike other same-state transitions elsewhere on the platform.
- Temporary credentials generated via a real CSPRNG, correctly distinguished from the `Random.Shared` used for non-secret identifiers.
- No password/hash/credential ever appears in a log statement or error response.
- `ActingStaffId` on every admin-driven write is taken at face value from the caller, never checked against a real staff record or session — consistent with this service having no session concept of its own, but worth knowing explicitly before `OpsApi` (its only caller) exists for real.

## 8. Build/tests — ran independently

`dotnet build -c Release`: 0 warnings, 0 errors. `dotnet test -c Release`: **90/90 passed**, matching the PR's claim. CI (`test`/`docker`) green; `deploy` correctly skipped (placeholder, as expected at this stage).

## Findings

1. **(Should fix, not a blocker)** Reassigning a Supervisor away from that role orphans any reports still pointing at them — a real, silent org-chart inconsistency the ratified design permits but never explicitly accepted as a known trade-off. Flagged to the owner as D-17.
2. **(Nit)** `Terminated → Terminated` leaves zero audit trail of the attempt — worth deciding deliberately, since it's the one case a compliance review would most want to see.
3. **(Nit)** `ActingStaffId` isn't validated against a real staff record — fine for today's single-caller design; `OpsApi` (T-028) needs to always pass a real, verified id.

No Blockers.

## What QA could not check

- No real `OpsApi` caller exists yet to exercise this from a real consumer's perspective.
- Real production deployment (placeholder `deploy` job, as expected at this stage).
- Exhaustive fuzzing of every validator (spot-checked, not exhaustive).

## Interview Q&A

1. **If an attacker gets the committed migration file, can they log in as the first Administrator?** No — the seeded hash can never authenticate against any password; independently verified against the real hasher. The account only becomes usable after an operator runs the one-time reset.
2. **What happens if a Supervisor gets reassigned and staff still point at them?** Nothing automatic — they keep pointing at someone no longer eligible. Matches the design's assignment-time-only choice, but hasn't been explicitly accepted as OK — now flagged as D-17.
3. **Why does the admin reset branch ignore any password an admin tries to supply?** So a real password for someone else's account is never typed into a request body — same rule staff creation already follows. A fresh credential is always generated server-side.
4. **If OpsApi's credential is compromised, can its actions be told apart from a legitimate admin's in the audit log?** Not fully — `ActingStaffId` is self-reported by the caller, not verified. An accepted consequence of this service having no session concept of its own; audit trustworthiness rests on OpsApi behaving correctly.
5. **Why does a repeated "terminate an already-terminated account" leave no audit trail?** It's a pure no-op — nothing changed, so nothing is written. Worth deciding deliberately rather than by default.
