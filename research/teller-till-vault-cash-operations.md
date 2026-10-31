# Teller till/vault cash operations in real banking, and whether GARBATI's existing architecture could model it (T-056)

**Task:** T-056. **Status:** Research only — no code, no schema, no feature design, no timeline. This is
forward-looking research informing a possible future task, the same way `research/staff-ops-identity-patterns.md`
(T-026) informed T-027/T-028 before anything was designed. GARBATI has no branch-deployment customer today;
nothing here is urgent, and nothing here is a recommendation to build.

**What this builds on, and does not re-derive:** the task file itself already confirmed, by a repo-wide
search across all six service repos, that GARBATI has zero matches for `Till`/`Vault`/`CashDrawer`/
`CashBalance`/`EndOfDay`/`EOD` anywhere in code or in any service's own `ARCHITECTURE.md`/`DESIGN.md`. This
brief does not re-verify that; it is already established fact. It also does not re-derive
`research/reconciliation-nibss-integration-patterns.md` (T-054) or `research/modern-treasury-gap-analysis.md`
(T-055)'s own findings about GARBATI's ledger model, but does use both as the baseline for the structural
comparison in Q5 below, and matches their sourcing/confidence-flagging conventions.

---

## Plain-language summary (read this first)

A real bank branch keeps physical cash in two places: a **vault** (the branch's bulk reserve, under joint/
dual custody so no single person can access it alone) and a **till** — a named sum of cash a specific teller
draws from the vault at the start of their own shift and is personally accountable for until they return it.
Moving cash from vault to till ("cash-out," the teller's opening float) and back ("cash-in," surplus sold
back mid-shift or the whole float returned at close) are real, controlled transactions, not informal
hand-offs. **"The till must balance" means a specific, two-sided check**: the system calculates what the
teller's drawer *should* contain (opening float, plus everything received from customers, minus everything
paid out, from the teller's own recorded transactions for that shift), and the teller (or a supervisor)
**physically counts the actual cash** and compares the two. A mismatch is a "cash difference" — short or
over — and what happens next (tolerate a small variance, escalate a larger one, who's accountable) is each
institution's own policy choice, not a universal rule; this research found real, named, but wildly different
dollar thresholds across different institutions' own published policies, the same "no universal number"
pattern T-054 already found for reconciliation-break tolerances and suspense-account SLAs.

Two real, named core-banking systems were found to model this concretely, with real sourcing-discipline
caveats stated plainly below: **Oracle FLEXCUBE** (a commercial core-banking platform, whose own published
glossary was read directly) represents a till as a configured entity linked to a **Cash GL account** (per
currency) and its own dedicated **Shortage/Overage GL account**, with real screen-level operations named
"Open Teller Till," "Till Position," and "Till Balancing." **Apache Fineract** (the open-source core-banking
engine GARBATI itself already takes its Apache 2.0 license cue from, per `TASKS.md` D-21) has real, currently
shipped `Teller`/`Cashier` entities with "allocate cash to cashier"/"settle cash from cashier" endpoints and
pre-existing `CASH_AT_MAINVAULT`/`CASH_AT_TELLER` general-ledger mapping constants — confirmed directly from
Apache's own documentation and a real, open GitHub pull request, not a secondary summary. A separate,
much richer Fineract "Teller Cash Management Redesign" pull request (shift-scoped cashier sessions, automatic
variance postings) was found too, but it was **closed, not merged**, and a Fineract maintainer publicly
flagged it as likely AI-generated and procedurally unacceptable — this research treats it as a useful,
honest data point about what a community contributor thought the gap looked like, not as confirmed,
shipped Fineract behaviour, and says so plainly rather than quietly upgrading its confidence.

**On Nigeria/microfinance specificity, stated plainly as the task requires:** no Nigeria-specific or
CBN-specific *regulatory or institutional-policy* primary source on teller/vault cash handling was found or
could be read — the CBN's own Microfinance Bank regulatory-guidelines PDF was fetched but rendered as
unreadable garbled text by this toolchain (the same `poppler-utils` limitation T-054 already documented), and
no CBN circular specifically addressing branch cash custody controls was located at all. The clearest
Nigeria-specific material actually found is **informal** — real, named Nigerian bank-teller and head-teller
job-description pages (not regulatory text) that independently confirm the same shape (float at shift start,
system-vs-physical reconciliation, differences reported to the Head Teller, dual-control vault participation,
a distinct "cash evacuation" term for moving bulk cash out of a branch) as the general, non-Nigeria-specific
literature. This research is honestly built mostly on general branch-banking and core-banking-vendor sources,
with Nigeria-specific color layered on top where it converges — not the reverse.

---

## Q1: The real cash custody chain — vault, till, cash-out, cash-in

**Confidence: medium.** The operational shape below converges consistently across several independent
sources, but most were read only as search-engine summaries of real pages (bankersonline.com forum threads
on this exact topic both returned HTTP 403 on direct fetch, consistent with that site's access pattern), not
independently fetched and read in full the way the Oracle/Fineract/profolio.ng sources below were.

- **The vault is never one person's to open.** It holds the branch's bulk cash and is explicitly under
  **dual control** — opening it, and any access to it, requires two authorized people acting together, so no
  single employee (teller or otherwise) can access the branch's bulk cash alone. A backup key or combination
  to even a single teller's own drawer is itself held under the same "joint custody" rule (two people
  required) rather than by one supervisor alone.
- **A till/cash drawer is assigned to one named teller for their shift and is that teller's own personal
  accountability**, distinct from the vault. Banks commonly set both a "working" limit (cash actively kept
  for transacting, often in a top drawer) and a "backup" limit (a locked reserve within the same drawer).
- **Cash-out (shift start):** the teller draws a fixed starting float from the vault, counted out under dual
  control and signed for by both parties — this is the number the teller's own balancing calculation will
  start from at close.
- **Cash-in (mid-shift or close):** when the drawer's cash climbs above its ceiling (a large deposit received),
  the excess is "sold" back to the vault; when it runs low on a denomination, the teller "buys" more from the
  vault. At close, the teller counts the drawer down to the fixed starting float again, selling the full
  remaining surplus back to the vault, and the branch secures everything back under dual control.
- [How Bank Branches Manage Cash from Vault to Teller Drawer — Sophisticated Cloud](https://www.sophisticatedcloud.com/all-blogs/how-bank-branches-manage-cash-from-vault-to-teller-drawer);
  [Is Your Branch Cash Secure? — Pinion Global](https://www.pinionglobal.com/blog/is-your-branch-cash-secure/)
  (both read via search-engine summary, not independently fetched and read in full)
- **Single-teller branches are a real, named edge case with compensating controls, not an unsolved problem.**
  Where a branch genuinely has only one person with vault access (no separate custodian role), the real-world
  compensating control named is a **surprise audit performed monthly by someone who has never had vault
  access**, plus requiring that other cash-adjacent functions (ATM balancing, branch-level reconciliation)
  be performed by someone other than that single custodian. [bankersonline.com forum — "Dual control in a
  small branch"] (title and gist confirmed via search-engine summary; the page itself 403'd on direct fetch)
- **Nigeria-specific corroboration (informal, not regulatory):** a real, directly-fetched Nigerian
  bank-teller job-description page states tellers "participate in dual-control vault operations, including
  cash lodgments, retrievals, and end-of-day vault balancing under the supervision of the Head Teller or
  Operations Supervisor," and names a distinct local term, **"cash evacuation,"** for coordinating the
  physical movement of bulk cash — a head-teller-level responsibility, not described elsewhere in the
  non-Nigeria sources this research found. [Bank Teller Job Description — Profolio.ng](https://www.profolio.ng/bank-teller/job-description)
  (fetched and read directly); Head Teller role detail ("plan work schedules, distribute and monitor teller's
  cash, investigate teller differences, assist tellers in balancing daily proof," "coordinate cash evacuation")
  read via search-engine summary aggregating several job-posting sites, not independently fetched as one
  single page.

**What I could not verify:** no primary bank-regulatory source (OCC, CBN, or otherwise) on vault dual-control
requirements was read directly in this research. The OCC's own "Cash Accounts" chapter of the Comptroller's
Handbook — a genuine US bank-regulator primary source on exactly this topic — was found by search but its
PDF could not be fetched at all in this research (connection refused, not a content issue). This is named
plainly as an access gap, not papered over.

---

## Q2: What "the till must balance" means, concretely, and what happens when it doesn't

**Confidence: medium-high** on the operational shape (several independent, converging sources, two read
directly); **low-medium** on specific escalation thresholds, which this research found are genuinely
institution-specific, not a cited regulatory figure.

- **Balancing is a two-sided reconciliation, not a trust-based sign-off, confirmed from two independently
  directly-read sources.** Oracle FLEXCUBE's own published glossary (fetched and read directly) defines
  **"Teller Till Balancing"** as "the task of reconciling the physical balance in a till with the balance
  according to the system records." [Oracle FLEXCUBE Universal Banking — Retail Teller Glossary](https://docs.oracle.com/cd/E99083_01/html/Retail_Teller/RT08_Gloss.htm)
  — worth flagging honestly: this same glossary frames the reconciliation as something "the operator performs
  ... during the Beginning of Day process," i.e. FLEXCUBE's own documentation ties balancing to the *start*
  of the next business day in its glossary wording, not strictly "before close of business" the way this
  task's own framing and every other source below describes it. This is a real, minor wording discrepancy
  across sources, named rather than smoothed over — the dominant description everywhere else (profolio.ng,
  the general cashier-balancing literature) is an end-of-shift/end-of-day count, and FLEXCUBE's own system
  clearly supports exactly that operation too (its "Till Balancing" (`DEDTILBL`) and "Open Teller Till"
  (`DEDTILOP`) are separate named functions) — this is a documentation-labelling nuance, not evidence that
  real balancing happens only at BOD.
- **The system-side "expected" figure is a calculation from the teller's own recorded transactions, not an
  estimate**: opening float, plus cash received from customers, minus cash paid out, for that teller/shift —
  exactly the arithmetic this task's own framing names. [Cashier balancing — Wikipedia](https://en.wikipedia.org/wiki/Cashier_balancing);
  [Understanding Teller Control and Balancing Procedures — CU*BASE](https://help.cubase.org/cubase/teller_control_and_balancing_procedures.htm)
  (title/gist via search; not independently fetched in full)
- **A "cash difference" is explicitly named "short" or "over," and the correction is a real general-ledger
  posting, not an informal note.** FLEXCUBE's glossary defines a dedicated **Shortage/Overage GL**: "the
  general ledger used for posting accounting entries related to differences between actual transaction
  amounts and amounts available in denominations" — confirming the difference is booked as a real debit or
  credit to a designated system account, the same shape as GARBATI's own fee-income/tax-payable system
  accounts (decision #46), just for a different purpose. [Oracle FLEXCUBE Glossary](https://docs.oracle.com/cd/E99083_01/html/Retail_Teller/RT08_Gloss.htm)
- **Thresholds for escalation/accountability are each institution's own policy choice, with real, named, but
  very different numbers — no universal figure found, matching T-054's own "no universal tolerance window"
  finding for reconciliation breaks.** Across several distinct, named institutional cash-handling policies
  found by search (not independently fetched/quoted verbatim in full — the specific dollar figures below are
  as summarized by the search tool from each named document, flagged at correspondingly lower confidence):
  a verbal warning at $5 cumulative shortage in a month at one institution; a $20 threshold counted as a
  reportable "frequency" at another; $25 aggregate before formal disciplinary action at a third; a single
  $100+ shortage reported to a director-level role at a fourth. [Cash Overages and/or Shortages Policy —
  Grambling State University](https://www.gram.edu/faculty/policies/docs/52202-Cash%20Over%20and%20Short%20Policy.pdf);
  [Reporting Cash Shortage/Overage — University of Memphis](https://www.memphis.edu/usbs/pdf/cash_handling_part_iv_2021.pdf);
  [Policy on Cash Handling and Management — ASI CSULB](https://www.asicsulb.org/corporate/documents/discover/policies/policy-on-cash-handling-and-management.pdf)
  — note plainly: these four are published US university cash-office policies, not banks, used here only
  because they are real, named, dollar-specific illustrations of the general point (the specific number is
  always an internal policy choice); they are not evidence of a banking-specific or Nigerian-specific figure.
- **Nigeria-specific corroboration on the accountability chain (informal, not regulatory):** the same
  directly-fetched Nigerian teller job-description page states "any over or short position must be reported
  immediately to the Head Teller and documented according to branch policy," and that "even minor
  discrepancies are flagged and investigated" with real career consequences — consistent in shape with every
  other source above, though again a careers/job-description page, not a regulatory or institutional-policy
  document. [Profolio.ng](https://www.profolio.ng/bank-teller/job-description) (fetched directly)

**What I could not verify:** no single source — Nigerian, regulatory, or otherwise — gives one canonical
escalation threshold or a universal definition of "small variance tolerated vs. large variance escalated."
This research treats that as a genuine finding (it is each institution's own policy), not a gap in the
research.

---

## Q3: How this is modeled in a real core banking system's own data model

**Confidence: medium-high** on Fineract (read directly from Apache's own current documentation and a real,
open, named GitHub pull request); **medium** on Oracle FLEXCUBE's deeper screen/GL-linkage detail, which
rests on a search-engine's summary of Oracle's own PDF documentation, not an independently re-verified full
direct fetch (every FLEXCUBE Retail Teller PDF attempted in this research returned unreadable binary/corrupted
content from this toolchain — a different instance of the same `poppler-utils` limitation T-054 already
documented, not a claim that the content doesn't exist).

**Oracle FLEXCUBE (commercial core banking, glossary read directly; screen/GL detail via search summary of
Oracle's own docs):**
- The glossary (read directly) defines **Till** as "a cash register used by a teller for the purpose of
  disbursing withdrawals and receiving deposits," usually linked to a safe or vault, and **Vault** as "a safe
  where the physical cash assets of a bank (or branch) are kept." [Oracle FLEXCUBE Glossary](https://docs.oracle.com/cd/E99083_01/html/Retail_Teller/RT08_Gloss.htm)
- Per the (search-summarized) FLEXCUBE Till/Vault Maintenance documentation: **"For each till or vault
  created in a branch, you must define a unique ID for the till/vault, the vault to which the till is linked,
  and the shortage/overage GL"** — and **"every currency maintained for a Till/Vault should be linked to a
  Cash GL,"** with transactions whose accounting-head is `TILL` posting to that configured Cash GL. Named
  function IDs confirm a real shift-scoped lifecycle: `DEDTILOP` ("Open Teller Till"), `DEDTILPO` ("Till
  Position"), `DEDTILBL` ("Till Balancing"), `DEDTVSET` (Till & Vault Parameter Maintenance). **A till, in
  this real commercial system, is explicitly something that opens, is positioned/queried mid-shift, and is
  balanced — not a permanent, always-on account.**
- This is the clearest concrete answer this research found to the task's own Q3 question ("is a teller's
  till commonly represented as its own kind of account in the GL sense"): **yes, but as a configured entity
  that points at a GL account (a Cash GL per currency, plus a dedicated Shortage/Overage GL), not necessarily
  as its own permanent chart-of-accounts row created once and never touched again** — the Till *object* has
  a shift lifecycle; the GL account it posts to can be comparatively static.

**Apache Fineract (open-source core banking; GARBATI's own Apache 2.0 license comes from this project's
example per `TASKS.md` D-21 — directly relevant as a real, comparable, same-license open-source system, not
a hypothetical):**
- Apache's own current, official API documentation (fetched directly) confirms real, already-shipped `Teller`
  and `Cashier` REST resources, including `POST tellers/{tellerId}/cashiers/{cashierId}/allocate` ("Allocate
  Cash To Cashier") and `POST tellers/{tellerId}/cashiers/{cashierId}/settle` ("Settle Cash From Cashier"),
  gated by permissions named `ALLOCATECASHTOCASHIER_TELLER`/`SETTLECASHFROMCASHIER_TELLER` — i.e. real,
  currently-functioning vault-to-cashier and cashier-to-vault movement endpoints exist in a real open-source
  core-banking codebase today. [Apache Fineract API Legacy Documentation](https://fineract.apache.org/docs/legacy/)
  (fetched and read directly)
- A real, currently open Fineract pull request (fetched and read directly) — fixing a validation
  inconsistency, not proposing new functionality — confirms Fineract **already has** pre-existing,
  already-shipped GL "financial activity" mapping constants `CASH_AT_MAINVAULT` (101) and `CASH_AT_TELLER`
  (102): "A deployment that maps them wrongly, which is easy since they are set once at configuration time,
  has every teller and vault movement landing on the wrong account." [apache/fineract PR #6311 — FINERACT-2778](https://github.com/apache/fineract/pull/6311)
  (fetched and read directly) — confirming, independently of the glossary-level FLEXCUBE finding above, that
  a second real system also represents vault cash and teller cash as two **distinct, named system/GL-mapped
  account types**, with cash movement between them as a real, GL-posting transaction.
- **An important, honestly-flagged caveat on a much richer finding:** a separate, much larger Fineract pull
  request (fetched and read directly) proposed a full "Teller Cash Management Redesign" — a new
  `m_cashier_sessions` table tracking session status (`OPEN`/`SETTLED`/`CLOSED`), transactions linked to a
  specific `cashier_session_id`, routing cash through a dedicated `11140 Teller Cash` GL account only while a
  session is open, and **automatic variance-correction postings** on an unbalanced settlement (to a
  `53920 Cash Shortage — Teller` account for shortfalls, or `43210 Miscellaneous Income` for overages). This
  is exactly the richer, session-scoped shape this task's Q5 asks about — **but this research must state
  plainly that this pull request was closed, not merged**, and a real Fineract maintainer publicly flagged it
  during review: "missing reference to a Jira ticket in the PR title! (waaaaayyy) too many commits... seems
  like most of this was done by some AI tool." [apache/fineract PR #5779](https://github.com/apache/fineract/pull/5779)
  (fetched and read directly, including the maintainer's review comment) — **this is not confirmed, shipped
  Fineract behaviour, and this research does not treat it as such.** It is used below only as an honest data
  point about what one real contributor's attempt at this exact gap looked like in shape (a session table,
  per-transaction session linkage, automatic variance GL postings), not as evidence that any production core
  banking system works exactly this way.

**What I could not verify:** the full Oracle FLEXCUBE Retail Teller PDF guide, in any release version
attempted, could not be rendered as readable text by this toolchain — every specific claim above attributed
to "search summary of Oracle's own docs" rests on a search engine's own extraction of that same unreadable
PDF content, not an independent direct read, and should be treated at correspondingly lower confidence than
the glossary page and the two Fineract sources that were genuinely fetched and read directly.

---

## Q4: What's different for a microfinance institution, and Nigeria specifically

**Confidence: low-medium**, stated plainly as the task requires — **no Nigeria-specific or
microfinance-specific regulatory or institutional primary source was found and successfully read** in this
research. What follows is mostly general branch-banking/core-banking-vendor material (Q1-Q3 above) with
genuinely Nigeria-specific color layered on top where it converges, not the reverse.

- **The named global bodies for microfinance-specific operational practice exist and were located**
  (MicroSave and CGAP both publish toolkits and training manuals specifically on MFI cash management, branch
  cash caps, and internal controls), **but their actual PDF content could not be reliably read by this
  toolchain** — attempted fetches returned unreadable binary/image-stream content, the same limitation
  affecting the FLEXCUBE PDFs above. This research therefore does **not** cite specific operational claims
  from MicroSave/CGAP, only notes their existence and general subject matter as a lead for a future
  researcher with better PDF-rendering tooling: [MicroSave — Financial Accounting Basic Toolkit](https://www.microsave.net/files/pdf/Financial_Accounting_Basic_Toolkit.pdf);
  [CGAP — Operational Risk Management for Microfinance Institutions](https://www.cgap.org/sites/default/files/researches/documents/CGAP-Training-Operational-Risk-Management-Course-2009.pdf)
  (both located, neither reliably read).
- **The one MFI-labelled document this research could read directly turned out to be a generic policy
  template, not a Nigeria- or Global-South-specific practice description — worth naming honestly rather than
  overclaiming its relevance.** A "Teller/Cashier Procedures" page on the Mifos X community wiki (fetched and
  read directly) is explicitly a fill-in-the-blank sample policy ("The total amount of cash maintained in any
  one drawer may not exceed (ENTER CASH LIMIT HERE)...") written in US-dollar/credit-union convention, meant
  as a starting point for an MFI's own policy manual, not a description of Mifos X's own software behaviour
  or of any specific institution's actual practice. [Mifos X User Zone — Teller/Cashier Procedures](https://mifosforge.jira.com/wiki/spaces/docs/pages/82706528)
  — its content (opening float from the vault under dual control, selling excess back to the vault during
  the day, counting cash three times before/during/after a transaction) matches the general shape in Q1-Q2
  above, which is itself a useful finding (MFI practice doesn't appear to differ structurally from general
  branch practice in this template's own framing), but it should not be read as confirmed MFI-specific or
  Nigeria-specific evidence.
- **The clearest genuinely Nigeria-specific findings are both informal (job-description content, not
  regulation or institutional policy), already used in Q1-Q2 above, repeated here for Q4's own framing:** a
  real Nigerian microfinance-sector teller job description explicitly separates a **"Teller"** role (handles
  all transaction types) from a distinct **"Cash Officer"** role at some institutions (focused specifically
  on vault management), and names a **"Vault Custodian"** as a real, named career step between teller and
  more senior operations roles — i.e. Nigerian microfinance practice, per this source, does distinguish
  teller-from-vault-custodian as separate roles even at relatively small institutions, not collapsing them
  into one person by default. [Bank Teller Job Description — Profolio.ng](https://www.profolio.ng/bank-teller/job-description)
  (fetched directly)
- **The single-teller-branch compensating-control finding from Q1 (monthly surprise audit by someone with no
  vault access; other cash functions performed by someone other than the sole custodian) is general
  small-branch banking practice, not sourced as microfinance-specific** — but it directly answers the task's
  own Q4 question about what changes when a branch has no separate vault-custodian role, and there is no
  reason found in this research to expect it would differ for a microfinance branch specifically; this is a
  reasoned application of Q1's general finding to Q4's specific question, not an independently-sourced
  microfinance claim, and is flagged as such.
- **A CBN-specific primary source was sought and not obtained — stated plainly, not silently skipped.** The
  CBN's own "Amended Regulatory and Supervisory Guidelines for Microfinance Banks" PDF was fetched as raw
  bytes but rendered only as garbled/corrupted text by this toolchain, the identical access pattern T-054
  documented repeatedly for `cbn.gov.ng` primary documents. No CBN circular specifically addressing branch
  cash custody, vault dual control, or teller balancing was located by search either (the CBN cash-policy
  material found by search concerns withdrawal/deposit limits and ATM vault replenishment monitoring, a
  different subject). **A qualified compliance officer must confirm whether any CBN-specific rule on branch
  cash custody or teller balancing exists at all** — this research did not find one, but cannot rule one out
  given the repeated, documented inability to read CBN's own primary documents directly.

---

## Q5: Does GARBATI's existing `AccountClass`/`LegPurpose` mechanism fit this, or does real practice need
something new? — a structural comparison, not a design

This is the question the task names as most directly useful to a future design task, so it is treated here
with the most care, and with an explicit effort not to force an answer either way beyond what Q1-Q4 actually
support.

**GARBATI's existing mechanism, read directly from `ARCHITECTURE.md` Sections 11-12 and `TASKS.md`'s own
decision log (not re-derived, summarized precisely because it matters for the comparison):** `LedgerApi`'s
`AccountClass` has exactly two values, `Customer` and `System`. System accounts are **permanent, pre-seeded
entities** — created once, in a database migration, one per currency, in fixed numbering ranges (decision
#25 for funding accounts, decision #46 for fee-income/tax-payable accounts) — discoverable via static
configuration (`FundingOptions`, `FeeIncomeOptions`, `TaxPayableOptions`) that fails startup validation if a
currency is missing. They never "open" or "close"; they are not assigned to a specific member of staff; they
exist for the lifetime of the deployment. `LegPurpose` is a fixed, short, deliberately closed list
(`Principal`/`Fee`/`Tax` only, decision #45, confirmed by the owner as not automatically extensible — a new
purpose code is "a new decision and a ledger code change," per `ARCHITECTURE.md` Section 9). A **chain**
posts a list of legs atomically, inside one database transaction, each crediting or debiting a real account,
with the rule that any Fee/Tax leg must credit a System-class account (the ledger checks the account's class,
not a specific account identity). A balance is always and only "the sum of this account's own ledger entries,
read live" (`AccountService.GetBalanceAsync`, confirmed directly by `research/modern-treasury-gap-analysis.md`)
— there is no external, non-ledger input anywhere in GARBATI's model that a balance is ever checked against.
`LedgerApi` itself "calls nobody" (decision #34) and has no concept of staff, shifts, or roles at all — that
knowledge lives in `StaffIdentityApi`/`OpsApi`, services `LedgerApi` never talks to.

**Where real till/vault practice fits this mechanism naturally, as a bounded extension:**

- **Vault-to-till and till-to-vault cash movement is, structurally, exactly what a chain/leg already does.**
  Both FLEXCUBE (Cash GL per Till/Vault) and Fineract (`CASH_AT_MAINVAULT`/`CASH_AT_TELLER`, confirmed real
  and shipped) represent this as a normal double-entry posting between two accounts — one debited, one
  credited, inside one atomic unit — which is precisely the shape GARBATI's existing `ChainService`/`LegPoster`
  already enforces for every transfer. A future "vault-to-till" or "till-to-vault" movement would plausibly
  need nothing structurally new from the chain-posting engine itself.
- **A cash-difference correction (shortage/overage) is, structurally, exactly what GARBATI's existing
  Fee/Tax-to-System-account mechanism already does.** FLEXCUBE's dedicated Shortage/Overage GL and (even in
  the unmerged Fineract redesign's own proposed shape) a dedicated shortage/variance account are both "post a
  correcting leg to a designated system-class account" — the same pattern decision #46 already built for
  fee-income/tax-payable. A new, narrowly-scoped `LegPurpose` value (something like `CashVariance`) and
  possibly one or two new System-account types (a shortage account, an overage account, or one combined
  account per FLEXCUBE's own pattern) would be a real but incremental, bounded extension of a mechanism that
  already exists — not a new mechanism.
- **GARBATI already has a working, built pattern for "more than one party must act together before this is
  allowed to happen"** — `IdentityApi`'s joint-account M-of-N approval and `OpsApi`'s staff-side maker-checker
  (both confirmed built, per `research/modern-treasury-gap-analysis.md` Section 8) — which is at least the
  right *shape* of mechanism a future dual-control vault-access requirement (Q1) could plausibly build on,
  though neither existing mechanism was built with vault access in mind and this is an observation, not a
  design claim.

**Where real till/vault practice needs something GARBATI's current architecture has no analogue for at
all — the honest, unforced finding:**

1. **A shift/session-scoped account lifecycle.** Every real source in Q1-Q3 — FLEXCUBE's own named "Open
   Teller Till" function, Fineract's cashier-to-teller assignment-with-dates, and even the unmerged Fineract
   redesign's `m_cashier_sessions` table with explicit `OPEN`/`SETTLED`/`CLOSED` status — treats a till as
   something that is **opened, bound to one specific staff member, used for one bounded period, and then
   closed/settled**, after which that same teller's next shift is, in some real systems, arguably a fresh
   instance. GARBATI's `AccountClass.System` accounts are the structural opposite of this on purpose: created
   once in a migration, numbered once, discoverable by static config, with no lifecycle, no owner, and no
   notion of "this instance is retired, make a new one for tomorrow." Modelling a till as just another
   permanent System account would be forcing a shift-scoped, staff-owned, opens-and-closes concept into a
   mechanism explicitly designed, by decision #25/#46, to be the opposite of that. This is the clearest
   candidate for "something genuinely new," not a stretch of what exists.
2. **A balance whose authoritative confirmation requires an external, human-supplied physical count — a kind
   of uncertainty nothing in GARBATI's current model has at all.** Every source in Q2 defines balancing as
   explicitly a two-sided reconciliation: the ledger's own calculated expected figure **compared against** a
   physical count performed by a person, with the difference itself a real, named, booked event (a shortage
   or overage) when the two don't match. GARBATI's `AccountService.GetBalanceAsync` computes a balance by
   summing entries and that number simply *is* the balance — full stop, with nothing external to check it
   against, for every account type that exists today. A till's "real" balance, by contrast, is never fully
   authoritative on the ledger's own say-so alone; it is provisional until a physical count confirms or
   contradicts it. This echoes, in a different and more literal way, the same gap
   `research/modern-treasury-gap-analysis.md` named for Modern Treasury's pending/posted/available balance
   tiers (which exist because an external rail can leave a balance genuinely unsettled for real time) — here
   the "external" thing a balance can diverge from is physical cash in a drawer, not a bank's settlement
   clock, but the shape of the gap (GARBATI currently has no notion of "this balance needs outside
   confirmation before it's trusted") is structurally the same kind of missing piece, independently arrived
   at from a completely different angle than T-055 took.
3. **GARBATI's "closest existing mechanism" — `OpsApi`'s testing-console `ledger.funding` scope — was
   deliberately built for something else and should not be stretched to cover this, per the task's own
   explicit instruction.** That scope lets a narrow, trusted ops channel credit an account directly, outside
   the normal customer-initiated flow, for test-data purposes. It is the nearest thing GARBATI has today to
   "move money into an account without a customer transaction initiating it," which is superficially
   adjacent to a vault-to-till cash-out — but it has none of what real till/vault practice actually needs:
   no shift binding, no dual control, no physical-count reconciliation, no shortage/overage posting, and it
   was never reviewed or designed with any of that in mind. Noting it here as the closest existing mechanism,
   exactly as the task asked, not as a foundation to build on without reconsideration.

**The honest, non-decided synthesis:** GARBATI's existing `AccountClass`/`LegPurpose` mechanism is
structurally well-suited to the *money-movement and correction-posting* half of till/vault practice — vault↔
till transfers and shortage/overage corrections both look like bounded, incremental extensions of a
mechanism that already exists (a new account type or two, a new leg purpose or two), not a new mechanism. It
is **not** suited, with nothing to stretch or extend, to the *lifecycle and confirmation* half — a till's
shift-scoped, staff-owned, opens-and-closes existence, and the fact that its balance needs an external,
physical human count before it can be trusted the way every other GARBATI balance is trusted today by
construction. Both of those missing pieces look, from this research alone, like they would involve
`StaffIdentityApi`/`OpsApi`/a future `OrchestratorApi` workflow (who holds a till, when it opens and closes,
who performed the physical count) more than they look like a `LedgerApi`-only extension — but that placement
question is explicitly left to a future design task, not decided here.

---

## What this does not settle

- **Does not design a till/vault feature, propose a schema, or recommend building anything**, per the task's
  explicit scope. Q5 names where GARBATI's existing mechanism fits and where it doesn't; it does not choose a
  shape, a service, or a schema for either half.
- **Does not assume urgency or propose a timeline.** GARBATI has no branch-deployment customer today; this
  fills a real gap in the platform's own self-understanding of a role it already defines (`Teller`), not a
  response to an active deadline.
- **Is not a Nigerian regulatory ruling and does not claim CBN has (or hasn't) a specific rule on this topic.**
  Q4 states plainly that no CBN primary document on branch cash custody was found or could be read, and that
  this absence is a research limitation, not evidence of absence. **A qualified compliance officer must
  confirm whether any CBN/NDIC rule on branch cash handling, vault custody, or teller balancing applies to
  Nigerian microfinance banks before any future design relies on this brief's silence on the question.**
- **Is not a Sharia ruling and does not touch Sharia questions.** Cash custody and till balancing are
  operational/physical-security topics; nothing in this brief bears on GARBATI's existing Sharia-governance
  open questions.
- **Treats the unmerged Fineract "Teller Cash Management Redesign" pull request (Q3) honestly as rejected/
  unconfirmed, not as evidence of what any production system actually does** — repeated here because it is
  the single most design-shaped piece of evidence this research found, and the easiest one to accidentally
  overstate.
- **Does not independently confirm the specific dollar-threshold figures cited in Q2** beyond what a
  search-engine summary reported from each named, real policy document; none were independently fetched and
  quoted verbatim in full.

## What this affects

This is new research, not an update to an existing cited gap — `ARCHITECTURE.md` does not currently name
teller/till/vault cash operations anywhere in Section 9's open questions (the task file's own repo-wide
search confirmed zero mentions). The coordinator should consider whether this research itself becomes a new
Section 9 entry (e.g., "Teller cash custody / till-vault modeling — not researched before T-056; see
`research/teller-till-vault-cash-operations.md`") the same way `research/staff-ops-identity-patterns.md` and
`research/sharia-rule-configuration-patterns.md` are already cited inline next to their respective open
questions, given that `StaffIdentityApi` already has a `Teller` role with no cash-handling workflow behind
it. This research does not resolve what that workflow should be — Q5 states plainly which half of the
problem GARBATI's existing `AccountClass`/`LegPurpose` mechanism can plausibly extend to cover (vault↔till
movement, shortage/overage correction) and which half it structurally cannot (shift-scoped account
lifecycle, physical-count-dependent balance confirmation) — as the grounding a future design task should
start from, not a decision in itself.

## Sources

- [Oracle FLEXCUBE Universal Banking — Retail Teller Glossary](https://docs.oracle.com/cd/E99083_01/html/Retail_Teller/RT08_Gloss.htm) (fetched and read directly)
- [Apache Fineract API Legacy Documentation](https://fineract.apache.org/docs/legacy/) (fetched and read directly)
- [apache/fineract PR #6311 — FINERACT-2778, teller/vault financial-activity mapping fix](https://github.com/apache/fineract/pull/6311) (fetched and read directly)
- [apache/fineract PR #5779 — "Teller Cash Management Redesign" (closed, not merged, maintainer flagged as likely AI-generated)](https://github.com/apache/fineract/pull/5779) and [its file diff](https://github.com/apache/fineract/pull/5779/files) (both fetched and read directly)
- [Bank Teller Job Description — Profolio.ng (Nigeria)](https://www.profolio.ng/bank-teller/job-description) (fetched and read directly)
- [Mifos X User Zone — Teller/Cashier Procedures (sample policy template)](https://mifosforge.jira.com/wiki/spaces/docs/pages/82706528) (fetched and read directly)
- [How Bank Branches Manage Cash from Vault to Teller Drawer — Sophisticated Cloud](https://www.sophisticatedcloud.com/all-blogs/how-bank-branches-manage-cash-from-vault-to-teller-drawer) (read via search-engine summary)
- [Is Your Branch Cash Secure? — Pinion Global](https://www.pinionglobal.com/blog/is-your-branch-cash-secure/) (read via search-engine summary)
- bankersonline.com forum — "Proper cash handling and teller controls" and "Dual control in a small branch" (titles/gist confirmed via search-engine summary; both pages returned HTTP 403 on direct fetch)
- [Cashier balancing — Wikipedia](https://en.wikipedia.org/wiki/Cashier_balancing) (read via search-engine summary)
- [Understanding Teller Control and Balancing Procedures — CU*BASE](https://help.cubase.org/cubase/teller_control_and_balancing_procedures.htm) (read via search-engine summary)
- [Cash Overages and/or Shortages Policy — Grambling State University](https://www.gram.edu/faculty/policies/docs/52202-Cash%20Over%20and%20Short%20Policy.pdf) (title/figures via search-engine summary, not independently fetched)
- [Reporting Cash Shortage/Overage — University of Memphis](https://www.memphis.edu/usbs/pdf/cash_handling_part_iv_2021.pdf) (title/figures via search-engine summary, not independently fetched)
- [Policy on Cash Handling and Management — ASI CSULB](https://www.asicsulb.org/corporate/documents/discover/policies/policy-on-cash-handling-and-management.pdf) (title/figures via search-engine summary, not independently fetched)
- [MicroSave — Financial Accounting Basic Toolkit](https://www.microsave.net/files/pdf/Financial_Accounting_Basic_Toolkit.pdf) (located, not reliably read)
- [CGAP — Operational Risk Management for Microfinance Institutions](https://www.cgap.org/sites/default/files/researches/documents/CGAP-Training-Operational-Risk-Management-Course-2009.pdf) (located, not reliably read)
- Internal: `ARCHITECTURE.md` Sections 9, 11, 12 (read directly — `AccountClass`, `LegPurpose`, decisions #34/#42/#44/#45/#46)
- Internal: `TASKS.md` D-21 (Apache Fineract as GARBATI's own Apache 2.0 license precedent), `tasks/T-004-ledger-fee-tax-accounts.md` (decision #46 detail, read directly)
- Internal: `research/reconciliation-nibss-integration-patterns.md` (T-054) and `research/modern-treasury-gap-analysis.md` (T-055), built on for sourcing/confidence conventions and the Q5 structural comparison, not re-derived
- Internal: `tasks/T-056-research-teller-till-vault-cash-operations.md` (the task file itself, read in full)

## What was attempted and blocked or unreadable (stated plainly)

- `https://www.occ.gov/publications-and-resources/publications/comptrollers-handbook/files/cash-accounts/pub-ch-cash-accounts.pdf` — connection refused on direct fetch. A genuine US bank-regulator primary source on cash-accounts controls; not read at all in this research.
- `https://www.bankersonline.com/forum/ubbthreads.php/topics/1042626/proper-cash-handling-and-teller-controls` and `.../topics/51782/dual-control-in-a-small-branch` — both HTTP 403 on direct fetch; used only via search-engine summary.
- `https://www.tcs.com/.../tcs-bancs-research-journal-16-understanding-cash-handling.pdf` — HTTP 403 on direct fetch.
- `https://www.cbn.gov.ng/out/2013/ccd/amended%20regulatory%20and%20supervisory%20guidelines%20for%20mfb.pdf` — fetched as raw bytes but rendered as unreadable/corrupted text by this toolchain, the same `poppler-utils`-related limitation `research/reconciliation-nibss-integration-patterns.md` already documented for `cbn.gov.ng` PDFs. No CBN-specific content on branch cash handling was obtained from this or any other source.
- Oracle FLEXCUBE's full "Retail Teller" guide PDF, every release version attempted — fetched as raw bytes, rendered as unreadable binary/image-stream content by this toolchain. The glossary HTML page was read directly instead and is the only Oracle source in this brief treated as fully, independently verified.
- `https://www.microsave.net/files/pdf/Financial_Accounting_Basic_Toolkit.pdf` and `https://www.findevgateway.org/.../mfg-en-toolkit-mfi-internal-audit-and-controls-trainers-manual-aug-2007.pdf` — both fetched as raw bytes, neither rendered as reliably quotable text; the latter's fetch attempt produced only generic, non-quoted paraphrase that this research chose not to cite as a verified claim.
- `https://cwiki.apache.org/confluence/display/FINERACT/Teller` and `https://issues.apache.org/jira/browse/FINERACT-176` — both connection refused on direct fetch.
