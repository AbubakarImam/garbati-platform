# T-063: LedgerApi real-payload gaps design (D-31)

**Status: done, ratified 3 Nov 2026.** Written as LedgerApi decisions #49-#52
(`LedgerApi/docs/DESIGN.md`, commit `f3aa1ff`), `Garbati.OrchestratorApi` decision #11 (commit
`e6ed2a5`), and the decision-#13 citation fix in `Garbati.IdentityApi/docs/DESIGN.md` (commit
`2470b05`). Next step per this task's own process (step 3): a separate build task, not yet written.

Owner: coordinator + you. Design task — produces decisions-log entries in `LedgerApi/docs/DESIGN.md`
and a small forwarding change spec for `OrchestratorApi`. **No code gets written from this task
directly** — once ratified, a separate build task goes to a builder, the same two-step pattern T-060
→ T-061 just used for the chart of accounts.

## Context

`research/real-corebanking-payload-gap-analysis.md` (T-058) triangulated six gaps found by comparing
a real production transfer payload (Sterling Bank MIZAN/SEABASS via KUSALA) against GARBATI's actual
code, against Apache Fineract and Oracle FLEXCUBE. The owner chose to close four of the six now
(D-31): value-date/multi-stage timestamps, branch attribution, category/source/channel attribution,
and named-actor traceability — explicitly **not** multi-entry (N:M) posting (conditional on an
undecided product-scope question) and **not** a literal human-name column anywhere in LedgerApi
(see the correction below).

**Correction made during scoping, before any design was drafted**: the original framing of the
named-actor gap ("GARBATI never persists who approved a transaction, only an ephemeral signed grant")
turns out to overstate the gap. Read directly against the real code:
`Garbati.IdentityApi`'s `AuthorizationDecision.RequestedBy` and, for joint accounts,
`ApprovalRecord.CustomerId` (one durable row per holder's approval, never deleted) **already persist
human-identifiable actors**, durably and queryably. The thing IdentityApi deliberately never persists
is narrower: the signed grant *token* itself (commonly, if inconsistently, cited in this codebase as
"decision #13" — see the documentation note below). **The real, remaining gap is on LedgerApi's own
side**: `transactions` has no reference back to the authorization decision at all. And the fix is
already half-built: `Garbati.OrchestratorApi`'s `TransferService.cs`/`ReversalService.cs` already
receive `authorization.Decision.DecisionId` from IdentityApi on every call — they just never forward
it to LedgerApi, using it only in their own audit logs. Closing this gap needs no new outbound call
from LedgerApi (decision #34 stays intact) and no duplicated human-readable data inside LedgerApi's
own database (P-5 stays intact) — just an opaque reference string, the same pattern Fineract's own
`acc_gl_journal_entry.createdby_id` FK uses (an id, not a denormalized name).

**A separate, small documentation bug found while tracing this, worth fixing in the same pass**:
`Garbati.IdentityApi/docs/DESIGN.md`'s decisions-log table numbers row 13 as "Audit history," but
several places in the same document's own body prose (e.g. the T-049/D-16 entry, `AuthorizationDecision.cs`'s
own code comment) cite "decision #13" to mean "the signed grant is never persisted" — a citation that
doesn't match the table. Decision #29's own "Numbering note" already fixed one round of this exact
class of drift in 2026; this is a fresh instance introduced since. Fix: a short numbering-note addendum
in `Garbati.IdentityApi/docs/DESIGN.md` pointing body citations at the right row (likely decision #4,
"Authorization evidence," combined with the general grant-lifecycle design in Section 6.2 — confirm the
right row when fixing), not a renumbering of the table itself.

## What this design must resolve (draft proposal below, for review and revision, not final)

1. **Value date** (Gap 1). A new nullable `value_date` (DATE, not a timestamp — value date is a
   calendar-day concept in every source checked) column on `transactions` and `transaction_chains`.
   Caller-supplied and optional; defaults to the transaction's own creation date if omitted (same-day
   value — zero behavior change for every existing caller). **Proposed, needs explicit sign-off**: for
   v1, the column exists and is stored, but a caller-supplied `value_date` different from the actual
   creation date is rejected (422) — this closes the schema gap (the field real systems have) without
   opening actual back/forward-value-dating as a capability, which is a separate, real business
   decision (it touches interest/settlement timing and has real fraud-reporting implications no one has
   asked to open yet). Loosening this later is a small, additive change; tightening it after callers
   depend on back-dating would not be.
2. **Branch attribution** (Gap 2). A new nullable `branch_code` (string) column on `transactions` and
   `transaction_chains`. Caller-supplied, optional. **Deliberately not populated by anything today** —
   no branch/office registry exists anywhere in the platform yet (`research/teller-till-vault-cash-operations.md`,
   T-056, confirmed zero till/vault/branch concept). Same "seed the shape, leave it correctly unused"
   discipline already used for `FeeRule`/`ProfitSharingTerms` (ProductApi) and the GL-account backfill's
   untouched System accounts (decision #48) — a future branch/teller design is the eventual real caller.
3. **Channel attribution** (Gap 5, narrowed). T-058 found the underlying need real but "no single
   universal shape" for a category/source/channel triple — several real platforms use different,
   specifically-purposed fields rather than one taxonomy. **Scoping decision, flagged for review**:
   build only the one piece with a clear real caller today — a single nullable `channel` (string,
   open-ended, e.g. `"OpsConsole"`, `"Api"`) on `transactions`/`transaction_chains` — and leave
   "category"/"source" as a separate, future question once there's an actual second channel (a real
   customer-facing one) to distinguish from the ops console's own testing console. Building a
   three-field taxonomy today, with exactly one real caller (`OpsApi`'s testing console) and one real
   future caller (`OrchestratorApi`/a channel that doesn't exist yet), risks inventing categories
   nothing will ever populate correctly — the same trap `FeeRule`/`ProfitSharingTerms` avoided by
   shipping empty rather than guessed.
4. **Decision-id traceability** (Gap 6, corrected scope — see above). A new nullable `decision_id`
   (string) column on `transactions` and `transaction_chains`. `Garbati.OrchestratorApi`'s
   `TransferService`/`ReversalService` forward `authorization.Decision.DecisionId` (which they already
   receive and already log, just never send onward) in the existing `TransferRequest`/`ChainRequest`
   payload to LedgerApi. LedgerApi stores it as an opaque string — it does not look it up, does not
   call IdentityApi, and does not learn or store any human name. "Who approved this" becomes answerable
   by following `decision_id` back to IdentityApi's own already-durable `AuthorizationDecision`/
   `ApprovalRecord` rows — a cross-service join a human operator or a future audit tool performs, not
   a live runtime dependency LedgerApi carries.

## What this design explicitly does not change

- **Decision #1 (derived balance), #7 (row locking), #34 ("LedgerApi calls nobody"), #45 (chains of
  legs), P-5 (no shared database).** All four additions above are nullable, caller-supplied-or-defaulted
  columns on existing tables — no new outbound call, no new cross-service data duplication, no change
  to how a balance is computed or a chain is posted.
- **No multi-entry (N:M) posting** (Gap 3) — explicitly deferred, conditional on an undecided
  retail-vs-bulk product-scope question T-058 itself flagged, not this task's call to make.
- **No literal human name anywhere in LedgerApi's own database** — `decision_id` is an opaque
  reference, not a denormalized copy of IdentityApi's `RequestedBy`/`ApprovalRecord.CustomerId`.
- **No category/source taxonomy** beyond the single `channel` field — deliberately narrowed, per item 3
  above, pending a real second channel to design against.

## Process

1. Coordinator drafts the full design (this file, expanded into real decisions-log entries in
   `LedgerApi/docs/DESIGN.md`, numbered against its own real sequence — confirm the next free number
   before writing anything) and a short, separate note for `Garbati.OrchestratorApi/docs/DESIGN.md`
   covering the `decision_id`-forwarding change, and presents both for review in the same conversation
   D-31 was decided in, before committing anything.
2. Owner reviews the two flagged sub-questions (value-date back/forward-dating restriction; narrowing
   category/source/channel down to just `channel`) specifically, plus anything else that stands out.
3. Once ratified, this task file is updated to `done`, and a separate build task (a new T-number) is
   written and dispatched to a builder — touching `LedgerApi` (the four new columns, validation,
   migration) and `Garbati.OrchestratorApi` (forwarding `decision_id`), the same two-repo-one-task shape
   T-046 (ProductApi↔OrchestratorApi) already used.

## Out of scope for this task

- No code changes. This task produces decisions and a short design note, nothing else.
- No change to `ProductApi`, `StaffIdentityApi`, or `OpsApi` in this task.
- Does not reopen or re-derive T-058's own six-gap findings, or re-litigate Gap 3's deferral.
