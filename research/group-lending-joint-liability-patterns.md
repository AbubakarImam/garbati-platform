# Group lending / joint-liability patterns, conventional and Islamic (T-066)

**Task:** T-066. **Status: research only.** No ruling, no design, no schema, no code, no task file for
a future build. This is one layer of input for a future financing-module design, the same supporting
role `research/sharia-rule-configuration-patterns.md` (T-050) already plays for that same future design
— it does not reopen or jump ahead of D-28's own gate (financing module build is gated on a qualified
Sharia scholar's sign-off, `TASKS.md`).

**What this builds on, and does not re-derive:** `research/fineract-core-functionality-gap-analysis.md`
(T-059), Area 2. That research already established, at high confidence: Fineract's real `Group`/`Center`
entities let several clients jointly hold one savings or loan account, with a `Center` as a scheduling
container above `Group`; GARBATI has zero structural equivalent anywhere; this gap matches GARBATI's
current, deliberate deposit-only scope (D-28) rather than an oversight; and group lending is historically
foundational to microfinance (Grameen Bank) but not the only live methodology today (individual lending
is a real, growing alternative, by Fineract's own account). None of that is re-verified here — it is
cited and built past.

---

## Plain-language summary (read this first)

Grameen Bank — the institution most people associate with "group lending" — does not, in its own
founder's words, actually use formal joint liability at all. Muhammad Yunus himself states plainly that
there is "no formal or legal... form of joint liability, i.e. group members are not responsible to pay
on behalf of a defaulting member" — group members carry only a *moral* responsibility, read directly from
Wikipedia's Solidarity lending page, which itself quotes Yunus. What actually happens when a Grameen
member struggles is social and voluntary: "the group usually comes forward to help," and the bank
separately runs a compulsory group fund (5% of each loan) and an emergency/insurance fund (a share of
accrued interest) that members can draw on for hardship — but these are support mechanisms, not an
automatic transfer of one member's unpaid balance onto another member's account. Many *other* real
microfinance institutions, by the same source's own account, implement the "solidarity group" model
differently and do run true joint liability — pursuing "any member of the solidarity group with the
capability to repay" a seriously delinquent peer's debt. **The honest finding is a real, named split
between two different things people call "joint liability," not one universal mechanism** — exactly the
distinction this task's own Q1 asked this research not to assume away.

On the Islamic side, real named institutions and real academic literature converge on **kafala** (mutual
guarantee) as the named concept Islamic group lending reaches for — Islami Bank Bangladesh's Rural
Development Scheme requires "personal guarantee from every member of one's group," with members "jointly
and severally liable," and PNM Mekaar Syariah (Indonesia) runs a named "tanggung renteng" (joint
responsibility) scheme reviewed specifically for Islamic-legal permissibility in an academic paper this
research could not read directly (blocked, see below). But kafala itself carries a real, live scholarly
tension that is directly relevant and must be flagged, not resolved here: classical jurisprudence
(Hanafi, Shafi'i, Maliki, Hanbali schools, per a converging secondary-source summary) treats kafala as a
**gratuitous (tabarru')** contract — guaranteeing someone else's debt for a fee is contentious precisely
because, if the guaranteed party then defaults, the guarantor-guaranteed relationship becomes an ordinary
debtor-creditor one, and a fee on top of that can look like riba. AAOIFI's own position (per a converging
secondary source) permits recovering only *actual administrative cost*, not a profit margin, for issuing
a guarantee. **This matters directly for any future GARBATI group-lending design that charges a "group
fund" contribution or insurance-like premium as part of a mutual-guarantee mechanism — whether that
contribution structure is itself Sharia-compliant is exactly the kind of question only a qualified scholar
can answer, and this research does not attempt to answer it.**

Structurally, Apache Fineract's real database schema (read directly, going one level deeper than T-059's
conceptual wiki page) confirms group-ness is a property layered onto a loan, not a separate contract
type: `m_loan` carries a `group_id`, a `client_id`, a `glim_id`, and a `loan_type_enum` with distinct
`individual`/`group`/`jlg` values side by side on the same table — and a Fineract loan *product*
definition is reused across these types via a `templateType` selector at application time, not a
separate product per loan type (medium confidence on that last point specifically, sourced via search
summary, not independently re-fetched). This is a real, structural data point in favor of this task's own
Q4 hypothesis that group lending is a liability/organizational layer separable from the underlying
contract type, not proof of it for Sharia contract types specifically, which this research did not find
any source addressing directly.

For GARBATI's own existing code: `IdentityApi`'s `Mandate` mechanism (confirmed directly from the real
OpenAPI contract, `contracts/Garbati.IdentityApi.openapi.json`) is structurally a **separate `Mandate` row
per `(customerId, ledgerAccountId)` pair, with a `requiredApprovals` count on the account** — several
mandates on *one* account, each a full-access grant once the threshold is met. That is authorization
(who may move money out of one shared pot), not liability (whose own separate balance is affected when
someone else fails to pay). Fineract's own `m_guarantor` table, read directly, confirms the same
distinction from the other side: it is built for an *external* guarantor's personal details (name, DOB,
address, phone), not for one group member's own loan record to reference another member's — Fineract's
actual intra-group joint-liability mechanism lives in `m_loan.group_id`/`glim_id` and `m_group_client`
membership, not in `m_guarantor`. GARBATI has no analogue to either the Fineract `m_loan.group_id`/`glim_id`
shape or to a per-member-balance-within-one-group-loan concept today — a genuinely new structural seam,
not a stretch of `Mandate`.

---

## Q1: How does joint liability actually work mechanically in a conventional (Grameen-style) group loan?

**Confidence: high** on the Grameen-specific finding (a direct Wikipedia quote of Yunus, cross-checked by
a converging secondary summary of the same institution's group-fund mechanics); **medium** on how
*other*, unnamed microfinance institutions implement true joint liability, since the clearest source
describing that side is itself a secondary synthesis, not a single named institution's own primary
account.

- **Grameen Bank itself does not use formal/legal joint liability — read directly, with a direct quote
  from the founder.** "[T]here is no formal or legal... form of joint liability, i.e. group members are
  not responsible to pay on behalf of a defaulting member" — Muhammad Yunus, quoted directly on
  [Wikipedia: Solidarity lending](https://en.wikipedia.org/wiki/Solidarity_lending) (fetched and read
  directly). Group members instead carry only "moral responsibility." Mechanically: "the group usually
  comes forward to help" a struggling member voluntarily, enforced by "subtle and at times not-so-subtle
  peer pressure" — described explicitly as social, not contractual or automatic.
- **Grameen does run a real, named compulsory group fund and an emergency/insurance fund — but these are
  support mechanisms the member or group can draw on, not an automatic re-billing of a defaulter's unpaid
  balance onto other members' own loan accounts.** "5 percent of loans are credited to a group fund, and a
  quarter of the active interest is credited and used in an emergency fund as an insurance against
  default... owing to death or disability or natural disaster." Loans from the group fund are available
  for emergencies and "must be repaid in one month." Crucially, the same source states plainly: "the group
  should not be made responsible to pay on behalf of a defaulting member" — confirming the fund is a
  hardship-support mechanism, not a joint-liability settlement mechanism. [GDRC — Grameen Bank, Bangladesh
  (Group Fund / Loan Repayment)](https://www.gdrc.org/icm/grameen-repay3.html) (via search-engine summary,
  not independently fetched and read in full — flagged at correspondingly reduced confidence relative to
  the directly-read Wikipedia quote above, though the two sources converge on the same "no automatic
  transfer of balance" conclusion).
- **A real, named split exists between Grameen's own model and "many other" institutions' use of the same
  "solidarity group" label — this research did not find this to be a universal single mechanism, matching
  the task's own instruction not to assume it is.** The same Wikipedia page states that many other
  microcredit organizations implement solidarity groups "essentially as a form of joint liability," and
  "will pursue... any member of the solidarity group with the capability to repay" a seriously delinquent
  peer's loan — i.e., a real, formal, collection-level joint liability, genuinely different from Grameen's
  own moral-responsibility framing. [Wikipedia: Solidarity lending](https://en.wikipedia.org/wiki/Solidarity_lending)
  (fetched and read directly). This research did not find one single named non-Grameen institution's own
  primary documentation stating its exact joint-liability mechanics (e.g., does a defaulting member's
  unpaid installment get posted as a debit against other members' own individual loan balances, or merely
  pursued as a collections action against them personally) — this remains genuinely unconfirmed at the
  mechanical, ledger-entry level for any specific non-Grameen institution.
- **Academic literature confirms joint liability (where used) functions primarily as a peer-monitoring and
  future-credit-access incentive, not solely (or even primarily) as a balance-transfer mechanism.** "If any
  member of a group defaults, all members are ineligible for Grameen Bank credit in the future" is the
  punishment mechanism most consistently described in the academic literature found — a credit-access
  penalty, not a balance-level transfer. [World Bank Policy Research Working Paper 6204 — "Grameen Bank
  Lending: Does Group Liability Matter?"](https://documents1.worldbank.org/curated/en/300451468200647730/pdf/wps6204.pdf)
  (title/content via search-engine summary; this research's own direct fetch of this specific PDF was not
  attempted after the pattern of repeated PDF-rendering failures elsewhere in this research, flagged
  honestly as a search-summary source, not independently verified against the primary text).

**What I could not verify:** the exact ledger-level mechanics for any specific non-Grameen institution that
does implement formal joint liability (i.e., whether a defaulting member's installment is actually debited
from co-members' own loan or savings balances, versus pursued only as a collections/credit-bureau action
against them personally while their own balance stays untouched) — every source found describes the
*incentive structure and collection pressure* clearly, but none gives a concrete, named institution's own
transaction-level description of what happens to another member's own balance, if anything.

---

## Q2: How do real Islamic/Sharia-compliant microfinance institutions structure group-based lending specifically?

**Confidence: medium** on the existence and naming of kafala as the real, named Islamic-finance mechanism
reached for; **low-medium** on any specific institution's precise mechanical implementation, since the one
most-directly-relevant primary source found (the PNM Mekaar Syariah Islamic-legal review) could not be
read (blocked, see below); **this research explicitly does not render any permissibility judgment and
flags every specific claim below as something only a qualified scholar can confirm**, matching the
discipline T-007/T-050/D-28 already established.

- **Kafala (mutual guarantee) is the real, named Islamic-finance concept the literature and real
  institutions converge on for group-based lending — not a single uncontested mechanism, but a named,
  recurring answer.** "Beneficiaries are formed into groups with mutual guarantee under the concept of
  kafala" is stated across multiple converging secondary summaries of academic literature (search-engine
  summary, not one single source independently fetched and read in full). [ResearchGate — "Introduction to
  Islamic Microfinance"](https://www.researchgate.net/publication/228201195_Introduction_to_Islamic_Microfinance)
  (via search-engine summary).
- **Takaful (mutual risk-pooling/insurance) is named as etymologically and conceptually derived from the
  same root as kafala** — "the word 'Takaful' is derived from 'kafala', which literally means 'to take
  care'" (via search-engine summary of multiple converging sources, not independently fetched in full).
  This research did not find a single real, named microfinance institution whose group-lending default
  mechanism is explicitly structured as a formal takaful scheme (a pooled, managed, actuarially-run fund)
  as opposed to an informal group fund or kafala-style personal guarantee — the takaful connection found is
  conceptual/etymological, not a confirmed real operational link for group *lending* specifically (as
  opposed to takaful's own, separate, real use as microinsurance).
- **A real, named institution states its group mechanism in exactly joint-and-several-liability language:**
  Islami Bank Bangladesh's Rural Development Scheme (RDS) requires investment "without collateral" that
  "functions on group guarantee mechanism, with personal guarantee from every member of one's group... and
  the members jointly and severally liable and responsible for repayment of the investment," with a weekly
  centre meeting all groups must attend. This is sourced via a search-engine summary describing IBBL's own
  published RDS material; this research's own attempt to fetch IBBL's website directly
  (`islamibankbd.com/microfinance`) failed with a connection error (see "blocked" section below), so this
  specific claim is flagged at reduced confidence — a real, named institution's own self-description,
  not independently re-confirmed by this research's own direct read.
- **A second real, named institution (PNM Mekaar Syariah, Indonesia) runs a named joint-responsibility
  scheme — "tanggung renteng" — reviewed specifically for Islamic-legal permissibility in an academic
  paper, but this research could not read that paper's own conclusion.** A search-engine summary states
  the scheme groups 10 members under "a joint liability (tanggung renteng) scheme," where "if one customer
  cannot pay installments, then other members in the group are responsible for paying the installments" —
  but the paper's own title (["An Islamic Legal Review of the Joint Liability System..."](https://e.journal.zabagsqupublish.com/index.php/zijis/article/download/20/49))
  signals it addresses exactly this task's own permissibility-adjacent question, and this research's direct
  fetch of that exact PDF rendered only as unreadable binary/stream content (blocked, see below) — **this
  research cannot state what that paper's own conclusion on permissibility is, and does not guess.** A
  qualified scholar would need to read that paper (or equivalent primary fiqh literature) directly before
  any claim about this specific mechanism's permissibility is treated as settled.
- **A genuine, directly relevant scholarly tension on kafala itself, not specific to group lending, that
  this research surfaces and does not resolve — exactly the kind of specific permissibility claim this
  task requires be flagged, not ruled on.** Classical jurisprudence across multiple schools (Hanafi,
  Shafi'i, Maliki, Hanbali, per a converging secondary-source summary) treats kafala as a **gratuitous
  (tabarru') contract** — charging a fee for providing a guarantee is contentious specifically because, if
  the guaranteed party then defaults, the guarantor's relationship to the guaranteed party becomes an
  ordinary debtor-creditor one, and a fee layered on top of that can constitute riba. AAOIFI's own stated
  position (per the same secondary summary) permits recovering only actual administrative cost of issuing a
  guarantee, not a profit margin. [ResearchGate — "A Few Methods in Charging Fees for Kafālah (Bank
  Guarantee)... among Islamic Banks in Malaysia"](https://www.researchgate.net/publication/302029128_A_Few_Methods_in_Charging_Fees_for_Kafalah_Bank_Guarantee_-_I_among_Islamic_Banks_in_Malaysia)
  (via search-engine summary, not independently fetched in full). **This research flags, and explicitly
  does not resolve, whether a compulsory "group fund" contribution (the kind Grameen itself runs, Q1) or
  any similar pooled-contribution mechanism inside a nominally Islamic group-lending scheme would itself
  need to be structured as a genuinely gratuitous kafala arrangement, rather than a fee-for-guarantee or
  insurance-premium-like contribution, to avoid this specific, named scholarly objection — this is squarely
  a qualified scholar's determination, not this research's.**
- **No clear convergence found on a single "the" Islamic-compliant group-lending mechanism — stated
  plainly rather than forced to one answer, per the task's own instruction.** This research found kafala
  named repeatedly as the relevant concept, and at least two real, named institutions (IBBL RDS, PNM Mekaar
  Syariah) running group-based schemes with joint-liability-flavored language — but no single source found
  states that one specific implementation (an unpaid, purely gratuitous personal guarantee vs. a pooled
  group fund vs. something else) is the settled, uncontested Sharia-compliant shape; the one source found
  that specifically interrogates permissibility at the mechanism level (the PNM Mekaar Syariah review) could
  not be read.

**What I could not verify:** the PNM Mekaar Syariah Islamic-legal review's own conclusion (blocked, PDF
unreadable); IBBL's own RDS material, independently, beyond a search-engine summary (connection error on
direct fetch); whether any named Islamic microfinance institution's group mechanism has been reviewed and
explicitly approved by a named Sharia supervisory board as distinct from being merely described in academic
literature as "Islamic"; whether a compulsory group-fund contribution inside an Islamic group-lending
scheme is itself considered Sharia-compliant by any named scholarly body — **a qualified Sharia scholar
must confirm any of these points before they inform a future design; nothing in this section is a ruling.**

---

## Q3: What does Fineract's real `Group`/`Center` data model actually look like, structurally?

**Confidence: high.** Fineract's own database-table documentation pages (`fineract.apache.org/docs/database/tables/`)
were fetched and read directly for every table named below, going one level deeper than T-059's own
conceptual wiki-page source, per this task's own instruction.

- **`m_group` is the one real table backing both `Group` and `Center` — they are not two separate tables.**
  Columns, read directly: `id`, `external_id`, `status_enum`, `activation_date`, `office_id`, `staff_id`,
  `parent_id` (a self-reference, enabling a `Center` row to be the `parent_id` of several `Group` rows —
  the structural mechanism for T-059's "Center is a scheduling container above Group" finding), `level_id`
  (references `m_group_level`, the field that actually distinguishes a Center-level row from a Group-level
  row), `display_name`, `hierarchy`, `closure_reason_cv_id`, `closedon_date`, `activatedon_userid`,
  `submittedon_date`, `submittedon_userid`, `closedon_userid`, `account_no`. [Fineract Platform
  Documentation — `m_group`](https://fineract.apache.org/docs/database/tables/m_group.html) (fetched and
  read directly).
- **`m_group_client` is the real join table membership lives in** — a group's member clients are rows in
  this table, not a column on `m_client` or `m_group` itself, confirmed by its presence in the same
  directly-fetched table index alongside `m_group_level` (hierarchy) and `m_group_roles` (role assignments
  within a group — e.g., a designated group leader, distinct from ordinary membership).
- **`m_loan` carries real, distinct columns for group-related loan structures, read directly:** `group_id`
  (foreign key to `m_group` — a loan can belong to a group as a whole), `client_id` (an individual loan, or
  one leg of a JLG loan), `glim_id` (foreign key to a `glim_accounts`-type table — "Group Loans Individual
  Monitoring," confirmed via a converging GitHub pull-request title, "Group loan Individual monitoring and
  group saving individual monitoring," found by search, not independently re-fetched — flagged at medium
  confidence for the exact expansion of this specific acronym, though the existence of the `glim_id` column
  itself was confirmed by this research's own direct fetch of the table page), and `loan_type_enum`, a
  genuinely distinct column whose real values (confirmed directly in Fineract's own legacy API documentation,
  which returns `"loanType": { "id": 1, "code": "loanType.individual", "value": "Individual" }` as a real
  response shape) include at minimum `individual`, `group`, and `jlg` as distinct values — a search-engine
  summary (not independently re-verified in full) describes `jlg` specifically as "joint liability group
  loan given to members in a group on individual basis... given to one or more members in a group," i.e. a
  genuinely distinct third shape from "one loan owned by the group as a whole" (`group`). [Fineract Platform
  Documentation — `m_loan`](https://fineract.apache.org/docs/database/tables/m_loan.html) (fetched and read
  directly for the column list and foreign keys; the exact prose definition of each `loan_type_enum` value
  is via search-engine summary, flagged accordingly).
- **`m_guarantor` — read directly — is built for an *external* personal guarantor, not for one group
  member to reference another member's loan.** Its columns are `id`, `loan_id`, `client_reln_cv_id`,
  `type_enum`, `entity_id`, `firstname`, `lastname`, `dob`, address fields, phone fields, `comment`,
  `is_active` — a shape clearly designed to capture a named individual's personal details as a loan-level
  guarantee, not a reference to another `m_group_client` row. [Fineract Platform Documentation —
  `m_guarantor`](https://fineract.apache.org/docs/database/tables/m_guarantor.html) (fetched and read
  directly). **This is a genuinely useful, specific finding beyond T-059's own conceptual-level description:
  Fineract's actual intra-group joint-liability mechanism is not built on a per-member-guarantees-another-member
  row at all — it lives structurally in `m_loan.group_id`/`glim_id` plus `m_group_client` membership, i.e.
  liability is expressed by which loan a group's members are collectively attached to, not by an explicit
  guarantee relationship between two named members.**
- **A loan product appears to be reused across loan types via a selector at application time, not a
  separate product definition per type — medium confidence, via search-engine summary, not independently
  re-fetched and read in full.** "The `templateType` value (individual, group, jlg, or jlgbulk) decides
  the required template data for creating a new loan application" is described as the real mechanism, which
  — if accurate — means a single Fineract loan product configuration (interest rate, charges, repayment
  schedule shape) is not inherently tied to being an individual, group, or JLG loan; group-ness is a
  property of the loan *application*/*account*, not the *product*. This research flags this at reduced
  confidence relative to the directly-fetched table-column findings above, since the specific claim comes
  from a search-engine summary of developer discussion, not an independently fetched and read product-API
  schema page.

**What I could not verify:** the exact full column list of `m_group_level`, `m_group_roles`, `m_group_client`,
or `glim_accounts` tables themselves (this research fetched and read `m_group`, `m_loan`, and `m_guarantor`
directly, but did not separately fetch every adjacent table's own page — named plainly as a scope limit,
not a hidden gap); whether a `Center`-level row (as opposed to a `Group`-level row) can itself ever be the
direct `group_id` on an `m_loan` row, or whether only `Group`-level rows can hold loans with `Center` rows
staying purely organizational — this research's direct reads did not resolve that specific sub-question.

---

## Q4: Is group lending itself separable from any specific loan contract type, or does it depend on which contract type is used?

**Confidence: medium.** The structural Fineract evidence (Q3) supports separability at the level of
Fineract's own generic loan-typing mechanism; this research found no source — conventional or Islamic —
that addresses this question specifically for named Sharia contract types (Murabaha, Musharaka, Mudarabah,
Qard Hasan, etc.), which is the actual question a future GARBATI financing-module design would need
answered, so this is flagged honestly as a gap, not papered over with an inference stretched further than
the evidence supports.

- **Fineract's own structural answer, read directly (Q3): group-ness is a property on the loan
  account/application (`loan_type_enum`, `group_id`, `glim_id`), not a property requiring a specific loan
  product or contract shape.** A `templateType` selector reportedly lets the same product be instantiated
  as individual, group, or JLG (medium confidence, search-summary-sourced, per Q3). If accurate, this is a
  real, concrete precedent that at least Fineract's own generic (non-Sharia-specific) joint-liability
  mechanism is contract-type-agnostic by design — the group/liability layer sits above the product/contract
  layer, not inside it.
- **This research did not find a single source — academic, institutional, or regulatory — that directly
  addresses whether a Qard Hasan group loan and, say, a Murabaha group loan would need meaningfully
  different joint-liability mechanics specifically because of their different underlying contract
  structures.** This is a real, named gap in what this research could confirm, not a question this research
  answers by inference from the Fineract finding above — Fineract's own generic mechanism answers "can one
  generic loan-tracking system support group-ness across loan types," which is a different and narrower
  question than "does the specific legal/Sharia structure of a Murabaha (a sale-based contract, debt arising
  from a deferred-price sale) versus a Qard Hasan (a benevolent, interest-free loan) change what a
  mutual-guarantee mechanism needs to look like underneath it."
- **A plausible, but unconfirmed, reasoning path, named here as reasoning rather than a sourced fact:**
  since kafala (Q2) is itself described in the literature as a generic guarantee contract — a third party
  (or, in a group context, a set of peers) guaranteeing a debtor's obligation — and since the *obligation
  being guaranteed* could in principle arise from any underlying contract (a Murabaha's deferred sale price,
  a Qard Hasan's repayment obligation, a Musharaka partner's capital-return obligation), there is no
  structural reason apparent from the sources found why the *guarantee* mechanism itself would need to
  differ by contract type — the thing being guaranteed differs, not necessarily the shape of the guarantee.
  **This is this research's own inference, explicitly flagged as such, not a conclusion any source states
  directly — and it does not address the separate kafala-fee/tabarru' tension named in Q2, which could
  plausibly interact differently with different contract types' own profit/fee structures (e.g., a
  Murabaha's markup versus a Qard Hasan's zero-return structure) in ways a qualified scholar, not this
  research, would need to assess.**

**What I could not verify:** any direct source addressing contract-type-specific joint-liability mechanics
for named Islamic contract types; whether any real Islamic microfinance institution runs the *same* named
group/kafala mechanism across more than one underlying contract type in practice (which would be the
strongest possible real-world confirmation of separability) — no such institution-level comparison was
found in this research.

---

## Q5: Which of GARBATI's existing service boundaries would a future group-lending design touch, and how?

**Confidence: high** on what GARBATI's real code/contracts currently are (read directly); this section
names tension points only, per the task's own scope — it does not design anything.

- **`IdentityApi`'s `Mandate` is confirmed, directly from the real OpenAPI contract
  (`contracts/Garbati.IdentityApi.openapi.json`, read directly in this research), to be an *authorization*
  concept, not a *liability* concept — the same distinction T-059 already flagged, now independently
  re-confirmed against the actual contract rather than repeated from that prior finding.** A `Mandate` is
  created per `(customerId, ledgerAccountId)` pair (`AttachMandateRequest.ledgerAccountId` plus the
  customer path parameter; `MandateResponse.customerId`/`ledgerAccountId`), and `ChangeApprovalPolicyRequest.requiredApprovals`
  is a single integer threshold set *on the account*, not on any specific group of customers. This is
  structurally "several people, each independently entitled to act, on one shared pot of money, until
  enough of them agree" — i.e. exactly the M-of-N approval shape `research/joint-account-approval-patterns.md`
  (T-014) already researched. **It has no field, concept, or mechanism anywhere for "this customer's own,
  separate loan balance changes because a different customer, on a different account, failed to pay."**
  A `Mandate` and a group-lending liability relationship are confirmed, directly against the real contract,
  to be genuinely different shapes — not a case where one could be relabeled as the other.
- **`LedgerApi`'s chain/leg model (decision #45, `ARCHITECTURE.md` Section 12, read directly) has a partial,
  bounded fit, and a real gap, for "N debtors share responsibility for one balance."** The chain mechanism
  already does, exactly as built today: post several tagged legs (`Principal`/`Fee`/`Tax`) atomically, one
  database transaction, up to 10 legs, each leg a normal two-entry (one debit, one credit) transaction. If
  a future group-liability design needed to express "post this one group loan's disbursement principal, and
  separately record which member's share of it this represents," the chain's "ordered list of legs posted
  atomically" shape is not a bad starting fit for the *posting* side — ARCHITECTURE.md Section 12.3's own
  `purpose` enum is explicitly closed to `Principal`/`Fee`/`Tax` today (D-7), so any group-liability-specific
  leg purpose (e.g., something representing one member's guaranteed share) would need a new, explicitly
  decided purpose code, the same bounded-extension cost Section 9's own "Leg purpose codes" entry already
  names for any new value. **What the chain model has no analogue for at all: a single *loan* (as opposed
  to a single *transaction*) that is jointly owned or jointly liable across several distinct customer
  accounts over its *entire multi-installment lifetime*** — Fineract's own `m_loan.group_id`/`glim_id`
  (Q3) is a property of a loan *account* that persists across many separate transactions over time (every
  repayment, every installment), not a property of one atomic chain of legs posted once. GARBATI has no
  concept today of a loan account at all (no financing module exists), so this is a genuinely new structural
  layer a future design would need — not a stretch of the chain model, which operates at the
  single-transaction level, not the loan-account level.
- **Neither `IdentityApi` nor `LedgerApi`, read directly, has any concept of a `Group` or `Center` entity
  itself** — no table, no field, nothing that groups several `customerId`s or several `ledgerAccountId`s
  into one named collective entity with its own identity, membership list, or meeting/scheduling concept
  (the `Center` half of Fineract's model, Q3). This confirms, independently, T-059's own "zero structural
  equivalent anywhere" finding, now checked specifically against the `Mandate` and chain mechanisms this
  task asked about by name, not merely repeated.
- **The honest, non-decided scoping conclusion:** a future group-lending design would need, at minimum, (1)
  a new `Group`/`Center`-equivalent organizational entity that does not exist in any service today, (2) a
  new liability relationship distinct from `Mandate`'s existing authorization relationship — `Mandate`
  cannot be stretched to express it, confirmed directly against the real contract, not inferred, and (3)
  a loan-account-level concept in whatever future financing service is built, since `LedgerApi`'s
  transaction-level chain model does not operate at the right lifecycle scope for a multi-installment group
  loan. This research does not propose where any of these three should live (a new service, inside a future
  financing service, inside `IdentityApi`, or elsewhere) — that is squarely a design question for whenever
  the financing module itself is scoped, consistent with this task's own explicit exclusion of schema/design
  work.

**What I could not verify:** whether `OrchestratorApi`'s own existing sequencing mechanism (the account-opening
and transfer flows, `ARCHITECTURE.md` Section 4) would need new steps for a group-loan disbursement — this
research did not read `OrchestratorApi`'s own code/contract in this pass, since the task's own Q5 named
only `IdentityApi`'s `Mandate` and `LedgerApi`'s chain/leg model explicitly; this is named as an honest scope
limit of this specific research pass, not a claim that `OrchestratorApi` has no relevant tension point.

---

## What this does not settle

- **No ruling on whether any specific group-lending or kafala-based mechanism is Sharia-compliant.** Every
  specific claim in Q2 is flagged as something a qualified scholar must confirm, not something this
  research determines — including, specifically, whether a compulsory group-fund contribution or any
  fee-bearing guarantee structure inside a nominally Islamic group-lending scheme is permissible, and
  whether PNM Mekaar Syariah's or IBBL RDS's specific real-world mechanisms have been reviewed and approved
  by a named Sharia supervisory board (not independently confirmed either way in this research).
- **No schema, entity design, or API shape for a `Group`/`Center` concept in GARBATI.** Q5 names tension
  points only (where `Mandate` and the chain model do and do not fit); it does not propose a table, field,
  or endpoint.
- **No task file for a future build or design task is written here**, per this task's explicit scope.
- **Does not reopen or re-verify `research/fineract-core-functionality-gap-analysis.md`'s (T-059) own
  Area 2 findings** — they are cited and built past, not re-derived.
- **Does not resolve the mechanical, ledger-level question of what happens to a co-member's own balance in
  any specific non-Grameen institution's formal joint-liability scheme** (Q1's one named open gap) — every
  source found describes the incentive/collection-pressure side clearly, none gives a concrete,
  institution-named, transaction-level account.
- **Does not resolve whether group lending's separability from contract type (Q4) holds specifically for
  named Islamic contract types** — the Fineract evidence found answers a narrower, generic-loan-typing
  question, not the Sharia-contract-type-specific one a future financing-module design would actually need
  answered; this is named as a real, unresolved gap, not inferred past what the evidence supports.

## What this affects

This extends `ARCHITECTURE.md` Section 9's existing **"Group/joint-liability lending (Groups and Centers)"**
entry (added by T-059, 1 Nov 2026) rather than requiring a new entry — that entry already names the gap as
real and load-bearing specifically once GARBATI's financing module is scoped; this research adds, to that
same entry, three concrete, new findings a future financing-module design should have in hand at that
point: (1) "joint liability" is not one universal mechanism even in conventional group lending — Grameen's
own real model is explicitly *not* formally joint-liable, so a future design should not assume Grameen's
name implies Grameen's mechanics; (2) Islamic group lending reaches for a named, real concept (kafala,
sometimes connected to takaful) but that concept itself carries a live, unresolved scholarly tension around
fee-bearing guarantees that would need a qualified scholar's input specifically, separate from and in
addition to D-28's existing contract-type-level sign-off gate; (3) Fineract's own real schema (`m_loan`'s
`group_id`/`glim_id`/`loan_type_enum`, read directly) suggests group-ness is structurally separable from
loan-product definition in at least one real system, and GARBATI's own `Mandate` mechanism is confirmed,
directly against its real contract, to be unable to express group liability — any future design starts
from "needs a new entity and a new relationship type," not "can extend `Mandate`." None of this changes
D-28's own gate: a qualified Sharia scholar's sign-off remains the precondition for building any part of
the financing module, group-lending-related or otherwise.

## Sources

- [Wikipedia — Solidarity lending](https://en.wikipedia.org/wiki/Solidarity_lending) (fetched and read
  directly, including the direct Yunus quote on Grameen's non-formal joint liability)
- [GDRC — Grameen Bank, Bangladesh (group fund / loan repayment mechanics)](https://www.gdrc.org/icm/grameen-repay3.html)
  (via search-engine summary, not independently fetched in full)
- [World Bank Policy Research Working Paper 6204 — "Grameen Bank Lending: Does Group Liability Matter?"](https://documents1.worldbank.org/curated/en/300451468200647730/pdf/wps6204.pdf)
  (title/content via search-engine summary; not independently fetched given the repeated PDF-rendering
  failure pattern found elsewhere in this research)
- [ResearchGate — "Introduction to Islamic Microfinance"](https://www.researchgate.net/publication/228201195_Introduction_to_Islamic_Microfinance)
  (kafala-in-group-lending claim via search-engine summary)
- Islami Bank Bangladesh Rural Development Scheme (RDS) group-guarantee/joint-and-several-liability
  description — via search-engine summary of IBBL's own published material; this research's own direct
  fetch of `islamibankbd.com/microfinance` failed (connection error, see below)
- "An Islamic Legal Review of the Joint Liability System [tanggung renteng, PNM Mekaar Syariah]" —
  [e.journal.zabagsqupublish.com](https://e.journal.zabagsqupublish.com/index.php/zijis/article/download/20/49)
  (title and existence confirmed via search; this research's own direct fetch rendered only unreadable
  binary PDF content — see "blocked" section; its own conclusion on permissibility could not be read)
- [ResearchGate — "A Few Methods in Charging Fees for Kafālah (Bank Guarantee) among Islamic Banks in
  Malaysia"](https://www.researchgate.net/publication/302029128_A_Few_Methods_in_Charging_Fees_for_Kafalah_Bank_Guarantee_-_I_among_Islamic_Banks_in_Malaysia)
  (kafala-as-tabarru'/fee-permissibility tension, via search-engine summary)
- [Apache Fineract — `m_group` table documentation](https://fineract.apache.org/docs/database/tables/m_group.html)
  (fetched and read directly)
- [Apache Fineract — `m_loan` table documentation](https://fineract.apache.org/docs/database/tables/m_loan.html)
  (fetched and read directly for columns/foreign keys; `loan_type_enum` value prose via search-engine
  summary)
- [Apache Fineract — `m_guarantor` table documentation](https://fineract.apache.org/docs/database/tables/m_guarantor.html)
  (fetched and read directly)
- [Apache Fineract API Legacy Documentation](https://fineract.apache.org/docs/legacy/index.html) (fetched
  directly; confirms the real `loanType` response shape)
- GLIM ("Group Loans Individual Monitoring") acronym expansion and `templateType` product-reuse claim —
  via search-engine summary of Apache Fineract GitHub/mailing-list developer discussion, not independently
  fetched and read in full
- Internal: `research/fineract-core-functionality-gap-analysis.md` (T-059), Area 2 — read in full, built
  on, not re-derived
- Internal: `contracts/Garbati.IdentityApi.openapi.json` (read directly — `Mandate`/`AttachMandateRequest`/
  `ChangeApprovalPolicyRequest` schemas)
- Internal: `ARCHITECTURE.md` Sections 2, 9, 12 (read directly — service map, Section 9 open questions
  including the existing Group/Center entry, the chain/leg model)
- Internal: `research/joint-account-approval-patterns.md` (T-014) and `research/reversal-authority-patterns.md`
  (T-019) — read in full, cross-referenced for the authorization-vs-liability distinction and GARBATI's
  own existing `Mandate` M-of-N shape
- Internal: `tasks/T-066-research-group-lending-joint-liability-patterns.md` (the task file itself, read in
  full)

## What was attempted and blocked or unreadable (stated plainly)

- `https://e.journal.zabagsqupublish.com/index.php/zijis/article/download/20/49` (the PNM Mekaar Syariah
  Islamic-legal joint-liability review) — fetched as raw bytes; rendered only as garbled, unquotable PDF
  binary/stream content, the same `poppler-utils`-adjacent limitation this project's prior briefs have
  repeatedly documented for other PDFs. This is the single most directly relevant source this research
  identified for Q2 and could not read — its own conclusion on permissibility remains unknown to this
  research.
- `https://www.cgap.org/sites/default/files/CGAP-Focus-Note-Islamic-Microfinance-An-Emerging-Market-Niche-Aug-2008.pdf` —
  fetched as raw bytes; rendered only as garbled PDF binary/stream content, same failure mode.
- `https://www.findevgateway.org/sites/default/files/publications/files/mfg-en-paper-islamic-microfinance-theory-policy-and-practice-feb-2008.pdf` —
  fetched as raw bytes; rendered only as garbled PDF binary/stream content, same failure mode.
- `https://www.islamibankbd.com/microfinance` — direct fetch failed with `connect ECONNREFUSED`, not a
  content-rendering problem; the RDS group-guarantee claim in this brief rests on a search-engine summary
  of this site's own content, not an independent direct read, flagged accordingly in Q2.
- Fineract's `m_group_level`, `m_group_roles`, `m_group_client`, and `glim_accounts`-equivalent table pages
  were not individually fetched in this pass (only `m_group`, `m_loan`, and `m_guarantor` were) — named as
  a scope limit, not a failure, since the task's own Q3 was answered sufficiently by the three tables
  actually read.
