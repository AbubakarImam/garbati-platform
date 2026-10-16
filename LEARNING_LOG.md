# Learning log

One entry per merged feature. You should be able to answer these in an interview without notes.

## Template
### <date>: <feature> (<repo>, PR #n)
**In plain words:** what it does and why it exists, as you would tell a friend.
**Five interview questions:**
1. Q / A
2. Q / A
3. Q / A
4. Q / A
5. Q / A
**One thing that nearly went wrong:** a trade-off or bug and how it was handled.

## Entries

### 2026-10-16: Chains of legs (LedgerApi, PR #26 + #27)
**In plain words:** Before this, every money movement in the ledger was exactly one debit and one credit — a transfer. But a real transfer often needs to also take a fee, and tax on that fee, as part of the *same* all-or-nothing operation: either the transfer, the fee, and the tax all happen, or none of them do. A "chain" is just an ordered list of up to 10 of these two-entry movements (called "legs") that the ledger posts together inside one database transaction. A plain transfer is just a chain with one leg, so nothing that existed before had to change.

**Five interview questions:**
1. Q: Why is each payer's balance checked once at the start of the chain instead of before every leg?
   A: So the customer gets one clear answer ("insufficient balance") covering everything they're paying in the chain — principal plus fee plus tax — instead of the principal succeeding and the fee failing separately, which would be confusing and leave a half-finished-looking result.
2. Q: Two chains run at the same time, touching the same two accounts in opposite directions. What stops them deadlocking?
   A: Before posting anything, each chain locks every account it's about to debit, once each, always in the same fixed order (ascending account number — not the order the legs happen to be listed). Since every chain agrees on that order, one chain always gets there first and the other just waits, instead of each one holding a lock the other needs.
3. Q: If the second leg of a three-leg chain fails, what happens to the other two?
   A: Nothing from any leg is kept — the database throws away the whole attempt, as if it never happened. Then a separate, second save records that the chain failed, on which leg, and why, so a retry with the same idempotency key returns that stored failure instead of silently trying again.
4. Q: Why does reversing a chain undo tax, then fee, then the principal — backwards from how it was posted?
   A: The fee and tax only exist because of the principal, so logically they get undone first. It also means if undoing the principal fails partway (say the recipient already spent it), the whole reversal rolls back together — you can't end up in a state where the fee was refunded but the principal wasn't, or vice versa.
5. Q: Why is a client blocked from using an idempotency key that starts with `chain:`?
   A: The ledger generates its own keys in that exact shape (`chain:{chainId}:{sequence}`) for the individual legs inside a chain. If a customer-chosen key could also look like that, there's a theoretical risk of two different things colliding under the same key, so it's simplest to just forbid that prefix everywhere a client supplies their own key.

**One thing that nearly went wrong:** This feature was actually built and merged straight to `main` (PR #26, then a follow-up fix in PR #27) *before* the task-board/QA process in this repo existed. It worked and CI passed, but there was no written QA review and no learning-log entry — exactly the kind of silent gap the process is meant to prevent. The coordinator caught this by reading the task file (which already had a builder's self-audit note at the bottom), ran an independent QA review against the merged code anyway, and asked the owner to formally ratify the choices that had already shipped (leg types limited to Principal/Fee/Tax, 10-leg max, a failed reversal cancels the whole thing, a reversal gives back the spending limit) rather than rubber-stamping it. QA also caught one real documentation bug: `ARCHITECTURE.md` still listed a `decision_id` column as part of this feature's data shape, but that column actually belongs to a *different*, not-yet-built decision (#47, the security token) — now fixed in the doc.
