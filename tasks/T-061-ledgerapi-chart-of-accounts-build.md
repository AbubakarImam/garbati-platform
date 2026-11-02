# T-061: LedgerApi chart-of-accounts build (decision #48, T-060)

Owner: builder. Implements the design ratified in T-060 and written into `LedgerApi/docs/DESIGN.md`
decision #48 (commit `bb70a2c`). Read that decision in full before starting — it is the spec. This
task does not reopen the design; it builds exactly what #48 describes. If something in #48 turns out
to be wrong or ambiguous once you're in the real code, stop and flag it rather than silently
deciding — this is a production ledger service.

## What to build

1. **`GlAccount` entity** (`Entities/GlAccount.cs`): `Id` (long), `Code` (string, unique), `Name`
   (string), `AccountType` (new enum `Asset`/`Liability`/`Equity`/`Income`/`Expense` — do not reuse
   the existing `AccountType` enum on `Account`, which means something unrelated: Wallet/etc. for a
   posting account's own type), `ParentGlAccountId` (long?, self-referencing FK), `CreatedAt`
   (DateTimeOffset).
2. **`Account` entity**: add `public long? GlAccountId { get; set; }`.
3. **Migration(s)**, following the exact precedent of `20261016170502_SeedFeeAndTaxAccounts` and
   `20260920064719_SeedFundingAccounts`:
   - Create `gl_accounts` table.
   - Add nullable `gl_account_id` column + FK on `accounts`.
   - Seed five rows: `1000` Cash and Cash Equivalents (Asset), `2000` Customer Deposits (Liability),
     `2100` Tax Payable (Liability), `4000` Fee Income (Income), `5000` Expense (Expense). All five
     are top-level (`ParentGlAccountId = null`) — no hierarchy populated yet, decision #48 only asks
     for the mechanism to exist.
   - Backfill existing `accounts` rows in the same migration (or a follow-up migration in the same
     PR — your call on data-migration structure, the mapping itself is not optional):
     - Every row seeded by `SeedFundingAccounts`/`SeedFundingAccountsForSupportedCurrencies`
       (decision #25) → `gl_account_id` = the `1000` row's id.
     - Every row seeded by `SeedFeeAndTaxAccounts`'s `FeeIncome:Accounts` half (decision #46) →
       `4000`.
     - Every row seeded by `SeedFeeAndTaxAccounts`'s `TaxPayable:Accounts` half (decision #46) →
       `2100`.
     - Every row where `account_class = 'Customer'` → `2000`.
     - Confirm you can actually distinguish the funding vs. fee-income vs. tax-payable System
       accounts from each other in the backfill (they're all `AccountClass = System`) — check the
       account-number prefix convention decision #25/#46 actually seeded
       (`<CCY>100000001`/`<CCY>200000001`/`<CCY>300000001` per the real `appsettings.json` maps) and
       use that, not a guess. If the real seeded data doesn't let you tell them apart cleanly, stop
       and flag it rather than mis-backfilling a production table.
   - **This migration changes a live, production `accounts` table.** Before this PR is merged (not
     before it's opened), the owner needs to explicitly sign off on the migration's exact SQL —
     flag this in the PR description, same precedent as T-044/PR#7's `KycTierPolicy` migration.
4. **`GlAccountsController`** (`Controllers/Admin/`), mirroring `SystemAccountsController`'s shape
   exactly: `[Authorize(Policy = LedgerScopes.Admin)]`, `api/admin/gl-accounts`.
   - `POST` — create a `GlAccount` (code, name, account type, optional parent id). Validate: code is
     unique, account type is one of the five, parent (if given) exists.
   - `GET` — list all `GlAccount` rows.
   - Follow the existing `IAdminAccountService`/validator/response-contract split
     (`Contracts/Requests`, `Contracts/Responses`, a validator class, a service interface + impl) —
     don't put logic directly in the controller.
5. **Extend `SystemAccountsController`'s existing create endpoint** (`CreateAccountRequest`) with an
   optional `GlAccountId` field, passed through to the created `Account` row. No change to its
   existing validation beyond: if `GlAccountId` is supplied, it must reference a real `GlAccount`.
6. **Trial-balance report**: `GET /api/reports/trial-balance` (`[Authorize(Policy =
   LedgerScopes.Read)]`), new `Controllers/ReportsController.cs`. For every `GlAccount`, sum the
   derived balance (decision #1's existing mechanism — `SUM(credits) - SUM(debits)` over
   `LedgerEntries`, do not add a second balance mechanism) of every `Account` tagged with that
   `GlAccountId` or any of its descendants (header accounts roll up their children). Accounts with
   `GlAccountId = null` are excluded from this report, not an error — not every account is
   categorized yet, by design (decision #48). Response shape: your call, but it should show each GL
   account's code/name/type and its rolled-up balance, structured so a header account's own entry is
   distinguishable from a detail account's.

## What this task does not include

- No change to `ProductApi` or `OrchestratorApi`. Neither service starts supplying `GlAccountId` at
  account-creation time in this task — that's a separate, later task if/when it's wanted. The new
  field on `CreateAccountRequest` is optional and today has no caller passing it.
- No product-to-GL *mapping rule* mechanism (decision #48 explicitly keeps that out of LedgerApi).
- No change to `ChainService`, `ChainRequestValidator`, `LegPoster`, `LegReverser`, or any existing
  posting/transfer/reversal endpoint. `Account`/`LedgerEntry`'s existing balance, locking, and
  posting behavior (decisions #1/#7/#34/#45) must be unaffected — this is purely additive.
- No deep GL hierarchy beyond the five flat seeded rows. The `ParentGlAccountId` mechanism just
  needs to work (tested with at least one real header/child pair in a unit test), not be populated
  for real categories yet.

## Acceptance

- Existing test suite still green, unchanged pass count plus new tests for: `GlAccountsController`
  create/list (including rejecting a duplicate code and an unknown parent id), the backfill migration
  (a test or a manual query confirming the right counts land in the right buckets — show your work),
  the trial-balance endpoint (flat case + at least one header/child rollup case), and
  `SystemAccountsController`'s extended create path (with and without `GlAccountId`, and rejecting an
  unknown one).
- `DESIGN.md` decision #48 updated if anything described above had to change from what's written
  there — note the deviation and why, same discipline as every other decision in that log.
- PR description calls out the production-data migration explicitly and asks for the owner's
  sign-off before merge.
