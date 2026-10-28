# T-050: Research — how do real Islamic-finance core systems make Sharia product rules *configurable*
per adopting institution, rather than hardcoded by the platform vendor?

## Context

T-007 (`research/sharia-product-rules.md`) researched *what* is prohibited and how fees/late charges are
typically treated under Sharia — a one-time content brief, still needing a qualified scholar's sign-off
before any specific rule is built on it (unresolved, flagged explicitly in that brief).

This task asks a different, structural question. GARBATI is open-source (D-21, Apache 2.0) — any bank or
microfinance institution can adopt it, and different adopting institutions will almost certainly follow
different Sharia supervisory boards, different schools of thought (madhahib), and different national
regulators (CBN's rules aren't the only ones that will ever apply). GARBATI cannot hardcode one
institution's Sharia rulings into the platform and expect every future adopter to be able to use it
as-is — the owner wants to know how real, comparable systems handle this: is there a common pattern for
separating "the platform's mechanism" from "this specific institution's configured rules," the same way
GARBATI already separates mechanism from policy elsewhere (e.g. `KycTierPolicy` — decision #11, "a
configurable, jsonb-backed policy table the platform reads at runtime, not a hardcoded enum/switch").

This directly bears on `Garbati.ProductApi`'s own `FeeRule`/`ProfitSharingTerms` tables (T-009, T-045) —
built with real table shapes but deliberately left **empty**, specifically because no scholar had signed
off on specific values yet. This research is about *whether that table shape is even the right shape* for
a platform meant to be adopted by many institutions with potentially different rulings, not about filling
it in.

## What to research

1. **AAOIFI and IFSB standards as a "common template" layer.** AAOIFI's Shari'ah Standards and
   Financial Accounting Standards, and IFSB's prudential standards, are the closest thing to an
   industry-wide reference most Islamic banks align to even when their own Sharia board's specific
   rulings differ in detail. Characterize how real systems treat these: as a fixed baseline every
   deployment inherits, as a reference an institution's own Sharia board customizes against, or
   something else. Cite the actual standards documents where possible, not just secondary summaries.
2. **How real core-banking vendors with Islamic-finance modules structure configurability.** Look at
   what's publicly documented about Path Solutions (iMAL), Temenos' Islamic banking capability, Mambu's
   Islamic finance support, ICSFS, and any other real, named vendor with public technical or product
   documentation — specifically how they let an adopting bank's own Sharia board configure permissible
   contract types (Murabaha, Musharakah, Ijarah, Qard Hasan, etc.), profit-sharing ratios, and
   late-payment/ta'widh-gharamah treatment, rather than shipping one fixed rule set. If detailed technical
   documentation isn't publicly available for a given vendor (likely, for enterprise software), say so
   plainly rather than guessing at their internals — note what's confirmable from marketing/product pages
   versus what's genuinely unknown.
3. **Does a single "Sharia board approval workflow" pattern exist** — i.e., do real systems model an
   institution's own Sharia Supervisory Board's sign-off on a specific product/rule as a first-class,
   auditable event (something closer to GARBATI's own maker-checker/approval patterns elsewhere), or is
   this typically handled entirely outside the software (a paper/manual governance process feeding into
   whatever the ops team configures)?
4. **What actually varies institution-to-institution versus what's structurally fixed.** For example: do
   profit-sharing ratios and fee amounts vary per institution while the *set of permissible contract
   types* stays essentially fixed (closer to GARBATI's existing `MandateOperation`-style fixed enum
   pattern), or does even the set of contract types itself vary enough across schools of thought/
   jurisdictions that it would need to be configurable too? This distinction directly shapes whether
   `ProductApi` needs one new configurable layer or two.
5. **Multi-tenancy precedent, if any exists outside Islamic finance specifically.** GARBATI already has
   one real internal precedent for "many adopters, many configured rule sets" in a different domain —
   `KycTierPolicy` (configurable tier/evidence requirements read at runtime, not hardcoded) and CBN's own
   tiered-KYC framework being one jurisdiction's specific instantiation of a more general concept. Note
   whether that same shape (a generic mechanism + a jurisdiction/institution-specific config row) is a
   reasonable structural analogy here, or whether Sharia-rule configuration has a meaningfully different
   shape that pattern wouldn't fit.

## Explicitly out of scope

- **Do not propose, invent, or imply any specific fee amount, profit-sharing ratio, or Sharia ruling.**
  That is exactly the thing T-007 already flagged as needing a qualified scholar's sign-off, and this
  task is about the *shape of the configuration mechanism*, not its contents.
- Do not write code, design a database schema, or draft a task file for `ProductApi` — this is a
  research brief only. A structural recommendation (e.g. "a template/baseline layer plus a per-institution
  override layer, mirroring X real system's documented approach") is in scope; a concrete table design is
  not.
- Do not assume AAOIFI/IFSB are legally binding anywhere specific — confirm their actual status (voluntary
  industry standard vs. adopted-into-law in specific jurisdictions) rather than asserting either.

## Output

A sourced brief at `research/sharia-rule-configuration-patterns.md`, following this project's existing
research-brief conventions (see `research/sharia-product-rules.md` and `research/staff-ops-identity-patterns.md`
for tone/structure — cite real, named, checkable sources; state plainly where something couldn't be
confirmed rather than filling the gap with a guess). End with an explicit, honest recommendation on
whether GARBATI's current `FeeRule`/`ProfitSharingTerms` table shape looks compatible with what real
systems do, or whether it would need rethinking before any institution-specific configuration could be
layered on top — flagged as input to a future owner decision, not a decision itself.
