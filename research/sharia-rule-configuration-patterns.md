# Research: how do real Islamic-finance core-banking systems make Sharia product rules *configurable* per adopting institution?

Task: T-050. Output informs a future owner decision about `Garbati.ProductApi`'s `FeeRule`/`ProfitSharingTerms`
table shape (built empty in T-009/T-045, per decision — pending a qualified scholar's sign-off before any
institution fills them in). **This document does not propose or imply any specific fee amount, profit-sharing
ratio, or Sharia ruling** — that is explicitly out of scope for this task and remains T-007's still-open
question (`research/sharia-product-rules.md`). This document is also not itself a Sharia ruling or a
Nigerian regulatory ruling; a qualified Islamic finance scholar and, separately, a Nigerian compliance
officer, must confirm anything here before it is treated as settled.

This is a different question from T-007. T-007 researched *what* the rules typically are (riba, Murabaha,
late-fee-to-charity). This brief researches the *structural* question: GARBATI is open-source (Apache 2.0,
decision D-21) — any bank or microfinance institution can adopt it, and different adopters will follow
different Sharia supervisory boards, schools of thought, and national regulators. How do real,
comparable systems let an adopting institution configure its own rules, rather than the platform vendor
hardcoding one fixed rule set?

---

## Plain-language summary (read this first)

A few terms, defined once:

- **AAOIFI**: Accounting and Auditing Organization for Islamic Financial Institutions, a Bahrain-based
  standard-setter. Publishes named Shari'ah, accounting, auditing and governance standards.
- **IFSB**: Islamic Financial Services Board, a Malaysia-based standard-setter focused on *prudential*
  (capital, risk, supervisory) standards for Islamic financial institutions, not Shari'ah rulings themselves.
- **SSB (Shari'ah Supervisory Board)**: an individual institution's own panel of qualified scholars who
  review and approve that specific institution's products. The generic, internationally-used term for
  what Nigeria's regulator calls an "Advisory Committee of Experts" (ACE) — see below.
- **Centralized vs. decentralized Sharia governance**: two different national models found in the real
  world. Decentralized (most of the GCC): each bank has its own SSB, and different banks' SSBs can and do
  reach different rulings on the same question. Centralized (Malaysia, and — in a modified three-layer
  form — Nigeria): a national body's ruling binds or constrains every institution's own board.

**Short answers to the five questions asked:**

1. **AAOIFI/IFSB are real, named, citable standards, but neither is universally binding, and neither
   replaces an institution's own Sharia board.** AAOIFI's own published adoption list states its Shari'ah
   standards are mandatory regulatory requirements (fully or partially) in a specific named set of
   countries — including Nigeria — and voluntary/guideline-only elsewhere. Even where AAOIFI standards are
   mandatory, individual institutions still maintain their own Sharia board, because AAOIFI standards set
   a common reference, not a complete, self-executing rule for every product. IFSB's standards are
   explicitly described, by IFSB's own account cited in secondary sources, as for voluntary adoption by
   member countries with no sanction mechanism. The honest shape: AAOIFI/IFSB function as a **shared
   template/vocabulary layer that most serious institutions align to**, not as a universal legal
   substitute for an institution's own Sharia board's sign-off.
2. **Real core-banking vendors (Path Solutions/iMAL, Temenos, Mambu, ICSFS/ICS BANKS) all publicly market
   configurability/"parameterization" of Islamic product terms as a selling point**, but none of their
   public marketing/product pages document the actual configuration mechanism (table shapes, approval
   workflow, data model) in the kind of technical detail GARBATI's own `DESIGN.md` documents would — this
   is expected for enterprise software and is stated plainly below rather than guessed at.
3. **A formal, documented, auditable Sharia-board product-approval step is a real regulatory requirement
   in at least one primary source read directly (Saudi Arabia's SAMA rulebook)** — committee sign-off,
   signed minutes, a kept record, for every product and every amendment. Nigeria's CBN runs a comparable,
   but more elaborate, **three-layer** version of this (see Q3 below). What I could not confirm is whether
   any of the four named *core-banking vendors'* software itself models this approval event as a
   first-class, structured record (an auditable workflow object, closer to GARBATI's own maker-checker
   patterns) — the regulatory requirement for documented sign-off is clearly real and primary-sourced; its
   presence as a literal software entity inside any specific named vendor's product is not something I
   could verify from public sources.
4. **Both layers vary, not just one.** Profit-sharing ratios and fee amounts clearly vary institution to
   institution, as expected. But the *set of permissible contract types itself is also not universally
   fixed* — the best-documented real example is *bay' al-'inah* and *organized tawarruq*: accepted and used
   by Malaysia's centralized Shari'ah Advisory Council, but ruled impermissible by Saudi Arabia's Council
   of Senior Scholars and disallowed by an International Fiqh Academy resolution, because most GCC
   scholars view it as economically equivalent to disguised riba. This means GARBATI's eventual financing
   module likely needs **two** configurable layers, not one: which contract types an institution's own
   board permits at all, and then the specific terms (ratios, fees) within each permitted contract type.
5. **GARBATI's own `KycTierPolicy` pattern (a jsonb-backed policy table the platform reads at runtime,
   decision #11) is a reasonable structural analogy at the "where does the number live" level, but the
   real-world evidence found here adds something `KycTierPolicy` does not currently have: a formal
   governance/provenance layer** — who (which board, which meeting, which reference) approved this
   specific value, kept as a checkable record, not just the value itself. Nigeria's own regulatory model
   (next section) is a closer domestic precedent than `KycTierPolicy` for this specific point, because it
   already combines "institution fills in its own specific rule" with "a named governance act approves it,
   and that act is itself subject to a second, higher-level check."

**Confidence overall: medium.** The *existence* of real, named standard-setters and a multi-layer
governance pattern is well corroborated by primary sources (AAOIFI's own adoption page, Saudi Arabia's
SAMA rulebook, academic literature with named authors, Nigeria's regulatory press/legal commentary). The
*internal software mechanics* of named core-banking vendors (exactly how Mambu/Temenos/Path Solutions/ICSFS
structure this in their own database) is **not** something public marketing pages document in enough
detail to verify directly — stated plainly below, not guessed at.

---

## Q1: AAOIFI and IFSB as a "common template" layer — fixed baseline, customizable reference, or something else?

**Short answer: a shared reference layer that ranges from legally mandatory (in a specific, named list of
countries) to purely voluntary guidance (elsewhere) — and even where mandatory, it coexists with, rather
than replaces, each institution's own Sharia board.**

- **AAOIFI's own published adoption page is the primary source here**, and it is direct and specific: AAOIFI
  Shari'ah standards are "adopted either fully or partially as mandatory regulatory requirements" in a
  named list that explicitly includes **Bahrain, Jordan, Kyrgyz Republic, Mauritius, Nigeria, Qatar, Qatar
  International Financial Centre (QIFC), Oman, Pakistan, Sudan, Syria, United Arab Emirates and Yemen** —
  and the same wording and country list is repeated separately for AAOIFI's Accounting standards and its
  Auditing/Governance/Ethics standards. Elsewhere, adoption is explicitly voluntary: Kuwait is listed as
  "recommended as guidelines," Bangladesh as "voluntary application" by individual banks, and Indonesia,
  Kazakhstan and Pakistan as a "basis for national standards" (i.e. adapted into local standards rather than
  adopted verbatim) ([AAOIFI, "Adoption of AAOIFI Standards"](https://aaoifi.com/adoption-of-aaoifi-standards/?lang=en)
  — read directly, not a secondary summary). AAOIFI's page itself carries a caveat worth repeating:
  adoption status "may be subject to verification from the jurisdictions as some of them may not publish
  it, and/or sources may be outdated" — AAOIFI is not claiming perfect, current knowledge of every
  jurisdiction's legal status, and neither does this brief.
- **Separately, I found at least one secondary source (an AI-generated web-search summary, not a primary
  citation) listing the UAE as voluntary** rather than mandatory — directly contradicting AAOIFI's own
  page, which lists the UAE under mandatory Shari'ah-standard adoption. Where AAOIFI's own primary page and
  a secondary summary disagreed, I have treated AAOIFI's own page as authoritative and flagged the
  discrepancy rather than silently picking one. This is exactly the kind of inconsistency a compliance
  officer should re-verify against the jurisdiction's own regulator directly before relying on it.
- **Even in a "mandatory" jurisdiction, AAOIFI adoption does not eliminate the institution's own Sharia
  board.** Every real system found in this research — regardless of whether AAOIFI compliance is mandatory
  there — still requires each institution to maintain its own Sharia Supervisory Board (SSB) or equivalent
  (Nigeria's "Advisory Committee of Experts," Saudi Arabia's "Shari'ah Committee") that reviews and signs
  off on that specific institution's own products. AAOIFI's standards function as a common vocabulary and
  a floor most serious institutions align to — a template an institution's own board customizes against,
  confirms compliance with, or in some documented cases (see Q4) explicitly departs from — not a
  self-executing ruleset a platform could hardcode and have every adopter simply inherit unmodified.
- **IFSB is a different, narrower thing than AAOIFI and should not be conflated with it.** IFSB (based in
  Kuala Lumpur, founded 2002) issues *prudential and supervisory* standards (capital adequacy, risk
  management, supervisory review) for Islamic financial institutions — not Shari'ah product rulings. Per
  Wikipedia's summary (cross-checked, not the only source, but consistent with what is publicly known
  about IFSB's remit), it had issued 32 Standards/Guiding Principles/Technical Notes as of October 2020
  across banking, capital markets and insurance ([Wikipedia, "Islamic Financial Services Board"](https://en.wikipedia.org/wiki/Islamic_Financial_Services_Board)).
  A separate secondary-source search found IFSB's own stated position that its "standards and core
  principles are for voluntary adoption by member countries," with "no legal obligation for member
  countries to adopt them and no sanctions against errant countries," and that IFSB runs a monitoring
  programme (the "Impact and Consistency Assessment Program," ICAP, from 2020) to track real-world
  implementation rather than enforce it — **I could not independently verify this exact "voluntary, no
  sanctions" wording against IFSB's own primary site**; it comes from a search-engine-generated summary of
  secondary sources, not a page I fetched and read myself. Treat the general shape (prudential standards,
  voluntary, member-driven) as reasonably confident; treat the precise "no sanctions" wording as
  unverified by me directly.

**What I could not verify:** the exact, current (2026) list of AAOIFI-mandatory jurisdictions could drift
over time and AAOIFI's own page flags possible staleness; I have not independently cross-checked every
country on that list against that country's own regulator. I could not fetch IFSB's own primary site
directly (not attempted via WebFetch in this research; relied on a secondary search synthesis plus one
Wikipedia cross-check) — a compliance officer wanting IFSB's precise current standards list should go to
ifsb.org directly.

---

## Q2: How do real core-banking vendors with Islamic-finance modules structure configurability?

**Short answer: every named vendor checked markets "configurable"/"parameterized" Islamic product terms
as a feature, but none publishes the underlying data model or workflow in the technical depth this
research was asked to characterize — this is stated plainly rather than inferred from marketing language.**

- **Path Solutions (iMAL).** Marketed explicitly as "built from the ground up to support Sharia banking
  operations," with "increased parameterisation capabilities" as a named advantage, and the only Islamic
  core-banking platform AAOIFI itself has certified (per Path Solutions' own account, corroborated by a
  2008 certification date and a later re-examination jointly conducted with Deloitte's Islamic Finance and
  Sharia professionals, reportedly finding "no exceptions on the compliance of iMAL ... with Sharia
  principles, and the Sharia and accounting standards issued by AAOIFI")
  ([Financial IT, "iMAL: Powering Islamic Banking Infrastructure"](https://financialit.net/publications/product-reviews/imal-powering-islamic-banking-infrastructure);
  [Retail Banker International, "Path Solutions"](https://www.retailbankerinternational.com/companies/path-solutions/)).
  **I could not find any public technical documentation describing iMAL's actual table/parameter
  structure** — "parameterisation" is asserted on marketing/review pages, not demonstrated with a schema or
  configuration screenshot I could access.
- **Temenos Islamic Banking.** Temenos's own product page states banks "can rapidly configure and introduce
  Shari'ah-compliant financing structures, accounts and deposits using advanced product capabilities,"
  with "configurable compliance to meet local regulatory needs" and "configurable controls and profit
  distribution capabilities" for transparency and auditability across "Islamic products and financing
  structures" ([Temenos, "Islamic Banking"](https://www.temenos.com/products/islamic-banking/); Temenos
  Islamic Banking factsheet PDF, same claims). **Same caveat**: this is product-marketing language
  confirming that configurability is the advertised model, not a technical specification of how that
  configurability is actually implemented.
- **Mambu Islamic Financing.** Mambu's own page gives the most concrete, closest-to-technical language
  found among the four vendors, read directly (not secondhand): "Configure Murabahah and Tawarruq products
  for personal, vehicle, and business financing using low-code tools"; "Define cost prices and profit
  margins (Ribh) with clear disclosure across financing contracts"; "Automate fee structures and
  late-payment charity contributions while maintaining Shari'ah compliance"; and — notably — a stated
  architectural guardrail rather than just a configuration option: "a native no-interest engine" intended
  to prevent "interest-based logic from being applied to financing products," alongside "Islamic and
  conventional products on a single foundation with operational and accounting segregation"
  ([Mambu, "Islamic Financing"](https://mambu.com/en/islamic-financing)). Mambu's public API documentation
  confirms Murabaha contracts exist as a distinct object with a lifecycle (`PENDING` → `ACTIVE` via
  activation, or withdrawn) ([Mambu docs, "Islamic financing murabaha contracts overview"](https://docs.mambu.com/api/api-v2/islamic_financing_murabaha_contracts/islamic-financing-murabaha-contracts-overview/)),
  but **the public API overview page does not expose the underlying parameter/table shape** (e.g. whether
  profit margin is a per-product template an institution sets once, or a per-contract override, or both) —
  that level of detail would require the full API reference or a customer-only portal, neither of which I
  could access.
- **ICSFS (ICS BANKS Islamic).** Marketed as "fully parameterised... designed and developed in compliance
  with international Islamic standards, including the AAOIFI and the Islamic Fiqh Academy," with modules
  for Murabaha, Ijara, Sukuk and "advanced Sharia auditing tools," explicitly stated to "cater to each
  bank's shari'ah regulations" ([ICSFS, "ICS BANKS Islamic Banking Solution"](https://icsfs.com/solutions/islamic-banking/)).
  Same limitation as the others: no public technical documentation of the actual data model found.

**Honest conclusion for Q2:** all four named, real vendors converge on the same *marketing claim* —
parameterizable/configurable Islamic product terms, explicit alignment to AAOIFI (named directly by Path
Solutions and ICSFS; implied by Temenos and Mambu's "Shari'ah compliance" framing without naming AAOIFI
specifically) — but **none publishes enough to confirm the actual mechanism** (one shared template plus
per-institution override rows, versus fully separate per-institution schemas, versus something else). This
is flagged plainly, per the task's own instruction, rather than guessed at. What can be said with more
confidence: enterprise core-banking vendors treat "an institution configures its own Islamic product terms
without needing a code change" as baseline expected behavior for this market segment, consistent with
what GARBATI is trying to achieve with `FeeRule`/`ProfitSharingTerms`.

---

## Q3: Does a single "Sharia board approval workflow" pattern exist as a first-class, auditable event?

**Short answer: yes, as a *regulatory/governance* requirement — confirmed directly from at least one
primary regulatory source — but I could not confirm it exists as a first-class *software* entity inside
any of the four named core-banking vendors. Nigeria's own model, read in detail below, is the most
concrete real-world precedent found, and it is a two-tier governance chain, not a flat single approval.**

- **Saudi Arabia (SAMA) — read directly from the regulator's own rulebook, not a secondary summary.** SAMA's
  "Shari'ah Governance Framework for Local Banks Operating in Saudi Arabia" requires, per its Article 18:
  "The committee approval shall be obtained for all Islamic banking products to be presented and for any
  subsequent amendments. The committee shall make detailed review of legal contracts and other documents
  related to the products and transactions" — covering "the concept, structure, terms and conditions,
  documentations, policies and procedures, pamphlets, brochures and advertising materials." Separately
  (Article 9), the framework requires committee decisions and voting results to be "documented and kept in
  a special and organized record," with meeting minutes including "discussions and deliberations," member
  names, and any objections, and requires "all attending members shall sign the minutes of meetings"
  ([SAMA Rulebook, "Shariah Governance Framework for Local Banks Operating in Saudi Arabia"](https://rulebook.sama.gov.sa/en/shariah-governance-framework-local-banks-operating-saudi-arabia)).
  This framework, notably, **does not reference AAOIFI standards** — it is SAMA's own independent
  requirement, reinforcing Q1's point that AAOIFI is a reference most institutions align to, not a
  universally legally-incorporated text.
- **Nigeria — a real, three-layer governance chain, more elaborate than a single board.** Every Nigerian
  Non-Interest Financial Institution (NIFI) must establish its own **Advisory Committee of Experts (ACE)**
  — the Nigerian-regulatory term for what the rest of the world calls a Sharia Supervisory Board. But an
  ACE's product-validation decisions are not final on their own: "the decision of each ACE as regards
  product validation is subject to ratification by **FRACE**" — the **Financial Regulation Advisory
  Council of Experts**, a national-level body (inaugurated by the CBN Governor in January 2013) that
  advises the CBN itself on Islamic commercial jurisprudence across *all* NIFIs, and whose members are, by
  rule, barred from sitting on any individual institution's own ACE (search-synthesis finding, triangulated
  across multiple independent news/legal-commentary sources: [Proshare, "CBN Unveils Guidelines on
  Advisory Body for Islamic Finance"](https://proshare.co/articles/cbn-unveils-guidelines-on-advisory-body-for-islamic-finance);
  corroborated by academic literature on FRACE, [ResearchGate, "Court referral and Nigeria's Financial
  Regulation Advisory Council of Experts (FRACE)"](https://www.researchgate.net/publication/334511794_Court_referral_and_Nigeria's_Financial_Regulation_Advisory_Council_of_Experts_FRACE)).
  I **could not read the primary CBN PDF guideline documents directly** — the ACE guidelines PDF and the
  FRACE governance-circular PDF both returned HTTP 403 or scanned-image content I could not extract text
  from (same access problem T-007 already flagged for CBN's other non-interest-banking circulars) — so
  this section rests on converging secondary/legal-commentary sources describing the primary documents,
  not my own reading of CBN's exact wording. A compliance officer should obtain and read the primary CBN
  ACE/FRACE guideline PDFs directly before relying on the specifics.
  The structural shape this leaves, even without the primary text: **institution-level board → national
  ratifying body → (above both) AAOIFI as a mandatory-in-Nigeria reference standard** — a genuine
  three-layer chain, not the simple two-layer "platform mechanism + institution config" shape GARBATI's
  `KycTierPolicy` currently has.
- **General Shari'ah-governance literature (secondary, synthesized, lower individual confidence per
  source, but broadly consistent across sources) describes the same shape in generic terms**: a product
  moves through "structuring, Shariah Board review, compliance sign-off, and periodic post-launch audit,"
  with a "Shariah Compliance Department" doing pre-approval review and a separate "Internal Shariah Audit"
  function independently testing whether approved products are actually implemented as certified,
  reporting through distinct lines specifically to preserve independence from commercially-pressured
  business units ([Medium/Shakuro, "Shariah-Compliant Banking Software Development"](https://medium.com/shakuro/shariah-compliant-banking-software-development-d8758157a0e6)
  — a software-vendor blog post, not a regulator or standard-setter, so treated as directionally
  consistent with the SAMA/Nigeria primary-ish findings above rather than as independent authority on its
  own).
- **What I could not verify:** whether any of the four named core-banking vendors (Q2) actually model this
  governance chain as a structured, queryable software object — e.g. a `ShariaApproval` entity with a
  status, an approver, a timestamp, and a link to the specific product/rule version it approves, the way
  GARBATI's own maker-checker and reversal-authority patterns work elsewhere in this codebase. Every
  source found describes the *governance requirement* (minutes, signed records, committee review) in terms
  of organizational process and document-keeping, not software architecture. It is a real, well-evidenced
  governance pattern; whether it is typically built as first-class software or handled as an external,
  paper/document-based process feeding into whatever the ops team then configures in the core-banking
  system is **genuinely unresolved** by anything I could access publicly.

---

## Q4: What actually varies institution-to-institution versus what's structurally fixed?

**Short answer: both layers vary — ratios/fees (expected), but also, less obviously, the set of
permissible contract types itself. This is the most concrete, well-documented finding in this brief, and
it directly affects whether `ProductApi` needs one configurable layer or two.**

- **Ratios and fees varying per institution is the uncontroversial, expected case** — nothing found
  contradicts GARBATI's own existing assumption that `ProfitSharingTerms`'s two ratio columns and
  `FeeRule`'s amount/percentage fields are exactly the kind of number that differs from one Sharia board's
  ruling to the next, even for the same named contract type (e.g. two Mudarabah-based savings accounts at
  two different institutions reasonably have different customer/institution profit-split ratios).
- **The genuinely informative finding: even the *permissible set of contract types* is not universally
  fixed across jurisdictions and schools of thought.** The best-documented real example is **bay'
  al-'inah** (a sale-and-buy-back structure) and **organized tawarruq** (a commodity-based liquidity
  structure), both used as alternative routes to interest-free cash financing:
  - Malaysia's centralized Shari'ah Advisory Council (Bank Negara Malaysia) has historically approved
    *bay' al-'inah* on grounds of *maslahah* (public interest/necessity, given the early lack of
    alternative Islamic products), subject to Bank Negara Malaysia's own standards and guidelines
    (search-synthesis finding across multiple academic sources on the "bay' al-inah controversy," converging
    on the same account: [Academia.edu, "The Bay' al-Inah Controversy in Malaysian Islamic Banking"](https://www.academia.edu/5167368/The_Bay_al_Inah_Controversy_in_Malaysian_Islamic_Banking)).
  - **The Saudi Council of Senior Scholars, by its own Resolution No. (3/11), 30 September 1977, ruled
    bay' al-'inah impermissible** — and most GCC-region scholars have not permitted its use, treating it as
    economically equivalent to disguised riba (same search-synthesis sourcing, cross-referenced across
    several academic papers on the Malaysia/GCC divergence).
  - **The International Council of Fiqh Academy's Resolution 179 specifically disallows *organized*
    tawarruq** (where the financing institution effectively arranges both legs of the commodity trade for
    the customer), on the grounds that the simultaneous, pre-arranged nature of the transaction — whether
    explicit, implicit, or by common practice — reproduces the economic substance of an interest loan
    rather than a genuine trade (search-synthesis finding; I was not able to read Resolution 179's primary
    text directly, only secondary academic summaries of it).
  - This is not a single source's claim — it recurs consistently across independent academic literature on
    Malaysia-vs-GCC Shari'ah governance, and is the standard textbook example cited for why "Malaysia's
    Shari'ah Advisory Council... created to harmonize Shari'ah interpretations" diverges in practice from
    "the decentralized [GCC] approach [where] Islamic banks have their own Sharia Boards... independent of
    the Central Bank," with "the national Sharia authority in GCC countries [not exercising] any
    supervision on Sharia practices at the institutional level" (search-synthesis across multiple
    comparative academic papers on Southeast Asia vs. GCC Shari'ah governance models; I did not read a
    single primary comparative study cover-to-cover, so treat the generalization as well-corroborated
    across sources rather than from one authoritative text).
- **Consequence for GARBATI:** the task file's framing ("do profit-sharing ratios and fee amounts vary
  while the set of permissible contract types stays fixed, closer to `MandateOperation`'s fixed-enum
  pattern — or does even the contract-type set vary too") has a clear, evidence-backed answer: **the
  contract-type set also varies**, at least across jurisdictions/schools, even if any single institution's
  own list is comparatively stable once its own board has ruled. This suggests GARBATI's eventual
  financing-contract configuration (not yet built; `ProductApi`'s current scope is deposit-side
  `Savings`/`Current` products only, per `ARCHITECTURE.md`'s "Financing services (Murabaha, etc.) — Later
  tranche") will likely need its *own* institution-configurable "which contract types do we even offer"
  layer, sitting structurally above whatever becomes the Murabaha/Musharakah-equivalent of today's
  `ProfitSharingTerms` — not a single fixed enum GARBATI ships once and every adopter inherits unmodified.
  This is a structural observation for a future task to weigh, not a schema proposal.

---

## Q5: Is `KycTierPolicy` a reasonable structural analogy, or does Sharia-rule configuration need a different shape?

**Short answer: reasonable at the "where does the configurable number live" level — both are a generic
mechanism (a table the platform reads at runtime) plus a jurisdiction/institution-specific row — but the
real-world evidence in this brief adds a dimension `KycTierPolicy` does not currently have: a formal,
documented governance/provenance layer over the value itself, which Nigeria's own ACE→FRACE chain (Q3)
makes into a genuine domestic precedent closer to what's needed than `KycTierPolicy` is today.**

- **Where the analogy holds.** `KycTierPolicy.RequiredEvidence` is exactly the pattern this research found
  real core-banking vendors describe at the marketing level for Islamic product terms too: a
  jurisdiction/product-scoped row, stored flexibly (jsonb in GARBATI's case), read at runtime, editable by
  an ops process without a code deploy. `FeeRule`/`ProfitSharingTerms` being `ProductCode`-scoped,
  versioned by `EffectiveFrom`/`EffectiveTo`, and genuinely empty until someone (an institution's own
  governance process) fills them in, is the same shape, applied to a different domain. Nothing found in
  this research contradicts that this is a workable mechanism for "many adopters, many configured rule
  sets" generally.
- **Where the analogy is incomplete.** `KycTierPolicy`'s placeholder rows were filled in by the GARBATI
  team itself via a migration, with a code comment flagging them as placeholders pending real confirmation
  — there is no modeled concept of *which external governance act* (a specific board meeting, a specific
  scholar's fatwa, a specific ACE ruling later ratified by a national body) justifies any specific row's
  value, nor any way to check, from the data alone, whether a given `KycTierPolicy` row has ever actually
  been through that process versus still being an unconfirmed placeholder. For Sharia-rule configuration
  specifically, the real-world evidence (SAMA's signed-minutes requirement; Nigeria's two-tier ACE→FRACE
  ratification chain) suggests this provenance gap matters more than it does for KYC tier policy, because
  a Sharia board's specific sign-off on a specific numeric rule is the entire basis for the institution
  being able to call that rule "Sharia-compliant" at all — not a convenience configuration, but the thing
  that makes the row legitimate in the first place.
- **What this suggests, as input to a future decision, not a decision:** a future Sharia-configuration
  layer built on the same `KycTierPolicy`-style mechanism would likely be strengthened by also recording
  *who/what approved this specific value* (a reference to a board decision, a date, possibly a document
  reference) alongside the value itself — not because this brief is proposing a schema (explicitly out of
  scope), but because every real governance source found treats the approval record as being at least as
  important as the number it approves, and nothing in GARBATI's current `KycTierPolicy` precedent captures
  that dimension yet either. Whether that provenance belongs inside `ProductApi` itself, in a separate
  governance/audit service, or purely in an external document (the way Nigeria's own ACE minutes
  presumably live outside any specific bank's core-banking software) is exactly the kind of judgment call
  this brief is flagging for the owner, not resolving.

---

## What this research does not settle

- **Does not propose, invent, or imply any specific fee amount, profit-sharing ratio, or Sharia ruling.**
  Every number mentioned above (ratios, resolution numbers, dates) describes what a cited real-world
  source says exists elsewhere — never a recommendation for GARBATI's own values.
- **Does not design a database schema or draft a task file.** The observations in Q4 and Q5 about what a
  future configurable layer might need (contract-type-level configurability; an approval/provenance
  reference alongside each value) are structural observations for the owner to weigh, not a table design.
- **Does not assume AAOIFI or IFSB are legally binding anywhere specific without citing the source for
  that claim** — Q1 states plainly which countries AAOIFI's own page lists as mandatory versus voluntary,
  flags one internal inconsistency found between AAOIFI's own page and a secondary summary, and states
  directly that IFSB's "voluntary, no sanctions" characterization rests on a secondary synthesis I could
  not independently verify against IFSB's own primary site.
- **Could not verify:**
  - The exact current (2026) AAOIFI mandatory-adoption list, beyond what AAOIFI's own page states today
    (which itself warns it may not be current for every jurisdiction).
  - IFSB's precise current standards list and the exact wording of its "voluntary adoption" policy —
    relied on a secondary synthesis, not IFSB's own primary site.
  - The primary text of Nigeria's CBN ACE guidelines and FRACE governance circular — both returned
    HTTP 403 or unreadable scanned-image content; this section rests on converging secondary/legal-
    commentary sources describing those documents, the same access limitation T-007 already flagged for
    other CBN non-interest-banking PDFs.
  - The primary text of International Fiqh Academy Resolution 179 on organized tawarruq — read only
    secondary academic summaries of it.
  - Whether any of the four named core-banking vendors (Path Solutions/iMAL, Temenos, Mambu, ICSFS)
    actually models a Sharia-board-approval event as a first-class, structured software entity, as
    opposed to handling it entirely as an external governance/document process feeding into whatever the
    ops team configures afterward — I found strong evidence for the governance *requirement* existing
    (SAMA, Nigeria's ACE/FRACE, general Shari'ah-governance literature) and no public evidence either way
    about how, or whether, any specific named vendor's *software* represents that event.
  - Whether GARBATI is intended to be deployed once per adopting institution (each institution running its
    own copy of `ProductApi`, which would make today's single-institution-shaped table design broadly
    adequate) or as a shared, multi-tenant platform serving several institutions from one deployment
    (which would require an explicit institution/tenant key on `Product`/`FeeRule`/`ProfitSharingTerms`
    that does not exist today) — nothing in `ARCHITECTURE.md` or the task files read for this research
    settles this, and it materially changes how urgent any schema rethink is. This is flagged as a
    prerequisite question for whoever makes the eventual decision, not answered here.

---

## Needs scholar / compliance-officer confirmation

As with `research/sharia-product-rules.md`, nothing in this brief should be cited as settled doctrine or
settled Nigerian regulatory fact:

1. The exact current legal status of AAOIFI/IFSB standards in any specific jurisdiction GARBATI might be
   deployed into, including Nigeria — confirm directly against the regulator, not against this brief or
   AAOIFI's own adoption page alone (which itself warns of possible staleness).
2. The current, exact wording and structure of Nigeria's ACE and FRACE guidelines — I could not read the
   primary CBN PDFs; a compliance officer must obtain and read them directly.
3. Whether, and how, GARBATI's eventual Sharia-governance design should mirror the ACE→FRACE two-tier
   ratification chain, a single-institution-board model, or something else — a question for the owner and
   GARBATI's eventual appointed Sharia advisor, informed by but not decided by this brief.
4. Anything about which contract types (Murabaha, Musharakah, Ijarah, Qard Hasan, Tawarruq, bay' al-'inah,
   etc.) GARBATI's own Sharia board will ultimately permit, and under what conditions — explicitly a
   scholar's decision, not a researcher's.

---

## Relevant open question in ARCHITECTURE.md

Section 9, "Sharia rules": *"Which product rules (profit-sharing instead of interest, late-fee-to-charity,
no prohibited charges) are enforced by ProductApi and which by the later financing services?"* This brief
does not answer that question either, but adds a closely related one the owner may want to fold into the
same eventual decision: **whether `ProductApi`'s configuration mechanism needs to support more than one
institution's differing rules simultaneously (multi-tenancy), and whether a Sharia-board approval
reference needs to be a modeled part of `FeeRule`/`ProfitSharingTerms` (or their financing-side
successors) rather than left as an external, undocumented fact** — both are structural questions this
research surfaced but does not resolve.

---

## Bottom line on `FeeRule`/`ProfitSharingTerms` — input to a future decision, not a decision

**The current table shape is a reasonable foundation and is not obviously wrong** — `ProductCode`-scoped,
`EffectiveFrom`/`EffectiveTo`-versioned, deliberately left empty until a real governance process fills it
in, read by the platform at runtime rather than hardcoded in a switch statement. This matches, at a
structural level, both GARBATI's own `KycTierPolicy` precedent and the general shape every named
core-banking vendor markets ("configure your own Islamic product terms without a code change").

**Three gaps are worth the owner's attention before real institution-specific Sharia configuration is
safely layered on top**, each one surfaced directly by the evidence above, not invented:

1. **No provenance/approval reference.** Neither table records *which* governance act (a board decision, a
   fatwa reference, an ACE ruling later ratified by a national body, a date) justifies the specific value
   in any given row. Every real governance source found (SAMA's signed-minutes rule; Nigeria's ACE→FRACE
   chain) treats that record as essential, not optional — and `KycTierPolicy` doesn't model it either, so
   this isn't a `ProductApi`-specific defect, but it is a gap worth closing in whatever comes next.
2. **No concept of "this fee/contract mechanism is excluded by this institution's ruling," only "no row
   exists yet."** Today, an empty `FeeRule` table and a `FeeRule` table where percentage-based late fees
   have been deliberately ruled impermissible look identical. T-007's own research already flagged that the
   line between a permitted flat/real-cost fee and a riba-like percentage fee is genuinely contested and
   institution-dependent — a future design may need to express a deliberate exclusion, not just an absence.
3. **No contract-type layer at all yet** — expected, since today's scope is deposit-side `Savings`/`Current`
   products only, but Q4's finding (even the *set* of permissible contract types varies by jurisdiction and
   school) means the eventual financing-side extension will likely need its own institution-configurable
   "which contract types do we offer" layer sitting above whatever becomes the Murabaha/Musharakah
   equivalent of `ProfitSharingTerms`, not a single fixed enum every adopter inherits unmodified.

None of this means the current shape must be torn up — it means a future task deciding how to fill
`FeeRule`/`ProfitSharingTerms` in for real, or how to extend them for financing contracts, should treat
these three points as known gaps to design around, not surprises to discover later. That decision belongs
to the owner, informed by this research, not to this brief.
