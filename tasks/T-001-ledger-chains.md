# T-001: LedgerApi chains of legs (decision #45)

Target repo: LedgerApi. Branch: `feat/ledger-chains`. Spec detail: ARCHITECTURE.md Section 12.

---

Implement **LedgerApi decision #45: chains of legs**. Branch + PR as usual; add #45 to `docs/DESIGN.md`. Out of scope: token verification (#47), fee/tax system accounts (#46).

**Idea.** A chain is an ordered list of legs posted together. Each leg is an ordinary two-entry transaction (one debit, one credit). All legs commit in ONE database transaction or none do. No fee/tax means a chain of one leg, which behaves like today's transfer. The one-debit-one-credit rule does not change.

**Data (one migration + migration test)**
- `transaction_chains`: id, unique reference, `initiated_by`, `idempotency_key`, `request_hash`, status (Pending/Success/Failed), `failure_reason`, `failed_sequence`, `reverses_chain_id` (nullable, unique), created/completed timestamps. Unique (`initiated_by`, `idempotency_key`) `NULLS NOT DISTINCT`, named index, caught by constraint name.
- `transactions`: nullable `chain_id`, `sequence`, `purpose` (`Principal`/`Fee`/`Tax`); unique (`chain_id`, `sequence`). Old rows stay null.
- Leg keys are generated: `chain:{chainId}:{sequence}`. Reject client keys starting with `chain:` on transfer, deposit, withdrawal and create-account (400).

**Endpoints**
- `POST /api/chains` (`ledger.transfer`): `idempotencyKey`, `narration?`, `legs[]` of debit, credit, amount, purpose. Sequence = array position. 400 if: 0 or more than 10 legs, first leg not `Principal` or a second `Principal`, any amount <= 0.
- `GET /api/chains/{reference}` (`ledger.read`).
- `POST /api/chains/{reference}/reversals` (`ledger.reverse`).
- Existing transaction responses gain optional `chainReference`, `sequence`, `purpose`.

**Posting rules**
- Same currency on all legs and currency decimal places (existing checks). `Fee`/`Tax` legs must credit a System account. Per-leg status rules as today.
- ONE balance check per paying account up front: total it pays across all legs (principal + fee + tax) must fit its balance; system overdraft floor as today.
- Lock every debited account once, deduplicated, ascending account number, before posting anything.
- Request hash: SHA-256 of the ordered legs as `[debit, credit, amount G29, purpose]`, narration excluded. Insert-first on the chain row; same key + same hash returns the stored result, different hash is 409.
- Refactor, don't copy: split `TransferService` into a `PostLegAsync` that runs inside a caller-owned DB transaction and throws on rejection without committing; `TransferAsync` becomes a thin wrapper with unchanged behaviour.

**Failure.** Business failure: roll back every leg, then a second small transaction stores the chain as Failed (reason + `failed_sequence`) with its audit row; same-key retry returns it (like #6). Technical failure: roll back, store nothing, rethrow (`CancellationToken.None` for rollback).

**Audit.** Same DB transaction as the postings: one row per leg (existing transfer action) plus one `ChainPosted` row.

**Reversal.** One reversing leg per original leg, last leg first, in one DB transaction, reusing the single-leg logic (extract it to run inside a caller-owned transaction). Stored as a chain with `reverses_chain_id`; second reversal gives the existing already-reversed error. Lock the accounts the reversing legs debit, ascending order. Frozen accounts may be debited (#38). Any leg failing rolls back the whole reversal. **The old single-reversal endpoint returns 422 for any transaction with a `chain_id`** ("part of chain X; reverse the chain").

**Tests**
- One-leg chain equals a plain transfer.
- Three legs post six entries with correct balances.
- A rejected middle leg leaves nothing posted; the Failed chain is stored and a retry returns it.
- Payer can afford the principal but not principal + fee + tax: fails, nothing posted.
- Same key with a changed fee: 409.
- Opposite-direction parallel chains over the same accounts: no deadlock.
- Chain reversal restores all balances, in Tax, Fee, Principal order; a second reversal is refused.
- A single leg reversed through the old endpoint: 422.
- `chain:` client key rejected on all four endpoints.
- Bad leg lists rejected: 0 legs, 11 legs, two `Principal`, a zero amount, mixed currencies.
- Migration leaves old rows valid and existing tests pass unchanged.

Run build and tests; report anything unverified or ambiguous.
