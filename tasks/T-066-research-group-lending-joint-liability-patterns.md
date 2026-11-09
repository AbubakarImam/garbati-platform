# T-066: Research — group lending / joint-liability patterns, conventional and Islamic

Owner: researcher. **Status: research only.** No ruling, no design, no schema, no code, no task
file for a future build. This builds on, and must not re-derive, `research/fineract-core-functionality-gap-analysis.md`
(T-059) Area 2 — read that section in full first (it's short). T-059 already established, at high
confidence: Fineract's real `Group`/`Center` entities let several clients jointly hold one savings or
loan account, with a `Center` as a scheduling container above `Group`; GARBATI has zero structural
equivalent anywhere (independently confirmed twice already); this gap matches GARBATI's current,
deliberate deposit-only scope (D-28) rather than an oversight; and group lending is historically
foundational to microfinance (Grameen Bank) but not the only live methodology today (individual
lending is a real, growing alternative, by Fineract's own account). Do not re-verify or restate these
findings — cite them and move on to what's genuinely new.

## Context: why this is research-only, not a design task

GARBATI has no financing/loan module at all yet — `Garbati.ProductApi` is deposit-only
(Savings/Current); `FeeRule`/`ProfitSharingTerms` exist as empty table shapes, deliberately unseeded,
because `research/sharia-product-rules.md` (T-007) found no qualified scholar has signed off on any
specific rule yet. D-28 (the financing module's initial contract-type scope — Murabaha, Musharaka,
Mudarabah, Qard Hasan, Wadiah, Ijarah, Istisna) is itself still gated on that same sign-off before any
of those uncontested structures get built. Group lending sits one layer *above* that — it's an
organizational/liability pattern that a loan product (any of the above contract types) could be
disbursed through, not a contract type itself. This task researches the pattern so a future financing-module
design has real input, the same role T-050 (Sharia rule configuration patterns) already plays for that
future design — it does not jump ahead of D-28's own gate.

## Research questions

1. **How does joint liability actually work mechanically in a conventional (Grameen-style) group loan?**
   When one group member defaults, what actually happens — do other members' own loan balances change,
   does the group's collective savings get drawn on, is it social pressure only with no automatic
   financial mechanism, or something else? Distinguish "legally/contractually joint liability" from
   "socially enforced repayment pressure with no formal joint-liability instrument" if real sources
   draw that distinction — don't assume they're the same thing.
2. **How do real Islamic/Sharia-compliant microfinance institutions structure group-based lending
   specifically?** Conventional joint liability is often paired with an implicit mutual-guarantee
   mechanism; Islamic finance has its own named concept for mutual guarantee (*kafala*) and mutual
   risk-pooling (*takaful*) — does real Islamic microfinance literature or a real named institution
   describe adapting group lending through one of these, or some other mechanism, to avoid structures
   that look like interest-bearing collective debt? **Flag any specific permissibility claim found as
   something a qualified scholar would need to confirm — do not render a Sharia opinion of your own, the
   same discipline T-007/T-050/D-28 already established.** If you find genuine disagreement or no
   clear convergence, say so plainly rather than picking a side.
3. **What does Fineract's real `Group`/`Center` data model actually look like, structurally?** T-059
   read the conceptual Mifos-forge wiki page; go deeper if a real, directly-fetchable source exists —
   Fineract's own database-table documentation (the same `fineract.apache.org/docs/database/tables/`
   path T-059/T-061 already used for other tables) for whatever group/center/joint-liability-loan
   tables exist, read directly if reachable. If blocked, say so plainly (same honest-blocker discipline
   every prior research brief in this project has used) rather than filling the gap with a
   search-engine summary presented as directly-read.
4. **Is group lending itself (the organizational pattern) separable from any specific loan contract
   type, or does it depend on which contract type is used?** E.g., would a Qard Hasan group loan and a
   Murabaha group loan need meaningfully different joint-liability mechanics, or is the group/liability
   layer genuinely contract-type-agnostic? This directly matters for whether a future "Group" concept
   could be designed once, generically, or would need to wait for a specific contract type to be chosen
   first.
5. **Structurally, which of GARBATI's existing service boundaries would a future group-lending design
   touch, and how?** Not a design — just naming the real tension points so a future design task has a
   running start. Specifically: does "several customers jointly liable for one loan balance" fit inside
   `IdentityApi`'s existing `Mandate`/joint-account concept (which is *authorization*, M-of-N approval
   on one account, not *liability*) — confirm directly against the real code whether these are
   structurally the same thing or genuinely different, the way T-059 itself already flagged this
   distinction. Does `LedgerApi`'s existing chain/leg model (decision #45, N legs in one atomic
   transaction) have any natural fit for "N debtors share responsibility for one balance," or does it
   require something new? You are not designing this — just scoping where the real seams are.

## Sourcing discipline (same standard as T-050/T-056/T-057/T-059)

- Distinguish directly-fetched sources from search-engine-summary sources explicitly, per source.
- State a confidence level per research question.
- If a primary source is blocked/unreadable (same `cbn.gov.ng`/`pdftoppm` class of issue prior briefs
  hit repeatedly), name the exact URL and failure mode plainly in a "What was attempted and blocked or
  unreadable" section — don't paper over it.
- End with "What this does not settle" and "What this affects" (name which `ARCHITECTURE.md` Section 9
  entry this extends — likely the existing Sharia-rules entry T-050 already extended, or a new one).

## What this task does not include

- No ruling on whether any specific group-lending mechanism is Sharia-compliant — that's a qualified
  scholar's call, same as every contract-type question D-28/T-007 already deferred.
- No schema, entity design, or API shape for a `Group`/`Center` concept in GARBATI.
- No task file for a future build or design task — if this research makes a strong case that a design
  round is worth scoping once the financing module itself is designed, say so in "What this affects,"
  but do not write that task file here.
- Does not reopen or re-verify T-059's own Area 2 findings — cite them, build past them.
