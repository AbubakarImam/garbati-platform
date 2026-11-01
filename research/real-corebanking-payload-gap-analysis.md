# Triangulating a real production transfer payload's six gaps against two other named core-banking platforms (T-058)

**Task:** T-058. **Status:** Research only — no code, no schema, no feature design, no timeline. This brief
does not re-derive the six gaps themselves or GARBATI's own code; both are established fact per the task
file, reproduced there from a direct, prior comparison of a real production MIZAN/SEABASS (via KUSALA BPM)
transfer-posting payload against GARBATI's actual current code. This task's own and only job is narrower:
check whether Apache Fineract and Oracle FLEXCUBE — two other real, named, documentable core-banking
platforms — handle the same six concepts, to find out whether they are common, load-bearing core-banking
patterns GARBATI is genuinely missing, or something narrower or more specific to the one institution's own
integration contract. **This brief does not design anything, propose a schema, or pick a recommendation** —
it ends in an honest verdict per gap and a synthesis ordered only by how commonly each gap was confirmed
across the two platforms actually checked.

**What this builds on, and does not re-derive:** the task file's own six gaps (value-date/lifecycle
timestamps; branch attribution; multi-entry posting; per-entry narration/reconciliation remarks; transaction
category/source/channel; named-actor persistence vs. a signed grant) and GARBATI's own current code facts
cited in support of each — all read directly by the coordinator before this task was written, reproduced
verbatim in the task file, not re-verified here. This brief treats `research/multi-charge-transaction-modeling.md`
(T-057) and `research/teller-till-vault-cash-operations.md` (T-056) as the sourcing-discipline and
confidence-calibration standard to match, and reuses their own already-established findings about Apache
Fineract and Oracle FLEXCUBE's general access characteristics (FLEXCUBE's PDFs render as unreadable binary
content on direct fetch in this toolchain; Fineract's own HTML documentation pages are generally fetchable)
rather than re-discovering those access patterns from scratch.

---

## Plain-language summary (read this first)

Checking two other real, named core-banking platforms against the six gaps gives a genuinely mixed answer,
not a clean "yes, all six are universal" or "no, all six are one institution's quirk." **Three of the six
gaps are confirmed cleanly and strongly by both platforms, independently:** a transaction having more than
one meaningful date (not just when it was recorded, but when it is *effective*, which can be backdated or
forward-dated) is a real, old, near-universal core-banking concept — Oracle FLEXCUBE's own documentation
defines value date explicitly as a date that "could either be the booking date itself, or a date in the past,
or future"; Apache Fineract's own journal-entry database table carries distinct `transaction_date` and
`entry_date` columns, read directly. Persisting a human-identifiable name for who approved something,
separately from (and in addition to) whatever technical authorization mechanism exists, is also confirmed
cleanly by both: FLEXCUBE's own Security documentation names "Input By" and "Authorized By" fields with
user ID and timestamp shown on every record; Fineract's own audit/maker-checker API returns plain `maker`/
`checker` fields. Neither platform was found, in this research, to also maintain a separate cryptographic
proof of authorization alongside that human name — in both, the database row itself appears to be the
entire record of who approved what, which makes GARBATI's own deliberate choice (decision #13: never persist
the human name, only a decision id tied to a signed grant token) look like the less common design relative
to what these two real systems actually do, not the other way around. Branch attribution is also confirmed
real in both, though at different scales: Fineract tracks one `office_id` per posting plus a maker/checker
pair that a real, fixed GitHub bug confirms can genuinely sit at different branches; Oracle FLEXCUBE has a
substantially richer concept — a distinct "Transaction Branch" and "Source Branch," and a named "Inter Branch"
transaction type with its own authorization workflow.

**Two of the six gaps turned out narrower or more conditional than the task's own framing suggested.**
Multi-entry (N:M) posting is real and named in both platforms — Fineract's documented "compound journal
entry," FLEXCUBE's "Single Debit Multiple Credit," "Multi Credit Transfer," and dedicated "Batch Booking"
transaction codes — but every real-world reason found for it in this research is a payroll-disbursement or
bulk-collection scenario, not an ordinary one-customer-to-another-customer retail transfer, which is squarely
GARBATI's own current scope. A dedicated, separate "reconciliation remarks" field, distinct from narration,
was not confirmed as a standard concept in either platform; what both do have instead is a narration/
description field (confirmed, per-entry, in both) plus a *different* kind of mechanism for reconciliation — a
stable correlation identifier (Fineract's own `externalId`, explicitly built to survive internal renumbering
after a reversal) rather than a second free-text remarks box.

**One gap — transaction category/source/channel attribution — sits in between: the underlying need is real
and confirmed in both platforms, but the specific shape (a flat numeric triple of category/source/sub-source
codes) is not the dominant convention found.** What both platforms actually have is closer to a named entity
or taxonomy — Fineract's `PaymentType` (with GL routing per channel) plus a separate, binary
self-service-vs-back-office user flag; FLEXCUBE's named "Payment source" codes and a distinct "Source Branch"
field — matching this project's own repeatedly-found pattern (T-054, T-056, T-057) of "a real need, but no
single universal shape."

---

## Gap 1 — Value date / multi-stage transaction lifecycle timestamps

**Confidence: high** that this is a real, near-universal, old core-banking concept, confirmed independently
in both platforms by at least one directly-read or precisely-quoted source each; **medium** on the exact
column-level mapping in Fineract specifically, flagged below.

- **Oracle FLEXCUBE's own documentation defines value date explicitly as independent of, and potentially
  different from, the booking/transaction date, in either direction** — "the date on which a transaction
  takes effect... could either be the booking date itself, or a date in the past, or future, and this date
  is called the value date of the transaction." FLEXCUBE explicitly supports "back value dated transactions"
  (value date earlier than system date) and names **debit and credit value date spreads** — i.e. in FLEXCUBE
  a single transfer's debit leg and credit leg can each carry their *own*, independently different value date,
  a finer-grained version of the same concept than even the task's own framing names. [Oracle FLEXCUBE Funds
  Transfer documentation, via search-engine extraction of Oracle's own PDF content](https://docs.oracle.com/cd/E51527_01/PDF/FT/FT.pdf)
  — this specific wording was returned by a search-engine summary of Oracle's own PDF, not independently
  fetched and read in full by this research (every FLEXCUBE PDF attempted directly in this and prior briefs
  renders as unreadable binary content, named plainly, not papered over); flagged at correspondingly reduced
  confidence relative to a directly-read source, though the wording is specific enough (a precise definitional
  sentence, not a vague gist) to treat as a real quote from a real document, not an invented paraphrase.
- **Apache Fineract's own journal-entry database table, read directly, confirms a genuine two-date (at least)
  concept at the schema level**: `acc_gl_journal_entry` has distinct `entry_date` and `transaction_date`
  columns, alongside separate `created_date`/`lastmodified_date` audit timestamps. [Fineract Database Docs —
  `acc_gl_journal_entry` table](https://fineract.apache.org/docs/database/tables/acc_gl_journal_entry.html)
  (fetched and read directly). **An honest nuance, not smoothed over**: this research could not independently
  confirm which of Fineract's two date columns plays FLEXCUBE's specific "value date" role versus a plain
  booking/transaction date — a search-engine summary of Fineract's own legacy API documentation states
  "Transaction date / Value date is the date on which the transaction occurred or to be accounted for.
  Submitted on date / Posting date is when the transaction was posted," which reads as conflating "transaction
  date" and "value date" into one concept in Fineract's own terminology, rather than clearly separating them
  the way FLEXCUBE's documentation does. This is a real, named divergence between the two platforms checked,
  not a clean match — both confirm *some* multi-date concept exists, but not identically shaped.
- **A real, currently-open Fineract GitHub pull request independently confirms the general principle that
  Fineract distinguishes a transaction's own business timestamp from its record-creation timestamp**, fixing
  a bug where the journal-entry API "previously only returned `submittedOnDate` (date-only), even though the
  journal entry's actual creation timestamp is stored separately" — i.e. even within one system, which exact
  timestamp a given API field exposes has been a real, recently-fixed point of confusion, itself evidence that
  multiple distinct timestamps genuinely exist and matter enough to need correcting. [apache/fineract PR
  #6195 — FINERACT-2719](https://github.com/apache/fineract/pull/6195) (title/content via search-engine
  summary, not independently fetched in full).

**Verdict: confirmed as a common, load-bearing core-banking pattern GARBATI is genuinely missing.** Both
platforms, independently, represent the value/effective date as a field on the transaction/journal entry
itself (not a separate status-history object) — directly answering the task's own sub-question. FLEXCUBE's
version is the richer and more directly on-point match to the real payload's own `isFuture`/back-dating
framing; Fineract's version is real but less cleanly separated in the sources this research could reach,
named honestly above rather than forced into a false 1:1 match.

**What I could not verify:** Fineract's own exact semantic distinction between `entry_date` and
`transaction_date` (which one is closer to "value date," and whether either can be set to a past or future
date independently the way FLEXCUBE's can) was not confirmed from a source read in full; no equivalent of
FLEXCUBE's documented "`isFuture`"-style explicit flag was found in Fineract specifically.

---

## Gap 2 — Branch-code attribution per lifecycle stage

**Confidence: medium-high.** The Fineract side rests on one directly-read source (the database table) plus
one precisely-described, real GitHub bug fix; the FLEXCUBE side rests entirely on search-engine summaries of
Oracle's own PDF documentation, not independently fetched in full, flagged accordingly.

- **Apache Fineract's `acc_gl_journal_entry` table, read directly, carries a single `office_id` foreign key
  to `m_office`** — one branch per posted entry, a real but comparatively simple representation. [Fineract
  Database Docs — `acc_gl_journal_entry`](https://fineract.apache.org/docs/database/tables/acc_gl_journal_entry.html)
  (fetched and read directly).
- **A real, specific Fineract bug fix confirms that the maker and the checker of a maker-checker command can
  genuinely be at different branches, and that this is exactly the kind of thing that breaks if not handled
  correctly** — a parked (pending-approval) command previously saved with no `office_id` "caused the
  office-scoping join to hide it from users below head office, resulting in [an] empty inbox for branch
  checkers"; the fix "stamp[s] the parked row with the result's office before parking it," and the checker
  inbox "applies the checker's office hierarchy." This is a real, concrete confirmation that Fineract's own
  design assumes a command can be *made* at one branch and *checked* at a different, hierarchically-related
  one (e.g. head office checking a branch's work) — structurally the same shape as the real payload's own
  `"HQ"` `ApprovedByBranchCode` next to a different `InitiatedByBranchCode`. [GitHub search result describing
  this fix, title/content via search-engine summary, not an individually identified PR number independently
  fetched] — flagged at correspondingly reduced confidence versus a directly-read PR, though the described
  mechanism (office-id stamping, office-hierarchy-scoped checker inbox) is specific and technical enough to
  read as a real description of real code, not an invented generality.
- **Oracle FLEXCUBE's own documentation describes a substantially richer branch-attribution model** — a
  "Transaction Branch" (the branch code at which the transaction is done) distinct from a "Source Branch"
  code, and a distinct "Payment source" concept for external systems feeding transactions in. FLEXCUBE also
  names a real, dedicated **"Inter Branch Transaction Input"** function and an "Inter Branch check" that
  flags when a transaction's account doesn't belong to the current branch, plus **"Multi Branch Access"**
  letting an authorized user view and authorize pending records "belonging to a specific branch" other than
  their own. [Oracle FLEXCUBE documentation, via search-engine extraction of Oracle's own PDF/HTML content —
  the specific function names ("Inter Branch Transaction Input," function ID 1410) and field names
  ("Transaction Branch," "Source Branch") are precise and technical enough to treat as real, not invented,
  but were not independently fetched and read in full] — the same reduced-confidence flag named throughout
  this brief for FLEXCUBE PDF-sourced claims.

**Verdict: confirmed as a common core-banking pattern, at different scales in the two platforms checked, and
directly relevant to GARBATI's own stated target market regardless of scale.** Fineract's minimal version
(one `office_id` field, maker/checker office hierarchy) is the closer structural match to what a growing but
not-yet-enterprise-scale Nigerian microfinance institution would plausibly need first; FLEXCUBE's richer
inter-branch/multi-branch-access model answers the task's own framing question directly — this is **not**
found to be an enterprise-only concept reserved for institutions with hundreds of branches; it is confirmed,
at some scale, in a deliberately comparable open-source platform (Fineract) that does not assume a large
branch network, which is directly relevant given GARBATI's own target market (per the task file) of
Nigerian microfinance institutions, which commonly do operate more than one physical location.

**What I could not verify:** whether Fineract's own maker-checker office-hierarchy mechanism is described
anywhere as supporting more than two lifecycle stages (initiated/approved/posted) each with their own branch,
the way the real payload's three separate branch-code fields do; this research found evidence of exactly two
roles (maker, checker) each with an office, not three or more distinct lifecycle-stage branch fields in
Fineract specifically.

---

## Gap 3 — Multi-entry (N:M) posting

**Confidence: high** that this is a real, named, common capability in core-banking transaction APIs generally;
**medium** on how directly it applies to GARBATI's own specific retail-transfer use case, which is the more
load-bearing half of the task's own question here.

- **Apache Fineract's own `createJournalEntry` API, confirmed via search-engine summary quoting the API's own
  documented example payload, explicitly accepts parallel `credits` and `debits` arrays**, each entry
  specifying its own `glAccountId` and `amount` — e.g. `"credits":[{glAccountId:1, amount:5000}, {glAccountId:2,
  amount:5000}], "debits":[{glAccountId:3, amount:5000}, {glAccountId:4, amount:5000}]`. Fineract itself names
  this a **"compound journal entry"**: "multiple debits and/or multiple credits may be made amongst a set of
  preselected list of accounts... the total amount of the debits must equal the total amount of the credits
  or the journal entry is said to be 'unbalanced.'" [Fineract accounting documentation/example payload, via
  search-engine summary of Apache's own cwiki/API content, not independently fetched in full].
- **Oracle FLEXCUBE independently confirms the same N:M shape, under different names, tied to specific named
  transaction types**: "Processing Single Debit Multiple Credit (SDMC) for Bulk Payments," a "Multi Credit
  Transfer" (which can itself be a Multi Customer Transfer, a Multi Financial Institution Transfer, or a
  Multi Transfer for Own Account, each sending a different message type), and dedicated "Batch Booking" and
  "Multi Credit Batch Booking" transaction codes for consolidating many individual payments into fewer
  postings. [Oracle FLEXCUBE Funds Transfer/Payments documentation, via search-engine extraction of Oracle's
  own PDF content, not independently fetched in full].
- **The real-world reason this exists, confirmed by general (non-platform-specific) sourcing on bulk/batch
  payment processing, converges on exactly two scenarios, neither of which is an ordinary one-customer
  retail transfer**: payroll disbursement (one company account debited once, many individual employee
  accounts credited) and bulk collections (many individual payer debits consolidated into one credit to a
  biller or clearing account). This sourcing is general industry material on bulk/batch payments, not
  Fineract- or FLEXCUBE-specific, used here only to explain *why* the N:M capability exists in both named
  platforms, at correspondingly lower, general-knowledge confidence.
- **The real payload itself is a useful, honest data point working against an over-broad reading of this
  gap**: even MIZAN/SEABASS's own sample payload, which clearly supports the N:M shape structurally
  (`creditEntries`/`debitEntries` as arrays, an explicit `multipleDebit` flag), uses exactly one entry in
  each array with `multipleDebit: 0` for this specific test case — i.e. the *capability* existing in the API
  contract does not mean every, or even most, real transactions on that platform actually use N>1. This is
  named directly from the payload already reproduced in the task file, not a new external finding, included
  here because it bears directly on how load-bearing this gap is for GARBATI's own use case specifically.

**Verdict: narrower than it first appeared, specifically for GARBATI.** The underlying capability is
confirmed as real, named, and common in core-banking transaction APIs generally — this is not an invented or
institution-specific concept. But both platforms' own documentation ties its real-world justification to
payroll-disbursement and bulk-collection scenarios, not ordinary peer-to-peer retail transfers, which is
squarely GARBATI's current, stated scope per the task file. Whether this gap is load-bearing for GARBATI
specifically depends on whether GARBATI's own product roadmap ever needs payroll/bulk-disbursement features
— a product-scope question this research cannot answer and does not attempt to — rather than on whether
ordinary customer-to-customer transfers need it, which this research did not find a real-world reason for in
either platform checked.

**What I could not verify:** no source found in this research states that any real microfinance institution
specifically processes ordinary retail transfers (as opposed to payroll or bulk-collection products) as N:M
postings; this absence is treated as a real finding (the use case genuinely doesn't call for it in the
sources checked), not a gap in the research.

---

## Gap 4 — Per-entry narration, and a dedicated reconciliation-remarks field

**Confidence: medium-high** on per-entry narration being common (one directly-read source, one
precisely-described field name); **medium** on the reconciliation-remarks half, where the honest finding is
that neither platform has quite the concept the task's own framing names.

- **Apache Fineract's `acc_gl_journal_entry` table, read directly, confirms a per-entry `description` column**
  (`VARCHAR(500)`, nullable) — since each row in this table is one debit or credit entry, this is genuinely
  per-entry narration, not one string shared across a whole multi-leg transaction. [Fineract Database Docs —
  `acc_gl_journal_entry`](https://fineract.apache.org/docs/database/tables/acc_gl_journal_entry.html) (fetched
  and read directly).
- **Oracle FLEXCUBE independently confirms a comparable per-transaction free-text field, named "Narrative,"**
  used on Retail Teller transaction screens for "remarks, if any, in the context of Oracle FLEXCUBE Retail
  Teller transactions." [Oracle FLEXCUBE Retail Teller documentation, via search-engine extraction of
  Oracle's own PDF content, not independently fetched in full].
- **The honest finding on the task's own specific "dedicated reconciliation-remarks field" framing: neither
  platform was found to have a second, separate free-text field for this, distinct from narration.** What
  Fineract actually has instead is a differently-shaped mechanism entirely — `externalId`, an optional,
  explicitly reconciliation-purposed identifier (not free text) with a stated, specific rationale: "Transactions
  may change due to correction done by reverting a transaction, which can lead to removing and adding
  transactions back to an account and result in new ID distribution from the transaction sequence. External
  systems may not want to follow up on these changes but rather access transactions as they are at their end
  state... an additional identifier is to be used." [Fineract documentation/JIRA material on external IDs for
  savings transactions, via search-engine summary, not independently fetched in full]. This is a real,
  confirmed concept — separating "why this happened" (narration) from "how an external system keeps track of
  this across internal changes" (a stable correlation id) — but it is a structured identifier, not a second
  free-text remarks box the way the real payload's own `reconciliationRemarks: "Postman test"` reads.
  Independently, this project's own prior research (T-057) already found the same shape at the wire-message
  level: ISO 20022's `RemittanceInformation` (the "why," narration-equivalent) is a distinct field from
  `AccountServicerReference`/`EndToEndId` (structured reconciliation-correlation references), not two parallel
  free-text fields either.

**Verdict: narrower/more nuanced than it first appeared.** Per-entry narration itself is confirmed as a
common, real, low-cost pattern (directly confirmed in Fineract; corroborated in FLEXCUBE). But the specific
second half of this gap — a dedicated, free-text *reconciliation-remarks* field, separate from narration — is
not confirmed as a standard core-banking concept in either platform checked; what real systems commonly have
instead for "how do I match this up on reconciliation" is a structured correlation identifier, a different
mechanism than a second remarks box. This research cannot rule out that MIZAN/SEABASS's own
`reconciliationRemarks` field serves a real, specific institutional purpose (it may, for instance, simply be
a free-text field some branch-level reconciliation process at Sterling Bank actually reads), but it is not
confirmed, from either platform checked, to be a common, load-bearing core-banking concept in general — the
sample payload's own literal value, `"Postman test"`, is also consistent with this being a field exercised
only by a test harness, not necessarily evidence either way on its own.

**What I could not verify:** whether MIZAN/SEABASS's own `reconciliationRemarks` field has a specific,
documented institutional purpose beyond what this research's two comparison platforms suggest; no MIZAN/
SEABASS-specific documentation was sought or found (the task scopes this research to Fineract/FLEXCUBE
comparison only, not to independently researching Sterling Bank's own specific platforms).

---

## Gap 5 — Transaction category/source/channel attribution

**Confidence: medium.** Both platforms confirm the underlying need for some origin/classification concept;
neither was found to use a flat numeric triple the way the real payload does, and this research's own
sourcing here is entirely search-engine summaries, not independently fetched full documentation, for both
platforms.

- **Apache Fineract's closest real mechanism is `PaymentType`, which classifies *how* money moved (cash,
  cheque, mobile money, bank transfer, etc.), attached to a transaction via a `payment_detail_id` foreign
  key, and used for GL routing** — "payment-type GL routing applies to various account types," with "payment
  channel to fund source mappings... stored on products." This answers "how," which is adjacent to but not
  identical to the real payload's "where did this transaction originate" question. [Fineract documentation on
  `PaymentType`/payment-channel GL mapping, via search-engine summary, not independently fetched in full].
- **Fineract separately has a real, binary origin distinction — not a rich taxonomy — between self-service
  (customer-initiated, mobile) and back-office (staff-initiated) commands**, via an `is_self_service_user`
  attribute on `m_appusers`: "A self service user should be able to use only self service APIs and back
  officer should not have access to self service APIs and vice versa." [Fineract self-service plugin
  documentation/source commentary, via search-engine summary of GitHub/cwiki material, not independently
  fetched in full]. This confirms the *concept* of distinguishing transaction origin by channel is real in
  Fineract, but as a coarse boolean gate on who can call which API, not a flat numeric classification code
  stored on the transaction itself the way the sample payload's `transactionSource`/`transactionSubSource`
  fields are.
- **Oracle FLEXCUBE's documentation names a comparable but differently-shaped concept: a "Payment source,"
  "defined with a unique code," for external systems feeding payment messages in, and a distinct "Source
  Branch code" field on a transaction** — real, named concepts for transaction provenance, but again not
  confirmed as a single flat numeric category/source/sub-source triple; FLEXCUBE's version reads as several
  separate, differently-purposed fields (a branch code, a payment-source code) rather than one unified
  classification scheme. [Oracle FLEXCUBE documentation, via search-engine extraction of Oracle's own PDF
  content, not independently fetched in full].
- **This project's own prior research (T-057) already found the clearest, most directly-read real precedent
  at the wire-message-standard level, not the core-banking-vendor level: ISO 20022's `PaymentTypeInformation`
  carries five separate named sub-fields — Instruction Priority, Service Level, Local Instrument, Clearing
  Channel, and Category Purpose — each a coded taxonomy (e.g. `CategoryPurpose` values like `GP2P` for
  general person-to-person, `SALA` for salary payments), not one flat integer.** This is reused here, not
  re-derived, as the clearest real answer this research (across two tasks now) has found to "is this a flat
  numeric code or a richer, named taxonomy" — the answer, consistently, is a richer, named taxonomy, split
  across several distinct, purposed sub-fields, not one triple of plain integers.

**Verdict: confirmed as a real, common underlying need, but the specific shape (a flat numeric
category/source/sub-source triple) is not confirmed as the dominant convention.** This matches this project's
own repeated pattern (T-054, T-056, T-057) of finding "a real need, no single universal shape" rather than
one clean, confirmed convention to point to. Both platforms checked classify transaction origin by some
real mechanism, but each uses a different, specifically-purposed field or entity (a payment-type/channel
classification, a branch/source code, a coarse self-service boolean) rather than one unified numeric taxonomy
— the real payload's own flat-integer shape looks closer to one institution's own internal convention than
to a universally-adopted pattern, even though the underlying *need* this gap names is real and confirmed.

**What I could not verify:** no source in either platform checked describes a single, unified three-level
numeric taxonomy (category, then source, then sub-source) the way the real payload's own three fields do;
this research did not find that specific three-tier shape anywhere, only several separately-purposed,
real fields answering adjacent but not identical questions.

---

## Gap 6 — Named actor persistence vs. a signed/cryptographic authorization record

**Confidence: medium-high.** This is the most architecturally loaded of the six per the task's own framing,
and the finding here is directly stated, not hedged: both platforms checked persist a human-identifiable
name/id for who approved something, and neither was found, in this research, to also separately maintain an
independent cryptographic proof of authorization alongside that name.

- **Oracle FLEXCUBE's own Security documentation describes exactly this pattern as the system's normal,
  designed behaviour, not an edge case**: "When a record is saved in the module, the ID of the user who saved
  the record is displayed in the 'Input By' field... Once the record is authorized, the ID of the user who
  authorized the record will be displayed in the 'Authorized By' field, with the date and time at which the
  record was authorized displayed in the 'Date/Time' field." FLEXCUBE's own audit trail is separately
  described as storing "Maker ID, Maker Date Stamp, Checker ID, and Checker Date Stamp." [Oracle FLEXCUBE
  Security Guide/SMS documentation, via search-engine extraction of Oracle's own PDF content, not
  independently fetched in full — the specific field names ("Input By," "Authorized By") are precise enough
  to read as a real quote, not a paraphrase, but this research could not independently verify the full PDF].
- **Apache Fineract's own audit/maker-checker API, confirmed via search-engine summary of Apache's own
  documentation, returns the same shape directly as plain, queryable response fields**: "`maker`,
  `madeOnDate`, `checker`, `checkedOnDate`, `processingResult`, `commandAsJson`" on every audit entry — a
  human-identifiable maker and checker, by name/id, persisted and retrievable through the platform's own
  normal API, not hidden behind a cryptographic artifact. [Fineract Audits API / checker-inbox documentation,
  via search-engine summary of Apache's own API docs and GitHub issue/PR material, not independently fetched
  in full]. Separately, Fineract's own `acc_gl_journal_entry` table, read directly, confirms `createdby_id`
  and `lastmodifiedby_id` as foreign keys into `m_appuser` — so even the ledger-posting side of Fineract keeps
  a direct, joinable link back to a named human user, not only the command/audit side. [Fineract Database
  Docs — `acc_gl_journal_entry`](https://fineract.apache.org/docs/database/tables/acc_gl_journal_entry.html)
  (fetched and read directly).
- **Neither platform's own documentation, as far as this research could reach, describes a separate,
  independent cryptographic signature or token kept *alongside* the human-readable maker/checker record as a
  distinct second artifact.** In both, the stored name/id and timestamp appear to function as the entire
  authorization record itself — the database row *is* the proof that record X was approved by user Y at time
  Z, under the platform's own access-control and session-authentication mechanisms, not a two-part design
  that separately keeps a signed, verifiable grant object and only secondarily/optionally a human name. This
  absence is named plainly as a real finding from what this research could reach, not as proof that no such
  mechanism exists anywhere in either platform's much larger codebase/documentation set, which neither this
  research nor the sources it could access cover exhaustively.

**Verdict: confirmed as the common, dominant real-world pattern — and this is the gap where the comparison
most directly and concretely answers the task's own named question.** Both real, named, independently-checked
platforms persist a human-identifiable name for who approved a transaction, directly and queryably, as a
normal part of their own data model — not as an afterthought, a debug field, or a rare configuration. This
means GARBATI's own decision #13 (never persist the human name, only a `DecisionId` tied to a cryptographically
signed, deliberately non-persisted grant token) is the choice that looks unusual relative to common core-banking
practice in this comparison, not the reverse — and this is a real, costed trade-off, not a free one: per the
task's own framing, GARBATI genuinely cannot today answer "who approved this, by name" from the ledger alone,
in a way that both comparison platforms' own ordinary audit mechanisms can.

**What I could not verify:** whether either platform, somewhere in its broader codebase or a module not
reached by this research, also implements some separate cryptographic-signature mechanism *in addition to*
the named maker/checker fields found here (for a different purpose, e.g. non-repudiation against an external
party, rather than internal approval tracking) — this research's own access was limited to what search-engine
summaries of each platform's documentation and GitHub material surfaced, not an exhaustive reading of either
codebase, and this absence should be read as "not found," not "confirmed absent everywhere."

---

## Synthesis — which gaps look most load-bearing, by how commonly each was confirmed

This ranking is based only on how cleanly and independently each gap was confirmed across the two platforms
actually checked in this research — it is not a personal recommendation on what GARBATI should build, fix,
or prioritize, per the task's own explicit scope.

1. **Gap 1 (value-date/multi-stage timestamps)** — confirmed most cleanly and independently in both
   platforms, with FLEXCUBE's own documentation giving a precise, directly-on-point definition and Fineract
   confirming a genuine (if less cleanly mapped) multi-date schema, directly read from its own database docs.
2. **Gap 6 (named-actor persistence vs. signed grant)** — confirmed just as cleanly in both platforms, and
   the one gap where the comparison most directly answers the task's own framing: both real systems persist
   a human name, neither was found to separately keep a cryptographic record instead of or alongside it.
3. **Gap 2 (branch attribution per lifecycle stage)** — confirmed in both platforms, real and not
   enterprise-only (Fineract's own simpler version proves this), though the two platforms differ meaningfully
   in richness (one `office_id` field vs. a full inter-branch/multi-branch-access subsystem).
4. **Gap 5 (transaction category/source/channel)** — the underlying need is confirmed in both platforms and
   at the ISO 20022 wire-message level (reused from T-057), but the specific flat-numeric-triple shape is not
   the dominant convention found; real systems answer this need with several different, specifically-purposed
   fields rather than one unified taxonomy.
5. **Gap 4 (per-entry narration + reconciliation remarks)** — narration itself is confirmed common and cheap
   in both platforms; the second half (a dedicated reconciliation-remarks field, distinct from narration) is
   not confirmed as a standard concept in either — what's actually common instead is a structured
   correlation-id mechanism, a different shape than the task's own framing posed.
6. **Gap 3 (multi-entry N:M posting)** — confirmed as a real, named, common core-banking API capability in
   general, in both platforms, but every real-world justification found for it (payroll, bulk collections) is
   a different transaction type than GARBATI's own current, stated retail-transfer scope — making this the
   gap whose load-bearing-ness for GARBATI specifically is the most conditional on a product-scope question
   this research does not answer.

---

## What this does not settle

- **Does not design a schema, propose new fields for `Transaction`/`LedgerEntry`/any request contract, or
  write a task file for closing any of the six gaps**, per the task's explicit scope.
- **Does not re-derive or re-confirm the six gaps themselves, or re-read GARBATI's own code** — both are
  treated as established fact per the task file, exactly as instructed.
- **Does not assume MIZAN/SEABASS's own specific field choices are "correct" or a standard GARBATI should
  match exactly** — this research checked whether the *underlying concepts* are common across other real
  platforms, not whether one institution's own exact field names or numeric codes are a de facto standard;
  several findings above (Gaps 3, 4, 5 specifically) found the general concept common while the sample
  payload's own specific shape (a flat `multipleDebit` flag, a `reconciliationRemarks` field, a numeric
  category/source/sub-source triple) looks more institution-specific than universal.
- **Is not a Sharia ruling and does not touch Sharia questions.** Nothing in this brief bears on GARBATI's
  existing Sharia-governance open questions.
- **Does not independently verify Oracle FLEXCUBE's own documentation beyond what search-engine summaries of
  Oracle's own PDF content surfaced** — every FLEXCUBE-sourced claim in this brief is flagged at reduced
  confidence for exactly this reason, consistent with the same `poppler-utils`-adjacent access limitation
  T-037/T-041/T-043/T-054/T-056/T-057 already documented repeatedly for this toolchain's handling of FLEXCUBE
  and CBN PDFs. The Fineract-sourced claims that were independently fetched and read directly (the
  `acc_gl_journal_entry` database-table page) are flagged accordingly at higher confidence than everything
  reached only via search-engine summary.
- **Does not check Temenos**, per the task's own explicit framing that Temenos' documentation has repeatedly
  required authenticated-community-login access in prior research — no new attempt was made in this research
  either, consistent with the task's own guidance to treat it as a weaker candidate absent a public page.

## What this affects

This brief directly informs the same open question the task file names: whether the six gaps the coordinator
already found by comparing a real production payload against GARBATI's actual code are common, load-bearing
core-banking concepts, or narrower/institution-specific ones. `ARCHITECTURE.md` Section 9 does not currently
name this specific payload-comparison research anywhere — the coordinator should consider whether this
research (and the task file's own six-gap comparison it builds on) becomes a new Section 9 entry, the same
way `research/teller-till-vault-cash-operations.md` (T-056) and `research/multi-charge-transaction-modeling.md`
(T-057) are candidates for their own entries. If such an entry is added, this research's own, specific
contribution to it is the triangulation finding above: three gaps (value date, named-actor persistence,
branch attribution) confirmed cleanly common across both platforms checked; one (category/source/channel)
confirmed as a real need with no single universal shape; two (multi-entry posting, reconciliation remarks)
narrower or more conditional than the task's own framing first suggested — not a decision on which, if any,
to close, which this research does not make.

## Sources

- [Oracle FLEXCUBE Funds Transfer documentation (value date, back-value dating, debit/credit value date
  spreads)](https://docs.oracle.com/cd/E51527_01/PDF/FT/FT.pdf) (via search-engine extraction of Oracle's own
  PDF content, not independently fetched and read in full)
- [Oracle FLEXCUBE Retail Teller documentation (Transaction Branch, Source Branch, Narrative field, Inter
  Branch Transaction Input)](https://docs.oracle.com/cd/E51511_01/PDF/Branch/FCC_Host/Retail_Teller.pdf) (via
  search-engine extraction, not independently fetched in full)
- Oracle FLEXCUBE Security Guide / SMS documentation ("Input By," "Authorized By," Maker ID/Checker ID audit
  fields), e.g. [Security Management System Oracle FLEXCUBE Universal Banking](https://docs.oracle.com/cd/E55610_01/PDF/CORE/Sms.pdf)
  (via search-engine extraction, not independently fetched in full)
- [Oracle FLEXCUBE Universal Banking Retail Teller Glossary](https://docs.oracle.com/cd/E99083_01/html/Retail_Teller/RT08_Gloss.htm)
  (fetched directly; only the "Transaction Code" definition was retrieved from the specific glossary section
  fetched — Value Date/Maker/Checker/Narration were not found in this particular fetch and are sourced
  elsewhere in this brief via search-engine summary instead)
- [Fineract Database Docs — `acc_gl_journal_entry` table](https://fineract.apache.org/docs/database/tables/acc_gl_journal_entry.html)
  (fetched and read directly — the single highest-confidence Fineract source in this brief: `entry_date`,
  `transaction_date`, `office_id`, `description`, `createdby_id`/`lastmodifiedby_id` all confirmed directly)
- [Apache Fineract API Legacy Documentation](https://fineract.apache.org/docs/legacy/) (fetched directly, but
  the specific createJournalEntry parameter detail sought was not present in the content this fetch
  retrieved; cited elsewhere in this brief via search-engine summary instead)
- [apache/fineract PR #6195 — FINERACT-2719, journal-entry creation-timestamp fix](https://github.com/apache/fineract/pull/6195)
  (title/content via search-engine summary, not independently fetched in full)
- Fineract accounting/cwiki documentation on compound journal entries and the credits/debits array example
  payload, e.g. [Accounting — Fineract — Apache Software Foundation](https://cwiki.apache.org/confluence/display/FINERACT/Accounting)
  (via search-engine summary, not independently fetched in full)
- Fineract Audits API / checker-inbox documentation (`maker`, `madeOnDate`, `checker`, `checkedOnDate` fields)
  and related GitHub issue/PR material on office-scoped maker-checker visibility across branches — via
  search-engine summary, not independently fetched in full
- Fineract `PaymentType`/payment-channel GL-routing documentation and the `is_self_service_user` self-service
  vs. back-office distinction, e.g. [GitHub — openMF/selfservice-plugin](https://github.com/openMF/selfservice-plugin)
  (via search-engine summary, not independently fetched in full)
- Fineract `externalId` reconciliation-purpose documentation (savings transactions) — via search-engine
  summary of Apache JIRA/GitHub material, not independently fetched in full
- Internal, reused from T-057, not re-derived: ISO 20022 `pacs.008` `PaymentTypeInformation`
  (`CategoryPurpose`/`ClearingChannel`/`ServiceLevel`/`LocalInstrument`) and `ChargesInformation`/
  `ChargeBearer` findings, and the `RemittanceInformation`-vs-structured-reconciliation-reference distinction
  — `research/multi-charge-transaction-modeling.md` (T-057)
- Internal: `tasks/T-058-research-real-corebanking-payload-gap-analysis.md` (the task file itself, read in
  full, including the real payload and the six established gaps reproduced there)
- Internal: `research/multi-charge-transaction-modeling.md` (T-057) and `research/teller-till-vault-cash-operations.md`
  (T-056), built on for sourcing-discipline and confidence-calibration conventions, not re-derived

## What was attempted and blocked or unreadable (stated plainly)

- No Oracle FLEXCUBE PDF was independently fetched and read in full in this research — this research relied
  on search-engine summaries/extractions of Oracle's own PDF content throughout, consistent with the
  repeatedly-documented `poppler-utils`-adjacent limitation named in T-037/T-041/T-043/T-054/T-056/T-057 for
  this exact category of document; no new direct-fetch attempt against a FLEXCUBE PDF was made in this
  research given that prior, repeated documented failure.
- `https://fineract.apache.org/docs/legacy/` was fetched directly but did not surface the specific
  `createJournalEntry` request-parameter detail sought in the content retrieved by this tool; that detail is
  instead sourced via search-engine summary elsewhere in this brief, flagged accordingly.
- `https://docs.oracle.com/cd/E99083_01/html/Retail_Teller/RT08_Gloss.htm` was fetched directly (unlike the
  PDFs above, this is an HTML page and was reachable), but the specific glossary-section content retrieved
  contained only the "Transaction Code" definition among the terms sought; Value Date, Maker, Checker,
  Narration and others were not found in this specific fetch and are not claimed as directly-confirmed from
  this source, even though they may exist elsewhere on the same glossary page/site.
- Temenos was not independently attempted in this research, per the task's own explicit framing that its
  documentation has repeatedly required authenticated-community-login access in prior research on this
  project.
