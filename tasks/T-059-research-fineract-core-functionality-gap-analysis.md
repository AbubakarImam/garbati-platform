# T-059: Research — full gap analysis of GARBATI against Apache Fineract's core functionality

## Context

Apache Fineract has come up repeatedly this project — it's GARBATI's own Apache 2.0 license precedent
(D-21), and three prior research tasks have each touched a narrow slice of it directly: teller/cashier
endpoints and `CASH_AT_MAINVAULT`/`CASH_AT_TELLER` GL mapping (`research/teller-till-vault-cash-operations.md`,
T-056), the `TaxGroup`-on-`Charge` compounding mechanism (`research/multi-charge-transaction-modeling.md`,
T-057), and the `acc_gl_journal_entry` schema plus maker/checker audit fields
(`research/real-corebanking-payload-gap-analysis.md`, T-058). **This task does not re-derive any of those
three findings** — it builds on them and fills in the much larger remaining surface: a full, structured
comparison of GARBATI against Fineract's *core* functionality, the way `research/modern-treasury-gap-analysis.md`
(T-055) did for Modern Treasury.

**One thing worth flagging before the research starts, confirmed directly by the coordinator**: Fineract has
an explicit **Groups**/**Centers** concept — the group-lending/center-meeting model common to real
microfinance institutions (several borrowers jointly responsible for a loan, organized into a group, groups
organized into a center with scheduled meetings). A repo-wide search across every GARBATI service found
zero matches for any such concept. This is the one genuinely *microfinance-specific* banking pattern (as
opposed to a generic commercial-bank pattern) in this comparison, and deserves real attention, not just a
passing mention.

## What to research — Fineract's core functionality areas, each compared against GARBATI's real current
code, not against GARBATI's own documentation about itself

For each area below: confirm what Fineract actually has (read Fineract's own documentation/API reference
directly, not a secondary summary, wherever reachable), confirm what GARBATI actually has by reading the
real service code (not `ARCHITECTURE.md`'s own description of itself — matching this project's "verify,
don't trust" discipline, `AGENTS.md`), and mark the verdict as one of: **GARBATI has a real equivalent**,
**genuinely missing, worth considering**, or **different by design** (with the real reason stated).

1. **Client/Customer management and KYC.** Fineract's `Client` entity, its identity-document model, and its
   KYC/status lifecycle, compared against `Garbati.IdentityApi`'s `Customer`/`CustomerIdentityData`/
   `KycVerification`/`KycTierPolicy` (already built — confirm what's real there directly, per decisions
   #11/#12/#16, D-26).
2. **Groups and Centers (group lending).** As flagged above — Fineract's own documented model for
   collective/joint-liability lending and center-meeting scheduling. Confirm GARBATI has nothing like this
   (already strongly suspected, confirm it isn't missed), and characterize honestly whether this is a real,
   near-universal microfinance pattern or specific to certain lending models/regions, since GARBATI's own
   scope today is deposit-side only (Savings/Current) with financing deliberately deferred (D-28).
3. **Savings/deposit products.** Fineract's savings-product configuration (interest posting, minimum
   balance, withdrawal rules) compared against `Garbati.ProductApi`'s `Product`/`ProductLimitPolicy`/
   `AccountProductBinding` (already built — read directly) and `LedgerApi`'s account model.
4. **Loan/financing products.** Fineract's loan-product configuration (disbursement, repayment schedules,
   interest calculation methods) compared against GARBATI's current state (no financing module exists yet —
   D-28 scoped the eventual contract types to Murabaha/Musharaka/Mudarabah/Qard Hasan/Wadiah/Ijarah/Istisna,
   but nothing is built). This comparison is necessarily about Fineract's real shape versus GARBATI's
   *absence* here, not a feature-for-feature match — be explicit about that rather than forcing one.
5. **Accounting/GL and journal entries.** Build on, don't repeat, T-058's own direct read of
   `acc_gl_journal_entry`. What this task should add: Fineract's chart-of-accounts model and how products
   map to GL accounts (`PaymentType` GL routing, already touched in T-058, plus the broader
   product-to-GL-account mapping concept) compared against GARBATI's own `AccountClass`/system-account
   model (decisions #25/#46).
6. **Roles, permissions, and maker-checker.** Build on, don't repeat, T-058's own finding on Fineract's
   `maker`/`checker` audit fields. What this task should add: Fineract's actual role/permission model
   (how granular permissions are defined and assigned) compared against GARBATI's own scope-based
   authorization (`identity.customer.write`-style scopes) and `StaffIdentityApi`'s role model.
7. **Standing instructions / recurring transactions.** Fineract's own standing-instruction feature compared
   against GARBATI's still-undesigned `ScheduleApi` ("later tranche," per `ARCHITECTURE.md`'s build order) —
   again, necessarily a real-shape-vs-absence comparison; note what Fineract's real implementation would
   usefully inform once `ScheduleApi` is actually designed, without designing it here.
8. **Multi-tenancy.** Fineract is documented as explicitly multi-tenant (a single deployment serving
   multiple separate institutions/tenants). This connects directly to GARBATI's own still-open multi-tenancy
   question (`research/sharia-rule-configuration-patterns.md`, T-050, and `ARCHITECTURE.md` Section 9).
   Confirm how Fineract actually implements tenant isolation (separate databases per tenant vs. a shared
   database with a tenant key) and report this as real, concrete input to that already-open GARBATI question
   — not a new question, a data point for an existing one.
9. **Interest/profit calculation.** How does Fineract compute and post interest (or, if documented, any
   profit-sharing-adjacent mechanism) — compared against GARBATI's current, deliberately-empty
   `ProfitSharingTerms` table (never seeded, same Sharia-scholar-sign-off gate as `FeeRule`). State plainly
   that nothing here is a Sharia ruling or a suggestion to seed that table — this is purely a structural
   "how would the computation and posting mechanism work, if GARBATI's own scholar-approved rules existed"
   question.
10. **Reporting.** What does Fineract's own reporting capability look like (built-in reports, a reporting
    API) compared against GARBATI's current complete absence of any reporting layer? Keep this brief — it's
    a real gap, but likely a shallow one to characterize (GARBATI has nothing; name what Fineract has).

## Explicitly out of scope

- Do not re-derive T-056/T-057/T-058's own already-established Fineract findings (teller/cashier,
  TaxGroup-on-Charge, journal-entry schema, maker/checker audit fields) — cite and build on them.
- Do not design any feature, propose a schema, or write a task file for closing any gap found — this is a
  structural comparison only, the same discipline T-055 held for Modern Treasury.
- Do not propose or imply any specific Sharia ruling, interest rate, or profit-sharing ratio — Q9
  specifically is about mechanism, not content, exactly as `research/sharia-product-rules.md` and
  `research/sharia-rule-configuration-patterns.md` already established for this whole area of the project.
- Do not assume Fineract's own choices are automatically the "correct" standard GARBATI should match — some
  of what Fineract has may be different by design (a different target market, a different regulatory
  environment) and the research should say so honestly where that's the finding, the same way T-055 found
  for several Modern Treasury concepts.

## Output

A sourced brief at `research/fineract-core-functionality-gap-analysis.md`, following this project's
existing research-brief conventions (see `research/real-corebanking-payload-gap-analysis.md` for the most
recent example of tone, sourcing discipline, and honest confidence-calibration/uncertainty-flagging — match
that standard). Structure it around the ten areas above (or however many Fineract's real site structure
actually supports covering — confirm the real structure directly rather than assuming this list is
complete or in the right order). End with a summary table (equivalent / genuinely missing / different by
design, per area) the way T-055 did, and a short synthesis naming which gaps look most relevant to
GARBATI's own stated microfinance focus specifically, not a generic "Fineract has more features" conclusion.
