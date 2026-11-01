# T-058: Research — triangulate a real core-banking transfer payload's gaps against GARBATI, using
1-2 other named platforms

## Context

The owner shared a real sample transfer-posting payload from a production core-banking integration they
work with professionally (Sterling Bank's MIZAN/SEABASS platforms, integrated via the KUSALA BPM system —
the owner's own employer, per their portfolio bio). This is real, first-hand ground truth, not a generic
example, and the coordinator already did a direct, field-by-field comparison against GARBATI's actual
current code (`TransferRequest`/`ChainRequest` contracts, `Transaction`/`LedgerEntry` entities, read
directly in both `Garbati.OrchestratorApi` and `LedgerApi`) before this task was written. **Do not re-derive
that comparison from scratch** — it is reproduced in full below as established fact. This task's job is
narrower: check whether 1-2 other real, named, documentable core-banking platforms handle the same six
gaps the same way, to triangulate whether they're common, load-bearing core-banking concepts GARBATI is
genuinely missing, or something narrower/more specific to the MIZAN/SEABASS/KUSALA integration pattern
specifically.

## The real payload (reproduced exactly, account numbers already placeholder values in the original)

```json
{
  "requestId": "7c81efc6-e2f4-4796-bec5-3f3ce98b5fed",
  "currencyCode": "NGN",
  "postedBy": "KUSALA",
  "approvedBy": "Test Approver-Kusala",
  "approvedDate": "{{today}}",
  "postedDate": "{{nowIso}}",
  "multipleDebit": 0,
  "valueDate": "{{today}}",
  "reconciliationRemarks": "Postman test",
  "isFuture": false,
  "transactionCategory": 0,
  "transactionSource": 1,
  "transactionSubSource": 1,
  "thirdPartySource": "Internal posting",
  "creditEntries": [
    { "creditAccountNumber": "<CREDIT ACCOUNT>", "creditAmount": 0, "creditNarration": "Postman test" }
  ],
  "debitEntries": [
    { "debitAccountNumber": "<DEBIT ACCOUNT>", "debitAmount": 0, "debitNarration": "Postman test" }
  ],
  "InitiatedBy": "Test User-KUSALA",
  "InitiatedByBranchCode": "001",
  "PostedByBranchCode": "001",
  "ApprovedByBranchCode": "HQ",
  "InitiatedDate": "{{nowIso}}"
}
```

## The six gaps already found, directly, against GARBATI's real code — established fact, not to be re-derived

1. **No value-date / multi-stage timestamp concept at all.** The payload tracks four distinct moments —
   `InitiatedDate`, `approvedDate`, `postedDate`, `valueDate` (the accounting-effective date, which can be
   backdated or forward-dated, with an explicit `isFuture` flag) — while GARBATI's `Transaction` entity
   (`LedgerApi/Entities/Transaction.cs`) has only `CreatedAt`/`CompletedAt`, because GARBATI posts
   synchronously with no settlement delay. **This is the same underlying gap `research/modern-treasury-gap-analysis.md`
   (T-055) already found in Modern Treasury's own pending/posted/available balance tiers** — a third,
   independent confirmation of the same structural fact from a real production system, not a new, isolated
   finding.
2. **No branch concept anywhere.** `InitiatedByBranchCode`/`PostedByBranchCode`/`ApprovedByBranchCode`
   (note `"HQ"` as a real value — approval happening somewhere different from initiation) versus zero
   branch representation anywhere in GARBATI — **the same finding `research/teller-till-vault-cash-operations.md`
   (T-056) already made independently.**
3. **No multi-entry (N-debit-to-M-credit) posting shape.** The payload's `creditEntries`/`debitEntries` are
   arrays with an explicit `multipleDebit` flag. GARBATI's `TransferRequest` is strictly one debit/one
   credit, and even `ChainRequest`'s individual legs (`LedgerApi/Services/Implementations/LegPoster.cs`)
   are each still exactly one debit + one credit — confirmed directly, not assumed.
4. **No per-entry narration, no reconciliation-remarks field.** The payload gives each entry its own
   narration plus a top-level `reconciliationRemarks` field; GARBATI's `Transaction.Narration` is one string
   for the whole transaction, `LedgerEntry` has no narration field at all.
5. **No transaction category/source/channel attribution.** `transactionCategory`/`transactionSource`/
   `transactionSubSource`/`thirdPartySource` classify where a transaction came from (branch teller vs.
   mobile app vs. BPM automation), independent of what it's for. GARBATI's only classification dimension is
   `LegPurpose` (`Principal`/`Fee`/`Tax`), which describes a leg's role, not the transaction's origin.
6. **No named actor tracking separate from the cryptographic grant.** The payload persists `InitiatedBy`/
   `approvedBy`/`postedBy` as plain, queryable strings (the last being a system name, the other two human
   names). GARBATI's approval lives entirely in IdentityApi's signed grant token, deliberately never
   persisted (decision #13) — only the `DecisionId` survives — so GARBATI cannot currently answer "who
   approved this, by name" from the ledger alone, a real, deliberate trade-off with a real cost.

**Where GARBATI already matches real practice, confirmed, not a gap:** one currency per transaction (the
payload's single top-level `currencyCode` matches GARBATI's own same-currency-per-chain rule); an
idempotency-key concept exists on both sides (`requestId` here, `IdempotencyKey` in GARBATI), functionally
equivalent despite different naming.

## What to research

For each of the six gaps above, check whether 1-2 other real, named, documentable core-banking platforms
(Apache Fineract and Oracle FLEXCUBE are good starting candidates — both have proven to have readable
public documentation in `research/teller-till-vault-cash-operations.md` and
`research/multi-charge-transaction-modeling.md`; Temenos' own documentation has repeatedly required
authenticated-community-login access in prior research and is a weaker candidate unless a public page is
found) handle the same concept, and how:

1. **Value date / multi-stage transaction lifecycle timestamps** — is this a standard, close-to-universal
   core-banking concept (expect yes, but confirm from a real source rather than assume), and how is it
   typically represented (fields on the transaction itself, vs. a separate status-history/audit-trail
   object)?
2. **Branch-code attribution per lifecycle stage** — is tracking *where* each stage of a transaction
   happened (not just who) a common pattern, or specific to institutions with many physical branches (which
   would still be directly relevant to GARBATI's own target market)?
3. **Multi-entry (N:M) posting** — how common is this in real core-banking transaction APIs, and what's the
   real-world reason for it (e.g., payroll fan-out, bulk collections) that would or wouldn't apply to
   GARBATI's own microfinance/retail-transfer use case specifically?
4. **Per-entry narration and a dedicated reconciliation-remarks field** — confirm whether this is a common,
   low-cost, high-value pattern (a cheap field to add) or whether real systems handle "why was this posted"
   differently (e.g., a separate transaction-memo/reference-code system).
5. **Transaction category/source/channel attribution** — how do real systems commonly classify transaction
   origin, and is this usually a flat numeric code (as in the sample payload) or a richer, named taxonomy?
6. **Named actor persistence vs. a signed/cryptographic authorization record** — this is the most
   architecturally loaded of the six: does checking 1-2 real systems suggest GARBATI's choice (never
   persist the grant, only the decision id) is unusual, or do other systems also separate "proof of
   authorization" from "a human-readable audit record of who approved," keeping both independently? This
   directly affects whether GARBATI's decision #13 trade-off is costing something real systems don't
   normally give up.

## Explicitly out of scope

- Do not design a schema, propose new fields for `Transaction`/`LedgerEntry`/any request contract, or
  write a task file for closing any of the six gaps — this is triangulation research only.
- Do not re-derive the six gaps themselves or re-read GARBATI's own code to re-confirm them — they are
  established fact per this task file, reproduced directly from the coordinator's own prior analysis.
- Do not assume MIZAN/SEABASS's own specific field choices are the "correct" standard GARBATI should match
  exactly — the point is checking whether these are common, load-bearing core-banking concepts generally,
  not replicating one specific institution's own API contract.

## Output

A sourced brief at `research/real-corebanking-payload-gap-analysis.md`, following this project's existing
research-brief conventions (see `research/multi-charge-transaction-modeling.md` for the most recent example
of tone, sourcing discipline, and honest confidence-calibration). Structure it around the six gaps above,
each with: what was found from 1-2 other real platforms, and an honest verdict (confirmed as a common,
load-bearing core-banking pattern GARBATI is genuinely missing / narrower or institution-specific than it
first appeared / could not be confirmed from an accessible source). End with a short synthesis — of the six,
which look like they matter most for a future design task to prioritize, based only on how commonly
confirmed each one is across the platforms actually checked, not a personal recommendation.
