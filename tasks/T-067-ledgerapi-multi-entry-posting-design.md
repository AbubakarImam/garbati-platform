# T-067: LedgerApi multi-entry (N:M) posting design (D-32)

Owner: coordinator + you. Design task — produces a decisions-log entry in `LedgerApi/docs/DESIGN.md`.
**No code gets written from this task directly** — once ratified, a separate build task goes to a
builder, the same two-step pattern T-060→T-061 and T-063→T-064/T-065 just used.

## Context

`research/real-corebanking-payload-gap-analysis.md` (T-058, Gap 3) found multi-entry (N:M) posting —
one debit balanced against many credits, or many debits against one credit — real and named in both
Apache Fineract ("compound journal entry," parallel `credits`/`debits` arrays) and Oracle FLEXCUBE
("Single Debit Multiple Credit," batch booking). Both platforms' own real-world justification ties
this to exactly two scenarios, neither an ordinary retail transfer: payroll-style disbursement
(one funding account, many recipients) and bulk collection (many payers, one biller/clearing
account). The owner decided (D-32) this is a real future need for GARBATI — directly relevant to a
group-loan disbursement (one funding account, N group members), input the group-lending research
(T-066) already flagged as a future structural seam, though that specific caller remains gated on
D-28's Sharia scholar sign-off and is explicitly **not** part of this task's scope.

**The mechanism choice, already decided (D-32): extend LedgerApi's existing chain model (decision
#45), not build a new, literal N:M compound-journal-entry primitive.** A chain is already "N atomic
2-entry legs, one database transaction, locked and balance-checked together" — a payroll
disbursement to 50 employees is structurally just 50 legs, each an ordinary [funding account debit →
one recipient credit] transfer, bundled atomically. This achieves the identical business outcome to
Fineract/FLEXCUBE's literal compound entry without touching the "exactly one debit and one credit
per transaction row" invariant decisions #1, #3, and #45 have each independently reaffirmed — a
genuine architectural reversal was the alternative, and is explicitly rejected here, not left open
for this design to reconsider.

**Both directions are in scope (D-32)**: one-to-many (disbursement) and many-to-one (collection).

## What's already in place, confirmed by re-reading decision #45 directly — the good news

Almost all of the existing chain mechanism's locking and balance-check logic already generalizes
correctly to N:M, with no new logic needed:

- **Locking** (12.5): "lock every account debited by any leg... once for the whole chain" already
  handles a Collection batch's N distinct debtors correctly — it was never written assuming exactly
  one debtor.
- **Balance check** (12.4 rule 6): "for each customer account that pays any leg, the total of
  everything it pays across the chain must not exceed its balance" already handles a Disbursement
  batch's single funding account paying the sum of N legs correctly — it was never written assuming
  exactly one leg per payer.

**What genuinely needs to change** — three things, all narrow and additive:

1. **The leg cap.** Decision #45 caps a chain at 10 legs — right for Principal+Fee+Tax, far too low
   for 50+ payroll recipients. **Open number, needs your explicit call**: proposed **1,000** legs for
   this new kind of chain specifically (the existing Principal+Fee+Tax chain's 10-leg cap is
   untouched) — high enough for a real small-institution payroll run or a sizable group-loan
   disbursement, bounded to keep one atomic database transaction's lock footprint and request-hash
   computation reasonable. A genuinely larger batch (thousands of legs) would need a different,
   chunked mechanism not designed here — named as a real, acknowledged limit, not silently assumed
   away.
2. **Decision #45's own rule 2** ("leg 1 is the only `Principal` leg") **structurally conflicts with
   N legs that are all, equally, the "main" leg of a disbursement/collection** — there is no single
   principal leg in a payroll run; every leg is equally a real payment. Proposed: a new, distinct
   chain *kind* — `Disbursement`/`Collection` — with its own validation rules, not a relaxation of the
   existing `Payment` kind's rules. Existing `Payment` chains (Principal+Fee+Tax, decision #45 as
   built) are completely untouched.
3. **Two new `LegPurpose` values**, extending the closed enum (decision #7: `Principal`/`Fee`/`Tax`
   only) — `Disbursement` and `Collection`. Each leg in a `Disbursement` chain is tagged
   `Disbursement`; each leg in a `Collection` chain is tagged `Collection`. No `Fee`/`Tax` legs inside
   a `Disbursement`/`Collection` chain in v1 — per-recipient fee computation is real complexity
   (`ProductApi`'s `FeeRule` is unseeded anyway, pending Sharia scholar sign-off) explicitly deferred,
   not designed here.

## What this design must resolve (draft proposal below, for review and revision, not final)

1. **New `ChainKind` discriminator** on `transaction_chains` (or equivalent), closed enum:
   `Payment` (today's existing Principal+Fee+Tax shape, decision #45, unchanged), `Disbursement`,
   `Collection`.
2. **`Disbursement` validation**: every leg must share the *same* debit account (the one funding
   account); credit accounts must all be distinct (no paying the same recipient twice in one batch —
   open question, flagged below, for whether that's actually a real constraint to enforce or an
   over-restriction). 1 to the new leg cap (item 1 above) legs. Every leg tagged `Disbursement`. No
   `Principal`-first rule.
3. **`Collection` validation**: every leg must share the *same* credit account (the one
   biller/clearing account); debit accounts must all be distinct. Same leg-count bound. Every leg
   tagged `Collection`.
4. **Reversal**: same all-or-nothing semantics as an existing chain reversal (decision #45, 12.8) —
   one reversing leg per original leg, atomic, no partial reversal. No new design needed here; this
   falls out of reusing the chain mechanism, not a new decision.
5. **Idempotency / request hash**: same reasoning as every prior metadata-field decision this
   project has made (decision #44's narration exclusion; decisions #49-#52's four-field exclusion) —
   the hash covers the ordered legs' debit/credit/amount/purpose, nothing else. No change to how
   chain hashing already works (12.7); `ChainKind` itself should be included in the hash (changing
   `Payment` to `Disbursement` for the same leg list *is* a materially different request, unlike a
   caller-supplied metadata field).
6. **Endpoint**: reuse the existing `POST /api/chains` (and its `GET`/reversal siblings) with the new
   `ChainKind` field, rather than a new endpoint or a new table — this is the same underlying
   mechanism (ordered legs, one database transaction, decision #45's `ChainService`/`LegPoster`/
   `LegReverser`), not a new one.

## Open question needing your explicit call, beyond the leg-cap number above

**Must every credit account in a `Disbursement` batch (or every debit account in a `Collection`
batch) be distinct?** Proposed default: yes — paying the same recipient twice in one disbursement
batch is more likely a caller bug than a real intended duplicate payment, and rejecting it early is
cheap. But a real bulk-collection system might legitimately need to collect two separate amounts from
the same payer in one batch (e.g., two different bills) — if that's a real GARBATI use case, the
"distinct accounts" rule should be dropped for `Collection` specifically. Flagged for your review, not
decided here.

## What this design explicitly does not change

- **Decision #1 (derived balance), #7 (row locking), #34 ("LedgerApi calls nobody"), P-5 (no shared
  database).** Unaffected — this reuses existing, already-tested locking/balance/posting machinery.
- **The existing `Payment` chain kind (decision #45, Principal+Fee+Tax)** — completely untouched,
  including its own 10-leg cap, its own `Principal`-first rule, and its own existing tests.
- **No caller wired up.** Like decisions #49-#52 (value-date/branch/channel/decision-id), this ships
  as a capability with no real caller yet — `OrchestratorApi` does not gain a payroll/disbursement
  endpoint in this task, and the group-lending use case (T-066) stays gated on D-28's scholar
  sign-off. This is purely LedgerApi's own primitive, shape now, content later, same discipline every
  prior foundational LedgerApi decision this session has used.
- **No per-leg fee/tax inside a `Disbursement`/`Collection` chain** — explicitly deferred, not
  designed here (see item 3 above).

## Process

1. Coordinator drafts the full design (this file, expanded into a real `LedgerApi/docs/DESIGN.md`
   decisions-log entry, numbered against its own real sequence — confirm the next free number before
   writing anything) and presents it for review in the same conversation D-32 was decided in, before
   committing anything.
2. Owner reviews, resolves the open question above (duplicate-recipient/payer restriction) and
   confirms or revises the proposed 1,000-leg cap.
3. Once ratified, this task file is updated to `done`, and a separate build task (a new T-number) is
   written and dispatched to a builder.

## Out of scope for this task

- No code changes. This task produces a decision, nothing else.
- No change to `OrchestratorApi`, `ProductApi`, or any other service.
- Does not reopen or re-derive T-058's own Gap 3 finding, or T-066's own group-lending findings.
- Does not design the eventual chunked-batch mechanism for a genuinely larger-than-the-cap batch —
  named as a real, acknowledged future limit, not solved here.
