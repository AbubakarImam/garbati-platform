# T-018: IdentityApi joint-account multi-approval workflow

Target repo: `Garbati.IdentityApi`, on top of T-016+T-017's merged code (`Customer`, KYC, `Mandates`, `MandateApprovalPolicies` always `RequiredApprovals = 1` so far, ES256 `GrantSigner`, single-holder `AuthorizationService`). Branch: `feat/joint-account-approvals`. Spec: `docs/DESIGN.md` Section 3.8 in full, plus 2.2a, 5.1, 5.2, 7.1, and decisions #19–#25 in Section 9 (the table is the corrected, authoritative numbering — don't trust a stray number cited elsewhere in the body text without checking the table).

---

This is the task that finally uses `MandateApprovalPolicy.RequiredApprovals` as something other than `1`. Every mechanic below was already decided by the owner (decisions #20–#25, informed by `garbati-platform/research/joint-account-approval-patterns.md`) — this task implements exactly those decisions, not new ones. If you find yourself needing to invent a rule the design doc doesn't already specify, stop and ask in the PR rather than guessing, same standard as T-016/T-017.

## What to build

### 1. `PendingApprovalRequest` and `ApprovalRecord` (3.8.2)

- `PendingApprovalRequests`: `Id`, `LedgerAccountId`, `RequestedBy` (`CustomerId`), `Kind` (`Debit`/`ThresholdChange` — see item 4), `Operation`, `DebitAccountId`, `CreditAccountId`, `Amount`, `Currency`, `IdempotencyKey`, `RequiredApprovals` (copied from the account's policy **at request-creation time** — a concurrent policy change must not retroactively change what an in-flight request needs), `Status` (`Pending`/`Approved`/`Expired`/`Cancelled`), `CreatedAt`, `ExpiresAt` (`CreatedAt + 1 hour`, decision #23).
- `ApprovalRecords`: `PendingApprovalRequestId`, `CustomerId`, `ApprovedAt`, `VoidedAt`, `VoidedReason` (nullable). Unique constraint on `(PendingApprovalRequestId, CustomerId)` — enforced by the database, not just application code, matching every other uniqueness rule this service has built so far.
- Creating a request immediately creates its first `ApprovalRecord` — the requester's own, since decision #21 confirmed the initiator counts toward the threshold.

### 2. `POST /v1/authorization/requests` now branches on `RequiredApprovals`

- `RequiredApprovals == 1`: **unchanged behavior from T-017.** Do not touch this path's existing logic or tests beyond what's structurally necessary to share code with the new path.
- `RequiredApprovals > 1`: create a `PendingApprovalRequest` (with its first `ApprovalRecord`, the requester's), check if the threshold is already met (it won't be, on creation, unless `RequiredApprovals == 1`, which is the branch above — so a fresh multi-holder request is always `Pending` after creation). Response: the pending request's id and status, not a grant yet.
- `POST /v1/authorization/requests/{id}/approve` (new endpoint): a **different** holder of the same account submits their approval. Validate: the request is `Pending` (not `Expired`/`Approved`/`Cancelled`), not past `ExpiresAt` (if it is, transition it to `Expired` first and reject the approve), the approving customer has an `Active` mandate on the account (and, for a `Debit` kind, permission for the operation), and they haven't already approved this exact request (the unique constraint is the backstop; check first for a clean error message). Record the `ApprovalRecord`.
- **The moment an approval would complete the threshold** (decision #24 — this is the one the task's research flagged as a real, documented hazard class if skipped): before honoring it, re-check **every** contributing approver's mandate is still `Active` right now, not just at the moment each approval was individually given. If any is no longer active, do not complete the request — it stays `Pending`, waiting for a replacement approval or expiry. If all check out: for a `Debit` request, run the same eligibility/grant-issuance logic T-017 built (requester's eligibility, account rules), issue the signed grant, mark the request `Approved`. For a `ThresholdChange` request, see item 4.
- `GET /v1/authorization/requests/{id}` — status and, if `Approved`, the decision id (never the grant token itself — same logging rule as T-017).

### 3. Stale-approval voiding on revoke (decision #24, the other half)

`MandateService`'s revoke path (built in T-017) must now also: find every `Pending` `PendingApprovalRequest` on the same `LedgerAccountId` where this customer has a non-voided `ApprovalRecord`, and void that record (`VoidedAt`, `VoidedReason`) in the **same database transaction** as the mandate revocation itself. This is the eager half of decision #24; the live re-check in item 2 is the defensive second half — both are required, not either-or (a revoke that happens *between* the live re-check and the grant actually being issued is an unavoidable tiny race in any system, but a revoke that happens before that instant must be caught by this eager void, not left to luck).

### 4. Reconfiguring the threshold (3.8.4, decision #22)

- `GET /v1/accounts/{ledgerAccountId}/approval-policy` — current `RequiredApprovals` and the account's active-holder count.
- `POST /v1/accounts/{ledgerAccountId}/approval-policy/change-requests` — body: `RequestedBy`, desired `RequiredApprovals` (must be between 1 and the account's current active-holder count inclusive — reject otherwise with a clear 400). Creates a `PendingApprovalRequest` with `Kind = ThresholdChange`, `RequiredApprovals` forced to the account's **current active-holder count** (every holder must consent to a threshold change — not just today's threshold's worth, which is the whole point of decision #22; re-read `docs/DESIGN.md` 3.8.4's reasoning if this isn't clear).
- Approving a `ThresholdChange` request uses the same `.../approve` endpoint as a `Debit` request (same entity, same flow) — once it reaches its required count (= every holder), update the `MandateApprovalPolicy.RequiredApprovals` to the requested value, set `LastChangedAt`/`LastChangedBy`, mark the request `Approved`. No grant is issued for this kind — there's no money movement to authorize, just a policy change.

### 5. Expiry

No background job/scheduler exists anywhere in this codebase, and this task should not introduce one. Check `ExpiresAt` **lazily**: whenever a `PendingApprovalRequest` is read (`GET`) or acted on (`.../approve`), if it's still `Pending` but past its `ExpiresAt`, transition it to `Expired` first (persist that transition) before doing anything else with it. A request that nobody ever looks at again after it expires simply stays `Pending` in the database until someone does look — that's acceptable and matches how a real system without a scheduler would behave; do not build a sweeper for this task.

## Explicitly out of scope

Planned key rotation and emergency key-compromise tooling (4.3.1/4.3.2) beyond what T-017 already built. Reversal authority (T-015, still undesigned). Any change to the `RequiredApprovals == 1` code path's existing tests or behavior beyond sharing code where natural. LedgerApi-side grant verification (decision #47, still deferred).

## Tests

- A 2-of-3 account: one approval isn't enough; a second distinct holder's approval completes it and issues a grant; the requester's own approval correctly counts as the first.
- A 2-of-2 account: the requester's own approval is 1 of 2, a second holder completes it.
- Approving twice as the same holder is rejected (clean error, not just relying on the DB constraint to surface a raw error).
- A request past its `ExpiresAt`: a `GET` after expiry shows `Expired`; an `.../approve` attempt after expiry is rejected and the request is (or already was) marked `Expired`.
- Revoking a holder's mandate while their approval is outstanding on a `Pending` request voids that approval; a subsequent attempt to complete the threshold using the old (now-voided) approval count fails — the request stays `Pending`.
- The live re-check at threshold-completion: construct a case where an approval was valid when given but the holder is revoked before the *last* approval arrives — confirm the grant is NOT issued and the request stays `Pending`.
- Threshold reconfiguration: a 2-of-3 account's change request needs all 3 holders, not just 2; completing it updates the policy; a subsequent debit respects the new threshold.
- Migration test for the new tables/constraints.

Run build and tests yourself (Docker should be available for Testcontainers — if not, say so explicitly, matching T-016/T-017's standard). Open a pull request from `feat/joint-account-approvals` against `main` on `AbubakarImam/Garbati.IdentityApi` (branch-protected — get CI green on your branch first). Include what changed, how it was tested, what you could not verify, every judgment call and why, and the five-question/plain-language LEARNING_LOG draft, per `AGENTS.md`'s "Done means" checklist. Do not merge.
