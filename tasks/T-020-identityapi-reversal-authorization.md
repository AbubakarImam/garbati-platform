# T-020: IdentityApi reversal authorization

Target repo: `Garbati.IdentityApi`, on top of T-016/T-017/T-018's merged code. Branch: `feat/reversal-authorization`. Spec: `docs/DESIGN.md` Section 8 in full (decisions #30–#35), plus the confirmation note added to 8.6 on 18 Oct 2026 (T-021 — the two-state `ReversalOutcomes` design is confirmed, do not add a third state or any lien/auto-sweep mechanism).

---

This is a smaller task than T-016/T-017/T-018: reversal authorization deliberately reuses existing grant-issuance machinery rather than building new approval-collection infrastructure. Read Section 8.1–8.2 carefully before starting — the key simplification is that a reversal request bypasses the account's own mandate/`MandateApprovalPolicy` entirely (decision #31) and comes only from a trusted ops-channel caller whose own approval process already ran *before* calling this endpoint (decision #30). **Do not build any `PendingApprovalRequest`-style workflow for reversals** — that would contradict the whole point of this design.

## What to build

### 1. New scope: `identity.ops.reversal` (decision #30)

Add it alongside the existing scopes (`Authorization/IdentityScopes.cs` or wherever the existing scope list lives). Wire it into the same options/validator pattern every other scope uses. It is never granted to a customer-facing channel's service identity in any seed/dev config — only to a dedicated `dev-ops` or similar client, matching how LedgerApi's own dev config separates `dev-payment-api` from `dev-ops`.

### 2. `"Reversal"` as a new `operation` value on `POST /v1/authorization/requests` (8.3, 8.5)

Extend the existing request contract so a reversal request can be made through the same endpoint used for transfers:

- `Operation = "Reversal"` (extend whatever enum/string-constant the existing operation values use).
- A reference to the original transaction/chain being reversed, replacing debit/credit accounts for this operation type (IdentityApi does not independently verify this reference exists at LedgerApi — there is nothing to call yet; accept the caller's assertion, same pattern as the `LedgerAccountId` `TODO` at mandate attachment, 2.4. Leave an equally clear comment here, don't silently skip the question).
- A required `ReasonCode`, validated against exactly: `DUPL`, `TECH`, `FRAD`, `CUST`, `UNAUTH` (8.4). Reject anything else with 400. **Do not implement `DISPUTE`** — it is deliberately deferred.
- Amount/currency, idempotency key — same canonical fields every other request already carries.

**Validation for a `Reversal` request, replacing the normal customer/mandate eligibility check used for `Transfer`:**
- The caller holds `identity.ops.reversal` (this is the only check that matters — no customer status check, no KYC check, no mandate lookup, no `MandateApprovalPolicy.RequiredApprovals` check of any kind, regardless of whether the account is single-holder or joint). If you find yourself writing a mandate lookup for a `Reversal` request, stop — that's exactly the thing decision #31 says not to do.
- The reason code is one of the five valid values.

On success: issue the signed grant exactly as `GrantSigner` already does for any other operation, with `operation: "Reversal"` and the reason code as an added claim (8.5). Persist an `AuthorizationDecision` exactly as for any other request. On failure (bad reason code, missing scope): deny cleanly, same status-code conventions already established (403 for the scope, 400 for a bad reason code) — do not invent a new error shape.

### 3. `ReversalOutcomes` (8.6 — confirmed two-state design, do not add a third state)

New entity: `AuthorizationDecisionId` (FK to the `AuthorizationDecision` the original reversal grant produced), `Outcome` (`Completed` / `WriteOff` — exactly two values, nothing else), `RecordedAt`, `RecordedBy`, `Notes` (nullable).

New endpoint: `POST /v1/authorization/requests/{id}/reversal-outcome` — body: `Outcome`, `RecordedBy`, `Notes?`. Scope: `identity.ops.reversal` (same as the request itself — this is the ops channel or the Orchestrator reporting back what LedgerApi's chain-reversal actually did, per 8.6's second paragraph). Validate: `{id}` refers to a decision whose `operation` was `"Reversal"` and that was actually `Approved` (you can't record an outcome for a denied or non-reversal decision — 404 or 409, your call which, say why in the PR); at most one outcome per decision (second attempt: 409, matching the established "terminal state, don't silently overwrite" convention from `Mandates`/`Customer`'s `Closed`).

This is a **thin audit record** — do not build any collections/lien/watch-listing logic. It only records that someone decided the outcome and what they decided.

### 4. Audit (6.3)

Audit events for: a reversal authorization requested, the resulting grant issued or the denial (same pattern as every other authorization decision), and a reversal outcome recorded. Never log the grant token itself (unchanged rule).

## Explicitly out of scope

Any `PendingApprovalRequest`/`ApprovalRecord`-style workflow for reversals (decision #30 exists specifically to avoid needing this). Any lien, auto-sweep, or third `ReversalOutcomes` state (decision #34, reconfirmed 18 Oct 2026 — T-021 found no real-world precedent for it). The `DISPUTE` reason code (8.4, deferred). The ops channel itself — it doesn't exist; this task only builds IdentityApi's side, same "no consumer yet" pattern already established for `OutboxMessages`. Any real LedgerApi existence-check for the original transaction reference (no LedgerApi code to call).

## Tests

- A caller without `identity.ops.reversal` gets 403 on both new endpoints.
- A caller with the scope but an invalid `ReasonCode` gets 400.
- A `Reversal` request succeeds (gets a valid, correctly-claimed grant) for an account that has **zero** mandates, a **single-holder** mandate, or a **joint** (`RequiredApprovals > 1`) mandate — all three must behave identically, proving the mandate bypass is real, not accidentally still gated by something. This is the single most important test in this task.
- A `Reversal` grant's claims include `operation: "Reversal"` and the reason code; tamper either and a verifier (the same simple in-test ES256 check T-017/T-018 already use, no real LedgerApi exists) rejects it.
- Recording a `Completed` outcome, and separately a `WriteOff` outcome, on two different approved reversal decisions.
- Recording a second outcome on the same decision is rejected.
- Recording an outcome against a decision that was `Denied`, or whose `operation` was not `"Reversal"`, is rejected.
- Migration test for the new table/constraints, mirroring prior tasks' style.

Run build and tests yourself (Docker should be available for Testcontainers — if not, say so explicitly, matching every prior task's honesty standard). Open a pull request from `feat/reversal-authorization` against `main` on `AbubakarImam/Garbati.IdentityApi` (branch-protected — get `test`+`docker` green on your branch first). Include what changed, how it was tested, what you could not verify, every judgment call and why, and the five-question/plain-language LEARNING_LOG draft, per `AGENTS.md`'s "Done means" checklist. Do not merge.
