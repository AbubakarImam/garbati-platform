# GARBATI Core Banking: Platform Architecture

**Status:** Draft v0. Proposed boundaries, not final. Each "Proposed" item needs your confirmation or change before it goes in the decisions log (Section 8).

This document is the map. It says which service owns which kind of truth and who may call whom. Each service keeps its own `docs/DESIGN.md` for the detail; this file never repeats it.

---

## 1. Purpose and principles

GARBATI is an open-source core banking platform for Sharia-compliant microfinance institutions, built in tranches (one service at a time, each designed before it is coded).

1. **One kind of truth, one owner.** Every fact has exactly one service that is allowed to say what is true. Other services ask; they never keep their own copy.
2. **Money truth lives only in LedgerApi.** No other service stores balances or posts entries.
3. **Fail closed.** If a service cannot establish a positive answer, the operation stops.
4. **Decisions and effects are separate.** A service that *decides* (is this allowed? what fee applies?) does not *do* (move money). Money moves only through LedgerApi, called by the orchestrator.
5. **Design before code.** Every service starts as a questions-only skeleton, then adversarial review, then a decisions log.

## 2. Service map (Proposed)

| Service | Owns (source of truth) | Does NOT own | Status |
|---|---|---|---|
| **LedgerApi** | Accounts, entries, balances, idempotency, ledger audit | Customers, products, fees, limits | Built, deployed |
| **IdentityApi** (Garbati.IdentityApi) | Customer identity, KYC status and tier, mandates (who may operate which account, including joint accounts with configurable multi-holder approval), suspension | Product rules, fees, tax, standing orders, transaction flow | v1 design fully built: Customer, KYC, Mandates, single-holder and joint-account (M-of-N) authorization grant issuance all merged (PR #1-#3). Reversal authority (T-015) still undesigned; key rotation/compromise tooling beyond the basic guard not yet built. [`docs/DESIGN.md`](https://github.com/AbubakarImam/Garbati.IdentityApi/blob/main/docs/DESIGN.md) |
| **ProductApi** (Product & Pricing) | Product catalog, account-to-product binding, product rules (minimum balance, allowed operations), fee and tax rules, limit definitions. Sharia product rules live here | Executing anything; storing money; customer identity | Not started |
| **OrchestratorApi** (Transaction Orchestrator) | The ordered steps of each customer operation; decision binding (`decisionId`); in-flight operation state; account-opening flow | Any truth of its own about customers, products or balances | Design v1 (minimal) ratified ([`Garbati.OrchestratorApi/docs/DESIGN.md`](https://github.com/AbubakarImam/Garbati.OrchestratorApi/blob/main/docs/DESIGN.md)); not yet built |
| **ScheduleApi** (Standing Orders) | Recurring instructions and their run history | Moving money (it submits requests to the orchestrator like any channel) | Later tranche |
| **Financing services** (Murabaha, etc.) | Financing contracts and schedules | Ledger postings | Later tranche |
| **Channel/Gateway** | Customer login, sessions, API for apps | Everything above | Later tranche |

## 3. Dependency rules

```
Channels / ScheduleApi
        |
        v
  OrchestratorApi  ----> IdentityApi   (who may do this?)
        |          ----> ProductApi    (what rules, fees, tax, limits apply?)
        v
    LedgerApi      (only the orchestrator posts money)
```

- Channels and ScheduleApi call **only** the orchestrator.
- The orchestrator calls IdentityApi, ProductApi and LedgerApi.
- IdentityApi and ProductApi never call each other. Anything needing both goes through the orchestrator.
- LedgerApi calls nobody (LedgerApi decision #34 stays true). It checks a decision token locally, using IdentityApi's public key (P-12).
- No service reads another service's database. No foreign keys across services.

## 4. End-to-end flows

**4.1 Customer transfer.** Channel authenticates the customer, then calls the orchestrator. The orchestrator asks IdentityApi for an authorization decision, asks ProductApi for rules, fees, tax and limits, and builds the full set of postings (the transfer, the fee, the tax). It posts them to LedgerApi with the `decisionId` bound to the exact request. The fee and tax are ordinary legs of the same chain (P-14), paid to system income and tax accounts. A transaction with no fee or tax is a chain of one leg.

**4.2 Account opening.** Orchestrator runs: create customer (Identity), wait for KYC, choose product (Product), create ledger account (Ledger), attach mandate (Identity), bind product (Product). If a later step fails, earlier steps are retried or reversed explicitly; no half-opened account stays silently valid.

**4.3 Standing order.** ScheduleApi fires on schedule and submits an ordinary request to the orchestrator, with idempotency key derived from the standing order id and the run date. A repeated trigger therefore cannot pay twice.

**4.4 Fee.** ProductApi computes the amount. The orchestrator posts it. ProductApi never touches the ledger.

## 5. Shared conventions (every service follows these)

- **Service authentication:** API key plus scopes, hash-only configuration, one client per calling service (LedgerApi decision #35).
- **Idempotency:** client-supplied key, unique per calling client, stored with a hash of the request; same key with a different body returns 409.
- **Audit:** written in the same database transaction as the change it describes, append-only.
- **Correlation id:** `X-Correlation-ID` on every call, passed on to every downstream call.
- **Errors:** 400 validation, 401 no/unknown key, 403 missing scope, 404 not found, 409 conflict, 422 business-rule failure, 500 unexpected; a timeout is never treated as approval.
- **Logs:** never contain identity numbers (BVN/NIN), documents, keys or full request bodies.
- **Versioning:** `/v1/` in the path; a breaking change creates `/v2/`.
- **Health checks:** `/health/live` and `/health/ready`.

## 6. Repository layout

One repository per service, each with `README.md` and `docs/DESIGN.md`. This repo (or a `garbati-platform` repo) holds only this file, the service map, and cross-service contracts.

```
garbati-platform/        ARCHITECTURE.md, contracts/, ADRs
LedgerApi/             docs/DESIGN.md   (exists)
Garbati.IdentityApi/     docs/DESIGN.md
Garbati.ProductApi/      docs/DESIGN.md
Garbati.OrchestratorApi/ docs/DESIGN.md
```

## 7. Build order (Proposed)

1. LedgerApi (done)
2. IdentityApi: design v2 ratified; next is building it
3. ProductApi: design first (the first Sharia-board review point is here)
4. OrchestratorApi: minimal design (Identity + Ledger only, no fee/reversal) ratified; next is building it. Product integration added once ProductApi exists.
5. ScheduleApi, Financing, Channel

**Deployment guidance:** design these as separate services, but you do not need separate deployments on day one. Each deployed service costs money and needs its own database, pipeline and monitoring. Start with Ledger and Identity deployed separately (their data sensitivity justifies it). Start Product and Orchestrator as separate projects inside one deployable, with the same boundaries enforced in code, and split them later if there is a reason.

## 8. Platform decisions log

| # | Decision | Options considered | Proposed choice | Why | Status |
|---|---|---|---|---|---|
| P-1 | Transaction flow owner | Channels call each service / Ledger calls Identity / separate orchestrator | Separate OrchestratorApi | One place enforces step order; the ledger stays independent; fixes the "nothing forces the gate" gap in Identity review | Decided (owner, D-2, 16 Oct 2026; matches the orchestration-based Saga pattern — `research/architecture-patterns-p1-p6.md`) |
| P-2 | Where product rules, fees, tax live | Inside IdentityApi / inside Orchestrator / separate ProductApi | Separate ProductApi | Identity answers "who", product answers "what rules"; they change for different reasons and on different schedules | Decided (owner, D-2, 16 Oct 2026; matches decide/execute separation seen in core-banking product-engine-vs-ledger splits — `research/architecture-patterns-p1-p6.md`) |
| P-3 | Standing orders | Inside Identity / inside Orchestrator / separate ScheduleApi | Separate ScheduleApi that calls the orchestrator | Scheduling needs its own run history and retry logic; money still moves only through the orchestrator | Decided (owner, D-2, 16 Oct 2026; matches the Scheduler/Agent/Supervisor pattern — `research/architecture-patterns-p1-p6.md`; retry-on-failure behaviour still open, see Section 9) |
| P-4 | Who computes vs who posts | Product posts fees / orchestrator posts | Product computes, orchestrator posts | Keeps "money moves only via the ledger" absolute | Decided (owner, D-2, 16 Oct 2026; same decide/execute split as P-2, applied per step — `research/architecture-patterns-p1-p6.md`) |
| P-5 | Cross-service data | Shared DB / copies / API calls only | API calls only, no shared DB, no cross-service FK | Services stay independently changeable | Decided (owner, D-2, 16 Oct 2026; matches the standard database-per-service pattern — `research/architecture-patterns-p1-p6.md`) |
| P-6 | Deployment shape | One service per deployment / modular monolith for new services | Separate design, minimal deployables initially | Solo developer and budget | Decided (owner, D-2, 16 Oct 2026; matches Fowler's "Monolith First" / modular-monolith guidance — `research/architecture-patterns-p1-p6.md`; later split not guaranteed easy, depends on keeping P-5's no-shared-database discipline intact in practice) |
| P-7 | Where account numbers are created | Move creation into IdentityApi / keep in LedgerApi | Keep in LedgerApi; the orchestrator runs the account-opening sequence (Section 10) | Only the ledger can guarantee its own account-number uniqueness; Identity would need ledger write access, so a breach of Identity could open accounts | Decided |
| P-8 | Aggregate limits | In orchestrator / in ledger / separate limits service | Orchestrator, using reserve, then confirm or release, with one atomic counter update | Orchestrator is the single gate every operation passes; Product owns the limit definitions | Decided (owner), mechanism proposed |
| P-9 | Fee and tax timing | Separate linked transactions / same ledger transaction | Same all-or-nothing unit as the transfer, built as a chain of legs committed in one database transaction (P-14); if the fee leg fails, the whole chain fails and is retried | No state where the transfer succeeded but the fee did not | Decided (owner); shape Decided (owner), option B |
| P-10 | Reversal of fees and tax | Reverse principal only / reverse everything | Reversing a chain reverses every leg, last leg first, in one database transaction. A single leg of a chain cannot be reversed on its own | Nobody can refund the principal and keep the fee | Decided (owner) |
| P-11 | Product changes | Edit in place / versioned | Immutable published versions; each account binds to a version; every fee records the version used; moving accounts to a new version is an explicit audited step | A past fee can always be explained | Decided (owner) |
| P-12 | Decision binding | Shared signing key / Identity signs, ledger verifies a public key / ledger confirms online with Identity | IdentityApi signs a decision token with a private key. The orchestrator verifies it, then the ledger verifies it again locally with the public key. Expiry 30 seconds, single use | No call from the ledger to Identity, so no added latency and no new dependency. The orchestrator cannot forge a decision. IdentityApi stays the source of truth for authorization, LedgerApi for financial transactions | Decided (owner); stale-window cost in Section 9 |
| P-13 | Create-account idempotency | None / client key with request hash | Idempotency key plus request hash on `POST /api/accounts` | A retried account-opening step must not create a second account | Decided (owner, D-2, 16 Oct 2026) — built, LedgerApi decision #43 (merged PR #24, 14 Oct 2026) |
| P-14 | Shape of multi-part transactions | One transaction with many entries (A) / chain of ordinary two-entry legs committed together (B) | B: a chain of legs. Each leg is a normal two-entry transaction (principal, fee, tax). All legs commit in one database transaction or none do. A transaction with no fee or tax is a chain of one leg | Keeps the tested guarantee of one debit and one credit per transaction, keeps `POST /api/transfers` unchanged, and lets reversal reuse the existing single-leg logic. Statements show one line per leg | Decided (owner); details in Section 12 |

## 9. Open questions

- **Token rules (P-12).** IdentityApi signs with ES256; only IdentityApi holds the private key. Claims: `iss`, `aud` (the ledger), `jti` (the `decisionId`), `iat`, `exp` (30 seconds), operation, debit account, principal amount, currency, and a hash of the exact request including the client's idempotency key. The orchestrator verifies signature, expiry and that the token matches the request it is about to send. The ledger verifies the same things again with the public key from configuration, pins the algorithm and issuer, allows about 5 seconds of clock difference, and reads a key id so keys can be rotated with two active at once.
- **Single use.** The ledger stores the `decisionId` with the posting under a unique constraint, in the same transaction. A second posting with the same token fails.
- **Retries.** The ledger looks up the idempotency key first. A retry of a request that already succeeded returns the stored result even if the token has since expired. Only a new key needs a valid token.
- **Failure behaviour.** A missing, malformed, expired or mismatched token fails closed: no posting. If IdentityApi is down, tokens already issued can still be verified, but no new customer operations can start.
- **Stale window.** A suspension or revocation within the 30 seconds does not cancel a token already issued, and the ledger cannot see it. This is the accepted in-flight window (Identity design 3.4), bounded by the expiry. IdentityApi marks expired grants as expired for audit; nothing needs revoking.
- **What the token cannot cover.** Fee and tax entries are added by the orchestrator from ProductApi's rules; the token covers only the customer's own request. A reconciliation job should compare posted fees with ProductApi's rules.
- **Orchestrator crash mid-operation.** A limit reservation was made but the ledger post never happened. Define reservation expiry and a reconciliation job that checks the ledger by idempotency key.
- **Does a reversal restore the limit allowance?** **Decided (owner, D-7, 16 Oct 2026): yes.** This still needs to be wired up when the limit-reservation mechanism (P-8) is actually built — nothing to restore yet, since limits aren't implemented.
- **Leg purpose codes.** **Decided (owner, D-7, 16 Oct 2026): `Principal`, `Fee`, `Tax` only, as built in decision #45.** A later code (e.g. `Charity` for Sharia late fees, pending T-007's research) is a new decision and a ledger code change, not an automatic extension.
- **Reversing when the recipient already spent the money.** **Decided (owner, D-7, 16 Oct 2026): the whole chain reversal fails; no manual remediation path defined yet.** Built and tested in decision #45 (`ChainReversal_WhenOneLegCannotBeReversed_RollsBackTheWholeReversal`).
- **Maximum legs per chain.** **Decided (owner, D-7, 16 Oct 2026): 10,** as built in decision #45.
- **Sharia rules.** Which product rules (profit-sharing instead of interest, late-fee-to-charity, no prohibited charges) are enforced by ProductApi and which by the later financing services?
- **Product version migration.** Moving accounts to a new version may need customer notice, especially for fee increases. Compliance must confirm the notice period.
- **Standing order retry-on-failure (P-3).** A real standing order (customer-initiated push payment) is traditionally *not* auto-retried on failure — unlike a direct debit, which can be re-presented by the payee's bank (`research/architecture-patterns-p1-p6.md`, Q3). ScheduleApi's design (T-010+ tranche) must decide: does a failed run (e.g. insufficient funds) get retried automatically, or just recorded and left for the customer to notice? Owner decision needed when ScheduleApi is designed, not now.

## 10. Account opening sequence (orchestrator-run)

| Step | Service | If it fails |
|---|---|---|
| 1. Create customer (pending) | IdentityApi | Nothing else exists; retry |
| 2. KYC verified | IdentityApi | Stop; customer stays pending |
| 3. Create ledger account (idempotency key) | LedgerApi | Retry with the same key; never creates a second account |
| 4. Attach mandate | IdentityApi | Account exists but cannot be used (no mandate); retry |
| 5. Bind product version | ProductApi | Same; retry |
| 6. Mark opening complete | Orchestrator | The operation record shows the current step |

## 11. Required LedgerApi amendments

1. **Request hash and per-client idempotency** (from the external review): **Done**, LedgerApi decisions #42 and #44 (merged 6 and 15 October 2026).
2. **Idempotency on `POST /api/accounts`** (P-13): **Done**, LedgerApi decision #43 (merged 14 October 2026). Keys are 1-128 characters of letters, digits, `-`, `_`, `.`, `:` (decision #44).
3. **Chains of legs** (P-9, P-10, P-14): a new chain endpoint posts an ordered list of legs in one database transaction. Every leg is still an ordinary two-entry transaction, so the "exactly one debit and one credit" rule is **not** amended. `TransferService` is split into a core that posts one leg inside a database transaction owned by the caller, and the existing wrapper. `ReversalService` keeps its single-leg logic and gains a chain reversal that loops over the legs. **Done**, LedgerApi decision #45 (built on `feat/ledger-chains`, merged as LedgerApi PR #26 on 15 Oct 2026 and PR #27 on 15 Oct 2026, ahead of this task board's QA step; independently QA-reviewed and approved 16 Oct 2026, `qa/reports/T-001.md`; owner-ratified 16 Oct 2026). Detail in Section 12.
4. **Fee-income and tax-payable system accounts per currency**, seeded like the funding accounts (decision #25). **Done**, LedgerApi decision #46 (merged as LedgerApi PR #28, 16 Oct 2026; QA-approved, `qa/reports/PR-28.md`). The accounts exist and are configured (`FeeIncome:Accounts`, `TaxPayable:Accounts`), but nothing routes a Fee/Tax leg to them yet — decision #45's chain validation still only checks `AccountClass == System`. Wiring a specific fee-income/tax-payable account into a transaction is future ProductApi/OrchestratorApi work.
5. **Token verification** (P-12): the ledger verifies an IdentityApi-signed decision token locally before posting a customer-originated transaction, and stores the `decisionId` under a unique constraint. This amends decision #34 narrowly: the ledger still calls nobody and stores nothing about identity. Proposed as decision #47.

## 12. Chain model for LedgerApi (draft of decision #45)

### 12.1 What it is, in plain words

A **chain** is a list of **legs** that the ledger posts together. Each leg is a normal LedgerApi transaction: one debit entry, one credit entry. The chain is the envelope that says "these legs belong together, and they either all happen or none do."

| Transaction | Legs |
|---|---|
| Transfer, no fee | 1: Principal |
| Transfer with a fee | 2: Principal, Fee |
| Transfer with a fee and tax on the fee | 3: Principal, Fee, Tax |

The ledger does not work out fees or tax and does not know which products have them. ProductApi computes, the orchestrator builds the list of legs, and the ledger posts exactly the list it is given (P-4).

### 12.2 Why all-or-nothing still holds

Each leg is posted inside **one** database transaction that the chain owns. If any leg fails, the database throws away every leg, exactly as it throws away the two entries of today's single transfer. Atomicity comes from the one database transaction, not from there being one transaction row. Legs are never committed one at a time.

### 12.3 Data shape

- New table `transaction_chains`: id, public reference, `initiated_by`, `idempotency_key`, `request_hash`, `status` (Pending, Success, Failed), `failure_reason`, `failed_sequence`, `reverses_chain_id` (unique, null unless this chain is a reversal), `created_at`, `completed_at`. `decision_id` (unique, from the token, P-12) is **deferred to decision #47** (token verification), not part of #45 — #45 shipped without it (D-6, token timing, is still open on the board; #45 simply does not need it yet).
- Unique index on (`initiated_by`, `idempotency_key`) with `NULLS NOT DISTINCT`, the same rule as transactions (decision #42).
- `transactions` gets three nullable columns: `chain_id`, `sequence`, `purpose`. Unique on (`chain_id`, `sequence`). Rows from before this change stay null and behave as today.
- `purpose` is a fixed list: `Principal`, `Fee`, `Tax` (confirmed by the owner, D-7, 16 Oct 2026).
- A leg's own idempotency key is generated by the ledger as `chain:{chainId}:{sequence}`, so it can never collide with a client's key. The single-transfer, deposit and withdrawal endpoints reject client keys that start with `chain:`.

### 12.4 Rules the ledger enforces on every chain

1. At least one leg, at most 10.
2. Sequences are `1..N` with no gaps. Leg 1 is the only `Principal` leg.
3. Every leg amount is greater than zero. A zero fee means no fee leg, and no fee means no tax leg. The caller never sends a zero leg.
4. Every leg uses the same currency, and each amount respects that currency's decimal places (the existing rule).
5. Fee and Tax legs must credit a System account (fee-income or tax-payable). The ledger checks the account class, not the amount.
6. **One balance check per paying account, up front.** For each customer account that pays any leg, the total of everything it pays across the chain (principal plus fee plus tax) must not exceed its balance. This gives one clear error ("Insufficient balance") instead of a principal that passes and a fee that fails. System accounts keep the existing overdraft floor.
7. The existing status rules apply per leg (source must be Active, destination must not be Blocked).

### 12.5 Locking

Before posting anything, the ledger locks **every account that is debited by any leg** with `SELECT ... FOR UPDATE`, in one fixed order (ascending account number), once for the whole chain. The fixed order stops two chains that touch the same accounts from waiting on each other forever (deadlock). Credited accounts are not locked, as today.

### 12.6 Failure and retry

- **Business failure** (insufficient funds, closed account, rule 1 to 7): the database transaction rolls back every leg. A second, small database transaction stores the chain as `Failed` with `failure_reason` and `failed_sequence` and writes the audit row. A retry with the same key and the same body returns that stored failure. To try again for real, use a new key. This matches today's decision #6 for single transfers.
- **Technical failure** (database down, crash): everything rolls back and nothing is stored. A retry with the same key runs again from the start.
- **Same key, different list of legs** (even a different fee): 409, as in decision #42.

### 12.7 Request hash

SHA-256 of the JSON of the ordered legs: for each leg, debit account, credit account, amount (normalised with `G29`) and purpose. Narration is excluded, as in decision #44.

### 12.8 Reversal

- `POST` a reversal against a **chain** reference. The ledger posts one reversing leg for each original leg, **last leg first** (Tax, then Fee, then Principal), all inside one database transaction. Each reversing leg uses the existing single-leg reversal logic, which keeps working because every leg still has exactly one debit and one credit. The `Single`/`First` assumptions in `ReversalService` stay valid.
- The reversal is stored as its own chain with `reverses_chain_id` set. A unique index on that column means a chain can be reversed only once (the same job `ix_reversals_original_transaction_id` does for a single transaction).
- **A leg that belongs to a chain cannot be reversed alone.** The old single-transaction reversal endpoint returns 422 "this transaction is part of chain X; reverse the chain". Without this rule someone could refund the principal and keep the fee.
- Locking: lock every account that the reversing legs debit (the original credit accounts), in the same ascending order as 12.5.
- Frozen accounts may be debited by a reversal (decision #38), applied per leg.
- If any reversing leg fails, the whole reversal rolls back.

### 12.9 Audit

In the same database transaction as the postings: one audit row per leg (so existing audit queries keep working) and one audit row for the chain as a whole.

### 12.10 Token (P-12)

The token binds the **principal leg and the idempotency key** (debit account, principal amount, currency, hash of the exact customer request). The ledger stores `decision_id` on the chain under a unique constraint. Fee and tax legs are added by the orchestrator from ProductApi's rules and are not covered by the token; the reconciliation job in Section 9 checks them.

### 12.11 Code changes (outline)

1. Split `TransferService`: `PostLegAsync` runs the checks and writes one leg inside a database transaction supplied by the caller. A rejected leg throws a business exception; it does not commit. The existing `TransferAsync` becomes a thin wrapper that opens the database transaction, calls `PostLegAsync` once and keeps today's behaviour and responses.
2. New `ChainService` that validates rules 1 to 6, locks accounts (12.5), calls `PostLegAsync` for each leg in order, and handles 12.6.
3. `ReversalService`: extract the single-leg reversal into a method that runs inside the caller's database transaction; add `ReverseChainAsync`; make the old endpoint refuse chain legs.
4. One EF migration for the new table and three columns, with a migration test like the earlier ones.
5. Responses for existing transaction reads gain optional `chainReference`, `sequence` and `purpose`.

### 12.12 Tests to write first

- One-leg chain gives the same balances and entries as a plain transfer.
- Three-leg chain where the fee leg is rejected leaves no entry from any leg, and the stored Failed chain returns on retry.
- Payer can afford the principal but not principal plus fee: fails with a clear message, nothing posted.
- Two chains touching the same accounts in opposite directions, run in parallel: no deadlock.
- Same key with a changed fee: 409.
- Chain reversal restores every balance; a second reversal returns the already-reversed error.
- Reversing a single leg of a chain: 422.
- A client key starting with `chain:` is rejected on the single-transfer endpoint.
