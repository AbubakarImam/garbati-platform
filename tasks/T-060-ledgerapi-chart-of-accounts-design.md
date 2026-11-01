# T-060: LedgerApi chart-of-accounts design (D-30)

Owner: coordinator + you. This is a design task — it produces (or revises) a section of
`LedgerApi/docs/DESIGN.md` and a new decision-log entry, the same way T-006/T-009/T-010/T-027 designed
Identity v2, ProductApi, OrchestratorApi, and StaffIdentityApi before any of them were built. **No code
gets written from this task directly** — once the design is reviewed and ratified, a separate build task
(or tasks) gets dispatched to a builder, the same two-step pattern every other service has followed.

## Context

`research/fineract-core-functionality-gap-analysis.md` (T-059, Area 5) found `LedgerApi` is "different by
design" from a real general-ledger/chart-of-accounts system — a deliberately narrow transactional posting
engine (`AccountClass`: `Customer`/`System` only, three fixed per-currency system-account config sections,
a closed three-value `LegPurpose` enum) rather than something that can categorize accounts for financial
reporting (Asset/Liability/Equity/Income/Expense, hierarchical header/detail accounts, a configurable
product-to-GL mapping mechanism) the way Fineract and Oracle FLEXCUBE both can. The owner decided (D-30)
to close this gap for real: a full chart of accounts, designed first before any code changes to a
live, deployed, production ledger service.

**The real precedent already in `LedgerApi`'s own decisions log, read directly, not assumed**: decision
#46 (fee-income/tax-payable system accounts) exists specifically "to keep fee income and tax payable
separately reportable per currency" — the same underlying goal a chart of accounts exists to serve, built
today as two hardcoded `IOptions` classes (`FeeIncomeOptions`, `TaxPayableOptions`) rather than a real,
queryable, extensible category system. This design task generalizes that already-proven instinct, it does
not invent a new one.

**The one hard constraint this design must respect, confirmed directly from the decisions log**: decision
#34, "LedgerApi calls nobody" — it checks a decision token locally and posts what it's given; no outbound
dependency on `IdentityApi`/`ProductApi`/`OrchestratorApi` exists today, and P-5 (`ARCHITECTURE.md`)
forbids a shared database or duplicating another service's data inside `LedgerApi`'s own. T-059's own Area
5 finding named this precisely: "Neither check can live purely inside LedgerApi without either breaking
decision #34... or duplicating another service's data inside LedgerApi's own database." This design must
be something `LedgerApi` can own entirely on its own data, the same way decision #46's accounts are
`LedgerApi`'s own rows, not a cross-service lookup.

## What this design must resolve (draft proposal below, for review and revision, not final)

1. **A new `GlAccount` entity** — the chart-of-accounts category itself, distinct from the existing
   `Account` entity (which remains the real, individually-addressable, balance-holding posting target —
   nothing about how money posts, locks, or derives its balance changes). Proposed shape: `Id`, `Code` (a
   real GL account code, e.g. `"1000"`), `Name` (e.g. `"Cash and Cash Equivalents"`, `"Fee Income"`),
   `AccountType` (a closed enum: `Asset`/`Liability`/`Equity`/`Income`/`Expense` — the standard five
   categories, confirmed as the real, near-universal shape by T-059's own Fineract reading), `ParentGlAccountId`
   (nullable, for a header/detail hierarchy — a header account aggregates its children's balances for
   reporting and is never itself a posting target; a detail/leaf account is what a real `Account` row tags
   itself under).
2. **Tag `Account` with its `GlAccountId`.** A new, nullable-at-migration-time foreign key on the existing
   `Account` entity. Existing accounts (the 20 funding accounts per decision #25, the 20 fee-income + 20
   tax-payable accounts per decision #46, and every already-opened customer account) need a real migration
   decision: leave existing rows `NULL` until explicitly categorized, or backfill sensible defaults at
   migration time (funding accounts → an Asset/Liability GL account depending on direction; fee-income
   accounts → an Income GL account; tax-payable accounts → a Liability GL account; customer accounts → a
   single default "Customer Deposits" Liability GL account, matching real banking practice where customer
   balances are the institution's own liability). **This specific choice — leave NULL vs. backfill — needs
   the owner's explicit call in review, not a default baked in here.**
3. **A new chart-of-accounts admin surface**, mirroring `SystemAccountsController`'s existing pattern (same
   admin-scoped shape, not a new authorization model): create/list `GlAccount` rows, and extend the
   existing system-account creation endpoint to accept an optional `GlAccountId` at creation time.
4. **A new report endpoint** — the actual payoff of this whole design: something like `GET
   /api/reports/trial-balance`, aggregating every account's own derived balance (decision #1's existing
   mechanism, unchanged — `SUM` over `LedgerEntries`), grouped by `GlAccount`, rolling up through the
   header/detail hierarchy. This is a pure, local, `LedgerApi`-internal computation — no new dependency,
   no violation of decision #34.
5. **Explicitly, deliberately out of this design**: a product-to-GL *mapping rule* mechanism (i.e.,
   "ProductApi's `OperationType` X automatically routes to GL account Y") stays outside `LedgerApi` — per
   decision #34/#46's own established pattern, the *caller* (`OrchestratorApi`/`ProductApi`, which already
   decide which specific fee-income/tax-payable account to credit today, per decision #46's own note) is
   who would supply a `GlAccountId` when creating a new system account, the same way it already supplies
   which account to credit for a fee leg. `LedgerApi` owns the categorization and the reporting; it does
   not own the business logic deciding which category a given product/operation maps to.

## What this design explicitly does not change

- **Decision #1 (derived balance).** Unchanged — `GlAccount` is a reporting/categorization layer on top of
  the existing balance computation, not a replacement for it.
- **Decision #7 (row locking) and decision #45 (chains/legs).** Completely unaffected — a chain still
  posts `Principal`/`Fee`/`Tax` legs exactly as today; no new `LegPurpose` value is needed for this
  specific design.
- **Decision #34 ("LedgerApi calls nobody") and P-5 (no shared database).** Respected by construction —
  everything above is `LedgerApi`'s own data, set by a caller at account-creation time, not looked up
  cross-service.

## Process

1. Coordinator drafts the full design (the section above, expanded into a real `docs/DESIGN.md` addition
   and a new decisions-log entry, numbered correctly against `LedgerApi`'s own real decision sequence —
   #46 is the last one currently in the log, confirm the actual next number before writing anything) and
   presents it for review in the same conversation D-30 was decided in, before committing anything.
2. Owner reviews, asks questions, requests changes — the same review cycle D-9 (Identity design v2) went
   through before being finalized.
3. Once ratified, this task file is updated to `done`, and a separate build task (a new T-number) is
   written and dispatched to a builder, the same two-step pattern T-009→T-045 and T-027→T-032 already
   used.

## Out of scope for this task

- No code changes. This task produces a design document and a decision, nothing else.
- No change to any other service (`ProductApi`, `OrchestratorApi`) in this task — if the eventual build
  needs those services to start supplying a `GlAccountId` at account-creation time, that's scoped
  separately once this design is ratified.
