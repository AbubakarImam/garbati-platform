# T-055: Research — read Modern Treasury's full documentation, note everything GARBATI is missing
or deviates from

## Context

Modern Treasury (`docs.moderntreasury.com`) is a real, widely-used "ledger as a service" / payment
operations platform — companies build on top of it rather than building their own ledger/payment-ops
layer from scratch. `research/reconciliation-nibss-integration-patterns.md` (T-054) already cited one
Modern Treasury doc page directly (their own definition of account reconciliation) and found it a useful,
concrete real-world example. This task goes much deeper: read Modern Treasury's documentation
comprehensively, not a single page, and produce an honest gap analysis against GARBATI's actual current
implementation.

**Important framing, state this explicitly in the brief**: Modern Treasury and GARBATI are not the same
kind of product. Modern Treasury is infrastructure other companies integrate with to add payment-ops/
ledger capability to their own product; GARBATI is a vertically-integrated core-banking platform for
Sharia-compliant microfinance institutions themselves. Some things Modern Treasury has and GARBATI
doesn't are genuine gaps worth knowing about; others are different by design because the two products
serve different roles, not oversights. Do not treat every Modern Treasury feature as something GARBATI
"should" have — judge each one on its own merits against what GARBATI is actually trying to be.

## What to do

1. **Actually read the documentation site thoroughly, not just the landing page.** Modern Treasury's docs
   are organized into sections (core concepts, ledgers, payment orders, counterparties, virtual accounts,
   expected payments, reconciliation, returns, and others — the exact current structure should be read
   directly, not assumed from this list). Fetch and read the real content of each major section you can
   reach. If the site's navigation/structure makes some sections unreachable (rate limits, JS-rendered
   navigation a fetch can't follow, paywalled/account-gated content), say so plainly rather than silently
   skipping sections.
2. **For each real Modern Treasury concept found, check it against GARBATI's actual current
   implementation — not against what `ARCHITECTURE.md` claims about itself, against the real code.**
   Read `ARCHITECTURE.md` first for the service map and stated design, then verify specific claims
   directly against the relevant service's actual source where it matters (e.g., if Modern Treasury has a
   concept GARBATI's docs claim to also have, confirm it's actually built, the way this project's own
   "verify, don't trust" discipline has required throughout — see `AGENTS.md`'s "Verify, don't trust"
   section for the standard to hold this research to).
3. **Produce a structured comparison**, organized by Modern Treasury concept area, each one marked as one
   of: **GARBATI has an equivalent** (name it, confirm it's real), **GARBATI is missing this, genuinely
   worth considering** (explain why it would matter for GARBATI specifically, not just "Modern Treasury
   has it"), or **Different by design, not a gap** (explain the real reason the two products differ here
   — audience, scope, regulatory model, etc.).

## Concepts worth specifically checking against (confirm these are still accurate against the real site
before treating them as the structure — Modern Treasury's docs may have changed)

- **Ledgers / Ledger Accounts / Ledger Transactions / Ledger Entries** — their core double-entry model.
  Compare directly against GARBATI's own `LedgerApi` model (`Account`, `Transaction`, `LedgerEntry`,
  chains/legs per decision #45).
- **Ledgerable associations** (attaching a ledger transaction to an arbitrary external business object) —
  does GARBATI have an equivalent extensibility mechanism, or not?
- **Payment Orders** and their lifecycle/status model — compare against how GARBATI's `OrchestratorApi`
  models an in-flight transfer/account-opening operation.
- **Expected Payments** (a declared, awaited incoming payment, matched against what actually arrives) —
  this is directly relevant to T-054's own reconciliation findings; does GARBATI have anything like this
  concept at all today? (Likely not — confirm directly rather than assume.)
- **Counterparties** and **Virtual Accounts** — does GARBATI have an equivalent to either concept?
- **Returns** (a payment that was initiated then reversed/returned by the rail, distinct from GARBATI's
  own reversal flow) — compare against GARBATI's own reversal authority design (decisions #30-#35,
  `Garbati.IdentityApi/docs/DESIGN.md` Section 8).
- **Reconciliation** — T-054 already covered Modern Treasury's own definition; don't re-research that
  specific page, but do check whether Modern Treasury's docs describe a reconciliation *feature/API* (not
  just the concept) that goes beyond what T-054 already found, and compare against T-054's own Q4/Q5
  findings about GARBATI's reconciliation gaps.
- **Webhooks/events model** — does GARBATI have anything equivalent to a documented, versioned webhook/
  event-notification system for external integrators? (Likely not, given GARBATI has no external
  integrator/channel built yet — confirm, and note this honestly as "not yet relevant" rather than "gap.")
- **API versioning/idempotency conventions** — compare directly against GARBATI's own idempotency-key
  discipline (used extensively across `LedgerApi`, `Garbati.IdentityApi`, etc.) — this is one area GARBATI
  likely already has a real, comparable mechanism; confirm and say so plainly rather than only listing gaps.

## Explicitly out of scope

- Do not propose building any of the missing pieces — this is a gap analysis, not a task file or design
  for closing any gap found.
- Do not re-research reconciliation from scratch — build on T-054's existing findings, cite them, don't
  duplicate the research.
- Do not treat this as a referendum on whether GARBATI should become more like Modern Treasury generally —
  GARBATI's actual product direction (Sharia-compliant microfinance core banking) is already decided; this
  research only informs the owner's own judgment about which specific gaps, if any, are worth closing
  later.

## Output

A sourced brief at `research/modern-treasury-gap-analysis.md`, following this project's existing
research-brief conventions (see `research/reconciliation-nibss-integration-patterns.md` for the most
recent example of tone/structure/sourcing discipline, including how to flag what couldn't be read/
verified). End with an honest summary table or list: equivalent / missing-worth-considering /
different-by-design, for every concept actually found on the site — not just the ones named above as a
starting point, since the real documentation may cover more or different ground than this task file
anticipates.
