# T-011: Research — industry-standard patterns behind P-1 to P-6

Output file: `research/architecture-patterns-p1-p6.md`. This is research only — no code, no repo other than `garbati-platform`. It exists to help the owner (a C#/.NET learner, not yet experienced in distributed-systems design) decide whether to ratify `ARCHITECTURE.md` Section 8's P-1 through P-6 as Decided, by checking them against established industry practice — not to make the decision itself.

## Context

`ARCHITECTURE.md` Section 8 proposes six platform-level boundaries for GARBATI, a core banking system:
- **P-1**: a separate `OrchestratorApi` owns the ordered steps of a money-moving operation (rather than channels calling each service themselves, or the ledger calling out to other services).
- **P-2**: a separate `ProductApi` owns product rules, fees, tax and limit definitions (rather than folding these into `IdentityApi` or the orchestrator).
- **P-3**: a separate `ScheduleApi` owns recurring instructions (standing orders), submitting ordinary requests to the orchestrator rather than moving money itself.
- **P-4**: `ProductApi` only computes amounts (fees, tax); only the orchestrator, calling `LedgerApi`, actually posts money. Decision and effect are separated.
- **P-5**: no service reads another service's database directly, no shared database, no cross-service foreign keys — services only ever talk to each other through APIs.
- **P-6**: design all of the above as separate services in code, but deploy only `LedgerApi` and `IdentityApi` as separate live services at first; `ProductApi` and `OrchestratorApi` can start as separate projects inside one shared deployable.

Read `ARCHITECTURE.md` Sections 1–8 in full for the complete reasoning already given for each.

## Questions to answer

1. **P-1**: Is a dedicated orchestrator that owns step ordering across services a recognized pattern for multi-step business transactions in a distributed system (e.g. the Saga pattern, orchestration-based sagas specifically vs. choreography-based sagas)? What are the named trade-offs industry sources give for orchestration vs. choreography, and does a core-banking/payments context tend to favor one?
2. **P-2 / P-4**: Is separating "who decides the rules/price" from "who executes the money movement" a recognized pattern (e.g. command/policy separation, or how other core banking systems split a product-catalog/pricing-engine concern from a ledger/posting engine)? Are there known core-banking or payments-industry examples (open-source or documented architectures) that draw this boundary the same way, differently, or not at all?
3. **P-3**: Is a separate scheduling/recurring-instruction service (separate from the component that actually executes the action) standard, and how do existing systems (e.g. standing orders/direct debits in banking, or general job-scheduling-triggers-a-workflow patterns) usually structure retry and run-history for this?
4. **P-5**: "Database-per-service, no shared database, API calls only" — is this standard microservices guidance, and what are the most commonly cited costs (e.g. harder cross-service queries/reporting, eventual consistency, more network calls) versus benefits (independent schema evolution, enforced ownership) in reputable sources? Does anything in the banking/financial-services context make this more or less appropriate than in a generic web app?
5. **P-6**: Is "design with clean service boundaries in code, but deploy fewer physical services at first and split later" a recognized pattern (e.g. "modular monolith," "monolith first," or similar named approaches)? What do credible sources say about the actual cost/difficulty of splitting a modular monolith into separate deployments later, so the owner can judge whether this is a safe starting point or a trap?

## Format

Write `research/architecture-patterns-p1-p6.md` with: a short plain-language summary up top (the owner is a C#/.NET learner — avoid unexplained jargon, define terms like "saga," "choreography," "modular monolith" in plain words on first use), then the five questions above each with a sourced answer citing what you read (prefer well-known, credible sources: established architecture texts/blogs, cloud vendor architecture guidance, microservices.io, martinfowler.com, or similar — not random blogspam), and for each of P-1 through P-6 a one-line verdict: "matches common practice," "a reasonable variant," or "unusual / worth double-checking," with a one-sentence reason. End with a "What this does not settle" section — this is industry-pattern research, not a decision; it does not replace the owner's own judgment about GARBATI's specific situation (solo developer, budget, demo-for-recruiters goal).

Report what you could not find or verify.
