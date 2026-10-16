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

### 2026-10-16: Service boundaries ratified — P-1 to P-6 (garbati-platform, no PR — a documentation decision)
**In plain words:** `ARCHITECTURE.md` draws the map of which future service does what: a traffic-cop `OrchestratorApi` that runs the steps of a transaction in order; a `ProductApi` that only *decides* fees/rules while the orchestrator is the only thing that actually *moves* money through `LedgerApi`; a `ScheduleApi` for recurring payments; no service ever reading another's database directly; and, for now, fewer deployed services than designed services to save cost. Before locking these in, a researcher checked each one against real, named industry patterns rather than taking the recommendation on faith.

**Five interview questions:**
1. Q: What is a "Saga" in distributed systems, and which style did GARBATI pick?
   A: A Saga is how you keep a multi-step operation consistent when it spans several services that each have their own database — instead of one big database transaction, each step commits on its own, and if a later step fails, earlier steps are deliberately undone by code written for that purpose (a "compensating transaction"), not automatically by the database. GARBATI picked the **orchestration** style: one component (the Orchestrator) is in charge of calling every step in order, rather than services reacting to each other's events with nobody in overall charge (**choreography**).
2. Q: Why split "who decides the fee" (ProductApi) from "who actually posts it" (via the Orchestrator, to LedgerApi)?
   A: So the two can change independently — pricing changes constantly and shouldn't need a ledger release, and real core-banking vendors (Temenos, Mambu) draw this same line between a product/pricing engine and the posting engine. If the ledger absorbed pricing logic, every product tweak would risk touching the one piece of code that must stay simple and trustworthy.
3. Q: What's the real cost of "no shared database between services," and is it worth it?
   A: You lose the easy stuff — a single SQL query that joins data across two services, or one database transaction spanning two services' data. You have to call another service's API instead, and accept that data might be briefly out of sync (eventual consistency). The benefit: each service can change its own schema without breaking anyone else, and for a ledger specifically, nobody else can quietly corrupt its one authoritative copy of the truth.
4. Q: GARBATI is designing four services but deploying only two for now. Is that a shortcut that will bite later?
   A: It's a recognized approach (Martin Fowler's "monolith first" / modular monolith) — build with clean internal boundaries, deploy fewer actual running services while the system is small, split later once there's a real reason. The honest caveat, straight from Fowler: splitting it later isn't guaranteed to be easy — it only stays easy if the "no shared database, API-only" discipline is actually followed in the code the whole time, not just on paper.
5. Q: Did the research prove GARBATI's design is correct?
   A: No — it proved the design isn't inventing anything unusual; each piece has a name and documented trade-offs elsewhere. It didn't (and couldn't) prove GARBATI specifically will work out, and it explicitly doesn't touch Sharia-compliance or Nigerian-regulatory questions, which still need a scholar and a compliance officer respectively.

**One thing that nearly went wrong:** The recommendation had already been written into `ARCHITECTURE.md` before this round, and it would have been easy to just ask the owner to rubber-stamp it. Instead, a researcher was sent to check each of the six proposed boundaries against real, citable sources first (Martin Fowler, microservices.io, Microsoft's own .NET architecture guide, Azure/AWS architecture docs) — confirming they match standard practice rather than trusting the plan on paper. That surfaced one real gap worth tracking: standing orders in the real world are usually *not* auto-retried on failure (unlike direct debits), which GARBATI's `ScheduleApi` design (not built yet) will need to decide explicitly — now recorded as an open question in `ARCHITECTURE.md` Section 9 rather than silently assumed.
