# T-007: Research — Sharia-compliant microfinance product rules

Output file: `research/sharia-product-rules.md`. This is research only — no code, no repo other than `garbati-platform`.

## Why this task exists

GARBATI's ProductApi (not started yet) will enforce product rules: profit-sharing instead of interest, how late payments are handled, which charges are allowed. Before anyone designs ProductApi, we need a plain-language brief on what Sharia-compliant microfinance actually requires, so the design questions in T-009 are answered with real constraints instead of guesses.

## Questions to answer

1. What is riba (interest) in this context, and what structures microfinance institutions use instead (e.g. Murabaha, Musharaka, Qard Hasan) for the kinds of products GARBATI is likely to need first (savings accounts, short-term financing)?
2. How are late payments / defaults handled under Sharia rules? Specifically: is a late fee permitted at all, and if so, where does the money go (the open question in `ARCHITECTURE.md` Section 9 flags "late-fee-to-charity" — confirm or correct this)?
3. What charges or fee structures are generally considered prohibited vs permitted (e.g. is a flat service fee different from a percentage-based fee tied to time elapsed)?
4. Are there existing open-source or published core banking / Islamic finance standards (e.g. AAOIFI standards) worth citing directly?
5. What does the research NOT settle — i.e. where does this need an actual scholar or Sharia board to confirm before it is trusted for production rules?

## Format

Write `research/sharia-product-rules.md` with: a short summary up top in plain words, then the five questions above each with a sourced answer (cite what you read), then a clearly labeled "Needs scholar confirmation" section listing every claim that should not be treated as final without an Islamic finance scholar's sign-off. Do not present any answer as settled doctrine — this document informs design questions, it does not make product decisions.

Report what you could not find or verify.
