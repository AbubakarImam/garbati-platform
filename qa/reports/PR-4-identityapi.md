# QA Report — Garbati.IdentityApi PR #4 (T-020 reversal authorization)

**Verdict: Approve** (no blockers; one nit sent back for a cheap pre-merge fix)

## Scope boundary — PASS

Matches the task file exactly: `identity.ops.reversal` scope, `Operation = "Reversal"` on the existing endpoint, five reason codes (no `DISPUTE`), `ReversalOutcomes` with exactly two states, the new outcome endpoint, audit events, migration. No `PendingApprovalRequest`-style workflow, no lien/sweep, no third outcome state — grepped and confirmed only present in comments explaining why they weren't built.

## The critical check: `ControllerAuthorizationTests.cs` exception — holds up

- The exception is a named `HashSet<(Type, string)>` naming exactly one method (`AuthorizationController.RequestAuthorization`). Confirmed it does not weaken the check for any other action — the structural test still asserts `actions.Count >= 7` and `Assert.Empty(violations)` on everything else.
- A companion test proves the exempted action still carries `[Authorize]` (no policy, but some key is still required) — it did not quietly become unauthenticated.
- The in-action logic is genuinely mutually exclusive — verified by reading the code, running the existing HTTP tests, and writing additional adversarial tests: an `identity.ops.reversal`-only caller gets 403 on a Debit; an `identity.authorization.check`-only caller gets 403 on a Reversal; a lowercase `"reversal"` operation string still correctly routes to the ops-reversal scope check (case-insensitive matching doesn't create a bypass).
- `dev-ops` holds `identity.ops.reversal` but not `identity.authorization.check`; `dev-orchestrator` holds the reverse. No customer-facing dev client holds `identity.ops.reversal`.

## Money safety

- `ReversalOutcomes.outcome` has a DB check constraint restricting it to exactly `'Completed'`/`'WriteOff'`, a real FK to `authorization_decisions.decision_id`, and a unique index enforcing at-most-one-outcome-per-decision — defense in depth beyond the app-level check, with the race-condition path (Postgres `23505`) correctly caught and remapped to a 409.
- Audit events are staged on the same `DbContext` and committed by the same `SaveChangesAsync` as the business change.
- `ReversalAuthorizationService` has zero mandate/customer/KYC lookups — confirmed by direct code read and by the three bypass tests (zero/single/2-of-3 joint mandate), all pass.
- Grant token never logged/persisted anywhere in the new code.

## Tests

14 service tests + 7 HTTP tests are genuine behavioral tests. Ran all 265 (240 baseline + 25 new) — pass. QA additionally wrote 6 of its own adversarial tests (zero/negative amount, missing reason code, case-insensitive operation/reason-code, idempotency-key reuse) against a live instance — all behaved as expected. No existing test weakened — the one structural test change is additive (narrower allowlist + new proof test).

## Findings

1. **[Nit — sent back for a pre-merge fix]** Reason-code casing accepted but not normalized: a lowercase `"dupl"` is accepted and persisted/signed verbatim rather than as the canonical `"DUPL"`. Low risk today (no real consumer exists yet), but a future exact-string-match verifier would silently miss a validly-accepted lowercase variant. Sent back to the builder.
2. **[Nit — ratified]** `#TBD-27` (reusing `DebitAccountId` to carry the original transaction reference, `CreditAccountId` made nullable) is a reasonable reading of an ambiguous task-file instruction. Ratified.
3. **[Nit — ratified, assigned decision #36]** `#TBD-31` (persisting `ReasonCode` on `AuthorizationDecision` beyond what decision #33 strictly required) is a sensible, low-risk addition — otherwise the reason code becomes unrecoverable after the 30-second grant expires. Assigned its own decision number rather than silently folding into #33's scope.

No blockers. `#TBD-28` (the scope-check exception), `#TBD-29` (nullable grant claims), `#TBD-30` (404/409 split), and `#TBD-32` (one audit event per decision) are all ratified as reasonable, consistent engineering choices.

## What could not be checked

- No real ops channel or LedgerApi exists, so end-to-end behavior through an Orchestrator could not be verified — explicitly out of scope per the task file.
- `deploy` CI job not checked (reports "skipping," documented elsewhere as unwired, unaffected by this change).
- Sharia/compliance open questions in Section 8.8 (write-off framing, CBN timing) not independently re-verified — explicitly marked unresolved in DESIGN.md itself, not this PR's job.

## Five interview questions for the owner

1. **Q: On a 2-of-3 joint account, why doesn't a reversal need 2 of the 3 holders to agree?**
   A: The approval already happened upstream, inside the ops team's own process, before IdentityApi is ever called. IdentityApi just trusts that decision — asking the account holders again would be asking the wrong people, since reversal was never their call.
2. **Q: The reversal endpoint reuses the same URL as a normal transfer request. How does the code pick which permission to check?**
   A: It looks inside the request body rather than the URL, and checks a different permission depending on the answer — a one-off exception to a rule the codebase otherwise enforces everywhere, which got its own named, tested exception rather than a silent change.
3. **Q: If someone sends a reason code in lowercase, what happens today?**
   A: It's accepted, but what gets saved and put into the signed grant is the lowercase version exactly as sent, not the standard uppercase form — a minor rough edge fixed in a follow-up commit before merge.
4. **Q: What stops someone from recording a reversal's outcome twice?**
   A: The second attempt is rejected outright — both by an explicit code check and, as a backstop, by a database rule that won't allow two outcome rows for the same decision even under a race condition.
5. **Q: Why was a third "maybe we'll recover this later" state deliberately left out?**
   A: Research specifically checked whether real financial systems default to automatically reaching into an account later to recover owed money, and found they don't — that always requires a separate, specially-granted power. Adding a vague in-between state would have invented behavior nothing else actually demonstrates exists by default.
