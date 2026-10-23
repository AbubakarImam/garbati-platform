# T-043: Research — does Nigerian law actually require encrypting BVN/NIN at rest?

## Context

GARBATI's `Garbati.IdentityApi` deliberately has no BVN/NIN column yet (`docs/DESIGN.md` Appendix B,
`#TBD-4`) — confirmed directly in code, nothing is stored today. This blocks real Tier 2/3 KYC
verification. The owner is deciding G3 (from `research/oss-core-banking-readiness-gaps.md`) and asked a
direct, specific question this research needs to answer as concretely as possible: **is there an actual
legal requirement to encrypt BVN/NIN at rest in Nigeria, or is it a "best practice" with no binding rule
behind it?**

This matters because the only thing in the repo suggesting a requirement exists is T-037's own secondary-
source summary (`research/oss-core-banking-readiness-gaps.md` G3): "Nigeria's Data Protection Act 2023
requires 'appropriate security measures proportionate to the sensitivity of data,' explicitly including
encryption at rest and in transit... (secondary-source summary, not primary NDPA text)." That same brief's
"what I could not verify" section already flagged this exact gap: "Whether Nigeria's Data Protection Act
2023 imposes a specific, technical encryption-at-rest standard... versus a general 'appropriate security
measures' principle." Nobody has actually read the primary NDPA text on this specific point yet.

## What to research

**1. Read the primary Nigeria Data Protection Act 2023 text directly** (not a law-firm summary) on data
security obligations for data controllers/processors. The Act and its implementing regulation (the Nigeria
Data Protection Regulation / NDPR, and the newer NDPA 2023 and any Nigeria Data Protection Commission
(NDPC) guidance issued under it) are the primary sources to find and read directly:
   - Does the Act/its regulations name encryption specifically as a required control for sensitive
     personal data (which BVN/NIN would qualify as), or only a general "appropriate technical and
     organizational measures" standard that leaves the specific control to the data controller's own risk
     assessment?
   - Is there a defined category like "sensitive personal data" in the Act, and does BVN/NIN fall into it
     by the Act's own definition (not just common sense)?
   - Does the NDPC (Nigeria's data protection regulator) have published guidance, codes of practice, or
     enforcement history specifically naming encryption-at-rest as expected/required for identity numbers
     like BVN/NIN?

**2. Check CBN's own BVN-specific framework separately from NDPA.** `research/nigerian-regulatory-
requirements.md` Q2 already found a secondary summary of the CBN "Revised Regulatory Framework for Bank
Verification Number (BVN) Operations and Watch-List for the Nigerian Banking Industry" (2021) but did not
read its primary text. Try again specifically for:
   - Does this framework itself impose a security/encryption requirement on institutions storing BVN,
     separate from and more specific than NDPA's general obligations?
   - Any CBN circular specifically about NIN handling (NIN is a National Identity Management Commission
     identifier, not originally a CBN one — check whether CBN has adopted or cross-referenced NIMC's own
     data-protection rules for NIN specifically).

**3. Real-world enforcement/practice check, treated as evidence, not as the answer.** The owner mentioned
their own workplace stores BVN/NIN unencrypted. Separately and honestly: has the NDPC actually taken
enforcement action against any Nigerian financial institution specifically for inadequate encryption of
identity data? Real enforcement history is useful evidence of how strictly this is actually interpreted in
practice, but it does **not** by itself prove something is legal just because it's common or
unenforced — state this distinction plainly rather than conflate "common practice" with "compliant."

**4. If no clear, specific encryption mandate is found**, say so plainly and explain what "appropriate
security measures proportionate to sensitivity" would most defensibly mean in practice for a system like
GARBATI's — not a legal opinion, but an honest technical/industry-standard read (e.g., does the absence of
a specific mandate mean storing BVN/NIN in plaintext in a production database would still be considered
unreasonable by any professional security standard, regardless of the exact legal wording?).

## Output

Write the brief to `research/ndpa-encryption-requirement.md`, matching this repo's existing research-brief
style (cited sources, explicit "what I could not verify" section, confidence levels per claim — distinguish
clearly between primary-text-confirmed and secondary-source-only throughout, the same discipline
`research/nigerian-regulatory-requirements.md` already used). State your confidence level explicitly for
the central question: is there a specific, binding encryption-at-rest requirement, or only a general
principle? Do not round up a general principle into a specific mandate, and do not round down a specific
mandate into "just best practice" either — report exactly what the primary text says, however unsatisfying
that answer turns out to be.

Read `research/oss-core-banking-readiness-gaps.md` (G3) and `research/nigerian-regulatory-requirements.md`
(Q2) first — don't repeat their already-documented CBN-PDF access failures without at least trying a
different angle (NDPC's own website structure may differ from CBN's; NDPA text itself may be more
accessible than CBN circulars have been).

## Out of scope

- Deciding whether GARBATI actually encrypts BVN/NIN, what pattern it uses, or whether it stores BVN/NIN
  at all versus verify-and-discard — that remains the owner's decision, informed by this research.
- Re-researching CBN's general AML/CFT retention-period rules — `research/nigerian-regulatory-
  requirements.md` Q2/Q3 already covers that; this task is specifically about the encryption question.
- Any code change.
