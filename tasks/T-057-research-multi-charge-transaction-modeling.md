# T-057: Research — how do real banking platforms model stacked, compounding transaction charges
(stamp duty, VAT-on-fee, FCY-specific charges, cheque charges), and how would GARBATI improve on what
it has?

## Context

The owner raised a real, concrete scenario GARBATI's current architecture cannot yet represent. A single
real withdrawal/transfer in Nigerian banking practice can carry several distinct, independently-sourced
charges that must all post together as one atomic event: a bank service fee, VAT calculated as a
percentage *of that fee* (a charge on a charge, not on the principal), and — for local-currency electronic
transfers specifically — stamp duty, a flat government levy distinct from both. Foreign-currency (FCY)
transactions commonly carry a different charge schedule entirely from local-currency (LCY) ones. The same
general problem (multiple, compounding, currency-dependent charges on one transaction) applies separately
to cheque-related transactions (processing fees, cheque-book issuance, stop-payment, bounced-cheque
charges), which GARBATI has no representation of at all today.

**Checked directly against the real code before this task was written — do not re-verify this, it's
already established fact:**
- `LedgerApi`'s `LegPurpose` enum (decision #45) is a hard-closed list of exactly three values:
  `Principal`, `Fee`, `Tax`. There is no distinct purpose code for a government levy like stamp duty —
  it would currently have to be folded into `Fee` or `Tax`, losing its own identity.
- `Garbati.ProductApi`'s `FeeRule`/`FeeQuoteService` computes **one** flat-or-percentage charge per
  `(ProductCode, OperationType)` call. There is no mechanism for a rule to reference another rule as its
  base (so VAT cannot currently be expressed as "7.5% of the Bank Fee row"), no currency-conditional
  branching in the rule shape, and no threshold-gated logic (e.g. "only applies when the amount is at
  least ₦X").
- None of this is wired into a live transaction today: `OrchestratorApi`'s `TransferService` never calls
  `ProductApi`'s fee-quote endpoint at all; `FeeRule` is deliberately left empty pending a qualified
  Islamic finance scholar's sign-off (`research/sharia-product-rules.md`). Every real transfer today posts
  a Principal leg only.
- GARBATI has zero representation anywhere of a cheque instrument or cheque-specific charges (confirmed by
  direct search across every service repo).

This is forward-looking research, the same way T-054/T-055/T-056 researched real-world practice before
anything was designed — not a request to build anything yet.

## What to research

1. **How real core-banking/fintech platforms represent a transaction's own "charge breakdown" as a first-
   class, structured concept** — not just a single fee amount, but an itemized list of distinct charge
   lines (service fee, VAT, stamp duty, etc.) that all belong to one logical transaction. Is this typically
   its own object/table (a "charge schedule" or "fee breakdown" entity per transaction), or is each charge
   just another ledger posting with a type tag, the way GARBATI's chain/leg model already works? Cite real,
   named platforms, vendor documentation, or standards (ISO 20022 charge-related fields are a real,
   citable source worth checking specifically) rather than generic summaries.
2. **How tax-on-a-fee (a charge calculated as a percentage of another charge, not of the principal) is
   actually modeled.** Does a real system represent this as a rule with an explicit "base" reference
   (charge X is a percentage of charge Y), calculated at quote time, or does it get computed and frozen at
   the moment each prior charge is finalized? What happens if the base charge itself is waived or reduced —
   does the dependent tax recompute, or is this treated as two independent, pre-calculated figures in
   practice, each individually configured?
3. **Nigerian stamp duty specifically, read from a real source, not assumed.** What is the actual current
   rule (amount, the threshold it applies above, which transaction types it applies to, who remits it and
   to whom) per the Stamp Duties Act / Finance Act framework and any CBN/NIBSS implementation guidance
   found and readable? State plainly if the primary source can't be read (the same `cbn.gov.ng`
   access pattern has recurred in T-037/T-041/T-043/T-054/T-056 — expect it here too and say so rather than
   guess a figure from memory or an unverified secondary source).
4. **FCY-specific charge differentiation** — how do real platforms structure a charge schedule that varies
   by currency (different fee amounts, different applicable charge types — e.g. stamp duty being an LCY-
   specific concept, not applicable to FCY transfers at all) without duplicating the entire fee-rule table
   per currency? Is this commonly a currency-scoped rule set, a currency field on each rule, or something
   else?
5. **Cheque-specific charges** — how are these typically modeled (processing fee at presentation, cheque-
   book issuance fee, stop-payment fee, bounced/returned-cheque fee with its own distinct reason-coded
   charge)? Separately, and honestly: is cheque usage actually significant in real Nigerian microfinance
   banking today, or is it a declining/minor channel relative to electronic transfers? This matters for how
   much weight this part of the research should carry — check rather than assume either way.
6. **How this maps onto GARBATI's own existing architecture — a structural comparison, not a design,**
   matching the discipline T-054/T-055/T-056 already established. Specifically: would a real multi-charge-
   type transaction need the `LegPurpose` enum to grow beyond `Principal`/`Fee`/`Tax` (and if so, is that a
   closed, enumerable list of real-world charge types, or does it need to become an open/extensible
   concept), and would `FeeRule`'s shape need a way to express "this rule's amount is derived from another
   rule's computed result" (the tax-on-a-fee compounding case) that it has no way to express today? Be
   honest about what's a bounded extension of what already exists versus what needs something genuinely
   new, the same way T-056's Q5 did for till/vault.

## Explicitly out of scope

- Do not design a charge-modeling feature, propose a schema, or write a task file for building it — this
  is research only.
- Do not invent or assert a specific stamp duty amount, VAT rate, or any other specific figure without a
  real, cited source — if a figure can't be confirmed from a source actually read, say so plainly rather
  than stating a remembered or assumed number as fact. This is exactly the kind of regulatory-adjacent
  claim this project's own discipline (T-037/T-041/T-043) requires sourcing carefully.
- Do not assume this is urgent or propose a timeline — `FeeRule` remains deliberately unseeded pending
  Sharia scholar sign-off regardless of what this research finds; nothing here changes that gate.

## Output

A sourced brief at `research/multi-charge-transaction-modeling.md`, following this project's existing
research-brief conventions (see `research/teller-till-vault-cash-operations.md` for the most recent
example of tone, sourcing discipline, and honest confidence-calibration/uncertainty-flagging — match that
standard). End with an honest assessment of whether GARBATI's existing `LegPurpose`/`FeeRule` mechanism is
structurally suited to extend to cover compounding, currency-differentiated, multi-type charges, or
whether real practice needs something those mechanisms don't have today — framed as input to a future
owner decision, not a decision itself.
