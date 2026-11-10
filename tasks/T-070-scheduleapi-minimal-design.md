# T-070: ScheduleApi minimal design (D-33)

**Status: done, 10 Nov 2026.** Owner: coordinator + you. Design task — produces a new repo
(`Garbati.ScheduleApi`) and its `docs/DESIGN.md` v1. **No build code was written** — a separate task
goes to a builder once this is reviewed, the same two-step pattern every other service has followed.

## Context

The owner asked for v1 to encompass GARBATI's full Fineract-equivalent standing-instruction
functionality, not a minimal subset — scoping this required a dedicated deep-dive research pass
(T-069, `research/fineract-standing-instructions-detail.md`) beyond T-059's own high-level finding.
Drafting the design against that full feature set surfaced three real, genuinely open questions,
resolved by the owner as D-33:

1. **Retry-on-failure** (`ARCHITECTURE.md` Section 9's own long-standing P-3 question) — decided: no
   auto-retry, matching both real-world convention and Fineract's own real, bug-fixed, shipped behavior.
2. **Joint accounts** — decided: single-holder only for v1, not pre-validated at creation (the existing
   `IdentityApi` authorization chain already denies a joint-account attempt cleanly on its first run).
3. **`AllowPartialTransfer`'s balance source** — decided: a narrow, read-only exception letting
   `ScheduleApi` query `LedgerApi`'s existing balance endpoint directly, mirroring `OpsApi`'s own
   existing funding-endpoint precedent, rather than adding a new `OrchestratorApi` endpoint or
   deferring the feature entirely.

## What was produced

- New repo `Garbati.ScheduleApi`, branch-protected, with the standard governance files (LICENSE,
  NOTICE, SECURITY.md, CODE_OF_CONDUCT.md, CHANGELOG.md, README.md).
- `docs/DESIGN.md` v1: `StandingInstruction`/`StandingInstructionRun` entities modeled against
  Fineract's full real feature set (T-069) — five transfer types, Savings/Loan on each side,
  Fixed/Dues amount, Periodic/AsPerDues recurrence, real in-place editing, soft delete,
  `AllowPartialTransfer` — with everything requiring GARBATI's not-yet-built financing module
  **modeled for schema parity but rejected clearly at creation time**, not silently half-built. A
  plain `BackgroundService` scheduling mechanism (no new dependency). The FINERACT-2914 month-day
  clamping lesson baked in deliberately. 7 endpoints, a decisions log, and explicit open questions.
- Also fixed, while in the repo-creation flow: `SECURITY.md`'s repo list was stale across all 8
  existing repos (missing `Garbati.ProductApi`, which predates this task and was never added) — fixed
  across all 8, plus the new repo, in the same pass.

## Process

1. Coordinator drafted the v1 shape, presented it for review in the same conversation D-33's three
   questions were decided, before creating anything.
2. Owner reviewed and ratified all three real open questions via direct decision.
3. Repo created, `docs/DESIGN.md` written and committed directly (not a separate "present then write"
   round-trip, since the owner's decisions were already captured) — matching the same flow T-027/T-028
   used for their own v1 designs.
4. Next: a separate build task (a new T-number) once the owner reviews the committed design.

## Out of scope for this task

- No code. This task produced a design document and a decision, nothing else.
- No change to `OrchestratorApi`, `IdentityApi`, `LedgerApi`, `ProductApi`, `OpsApi`, or
  `garbati-ops-console` — the design confirms both existing endpoints it needs (`OrchestratorApi`'s
  transfer endpoint, `LedgerApi`'s balance endpoint) already exist with no contract change required.
- No financing-module design or Sharia ruling — everything requiring a loan is explicitly deferred,
  consistent with D-28's own gate.
