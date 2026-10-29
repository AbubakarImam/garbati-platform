# T-054: Research — how do real banks handle reconciliation, and what does real NIBSS integration
actually look like?

## Context

GARBATI has referenced a "reconciliation job" as a real need in at least three places without ever
researching or designing one: `ARCHITECTURE.md` Section 5/9 names a reconciliation job to "compare
posted fees with ProductApi's rules" and another to catch an orchestrator crash mid-operation ("a
reservation was made but the ledger post never happened... a reconciliation job that checks the ledger
by idempotency key"); Section 8's grant-token design explicitly says "the reconciliation job in Section 9
checks them" for fee/tax legs the signed grant doesn't cover. T-037's own OSS-readiness research (G12)
separately found that NIBSS (Nigeria's interbank settlement operator) is actively migrating to ISO 20022
via its "National Payment Stack" (live rollout as of October 2026) — correctly scoped out of GARBATI's
current build (no live interbank integration exists yet), but flagged as a fact to track, not researched
further.

GARBATI today only moves money *within* its own ledger — customer-to-customer, inside one platform. It
has no real bank-to-bank settlement rail integration at all. This research is forward-looking, informing
a future service/capability (reconciliation, and eventually real NIBSS integration), the same way
T-026/T-041/T-050 researched StaffIdentityApi, Nigeria hosting, and Sharia-rule configurability before
any of those were designed or built.

## What to research

1. **How real banks structure reconciliation generally.** What's the actual shape of a reconciliation
   process at a real financial institution — end-of-day vs. intraday reconciliation, what a "break"
   (unmatched entry) actually is, how matching keys/tolerance windows work, auto-match vs. manual
   exception handling, and where reconciliation sits organizationally (back-office/ops function vs. a
   core-banking-system feature). Cite real, named sources — vendor documentation, regulatory guidance,
   accounting/treasury operations literature — not generic blog summaries.
2. **Nostro/vostro and interbank-specific reconciliation.** How does reconciliation differ when money
   actually crosses between institutions (interbank), versus GARBATI's current intra-platform-only
   transfers? What's a suspense account, when does one get used, and how does a real institution clear
   one down over time?
3. **What NIBSS integration actually looks like for a Nigerian financial institution, concretely.** NIP
   (NIBSS Instant Payment), direct debit, and the newer National Payment Stack/ISO 20022 migration T-037
   already found — what does the real integration shape look like (message formats, settlement cycles,
   who initiates vs. who confirms, typical cut-off/settlement windows)? How does an institution reconcile
   its own ledger against NIBSS's own settlement reports/files at end of day? Read NIBSS's own public
   documentation directly where available, not just secondary summaries — and state plainly whichever
   parts are blocked/unavailable (T-037/T-041/T-043 have each hit real primary-source access problems
   with Nigerian financial-regulatory bodies; expect the same here and say so rather than guess).
4. **How this maps onto GARBATI's own three already-named reconciliation needs**, specifically — not
   generically. For each of the three gaps named in `ARCHITECTURE.md` (fee/tax posting vs. ProductApi's
   rules; orchestrator-crash-mid-operation idempotency-key check; the grant-token's own "fee/tax legs
   aren't covered, the reconciliation job checks them" note), is the real-world pattern found in Q1-Q3
   actually the right shape for that specific gap, or does GARBATI's situation (an orchestrated, chain-
   based ledger with idempotency keys already built in) suggest something simpler than what a real,
   larger institution needs? Be honest about where GARBATI's existing architecture (decision #1's
   append-only ledger, decision #45's atomic chains) already does some of reconciliation's job
   structurally, rather than assuming a textbook reconciliation service is needed wholesale.
5. **What a future `ReconciliationService` (or equivalent) would need to own**, as a structural question
   only — not a schema or task file. Does this look like its own service (a new "later tranche" addition
   alongside `ScheduleApi`/financing/`Channel`), or does it belong inside an existing service (e.g.
   `LedgerApi` itself, since it already owns the one authoritative ledger)? What's the real argument either
   way, grounded in what was found in Q1-Q4, not a generic microservices-boundaries opinion.

## Explicitly out of scope

- Do not design a `ReconciliationService` or propose a schema/API. This is research informing a future
  design task, the same way T-026 informed T-027/T-028 rather than building them itself.
- Do not assume NIBSS integration is imminent or propose a timeline — GARBATI has no live interbank
  integration today and this research doesn't change that; it's preparatory.
- Do not re-research ISO 20022 generically — T-037's G12 finding already covers the "NIBSS is migrating to
  it" fact; build on that rather than re-deriving it.

## Output

A sourced brief at `research/reconciliation-nibss-integration-patterns.md`, following this project's
existing research-brief conventions (see `research/reversal-authority-patterns.md` and
`research/oss-core-banking-readiness-gaps.md` for tone/structure/sourcing discipline — cite real, named,
checkable sources, state plainly wherever something couldn't be confirmed). End with an honest assessment
of whether GARBATI's three already-named reconciliation gaps need a dedicated service or fit inside
`LedgerApi`'s existing ownership of the one authoritative ledger — framed as input to a future owner
decision, not a decision itself.
