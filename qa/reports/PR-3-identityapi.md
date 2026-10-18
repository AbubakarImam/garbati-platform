# QA Report — Garbati.IdentityApi PR #3 (T-018 joint-account approvals)

**Verdict: Changes needed** (one Blocker, two Should-fix; everything else clean)

## CI

`gh pr checks 3`: `test` and `docker` pass, `deploy` skips (unwired, expected). Independently reproduced: `dotnet build` clean, `dotnet test` 238/238 passing with real Postgres via Testcontainers.

## Findings

1. **[Blocker] Concurrent final approvals from two different holders can both complete the threshold and both receive a signed grant.** `JointApprovalService.ApproveAsync` has no row lock, optimistic-concurrency token, or re-read-before-write guard around the "load request → count valid approvals → complete if ≥ M" critical section. Reproduced 5/5 times with a scratch concurrency test (written, run, removed — not part of the PR): on a 2-of-3 account with 1 approval already on file, two different holders approving simultaneously each independently see the pre-existing count, each add their own approval, each independently conclude the threshold is met, and **both** complete the request — producing two distinct `AuthorizationDecision` rows and two valid signed grants for a request that should resolve exactly once. The second save also silently overwrites `PendingApprovalRequest.DecisionId`, breaking traceability for whichever grant's ID got clobbered. Mitigating factor: both grants share the same `IdempotencyKey`, so a downstream LedgerApi would likely collapse a double-post attempt via its own idempotency enforcement — but that's unverified (LedgerApi doesn't exist yet) and decision #47 (LedgerApi-side grant verification) is explicitly deferred, so nothing downstream is confirmed to catch this today. Must be fixed with a database-enforced lock (row lock or concurrency token), not an application-level check.

2. **[Should fix — verified, not a blocker] The self-flagged eager-void gap (`#TBD-23`) is real but is correctly caught by the defensive live re-check.** Confirmed directly: `CustomerService.CascadeMandateRevocationAndFreezeAsync` mutates `mandate.Status` directly and never calls `MandateService.RevokeMandateAsync`, so the eager void genuinely does not fire on that path — the builder's disclosure is accurate. However, `JointApprovalService.ApproveAsync`'s live re-check at threshold-completion queries the `Mandates` table directly for every contributing approver, independent of `ApprovalRecord.VoidedAt` — so it does not matter how the mandate got revoked. Verified this is genuinely tested (`LiveRecheckAtThresholdCompletion_CatchesAHolderRevokedWithoutGoingThroughTheEagerVoidPath` directly simulates the bypass and confirms the grant is still refused). The residual gap is narrower than money-safety: a stale approval on a request that *never reaches a completion attempt* stays displayed as "counted" by a status check until someone tries to complete it or it expires — a traceability/UX staleness issue, not a money-safety one. **This should not block merge** — consistent with how T-016/T-017 handled the Closed/Deceased cascade gap (an honest, disclosed, tested no-op), except here the defensive check actually closes the money-safety hole, verified by test. The owner should still make `#TBD-23`'s scope explicit (widen `RevokeMandateAsync`'s eager-void call to cover the cascade, or formally accept the narrower scope) rather than leave it silently open.

3. **[Should fix] No test exercises the real `Closed`/`Deceased` cascade against an outstanding joint-account approval end-to-end.** The existing test simulates the bypass by direct entity mutation; nothing calls `CustomerService.CloseCustomerAsync`/`MarkDeceasedAsync` on a holder with an outstanding approval and then exercises `JointApprovalService` for real. Cheap to add, removes the remaining "probably behaves the same way" inference.

4. **[Nit — ratified by coordinator]** `#TBD-21` (scopes for the two new `approval-policy` endpoints reuse `identity.mandate.read`/`identity.authorization.check`) is a reasonable judgment call, flagged prominently by the builder. Ratified: reusing already-approved scopes for new endpoints, not creating new authorization concepts.
5. **[Nit — ratified by coordinator]** `#TBD-18` (`RequestedRequiredApprovals` column) is a new column on a brand-new table, not an edit to existing data. Ratified: consistent with every other migration this project has added for new tables.

## Verified, no issue

- `RequiredApprovals == 1` path genuinely untouched: `git diff --stat origin/main feat/joint-account-approvals -- tests/.../AuthorizationServiceTests.cs` shows zero diff.
- Design/task-file conformance otherwise checks out: entities and DB constraints match Section 3.8.2 exactly (including the real unique `(PendingApprovalRequestId, CustomerId)` index); branching in the authorization endpoint (≤1 unchanged, >1 opens a `PendingApprovalRequest`) is correct; lazy expiry is checked on both `GET` and `.../approve`, persisted before acting, with a test proving expiry-on-approve; threshold reconfiguration correctly requires every active holder, not just today's threshold, with a test proving 2-of-3 isn't enough; no grant issued for `ThresholdChange`; duplicate-approval cleanly rejected (explicit check plus DB-constraint fallback, both tested); `RequiredApprovals` bounded to `[1, activeHolderCount]`. The live re-check genuinely iterates every contributing approval, not just the one that just approved.

## What could not be checked

- Could not exercise this against a real OrchestratorApi/LedgerApi (neither exists yet) — so whether LedgerApi's own idempotency enforcement would collapse finding #1's double grant into one posted transaction is unconfirmed. Doesn't change that it needs fixing, but changes whether it's an actual-funds risk vs. an internal-consistency/audit-trail risk today.
- Did not review the discriminated-union response shape against any actual channel/Orchestrator consumer, since none exists yet.
- Did not re-test the ES256 grant-signing path itself (T-017 already hardened and tested that, out of this task's scope).

## Five interview questions for the owner

1. **Q: Two holders approve a 2-of-3 request "at the same time." What stops both from getting a valid grant?**
   A: Right now, nothing does — reproduced directly: both approvals go through, two separate signed grants get issued for what should have been one decision. The fix needs the system to lock the request while checking "are we done yet?" so a second approval arriving in that same instant has to wait and re-check rather than racing ahead on stale information.
2. **Q: Does a closed customer's old approval still count toward completing a request?**
   A: No — tested directly by simulating exactly that bypass. A second, independent safety check runs the instant a request would be completed: it looks up every approver's mandate status fresh, right then, regardless of whether the "immediate cancel" step ran. It caught it every time in testing. The only thing that stays briefly stale is what a status check displays while nobody's trying to finish the request — not whether money moves.
3. **Q: Why does the requester's own approval count toward the threshold?**
   A: A deliberate choice between two real patterns — "the requester can never approve their own request" (stricter, common in back-office banking) versus "anyone with standing, including the requester, counts" (how most multi-signature wallets work). The simpler option was chosen on purpose, recorded as its own numbered decision.
4. **Q: Why does changing the approval threshold need every holder's agreement, not just today's required number?**
   A: The looser rule has a real loophole: on a "2-of-3" account, two people could quietly agree to drop it to "1-of-3," permanently locking the third person out of ever having a say. Requiring everyone closes that hole.
5. **Q: Could an expired approval request sit there forever still showing "waiting"?**
   A: Yes, by design — nothing runs on a timer. Expiry is only written down the next time anyone looks (a status check or approval attempt), at which point it's correctly flipped before anything else happens. If nobody ever looks again, it stays showing "waiting," but it can never be approved past its real deadline since that same check runs first.
