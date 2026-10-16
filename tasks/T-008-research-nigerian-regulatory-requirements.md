# T-008: Research — Nigerian regulatory requirements for a microfinance core system

Output file: `research/nigerian-regulatory-requirements.md`. This is research only — no code, no repo other than `garbati-platform`.

## Why this task exists

GARBATI targets Sharia-compliant microfinance institutions, and the first real-world deployment context is Nigeria (CBN-regulated). IdentityApi and ProductApi design decisions (KYC tiers, record retention, limits) need real regulatory constraints, not assumptions.

## Questions to answer

1. What are the CBN's tiered KYC requirements for microfinance banks (the tiers, what identification is required at each, and the transaction/balance limits per tier)?
2. What customer identity data must be collected and retained, and for how long (relevant to IdentityApi's data model and GARBATI's rule of never logging BVN/NIN)?
3. What record-keeping / audit requirements apply specifically to microfinance banks (as distinct from commercial banks), including any specific to Sharia/non-interest banking windows?
4. Are there CBN rules specific to non-interest (Sharia-compliant) banking in Nigeria that would affect product design (e.g. the CBN's non-interest banking guidelines)?
5. What does the research NOT settle — where would this need confirmation from a compliance professional or direct reading of the primary CBN circular before being treated as final?

## Format

Write `research/nigerian-regulatory-requirements.md` with: a short plain-language summary up top, then the five questions each with a sourced answer (cite the specific CBN circular/guideline where possible), then a "Needs compliance confirmation" section. Flag anything that looks outdated (circulars get superseded) by noting the date of what you found.

Report what you could not find or verify.
