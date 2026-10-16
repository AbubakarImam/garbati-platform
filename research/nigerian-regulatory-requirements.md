# Nigerian regulatory requirements for a microfinance core system

Researcher note: this is background research, not legal advice and not a compliance sign-off. CBN circulars get superseded; every date below is the date of the document I actually found, not a guarantee it is still current. **A qualified compliance officer (and, for anything Sharia-specific, a qualified scholar) must confirm every figure and rule here against the live CBN circular before IdentityApi or ProductApi is built on it.**

## Plain-language summary

Nigeria's central bank (CBN) regulates microfinance banks (MFBs), including Sharia-compliant ("non-interest") ones, through a stack of overlapping documents: a general AML/CFT regulation that applies to all CBN-regulated financial institutions, a general microfinance-bank guideline, and a separate non-interest-banking guideline layered on top for Islamic institutions. For GARBATI this means:

1. **Tiered KYC** is real and current (a "Tiered Know-Your-Customer" rule is written into the 2022 AML/CFT regulation itself), but I could only confirm the *concept* from a primary source — the exact current Naira limits per tier come from secondary sources that broadly agree with each other but that I could not check against the original circular.
2. **Identity/transaction record retention is at least 5 years** after the transaction or after the account/relationship ends — this I verified directly from the primary 2022 regulation's text. A secondary source claims microfinance banks specifically must keep books of account for 6 years, which I could not verify against the primary MFB document (the CBN website blocked my access).
3. **Microfinance-specific record-keeping/audit** rules exist in a dedicated MFB guideline and (for non-interest MFBs) an additional non-interest-MFB guideline, but CBN's website returned "403 Forbidden" on both primary PDFs when fetched directly, so most of what I have here is from secondary summaries, not the primary text.
4. **Non-interest (Sharia) banking guidelines** I verified in full from the primary 2011 CBN document — it covers licensing, a mandatory internal Sharia governance committee, profit-sharing-investment-account rules, and audit standards (AAOIFI/IFSB where local standards don't cover something). A newer, non-interest-microfinance-bank-specific guideline (2016/2017) exists but I could only get secondary-source summaries of it.

## Q1: CBN tiered KYC requirements and limits for microfinance banks

**What I found and how confident I am:** Medium confidence on the existence and mechanics of the tiered framework (verified from a primary source); low-to-medium confidence on the exact current Naira figures (multiple independent secondary sources agree, but I could not check them against a primary CBN circular).

The tiered KYC concept was introduced by a CBN circular reportedly dated **18 January 2013** ("Three-Tiered Know-Your-Customer (KYC) Requirements"). I could not retrieve this circular directly — my information on it comes from secondary summaries (law-firm articles, compliance-vendor blogs), which is why confidence on the exact figures below is limited.

The tiered approach is now also written directly into a primary document I did verify: the **CBN (Anti-Money Laundering, Combating the Financing of Terrorism and Countering Proliferation Financing of Weapons of Mass Destruction in Financial Institutions) Regulations, 2022** (S.I. No. 43 of 2022, made 12 May 2022 by the Attorney-General of the Federation, published in the Federal Republic of Nigeria Official Gazette No. 94, 20 May 2022, and circulated by CBN's Financial Policy and Regulation Department on 20 June 2022, Ref. FPR/DIR/PUB/CIR/001/osa). Regulation 26 ("Tiered KYC measures") says:

> "A tiered KYC standard shall be utilized to ensure application of flexible account opening requirements for low-value and medium value accounts which shall be subject to caps and restrictions… As the transactions and value increases, the account opening requirements shall increase progressively with less restrictions as may be specified by the CBN on operations."

Note that this Regulation does **not** itself state the Naira figures — it delegates them to a separate CBN specification (almost certainly the 2013 circular and any amendments to it). This 2022 Regulation also expressly revoked the prior CBN AML/CFT Regulations 2013 (Regulation 71(2)), so the 2013 AML/CFT regulation is no longer current, though the separate 2013 tiered-KYC circular appears to be a different document and (as far as I can tell from secondary sources) still in force.

Secondary sources (multiple independent compliance/legal publications, 2024–2026) consistently report the following **current** figures, which I was not able to verify against a primary circular:

| Tier | Identification required | Daily transaction limit | Cumulative balance limit |
|---|---|---|---|
| Tier 1 | Self-declared identity (name, DOB, address, phone), plus BVN or NIN | ₦30,000 | ₦300,000 |
| Tier 2 | Verified government ID, confirmed address, BVN/NIN | ₦500,000 | ₦500,000 |
| Tier 3 | Full KYC: both BVN and NIN, additional ID, source-of-funds documentation, beneficial-ownership verification for corporates | No limit (one source reports a ₦5,000,000 daily cap for a wallet product, not clearly a bank-account rule) | No limit |

Some secondary sources (apparently reflecting the **original 2013** figures before an unconfirmed later increase) instead give Tier 1 as ₦20,000 daily / ₦200,000 cumulative and Tier 2 as ₦50,000 daily / ₦400,000 cumulative. One article's headline literally says "CBN Raises Limits On Three-Tiered KYC Requirements," which supports the idea that the lower, older figures were later revised upward, but I could not find or date the revising circular.

CBN's 2022 AML/CFT Regulation also sets one concrete, primary-source-verified threshold: Regulation 19(1)(b) requires full Customer Due Diligence for "occasional transactions above the applicable and designated threshold of US$1,000 or its equivalent … or as may be determined by CBN from time to time."

**What this affects:** IdentityApi's KYC-tier data model and ProductApi's/OrchestratorApi's limit enforcement (ARCHITECTURE.md's open question on "aggregate limits," decision P-8, and the limit-definition ownership split between ProductApi and the orchestrator).

## Q2: Customer identity data — what must be collected/retained, and for how long

**Confidence:** High for the retention period (verified directly from primary source text); medium for what GARBATI should infer about logging BVN/NIN (no primary rule found either way).

Primary source: CBN (AML, CFT and CPF) Regulations 2022 (same document as above), Part VI "Maintenance of Records," Regulations 35–36.

- **Regulation 35(1):** "Financial Institutions shall maintain all necessary records of transactions, both domestic and international for **at least five years** after completion of the transaction or such longer period as may be required by the CBN and other competent authorities provided that this requirement shall apply regardless of whether the account or business relationship is on-going or has been terminated."
- **Regulation 35(2):** the record must include: customer and beneficial-owner CDD records (copies of official ID like passports, ID cards, driver's licences), nature and date of the transaction, type and amount of currency, type and identifying number of the account, results of any analysis of complex/unusual transactions, and business correspondence (paper and electronic).
- **Regulation 35(3):** identification data, account files, business correspondence and analysis results must be kept for **at least five years after the account or business relationship ends** (or longer if CBN specifies).
- **Regulation 35(4):** records must be producible to CBN/competent authorities within **48 hours** of request.
- **Regulation 36(1):** records of the supporting evidence and methods used to verify identity must be retained for a minimum of **five years after the account is closed** or the relationship ends.

This is a floor, not a ceiling — CBN can require longer retention, and (see Q3) a secondary source claims microfinance banks specifically must keep "records of all transactions" for a minimum of **six years**, which is longer than the general AML/CFT floor. I could not verify the six-year figure against a primary MFB document (CBN's site returned 403 Forbidden on the relevant PDFs). **If this six-year figure is real, it — not the five-year AML/CFT floor — is the binding number for a Nigerian MFB, and IdentityApi's retention design should use the longer period pending confirmation.**

On BVN access specifically: a secondary summary of the CBN "Revised Regulatory Framework for Bank Verification Number (BVN) Operations and Watch-List for the Nigerian Banking Industry" (dated 2021 per the file name) states that BVN database access is restricted to CBN-licensed financial institutions. I did not fetch the full primary text of this framework, so I cannot confirm its retention or data-handling provisions in detail.

**On GARBATI's own rule of never logging BVN/NIN:** I could not find any CBN text that specifically addresses *application/diagnostic logs* as distinct from the official KYC/CDD record. CBN's retention rules are about keeping the official identification and transaction record for the stated period — they say nothing about whether BVN/NIN may appear in incidental system logs, and if anything imply the opposite concern (records must be *available*, not minimized). GARBATI's stricter internal rule (never log BVN/NIN) looks like good data-minimization/security practice and is not contradicted by anything I found, but it is not something I can point to a specific CBN mandate for. A compliance officer should confirm whether Nigeria's data-protection law (the Nigeria Data Protection Act 2023, outside the CBN scope of this research) imposes an independent requirement here.

**What this affects:** IdentityApi's data retention policy and schema design (how long customer identity records, including document images/references, must be kept after account closure), and the open question in ARCHITECTURE.md about audit/reconciliation data lifetimes.

## Q3: Microfinance-specific record-keeping / audit requirements

**Confidence:** Low-to-medium. CBN's website blocked direct access (403 Forbidden) to every MFB-specific guideline PDF I tried, including a mirrored copy that turned out to be image-only with no extractable text. Everything below is from secondary summaries, not primary text I read myself.

General banking record-keeping (Q2's five-year floor, from the CBN AML/CFT Regulations 2022) applies to microfinance banks because they fall within that Regulation's broad definition of "Financial Institutions" (Regulation 73).

On top of that, secondary sources describe a dedicated MFB guideline — referred to as the **"Reviewed/Revised Guidelines for the Regulation and Supervision of Microfinance Banks (MFBs) in Nigeria"**, with a circular reportedly dated **3 March 2020** covering a guideline document dated **January 2020** — that requires (per secondary summary, not primary text):

- Proper books of account kept in English only, at the principal office and every branch.
- Books of account must contain all information needed to explain the institution's financial condition and transactions.
- Records must be arranged sequentially and made available to CBN and other regulators on demand.
- **Minimum retention period of six years** for all transaction records (longer than the general five-year AML/CFT floor — see Q2's flag on this).

This 2020 document appears to be a revision of an earlier **"Amended Regulatory and Supervisory Guidelines for Microfinance Banks"** (CBN, dated 2013 per the file name) and an even earlier **"Microfinance Policy, Regulatory and Supervisory Framework for Nigeria"** (originally 2005, revised 2011 per secondary references). I could not fetch primary text for any of these three documents.

**Sharia/non-interest-specific record-keeping and audit** is covered by the 2011 non-interest-banking guideline (full primary text verified — see Q4), which requires non-interest institutions, including microfinance banks, to follow Nigerian Accounting Standards Board (NASB) standards and IFRS/IAS, falling back to AAOIFI (Accounting and Auditing Organisation for Islamic Financial Institutions) standards for anything NASB/IFRS don't cover, with local NASB standards prevailing in case of conflict, plus a mandatory internal review/audit mechanism specifically to check compliance with the non-interest principles (not just general financial-statement accuracy).

A non-interest-microfinance-bank-specific guideline — **"Guidelines on the regulation and supervision of non-interest (Islamic) microfinance banks (NIMFBs)"**, reported as released around **April 2017** (the file is hosted under a 2016 CBN folder path, so the drafting/posting date and the "unveiling" date may differ) — reportedly adds microfinance-specific prudential rules on top of the general non-interest guideline:

- Minimum capital by category: Unit NIMFB ₦20 million, State NIMFB ₦100 million, National NIMFB ₦2 billion (secondary source; I could not verify against primary text).
- Must hold at least 5% of deposit liabilities in Sharia-compliant liquidity-management instruments, with a maximum investment ceiling of 10%; non-compliance draws a 1% penalty on the shortfall.
- Capital Adequacy Ratio consistent with whatever CBN prescribes generally (same mechanism as the 2011 general guideline).
- Maximum 20% of shareholders' funds may be invested in fixed assets; a 1% penalty applies to excess.
- Must maintain a reserve fund fed by after-tax profits, to cushion against delinquent-asset erosion.
- Non-compliance with these rules "shall constitute grounds for the revocation of the operating licence" (secondary source's direct quote from the guideline).

I could not retrieve this document's primary text (two separate fetch attempts returned 403 Forbidden), so none of the figures in this paragraph are independently verified by me.

**What this affects:** The record-retention design for both IdentityApi and LedgerApi's audit trail, and ProductApi's Sharia-specific product-rule validation (ARCHITECTURE.md Section 9's "Sharia rules" open question).

## Q4: CBN non-interest (Sharia-compliant) banking guidelines

**Confidence:** High for the core 2011 guideline (I read the full primary document). Low for the more recent Sharia-governance-committee document and the 2016/2017 NIMFB-specific guideline (could not extract text from either).

Primary source, read in full: **"Guidelines for the Regulation and Supervision of Institutions Offering Non-Interest Financial Services in Nigeria"**, CBN Financial Policy and Regulation Department. The file name indicates **20 June 2011**; the document text itself does not carry an explicit date line, but its legal basis — the CBN Act 2007, BOFIA 1991 (as amended), and the "Regulation on the Scope of Banking Activities and Ancillary Matters, No. 3, 2010" — is consistent with a 2011 issue date.

Key points directly relevant to GARBATI:

- **Scope.** Applies to "Institutions Offering Islamic Financial Services" (IIFS), which the guideline explicitly defines to include a "full-fledged Islamic microfinance bank" (category iii of eight listed categories), as well as Islamic branches/windows/subsidiaries of conventional banks.
- **Licensing.** Requires a technical agreement with an established, reputable Islamic bank or financial institution, lasting at least 3 years from commencement of operations. Licenses are regional (6–12 contiguous states within at most 2 geo-political zones, plus FCT) or national.
- **Financing instruments.** IIFS may only use financing modes/instruments that are both Sharia-compliant and CBN-approved — this is a closed list, not an open-ended "anything Sharia scholars say is fine" rule.
- **Fees/commissions.** Permitted per the general "Guide to Bank Charges"; fee/commission income belongs to the bank and is explicitly **not shared with depositors** — this distinguishes it from profit-sharing investment-account returns.
- **Governance.** Every licensed IIFS must have an internal review mechanism plus an **"Advisory Committee of Experts" (ACE)** that reviews and approves every contract, product and service before launch; new products additionally need the CBN's prior written approval. Product literature must name the ACE members who certified the product. The guideline also establishes a CBN-level "Advisory Council of Experts" to advise CBN itself.
  - **Terminology flag:** a separate CBN document, "Guidelines on Shariah Governance for Non-Interest Financial Institutions" (also hosted on cbn.gov.ng under a 2011 path), appears to use different terminology ("Shariah Advisory Committee (SAC)" per a secondary summary I found before attempting the primary fetch). I could not extract readable text from this second document (it is an image-only PDF with no OCR text layer in what I retrieved), so I cannot confirm whether "ACE" and "SAC" are the same body under different names, whether one document supersedes the other, or whether both apply with different scopes. **This needs direct confirmation from a qualified Sharia-governance/compliance professional before ProductApi's design assumes a specific committee structure.**
- **Profit Sharing Investment Accounts (PSIA).** Clients must be told in writing, before the relationship starts, that under profit-sharing/loss-bearing contracts the risk of loss sits with the client — the institution shares in a loss only where its own negligence or misconduct caused it. The guideline permits (not requires) a Profit Equalization Reserve and an Investment Risk Reserve as smoothing/cushioning tools, provided the computation basis is pre-defined and disclosed.
- **Prohibited transaction types** listed explicitly: uncertainty/ambiguity in subject matter or terms, gambling, speculation, unjust enrichment, exploitation/unfair trade practices, and dealing in pork, alcohol, arms & ammunition, or pornography — plus a catch-all for anything else non-compliant with Islamic commercial jurisprudence.
- **Audit/accounting.** NASB local standards and IFRS/IAS apply first; AAOIFI (Accounting and Auditing Organisation for Islamic Financial Institutions) standards apply to anything not covered by those; NASB prevails over international standards in a conflict. Separate from ordinary financial audit, each IIFS needs an internal review/audit mechanism specifically evaluating compliance with the non-interest principles.
- **Prudential.** Capital Adequacy Ratio must track whatever CBN sets for conventional banks generally (no separate, lower bar for Islamic institutions). Funds cannot be invested in interest-bearing instruments or activities.
- **AML/CFT.** Same general screening obligations as conventional institutions (shareholders, customers, counterparties, transactions, products).
- **Branding.** The registered/licensed name cannot include the word "Islamic" without the CBN Governor's specific consent; instead, institutions use a CBN-designed uniform symbol on signage and promotional material.
- **Non-discrimination.** Explicitly prohibits discrimination on grounds of faith or ethnicity in participation as promoter, shareholder, depositor, employee or customer — this applies even though the institution itself operates on Islamic principles.

A later, microfinance-specific layer — the NIMFB guideline discussed in Q3 (reported ~2016/2017, text not independently verified by me) — appears to add capital-tier, liquidity-ratio, fixed-asset-investment, and reserve-fund rules on top of the above general framework, specifically for non-interest microfinance banks.

**What this affects:** ProductApi's Sharia product-rule design directly, which is the named open question in ARCHITECTURE.md Section 9 ("Sharia rules. Which product rules (profit-sharing instead of interest, late-fee-to-charity, no prohibited charges) are enforced by ProductApi and which by the later financing services?"). The PSIA loss-sharing disclosure requirement, the prohibited-transaction list, and the mandatory product-approval-by-committee step are all concrete inputs to that decision. It also affects the account-opening sequence (Section 10) if product binding needs to record which committee approved the product version (ARCHITECTURE.md's P-11 immutable-product-version decision).

## Needs compliance confirmation

This list is deliberately explicit — do not treat any of these as settled:

1. **Exact current Tier 1/2/3 Naira limits (Q1).** I found consistent figures across several 2024–2026 secondary sources (₦30,000/₦300,000, ₦500,000/₦500,000, unlimited) but never located or read the primary CBN circular (reportedly 18 January 2013, possibly amended since) that actually sets them. There is also direct evidence (an older, consistently different set of figures, and a headline referring to a limit increase) that these numbers changed at least once; I don't know when the current set took effect or what circular enacted it.
2. **Whether microfinance-bank record retention is 5 years (general AML/CFT floor, Q2, primary-source-verified) or 6 years (MFB-specific, Q3, secondary-source-only).** This is a real conflict in the numbers I found, and it directly affects IdentityApi's retention design. Use the longer period until a compliance officer confirms which applies.
3. **The entire NIMFB-specific guideline content (Q3)** — capital tiers, liquidity ratio, fixed-asset cap, reserve-fund mechanics — is secondary-source-only. CBN's website returned 403 Forbidden on both direct-fetch attempts for the primary PDF (`cbn.gov.ng/out/2016/fprd/nimfb.pdf`).
4. **Whether "Advisory Committee of Experts (ACE)" and "Shariah Advisory Committee (SAC)" are the same governance body, two names for the same thing across different CBN documents, or genuinely different requirements (Q4).** I could not extract text from the Shariah-governance-specific CBN PDF to resolve this. This must be settled by a qualified Sharia-governance/compliance professional before ProductApi's design assumes a specific committee/approval structure.
5. **Whether the 2013 three-tier-KYC circular is still the operative document**, or whether it has itself been amended or replaced since 2013 (separately from the 2022 AML/CFT Regulations, which only revoked the 2013 *AML/CFT* regulation, not the tiered-KYC circular). I found references to a 2026 CBN circular (BSD/DIR/PUB/LAB/019/002, reportedly dated 10 March 2026) described as introducing automated-AML-solution standards, which post-dates everything else in this brief — I did not fetch or read it, so I cannot say whether it touches the tier limits.
6. **No specific CBN rule against logging BVN/NIN in application/diagnostic logs was found (Q2).** GARBATI's internal rule is reasonable practice but is not something I can cite a CBN mandate for; it may instead (or additionally) be grounded in Nigeria's data-protection law, which is outside what I researched here.
7. **General caution:** every circular above should be re-checked on cbn.gov.ng's current circulars listing before being relied on, since several of the pages I tried to fetch directly returned 403 Forbidden, meaning I cannot confirm the documents I did read (via a web-fetch tool's cache) are still the live, un-superseded versions.

## Sources

- [CBN (Anti-Money Laundering, Combating the Financing of Terrorism and Countering Proliferation Financing of Weapons of Mass Destruction in Financial Institutions) Regulations, 2022 — full primary text read](https://www.stanbicibtcbank.com/static_file/Nigeria/nigeriabank/Personal/Products/Ways%20To%20Bank/Self%20service%20banking/Agency%20Banking/AML%20CIRCULAR%20AND%20REGULATIONS%20MERGED%202022---.pdf) (S.I. No. 43 of 2022, made 12 May 2022, CBN circular Ref FPR/DIR/PUB/CIR/001/osa dated 20 June 2022)
- [Guidelines for the Regulation and Supervision of Institutions Offering Non-Interest Financial Services in Nigeria — full primary text read](https://www.cbn.gov.ng/OUT/2011/CIRCULARS/FPR/NON-INTEREST%20BANKING%20GUIDELINES%20JUNE%2020%202011.PDF) (CBN, dated per filename 20 June 2011)
- [Guidelines on Shariah Governance for Non-Interest Financial Institutions — PDF located, text not extractable (image-only)](https://www.cbn.gov.ng/OUT/2011/CIRCULARS/FPR/FINAL%20GUIDELINES%20ON%20SHARIAH%20GOVERNANCE.PDF)
- [NIMFB guideline PDF — located but access blocked (403 Forbidden) on two attempts](https://www.cbn.gov.ng/out/2016/fprd/nimfb.pdf)
- [CBN Customer Due Diligence Regulations 2023 — located but access blocked (403 Forbidden)](https://www.cbn.gov.ng/Out/2023/CCD/CBN%20Customer%20Due%20diligence%20Reg.%202023-combined.pdf)
- [Reviewed Guidelines for the Regulation and Supervision of Microfinance Banks — mirrored PDF located, not text-extractable](https://oldsite.jee.africa/wp-content/uploads/2020/03/Reviewed-Guidelines-for-the-Regulation-and-Supervision-of-Microfinance-Banks.pdf)
- [Amended Regulatory and Supervisory Guidelines for MFB (2013) — located but access blocked (403 Forbidden)](https://www.cbn.gov.ng/out/2013/ccd/amended%20regulatory%20and%20supervisory%20guidelines%20for%20mfb.pdf)
- [Prudential Guidelines for Microfinance Banks (2019) — located but access blocked (403 Forbidden)](https://www.cbn.gov.ng/out/2019/fprd/mfbs.pdf)
- [dfsobservatory.com — Central Bank of Nigeria Three-Tiered KYC Requirements (secondary summary)](https://dfsobservatory.com/content/central-bank-nigeria-three-tiered-know-your-customer-kyc-requirements)
- [Tekedia — CBN 3-Tier KYC Framework (secondary summary, cites 18 Jan 2013 and older/lower tier figures)](https://www.tekedia.com/a-look-at-the-cbn-3-tier-know-your-customer-kyc-framework-in-nigeria/)
- [youverify.co — CBN KYC/AML Requirements 2026 guide (secondary summary of current tier figures and recent circulars)](https://youverify.co/en/blogs/cbn-kyc-aml-requirements-2026)
- [investdata.com.ng — "Mobile Money: CBN Raises Limits On Three-Tiered KYC Requirements" (secondary, evidence of a limit increase at an unconfirmed date)](https://investdata.com.ng/mobile-money-cbn-raises-limits-three-tiered-kyc-requirements/)
- [SRJ Legal — Impact of CBN's New Tier 1 KYC Rules (secondary summary)](https://srjlegal.com/impact-of-cbns-new-tier-1-kyc-rules-on-financial-inclusion-in-nigeria/)
- [vantrace.io — KYC Requirements Nigeria, CBN Three-Tier Framework (secondary summary)](https://vantrace.io/kyc-requirements-nigeria.html)
- [businessday.ng — What Nigeria's new BVN rules mean for bank customers (secondary, BVN rule changes effective 1 May 2026)](https://businessday.ng/news/article/what-nigerias-new-bvn-rules-mean-for-bank-customers/)
- [nibss-plc.com.ng — What CBN's New BVN Regulations Mean (secondary summary of BVN framework access restrictions)](https://nibss-plc.com.ng/what-cbns-new-bvn-regulation-mean-for-the-nigerian-financial-ecosystem/)
- [brandspurng.com — CBN Unveils Guidelines on Non-interest MFBs (secondary summary of the 2017 NIMFB guideline's liquidity/capital figures)](https://brandspurng.com/2017/04/17/cbn-unveils-guidelines-on-non-interest-mfbs/)
- [omaplex.com.ng — The Legal Framework for Establishing a Microfinance Bank in Nigeria (secondary summary, cites the March 2020 MFB circular)](https://omaplex.com.ng/the-legal-framework-for-establishing-a-microfinance-bank-in-nigeria/)
