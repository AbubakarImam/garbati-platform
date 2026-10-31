# T-056: Research — how do real banks (especially microfinance branches) handle teller till/vault
cash operations and end-of-day balancing?

## Context

GARBATI's `StaffIdentityApi` has a `Teller` role (confirmed directly: `Entities/Role.cs`), and `OpsApi`'s
console has a "testing console" that can fund an account directly through a narrow `ledger.funding` scope
— but that is the full extent of it. Checked directly across every repo before writing this task
(`Till`/`Vault`/`CashDrawer`/`CashBalance`/`EndOfDay`/`EOD`, zero matches anywhere in code; zero mentions
in `ARCHITECTURE.md` or any service's own `DESIGN.md`): **GARBATI has no concept at all of a teller's own
cash position, a vault, cash movement between vault and till, or end-of-day cash balancing.** `Teller` is
currently just a role label with no cash-handling workflow attached to it. This is a real, structural gap
in a platform whose whole premise is serving branch-based microfinance institutions, where a human teller
handling physical cash is the normal, everyday front line — not an edge case.

This is forward-looking research, the same way T-026/T-054/T-055 researched real-world practice before
anything was designed, not a request to build anything yet.

## What to research

1. **How a real bank branch's cash custody chain actually works, end to end.** Vault (the branch's bulk
   cash holding, usually under dual-custody/joint control), till/cash drawer (an individual teller's own
   working cash, assigned to them for a shift), the cash-out process (a teller draws a starting float from
   the vault at shift start), and the cash-in process (the teller returns cash to the vault, e.g. above a
   threshold, mid-shift or at close). Who is authorized to do what, and what controls exist at each step
   (dual control on vault access, a teller never having unsupervised vault access, etc.)?
2. **What "the till must balance before close of business" actually means and requires, concretely.** How
   is a teller's expected till balance calculated (starting float + cash received from customers − cash
   paid out to customers, reconciled against the system's own transaction log for that teller/shift)? What
   is a "cash difference" / "teller shortage or overage," what's the real process when one is found (is a
   small variance tolerated, is a larger one escalated, who's accountable), and what does "balancing" mean
   operationally (a physical count reconciled against a system-calculated expected figure, not just a
   trust-based sign-off)?
3. **How this is typically modeled in a core banking system's own data model**, concretely — is a teller's
   till commonly represented as its own kind of account (a "teller till account" or "cash-in-hand account"
   in the GL sense), with vault-to-till and till-to-vault movements as real transactions between those
   accounts, and each customer cash transaction (deposit/withdrawal) also hitting the teller's own till
   account as one leg? Cite real, named sources (core-banking vendor documentation, banking-operations
   literature, published GL/chart-of-accounts examples) wherever possible, not generic blog summaries.
4. **What's different, if anything, for a microfinance institution specifically versus a full commercial
   bank** — smaller branches, lower cash volumes, possibly a single teller per branch with no separate
   vault custodian role, more frequent or simplified reconciliation. If real sources describe
   microfinance-specific cash-handling practice (Nigeria-specific if findable, given GARBATI's actual
   target market — state plainly if nothing Nigeria-specific is found and general microfinance/branch-
   banking literature is used instead), prioritize that over generic large-bank treasury-operations
   sources.
5. **How this maps onto GARBATI's own existing architecture — a structural comparison, not a design.**
   `LedgerApi` already has an `AccountClass` concept (`Customer`/`System`) with real system accounts (fee-
   income, tax-payable, per currency, decision #46) and a chain/leg-purpose mechanism (`LegPurpose`:
   `Principal`/`Fee`/`Tax`, decision #45). Does a till/vault model fit naturally as new system-account types
   and new leg purposes within GARBATI's existing mechanism, or does real-world till/vault practice need
   something GARBATI's current architecture doesn't have at all (e.g., a per-teller/per-shift scoped
   account that opens and closes, rather than a permanent system account)? Be honest either way — this is
   the question most directly useful to a future design task, so don't force-fit an answer that isn't
   actually supported by what you find in Q1-Q4.

## Explicitly out of scope

- Do not design a till/vault feature, propose a schema, or write a task file for building it — this is
  research only, informing a possible future task the same way T-026 informed T-027/T-028.
- Do not assume this is urgent or propose a timeline — GARBATI has no branch-deployment customer yet;
  this fills a real gap in the platform's own self-understanding, not a response to an active deadline.
- Do not conflate this with `OpsApi`'s existing `ledger.funding` scope (a narrow, already-built exception
  for direct account funding in the testing console) — note it as the closest existing mechanism if
  relevant, but the research question is about real teller/vault cash custody, which that mechanism was
  never designed to model.

## Output

A sourced brief at `research/teller-till-vault-cash-operations.md`, following this project's existing
research-brief conventions (see `research/reconciliation-nibss-integration-patterns.md` and
`research/modern-treasury-gap-analysis.md` for the most recent examples of tone, sourcing discipline, and
how to honestly flag what couldn't be confirmed). End with an honest assessment of whether GARBATI's
existing `AccountClass`/`LegPurpose` mechanism is structurally suited to model this, or whether real
till/vault practice needs a genuinely new mechanism — framed as input to a future owner decision, not a
decision itself.
