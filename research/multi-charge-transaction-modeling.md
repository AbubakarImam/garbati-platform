# How real banking/fintech platforms model stacked, compounding transaction charges, and what this means for GARBATI (T-057)

**Task:** T-057. **Status:** Research only — no code, no schema, no feature design, no timeline. This is
forward-looking research informing a possible future task, the same way `research/teller-till-vault-cash-operations.md`
(T-056) informed a possible future design before anything was built. `ProductApi`'s `FeeRule` remains
deliberately unseeded pending a qualified Islamic finance scholar's sign-off regardless of anything found
here — nothing in this brief changes that gate.

**What this builds on, and does not re-derive:** the task file itself already confirmed, by reading the real
code before this task was written, that `LedgerApi`'s `LegPurpose` enum (decision #45) is a hard-closed list
of exactly `Principal`/`Fee`/`Tax`, and that `Garbati.ProductApi`'s `FeeRule`/`FeeQuoteService` compute one
flat-or-percentage charge per `(ProductCode, OperationType)` call with no base-rule reference, no currency
branching, and no threshold gate. This brief does not re-verify those facts; it re-reads the actual
`FeeRule.cs` and `FeeQuoteService.cs` source directly only for the structural comparison in Q6, to describe
their exact current shape precisely rather than from memory, and treats `research/teller-till-vault-cash-operations.md`
(T-056) as the sourcing-discipline and confidence-calibration standard to match, not something to re-derive.

---

## Plain-language summary (read this first)

A single real Nigerian bank transfer today can carry two genuinely different kinds of extra charge, and
real, current, directly-checkable sources confirm both. First, **VAT really is calculated on the fee, not on
the money being sent** — a real Nigerian digital bank's own published customer explainer, read directly,
gives the worked example of sending ₦50,000 with a ₦50 transfer fee: "the Federal Government charges 7.5%
VAT on the transfer fee" (₦3.75), and the recipient still gets the full ₦50,000 — VAT is a tax on a fee, not
on the principal, confirmed in a real institution's own words, not assumed. A real Nigerian commercial bank's
own published tariff guide shows the identical pattern for a completely different charge type (a stopped
cheque, "₦500 + ₦37.50 VAT," and cheque-book issuance, "₦750 + ₦56.25 VAT") — two independent real
institutions, two different charge types, the same tax-on-fee shape, both read directly. Second, **stamp
duty is a real, separate, flat government levy with its own statutory basis** — the Finance Act 2020 added
section 89A to the Stamp Duties Act, creating the Electronic Money Transfer Levy (EMTL): a flat ₦50 on
electronic transfers of ₦10,000 or more, collected by the receiving bank and remitted to the Federal Inland
Revenue Service (FIRS), split 15%/85% between the federal and state governments. Multiple 2026 secondary
sources converge on a real, recent change: Nigeria's 2025 tax reform (the Nigeria Tax Act) is reported to
shift who pays this levy, from the receiver (the original 2020/2022 design) to the sender, effective 1
January 2026 — though this specific change was not confirmed by reading the Tax Act's own primary text, only
by several converging news sources. **A genuine, honestly-unresolved tension surfaced in this research and
is stated plainly rather than papered over:** one real professional-services summary of the 2022 EMTL
Regulations (KPMG Nigeria, read directly) states the levy also applies to *foreign-currency* transactions,
"charged at the exchange rates determined by the Central Bank of Nigeria" — which complicates this task's own
framing of stamp duty as an LCY-specific concept. This research could not read the Regulations' own primary
text to resolve that tension and says so rather than picking a side.

**On the CBN's own primary documents specifically — this research hit the identical access wall T-037, T-041,
T-043, T-054 and T-056 already documented, and says so plainly rather than guessing a figure from memory.**
Every `cbn.gov.ng`-hosted PDF attempted in this research (the 2020 and 2017 "Guide to Charges" circulars, the
2006 MFB Regulatory Guidelines) was fetched as raw bytes but rendered only as garbled, unreadable binary
content by this toolchain — the same `poppler-utils`-adjacent limitation named in every one of those prior
briefs. The CBN's own April 2026 exposure draft of a revised "Guide to Charges" could not even be located at
a working URL in this research; what is cited below about it comes from legal-commentary articles that
themselves state plainly it is a **draft, not yet final law** as of this research.

**On real core-banking/fintech modeling patterns:** no real system found in this research represents
"tax-on-a-charge" as a generic rule that references another rule's row by id. The one real, directly-relevant
mechanism found — Apache Fineract's `TaxGroup` attached to a `Charge` — bundles the tax calculation *into the
charge's own definition* (a charge optionally carries a tax group; the charge's total amount includes the
tax, computed together, not as an independently configured second rule pointing back at the first). ISO
20022's own payment messages (`pacs.008`) itemize charges as a repeatable array attached to the one
transaction, each with a bearer code (`DEBT`/`CRED`/`SHAR`/`SLEV`) — structurally closer to GARBATI's own
"charge is just another tagged ledger leg" instinct than to a separate "charge schedule" entity. On the final,
most load-bearing question — whether GARBATI's `LegPurpose`/`FeeRule` mechanism is structurally suited to
extend to cover this — the honest answer, read directly from `FeeRule.cs`/`FeeQuoteService.cs`, is **partially**:
the leg/chain side (post many tagged legs atomically) already does most of what real systems do; the
rule-computation side (one flat-or-percentage number per product/operation call, with no currency field, no
base reference, no threshold) does not yet have an equivalent of either real pattern found.

---

## Q1: How real platforms represent a transaction's "charge breakdown" as a first-class concept

**Confidence: medium.** The clearest, most directly-read evidence (ISO 20022's own charge fields) answers
this cleanly; the core-banking-vendor side rests more on search-engine summaries of vendor material than
independently fetched full documentation, named plainly below.

- **At the wire-message/settlement layer, a real, live global standard answers this directly: charges are an
  itemized, repeatable array attached to the one payment message itself, not a separate "charge schedule"
  object queried elsewhere.** ISO 20022's `ChargesInformation` element in `pacs.008` (the FI-to-FI customer
  credit transfer message) "is a repetitive field... it may be added at each payment leg," paired with a
  mandatory `ChargeBearer` code (`DEBT`/`CRED`/`SHAR`/`SLEV`) that governs whether `ChargesInformation` must,
  may, or need not be present. [ISO20022Payments.com — Charges](https://www.iso20022payments.com/cbpr/charges/)
  (fetched and read directly).
- **The statement-reporting side of the same standard (`camt.053`) carries its own charge *type* taxonomy as a
  real field value, not a free-text description** — bank-transaction-code guides for `camt.053` list distinct
  codes including `CHRG` ("Charges — Generic") and `TAXE` ("Taxes — Generic") as separate values. [Nordea —
  Bank transaction codes for camt.053.001.02](https://www.nordea.com/en/doc/nordea-bank-transaction-codes-camt.053.001.02.pdf)
  (title/codes via search-engine summary, not independently fetched in full — flagged at correspondingly
  lower confidence than the directly-read `ChargesInformation`/`ChargeBearer` finding above). This matters for
  the task's own Q1 framing: a real standard already treats "this is a tax" vs "this is a generic charge" as
  a distinct, first-class code value on the posting itself — structurally the same shape as GARBATI's own
  `LegPurpose` tag on a leg, not a separate lookup table.
- **At the product/fee-configuration layer, a different, real open-source core-banking system (Apache
  Fineract, GARBATI's own Apache 2.0 license precedent per `TASKS.md` D-21) answers the same question
  differently: a `Charge` is its own first-class, named entity type**, attachable to a loan, savings account,
  client, or product, separate from a generic ledger posting, with its own calculation-type field. [Fineract
  Platform Documentation](https://fineract.apache.org/docs/stable/) (fetched directly, but this specific page's
  own content on the `Charge` entity's exact field list was thin; the calculation-type detail below is
  corroborated by multiple Fineract Jira/mailing-list threads found by search, not one single page
  independently fetched in full, and is flagged at medium confidence accordingly). Fineract's charge
  calculation types found across these sources include percentage of amount, percentage of amount plus
  interest, percentage of interest, and percentage of amount disbursed — i.e. always a percentage of some
  *loan-level* quantity, never, in anything found in this research, a percentage of another charge's own
  computed result (relevant directly to Q2 below).
- **The honest synthesis:** real practice answers the task's own "is this its own object, or just a tagged
  ledger posting" question differently depending on which layer you look at. The settlement-message layer
  (ISO 20022) and the statement-reporting layer both look like GARBATI's existing instinct — a tagged posting,
  not a separate schedule object. The product/fee-configuration layer (Fineract) looks like a separate,
  named `Charge` entity distinct from the posting it eventually produces — closer to what `FeeRule` was
  clearly intended to be, but, as Q6 details, not currently shaped to express the relationships this task asks
  about.

**What I could not verify:** no commercial core-banking vendor's (Temenos, Finacle, Oracle FLEXCUBE) own
"charge schedule" or "fee engine" documentation was independently fetched and read in full in this research —
every attempt either required an authenticated developer-community login (Temenos Basecamp, confirmed
by a search result referencing a community thread about configuring "two separate taxes... calculated on the
same ta[x] charge property," title/gist only, not independently read) or returned an unreadable/garbled PDF
when fetched directly (the pattern described throughout this brief). This is named as a real gap, not papered
over with a remembered claim about any of these three specific vendors.

---

## Q2: How tax-on-a-fee (a charge calculated as a percentage of another charge) is actually modeled

**Confidence: medium-high** on the one real, directly-read mechanism found (Fineract's `TaxGroup`-on-`Charge`);
**medium** on how it behaves if the base charge is waived, which is reasoned from that mechanism's own
behaviour, not independently confirmed by a source that states it explicitly; **high** on the Nigerian
real-world example (VAT-of-fee), since two independent, real, named institutions' own published material
converge on it.

- **The one real, concrete mechanism found does not match either option the task's own framing poses
  cleanly — it is closer to a third shape.** The task asks whether tax-on-a-fee is modeled as "a rule with an
  explicit base reference (charge X is a percentage of charge Y)" or "computed and frozen... two independent,
  pre-calculated figures." Apache Fineract's real, shipped mechanism is neither exactly: a `TaxGroup` is
  attached directly to a specific `Charge`'s own definition (not a second, independent charge that happens to
  point back at the first) — "when a tax group is attached to a loan charge, the charge amount includes the
  tax amount (for example, if tax is 10% and the loan charge amount is 100, the total becomes 110)," confirmed
  by a real Fineract GitHub pull request fixing a bug in exactly this mechanism's accounting arithmetic.
  [apache/fineract PR #6692 — FINERACT-2969, "Subtract a taxed charge's tax once across its rows"](https://github.com/apache/fineract/pull/6692)
  (fetched and read directly). The bug itself — a taxed charge spanning multiple instalment rows was having
  its tax subtracted once *per row* instead of once total — is itself real, useful evidence that Fineract
  treats the charge-plus-its-tax as one bundled unit computed together, not two independently tracked numbers.
- **What happens if the base charge is waived or reduced — reasoned, not independently confirmed.** Because
  Fineract's tax is a property of the charge's own amount (computed together, per the mechanism above, and
  prorated/subtracted as one unit across the charge's own rows), the reasoned inference is that reducing the
  base charge to zero would reduce its attached tax to zero in the same computation, since they are the same
  object — but no source read in this research states this outcome explicitly for a waived/reduced case, and
  this is named as an inference from the mechanism's own described behaviour, not a directly confirmed fact.
- **The real Nigerian example found confirms the task's own framing exactly, from two independent, named,
  real institutions' own published material, not a secondary summary of either.** A real Nigerian digital
  bank's own consumer-facing blog post states plainly: "You send ₦50,000 to your friend. Your bank charges a
  transfer fee of ₦50 and the Federal Government charges 7.5% VAT on the transfer fee" — 7.5% of ₦50 is
  ₦3.75, and "your transfer or payment amount stays the same... The VAT is only on the service fee." [Kuda —
  What The 7.5% VAT On Transaction Fees Really Means](https://www.kuda.com/blog/what-the-7-5-vat-on-transaction-fees-really-means/)
  (fetched and read directly). A real Nigerian commercial bank's own published tariff guide shows the
  identical shape for a different charge entirely — cheque-book issuance ("25 leaves: ₦750 + ₦56.25 VAT") and
  a stopped-cheque charge ("₦500 + ₦37.50k VAT") — VAT shown as a separate line, always 7.5% of the
  *preceding fee*, never of any transaction principal. [Access Bank Plc — Rates Guide](https://www.accessbankplc.com/Rates-Guide/index.html)
  (fetched and read directly). Two different real institutions, two different charge types (an electronic
  transfer fee and a cheque fee), the same tax-of-fee computation — this is not an isolated or assumed
  pattern.

**What I could not verify:** whether any real system models this specifically as "a rule with a base-rule-id
field, recomputed live at quote time if the base rule's own amount later changes" — the one real mechanism
found (Fineract) bundles tax into the charge's own definition rather than keeping two independently
cross-referencing rule rows, so this research did not find a real example of the generic "rule references
another rule's id" shape the task names as one of the two options, only this third, more tightly-coupled one.

---

## Q3: Nigerian stamp duty, read from a real source, not assumed

**Confidence: medium** on the Finance Act 2020/EMTL mechanism's existence, statutory section, and 2022
Regulations' mechanics (converging, directly-read, named secondary legal-analysis sources); **low-medium** on
the January 2026 sender-pays change and the FCY-applicability question, both flagged explicitly below as
unresolved tensions, not settled facts; **the CBN's own primary text could not be read at all**, stated
plainly, not guessed around.

- **The statutory basis is real and specific: Finance Act 2020, section 48, inserted a new section 89A into
  the Stamp Duties Act**, creating the Electronic Money Transfer Levy — "a levy... on electronic receipts or
  electronic transfer for money deposited in any deposit money bank or financial institution, on any type of
  account." [International Bar Association — "Not So Novel: Nigeria Electronic Money Transfer Levy"](https://www.ibanet.org/not-so-novel-Nigeria-Electronic-Money-Transfer-Levy)
  (fetched and read directly).
- **The flat amount and threshold, confirmed by two independent, directly-read professional-services sources:
  ₦50 flat, on electronic transfers/receipts of ₦10,000 or more.** [IBA, above]; [KPMG Nigeria — "The
  Electronic Money Transfer Levy Regulations, 2022"](https://kpmg.com/ng/en/home/insights/2022/12/the-electronic-money-transfer-levy-regulations--2022.html)
  (fetched and read directly).
- **Collection and remittance, per the same two sources:** the *receiving* bank collects the levy and must
  remit it to the Federal Inland Revenue Service (FIRS) by the next working day after the transaction;
  proceeds are split 15% to the federal government/FCT and 85% to state governments by derivation. Banks must
  keep records (seven years, per KPMG's reading) and file monthly returns.
- **A real, honestly-unresolved tension on who pays, surfaced by this research rather than smoothed over.**
  KPMG's own 2022 reading of the Regulations states the levy "targets the recipient" of the transfer. Several
  independent 2026 Nigerian news sources (not independently fetched and read in full — titles/gist via
  search-engine summary) report that Nigeria's 2025 tax reform (the Nigeria Tax Act) and CBN implementation
  guidance shift this so the *sender* pays, effective 1 January 2026. [Hallmark News — "CBN orders banks to
  effect ₦50 stamp duty on transfers above ₦10,000"](https://hallmarknews.com/cbn-orders-banks-to-effect-n50-stamp-duty-on-transfers-above-n10000/);
  [National Accord Newspaper](https://nationalaccordnewspaper.com/nigerian-banks-to-begin-%E2%82%A650-stamp-duty-on-transfers-above-%E2%82%A610000-from-january-1-2026/)
  (both title/gist via search, not independently fetched). This research did not read the Nigeria Tax Act's
  own primary text to confirm this change, and says so plainly rather than treating the news coverage as
  equivalent to having read the Act.
- **A second, more consequential tension: does this levy even apply only to local currency (LCY), as this
  task's own framing assumes?** KPMG's own reading of the 2022 Regulations states plainly: "For foreign
  currency transactions, the levy will be charged at the exchange rates determined by the Central Bank of
  Nigeria" — i.e., per this one real, named, directly-read source, the EMTL/stamp duty is **not** described as
  an LCY-only concept; it is described as applying to FX transactions too, converted at a CBN-determined rate
  for levy-calculation purposes. This directly complicates — without this research resolving it either way —
  this task's own premise that stamp duty is "distinct... for local-currency electronic transfers
  specifically." Stated plainly as an open, unresolved question this research surfaced, not quietly
  overridden in either direction.
- **The 2026 CBN "Guide to Charges" exposure draft — explicitly draft, not final, per the legal commentary
  reading it, and not independently located or read by this research at all.** Multiple April/May 2026 legal
  and news commentary sources describe a CBN exposure draft revising the electronic-transfer-fee tiers (e.g.
  "transfers below ₦5,000: free; ₦5,000–₦50,000: ₦10; above ₦50,000: ₦50") and a stopped-cheque charge of
  ₦5,000 — but one of these sources states explicitly, in its own words, "we did not find a final 2026 guide
  published by the CBN" and that the 2020 figures remain "the last confirmed baseline" until a final version
  is published. [The Gazette NG — "Bank Charges in Nigeria 2026: What the CBN Guide Confirms and What Is
  Still Draft"](https://thegazette.ng/bank-charges-nigeria-2026-cbn-guide/) (fetched and read directly);
  [Candide-Johnson Law Practice — "Nigeria's New Banking Charges Framework"](https://candidejohnsonlaw.com/articles/nigerias-new-banking-charges-framework-what-every-client-needs-to-know/)
  (fetched and read directly, same draft-status caveat). This research did not find a working URL for the
  exposure draft itself to read directly; a guessed `cbn.gov.ng` URL for it returned a plain 404.
- **The CBN's own primary text — attempted directly, and confirmed unreadable, the same wall as every prior
  CBN-sourcing attempt on this project.** Both the 2020 "Guide to Charges" circular and the 2017 predecessor
  circular were fetched as raw bytes directly from `cbn.gov.ng`, and both rendered only as garbled,
  non-quotable binary content by this toolchain — stated plainly, exactly as T-037/T-041/T-043/T-054/T-056
  already documented for other CBN PDFs, not papered over or guessed around. One figure surfaced only through
  a search engine's own extraction of that same unreadable 2020 PDF — "stopped cheques: ₦500 per order,
  counter cheques: ₦50 per leaflet, no charge for returned cheques for reasons other than the account being
  unfunded" — is named here at low confidence precisely because this research's own direct fetch of the same
  document could not independently confirm it.

**What I could not verify:** the Nigeria Tax Act 2025's own primary text (for the sender/receiver-payer change
and any FX-specific provision); the CBN's own primary text of any "Guide to Charges" circular, any version;
whether the 2026 exposure draft has since been finalized. **A qualified compliance officer or tax adviser
must confirm the current, binding rule** before any future GARBATI design relies on a specific figure from
this brief — this is exactly the kind of regulatory-adjacent claim this project's own discipline
(T-037/T-041/T-043) requires sourcing carefully, and this research states plainly that it could not fully
discharge that sourcing on the CBN primary-text side.

---

## Q4: FCY-specific charge differentiation

**Confidence: medium.** One real, directly-read, named Nigerian bank's own published tariff is the strongest
evidence; the rest is weaker, named plainly below.

- **One real, named Nigerian commercial bank's own published rate guide structures LCY and FCY charges as
  separate named sections of one document, not a currency column on one shared fee table.** Access Bank's own
  Rates Guide documents Naira transfer charges separately from dedicated "Foreign Funds Transfers" and "Form
  'A' Charges" sections, with their own commission and offshore-charge schedules. [Access Bank Plc — Rates
  Guide](https://www.accessbankplc.com/Rates-Guide/index.html) (fetched and read directly). This is a real,
  concrete answer to one of the task's own named options: at least this one real institution's own published
  schedule, FCY is not a field value on an otherwise-shared LCY fee-rule row; it is its own separate schedule.
- **A different real platform's own fee-definition schema shows the opposite structural choice: currency as a
  field on one shared rule shape, not a separate schedule.** Mambu's custom predefined fee API is described
  (via a search-engine summary of Mambu's own documentation, not independently fetched in full — the direct
  fetch attempted in this research returned only a generic documentation-hub landing page, not the actual
  schema content, so this is named at correspondingly lower confidence) as including "fields for amount,
  currency, active status..." with currency as a three-letter ISO code on the fee definition itself.
- **The honest synthesis, consistent with this project's own repeated finding of "no single universal shape"
  (T-054, T-056): both real structural options exist in practice, and this research did not find one
  dominant answer.** A currency-scoped separate schedule (Access Bank's own real, published pattern) and a
  currency field on one shared rule (Mambu's apparent pattern, at lower confidence) are both real, and the
  task's own framing names both as live possibilities for good reason — neither was found to be the
  dominant convention.
- **Whether stamp duty/EMTL itself is LCY-only is not settled by this research** — see Q3's own explicit
  flag of the KPMG-sourced FX-applicability tension. This research does **not** confirm the task's own
  framing that stamp duty is "distinct... for local-currency electronic transfers specifically"; it surfaces
  a real, named source that complicates that framing and leaves the question open.

**What I could not verify:** no non-Nigerian or Nigerian core-banking vendor's own detailed fee-engine
documentation on currency-scoped charge configuration (beyond the Access Bank and Mambu findings above) was
independently read in full. A Caribbean regional banking group's own tariff guide (not Nigerian, used here
only as a weak, additional structural data point, not as Nigeria-specific evidence) has section headers
visible in search results naming "Inward LCY and FCY" as a structural grouping, but the PDF itself rendered
as unreadable binary content when fetched directly in this research, so only the header text visible in the
search result itself could be used, not the document's actual content. [NCB Group — Tariff Guide](https://ncbagroup.com/wp-content/uploads/sites/2/2026/05/Final-Tariff-Guide-A3-Original-05.26.pdf)
(header text only, via search result; PDF itself unreadable on direct fetch).

---

## Q5: Cheque-specific charges, and whether cheque usage is actually significant for Nigerian microfinance banking today

**Confidence: medium-high** on cheque usage being a small and rapidly declining channel overall in Nigeria
(converging, named, NIBSS-sourced figures); **medium-high** on Nigerian MFBs specifically being structurally
barred from direct cheque-clearing participation (a real, specific regulatory finding, though sourced only
via search-engine summary of an unreadable CBN PDF, flagged accordingly); **medium** on the specific cheque
charge figures (one real, directly-read institution's own tariff).

- **Real charge types and figures, from one real, named Nigerian commercial bank's own published tariff,
  fetched and read directly, matching the task's own named list closely:** cheque-book issuance, tiered by
  leaf count and including VAT as a separate line ("25 leaves: ₦750 + ₦56.25 VAT"; "50 leaves: ₦1,500 +
  ₦112.5 VAT"; "100 leaves: ₦3,000 + ₦225 VAT"); a stopped-cheque charge ("₦500 + ₦37.50 VAT"); and a
  returned/bounced-cheque charge structured as a percentage-with-floor rather than a flat amount ("1% of
  amount or ₦5,000, whichever is higher," with no VAT line shown for this specific charge in the source
  read). [Access Bank Plc — Rates Guide](https://www.accessbankplc.com/Rates-Guide/index.html) (fetched and
  read directly). This confirms the task's own named shape (processing/stop-payment/returned-cheque as
  distinct, separately reason-coded charges) is real, not a guessed taxonomy — though a cheque-presentment
  processing fee specifically (as opposed to issuance/stop/return) was not separately itemized in the one
  source read.
- **Cheque usage in Nigeria overall is real, named, and sharply declining — checked rather than assumed, as
  the task required.** Press reporting citing NIBSS data states cheque transaction *value* fell roughly
  49.8% to ₦1.837 trillion in 2025 from ₦3.658 trillion in 2024, and cheque transaction *volume* fell roughly
  57.8% to 1.49 million from 3.53 million over the same period, while electronic-transfer volumes grew into
  the tens of billions of transactions annually over the same few years. This is a search-engine composite
  summary drawing on several named Nigerian business-press outlets, not one single article independently
  fetched and read in full, and is flagged at correspondingly reduced (though still medium-high) confidence —
  the clear overall direction (cheques a small and shrinking share of Nigerian payments) is consistent across
  every source found, even if the precise year-over-year multiples vary somewhat between individual articles.
- **The single most directly relevant finding for weighting this question specifically for GARBATI (a
  microfinance platform, not a commercial bank): Nigerian microfinance banks are reported to be explicitly
  barred from direct cheque-clearing-house participation by CBN's own regulatory guidelines** — "no
  microfinance bank (MFB) shall engage in cheque clearing activities," listed among the regulatory
  guidelines' own non-permissible-activities list for MFBs, while MFBs are separately permitted to "collect
  money or proceeds of banking instruments on behalf of customers, including clearing of cheques through
  correspondent banks" — i.e., an MFB can facilitate a customer's cheque being cleared only by routing it
  through a correspondent commercial bank, not by running its own clearing relationship. This is sourced only
  via a search-engine summary of CBN's own 2006 "Regulatory and Supervisory Framework for Microfinance Banks"
  PDF — this research attempted to fetch that exact document directly and it rendered as garbled, unreadable
  binary content, the same CBN access wall described throughout this brief — so this specific, structurally
  important finding is flagged at reduced confidence precisely because this research's own direct read could
  not independently confirm what the search engine's own extraction reported.
- **The honest answer to the task's own "check rather than assume" instruction:** cheque usage is real, but
  genuinely minor and declining in Nigerian payments generally, and — per the one regulatory finding above,
  at reduced confidence because the primary text could not be independently read — structurally more
  constrained for a microfinance institution specifically than for a commercial bank like the one whose
  tariff guide supplied the charge figures above. This research did not find a reason to weight cheque-charge
  modeling heavily for GARBATI relative to the electronic-transfer charge questions in Q2/Q3, though it also
  did not find a source stating plainly that Nigerian MFBs never issue cheque books at all — the finding is
  "structurally limited and declining," not "zero," and a qualified compliance officer should confirm exactly
  what cheque-adjacent services, if any, a specific licensed MFB may offer before this question is weighted
  further in a future design.

**What I could not verify:** the CBN's own primary MFB regulatory-guidelines text (to independently confirm
the cheque-clearing restriction above); any Nigerian-MFB-specific (as opposed to general Nigerian banking)
cheque-volume or cheque-charge data — every figure found in this section is either general-Nigerian-banking
(the NIBSS decline figures) or from one real commercial bank (Access Bank, not an MFB).

---

## Q6: Does GARBATI's existing `LegPurpose`/`FeeRule` mechanism fit this, or does real practice need
something new? — a structural comparison, read directly from the actual code and `ARCHITECTURE.md`

This is the question the task names as most directly useful to a future design task, so — matching T-056's
own Q5 discipline — it is treated with the most care and an explicit effort not to force an answer either
way beyond what Q1-Q5 actually support.

**GARBATI's existing mechanism, read directly from `Garbati.ProductApi/src/Garbati.ProductApi/Entities/FeeRule.cs`,
`Services/Implementations/FeeQuoteService.cs`, and `ARCHITECTURE.md` Sections 9/11/12 (not re-derived from the
task file's own summary, read again here because the exact current shape matters for this comparison):**

- `FeeRule` has exactly six fields: `Id`, `ProductCode`, `OperationType`, `FeeType` (a string, used as
  `"Flat"`/`"Percentage"`/anything-else-means-`"None"` in a C# `switch` in `FeeQuoteService`), a nullable
  `Amount`, and `EffectiveFrom`/`EffectiveTo`. There is no `Currency` field anywhere on this entity, and no
  field of any kind that references another `FeeRule`'s id or row.
- `FeeQuoteService.GetQuoteAsync` looks up candidate rows filtered only by `(ProductCode, OperationType)` and
  picks the one effective "now," computing `Flat => Amount` or `Percentage => request.Amount * Amount / 100`
  — the percentage is always of the *caller's own request amount* (the transaction's principal), never of
  another rule's computed result. One call returns exactly one `FeeAmount`/`FeeType` pair, or
  `NoFeeConfigured`; there is no concept of "the list of rules that apply to this operation," only "the one
  rule that matches this product/operation pair right now."
- `ARCHITECTURE.md` Section 12.3 confirms `LegPurpose` is `Principal`/`Fee`/`Tax` only, closed by decision
  #45/D-7, with a new value explicitly named as "a new decision and a ledger code change, not an automatic
  extension." Section 11 item 4 confirms fee-income and tax-payable system accounts already exist per
  currency (decision #46), but "nothing routes a Fee/Tax leg to them yet."

**Where real practice fits this mechanism naturally, as a bounded extension:**

- **Posting several distinct, tagged charge legs atomically on one transaction is, structurally, exactly
  what GARBATI's existing chain/leg mechanism (P-14, decision #45) already does, and matches the clearest
  real-standard finding in Q1.** ISO 20022's own `ChargesInformation` array attached to one `pacs.008`
  payment, and GARBATI's own chain of tagged legs (`Principal`, `Fee`, `Tax`) committed in one database
  transaction, are the same shape: several itemized charge records belonging to one logical transaction,
  each individually identifiable, all atomic. A future stamp-duty leg or a VAT leg, as individually tagged
  legs within the existing chain mechanism, would need nothing structurally new from `ChainService`/
  `LegPoster` itself — this is squarely what the chain model was already built to do.
- **A new, narrowly-scoped `LegPurpose` value (e.g. `GovernmentLevy` for stamp duty) and a corresponding new
  System-account type would be a real but incremental, bounded extension of a mechanism that already exists
  — not a new mechanism** — the same shape as decision #46's existing fee-income/tax-payable accounts, just
  a third purpose and a third account type, exactly as `ARCHITECTURE.md` Section 9 itself already names as
  the expected cost of a new purpose code ("a new decision and a ledger code change").

**Where real practice needs something `FeeRule`'s current shape has no analogue for at all — the honest,
unforced finding:**

1. **No currency field or currency-scoped rule set exists on `FeeRule` at all, and Q4 found real systems take
   either of two different approaches (a currency field on a shared rule, or a wholly separate schedule) —
   `FeeRule` today has neither.** A single `(ProductCode, OperationType)` lookup with no currency dimension
   cannot express "this operation's charge schedule is different for USD than for NGN" in any form today;
   this is a genuine gap, not a stretch of an existing field.
2. **No threshold-gated logic exists** — stamp duty's own real-world shape (Q3: a flat ₦50 that applies only
   "on any transfer of ₦10,000 or more") cannot be expressed by `FeeRule`'s `Flat`/`Percentage`/`None`
   calculation types, which apply unconditionally to every amount once a rule matches. This is a real,
   specific gap this research's own reading of the Nigerian EMTL mechanism surfaced directly, not a
   generic "might need more flexibility someday" concern.
3. **No mechanism exists for one charge to be computed from another charge's own result, and — per Q2's own
   finding — the one real system found (Fineract) does not solve this with a generic "rule references
   another rule's id" field either.** `FeeRule`'s `Percentage` calculation type multiplies only against
   `request.Amount` (the transaction principal); it has no way to multiply against a *previously computed fee
   amount* instead. Fineract's own real mechanism (`TaxGroup` bundled directly into a `Charge`'s own
   definition, Q2) suggests the more realistic extension is not "let one `FeeRule` row point at another
   row's id," but something closer to "let a `FeeRule` optionally carry its own attached tax specification,
   computed from its own resulting amount" — a real, different, and narrower shape than a fully generic
   cross-reference mechanism, and this research does not pick between the two; it names both as live options
   for a future design task, consistent with the task's own instruction not to design anything here.
4. **`FeeQuoteService` returns exactly one fee for one `(ProductCode, OperationType)` call — there is no
   concept of "the ordered list of charges that apply to this operation," which every real multi-charge
   scenario in this brief (VAT-of-fee, stamp duty, cheque issuance-plus-VAT) structurally needs.** Even if
   currency, threshold, and base-reference gaps above were each individually solved, the service's own
   "one call, one answer" shape would still need to become "one call, an ordered list of answers" for the
   orchestrator to build a multi-leg chain the way `ARCHITECTURE.md` Section 4.1 already describes ("the
   orchestrator... builds the full set of postings") — this is a real, structural gap in `FeeQuoteService`'s
   own current contract shape, separate from anything about `FeeRule`'s own table columns.

**The honest, non-decided synthesis:** GARBATI's existing chain/leg-posting mechanism (the *execution* half
of this problem) is structurally well-suited to multi-charge transactions — real practice's clearest
standard (ISO 20022) itemizes charges in almost exactly this shape, and GARBATI already has it, built and
tested, today. `FeeRule`/`FeeQuoteService` (the *computation* half) is **not** currently suited, with nothing
to stretch, to express any of the four real-world requirements this research found (currency-scoped
schedules, threshold gates, charge-on-charge compounding, and a list rather than a single answer per call) —
each is a genuine gap against what real systems (ISO 20022's own charge taxonomy, Fineract's own
`TaxGroup`-on-`Charge` mechanism, and two real Nigerian banks' own published tariffs) actually do, not a
stretch of a field or a calculation type that already exists. This is explicitly left as input to a future
design task, not a decision made here — this research does not propose a schema, a new `FeeType` value, or a
new API shape for `FeeQuoteService`, per the task's own scope.

---

## What this does not settle

- **Does not design a charge-modeling feature, propose a schema, or write a task file for building one**, per
  the task's explicit scope. Q6 names where `FeeRule`'s current shape fits real practice and where it does
  not; it does not choose a column, a `FeeType` value, or an API contract for either half.
- **Does not assert a specific stamp duty amount, VAT rate, or cheque charge figure as settled, binding
  current law.** Every figure in this brief is attributed to a specific, named, dated source at an explicitly
  stated confidence level, with two real tensions (the sender/receiver-payer timing, and the LCY/FCY
  applicability of stamp duty) stated plainly as unresolved rather than picked one way or the other. **A
  qualified tax/compliance officer must confirm the current, binding rule** before any future design relies
  on a figure from this brief.
- **Is not a Sharia ruling and does not touch Sharia questions.** Charge/fee/tax mechanics are a
  technical-modeling and regulatory topic here; nothing in this brief bears on GARBATI's own existing
  Sharia-governance open questions (`research/sharia-product-rules.md`, `research/sharia-rule-configuration-patterns.md`),
  and `FeeRule` remains unseeded regardless of this research, exactly as the task required stating plainly.
- **Does not assume urgency or propose a timeline.** `FeeRule` is deliberately empty pending scholarly
  sign-off; nothing here changes that, and nothing here claims this gap needs closing before that happens.
- **Does not independently confirm several figures cited at reduced confidence** — most importantly the
  Nigerian-MFB cheque-clearing restriction (Q5) and the 2020 CBN "Guide to Charges" cheque figures (Q3), both
  sourced only via a search engine's own extraction of a CBN PDF this research's own direct fetch could not
  render as readable text.

## What this affects

This is new research; `ARCHITECTURE.md` Section 9 does not currently name multi-charge/compounding-fee
modeling anywhere in its own open-questions list (the closest existing entries — "Leg purpose codes" and the
Sharia-rules entry — name the closedness of `LegPurpose` and the unseeded state of `FeeRule`, but not the
specific currency/threshold/charge-on-charge gaps this research found). The coordinator should consider
whether this research becomes a new Section 9 entry (e.g., "Multi-charge, compounding, currency-dependent
transaction charges — not researched before T-057; see `research/multi-charge-transaction-modeling.md`"), the
same way `research/teller-till-vault-cash-operations.md` (T-056) is a candidate for its own new entry, given
that this research names four specific, concrete gaps in `FeeRule`'s current shape (no currency field, no
threshold logic, no charge-on-charge reference, and `FeeQuoteService`'s "one call, one answer" contract) that
the existing "Leg purpose codes" and Sharia-rules entries do not currently capture. This research does not
resolve which of those four gaps should be closed first, or how — that is explicitly left to a future design
task, informed by this brief, exactly as this task's own scope requires.

## Sources

- [ISO20022Payments.com — Charges (ChargesInformation, ChargeBearer in pacs.008)](https://www.iso20022payments.com/cbpr/charges/) (fetched and read directly)
- [Nordea — Bank transaction codes for camt.053.001.02 (CHRG/TAXE charge-type codes)](https://www.nordea.com/en/doc/nordea-bank-transaction-codes-camt.053.001.02.pdf) (title/codes via search-engine summary, not independently fetched)
- [Fineract Platform Documentation](https://fineract.apache.org/docs/stable/) (fetched directly; Charge-entity field detail thin on this specific page)
- [apache/fineract PR #6692 — FINERACT-2969, taxed-charge bug fix](https://github.com/apache/fineract/pull/6692) (fetched and read directly)
- [Kuda — "What The 7.5% VAT On Transaction Fees Really Means"](https://www.kuda.com/blog/what-the-7-5-vat-on-transaction-fees-really-means/) (fetched and read directly)
- [Access Bank Plc — Rates Guide](https://www.accessbankplc.com/Rates-Guide/index.html) (fetched and read directly)
- [International Bar Association — "Not So Novel: Nigeria Electronic Money Transfer Levy"](https://www.ibanet.org/not-so-novel-Nigeria-Electronic-Money-Transfer-Levy) (fetched and read directly)
- [KPMG Nigeria — "The Electronic Money Transfer Levy Regulations, 2022"](https://kpmg.com/ng/en/home/insights/2022/12/the-electronic-money-transfer-levy-regulations--2022.html) (fetched and read directly)
- [The Gazette NG — "Bank Charges in Nigeria 2026: What the CBN Guide Confirms and What Is Still Draft"](https://thegazette.ng/bank-charges-nigeria-2026-cbn-guide/) (fetched and read directly)
- [Candide-Johnson Law Practice — "Nigeria's New Banking Charges Framework: What Every Client Needs to Know"](https://candidejohnsonlaw.com/articles/nigerias-new-banking-charges-framework-what-every-client-needs-to-know/) (fetched and read directly)
- [OpenAfrica Podcast (Substack) — "The CBN has announced new charges"](https://openafricapod.substack.com/p/the-cbn-has-announced-new-charges) (fetched and read directly)
- [Hallmark News — "CBN orders banks to effect ₦50 stamp duty on transfers above ₦10,000"](https://hallmarknews.com/cbn-orders-banks-to-effect-n50-stamp-duty-on-transfers-above-n10000/) (title/gist via search, not independently fetched)
- [National Accord Newspaper — "Nigerian banks to begin ₦50 stamp duty on transfers above ₦10,000 from January 1, 2026"](https://nationalaccordnewspaper.com/nigerian-banks-to-begin-%E2%82%A650-stamp-duty-on-transfers-above-%E2%82%A610000-from-january-1-2026/) (title/gist via search, not independently fetched)
- NIBSS-sourced cheque-decline figures (2024-2025 value/volume) — search-engine composite summary across several named Nigerian business-press outlets, not one article independently fetched and read in full
- CBN's own "Regulatory and Supervisory Framework for Microfinance Banks" (2006) — MFB cheque-clearing restriction found via search-engine summary only; this research's own direct fetch rendered the PDF as unreadable binary content
- [Mambu — custom predefined fee API v1 schema](https://docs.mambu.com/api/api-v1/schemas/schemas/custompredefinedfeeapiv-1/) (currency-field claim via search-engine summary; this research's own direct fetch returned only a generic documentation-hub landing page, not the schema content itself)
- [NCB Group — Tariff Guide](https://ncbagroup.com/wp-content/uploads/sites/2/2026/05/Final-Tariff-Guide-A3-Original-05.26.pdf) (non-Nigerian; section-header text only, via search result; PDF itself unreadable on direct fetch)
- Temenos Basecamp community forum — thread referencing "two separate taxes... calculated on the same charge property" (title/gist only via search; likely authentication-gated, not independently read)
- Internal: `ARCHITECTURE.md` Sections 4, 9, 11, 12 (read directly — `LegPurpose`, decisions #45/#46, P-9/P-14, Section 12.3)
- Internal: `Garbati.ProductApi/src/Garbati.ProductApi/Entities/FeeRule.cs` and `Services/Implementations/FeeQuoteService.cs` (read directly)
- Internal: `research/teller-till-vault-cash-operations.md` (T-056), built on for sourcing/confidence conventions, not re-derived
- Internal: `tasks/T-057-research-multi-charge-transaction-modeling.md` (the task file itself, read in full)

## What was attempted and blocked or unreadable (stated plainly)

- `https://www.cbn.gov.ng/Out/2022/CCD/Guide%20to%20Charges%20by%20Banks%20Other%20Financial%20and%20Non-Financial%20Institutions%20eff%20Jan%201%202020.pdf` — fetched as raw bytes; rendered only as garbled binary/stream content by this toolchain, the same `poppler-utils`-adjacent limitation named in T-037/T-041/T-043/T-054/T-056. Also attempted to read the saved binary file directly with this toolchain's PDF-page-reading capability; failed with "`pdftoppm` is not installed," confirming the limitation is in this environment's own tooling, not a one-off fetch glitch.
- `https://www.cbn.gov.ng/Out/2017/FPRD/Guide%20to%20Bank%20Charges%20Circular%20to%20all%20Banks%20Other%20Financial%20Institutions%20and%20Mobile%20Payments%20Operators.pdf` — same result, garbled/unreadable.
- `https://www.cbn.gov.ng/OUT/PUBLICATIONS/GUIDELINES/DFD/2006/REGULATORY%20GUIDELINES.PDF` (MFB regulatory framework) — same result, garbled/unreadable.
- `https://www.cbn.gov.ng/out/2020/fprd/emtl%20guidelines.pdf` (a guessed URL for CBN's own EMTL guidance) — 404 Not Found.
- `https://www.cbn.gov.ng/Out/2026/CCD/EXPOSURE%20DRAFT%20OF%20THE%20GUIDE%20TO%20CHARGES...pdf` (a guessed URL for the 2026 exposure draft) — 404 Not Found; no working URL for this document was located by search either.
- `https://finecko.com/docs/guides/charges.html` (a third-party Fineract charges guide) — HTTP 522 (server-side timeout/unknown status), not read.
- `https://cwiki.apache.org/confluence/display/FINERACT/Slab+Based+Charges` — connection refused on direct fetch, the same failure mode T-056 already documented for other `cwiki.apache.org` pages.
- `https://ncbagroup.com/wp-content/uploads/sites/2/2026/05/Final-Tariff-Guide-A3-Original-05.26.pdf` — fetched as raw bytes; rendered only as PDF structural/font metadata, not readable tariff content.
- `https://docs.mambu.com/api/api-v1/schemas/schemas/custompredefinedfeeapiv-1/` — fetched successfully but returned only a generic documentation-hub landing page, not the actual schema content being sought.
