# T-017: IdentityApi mandates + single-holder authorization grant issuance

Target repo: `Garbati.IdentityApi`, on top of T-016's merged foundation (`main` now has `Customer`, KYC, idempotency, audit, API-key auth). Branch: `feat/mandates-and-grants`. Spec: `docs/DESIGN.md` — read it in full, especially Sections 2.2, 2.2a, 2.4, 3 (3.1–3.7 only — **not** 3.8), 4.3.1, 5, 6, 7, and the decisions log (Section 9, now correctly numbered — cite the table row that actually matches what you're describing, the same way T-016's follow-up fix did).

---

**This is the second of (at least) two build tasks.** It adds `Mandates` and single-holder (`RequiredApprovals = 1`) authorization grant issuance. **Do not build Section 3.8's joint-account approval workflow** — no `MandateApprovalPolicies` table beyond always creating it with `RequiredApprovals = 1`, no `PendingApprovalRequests`/`ApprovalRecords` entities, no `/v1/authorization/requests/{id}/approve` endpoint, no multi-holder approval collection logic. That is a separate future task (T-018, not started). This task's authorization endpoint only ever has to satisfy `M = 1` — the requester's own mandate is always enough, because nothing yet lets an account have `RequiredApprovals > 1`.

**New dependency, already approved by the owner:** `Microsoft.IdentityModel.JsonWebTokens` and `Microsoft.IdentityModel.Tokens` (the standard ASP.NET Core-maintained JWT libraries) for signing and representing the grant. Do not hand-roll JWT/JWS encoding or ECDSA signing by hand — use these packages' intended APIs (`JsonWebTokenHandler`, `SigningCredentials` with an `ECDsaSecurityKey`, etc.).

## What to build

### 1. `Mandates` and `MandatePermissions` (`docs/DESIGN.md` 2.2, decision #19)

- `Mandates`: `Id`, `CustomerId`, `LedgerAccountId` (a plain string/cross-service identifier — **not** a foreign key to anything, LedgerApi's database is separate), `Status` (`Active`/`Revoked`), `CreatedAt`, `CreatedBy`, `RevokedAt`, `RevokedBy`, `RevocationReason`.
- `MandatePermissions`: `MandateId`, `Operation` (`View`/`Debit`/`Credit`).
- **The real constraint, not just a comment:** a database partial unique index on `(CustomerId, LedgerAccountId) WHERE Status = 'Active'` — at most one active mandate per customer per account, but multiple *different* customers can each have their own active mandate on the same `LedgerAccountId` (that's what makes an account joint; this task doesn't exercise that case, but the constraint must allow it, not prevent it).
- A mandate cannot become `Active` referencing a `LedgerAccountId` that hasn't been confirmed to exist. `docs/DESIGN.md` 2.4 describes the one narrow, read-only exception to "IdentityApi never calls LedgerApi" for exactly this check — but **LedgerApi has no code yet either**, so there is nothing to actually call. For this task: accept `LedgerAccountId` as an opaque string the caller asserts exists, skip the real HTTP call, and leave a clearly commented TODO (same honesty standard as T-016's cascade stub) noting that real account-existence verification needs LedgerApi to exist first. Do not fake a call to a service that isn't there.

### 2. `MandateApprovalPolicy` (`docs/DESIGN.md` 2.2a, decision #20 — shape only, not the workflow)

- One row per `LedgerAccountId`, created the moment that account's first mandate is created. `RequiredApprovals` always `1` in this task (there is no endpoint yet to set it otherwise). `CreatedAt`, `LastChangedAt`, `LastChangedBy` (nullable — nothing changes it yet).
- This exists so the schema is ready for T-018 without a disruptive migration later. Do not build any logic that reads `RequiredApprovals` as anything other than `1` — if you find yourself writing an `if (requiredApprovals > 1)` branch, stop; that branch has no way to be exercised or tested in this task and belongs in T-018.

### 3. Mandate endpoints (`docs/DESIGN.md` 5.1)

- `POST /v1/customers/{customerId}/mandates` — attach a mandate (and, if one doesn't already exist for this `LedgerAccountId`, create its `MandateApprovalPolicy`).
- `GET /v1/customers/{customerId}/mandates` — list a customer's mandates.
- `GET /v1/accounts/{ledgerAccountId}/mandates` — list every active holder of an account.
- `GET /v1/mandates/{mandateId}` — one mandate's status and permissions.
- `POST /v1/mandates/{mandateId}/revoke` — revoke, recording actor and reason (mirror T-016's `CustomerStatusChangeRequestValidator` pattern: reason is required, not just documented as required).

### 4. Wire the Closed/Deceased cascade (`docs/DESIGN.md` 2.1, decision #14 — finishing what T-016 deliberately left as a stub)

`CustomerService.CascadeMandateRevocationAndFreezeAsync` (or whatever it's now called — check T-016's actual code) is currently a no-op with a comment describing exactly what belongs there. Implement it:

1. Query every `Active` mandate this customer holds.
2. Revoke each one (same transaction as the customer's own status change and its audit event — this must all commit together or not at all).
3. For each distinct `LedgerAccountId` across those mandates, check whether any *other* customer still holds an `Active` mandate on it. If not — this customer was the sole holder — write an `OutboxMessages` row using the `AccountFreezeRequestedPayload` shape T-016 already built. If another holder remains active, do **not** queue a freeze (a joint account with surviving holders stays open, per 2.1's table).
4. There is a test from T-016 (`CloseCustomerAsync_WritesNoOutboxMessage_BecauseMandatesDoNotExistYet` or similarly named) that currently asserts **zero** outbox rows on this path, specifically written to force this exact task to update it. Find it, and change it to assert the real behavior now that mandates exist (zero rows when there were no mandates to revoke; one row per solely-held account when there were).

### 5. Signing key configuration (`docs/DESIGN.md` 4.3.1, decisions #4/#26)

- Generate (or accept as config) an ES256 (P-256) signing keypair. For local/dev, generate one and commit it the same way T-016 committed dev API key hashes — openly, clearly labeled dev-only, never presented as a real secret. Give it a `kid`.
- `docs/DESIGN.md` 4.3.1 (planned rotation) and 4.3.2 (emergency compromise) describe the *procedures* this key management needs to support eventually — this task does not need to build key rotation or an emergency-revocation endpoint, but the configuration shape (keyed by `kid`, more than one key nameable in config) must not make adding that later a breaking change. A single configured key today, structured so a second one could be added alongside it, is enough.

### 6. Authorization grant issuance (`docs/DESIGN.md` 3.1–3.7, decisions #4, #9, #10)

- `POST /v1/authorization/requests` — body matches 3.8.2's canonical fields even though the joint-account entity itself isn't built: `RequestedBy` (`CustomerId`), `Operation`, `DebitAccountId`, `CreditAccountId`, `Amount`, `Currency`, `IdempotencyKey`. Validate: the customer is `Active`, KYC-eligible for the operation (per their `KycTier` — use whatever placeholder tier-policy logic T-016 built), has an `Active` mandate on `DebitAccountId` permitting `Operation`, and that account's `MandateApprovalPolicy.RequiredApprovals` is `1` (it always will be in this task, but check it anyway — defensive, not decorative, since this is the exact assumption the rest of this task's logic depends on).
- On success: issue a signed grant with the claims in 3.3/3.5 (`iss`, `aud`, `typ: "transaction-authorization"`, `jti`, `sub`, `kid`, `operation`, `debitAccountId`, `creditAccountId`, `amount` — canonical `G29`-equivalent formatting, no numeric-vs-string mismatch — `currency`, `idempotencyKey`, `mandateId`, `iat`, `exp` = `iat + 30s`). Never log the token itself (decision #13's logging rule extends here — log the `jti`/decision id, never the signed string).
- On denial (ineligible customer, no mandate, wrong operation): a clear reason code, no grant, no exception leaking a stack trace.
- Fail closed (decision #10): if anything needed to make the decision can't be established (e.g., the mandate lookup itself errors), return a failure — never a grant.
- `GET /v1/authorization/requests/{id}` — status/result of a past request (approved + the decision id, or denied + reason). This endpoint does **not** need a `PendingApprovalRequests` table (that's T-018's joint-account entity) — a single-holder request resolves synchronously in the same call that creates it, so this GET is for after-the-fact lookup/audit, not for polling a pending multi-party approval.
- No idempotency key requirement on this endpoint itself — `docs/DESIGN.md` 5.1 doesn't ask for one here, and each call is a fresh, fast decision-and-sign step, not a record-creation step with duplicate-row risk the way customer creation has. If you think it needs one anyway, say so as a question in the PR rather than silently adding it.

## Explicitly out of scope

Everything in `docs/DESIGN.md` Section 3.8 (`PendingApprovalRequest`, `ApprovalRecord`, the collect-approvals flow, the 1-hour timeout, reconfiguration consent, stale-approval voiding) — that's T-018. `/v1/accounts/{id}/approval-policy` and its change-request endpoint. LedgerApi-side grant verification (decision #47, still deferred per D-6 — nothing here should assume LedgerApi checks anything). Reversal authority (T-015, still undesigned). Real LedgerApi account-existence calls (LedgerApi has no relevant endpoint changes for this and no code to call anyway — see item 1's note).

## Tests

- Mandate attach/list/revoke, including the reason-required validation on revoke.
- The unique-active-mandate-per-(customer, account) constraint: two different customers can each get an active mandate on the same account; the same customer cannot get two.
- The Closed/Deceased cascade, now for real: closing a customer with one mandate on a solely-held account writes exactly one outbox row with the right account id; closing a customer who shares an account with another active holder writes none; the updated T-016 test reflects this.
- Grant issuance: a customer with an eligible, active mandate on the debit account gets a valid, correctly-claimed, ES256-signed grant; an ineligible customer, a missing mandate, a mismatched operation, and a suspended/closed customer each get a clean denial, not a grant and not an unhandled exception.
- Security-acceptance items from Appendix A that apply to this task's scope: a grant's claims match the request exactly (tamper one field, confirm a would-be verifier — even a simple in-test check, since no LedgerApi exists yet to verify for real — would reject it); a revoked mandate cannot obtain a new grant; the grant is never logged (check your own logging/audit code doesn't emit the raw token).
- Migration test confirming the new tables/constraints, mirroring T-016's style.

Run build and tests yourself (Docker should be available for Testcontainers — if not, say so explicitly and report exactly what you could and couldn't run, matching T-016's honesty on this point). Open a pull request from `feat/mandates-and-grants` against `main` on `AbubakarImam/Garbati.IdentityApi` (branch-protected — get `test`+`docker` green on your branch first). Include what changed, how it was tested, what you could not verify, every judgment call you made, and the five-question/plain-language LEARNING_LOG draft, per `AGENTS.md`'s "Done means" checklist. Do not merge.
