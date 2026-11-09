# T-068: LedgerApi multi-entry (N:M) posting build (decision #53, T-067)

Owner: builder. Implements the design ratified in T-067 and written into `LedgerApi/docs/DESIGN.md`
decision #53 (`main` @ `2c98574`). Read that decision in full before starting — it is the spec. This
task does not reopen the design; it builds exactly what #53 describes. If something turns out to be
wrong or ambiguous once you're in the real code, stop and flag it rather than silently deciding — this
is a production ledger service, and decision #45 (the mechanism being extended) is some of its most
heavily-tested code.

## What to build

1. **`ChainKind` enum** (`Entities/ChainKind.cs`): `Payment`, `Disbursement`, `Collection`.
2. **Two new `LegPurpose` values** (`Entities/LegPurpose.cs`): `Disbursement`, `Collection`, added to
   the existing `Principal`/`Fee`/`Tax` enum. `ChainRequestValidator.ParsePurpose` already enumerates
   `LegPurpose` names dynamically — confirm it picks up the two new values with no change needed there,
   rather than assuming it does.
3. **`TransactionChain` entity**: add `ChainKind ChainKind { get; set; }`.
4. **Migration**: add a `chain_kind` column to `transaction_chains` — **`NOT NULL`, `DEFAULT 'Payment'`**
   (unlike decisions #49-#52's nullable metadata columns, every chain conceptually has a kind; there is
   no "not yet categorized" state here). Every pre-existing row becomes `Payment`, preserving their
   exact current behavior.
5. **`ChainRequest` contract** (`Contracts/Requests/ChainRequest.cs`): add `ChainKind ChainKind =
   ChainKind.Payment` (optional, defaults to `Payment` — every existing caller's request is
   unaffected). `ChainLegRequest` is unchanged — it already carries a free-text `Purpose` string per
   leg; no new field needed there.
6. **`ChainRequestValidator`** (`Validation/ChainRequestValidator.cs`) — this is the bulk of the real
   work. Today it hardcodes `MaxLegs = 10` and two `Payment`-specific rules ("leg 1 must be the
   Principal," "only one Principal leg") unconditionally. Make these kind-dependent:
   - **Leg cap**: `Payment` → 10 (unchanged, do not touch this constant's existing behavior or tests).
     `Disbursement`/`Collection` → 1,000 (decision #53's own explicit number).
   - **`Payment`**: keep today's exact rules unchanged — leg 1 must be `Principal`, at most one
     `Principal` leg, every leg's purpose must be `Principal`/`Fee`/`Tax` (reject `Disbursement`/
     `Collection` purposes on a `Payment` chain).
   - **`Disbursement`**: every leg's purpose must be `Disbursement` (reject any other purpose,
     including `Principal`/`Fee`/`Tax`). Every leg must share the *same* `DebitAccountNumber`. Every
     leg's `CreditAccountNumber` must be **distinct** across the whole chain (decision #53's explicit
     owner call — reject a duplicate recipient as a validation error, not silently allow it or merge
     amounts). No "leg 1 is Principal" rule — there is no principal leg in this kind.
   - **`Collection`**: mirror image — every leg's purpose must be `Collection`. Every leg must share the
     same `CreditAccountNumber`. Every leg's `DebitAccountNumber` must be **distinct**.
   - Write this as a genuine kind-dependent dispatch (e.g., a switch on `request.ChainKind` selecting
     which rule set runs), not a pile of `if (kind == X)` conditionals scattered through the existing
     `Payment`-only logic — keep the three rule sets legible and independently testable.
7. **`ChainService`** (`Services/Implementations/ChainService.cs`):
   - Confirm, by actually reading the code (not assuming decision #53's own claim), that the existing
     locking logic ("lock every account debited by any leg, once, ascending order") and the existing
     balance-check logic ("for each account that pays any leg, the total it pays across the chain must
     not exceed its balance") genuinely require zero changes for `Disbursement`/`Collection` — decision
     #53 asserts this based on reading the design, not the implementation; verify it holds in the real
     code once you're in it, and flag it immediately if it doesn't.
   - The existing rule "`Fee`/`Tax` legs must credit a System account" (around line 303, checking
     `credit.AccountClass != AccountClass.System`) must not fire for `Disbursement`/`Collection` legs —
     confirm the condition is still scoped to `LegPurpose.Fee or LegPurpose.Tax` specifically, not
     accidentally broadened.
   - **`HashLegs`** (the request-hash computation) must include `ChainKind` in the hashed payload,
     alongside the existing ordered-legs fields — decision #53 is explicit that changing `Payment` to
     `Disbursement` for the same leg list is a materially different request, unlike the four metadata
     fields (decisions #49-#52) that are correctly excluded. Write a regression test: same idempotency
     key, same legs, different `ChainKind` on retry → **409** (a different request reusing the same
     key, same as decision #42/#44's existing "same key, different hash" behavior) — not a silent
     replay, and not a second chain created.
8. **Response contract** (`Contracts/Responses/ChainResponse.cs`): add `ChainKind`.
9. **Reversal**: no new code should be needed — decision #53 says chain reversal (12.8) already
   applies unchanged. Confirm this with a real test (reverse a `Disbursement` chain, confirm every leg
   reverses atomically, same as an existing `Payment` chain reversal) rather than assuming it from the
   design text alone.

## What this task does not include

- No change to the existing `Payment` chain kind's behavior, tests, or the single-transfer endpoint —
  this must be purely additive. Run the full existing chain test suite before and after; it must pass
  unchanged.
- No `Fee`/`Tax` legs inside a `Disbursement`/`Collection` chain in v1 — the validator rejects them
  (see item 6 above), this is not a gap to fill.
- No change to `Garbati.OrchestratorApi`, `ProductApi`, or any other service. No payroll/disbursement
  endpoint anywhere upstream — this ships as a capability with no real caller, same as decisions
  #49-#52. Do not invent a caller to "demonstrate" it.
- No chunked/batched mechanism for a batch larger than 1,000 legs — named as a real, acknowledged
  future limit in decision #53, not this task's job to solve.
- No group-lending-specific code or schema — the group-lending use case (`research/group-lending-joint-liability-patterns.md`,
  T-066) stays gated on D-28's Sharia scholar sign-off, unrelated to this task's own scope.

## Acceptance

- Full existing test suite green and unchanged in count for every existing `Payment`-kind test — this
  is additive, not a refactor of tested behavior.
- New tests covering: the leg-cap difference per kind (reject leg 1,001 on `Disbursement`/`Collection`,
  reject leg 11 on `Payment` — unchanged, confirm this old test still exists and still passes); the
  `Principal`-first rule applying only to `Payment`; the distinct-recipient/distinct-payer rule for
  `Disbursement`/`Collection` respectively (reject a duplicate); the shared-debit/shared-credit
  requirement; the hash-includes-`ChainKind` regression test (item 7 above) — proven the same way this
  project always proves a regression test (break it on purpose, confirm it fails with the exact
  expected message, then fix it and confirm it passes); a full `Disbursement` chain posting and
  balance-check end to end (one funding account, several recipients, confirm each recipient's balance
  and the funding account's single aggregate debit); the same for `Collection`; a `Disbursement` chain
  reversal.
- `DESIGN.md` decision #53 updated if anything above had to change from what's written there, with the
  deviation and why, same discipline as every other decision in that log.
- No production-data migration concern — the new column is `NOT NULL DEFAULT 'Payment'`, which Postgres
  can backfill as part of the `ADD COLUMN` itself with no separate data-mutation step and no risk to
  existing rows' meaning (they genuinely were, and remain, `Payment` chains). Ordinary review is enough,
  no special sign-off gate, unlike T-061's chart-of-accounts migration.
