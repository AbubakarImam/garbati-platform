# T-004: LedgerApi fee-income and tax-payable system accounts (decision #46)

Target repo: LedgerApi. Branch: `feat/fee-tax-accounts`. Spec detail: `ARCHITECTURE.md` Section 11, item 4.

---

Implement **LedgerApi decision #46: fee-income and tax-payable system accounts per currency**, seeded the same way the funding accounts were (decision #25). Branch + PR as usual; add #46 to `docs/DESIGN.md` in the same style as the rest of the decisions log. Out of scope: anything that changes how `ChainService`/`LegPoster` picks which System account a Fee or Tax leg credits — they still only check `AccountClass == System`, as decision #45 (chains) left them. Choosing the *specific* fee-income/tax-payable account for a transaction is the future Orchestrator's/ProductApi's job, not LedgerApi's; this task only makes the accounts exist and discoverable, exactly as the funding accounts already do for deposits/withdrawals. Do not touch `ChainService`, `ChainRequestValidator`, `LegPoster` or `LegReverser`.

**Why this exists.** Right now a chain's Fee/Tax leg can credit *any* System account (including, accidentally, a funding account) because nothing dedicated exists yet. This task creates the dedicated accounts so a future caller (orchestrator) has a real place to send fees and tax, per currency — it does not change today's validation.

**Pattern to follow — mirror decision #25 exactly:**
- Look at `src/LedgerApi/Migrations/20260920064719_SeedFundingAccounts.cs` and `20261002193410_SeedFundingAccountsForSupportedCurrencies.cs` for the seeding style (raw SQL `INSERT ... ON CONFLICT (account_number) DO NOTHING`, paired `Down` with a `DELETE`), and `src/LedgerApi/Configuration/FundingOptions.cs` / `FundingOptionsValidator.cs` for the configuration + startup-validation style.
- Seed **one Fee-income account and one Tax-payable account for every currency in `SupportedCurrencies`** (currently 20 — the same list the funding accounts now cover). Do this as **one new migration** (not two, unlike the historical funding accounts, which were seeded in two migrations only because the currency list itself grew over time — the currency list is already complete now).
- **Account numbering:** reuse the funding accounts' `<CCY>100000001` shape, but a different number range so there is no collision: `<CCY>200000001` for the Fee-income account, `<CCY>300000001` for the Tax-payable account. Name them `"<CCY> Fee Income Account"` and `"<CCY> Tax Payable Account"`. Same `account_type` (`Wallet`), `account_class` (`System`), `status` (`Active`) as funding accounts.
- **Configuration:** add `FeeIncomeOptions` (`FeeIncome:Accounts`, a currency-to-account-number map) and `TaxPayableOptions` (`TaxPayable:Accounts`), each with its own `IValidateOptions<T>` that fails startup if any supported currency is missing — same shape as `FundingOptionsValidator`. Wire both into `Program.cs` next to where `FundingOptions` is registered. Add the new sections to `appsettings.json` (and any environment-specific config files that already list `Funding:Accounts`), with every account number matching exactly what the migration seeds — same discipline as decision #25's comment ("These account numbers must match the Funding:Accounts map").
- These two option classes are **not used by any endpoint yet** — nothing currently reads `FeeIncome:Accounts` or `TaxPayable:Accounts` (that wiring is future orchestrator/ProductApi work). Their only job right now is to exist, validate at startup, and give the next builder a documented, tested place to seed from. Say this plainly in the PR description so nobody mistakes "configured" for "wired up."

**Tests**
- A migration test (same style as `ChainMigrationTests`/the funding-account migration tests, if one exists — check `tests/LedgerApi.Tests/Data/` first) that, after migrating, every supported currency has exactly one Active System account at `<CCY>200000001` and one at `<CCY>300000001`.
- `FeeIncomeOptionsValidator`/`TaxPayableOptionsValidator` unit tests: missing a currency fails startup validation with a clear message (mirror `FundingOptionsValidatorTests` if it exists).
- Re-run the full existing suite to confirm nothing about chains, transfers, deposits or withdrawals changed (it shouldn't — this task adds rows and config, nothing else).

**Docs.** Add decision #46 to `docs/DESIGN.md`'s decisions log: what was considered (e.g., fold fee/tax accounts into the existing `Funding:Accounts` map vs. dedicated accounts and config), the chosen shape, and why (keeps fee income and tax payable separately reportable per currency, and keeps them distinct from the funding/settlement accounts that deposits and withdrawals use).

Run build and tests; report anything unverified or ambiguous. If you find the two-migration history for funding accounts means there's an easier or different established pattern than described above, follow what the code actually shows and say so — this file describes the pattern as read today, not a guess.
