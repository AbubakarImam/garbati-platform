# Full gap analysis: GARBATI against Apache Fineract's core functionality (T-059)

**Task:** T-059. **Status:** Research only — no code, no schema, no feature design, no timeline, no Sharia
ruling, no interest rate, no profit-sharing ratio. This is the broad comparison `research/modern-treasury-gap-analysis.md`
(T-055) did for Modern Treasury, done here for Apache Fineract — GARBATI's own Apache 2.0 license precedent
(D-21) — across ten core-functionality areas, not the narrow single-feature slices T-056/T-057/T-058 already
covered.

**What this builds on, and does not re-derive:** `research/teller-till-vault-cash-operations.md` (T-056,
`Teller`/`Cashier` endpoints, `CASH_AT_MAINVAULT`/`CASH_AT_TELLER`), `research/multi-charge-transaction-modeling.md`
(T-057, `TaxGroup`-on-`Charge`), and `research/real-corebanking-payload-gap-analysis.md` (T-058,
`acc_gl_journal_entry`'s `entry_date`/`transaction_date`/`office_id`/`description`/`createdby_id` columns,
and Fineract's `maker`/`checker` audit fields) are treated as established fact and cited, not re-read or
re-argued. Area 5 (accounting/GL) and Area 6 (roles/maker-checker) below add only what those two briefs did
not already cover — chart-of-accounts/product-to-GL mapping, and the actual role/permission *granularity*
model, respectively.

**One thing confirmed directly, not assumed, before writing anything else below**: a fresh, independent
repo-wide search of all six GARBATI service repos (`Garbati.IdentityApi`, `Garbati.ProductApi`, `LedgerApi`,
`Garbati.OrchestratorApi`, `Garbati.StaffIdentityApi`, `Garbati.OpsApi`) for `Group`/`Center`/`JointLiability`/
`GroupLending` found zero real matches anywhere (the only grep hits were `.csproj` files' own generic MSBuild
`<ItemGroup>` XML tags, a false positive, not a domain concept) — independently re-verifying, rather than
trusting, the coordinator's own claim in the task file. This is the one area of this whole comparison that
is genuinely microfinance-specific, not a generic commercial-banking pattern, and is treated with the most
care below (Area 2).

---

## Plain-language summary (read this first)

Apache Fineract is a much bigger, much older piece of software than GARBATI, and comparing GARBATI against
"all of Fineract" risks a flat, unhelpful "Fineract has more" conclusion. The honest, more specific picture
found here: **most of what Fineract has that GARBATI doesn't is either (a) a feature GARBATI simply hasn't
built yet, matching an already-named, deliberate scope deferral (loan/financing products, D-28; standing
instructions, "later tranche"), (b) a structural choice GARBATI made on purpose for reasons that still hold
(a small, fixed `AccountClass`/system-account model instead of a general chart of accounts; a coarse,
hardcoded staff-role model instead of Fineract's fully granular, configurable permission system), or (c)
one specific, real, concrete, already-cited open question** (multi-tenancy) **that this research now has a
genuinely useful new data point for, not a new question of its own.**

Only one gap found here is a *microfinance-specific* pattern rather than a generic commercial-banking one:
**Fineract's `Group`/`Center` entities**, the real, named, documented model for organizing clients into
joint-liability groups and groups into centers with scheduled meetings — confirmed directly from Fineract's
own Mifos-forge documentation ("Groups include multiple clients who can act as a unit... all the clients in
that group contribute to it" for a shared savings or loan account; "If there is a center, it determines the
meeting schedule... for all the groups contained in it"). GARBATI has genuinely nothing like this anywhere —
confirmed above, independently, not just repeated from the task file. But this is honestly a *lending-and-
collective-savings* concept, not a deposit-account concept, and GARBATI's own current scope is individual
deposit accounts only, with financing deliberately deferred (D-28) — so this is real, worth real attention
exactly as the task asks, but it is a gap that becomes load-bearing specifically *when* GARBATI's financing
module gets designed, not something missing from today's actual product.

**The most concretely useful single finding in this whole brief is Area 8 (multi-tenancy).** Fineract's real
answer to "how do you serve more than one institution from one deployment" is **not** a tenant-id column on
shared tables — it is a **separate database per tenant**, with a small, separate "tenant registry" database
holding each tenant's own connection details, and a tenant-identifying HTTP header
(`Fineract-Platform-TenantId`) routing each request to the right tenant's own, fully isolated schema. This
directly updates — not replaces — `research/sharia-rule-configuration-patterns.md` (T-050)'s own framing of
GARBATI's open multi-tenancy question, which implicitly assumed "add an institution/tenant key to shared
tables" as the likely shape; Fineract's real, shipped precedent is a materially different architectural
answer than that.

**On interest/profit calculation, stated as plainly as the task requires: nothing below proposes, implies,
or suggests any rate, ratio, or Sharia position.** The only finding here is structural: Fineract's product
configuration carries real fields for *when and how often* interest accrues and gets posted
(`interestCompoundingPeriodType`, `interestPostingPeriodType`, `interestCalculationType`), and a scheduled
background job performs the actual accrual-and-posting. GARBATI's `ProfitSharingTerms` table is a real but
completely unwired placeholder (confirmed directly from its own code, matching T-057's findings), and —
separately and more structurally — **no GARBATI service anywhere has a scheduled-job/background-worker
mechanism at all today**, confirmed by reading the actual code, not assumed. Whatever ratio a future Sharia
scholar eventually approves, there is currently no infrastructure anywhere in GARBATI to execute a
periodic accrual-and-post cycle against it. That is a mechanism gap, not a ruling.

---

## A note on Fineract's own real site structure, checked directly rather than assumed

Fineract's current "stable" documentation hub (`fineract.apache.org/docs/current/`, fetched directly) is
organized mostly around *technical/implementation* topics — architecture, command processing, testing,
deployment — not a product-feature catalog. The richer, product-feature-level detail this brief actually
needed (Clients, Groups, Savings, Loans, Accounting, Roles, Standing Instructions, Reports) came mostly from
Fineract's **legacy API documentation** (`fineract.apache.org/docs/legacy/`, fetched directly and the single
most load-bearing primary source in this brief — same site T-056/T-058 already relied on), Fineract's own
**database-table reference docs** (`fineract.apache.org/docs/database/tables/...`, fetched directly, the
highest-confidence source type per T-058's own convention), and a small number of Mifos-forge/cwiki pages
(mixed success — several `cwiki.apache.org` fetches failed with `connect ECONNREFUSED`, the same repeated
failure T-057/T-058 already documented for that specific host; one `mifosforge.jira.com` page did fetch
successfully). This confirms the task's own ten-area list is a reasonable, coverable structure — it does not
map onto one single Fineract documentation page, but each area maps onto a real, independently-findable
Fineract resource, named below.

---

## Area 1 — Client/Customer management and KYC

**Confidence: high** on both sides (Fineract's legacy API docs fetched directly; GARBATI's code read directly).

**Fineract, read directly:** a `Client` is explicitly "people and businesses that have applied (or may apply)
to an MFI for loans," created `Pending` or directly `Active`, with `activationDate`, `externalId`, and a
name split (`firstname`/`middlename`/`lastname` *or* `fullname`) supporting both individual and business-style
naming. Status transitions include activate/reject/withdraw/reactivate/undo-rejection/undo-withdrawal — a
richer lifecycle than a flat enum. A separate `clients/{clientId}/identifiers` resource holds client
identifiers (the ID-document equivalent), and a separate `Documents` API attaches uploaded files (e.g. a
scanned ID) to a client. [Apache Fineract API Legacy Documentation](https://fineract.apache.org/docs/legacy/)
(fetched and read directly).

**GARBATI, read directly from `Garbati.IdentityApi/src/Garbati.IdentityApi/Entities/`:** `Customer` carries
`CustomerStatus` (`Active`/`Suspended`/`Deceased`/`Closed`) and `KycStatus` (`Pending`/`Verified`/`Rejected`/
`Expired`) as two separate fields/lifecycles, not one combined status the way Fineract's `Client` reads.
`CustomerIdentityData` holds `FullName`/`Email`/`PhoneNumber`/`Bvn`/`Nin` as a deliberately separated,
sensitive-data table (per its own code comment, tied to decision #26's "store unencrypted for now" choice).
`KycVerification` is an append-only log of verification *attempts*, each with `VerificationType`, `Provider`,
`ProviderReference`, `Status`, `ResultCode`, `VerifiedAt`/`ExpiresAt` — structurally richer, in one specific
way, than Fineract's plain identifier list: it tracks *which provider* verified *what kind* of evidence and
*when it expires*, not just a static identifier value. `KycTierPolicy` is a configurable, jsonb-backed
tier/evidence table (decision #11), the mechanism `research/sharia-rule-configuration-patterns.md` (T-050)
already analyzed in depth.

**What GARBATI has no equivalent for at all:** Fineract's separate `Documents` API (attaching an uploaded
file — a scanned ID photo, a selfie — as a first-class, retrievable entity linked to a client) and its
individual-vs-business client distinction (`Customer.FullName` is a single string field with no "is this a
business" concept anywhere). Neither is a headline gap for a deposit-only, individual-customer platform
today, but both are real, named absences.

**Verdict: GARBATI has a real equivalent.** The core shape — an identity record, a KYC status/tier, and a
record of what evidence was checked — is present and, in the verification-provenance dimension specifically
(`KycVerification`'s provider/type/expiry tracking), arguably more structured than Fineract's own plain
identifier list. Document storage and business-vs-individual client typing are real, named, minor absences,
not load-bearing for GARBATI's current individual-customer, deposit-only scope.

---

## Area 2 — Groups and Centers (group lending) — the genuinely microfinance-specific gap

**Confidence: high** on Fineract's real documented shape (one page fetched and read directly, a second
corroborated via search); **high** on GARBATI's absence (independently re-verified, not just repeated from
the task file, per this brief's own opening paragraph).

**Fineract, read directly:** "Groups include multiple clients who can act as a unit. In a scenario where a
group has a savings or a loan account, all the clients in that group contribute to it" — i.e. a `Group` can
itself hold a savings or loan account that its individual members are jointly responsible for, not merely a
label applied to several otherwise-independent individual accounts. "If there is a center, it determines the
meeting schedule (if there is one) for all the groups contained in it" — a `Center` is a scheduling/
organizational container above `Group`, typically overseen by one loan officer across several groups.
[Mifos-forge — Clients, Groups and Centers](https://mifosforge.jira.com/wiki/spaces/docs/pages/52035634/Clients+Groups+and+Centers)
(fetched and read directly). Fineract's own API separately confirms real, named "collection sheet" generation
for both groups and centers (`groups/{groupId}?command=generateCollectionSheet`) and meeting-attendance
recording (`/v1/{groups|centers}/{entityId}/meetings/{meetingId}?command=saveOrUpdateAttendance`) — real,
shipped functionality, not a documentation-only concept. [Apache Fineract API Legacy Documentation](https://fineract.apache.org/docs/legacy/)
(fetched directly); meeting-attendance endpoint confirmed via search-engine summary of the same documentation
set, not independently re-fetched, flagged at slightly lower confidence for that one specific endpoint path.

**GARBATI:** zero matches anywhere, independently confirmed by this research's own fresh repo-wide search
(stated in this brief's opening section), not merely repeated from the coordinator's claim.

**Is this a near-universal microfinance pattern, or specific to certain lending models/regions — the honest
answer, not forced either way, as the task requires:** group lending (joint-liability groups, center
meetings) is the historically foundational methodology most associated with microfinance as a field —
Grameen Bank's own model, and the reason Fineract (and its predecessor Mifos) were built this way from the
start. But it is **not the only live microfinance model today**: a separate, real, converging finding (search
result on Fineract's own generational history) states Fineract "Gen 2... was built with greater flexibility
and support for individual lending" while still "deeply catered to group and center-based operations" — i.e.
even Fineract's own community treats group lending as the historically dominant but no longer the *only*
real microfinance methodology, individual lending having grown as a real alternative over time. This matters
directly for how to read GARBATI's own absence: GARBATI's current scope (individual deposit accounts, with
`Mandate`'s joint-account M-of-N approval being a *multi-holder authorization* concept on one individual
account, not a *group liability* concept) sits squarely in the "individual" branch of real microfinance
practice, not evidence that GARBATI has missed something every MFI needs.

**Verdict: genuinely missing, worth considering — the one real, microfinance-specific (not generic
commercial-banking) pattern in this entire comparison, but one whose absence matches GARBATI's own current,
deliberately deposit-only scope (D-28 defers financing entirely) rather than an oversight in what's already
built.** This becomes load-bearing specifically if and when GARBATI's financing module is designed — group/
joint-liability lending is a real, named, commonly-supported methodology a Nigerian microfinance institution
adopting GARBATI could plausibly want, and nothing in GARBATI's architecture today (mandates, products,
accounts) has any structural analogue for "several people jointly liable for one account" the way `Mandate`'s
M-of-N approval has an analogue for "several people jointly *authorized* on one account" — liability and
authorization are different concepts, and this research found no existing GARBATI mechanism that does the
former at all.

**What I could not verify:** whether Fineract's own group/center meeting-attendance and collection-sheet
endpoints are still actively used/maintained in current Fineract releases versus being older, less-touched
parts of the codebase — this research did not check recent commit activity on these specific endpoints.

---

## Area 3 — Savings/deposit products

**Confidence: medium-high.** Fineract's field *names* are confirmed directly from its own legacy API
documentation; the full prose description of each field was not retrievable from the specific page content
this toolchain could extract, flagged honestly rather than filled in from memory.

**Fineract, read directly (field names confirmed; full field-level prose descriptions not retrievable from
this fetch):** a savings product carries, as real, named, distinct fields: `interestCompoundingPeriodType`,
`interestPostingPeriodType` (daily, monthly-on-the-first, quarterly, bi-annual, and several anniversary-based
variants keyed to the account's own activation date — this part confirmed via a converging search-engine
summary of the same legacy documentation, not independently re-fetched verbatim), `interestCalculationType`
(daily balance vs. average daily balance), `interestCalculationDaysInYearType`, `minRequiredBalance` and
`enforceMinRequiredBalance` as two separate fields (the number and whether it's actually enforced are
independently toggleable), `withdrawalFeeForTransfers`, and `lockinPeriodFrequency`. [Apache Fineract API
Legacy Documentation](https://fineract.apache.org/docs/legacy/) (fetched directly — field names present in
the page; descriptive prose for each not present in the specific content this fetch returned, named plainly
rather than invented).

**GARBATI, read directly from `Garbati.ProductApi/src/Garbati.ProductApi/Entities/`:** `Product` is a thin
catalog row — `ProductCode`/`ProductName`/`JurisdictionCode`/`CurrencyCode`/`ProductType` (a plain string,
"Savings" or "Current") plus `EffectiveFrom`/`EffectiveTo`. `ProductLimitPolicy` adds exactly `MinimumBalance`
(a single decimal, no separate "is this enforced" toggle the way Fineract has) and `AllowedOperations` (a
configurable jsonb string list). `AccountProductBinding` links one `LedgerApi` account to one product code.
There is no field, table, or concept anywhere matching Fineract's compounding/posting-period types, lock-in
period, or a withdrawal-fee-specific field distinct from the generic (and currently unseeded) `FeeRule`.

**Verdict: genuinely missing, worth considering — but matching a named, deliberate current scope, not a
surprise.** `ProductApi`'s own `DESIGN.md`/`ARCHITECTURE.md` framing (Section 2 table) already states this is
a "v1 (minimal) foundation," with `FeeRule`/`ProfitSharingTerms` deliberately unseeded pending scholarly
sign-off. The *mechanical* configurability gaps found here (no lock-in-period concept, no separate
enforce-vs-value toggle on minimum balance, no interest-compounding/posting-period concept at all) are real
and independent of any Sharia content question — a future task building out real savings-product
configurability, even before any profit-sharing ratio exists, would need to add some of these fields
regardless of what number eventually fills them.

**What I could not verify:** the exact prose Fineract's own documentation uses to describe each field
individually (this toolchain's fetch of the legacy docs page returned the field *names* but not a fully
retrievable per-field description for every one of them); no independent confirmation of the specific
anniversary-posting-period variants beyond the search-engine summary cited above.

---

## Area 4 — Loan/financing products

**Confidence: medium-high** on Fineract's real shape (field names confirmed directly; richer behavioral
detail via converging search summaries); **this comparison is necessarily absence-vs-shape, not
feature-for-feature, exactly as the task requires, and is treated that way below.**

**Fineract, read directly and via converging search summaries of the same legacy documentation:** a loan
product carries `interestRatePerPeriod`, `interestRateFrequencyType`, `amortizationType`, `interestType`,
`interestCalculationPeriodType`, and a `transactionProcessingStrategyCode` governing how a repayment is
allocated across principal/interest/fees/penalties. Fineract supports two real, named repayment-schedule
*shapes* — "Cumulative" (interest calculated on the declining balance of principal, the older, dominant
model) and "Progressive" (interest calculated against the full planned-disbursement amount, recalculating
the remaining schedule whenever a new tranche is disbursed, with newly disbursed money accruing interest only
from its own disbursement date forward, not retroactively). Loan products also carry a "grace period" concept
(interest still calculated during the grace window but its payment deferred to later installments) and an
interest-recalculation flag for what happens if a scheduled payment is missed. [Apache Fineract API Legacy
Documentation](https://fineract.apache.org/docs/legacy/) (field names fetched directly); schedule-type and
grace-period behavioral detail via search-engine summaries of Fineract's own cwiki/academy material, not
independently fetched in full, flagged at correspondingly reduced confidence for the behavioral (not
field-name) claims specifically.

**GARBATI:** no financing module exists at all — confirmed directly, there is no `Garbati.FinancingApi` (or
equivalent) repository anywhere in the project's workspace, and `ARCHITECTURE.md`'s own service map lists
"Financing services (Murabaha, etc.)... Later tranche," with D-28 explicitly scoping the *eventual* contract
types (Murabaha, Musharaka, Mudarabah, Qard Hasan, Wadiah, Ijarah, Istisna) without anything built.

**Verdict: not evaluated as missing/present — a deliberate, already-named scope deferral (D-28), not a gap
to weigh.** As the task itself requires, this is "necessarily about Fineract's real shape versus GARBATI's
absence... not a feature-for-feature match," stated here plainly rather than forced into a verdict that would
misleadingly suggest a surprise finding. The one thing worth naming for a future design task, without
designing anything here: Fineract's `transactionProcessingStrategyCode` concept (a named, swappable strategy
for how one repayment amount gets split across principal/interest/fees/penalties) is structurally close to
GARBATI's own leg/chain mechanism (P-14) — "post several tagged legs that must sum correctly" — the same
structural convergence `research/multi-charge-transaction-modeling.md` (T-057) already found between
GARBATI's chain model and ISO 20022's `ChargesInformation` array, now found a third time in a different real
system's loan-repayment allocation concept specifically. Not a design recommendation — an observation for
whoever eventually designs GARBATI's financing module.

**What I could not verify:** the exact arithmetic Fineract uses inside "Progressive" vs "Cumulative"
schedules beyond the general shape described above; no primary Fineract source describing the full
mathematical formula for either was independently fetched and read in full.

---

## Area 5 — Accounting/GL and journal entries (building on, not repeating, T-058)

**Confidence: medium-high** on Fineract (field/endpoint names confirmed directly from two sources; the
semantic detail of account-type hierarchy via converging search summary); **high** on GARBATI (read directly
from code, confirming T-058's own prior finding and adding nothing new to that specific part).

**What T-058 already established and is not repeated here:** `acc_gl_journal_entry`'s own schema
(`entry_date`/`transaction_date`/`office_id`/`description`/`createdby_id`/`lastmodifiedby_id`) and Fineract's
`PaymentType`-based GL routing for payment channels.

**What this area adds — the chart-of-accounts model and product-to-GL mapping, read directly:** Fineract's
own legacy API documentation confirms a real `glaccounts` resource (create/list/retrieve/update/delete a
"General Ledger Account"), a separate `accountingrules` resource ("Accounting Rules" — the actual
product-to-GL-account mapping mechanism), and a separate `financialactivityaccounts` resource ("Financial
Activity to Account Mapping," the same `CASH_AT_MAINVAULT`/`CASH_AT_TELLER`-style financial-activity
constants T-056 already found). [Apache Fineract API Legacy Documentation](https://fineract.apache.org/docs/legacy/)
(fetched directly). GL accounts are organized into the standard five categories — Asset, Liability, Equity,
Revenue (Income), Expense — arranged hierarchically with "main accounts and sub-accounts" (header vs. detail
accounts), and a savings/loan product's own charges, interest, and principal postings are configured to map
to specific GL accounts of the appropriate type via these accounting-rule/mapping resources (search-engine
summary of Fineract's own accounting documentation, converging across several sources, not independently
fetched as one single page — flagged at correspondingly reduced confidence for this specific hierarchical-
structure claim, though the `glaccounts`/`accountingrules`/`financialactivityaccounts` endpoint names
themselves are directly confirmed).

**GARBATI, read directly (confirming and extending T-058's own finding, not re-deriving it):** `LedgerApi`'s
entire GL-adjacent surface is `AccountClass` (exactly two values: `Customer`, `System`) plus three fixed,
narrowly-purposed configuration sections — `FundingOptions`, `FeeIncomeOptions`, `TaxPayableOptions` — each
pre-seeding one system account per currency in a database migration (decisions #25/#46). There is no
`glaccounts`-equivalent resource, no Asset/Liability/Equity/Income/Expense categorization anywhere, no
hierarchical header/detail account concept, and no generic, extensible product-to-GL-account mapping
mechanism — `LegPurpose` is a hard-closed three-value enum (`Principal`/`Fee`/`Tax`, decision #45/D-7),
and a new value is explicitly "a new decision and a ledger code change, not an automatic extension"
(`ARCHITECTURE.md` Section 9). A `SystemAccountsController` exists (`api/admin/system-accounts`) to create a
system account, but it creates one specific account, not a configurable chart-of-accounts entry with its own
type/hierarchy.

**Verdict: different by design, with the real reason stated plainly, not papered over.** GARBATI's
`LedgerApi` is a transactional posting engine with a small, fixed, purpose-built set of system accounts, not
a general-ledger/chart-of-accounts system that produces a full trial balance or financial statements the way
Fineract — which often *is* the institution's entire accounting system of record in a real deployment — is
built to do. This is a genuine, reasoned structural choice (P-14's own stated reasoning: "keeps the tested
guarantee of one debit and one credit per transaction," a deliberately narrow surface), not an oversight.
**Honest caveat, not smoothed over:** if GARBATI's own institutional customers eventually need to produce
standard financial statements, trial balances, or a fully configurable chart of accounts of their own
(rather than relying on an external accounting system fed by GARBATI's transaction data), this design choice
would then become a genuinely missing capability, not merely a different one — this research does not resolve
which of those two framings will turn out to be the lasting one, and names both honestly.

---

## Area 6 — Roles, permissions, and maker-checker (building on, not repeating, T-058)

**Confidence: medium-high.** Fineract's permission-granularity model and maker-checker configuration
mechanism are confirmed via converging search-engine summaries of two distinct Fineract/cwiki-adjacent
sources (direct `cwiki.apache.org` fetch failed with `connect ECONNREFUSED`, the same recurring failure
T-057/T-058 already documented for that host); GARBATI's role model is read directly from code.

**What T-058 already established and is not repeated here:** Fineract's `maker`/`checker` audit fields and
`createdby_id`/`lastmodifiedby_id` on the ledger table itself.

**What this area adds — the actual role/permission granularity model:** Fineract's roles are built from a
fine-grained **per-task** permission list, each permission independently markable as Read/Change/Delete
("Read means the user can only view... Change means the user can view and make changes... Delete means the
user can view, change and delete... with respect to that role"), with a UI convenience where checking a
higher-privilege box auto-checks the lower ones. Maker-checker is **not** a fixed, hardcoded set of
workflows — it is enabled per individual permission/task via the Permissions API itself (`GET
/fineract-provider/api/v1/permissions?makerCheckerable=true` lists which tasks currently require it; sending
`{"permissions": {"APPROVE_LOAN": true}}` turns it on for that one specific task), globally switchable on/off
first via a separate Global Configuration flag. A maker needs the task's ordinary operation permission; a
checker separately needs `CHECKER_SUPER_USER` or a task-specific checker permission — two independently
assignable permissions per workflow, not one combined "supervisor" role. [Roles And Permissions — Fineract]
(cwiki page title/content via converging search-engine summary; direct fetch failed) and the equivalent
maker-checker mechanism description (search-engine summary of a third-party Fineract documentation site,
`finecko.com`, itself summarizing the same underlying Permissions API — not independently fetched in full by
this research, flagged accordingly).

**GARBATI, read directly from `Garbati.StaffIdentityApi/src/.../Entities/Role.cs` and
`Garbati.OpsApi/src/.../Authorization/OpsRoles.cs`:** exactly seven fixed, named roles (`Teller`,
`CustomerServiceOfficer`, `Supervisor`, `ComplianceOfficer`, `RiskOfficer`, `Auditor`, `Administrator`), with
no sub-permission structure at all — a staff member's `Role` is a single claim, and `OpsApi` enforces access
per endpoint using ASP.NET Core's own built-in `[Authorize(Roles = "...")]`, hardcoded per endpoint (e.g.
`TellerOrSupervisor` for the testing console, `CustomerServiceOfficerOrSupervisor` for customer-detail edits).
Maker-checker is not a generically-configurable "which tasks need a checker" switch at all — it is hardcoded,
per specific workflow, directly in that workflow's own entity/service code (e.g. `OpsReversalRequest`'s own
code comment: "the deciding session must currently hold Supervisor, re-checked live at decision time"). There
is no equivalent anywhere of Fineract's separate, independently-assignable "maker permission" vs. "checker
permission" — in GARBATI, a staff member either holds `Supervisor` (and can check) or doesn't, full stop, for
every maker-checker workflow that exists.

**Verdict: different by design, with the real reason stated, not a stretch.** GARBATI's staff/ops surface is
genuinely small today (seven fixed roles, a handful of named workflows: reversal requests, customer edits, a
testing console) — a coarse, hardcoded role-to-endpoint mapping is a reasonable, deliberate choice at this
scale, and matches `OpsApi`'s own code comment reasoning directly ("a staff member's Role is a single claim
already resolved once per request... the framework's own role check is the one shared, reusable check" —
i.e. chosen explicitly over "a bespoke policy-per-scope mechanism"). Fineract's much finer-grained,
independently-configurable permission model exists because Fineract must let one single deployment's
administrators define arbitrary custom roles across a vastly larger surface (hundreds of distinct
loan/savings/accounting/client operations) without a code change — a real need GARBATI's current, much
smaller surface does not yet have. **Honest caveat:** if GARBATI's own staff/ops surface grows substantially,
or if the platform is ever deployed as a genuinely multi-tenant system where different adopting institutions
want to define their *own* distinct roles (directly touching the Area 8 multi-tenancy question below), this
coarse, hardcoded model would then need to become something closer to Fineract's configurable one — not a
decision made here, but a real, named trade-off worth the owner's attention if either of those conditions
changes.

**What I could not verify:** the complete, current list of every individual Fineract permission/task code
(this research found the *mechanism* — per-task Read/Change/Delete, independently markable maker-checkerable
— not an exhaustive list of every task Fineract defines); no primary `cwiki.apache.org` page on this topic
was independently fetched and read in full, the same recurring access limitation this project has now
documented across four research tasks (T-056/T-057/T-058/T-059) for that specific host.

---

## Area 7 — Standing instructions / recurring transactions

**Confidence: medium-high** on Fineract's real, named resource shape (confirmed directly from its own legacy
API documentation); GARBATI's own absence here is simply the already-stated "later tranche" fact from
`ARCHITECTURE.md`, not independently re-derived.

**Fineract, read directly:** a real `standinginstructions` resource (create/list/retrieve/update/delete one
standing instruction) exists as its own thing, **separate** from a second, dedicated
`standinginstructionrunhistory` resource that tracks each individual *execution* of that instruction over
time. [Apache Fineract API Legacy Documentation](https://fineract.apache.org/docs/legacy/) (fetched and read
directly — both resource names and their separation are confirmed directly from the page content, not a
search summary).

**GARBATI:** `ScheduleApi` ("Standing Orders") is explicitly "Later tranche" per `ARCHITECTURE.md`'s own
service map, with a one-line scope statement ("Owns: Recurring instructions and their run history") and no
code anywhere.

**Verdict: necessarily absence-vs-shape, as the task requires, stated plainly rather than forced into a
false comparison.** The one thing worth naming for whoever eventually designs `ScheduleApi`, without
designing anything here: `ARCHITECTURE.md`'s own one-line scope statement for `ScheduleApi` ("recurring
instructions and their run history") already independently names the exact same **two-part split** — a
definition object plus a separate run-history object — that Fineract's own real, shipped implementation uses
(`standinginstructions` vs. `standinginstructionrunhistory`). This is a genuine, useful, external confirmation
that GARBATI's own already-written framing is a real, externally-validated shape, not a new design input —
worth noting as reassurance for the eventual design task, not a new requirement. Separately, this connects
directly to `ARCHITECTURE.md` Section 9's own still-open question on standing-order retry-on-failure
behaviour (P-3): a dedicated run-history resource, as Fineract has it, is exactly the kind of place a
per-run status (succeeded/failed/retried) would need to live, which is one concrete, specific reason that
open question matters once `ScheduleApi` design actually starts — not a resolution of it.

**What I could not verify:** the specific status values Fineract's own `standinginstructionrunhistory`
records for a failed run (i.e. whether Fineract itself auto-retries a failed standing instruction, which
would bear directly on `ARCHITECTURE.md` Section 9's own open P-3 question) — this research found the
resource's existence and its separation from the instruction definition, but not its own documented
retry-on-failure behaviour specifically.

---

## Area 8 — Multi-tenancy (concrete new input to an already-open GARBATI question, not a new question)

**Confidence: medium-high.** The core architectural fact (separate database per tenant, not a shared-table
tenant key) is confirmed by two independently converging sources; the exact tenant-header mechanism by a
third, separate search-confirmed detail.

**Fineract, confirmed via two independently converging sources:** "Every tenant has its own: **Database** —
all loan data, client records, savings accounts, transactions, accounting entries, and configuration are
stored in a dedicated database that no other tenant can access." [Finecko — Multi-Tenancy in Apache
Fineract](https://finecko.com/docs/fineract/features/multitenancy.html) (fetched and read directly). A
separate, small **tenant registry** database (not the same as any tenant's own data database) stores each
tenant's identifier, schema name, host, port, username, and password, and Fineract "always connects to this
[registry] database to fetch tenant details and connection information first" before routing to that
tenant's own isolated schema. When a new Fineract version is deployed, its own database-migration system
"automatically runs pending schema changes against every registered tenant's database... in the same
deployment cycle" (search-engine summary of Fineract's own configuration documentation, converging
independently with the Finecko source above on the same "separate database, central registry" shape, not
independently fetched as one single page). The specific routing mechanism: a request-level HTTP header,
`Fineract-Platform-TenantId` (e.g. `Fineract-Platform-TenantId: default`), tells Fineract which tenant's
registry entry — and therefore which tenant's database — to use for that request (search-engine summary of
Fineract developer-configuration material, not independently fetched in full, but specific and consistent
enough across multiple independent search results to treat as a real, accurately-named header, not an
invented one).

**GARBATI:** zero `TenantId`/`TenantCode`/`InstitutionId`/`OrganizationId` concept anywhere, confirmed by a
direct grep across `Garbati.ProductApi` and `LedgerApi` source (the two services most load-bearing for
Sharia-rule/product configuration, per T-050's own framing) — and, checked further than T-050's own scope,
the same absence holds across every other GARBATI service too (`Garbati.IdentityApi`, `Garbati.StaffIdentityApi`,
`Garbati.OrchestratorApi`, `Garbati.OpsApi`); this research found no tenant concept anywhere in the platform,
not just in `ProductApi`.

**This is concrete, new input to `research/sharia-rule-configuration-patterns.md` (T-050)'s own open
multi-tenancy question and `ARCHITECTURE.md` Section 9 — not a new question, exactly as the task requires.**
T-050's own framing implicitly assumed the likely shape of a future fix would be "an explicit
institution/tenant key on `Product`/`FeeRule`/`ProfitSharingTerms`" (a shared-database, tenant-column
approach). Fineract's real, shipped precedent answers the underlying "how would a real, comparable
core-banking platform actually do this" question **differently**: not a tenant-id column added to shared
tables at all, but **fully separate databases per tenant**, with one small, separate, shared registry
database holding only connection metadata. This doesn't resolve which approach GARBATI should eventually
take — P-6/P-15's own existing "separate design, minimal deployables, no shared database across services"
philosophy (decision log) is arguably *more* naturally compatible with Fineract's "separate database per
tenant" shape than with a shared-table tenant-key approach, since GARBATI's services already avoid shared
databases between *services*; extending that same discipline to "and also separately per *institution*" is
at least directionally consistent with a choice GARBATI has already made elsewhere, for different reasons.
This is an observation, not a recommendation — the decision itself remains with the owner, informed by this
new data point.

**What I could not verify:** the exact mechanics of what happens during a schema migration if one tenant's
database is temporarily unreachable while others succeed (partial-rollout failure handling); whether
Fineract's tenant registry itself is ever deployed in a high-availability/replicated configuration, or is a
real, named single point of failure in practice — neither question was answered by any source this research
could reach.

---

## Area 9 — Interest/profit calculation mechanism (structural only — no ruling, rate, or ratio implied)

**Stated as plainly as possible, matching the task's own explicit instruction:** nothing in this section
proposes, implies, or suggests any specific interest rate, profit-sharing ratio, or Sharia position. The only
question asked and answered here is: *if* GARBATI's own scholar-approved rules existed, what mechanism would
need to exist to compute and post them — and does any part of that mechanism exist in GARBATI today, as a
pure infrastructure question, independent of content.

**Confidence: high** on both the Fineract mechanism-shape and GARBATI's current absence (both read directly:
Fineract's field names from its own legacy docs, discussed already in Area 3; GARBATI's `ProfitSharingTerms`
read directly from its own code).

**Fineract's mechanism, as a shape, reusing field names already confirmed directly in Area 3:** a product
carries a rate/ratio-adjacent field (`interestRatePerPeriod`/the savings-side compounding fields) plus a
**period** field (`interestCompoundingPeriodType`, `interestPostingPeriodType`) that determines how often
accrual happens and how often it gets posted to the ledger as a real entry. This requires, as infrastructure,
**a scheduled job that runs periodically, computes an accrual for every affected account, and posts it** —
confirmed as real, not inferred, by two real, named Fineract GitHub pull requests found in prior research
(`research/teller-till-vault-cash-operations.md`'s own sourcing norms) with titles directly naming this job:
"Post Interest for Savings Job Fix" and "Savings Interest Posting Job Fix" (titles only, via search-engine
summary, not independently fetched and read in full in this specific research, though the existence of a
named, real, scheduled "Savings Interest Posting Job" in Fineract's own codebase is corroborated by multiple
independent search results converging on the same job name). **The mechanism shape, in plain words: a rate,
applied to a balance, over a period, computed by a background job, posted to the ledger on a schedule** — not
a side effect of any single customer-initiated transaction.

**GARBATI, read directly from `Garbati.ProductApi/src/Garbati.ProductApi/Entities/ProfitSharingTerms.cs`:**
the table exists — `ProductCode`, `ProfitShareRatioCustomer`, `ProfitShareRatioInstitution`,
`EffectiveFrom`/`EffectiveTo` — but its own code comment states plainly: "shape only, deliberately empty, AND
NOT WIRED TO ANYTHING IN V1. No endpoint, service method, controller action or validator anywhere in this
codebase reads or writes this table." This matches `research/multi-charge-transaction-modeling.md` (T-057)'s
own prior finding about `FeeRule`'s similarly unwired state, not a new discovery about that specific table.

**What this research adds, read directly, that is new:** separate from `ProfitSharingTerms`'s own content
being empty, **no GARBATI service anywhere has a scheduled-job/background-worker mechanism of any kind
today** — `LedgerApi` "calls nobody" (decision #34) and has no outbound scheduler; `OrchestratorApi`'s own
`AccountOpening` entity is driven entirely by explicit, caller-initiated retries (`POST
/v1/account-openings/{id}/retry`), not a background sweep; no `IHostedService`/cron/timer-triggered pattern
was found in any of the six service repos' source during this research. This is a genuinely different,
lower-level gap than "the ratio table is empty" — it means that even once a ratio is approved and the table
is filled in, **the infrastructure to periodically compute and post an accrual at all does not yet exist
anywhere in GARBATI**, independent of what number eventually goes in the table.

**Verdict: genuinely missing, worth considering — specifically for the mechanism piece (a scheduled
accrual-and-post job), separate from and more structural than the already-known "the ratio table is empty"
fact.** The storage shape for a ratio (`ProfitSharingTerms`) already exists, in the same way
`KycTierPolicy`/`FeeRule` already exist as configurable-but-empty tables — that part is a reasonable
foundation, not a gap. What is a genuine, infrastructure-level gap, confirmed by reading the actual code
rather than assumed, is that **no GARBATI service has ever built a periodic/scheduled execution mechanism of
any kind** — `ScheduleApi` itself (Area 7) is "later tranche" and unbuilt, and that absence is not confined to
standing orders; it is the same missing piece this area's own interest/profit-posting mechanism would also
need. This is worth the owner's attention as a shared piece of future infrastructure (one scheduled-job
capability, potentially serving both `ScheduleApi`'s eventual standing orders *and* a future profit-posting
mechanism), not two separate problems — though this research does not design that infrastructure, per its own
explicit scope.

---

## Area 10 — Reporting

**Confidence: high** on Fineract (its own database-table documentation fetched and read directly); **high**
on GARBATI's absence (no `Report`-named controller, entity, or endpoint found anywhere in any of the six
service repos).

**Fineract, read directly:** a real, native, database-backed reporting mechanism exists independent of any
external plugin — the `stretchy_report` table itself (columns: `id`, `report_name`, `report_type`,
`report_subtype`, `report_category`, `report_sql`, `description`, `core_report`, `use_report`,
`self_service_user_report`) stores SQL-defined reports directly inside Fineract's own database. [Fineract
Database Docs — `stretchy_report` table](https://fineract.apache.org/docs/database/tables/stretchy_report.html)
(fetched and read directly). This is distinct from the separately-licensed **Pentaho reporting plugin**
(confirmed via a different, independently fetched source: "Currently Fineract does not support Pentaho
reports due to licensing, which is why it exists as a separate plugin project" — [GitHub — vorburger/fineract-pentaho,
via Mifos docs](https://docs.mifos.org/external-3rd-apps/pentaho-reporting-plugin), fetched directly, though
this specific page did not itself describe the native `stretchy_report` mechanism, confirmed instead from the
database-docs page above). Fineract's own legacy API exposes this via a `reports` resource
(create/list/retrieve/update/delete) and a `runreports/{reportName}` execution endpoint. A search-engine
composite summary (not independently verified in full) states Fineract "ships with a library of standard
reports covering portfolio performance, arrears, client statistics, and accounting statements" — named,
specific enough to be a real claim, but flagged at reduced confidence since no single source listing every
shipped report by name was independently fetched and read in full.

**GARBATI:** no reporting layer of any kind — confirmed by a direct search for "Report"/"report" across
every controller directory in every GARBATI service repo, finding zero matches anywhere. Account/transaction
data is only retrievable via each service's own narrow, purpose-built lookup endpoints (e.g. `LedgerApi`'s
account/transaction GET endpoints), with no aggregate, parameterized, or scheduled reporting capability
anywhere.

**Verdict: genuinely missing — a real gap, and, as the task predicted, a shallow one to characterize.**
GARBATI has nothing; Fineract has a real, native, SQL-report mechanism (plus an optional external Pentaho
plugin for more elaborate output formats) shipped as a standard part of the platform. This is squarely a
"not built yet" gap rather than a structural or design question, and nothing about GARBATI's own current
architecture (database-per-service, no shared reads across services) obviously prevents a future reporting
layer from being added — it would most naturally need its own read access (or its own replicated/aggregated
copy of) each service's data, a genuinely new concern this research does not attempt to resolve.

---

## Summary table

| # | Area | Fineract's real shape | GARBATI's real state | Verdict |
|---|---|---|---|---|
| 1 | Client/Customer & KYC | `Client` entity, identifier/Documents APIs, rich status lifecycle | `Customer`/`CustomerIdentityData`/`KycVerification`/`KycTierPolicy`, built | **GARBATI has a real equivalent** (minor gaps: no document storage, no business-client typing) |
| 2 | Groups & Centers (group lending) | `Group`/`Center` entities, joint-liability accounts, meeting/collection-sheet workflows | Zero, independently re-confirmed | **Genuinely missing, worth considering** — the one real microfinance-specific gap, but matches GARBATI's current deposit-only scope (D-28) |
| 3 | Savings/deposit products | Compounding/posting-period types, lock-in period, enforce-vs-value minimum-balance split | Thin `Product`/`ProductLimitPolicy`/`AccountProductBinding`, no interest-period or lock-in concept | **Genuinely missing, worth considering** — mechanical gaps independent of any Sharia content |
| 4 | Loan/financing products | Amortization types, Cumulative/Progressive schedules, grace periods, repayment-allocation strategy | Nothing built; D-28 defers financing entirely | **N/A — deliberate scope deferral**, not a gap to weigh |
| 5 | Accounting/GL | `glaccounts`, `accountingrules`, `financialactivityaccounts`; Asset/Liability/Equity/Income/Expense hierarchy | Fixed `AccountClass` (Customer/System) + three narrow seeded system-account config sections; closed `LegPurpose` | **Different by design** — a transactional posting engine, not a general ledger/chart-of-accounts system; may become genuinely missing if GARBATI ever needs to produce its own financial statements |
| 6 | Roles, permissions, maker-checker | Per-task, independently markable Read/Change/Delete permissions; maker-checker configurable per task | Seven fixed roles, hardcoded `[Authorize(Roles=...)]` per endpoint, maker-checker hardcoded per workflow | **Different by design** — reasonable at today's small staff-ops surface; may need to converge toward Fineract's model if that surface (or multi-tenancy) grows |
| 7 | Standing instructions | `standinginstructions` + separate `standinginstructionrunhistory` | `ScheduleApi` not designed/built ("later tranche") | **N/A — absence matching a named deferral**; Fineract's two-part shape independently confirms `ARCHITECTURE.md`'s own existing one-line framing |
| 8 | Multi-tenancy | Separate database per tenant + a small shared tenant-registry database + `Fineract-Platform-TenantId` header | Zero tenant concept anywhere in any of the six services | **Genuinely missing — concrete new input to the already-open T-050/Section 9 question**, not a new question |
| 9 | Interest/profit mechanism (structural only) | Rate/ratio + period fields, computed and posted by a real scheduled job | `ProfitSharingTerms` unwired; **no scheduled-job mechanism anywhere in GARBATI at all** | **Genuinely missing, worth considering** — the scheduling infrastructure, not any rate/ratio/ruling |
| 10 | Reporting | Native `stretchy_report` SQL-report mechanism + optional Pentaho plugin | Zero reporting layer anywhere | **Genuinely missing** — real but shallow, as expected |

---

## Synthesis — which gaps matter most for GARBATI's own stated microfinance focus, specifically

Not a generic "Fineract has more features" conclusion, as the task requires. Ranked by relevance to
GARBATI's own stated purpose (a Sharia-compliant microfinance platform for Nigerian institutions), not by
raw feature-count:

1. **Groups/Centers (Area 2) is the only gap in this entire comparison that is specific to microfinance as a
   field, rather than generic commercial banking** — and it is honestly, currently, a non-issue for GARBATI's
   *today* (deposit-only, individually-held accounts), becoming genuinely relevant only at the moment
   GARBATI's financing module is actually designed, since joint-liability group lending is a real, commonly-
   offered methodology a Nigerian MFI adopting GARBATI could reasonably want. This is worth a specific,
   explicit flag for whoever scopes the financing module next — not something to build now, but something to
   not accidentally design the financing data model without room for later.
2. **Multi-tenancy (Area 8) is the most concretely actionable finding for an already-open question.** This
   isn't a new gap — it's a real, named architectural answer (separate database per tenant, not a
   shared-table key) to a question GARBATI already knows it has open (T-050/Section 9), and one that is
   arguably more compatible with GARBATI's existing "no shared database" discipline (P-5/P-6) than the
   shared-key approach that question's own framing implicitly assumed. This is the single highest-value,
   lowest-effort-to-act-on finding in this brief.
3. **The interest/profit scheduled-job gap (Area 9) matters specifically because GARBATI is Sharia-compliant,
   not despite it** — whatever profit-sharing mechanism a scholar eventually approves, Nigerian microfinance
   customers (often Mudarabah/Wadiah-style savings) will need *some* periodic accrual-and-post cycle sooner
   or later, and this research found that the infrastructure for that doesn't exist anywhere in GARBATI
   today, independent of the ratio question. Framed usefully: this is shared infrastructure with `ScheduleApi`
   (Area 7), not a separate problem — one scheduled-execution capability would plausibly serve both.
4. **Savings-product configurability gaps (Area 3) are real but second-order** — they matter once real
   savings products need lock-in periods or interest-posting cadences, which, again, is squarely where
   GARBATI's actual product (savings/current accounts for Nigerian retail/microfinance customers) lives, more
   than the loan-product gap (Area 4) does, since GARBATI has no financing module to compare against at all
   yet.
5. **Accounting/GL (Area 5), roles/maker-checker (Area 6), and reporting (Area 10) are real gaps, but are
   generic commercial-banking/core-banking-platform concerns, not microfinance-specific ones** — every
   commercial bank, not just an MFI, needs a chart of accounts, granular staff permissions, and reporting.
   These matter for GARBATI's eventual maturity as *a* core-banking platform, not specifically because it
   serves microfinance customers, and are correspondingly ranked lower here even though they are each real
   and independently confirmed gaps or deliberate differences.
6. **Loan/financing products (Area 4) and standing instructions (Area 7) are not really "gaps" at all in the
   sense this ranking cares about** — both are named, deliberate, already-tracked scope deferrals
   (D-28, "later tranche"), not surprises this research uncovered.

---

## What this does not settle

- **Does not design a Groups/Centers feature, a chart-of-accounts schema, a scheduled-job framework, a
  reporting layer, or a multi-tenancy mechanism** — every area above ends in a verdict and an honest reason,
  not a schema, an API shape, or a recommendation to build something, per the task's explicit scope.
- **Does not propose or imply any specific Sharia ruling, interest rate, or profit-sharing ratio.** Area 9 is
  stated, deliberately, as a pure infrastructure/mechanism question — the finding ("no scheduled-job
  mechanism exists anywhere in GARBATI") would be exactly the same regardless of what number a scholar
  eventually approves, and this brief does not suggest one.
- **Does not treat any of Fineract's own design choices as automatically correct for GARBATI to match.**
  Areas 5 and 6 specifically name real, reasoned justifications for why GARBATI's narrower choices may be
  the right ones *for GARBATI's current scale and target market*, not inferior copies of what Fineract does —
  while also naming honestly where that reasoning could stop holding as GARBATI grows.
- **Does not resolve the open multi-tenancy question (T-050/`ARCHITECTURE.md` Section 9)** — Area 8 adds a
  real, concrete data point (separate-database-per-tenant as Fineract's actual shipped answer) to that
  already-open question; the decision itself remains the owner's.
- **Does not independently verify several Fineract claims beyond what this research's own access could
  reach** — most notably the complete list of Fineract's shipped built-in reports (Area 10), the full prose
  description of every savings-product field (Area 3), and the exact mathematical formula behind Fineract's
  Progressive loan schedules (Area 4) — each flagged plainly in its own section above, not silently assumed.

## What this affects

This research directly names two already-open items in `ARCHITECTURE.md` Section 9, exactly as the task
requires, and does not introduce a new one:

- **Multi-tenancy (T-050's own open question, Section 9's "Sharia rules" entry).** Area 8 above is the
  concrete, real-world data point that entry's own framing asked for: Fineract's actual, shipped answer is a
  separate database per tenant with a small shared tenant-registry database and a tenant-identifying request
  header — not a tenant-id column added to shared tables, which is what that entry's own prior framing
  implicitly assumed as the likely shape. The coordinator/owner should read this as new input to that
  existing question, not a reason to open a new one.
- **Standing-order retry-on-failure behaviour (Section 9's existing P-3 entry).** Area 7 confirms Fineract's
  own real shape (a definition resource plus a separate run-history resource) matches `ScheduleApi`'s own
  already-written one-line scope statement in `ARCHITECTURE.md`'s service-map table — useful confirmation,
  not new information, for whoever eventually designs `ScheduleApi` and has to answer that still-open
  question.

Two further findings are plausible candidates for the coordinator to consider as genuinely new Section 9
entries, the same way T-056/T-057/T-058 each became (or were proposed as) their own entries:

- **Groups/Centers (group lending) — not researched before T-059.** GARBATI has zero equivalent anywhere,
  confirmed independently; this is the one microfinance-specific (not generic commercial-banking) pattern in
  this whole comparison, directly relevant to whenever GARBATI's financing module (D-28) is actually scoped.
- **No scheduled-job/background-worker mechanism exists anywhere in GARBATI today — not researched before
  T-059.** This is a structural, infrastructure-level absence this research found by reading the actual code
  across all six services, relevant to both `ScheduleApi`'s own eventual design and any future
  interest/profit-posting mechanism (Area 9) — a shared piece of missing infrastructure, not two separate
  gaps.

## Sources

- [Apache Fineract API Legacy Documentation](https://fineract.apache.org/docs/legacy/) (fetched and read
  directly, multiple times, for Areas 1, 2, 3, 4, 5, 7, 10 — the single most load-bearing primary source in
  this brief, same site T-056/T-058 already relied on)
- [Fineract Platform Documentation — docs/current (stable hub)](https://fineract.apache.org/docs/current/)
  (fetched and read directly, used only to confirm the real site-structure note above)
- [Fineract Database Docs — `stretchy_report` table](https://fineract.apache.org/docs/database/tables/stretchy_report.html)
  (fetched and read directly, Area 10)
- [Mifos-forge — Clients, Groups and Centers](https://mifosforge.jira.com/wiki/spaces/docs/pages/52035634/Clients+Groups+and+Centers)
  (fetched and read directly, Area 2)
- [Finecko — Multi-Tenancy in Apache Fineract](https://finecko.com/docs/fineract/features/multitenancy.html)
  (fetched and read directly, Area 8)
- [Mifos Docs — Pentaho Reporting Plugin](https://docs.mifos.org/external-3rd-apps/pentaho-reporting-plugin)
  (fetched and read directly, Area 10 — confirmed the plugin's separateness from Fineract's native reporting,
  though did not itself describe the native mechanism)
- Fineract's "Roles And Permissions" cwiki page and `finecko.com`'s maker-checker guide (both Area 6) — title/
  content via converging search-engine summaries; direct `cwiki.apache.org` fetch failed with `connect
  ECONNREFUSED`, the same recurring failure T-057/T-058 already documented for that host
- Fineract Chart-of-Accounts/Accounting-Rules hierarchical-structure claim (Area 5) — converging search-engine
  summary across several sources, not independently fetched as one single page
- Fineract savings interest-posting-period variants and loan Progressive/Cumulative schedule behavioral detail
  (Areas 3, 4, 9) — converging search-engine summaries of Fineract's own legacy/cwiki/academy documentation,
  not independently fetched in full for the behavioral (as opposed to field-name) claims specifically
  — [apache/fineract PR #2317, "Savings Interest Posting Job Fix"](https://github.com/apache/fineract/pull/2317)
  and [apache/fineract PR #2322, "Post Interest for Savings Job Fix"](https://github.com/apache/fineract/pull/2322)
  (titles via search, not independently fetched in full, Area 9)
- Fineract tenant-header mechanism (`Fineract-Platform-TenantId`) — converging search-engine summary of
  Fineract developer-configuration material, Area 8
- Internal: `ARCHITECTURE.md` (read directly — Sections 2, 7, 8, 9)
- Internal: `research/sharia-rule-configuration-patterns.md` (T-050), built on directly for Area 8, not
  re-derived
- Internal: `research/teller-till-vault-cash-operations.md` (T-056), `research/multi-charge-transaction-modeling.md`
  (T-057), `research/real-corebanking-payload-gap-analysis.md` (T-058) — built on throughout for sourcing-
  discipline/confidence-calibration conventions and specific prior findings (Areas 5, 6, 9), not re-derived
- Internal, read directly for GARBATI's own real current code (not `ARCHITECTURE.md`'s description of
  itself), across all ten areas:
  `Garbati.IdentityApi/src/Garbati.IdentityApi/Entities/{Customer,CustomerIdentityData,KycVerification,
  KycTierPolicy,KycStatus,CustomerStatus,Mandate,MandateApprovalPolicy}.cs`,
  `Garbati.IdentityApi/src/Garbati.IdentityApi/Authorization/IdentityScopes.cs`,
  `Garbati.ProductApi/src/Garbati.ProductApi/Entities/{Product,ProductLimitPolicy,AccountProductBinding,
  ProfitSharingTerms,FeeRule}.cs`,
  `Garbati.StaffIdentityApi/src/Garbati.StaffIdentityApi/Entities/{Role,StaffUser}.cs`,
  `Garbati.OpsApi/src/Garbati.OpsApi/Authorization/OpsRoles.cs`,
  `Garbati.OpsApi/src/Garbati.OpsApi/Entities/OpsReversalRequest.cs`,
  `Garbati.OrchestratorApi/src/Garbati.OrchestratorApi/Entities/AccountOpening.cs`,
  `LedgerApi/src/LedgerApi/Entities/{Account,AccountClass,AccountType}.cs`,
  `LedgerApi/src/LedgerApi/Authorization/LedgerScopes.cs`,
  `LedgerApi/src/LedgerApi/Controllers/Admin/SystemAccountsController.cs`,
  `LedgerApi/src/LedgerApi/Configuration/{FundingOptions,FeeIncomeOptions,TaxPayableOptions}.cs`
- Internal: `tasks/T-059-research-fineract-core-functionality-gap-analysis.md` (the task file itself, read in
  full)

## What was attempted and blocked or unreadable (stated plainly)

- `https://cwiki.apache.org/confluence/display/FINERACT/Group-Based+Financial+Inclusion+Methodologies` —
  `connect ECONNREFUSED`, the same `cwiki.apache.org` access failure T-057/T-058 already documented
  repeatedly in this project. Area 2's Fineract sourcing instead rests on the Mifos-forge page, which did
  fetch successfully.
- `https://cwiki.apache.org/confluence/display/FINERACT/Roles+And+Permissions` — `connect ECONNREFUSED`,
  same host, same failure mode. Area 6's permission-granularity detail instead rests on converging
  search-engine summaries, flagged accordingly.
- `https://cwiki.apache.org/confluence/spaces/FINERACT/pages/75958151/Deposit` — `connect ECONNREFUSED`,
  same host, same failure mode. Area 3's savings-interest detail instead rests on the legacy API
  documentation (field names, fetched directly) plus converging search summaries for behavioral prose.
- No independent, full-document read of any Fineract Islamic-banking/profit-sharing extension was possible —
  a search specifically for Fineract support of Musharaka/Mudarabah-style profit-sharing mechanics found no
  Fineract-specific result at all (every result returned was general Islamic-finance literature, not
  Fineract documentation), consistent with this research's own finding that Fineract's product model is
  built around conventional interest fields, not a profit-sharing-adjacent mechanism — stated as a real
  absence found, not a research gap papered over.
- No independent fetch of a single, complete Fineract document listing every shipped built-in report by
  name (Area 10) was obtained; the `stretchy_report` table structure itself was fetched and read directly,
  but its row-level content (the actual list of reports Fineract ships with) was not independently
  retrieved.
